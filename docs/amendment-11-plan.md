# Amendment 11 Detailed Plan: Wrapper-Free Local Finalization

## 1. Purpose and authority

This plan and the concrete [test plan](amendment-11-test-plan.md) specify the
revision for a fresh implementation session. Read the current
[prototype description](prototype-desc.md), then the current additive code and
tests. Tests and implementation establish existing behavior; this plan defines
the intended change where they differ. Keep planning labels such as
“Amendment 11” out of PROCTOR and Crat implementation surfaces. Keep
`proctor/` self-contained. Do not change earlier detailed plans or their
historical sections in [prototype-plan.md](prototype-plan.md).

The local prototype currently builds one accepted function SCC at a time,
then `finalize-project` restores excluded `main` functions and creates
permanent library API wrappers whenever source and target signatures differ.
The general `crat-tool replace` operation predates this additive flow and
creates its own compatibility wrappers, but the active local stage no longer
calls it. This revision removes **all wrapper generation from local
finalization** and removes the unused general `replace` operation.

## 2. Fixed decisions and scope

- The final Rust function remains at its original crate-relative path with
  its accepted transformed signature, body, visibility, calling convention,
  and export attributes. Do not compare source and target API signatures to
  decide on a wrapper or conversion. Do not generate a wrapper for references,
  boxes, slices, boxed slices, their optional forms, or any other type.
- The local stage does **not** promise to preserve the original C ABI after
  this revision. An exported transformed slice reference or boxed slice may
  have a different ABI from the original C pointer. The stage's Cargo build
  proves only that the final Rust project builds; it is not an ABI check or a
  semantic test. This limitation is deliberate and must be described in the
  current-facing prototype documentation and test expectations.
- Do not change, invoke, or add the ordinary Crat `interface` pass in this
  revision. The checked-in local pipeline does not select it after local
  transformation. A separate future interface step may address C ABI
  compatibility; this plan neither implements nor assumes one.
- Remove the general `crat-tool replace` CLI operation, its exported Rust
  tools entry points, the corresponding Python protocol/tooling method, and
  tests dedicated to that operation. Do not remove Python's `str.replace` or
  `os.replace`, Crat's `add-functions`, or any shared helper still used by the
  active additive path.
- Keep `finalize-project` as the completion operation. It still validates the
  target kind, API list and matching source functions, complete accepted
  function set, and preserved non-function context; restores excluded `main`
  functions, including the existing fixed two-argument `main_0` boundary;
  and stages final source and `proctor.toml` for the final full build. The
  manifest's `wrappers` array stays empty. A nonempty input wrapper list
  remains unsupported.
- Preserve SCC scheduling, skeleton views, prompting, structural validation,
  additive insertion, source-copy observation extraction, rule behavior,
  statement-pair reporting, and transactional builds. No stage envelope,
  prompt, rule, or observation document shape changes are requested.

## 3. Current code and deletion boundaries

The active path in `proctor/stages/local-transformation/stage.py` prepares an
immutable complete analysis project, installs a function-free partial library,
adds each accepted SCC through `crat-tool add-functions`, validates/builds
each candidate, extracts observations from separate scratch source, and calls
`crat-tool finalize-project`. Final source and manifest are installed as a
paired attempt and checked with an ordinary `cargo build`. It never calls the
general `replace` operation.

In `proctor/stages/crat/crates/tools/src/item_replacer.rs`, remove
`replace_items` and `replace_items_with_observations` and the call redirect,
temporary wrapper, observation assembly, and item insertion helpers used only
by them. Remove the finalizer's changed-signature branch, wrapper allocation,
export transfer, and `FinalWrapper` result data. Delete wrapper-only type
classification and conversion helpers once their remaining uses have been
checked. Simplify `compose_implementation`'s wrapper flag if only the additive
`false` case remains. Keep the request model and validators,
`add_functions_with_observations`, `make_initial_source`,
`finalize_additive_source`, `parse_body` for additive source stubs, the
compiler-resolved source-copy call rewriter, correspondence validation,
statement-pair handling, and every helper still referenced by those paths.
For example, `signature_types` is still used to decide whether an accepted
callee needs an old-signature **observation-only stub**; that comparison is
unrelated to API wrapper generation and must remain.

In `proctor/stages/crat/src/bin/crat-tool.rs`, remove the `Replace` command
variant and handler. Keep the `AddFunctions`, `MakeInitial`,
`FinalizeProject`, and `ExtractObservations` commands and their shared request,
path-validation, serialization, and publication machinery. Finalization still
accepts separate candidate source and manifest output paths. Since no
wrapper entries are appended, write the validated input manifest through
unchanged, preserving its other fields and formatting. Keep the empty-wrapper
input check. Do not delete shared helpers merely because their names contain
“replace”.

In `proctor/stages/local-transformation/protocol.py`, remove only
`replace_command`. In `tooling.py`, remove the unused `CratTools.replace`
method and its import. Keep `replacement_request`, `CratTools.add_functions`,
its output validation, candidate transaction, finalization tooling, and the
two Crat binary builds: the local stage still uses `crat` for preparation and
`crat-tool` for additive work. The active stage should retain the same
sequence and failure envelope, with no new postprocessing pass.

General-replace-specific Crat tests are concentrated in
`crates/tools/src/item_replacer/tests.rs`, with two direct
`replace_items` callers in `crates/tools/src/skeleton/tests.rs`. Remove or
rewrite only tests that exercise the deleted operation; retain additive,
finalization, skeleton, correspondence, and observation tests that still
exercise supported behavior. In `proctor/tests/test_local_transformation.py`,
remove the `replace_command` and `CratTools.replace` tests and old fake-tool
branches for that command; retain stage tests for `add-functions` and final
publication. The version-2 additive observation metadata and its null
`wrapper_path` fields remain the current protocol. Remove the legacy
version-1 metadata emitter only if it becomes dead after deleting `replace`;
keep version-1 metadata reading for prior artifacts. Do not change version-2
output or make a broad metadata refactor.

## 4. Finalization algorithm

1. Parse the immutable analysis source and the accepted partial source using
   the current compiler-resolved path mapping. Keep existing target-kind and
   API-list checks, including invalid `main`, unmatched API entries, and
   matching by Rust function name or explicit export name. Keep deduplication
   where several API entries identify the same function. Continue checking
   that the partial project contains exactly the expected accepted non-`main`
   functions and the same non-function context.
2. Leave every accepted function in place, including API functions. Do not
   inspect a source/target signature difference for compatibility purposes,
   allocate a name, clone a declaration, convert a parameter/return, strip an
   export attribute, change `extern`, or insert a wrapper. Keep original
   export metadata already carried by additive insertion on the transformed
   implementation. The finalizer must not reject a transformed function
   solely because its API type is a slice or boxed slice.
3. Restore excluded `main` items exactly as today: use the fixed forwarding
   sibling `main` for a supported two-argument `main_0`, require its sibling,
   and retain ordinary or zero-argument `main` bodies under their current
   rules. This remains necessary for executable and library projects.
4. Produce final source and a copy of the validated manifest with
   `wrappers = []`. Publish both candidate files together or neither. The
   Python stage validates target/API fields, installs the pair with rollback
   files, runs final ordinary `cargo build`, and publishes the project only
   after success. Finalization or build failure remains terminal; no SCC
   repair or partial output is exposed.

This removes source-versus-target API type conversion policy. The additive
path's type comparisons used to prepare observation-only stubs are retained,
because they support typed extraction and do not create deliverable wrappers.

## 5. Implementation sequence

1. Update focused tests from the [companion test plan](amendment-11-test-plan.md)
   to assert that transformed APIs keep their direct paths and export
   attributes, the final manifest stays empty, `main` restoration and final
   build still work, and general `replace` is no longer exposed. Include
   direct slice and boxed-slice API results so the deliberate ABI limitation
   is visible in acceptance tests.
2. Remove the Crat CLI/library general replacement path and its dedicated
   tests. Prune only helpers made unused by this deletion, checking shared
   additive references before each removal. Keep the active metadata and
   source-copy behavior stable.
3. Remove finalization's permanent wrapper construction and manifest append
   logic. Preserve API validation and main restoration. Remove the Python
   general-replace command/method and their tests; leave the active stage
   sequencing unchanged.
4. Update current-facing [prototype-desc.md](prototype-desc.md) using the
   `$prototype-desc` skill. Correct its replacement/compatibility section,
   command inventory, and supportedness claims to state that local output may
   break the C ABI. Update the workspace skill references
   `.codex/skills/crat/references/crat-tool.md` and
   `.codex/skills/proctor/references/contracts-and-stages.md`, whose current
   command and stage descriptions still advertise general replacement or
   wrapper correspondence. Update the wrapper-conversion supportedness
   subsection in [unsupported.md](unsupported.md), and any other
   current-facing PROCTOR/Crat documentation with the same stale claims. Leave
   all historical detailed plans and their `prototype-plan.md` sections
   unchanged.

## 6. Verification and completion criteria

From `proctor/stages/crat`, run focused tools tests for additive insertion,
finalization, skeletons, and observations, then `cargo test -p tools`,
`cargo fmt`, and `cargo clippy --workspace --all-targets`. Crat tests must
remain in source modules, use in-memory inputs, and neither call the
`crat-tool` CLI nor mutate the filesystem. From `proctor`, run
`uv run pytest tests/test_local_transformation.py`, the default `uv run pytest`
suite, `uv run ruff check .`, `uv run ruff format --check .`, and
`uv run mypy proctor`; validate `configs/c2rust_crat_local.toml` if its
integration is touched. Focused tests must check that deleted legacy command
surfaces are absent and shared active commands still work.

Completion means additive SCC acceptance and observations still work; final
functions are exported directly with accepted signatures; no local
compatibility wrapper or manifest relation is emitted; `replace` is absent
from Crat and Python public tooling; finalization still restores `main` and
passes the paired final build transaction; and current documentation states
the resulting C ABI limitation plainly.
