---
name: crat
description: "Domain guide for working on Crat under `crat`: C2Rust translation improvement passes, plugin pipeline, rustc-private APIs, static analyses, and conventions. Use when modifying or understanding Crat passes, analyses, configs, generated Rust transformations, or Crat's use of rustc APIs."
---

# Crat

## Ground Rules

- Treat `crat/` as the Crat source tree and as an independent git repo. Commit or inspect git state inside `crat/` when the task is about Crat, not at the workspace root.
- Read `crat/src/bin/crat.rs` first when changing passes, CLI/config wiring, dependencies, or output behavior.
- Keep changes surgical. Most passes are rustc-private AST/HIR/MIR transforms over C2Rust output, so preserve local style and invariants.
- Crat improves C2Rust translations through multiple passes and analyses. Many passes write transformed `lib.rs`; some mutate the project tree or Cargo.toml.
- After fixing Crat code, run validation from `crat/`: `cargo fmt`, `cargo clippy --workspace --all-targets`, and `cargo test --workspace`.

## Entrypoint

- Transformation passes in `crat/src/bin/crat.rs`: `Expand`, `Preprocess`, `Extern`, `Unsafe`, `Unexpand`, `Split`, `Bin`, `Check`, `Format`, `Interface`, `Libc`, `OutParam`, `Lock`, `Union`, `Punning`, `Enum`, `Io`, `Pointer`, `Static`, `Simpl`.
- Analyses in `crat/src/bin/crat.rs`: `Andersen`, `OutParam`.
- The CLI finds the crate lib path, optionally copies input to output, then calls `utils::compilation::run_compiler_on_path(&file, |tcx| ...)` for each pass.
- Dependency side effects are centralized in the entrypoint: `bytemuck`, `num-traits`, and `tempfile` are added only when pass results request them.
- `c_exposed_fns` is propagated into passes/analyses that must preserve C ABI boundaries: interface, unsafe, pointer, Andersen, outparam, and punning.
- `points_to_file` connects Andersen output to union and outparam analyses.

## Common rustc APIs

- Use `utils::compilation::{run_compiler_on_path, run_compiler_on_str}` for rustc setup. It drives `rustc_interface::run_compiler`, creates the global context, uses edition 2021, sets rlib crate type, loads deps from `deps_crate/target/debug/deps`, and suppresses warnings through a custom emitter.
- For AST transforms, use `utils::ast::expanded_ast(tcx)`, then `utils::ast::make_ast_to_hir(&mut krate, tcx)` when AST nodes must map back to HIR, and print with `rustc_ast_pretty::pprust`.
- AST visitors usually implement `rustc_ast::mut_visit::MutVisitor`. Helper macros such as `utils::{expr, item, items, stmt, pat, ty, param, attr}` parse tiny Rust snippets into AST.
- HIR analysis usually uses `tcx.hir_visit_all_item_likes_in_crate`, `rustc_hir::intravisit::Visitor`, `NestedFilter = nested_filter::OnlyBodies`, `maybe_tcx = self.tcx`, and `tcx.typeck(owner)` for `expr_ty`, `expr_ty_adjusted`, and `node_type`.
- MIR analysis uses `tcx.mir_drops_elaborated_and_const_checked(def_id)`, `tcx.optimized_mir(def_id)`, or `tcx.mir_for_ctfe(def_id)` plus `Body`, `Local`, `Location`, `BasicBlock`, `StatementKind`, `TerminatorKind`, and `rustc_mir_dataflow`.
- Mapping helpers in `utils::ir` are central: `AstToHir`, `map_hir_to_thir`, `map_thir_to_mir`, `mir_ty_to_string`, `def_id_to_symbol`, `ty_size`, `array_of_as_ptr`, and `file_param_index`.
- Common keys are `LocalDefId`, `DefId`, `HirId`, `Span`, `Local`, and `Location`; common collections are `FxHashMap`, `FxHashSet`, `IndexVec`, `DenseBitSet`, and `ChunkedBitSet`.
- Many transforms call `utils::ast::remove_unnecessary_items_from_ast` before pretty-printing or applying changes.

## Expand

- Goal: turn a translated crate into a normalized single-file form with required feature/allow attributes.
- Files: `crat/crates/passes/src/expander.rs`; entry `expander::expand(Config { keep_allows }, tcx) -> String`.
- Operations: steals the lowering resolver AST with `tcx.resolver_for_lowering().steal()`, clears crate attrs, inserts warning/allow and feature attrs such as `c_variadic`, `extern_types`, `thread_local`, `rustc_private`, and `core_intrinsics`.
- Operations: renames module `mod` to `rmod`, rewrites expanded intrinsic/format paths, removes field attrs named `bitfield`, removes unnecessary items, and prints with `pprust::crate_to_string_for_macros`.
- Important APIs/types: `MutVisitor`, `Crate`, `ItemKind`, `AttrStyle`, `Config.keep_allows`.

## Preprocess

- Goal: simplify C2Rust artifacts before heavier analyses.
- File: `crat/crates/passes/src/preprocessor.rs`; entry `preprocessor::preprocess(tcx) -> String`.
- Analysis: a HIR visitor records param/local use, pointer uses, string literal statics, call args, fresh pointer lets, and HIR ids needed by the AST visitor.
- Transformations: deduplicate repeated assert blocks; truncate unreachable code; drop constant-false dead code; simplify nested `unwrap`; remove param-assigned vars; hoist pointer/API/bitfield args; merge pointer offset chains.
- Transformations: replace FILE function pointer aliases, remove `let ref`, convert byte-string transmutes, replace C `offsetof` patterns with `core::mem::offset_of!`, and replace inline `atoi`/`atol`/`atof` with extern declarations.
- Important helpers: `try_replace_offsetof`, `try_merge_pointer_offsets`, `hoist_self_ref_place_expr`, `replace_inline_extern_fns`, `transmute_expr`, `eval_expr`.

## Extern

- Goal: replace duplicate `extern` declarations with `use crate::...` references to local definitions.
- Files: `crat/crates/passes/src/extern_resolver/mod.rs`, `cmake_reply.rs`; entry `extern_resolver::resolve_extern(&Config, tcx) -> String`.
- Analysis: collects public ADTs/type aliases/functions/consts/statics and foreign decls, groups candidates by symbol and structural type with `TypeComparator`, requires duplicate consts to have equal body snippets, and treats unnamed C2Rust ADTs through disjoint sets.
- Resolution: chooses representatives by def path, config hints, CMake source/link priority, or `choose_arbitrary`; can ignore return or param type, and can cast call args when param types differ.
- Transformations: removes resolved duplicate items/impls/foreign items, rewrites paths to reps, inserts module-local `use crate::<def_path>;`, and uses full `crate::...` paths for unnamed types.
- Important config: `cmake_reply_index`, `build_dir`, `source_dir`, `function/static/type_hints`, `ignore_return_type`, `ignore_param_type`.

## Unsafe

- Goal: remove unnecessary unsafe/public/extern surface and optionally dead code while preserving C-exposed ABI.
- File: `crat/crates/passes/src/unsafe_resolver/mod.rs`; entry `unsafe_resolver::resolve_unsafe(&Config, tcx) -> String`.
- Analysis: builds a use graph from functions, impls, traits, uses, constructors, paths, trait-method THIR calls, local bindings, extern C fn pointer uses, mains, `export_name`, C-exposed names, and `#[used]`.
- Analysis: `find_unsafe_fns` uses `utils::unsafety::check_unsafety`; calls to unsafe local fns propagate through SCCs, and non-call unsafe ops keep a function unsafe.
- Transformations: removes unused items/uses/foreign/assoc items; drops empty mods/extern blocks; converts public to `pub(crate)` unless exposed/main; removes `extern "C"` and `unsafe` where safe; wildcards unused params; strips `#[no_mangle]` if configured.
- Important config: `remove_unused`, `remove_no_mangle`, `remove_extern_c`, `replace_pub`, `c_exposed_fns`.

## Unexpand

- Goal: reverse selected macro/derive expansions into idiomatic source.
- File: `crat/crates/passes/src/unexpander.rs`; entry `unexpander::unexpand(Config { use_print }, tcx) -> String`.
- Analysis: previsitor detects derived trait impls, bytemuck impls, bitfield accessor info, intrinsic/transmute use, thread-local expansions, and slice cursor impls.
- Transformations: restores `#[derive(...)]`, bytemuck derives, `#[bitfield(...)]`, `thread_local!`, `panic!`, `unreachable!`, `format!`, `write!`, optional `print!`/`eprint!`, and `core::mem::offset_of!`.
- Transformations: coalesces slice cursor impls into `impl_readable_index!` and `impl_mutable_index!`; prunes crate features no longer used.
- Important APIs/types: `MutVisitor`, feature attr filtering, `pprust`, `Config.use_print`.

## Split

- Goal: split a single expanded Rust file back into a module tree.
- File: `crat/crates/passes/src/splitter.rs`; entry `splitter::split(dir, lib_name)`.
- Operations: parses root with `utils::ast::parse_crate`, recursively writes inline modules into `<name>.rs` or `mod.rs`, keeps inner attrs, and replaces inline modules with `pub mod name;`.
- Important convention: this pass mutates files under the crate directory and does not return code to the entrypoint.

## Bin

- Goal: generate Cargo bin targets for translated `main` functions.
- File: `crat/crates/passes/src/bin_file_adder.rs`; entry `add_bin_files(dir, &Config, tcx)`.
- Analysis: HIR visitor finds functions named `main`, skipping configured def-path prefixes.
- Transformations: writes wrapper files named from def paths and appends `[[bin]]` tables to `Cargo.toml` with `toml_edit`; uses configured `name` only when exactly one bin is found.
- Important config: `ignores`, optional `name`.

## Check

- Goal: compile/type-check the current transformed crate at this point in the pipeline.
- Entrypoint: `Pass::Check` calls `run_compiler_on_path(&file, utils::type_check)`.
- Operation: `utils::type_check` itself is intentionally empty; all useful work is rustc parsing, lowering, type checking, and diagnostics through the compiler driver.

## Format

- Goal: apply Crat's AST formatting/writeback pass.
- File: `crat/crates/passes/src/formatter.rs`; entry `formatter::format(tcx)`.
- Operation: calls `utils::ast::transform_ast(|_| true, tcx).apply()` so the compiler-expanded AST is printed/applied without a separate code string.

## Interface

- Goal: keep C ABI wrappers around functions whose internal signatures were made more Rust-like.
- File: `crat/crates/passes/src/interface_fixer.rs`; entry `interface_fixer::fix_interfaces(&Config, tcx) -> String`.
- Analysis: scans C-exposed functions for params typed as `&[T]`, `&mut [T]`, `SliceCursor<T>`, or `SliceCursorMut<T>`.
- Transformations: renames original function to `<name>_internal`, creates a raw-pointer wrapper with the exposed name, and calls internal code with null-safe empty slices or `std::slice::from_raw_parts(_mut)(p, 1024)`.
- Transformations: rewrites Rust callers, uses, and path expressions to the internal name.
- Important config: `c_exposed_fns`.

## Libc

- Goal: replace common libc APIs with Rust std code and small `c_lib` helpers.
- Files: `crat/crates/passes/src/libc_replacer/{mod.rs,errno.rs,mem_utils.rs,str_utils.rs,strto.rs}`; entry `libc_replacer::replace_libc(tcx) -> TransformationResult`.
- Transformations: rewrites char/math/process APIs (`tolower`, `toupper`, `exp`, `fabs`, `floor`, `fmod`, `pow`, `sqrt`, `abort`, `__ctype_b_loc` checks), string/parse APIs (`strlen`, `strncpy`, `strcspn`, `strto*`, `atoi`, `atof`), and `memcpy`/`memset` on arrays/slices.
- Analysis: `errno::find_errno_calls` maps `__errno_location` comparisons to dominating foreign calls using HIR/THIR/MIR and dominators, then replaces errno checks for `pow` and `strto*`.
- Output: returns code plus `bytemuck` and `num_traits` flags; inserts only used `LibItem` helpers into an existing or new `mod c_lib`.
- Important helpers/types: `TransformationResult`, `LibItem`, `strto`, `mem_utils`, `bytemuck::cast_slice`.

## OutParam

- Goal: transform pointer output parameters into Rust return values, `Option`, or `Result` forms.
- Files: `crat/crates/outparam_replacer/src/{lib.rs,ai/analysis.rs,ai/domains.rs,ai/pre_analysis.rs,ai/semantics.rs,transform.rs}`; entry `outparam_replacer::transform::transform(tcx, &Config, verbose) -> String`.
- Analysis: reads JSON analysis or runs `ai::analysis::analyze`; uses Andersen points-to, alias preanalysis, call graph SCC summaries, and MIR abstract interpretation to detect must/may output params, complete writes, write-for-return patterns, removable checks, and success/failure returns.
- Transformations: removes output params from signatures and calls, returns must params as values, may params as `Option<T>` or `Result<T, OrigRet>`, creates temp value/ref/flag locals, rewrites pointer writes and return expressions, and adjusts call sites to bind returned values.
- Simplification: can remove unused temp declarations, direct-return assignments, dead flag sets, and some pointer derefs.
- Important types: `FnAnalysisRes`, `OutputParam`, `CompleteWrite`, `ReturnValues`, transform `Param`, `Func`, `ReturnTyItem`, `SuccValue`, `ParamIdx`, `RetIdx`.

## Lock

- Goal: placeholder pass.
- Entrypoint: `Pass::Lock => todo!()` in `crat/src/bin/crat.rs`.
- Convention: treat any request involving this pass as new implementation work and first clarify the expected lock transformation semantics.

## Union

- Goal: identify C tagged-union patterns and transform them into enums with tag/set/get methods.
- Files: `crat/crates/union_replacer/src/{tag_analysis.rs,must_analysis/*,ty_finder.rs,util/*}`; entry `union_replacer::tag_analysis::analyze(&Config, verbose, tcx) -> Statistics`.
- Analysis: finds structs with integral/bitfield tag candidates and anonymous `C2RustUnnamed*` union fields; builds type paths; runs Andersen may-points-to and a custom must-analysis over MIR to associate union fields with tag values.
- Analysis: matches HIR `match`/`if` guards and MIR switch/basic-block info to collect access tags, object tags, field values, aggregates, and local maps.
- Transformations: uses `AstSuggestions` and `TransformVisitor` to remove tag fields, replace union definitions with enums, add `new`, `get_*`, `deref_*_mut`, and struct `set_tag` methods, rewrite matches/ifs/aggregates/accesses, then applies edits via `utils::ast::transform_ast(...).apply()`.
- Important types: `Config { points_to_file, targets }`, `Statistics`, `UnionUse`, `VariantTags`, `TaggedUnion`, `TaggedStruct`, `Tag`, `AccElem`, `Nums`.

## Punning

- Goal: rewrite safe-enough type-punning unions into raw-byte structs with generated typed accessors.
- Files: `crat/crates/union_replacer/src/punning/{transform.rs,analysis.rs,raw_struct.rs,bytemuck.rs,reverse_cfg.rs,callgraph.rs,ty_visit.rs}`; entry `union_replacer::punning::replace_unions(tcx, verbose, &Config) -> TransformationResult`.
- Analysis: collects local non-foreign unions and related parent/field types; classifies union field types as `Pod`, `AnyBitPattern`, `NoUninit`, or `Other`; runs Andersen-backed union-use analysis and call-context expansion, then reverse-CFG reaching-write analysis to detect actual punning.
- Safety: skips nested unions and single-syntax-field unions; allows reads/writes only when field classes support them; uses `c_exposed_fns` in Andersen config.
- Transformations: derives needed bytemuck traits on field structs, rewrites target union definitions to `#[repr(C, align(N))] struct U { raw: [u8; size] }`, adds `new_*`, `get_*`, `get_*_ref`, `get_*_mut`, and `set_*` methods, and rewrites field expressions to method calls.
- Output: returns code, `needs_bytemuck`, and union stats; entrypoint ensures `bytemuck` with `derive` and `min_const_generics` when needed.
- Important types: `UnionAccessField`, `UnionMemoryInstance`, `UnionUseResult`, `UnionFieldClassification`, `FieldTypeClass`, `BytemuckDerivePlan`, `ReverseCfgResult`.

## Enum

- Goal: replace C2Rust integer type aliases plus typed constants that model C enums with Rust fieldless enums.
- File: `crat/crates/passes/src/enum_replacer/mod.rs`; tests in `crat/crates/passes/src/enum_replacer/tests.rs`; entry `enum_replacer::replace_enums(tcx) -> String`.
- Analysis: `analyze_enums` finds integer alias candidates and const variants, evaluates and sorts discriminants, models enum flow through aliases, locals, statics, function/foreign signatures, fields, pointers/refs, arrays/slices/tuples, returns, calls, method calls, struct literals, derefs, and indexing.
- Safety: only transforms aliases with typed const variants and no reject reasons; rejects duplicate or unevaluable discriminants, integer literals/arithmetic/casts/unknown expressions assigned to enum contexts, wrong enum values, wrong enum args/returns, and compound assignments.
- Transformations: replaces the alias with `#[repr(<int>)] #[derive(Clone, Copy, PartialEq, Eq, PartialOrd, Ord)] enum`, preserves alias attrs/visibility, removes original variant consts, inserts same-visibility `use Enum::Variant` reexports, rewrites `as Enum` casts to `as <repr>`, inserts enum-to-integer casts at recorded integer-use sites, and adds `coverage_attribute` when replacements occur.
- Interactions: generated variant reexports rely on unsafe resolver keeping used enum variant imports across modules; pointer raw-bridge defaults can materialize local fieldless enums by selecting the zero-discriminant variant.
- Important types: `EnumAnalysis`, `EnumInfo`, `IntegerRepr`, `VariantInfo`, `DiscriminantValue`, `RejectReasonKind`, `CastSite`, `CastSiteKind`, `HirEnumTy`, `EnumTransformPlan`.

## Io

- Goal: rewrite C `FILE*` and stdio APIs to Rust stream types and helpers.
- Files: `crat/crates/io_replacer/src/{lib.rs,file_analysis.rs,error_analysis.rs,transformation/*}` plus per-API modules such as `fopen.rs`, `fscanf.rs`, `fprintf.rs`, `fread.rs`, `fwrite.rs`, `fseek.rs`.
- File analysis: tracks MIR locations `Stdin`, `Stdout`, `Stderr`, `Extern`, `Var`, and `Field`; computes permissions and origins from `utils::file::api_list`; marks unsupported externs, error handling, permission conflicts, close cycles, and std stream assignments.
- Error analysis: walks HIR/MIR backward from `ferror`/`feof` handling, tracking `Indicator::{Error,Eof}` through calls, returns, params, and no-source locations.
- Transformations: rewrites stream types, generic bounds, error indicator params/returns/local vars, unsupported std stream fallbacks, union fields to `ManuallyDrop`, and many stdio calls; adds only used `c_lib` helpers.
- Output: returns code, dependency flags (`tempfile`, `bytemuck`, `num-traits`), unsupported reasons, timing, and analysis stats.

## Pointer

- Goal: replace raw pointers with safer Rust references, slices, boxes, options, or slice cursors where whole-program analysis permits.
- Files: `crat/crates/pointer_replacer/src/{lib.rs,rewriter/*,collector.rs,decision.rs,analyses/*}`; entry `pointer_replacer::replace_local_borrows(&Config, tcx) -> (String, bool)`.
- Analysis: runs Andersen and param-alias checks; collects a `RustProgram`; computes mutability/fatness qualifiers, output params, ownership, source-variable grouping, borrow promotion, and offset sign.
- Decision: `DecisionMaker` chooses `PtrKind::{OptRef, OptBox, Raw, OptBoxedSlice, Slice, SliceCursor}`. Raw is retained for void/file/local-struct mutable aliases, locked function pointer signatures, and unsupported ownership cases.
- Transformations: rewrites signatures, locals, assignments, calls, returns, derefs, null checks, offsets, and free calls; inserts `SliceCursor` helpers when needed; downgrades unsupported boxes to raw.
- Output: returns code and a `bytemuck` flag.
- Diagnostics: set `CRAT_POINTER_DECISION_DIAGNOSTICS=summary`, `raw`, or `full` to emit pointer decision summaries/events to stderr without changing transformed output.

## Static

- Goal: replace many `static mut` globals with immutable statics, `Cell`, or `RefCell`.
- Files: `crat/crates/static_replacer/src/{lib.rs,transformation.rs}`; entry `static_replacer::replace_static(tcx) -> String`.
- Analysis: records static uses and mutability context with `find_context`. Never-mutated statics become immutable; mutated but not borrowed/method-called statics become thread-local `Cell<T>`; the rest become thread-local `RefCell<T>`.
- Transformations: emits `thread_local!`, rewrites reads to `.get()`, writes to `.set()`, array-cell access to `as_array_of_cells`, and refcell uses to `with_borrow`/`with_borrow_mut` blocks with temporaries.
- Feature attrs: adds `never_type`, `thread_local_internals`, and `as_array_of_cells` when required.

## Simpl

- Goal: lightweight cleanup after other transforms.
- Files: `crat/crates/passes/src/simplifier.rs`, `simplifier/unused_assignments.rs`; entry `simplifier::simplify(tcx) -> String`.
- Analysis: finds unused assignments with MIR `MaybeLiveLocals`, mapping HIR/THIR/MIR back to AST statements and locals.
- Transformations: removes side-effect-free assignment statements, converts unused initializing lets into declarations with explicit types, simplifies literal integer casts, redundant same-size cast chains, parens, paths, FILE/libc aliases, zero erasing ops, and unsigned zero comparisons.
- Important APIs/types: `rustc_mir_dataflow`, HIR/THIR/MIR maps, `MaybeLiveLocals`, AST `MutVisitor`.

## Andersen Analysis

- Goal: whole-program may-points-to analysis used by pointer, outparam, union, and punning logic.
- Files: `crat/crates/points_to/src/{andersen.rs,alloc_finder.rs}`; entry `points_to::andersen::run_analysis(&Config, tcx) -> Solutions`.
- Preanalysis: builds type shapes, allocation function set, bodies, call graph, direct/indirect call args, globals, vars, index metadata, field graph, union offsets, exposed C fn args, and typed location ranges.
- Analysis: transfers MIR address/ref/raw-ptr/copy/move/cast/aggregate/call operations into Andersen constraints (`l=&r`, `l=r`, `l=*r`, `*l=r`, indirect calls) and solves with `ChunkedBitSet` points-to sets.
- Postanalysis: adds deref edges from solutions, computes address-taken prefixes, writes and bitfield writes per loc, indirect calls, SCCs, and function write sets.
- Important types: `Config { use_optimized_mir, c_exposed_fns }`, `Loc`, `PrefixedLoc`, `ProjectedLoc`, `Graph`, `LocGraph`, `LocEdges`, `AnalysisResult`, `Solutions`.

## OutParam Analysis

- Goal: standalone analysis mode for output-parameter detection, usually feeding the `OutParam` transform.
- Files: same as `OutParam`, especially `crat/crates/outparam_replacer/src/ai/{analysis.rs,domains.rs,pre_analysis.rs,semantics.rs}`.
- Entrypoint: `Analysis::OutParam` calls `outparam_replacer::ai::analysis::analyze(&config.outparam, verbose, tcx)`, prints counts of functions/must/may params, and writes JSON with `write_analysis_result`.
- Analysis: combines Andersen points-to, alias detection, call graph SCC summaries, return value ranges, loop state caps/widening, and MIR abstract interpretation over `AbsState`, `AbsValue`, `AbsPlace`, `AbsPtr`, `AbsOption`, and path sets.
- Important config: `max_loop_head_states`, `check_global_alias`, `check_param_alias`, `no_widening`, `points_to_file`, debug timing/printing options.
