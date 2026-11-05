# Borrow/ownership pointer analysis and rewriting

## Entrypoint and integration

- Use `--pass pointer-bo` or `passes = ["pointer-bo"]`, with `c_exposed_fns` under `[pointer-bo]`. Top-level exposed-function names also propagate here. The original `pointer` pass and `crat-tool` skeletons continue to use `pointer_replacer`; see [pointer.md](pointer.md).
- Start with `crates/pointer_replacer_bo/src/lib.rs` for exports and `bo_pipeline.rs` for `rewrite_bo(input, c_exposed_fns) -> Result<BoReceipt, String>`. This entrypoint manages compiler sessions, source persistence, and required bytemuck dependency declaration; it does not return source text for the CLI to write.
- Follow `bo_rewriter/mod.rs` for analysis-to-decision orchestration, planning, emission, and verification. `analyses/borrow_ownership/` owns the coupled analysis and its borrow/mutability forks; other auxiliary analyses and utilities are reused from `pointer_replacer`. The public `rewriter` reexport is legacy; keep BO expression rewriting in its separate `bo_rewriter` modules.
- Use the pinned toolchain, native Z3 setup, and host-targeted `deps_crate` build described in [crat.md](crat.md#entrypoint-and-compiler-utilities). Follow current call paths and tests when module headers describe narrower or older behavior.

## Coupled inference

- Start with `analyses/borrow_ownership/construction.rs` for construction and `solve_bo_overlap_config_reporting`. `crate_slots.rs`, `slots.rs`, and `resolve.rs` connect compiler-resolved local pointer chains and global struct-field slots. Slot construction models up to three pointer levels; ownership emission separately uses precision two. Extending one does not automatically extend the other.
- Keep `SlotKind::{Raw, Ref, Owning}` separate from final Rust representations. `solver.rs` prefers references, then ownership, then raw pointers within the admissible models; that objective does not establish emission eligibility.
- Use `analyses/borrow_ownership/mod.rs`, `infer.rs`, and `ssa/` for crate-wide ownership constraints against shared function-signature variables and versioned local ownership. Preserve coherence and safety constraints; allocation and release selectors are retractable assumptions, not unconditional permission to construct an owner or insert a drop.
- Use `borrow_verify.rs`, `borrow_engine/`, and `borrow_fork.rs` for model-dependent validation to a fixpoint. Production combines native origin flows, precise parameter-overlap replay, selected copy lends, and return refinement. Keep constraint generation, borrow replay, and emitted copy-lend adaptations consistent.
- The active repair is guarded, with fallback to plain repair on decline. Plain repair excludes representative references monotonically. Preserve retirement, protected-entry, and reader obligations alongside alias-conflict checks; resolving an ordinary borrow conflict alone does not admit an owner.
- Use `source_events.rs`, `retirement.rs`, `retirement/`, and `licensing/` for source lifetime termination and ownership evidence. Source-event identities describe abstract objects and static sites, not dynamic allocation epochs or generated drops. Keep qualifier/export observations distinct from proofs: missing evidence remains missing, and null-use flags do not prove nonnullness.

## Decisions, emission, and recovery

- `bo_rewriter/mod.rs::finish_decide` combines the accepted model with construction, mutability, fatness, nullability, extent, lifetime, and boundary evidence. `decision/` owns the completed decision table, including references, optional forms, slices, cursors, ownership plans, and attributed raw fallbacks. Preserve compiler identity across parameters, named locals, fields, returns, and their uses.
- Preserve allocator provenance, retained C releases, and explicit waivers for generated scope-exit or overwrite drops in ownership plans. Borrow validation permits some raw-path overwrites of owning cells; the rewrite must use non-dropping stores there. Use each API's matcher in `analyses/borrow_ownership/boundary_table.rs` rather than replacing resolved recognition with surface-name tests.
- Capture the pristine expanded AST and AST/HIR mappings before analysis, through `ast_transform::capture_ast`. Rustc lowering consumes resolver state; each emission round must transform a clone of that capture rather than recapturing after HIR/MIR queries.
- Keep decisions complete before `plan/` builds and finalizes signature classes and seals call/return adaptations. `round_files` emits through `ast_transform::ast_emitted_files_from`; `ast_bridge.rs` installs changed item regions while retaining original bytes outside them. Comments inside reprinted bodies are not preserved.
- `additive.rs` preserves earlier deliveries as additional rewrite families are considered. Withdraw unsupported candidates at their subject or owner scope. Keep `effective_withheld_classes`, field transactions, certified return groups, terminal adaptations, and their receipts consistent when dependencies are held or reverted.
- `verify_and_revert` compiles candidates, subtracts original-input diagnostics, attributes new failures, and re-emits after subject or signature-class reversion. Reverting everything must restore exact input bytes and yields degradation. A successful pass return can also report no emitted tree; inspect `BoReceipt` before interpreting it as a delivered rewrite.
- Preserve the implemented verification assumptions: emitted candidates are checked with `panic=abort`, and parameter-overlap analysis assumes a closed, frozen whole-program call graph without unknown callers. Some slice/cursor adaptations still use reported fabricated lengths. Compiler acceptance does not prove every bound, behavioral equivalence, or general C ABI compatibility.

## Run controls and artifacts

- `--bo-receipt PATH` writes the rendered emission receipt; the CLI always prints its summary. `CRAT_BO_ARTIFACT_DIR` writes `subjects.tsv` with model kinds, decisions, delivery outcomes, and first hold reasons, plus revert and evidence reports. Artifact write failures are diagnostic and do not change acceptance.
- `--bo-query-timeout SECS` overrides `CRAT_BO_QUERY_TIMEOUT_MS`; the default is 600 seconds per Z3 query. Keep typed Unknown handling in `analyses/borrow_ownership/execution_guard.rs` distinct from an unsatisfiable result or an ordinary analysis decline.
- `CRAT_BO_ANALYSIS_BUDGET_S` and `CRAT_BO_ANALYSIS_BUDGET_GB` optionally bound the whole analysis at checkpoints. They do not interrupt an individual step. Exhaustion retains the input tree and returns a degraded receipt so later passes can continue.
- `CRAT_BO_CACHE_DIR` enables model caching; cached and freshly solved models rejoin the same decision path. Preserve source/toolchain/dependency fingerprints and update `analyses/borrow_ownership/code_fingerprint.rs::LEGACY_FILES` for new legacy analysis dependencies and their transitive dependencies; existing prefix assertions do not discover every omission. Models using timeout incumbents are not cached. Use `CRAT_BO_PHASE_LOG=on` for timing and memory diagnostics.

## Focused verification

- Keep tests within the skill's library and filesystem constraints. Inspect helpers before widening filters: existing cache, artifact, and end-to-end verification fixtures include tests that write temporary trees.
- Run `cargo test --bin crat` for pass selection, TOML routing, and timeout argument parsing.
- Run `cargo test -p pointer_replacer_bo --lib analyses::borrow_ownership::qualifier_facts` for qualifier capture/export. For query and solver changes, select `analyses::borrow_ownership::execution_guard::tests`, `analyses::borrow_ownership::solver::incumbent_tests`, or `analyses::borrow_ownership::solver::unknown_tests` with the same package and `--lib` options.
- Run `cargo test -p pointer_replacer_bo --lib bo_rewriter::decision::extent_proof_tests` for extent evidence, or `cargo test -p pointer_replacer_bo --lib bo_rewriter::plan::class_split::sealed_source_input_tests` for sealed input/dependency behavior. Add relevant in-memory coverage when changing ownership, borrow replay, or emission contracts.
