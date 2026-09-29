# Amendment 12 Detailed Plan: Temporary Function Declarations

## 1. Purpose and authority

This plan and the companion [test plan](amendment-12-test-plan.md) are guidance
for a fresh implementation session. Read the current
[prototype description](prototype-desc.md), then the current code and tests.
Tests and implementation establish existing behavior; this plan specifies the
intended revision where they differ. Keep planning labels such as
“Amendment 12” out of PROCTOR or Crat code, comments, tests, fixtures,
diagnostics, configuration, and generated files. Keep `proctor/` self-contained.
Do not change earlier detailed plans or historical sections in
[prototype-plan.md](prototype-plan.md).

The local stage currently calls `crat-tool make-initial` to remove all
source-defined free functions from the normalized library while retaining
`use` items. It immediately runs `cargo build --lib`. A retained import of an
omitted function therefore fails before any SCC can be transformed. The
[`B01` run](../proctor/runs/B01_synthetic_001_helloworld_with_header-20260929T113043-4a86b2/stages/02-local_transformation/out/artifacts/local-transformation.log)
shows one E0432 import of `helloworld`; the
[`P01` run](../proctor/runs/P01_sphincs_plus_005_sphincs_PQCgenKAT_sign_blake_128f_simple-20260929T113152-42882e/stages/02-local_transformation/out/artifacts/local-transformation.log)
shows 66 E0432 imports. Those logs establish the initial-build defect, not
that the later SCCs or final project will succeed after it is fixed.

## 2. Fixed decisions and scope

- Keep the original retained imports and other non-function context. Give
  each source-defined free function except `main` a temporary declaration at
  its original inline-module path and source position. Its complete syntax is the
  original visibility followed by `fn` and the original identifier, an empty
  parameter list, and an empty body: `[vis] fn name() {}`. An empty visibility
  means a private function. Preserve raw identifiers and restricted
  visibility through the source AST rather than string-derived names.
- A temporary declaration has no original signature, lifetimes, generics,
  return type, safety qualifier, ABI, export attribute, or other function
  attribute. It is a name-resolution aid, not an executable implementation or
  a target-type skeleton. Crat continues to take the accepted function's
  visibility and metadata from the complete normalized analysis source, and
  its transformed signature/body from the selected validated skeleton and
  returned transformation.
- Keep the complete analysis project, dual skeleton views, leaf-first SCC
  schedule, prompt/validator behavior, rule fallback, transactional Cargo
  builds, observation extraction, report generation, and final full build.
  Every SCC still replaces its members together and is accepted only after
  its existing `cargo build --lib` transaction succeeds.
- Do not add a placeholder-identity check or a new finalization-completeness
  check. Requested path lookup is intrinsic to replacement; it must not be
  expanded into a comparison against the temporary body or signature. The
  stage's existing accepted correspondence is still needed for typed
  observations. Remove or adapt old function-set guards whose assumption that
  the partial project contains only accepted functions is no longer true.
- Do not change the stage envelope, `ReplacementRequest`, observation
  metadata, rule documents, manifests, LLM prompts, or API-wrapper behavior.
  Do not run a test package within this stage.

## 3. Current implementation and ownership

`proctor/stages/local-transformation/stage.py` prepares and normalizes a
complete `work/analysis` project, copies it to `work/current`, calls
`make-initial`, and builds `work/current` as a library before scheduling SCCs.
For each SCC, it calls `add-functions`, validates sidecars, installs and builds
the candidate transactionally, then extracts observations from a separate
scratch source only after acceptance. It calls `finalize-project` and builds
the ordinary project after the schedule finishes. The stage owns these
operations and should not gain Rust syntax manipulation.

`proctor/stages/crat/crates/tools/src/item_replacer.rs` owns the required
changes. `make_initial_source` currently calls `retain_non_function_items`,
which deletes each body-defined free function recursively. The same helper
also strips functions from copies used solely to compare non-function context;
keep that comparison role distinct from the new initial projection.
`add_functions_with_observations` currently rejects a requested path already
present in the partial source and appends the accepted implementation to both
candidate and observation source. Its current partial-function count must be
adjusted because all pending names will now be present. It must still add
separately named original source copies and old-signature call stubs to the
scratch observation source. `finalize_additive_source` restores excluded
`main` functions and currently checks a path count that no longer indicates
acceptance once pending declarations are present. Keep `main` absent from the
initial project and retain its existing finalization insertion.

`proctor/stages/crat/src/bin/crat-tool.rs` remains a thin file-I/O wrapper
around `make-initial`, `add-functions`, and `finalize-project`. The Python
`protocol.py` and `tooling.py` command shapes do not need to change.

## 4. Initial project and SCC replacement

1. Generate the initial source from the complete normalized analysis source.
   Keep every supported non-function item, crate attribute, and inline-module
   structure as before. Replace each non-`main` body-defined free function
   item in place with the minimal temporary declaration. Continue omitting
   source-defined `main`. Use the original item visibility
   and identifier exactly; discard its original attributes and all other
   function syntax. Leave foreign declarations, which have no body, as
   foreign declarations. Do not save an import-restoration map or remove
   `use` items.
2. Keep the initial `cargo build --lib` as the stage's first target build.
   It must now resolve ordinary direct imports, aliases, and reexports of
   pending non-`main` function names that the original visibility permits.
   A failure for another reason retains the current terminal initial-build
   behavior.
3. For each requested SCC member, use the full crate-relative path to select
   its item in the partial target, then replace that item at its existing
   position with the canonical transformed implementation. Perform the same
   replacement in the separate observation source so that source copies can
   be added under their generated names without a duplicate original name.
   Keep the existing composition and canonical restoration from the complete
   analysis source; no header, body, or attribute is copied from the temporary
   declaration into the accepted function.
4. Keep original source copies, optional old-signature observation call stubs,
   label handling, digest-bound metadata, and correspondence paths as today.
   These scratch items still do not enter the candidate or published project.
   Retain the existing source-copy call rewriting and macro-token rejection.
5. The old partial-function guard equates all partial functions with
   `accepted_correspondence`. That equation becomes false immediately after
   initialization. Remove that obsolete equality while keeping request
   validation, full-path selection, accepted-correspondence validation and
   original-versus-accepted signature comparison required for source stubs.
   No separate placeholder shape or completion validation replaces it.
   The finalizer's old count of non-`main` paths becomes vacuous because the
   initial project already contains all those paths; remove that obsolete
   count check. Keep its other target-kind, API, context, and `main`
   restoration checks unchanged.
6. Keep the Python candidate transaction unchanged: build the new candidate,
   restore the previous library source on build failure, and promote
   correspondence, reports, and observations only after a successful build.
   A failed attempt leaves its prior pending declarations in place, and
   applied-view fallback retries the same SCC through the baseline view.

## 5. Completion, limitations, and documentation

The stage's schedule must continue to process every non-`main` function
record from the complete analysis source, or stop on failure. After the last
accepted SCC, the accepted functions occupy their original paths; no separate
pass removes remaining temporary declarations. Do not add a final sweep or
placeholder-count assertion. A direct call to `finalize-project` with an
incomplete partial source is outside the stage's completion guarantee.
The existing finalization step inserts excluded `main` functions, including
the fixed two-argument `main_0` sibling boundary, exactly as before.

This change assumes retained non-function items use pending functions only
for name resolution through imports or reexports. A static, const, or other
retained item requiring a pending function's original type or value can still
fail the initial build or a later build. The minimal declaration also cannot
type-check calls to pending functions if a direct-call dependency escaped the
current SCC schedule. Existing Cargo-build and repair boundaries remain
authoritative; do not claim this plan repairs those cases. In particular,
omitting a return type from a placeholder does not cause the initial build to
detect all return-position `impl Trait` cases. There is no new explicit RPIT
check or categorical new RPIT exclusion in this revision.
An import or reexport of excluded `main` remains unsupported by the initial
library build; neither cited run shows that case.

After implementing and testing, update [prototype-desc.md](prototype-desc.md)
using the `$prototype-desc` skill: describe the pending declarations,
replacement, accepted-only observations, and the supportedness limit above.
Update stale current-facing PROCTOR/Crat skill references if their descriptions
of initial assembly or additive insertion disagree. Leave historical plans,
their test plans, and their sections in `prototype-plan.md` unchanged.

## 6. Implementation sequence and verification

1. Update the focused in-memory Crat tests and the Python fake-tool/stage
   tests specified in [amendment-12-test-plan.md](amendment-12-test-plan.md).
   Assert concrete initial source, per-SCC replacement in both sources,
   rollback, observation timing, final source, and terminal failures. Include
   the two reported import patterns as regression shapes.
2. Change only the required initial projection, path replacement, obsolete
   function-count guards, and any directly affected test expectations. Keep
   Crat's in-memory tests beside its implementation. They must not invoke the
   `crat-tool` CLI, mutate the filesystem, or use a Crat-root `tests/` tree.
3. From `proctor/stages/crat`, run
   `cargo test -p tools item_replacer::tests`,
   `cargo test -p tools observation::tests`, `cargo test -p tools`,
   `cargo fmt`, and `cargo clippy --workspace --all-targets`. Resolve Clippy
   warnings; use targeted allowances only for the permitted lint cases.
   From `proctor`, run `uv run pytest tests/test_local_transformation.py`,
   then the normal Python suite and static checks (`uv run pytest`,
   `uv run ruff check .`, `uv run ruff format --check .`,
   `uv run mypy proctor`). Run real stage replays for the two reported cases
   only when their local toolchain and input artifacts are available; success
   requires progressing beyond the formerly failing initial import build,
   not an unsupported claim of full translation success.

Completion requires an initial library that resolves retained function
imports, exactly one accepted implementation at each transformed path after
its SCC, unchanged SCC validation/build and observation semantics, and a final
ordinary Cargo build before publication. The output still has no behavioral
test or C ABI compatibility guarantee.
