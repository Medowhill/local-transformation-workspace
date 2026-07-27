# Crat pass and analysis CLI

## Contents

- Entrypoint and compiler utilities
- Structural passes
- Safety and interface passes
- Data-representation passes
- Library and cleanup passes
- Standalone analyses

## Entrypoint and compiler utilities

- Start with `src/bin/crat.rs` when changing pass order, CLI/config wiring, project copying, dependency changes, or output behavior.
- Expect transformation passes `Expand`, `Preprocess`, `Extern`, `Unsafe`, `Unexpand`, `Split`, `Bin`, `Check`, `Format`, `Interface`, `Libc`, `OutParam`, `Lock`, `Union`, `Punning`, `Enum`, `Io`, `Pointer`, `Static`, and `Simpl`; expect standalone analyses `Andersen` and `OutParam`.
- The CLI locates the crate library, optionally copies the input project, and invokes most operations through `utils::compilation::run_compiler_on_path`.
- Keep dependency side effects in the entrypoint. Pass results currently request `bytemuck`, `num-traits`, or `tempfile` as needed.
- Propagate `c_exposed_fns` through interface, unsafe, pointer, Andersen, outparam, and punning configs. Use `points_to_file` to connect Andersen output to union and outparam logic.

Use these common rustc patterns:

- Use `utils::compilation::{run_compiler_on_path, run_compiler_on_str}` for edition-2021 rustc setup, rlib compilation, `deps_crate` externs, and error-only diagnostics.
- Use `utils::ast::expanded_ast(tcx)` and `utils::ast::make_ast_to_hir` when surface AST nodes must map to HIR; print with `rustc_ast_pretty::pprust`.
- Implement AST rewrites with `rustc_ast::mut_visit::MutVisitor`; use `utils::{expr, item, items, stmt, pat, ty, param, attr}` for parsed snippets.
- Use `tcx.hir_visit_all_item_likes_in_crate`, `rustc_hir::intravisit::Visitor`, `nested_filter::OnlyBodies`, and `tcx.typeck(owner)` for HIR analysis.
- Use `tcx.mir_drops_elaborated_and_const_checked`, `tcx.optimized_mir`, or `tcx.mir_for_ctfe` plus `rustc_mir_dataflow` for MIR analysis.
- Reuse `utils::ir` mappings such as `AstToHir`, `map_hir_to_thir`, `map_thir_to_mir`, `mir_ty_to_string`, `def_id_to_symbol`, `ty_size`, `array_of_as_ptr`, and `file_param_index`.
- Call `utils::ast::remove_unnecessary_items_from_ast` before printing when following an existing transform's pattern.

## Structural passes

### Expand

- Goal: normalize a translated crate into one expanded file with required feature and allow attributes.
- File/entry: `crates/passes/src/expander.rs`; `expander::expand(Config { keep_allows }, tcx) -> String`.
- Steal the lowering resolver AST, clear crate attributes, insert Crat's features, rename a module named `mod` to `rmod`, rewrite selected expanded paths, remove `bitfield` field attributes, and print the crate.

### Preprocess

- Goal: simplify C2Rust artifacts before heavier analyses.
- File/entry: `crates/passes/src/preprocessor.rs`; `preprocessor::preprocess(tcx) -> String`.
- Analyze parameter/local use, pointer use, call arguments, string statics, and AST-to-HIR identities.
- Transform repeated assertions, unreachable and constant-false code, nested `unwrap`, pointer/API argument forms, pointer-offset chains, FILE aliases, byte-string transmutes, C `offsetof`, and inline numeric-conversion functions.

### Extern

- Goal: replace duplicate foreign declarations with imports of local definitions.
- Files/entry: `crates/passes/src/extern_resolver/{mod.rs,cmake_reply.rs}`; `resolve_extern(&Config, tcx) -> String`.
- Group candidates by symbol and structural type, resolve representatives through hints or CMake/source priority, remove duplicates, rewrite paths, and add module-local imports.
- Important config: `cmake_reply_index_file`, `build_dir`, `source_dir`, `function_hints`, `static_hints`, `type_hints`, `choose_arbitrary`, `ignore_return_type`, and `ignore_param_type`.

### Unexpand

- Goal: restore selected macros and derives after expanded transformations.
- File/entry: `crates/passes/src/unexpander.rs`; `unexpander::unexpand(Config { use_print }, tcx) -> String`.
- Restore derives, bytemuck derives, bitfield attributes, thread locals, panic/format/write/print macros, `offset_of!`, and slice-cursor indexing macros; remove unused feature attributes.

### Split

- Goal: split one expanded source file into a module tree.
- File/entry: `crates/passes/src/splitter.rs`; `splitter::split(dir, lib_name)`.
- Write inline modules as `<name>.rs` or `mod.rs` and replace them with `pub mod` declarations. This pass mutates the project tree instead of returning code.

### Bin

- Goal: generate Cargo binary targets for translated `main` functions.
- File/entry: `crates/passes/src/bin_file_adder.rs`; `add_bin_files(dir, &Config, tcx)`.
- Find nonignored `main` functions, write wrapper sources, and append `[[bin]]` tables; use the configured name only for a single discovered binary.

### Check and Format

- `Check` invokes `run_compiler_on_path(&file, utils::type_check)`; the empty callback relies on rustc parsing, lowering, and type checking.
- `Format` invokes `formatter::format(tcx)`, which applies Crat's AST transformation/writeback path.

## Safety and interface passes

### Unsafe

- Goal: remove unnecessary unsafe, public, extern, and optionally unused surface while preserving C boundaries.
- File/entry: `crates/passes/src/unsafe_resolver/mod.rs`; `resolve_unsafe(&Config, tcx) -> String`.
- Build the item-use graph, propagate unsafe calls through SCCs, retain functions with non-call unsafe operations, remove unused items, narrow visibility, and strip configured ABI/export surface.
- Important config: `remove_unused`, `remove_no_mangle`, `remove_extern_c`, `replace_pub`, and `c_exposed_fns`.

### Interface

- Goal: preserve raw C interfaces around exposed functions whose internal parameters became slices or slice cursors.
- File/entry: `crates/passes/src/interface_fixer.rs`; `fix_interfaces(&Config, tcx) -> String`.
- Rename the implementation to `<name>_internal`, generate the raw-pointer wrapper at the exposed name, and rewrite Rust paths and calls to the implementation.
- Convert null pointers to empty slices/cursors and nonnull pointers with the provisional `1_000_000` bound.

## Data-representation passes

### OutParam

- Goal: convert pointer output parameters into returned values, `Option`, or `Result`.
- Files/entry: `crates/outparam_replacer/src/{ai/*,transform.rs}`; `transform(tcx, &Config, verbose) -> String`.
- Use Andersen-backed alias information and MIR abstract interpretation to identify must/may outputs, complete writes, return patterns, and removable checks; rewrite signatures, bodies, and call sites.

### Lock

- Treat `Pass::Lock => todo!()` as a placeholder. Clarify the intended lock transformation semantics before implementation.

### Union

- Goal: recover tagged unions as enums.
- Files/entry: `crates/union_replacer/src/{tag_analysis.rs,must_analysis/*,ty_finder.rs,util/*}`; `tag_analysis::analyze(&Config, verbose, tcx) -> Statistics`.
- Combine Andersen may-points-to results with MIR must-analysis, infer tag/field relations, replace definitions, add accessors and tag setters, and rewrite aggregates and accesses.

### Punning

- Goal: replace safe-enough type-punning unions with aligned raw-byte structs and typed accessors.
- Files/entry: `crates/union_replacer/src/punning/*`; `punning::replace_unions(tcx, verbose, &Config) -> TransformationResult`.
- Classify field types by bytemuck traits, analyze reaching writes and call contexts, skip unsafe shapes, generate raw storage/accessors, and request bytemuck derives when needed.

### Enum

- Goal: replace C2Rust integer aliases and typed constants that model C enums with fieldless Rust enums.
- Files/entry: `crates/passes/src/enum_replacer/{mod.rs,tests.rs}`; `replace_enums(tcx) -> String`.
- Analyze enum flow and reject unsafe integer uses, then replace aliases/constants, reexport variants, and insert required casts.

### Static

- Goal: replace eligible `static mut` values with immutable statics, thread-local `Cell`, or thread-local `RefCell`.
- Files/entry: `crates/static_replacer/src/{lib.rs,return_escape.rs,transformation.rs}`; `replace_static(tcx) -> String`.
- Leave escaping statics unchanged; classify remaining uses, rewrite reads/writes/borrows, and add `never_type`, `thread_local_internals`, and `as_array_of_cells` when required.

## Library and cleanup passes

### Libc

- Goal: replace supported libc APIs with Rust standard-library code and small `c_lib` helpers.
- Files/entry: `crates/passes/src/libc_replacer/*`; `replace_libc(tcx) -> TransformationResult`.
- Rewrite character, math, process, string, parse, errno, `memcpy`, and `memset` patterns; insert only used helper items.
- Handle result flags `bytemuck`, `bytemuck_derive`, and `num_traits` in the CLI.

### Io

- Goal: replace C `FILE*` and stdio operations with Rust stream types and helpers.
- Files/entry: `crates/io_replacer/src/{file_analysis.rs,error_analysis.rs,transformation/*}`; `replace_io(Config, tcx) -> TransformationResult`.
- Analyze stream origins, permissions, unsupported flows, and error indicators; rewrite stream types and supported calls and return dependency, unsupported-reason, timing, and analysis statistics.

### Simpl

- Goal: remove lightweight artifacts after other transforms.
- Files/entry: `crates/passes/src/{simplifier.rs,simplifier/unused_assignments.rs}`; `simplifier::simplify(tcx) -> String`.
- Use MIR `MaybeLiveLocals` to remove unused assignments and simplify casts, parentheses, paths, FILE/libc aliases, zero-erasing operations, and unsigned comparisons with zero.

## Standalone analyses

### Andersen

- Files/entry: `crates/points_to/src/{andersen.rs,alloc_finder.rs}`; `points_to::andersen::run_analysis(&Config, tcx) -> Solutions`.
- Build typed locations, allocation/call/field metadata, generate Andersen address/copy/load/store/call constraints, solve may-points-to sets, and postcompute deref edges, writes, indirect calls, SCCs, and function write sets.
- Important config: `use_optimized_mir` and `c_exposed_fns`.

### OutParam analysis

- Use `outparam_replacer::ai::analysis::analyze` to run the standalone output-parameter analysis and `write_analysis_result` to emit JSON.
- Expect Andersen points-to data, alias checks, call-graph SCC summaries, loop widening/state caps, return ranges, and MIR abstract values.
- Important config: `max_loop_head_states`, `check_global_alias`, `check_param_alias`, `no_widening`, `points_to_file`, and debug timing/printing options.

## Focused verification

- Run the affected crate's tests first, for example `cargo test -p passes`, `cargo test -p io_replacer`, or `cargo test -p union_replacer`.
- Run `cargo test --workspace` when changing the CLI, shared `utils`, pass interactions, or public cross-crate APIs.
