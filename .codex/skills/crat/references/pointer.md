# Shared pointer analysis and rewriting

## Shared analysis

- Start with `crates/pointer_replacer/src/lib.rs` for exports and `rewriter/mod.rs` for the shared analysis pipeline.
- Use `rewriter/decision.rs` for `PtrKind`, decision stages, and raw fallbacks; use `rewriter/collector.rs` for signature/binding decision collection.
- Expect `PtrKind::{Ref, OptRef, Box, OptBox, Raw, BoxedSlice, OptBoxedSlice, Slice, SliceCursor}`. Boolean payloads encode mutability where present.
- Build a `RustProgram`, run Andersen points-to and parameter-alias analysis, and compute mutability, fatness, output parameters, ownership, borrow promotion/lifetime flows, source-variable grouping, offset signs, nullity, struct-copy facts, and function-pointer groups.
- Preserve `c_exposed_fns` in `pointer_replacer::Config`; it affects points-to and boundary decisions.

## Ordinary `crat` pointer pass

- Run pointer stages from `Pass::Pointer` in this order:
  1. `rewrite_struct_arrays`
  2. `rewrite_epoch_split`
  3. `rewrite_array_local_provenance`
  4. `replace_local_borrows`
- Recompile between changed preprocessing stages because each returns rewritten source plus a `changed` flag.
- Use default `PointerDecisionOptions`, including offset-sign analysis. Select `SliceCursor` or `SliceCursorMut` when potentially negative offsets require position tracking.
- Expect `replace_local_borrows(&Config, tcx) -> (String, BytemuckDependency)`. Handle `None`, `Runtime`, and `Derive` in the CLI.
- Rewrite signatures, locals, fields, assignments, calls, returns, dereferences, null checks, offsets, allocations, and frees; allow rewrite-time safety checks to retain or demote unsupported cases to raw pointers.
- Append the `slice_cursor` module only when the final rewrite uses it.
- Set `CRAT_POINTER_DECISION_DIAGNOSTICS=summary`, `raw`, or `full` for deterministic decision diagnostics without changing generated source.

## `crat-tool` skeleton decisions

- Call `initial_pointer_decisions(&Config::default(), PointerDecisionOptions { assume_nonnegative_offsets: true }, tcx)` from skeleton generation.
- Reuse the ordinary analysis and decision machinery; do not duplicate pointer inference in `crates/tools`.
- Treat the returned `InitialPointerDecisions { signatures, bindings }` as fixed target types for function signatures and local bindings.
- Do not run the pointer preprocessing stages, expression rewriter, rewrite-time demotion, dependency insertion, or slice-cursor module generation while building skeletons.
- Assume upstream non-local transformation removed actual negative offsets. With the tools-only option, offset-sign facts are empty and skeleton decisions do not select slice cursors for negative-offset handling.
- Keep the two-argument `main_0` `argv: &mut [&mut [i8]]` override in skeleton generation; it takes precedence over the ordinary pointer decision for that parameter.

## Cross-consumer invariants

- Keep `PointerDecisionOptions::default()` behavior unchanged for the ordinary pointer pass. Only `crat-tool` should opt into `assume_nonnegative_offsets`.
- Keep initial decisions separate from final rewrite output: `crat-tool` intentionally fixes pre-demotion analysis results, while `crat` may adjust representations when rewriting cannot support them.
- When changing shared analysis or type rendering, test both consumers. A tools-only presentation change must not silently alter ordinary pointer-rewriter output.

## Focused verification

- Run `cargo test -p pointer_replacer` for shared analysis, decision, or ordinary rewrite changes.
- Run `cargo test -p tools skeleton::tests` for initial-decision consumption, type rendering, lifetime, or tools-only option changes.
- Run both suites when changing `PtrKind`, `InitialPointerDecisions`, `PointerDecisionOptions`, `SigDecision`, or `SigDecisions`.
