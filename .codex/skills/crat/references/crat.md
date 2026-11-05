# Crat pass and analysis CLI

## Contents

- Entrypoint and compiler utilities
- Structural passes
- Safety and interface passes
- Data-representation passes
- Library and cleanup passes
- Standalone analyses
- Finder and merge tools

## Entrypoint and compiler utilities

- Start with `src/bin/crat.rs` when changing pass order, CLI/config wiring, project copying, dependency changes, or output behavior.
- All paths and commands in this reference are relative to the Crat root, `proctor/stages/crat/` in the workspace. `rust-toolchain.toml` pins `nightly-2025-06-23` with rustc development components; use that toolchain for compiler-private APIs.
- Ownership analysis also requires native Z3 development dependencies (`libz3-dev` on Ubuntu/Debian); check `README.md` and both pointer crates' `Cargo.toml` files for setup.
- Expect transformation passes `Expand`, `Preprocess`, `Prepare`, `Extern`, `Unsafe`, `Unexpand`, `Split`, `Bin`, `Check`, `Format`, `Interface`, `Libc`, `OutParam`, `Lock`, `Union`, `Punning`, `Enum`, `Io`, `Pointer`, `PointerBo`, `PointerStructArrays`, `PointerStructParamFields`, `PointerEpochSplit`, `PointerArrayLocalProvenance`, `Static`, and `Simpl`; expect standalone analyses `Andersen` and `OutParam`.
- The CLI locates the crate library, optionally copies the input project, and invokes most operations through `utils::compilation::run_compiler_on_path`.
- TOML configuration supplies pass-specific defaults; CLI pass lists extend configured lists, boolean options enable configured flags, and supplied scalar options override them. Passes run in the requested order. Choose either transformation passes or one standalone analysis, never both.
- Keep dependency side effects at the pass's publication boundary. Pass results currently request `bytemuck`, `num-traits`, `tempfile`, or `proctor-libc` as needed; `pointer-bo` publishes source and required bytemuck dependencies through its library entrypoint.
- Propagate `c_exposed_fns` through interface, unsafe, `pointer`, `pointer-bo`, Andersen, outparam, and punning configs. Use `points_to_file` to connect Andersen output to union and outparam logic.
- Compiler helpers resolve extern crates from `deps_crate/target/<host>/debug/deps`, rather than the input project's Cargo build. Build `deps_crate` with an explicit host target before compiler-backed tests or analyses, as in `.github/workflows/ci.yml`; use `DIR=<Crat-root>` when invoking helpers away from their supported working directories. Do not assume building the main workspace prepares this excluded crate.

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
- The CLI removes superseded Rust source files while retaining explicitly declared in-project Cargo binary paths.

### Preprocess

- Goal: simplify C2Rust artifacts before heavier analyses.
- File/entry: `crates/passes/src/preprocessor.rs`; `preprocessor::preprocess(tcx) -> String`.
- Analyze parameter/local use, pointer use, call arguments, string statics, and AST-to-HIR identities.
- Transform repeated assertions, unreachable and constant-false code, nested `unwrap`, pointer/API argument forms, pointer-offset chains, FILE aliases, byte-string transmutes, C `offsetof`, and inline numeric-conversion functions.

### Prepare

- Goal: normalize compiler-resolved source for local transformation.
- File/entry: `crates/passes/src/preparer.rs`; `preparer::prepare(tcx) -> Result<PreparationResult, PrepareError>`.
- Wrap non-block match arms, lift function-local statics to their owning module and rewrite resolved references, remove ignored `setlocale` calls, and rewrite supported ctype-table accesses through `proctor-libc`.
- Publish a successful result through `PreparationResult::publish`, which ensures the optional `proctor-libc` dependency before writing source. Preparation errors leave source unchanged.

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
- Consume `field_spec_map.json` from pointer struct-parameter specialization to recover C-facing field wrappers, leaving the specialized implementation in place. This wrapper pass is separate from `crat-tool` finalization, which creates no general API wrappers.

## Data-representation passes

### Pointer and PointerBo

- `--pass pointer` uses `crates/pointer_replacer` and `[pointer]`; see [pointer.md](pointer.md) for its preprocessing, shared inference, expression rewriting, and skeleton consumer.
- `--pass pointer-bo` uses `crates/pointer_replacer_bo` and `[pointer-bo]`; see [pointer-bo.md](pointer-bo.md) for the coupled borrow/ownership solver and its separate verified rewrite pipeline.

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

## Finder and merge tools

- `src/bin/crat-finder.rs` dispatches the `Example`, `Macro`, `Mapper`, `Mir`, `Unsafe`, and `Void` inspectors in `crates/finders/`. Use them for compiler structure or unsafe-operation inspection; `Void` emits JSON findings.
- `src/bin/crat-merge.rs` calls `crates/merger/src/lib.rs` to merge variants listed in `translations.json`. The merger checks shared Cargo/build/toolchain files and combines Rust items under configuration-derived features. It writes a project and is independent of local SCC assembly.

## Focused verification

- Run the affected crate's tests first, for example `cargo test -p passes`, `cargo test -p io_replacer`, or `cargo test -p union_replacer`. Use `cargo test -p passes preparer::tests` for `Prepare`.
- Run `cargo test --bin crat` for CLI parsing and configuration. For shared `utils`, pass interactions, or public cross-crate APIs, select coverage across the affected crates while respecting the skill's filesystem constraints; the BO suite includes fixtures that write temporary trees.
