# Amendment 10 Detailed Plan: Additive Local Transformation

## 1. Purpose and authority

This is implementation guidance for a fresh session. Read the current
[prototype description](prototype-desc.md), this plan, and the concrete
[test plan](amendment-10-test-plan.md) before changing code. Current tests and
implementation establish existing behavior; this plan specifies the intended
change where they differ. Do not put “Amendment 10” or other planning labels in
PROCTOR or Crat code, tests, fixtures, diagnostics, configuration, or generated
files. Keep `proctor/` self-contained. Do not edit prior detailed plans or their
historical sections in `prototype-plan.md`.

The research objective is to learn reusable typed local rewriting rules from
successful Rust transformations. Whole-program pointer analysis still fixes
target types and skeletons before SCC work. The current stage compiles the
entire crate after replacing each SCC, so Crat inserts compatibility wrappers
and redirects untouched callers to make that mixed-signature crate compile.
This creates an unwanted behavioral boundary between transformed Rust I/O and
untouched libc I/O, among other possible effects. Incremental behavioral tests
against that mixed crate are deliberately deferred. Instead, build an additive
crate containing only already accepted functions, and perform compilation
validation after every addition. Compilation establishes type/build validity,
not semantic equivalence. Test execution and whole-program agent repair are
future work and are not part of this change.

## 2. Fixed decisions and scope

- Analyze the complete prepared library once. Keep its skeleton records and
  leaf-first SCC order fixed throughout the run. Do not rerun pointer analysis
  on the shrinking or growing partial crate.
- Start the accepted target source with all non-function crate context needed
  for compilation: crate attributes, inline module structure, imports, foreign
  declarations, type definitions, constants, statics, and other supported
  non-function items. Omit every source-defined free function, including
  `main`. Preserve source order within the retained context.
- Run `cargo build --lib` on that initial partial project. Failure is terminal.
  In particular, globals that require omitted function definitions are outside
  the supported input; do not synthesize placeholders or defer them.
- Add one SCC's transformed implementations to the partial target source at a
  time. Do not insert temporary wrappers or untransformed caller bodies there.
  A candidate is accepted only after existing structural validation when LLM
  output was used and a transactional `cargo build --lib` succeeds.
- Preserve the current applied-view, whole-SCC baseline fallback, bounded LLM
  repair, deterministic mechanical conversion, statement-pair, and accepted
  observation behavior. Retain every eligible observation actually extracted
  from an accepted SCC; conservative extraction may yield none for a region.
  No repair-based invalidation exists yet.
- For a library, after all SCCs, add compatibility wrappers only for
  `proctor.toml` API functions whose accepted signatures differ from their
  original signatures. Call the existing `build_wrapper`/conversion/name and
  export logic unchanged. Record only these permanent wrapper relationships in
  `proctor.toml`.
- After all SCCs, restore every excluded source-defined `main` consistently
  for libraries and executables. For a supported two-argument `main_0`, use
  the exact existing fixed sibling-`main` forwarding body; for a zero-argument
  form and any other ordinary `main`, retain the original body. `main` is
  never an API entry or SCC member. The generated Cargo bin shim remains
  untouched. Run a final ordinary `cargo build` on the complete project for
  either target kind.
- Assume no wrappers from earlier stages: a nonempty input
  `proctor.toml` `wrappers` list is unsupported. The stage does not infer or
  repair unrecorded upstream wrappers. Calls requiring redirection inside
  macro token input are unsupported and fail clearly.
- Do not change the stage envelope, required/produced artifact kinds, rule
  documents, prompt/validator semantics, wrapper conversion formulas, or
  formatting/libc behavior. Do not execute a test package within the stage.

## 3. Present implementation and ownership

The Python entry point is `proctor/stages/local-transformation/stage.py`.
`model.py` owns skeleton records, graph, SCC schedule, and replacement metadata;
`protocol.py` builds tool requests; `tooling.py` invokes Crat and Cargo.
Currently the stage copies input to `work/current`, prepares it, makes
skeletons, normalizes safety, performs one full `cargo build`, invokes Crat
`replace` for each SCC, installs its complete candidate source around another
full build, extracts observations after acceptance, and publishes a project
plus reports. The candidate contains untouched callers and temporary wrappers.

`proctor/stages/crat/src/bin/crat-tool.rs` owns file I/O and atomic publication.
`proctor/stages/crat/crates/tools/src/item_replacer.rs` owns AST/HIR target
resolution, canonical implementation composition, wrappers, call rewriting,
and the fixed `main_0` boundary. `observation.rs` owns typed extraction and
digest-bound correspondence. New compiler-resolved assembly belongs in Crat,
while scheduling, transactions, Cargo builds, and reporting stay in Python.
The old `replace` operation can remain for existing direct callers, but the
revised local stage must use a new additive operation and must not invoke
`build_wrapper` for each SCC. Otherwise an internal signature change with an
unsupported wrapper conversion would still fail before the final API boundary.

The input manifest is created by `proctor/stages/crat-adapter/main.py` and
contains `target_kind`, `target_name`, `api_functions`, and `wrappers`.
`proctor/proctor/contracts/manifest.py` provides its typed reader/writer.
The local stage currently ignores this manifest; it must now read it, reject a
nonempty wrapper list, and publish an updated copy after finalization. Its
`ProjectManifest.load`/`dump` round trip omits unknown TOML fields, so use a
surgical TOML update rather than that writer when recording wrappers.

## 4. Data flow and retained invariants

Keep two source roles in the stage work directory:

1. An immutable, complete, normalized **analysis source**. Produce it by
   copying the input, applying the existing dependency/preparation sequence,
   generating the dual-view skeleton records, and normalizing safety. Crat
   compiles this full source for each compiler-resolved target lookup. Never
   install partial candidates into it.
2. A mutable **accepted target project** with the same Cargo layout and
   dependencies. Replace only its library source with Crat's non-function
   projection, then append build-accepted SCC implementations. Its binary
   source and manifest remain in the copied project but `--lib` avoids the
   binary until finalization. Scratch observation source and temporary stubs
   never enter this project.

For every SCC, the additive Crat operation receives the immutable analysis
project/source, the currently accepted partial target source, the existing
version-1 replacement request, and distinct output paths. It compiles the
complete normalized source to resolve original item paths and direct calls,
validates the requested items against their original functions, and uses the
same canonical restoration and header checks as `replace`. It inserts only
the new implementations into the corresponding modules of the partial target
AST. It may reuse implementation-composition code, but cannot create a
compatibility wrapper, redirect a target call to one, or require
`input_conversion`/`output_conversion` at this step. The candidate contains
each accepted function exactly once at its logical original crate-relative
path. Repeated paths, unsupported module shape, a mismatch between the
request/analysis source/partial target, or a non-monotone accepted set are
fatal protocol errors. Insert all members of an SCC atomically, so recursion
and mutual calls become available together. Preserve unrelated accepted
function bytes or AST meaning and retained non-function context.

Function bodies in the target call accepted callees at their target
signatures. The leaf schedule ensures current members cannot call a later
unaccepted local function through supported direct-call edges. The target
build, not a rewritten old caller, checks this property. Do not place
unaccepted signatures or `todo!()` placeholders into the target merely to
make a build pass. Supported projects cannot rely on function pointers or
other hidden dependencies outside the existing graph contract.

Define three thin `crat-tool` operations with exact path roles:

```text
crat-tool make-initial --output <initial.rs> <analysis-project>
crat-tool add-functions --request <request.json> \
  --current-project <partial-project> --output <candidate.rs> \
  --statement-pairs-output <pairs.json> \
  --observation-source-output <observation.rs> \
  --observation-metadata-output <metadata.json> <analysis-project>
crat-tool finalize-project --manifest <proctor.toml> \
  --current-project <partial-project> --output <final.rs> \
  --manifest-output <candidate-proctor.toml> <analysis-project>
```

`add-functions` consumes the existing version-1 `ReplacementRequest`, whose
`accepted_correspondence` identifies functions already installed in the
partial project. It emits candidate source, the existing version-1 canonical
statement-pair sidecar, separate observation source, and digest-bound
metadata. Keep all output paths distinct from each other and all input paths;
publish each operation's outputs together or none at all, following the
existing `replace` file-publication pattern. The new observation metadata is
schema version 2: retain every existing field and add required
`source_stubs: [{"item_id": <u64>, "path": <crate-relative path>}, ...]`,
sorted by item ID, for precisely the emitted crate-visible old-call stubs. Here
`path` is the actual generated stub path in the observation source, for
example `inner::__proctor_source_stub_leaf`, not the original logical
`inner::leaf` path. Use `__proctor_source_stub_<original name>` as the base
generated name; collision suffixes follow the existing generated-name
allocator.

`accepted_correspondence`, `new_correspondence`, and `current_items` continue
to use the existing closed shapes, but new entries carry `wrapper_path: null`
because no compatibility wrapper exists. The existing observation extractor
must accept both old version-1 metadata from `replace` and new version-2
metadata from `add-functions`, while the revised Python stage validates the
new version and its stub mappings. Do not change the public stage schema.

## 5. Observation source without target wrappers

The observation source is temporary compiler input for typed extraction, not
an output crate or acceptance candidate. Construct it from the current
partial target candidate, then add, in their original modules:

- labeled source copies of exactly the current SCC's original functions, with
  collision-free `pub(crate)` names and export attributes removed;
- old-signature `pub(crate)` `todo!()` call stubs for accepted callees used by
  those source copies when old call syntax needs an old-signature destination;
  and
- any compiler-only labeling of the current accepted implementations required
  by the existing observation extractor.

Resolve original calls with the complete analysis `TyCtxt`. Within a source
copy, a call to another current SCC member (including self recursion) names
that member's source copy; a call to a previously accepted callee needing its
old signature names that callee's crate-visible stub. Calls to foreign declarations
retain their resolved target. No original caller bodies outside the current
SCC are copied. Reject a call requiring one of these rewrites if it occurs
only in macro token input. Generated stubs do not perform conversion, are not
observed as transformations, and cannot enter candidate/accepted source.
Give source copies and stubs `pub(crate)` visibility even when their original
function was private, so compiler-resolved cross-module SCC calls and old
calls can reach their generated destinations. Strip export attributes from
both kinds of scratch item. Their visibility and bodies affect only the
temporary observation source. `current_items` maps each source copy's
compiler identity to its logical item ID, while metadata `source_stubs` maps
each stub's compiler `DefId` to its accepted logical item ID. Stub paths are
distinct from `wrapper_path` and do not persist in stage accepted
correspondence. Allocate generated names against the complete occupied
namespace so source copies and stubs cannot collide
with accepted implementations, imports, or one another.

Keep the current source-to-target label alignment, `printf` argument recovery,
statement-pair selection, and observation document shape. Map the stub's
compiler identity, the accepted implementation identity, and current
source-copy identity to the same logical function ID as appropriate. Extend
the correspondence checks narrowly so every referenced identity exists and
maps unambiguously. The source copy retains original types and original call
syntax, while the target implementation has transformed types. A stub is a
typing aid only; its body is never executed. Preserve the current rule that
observations are extracted only after the candidate's successful Cargo build,
and that a failure during extraction is terminal rather than prompting LLM
repair. Rule-complete and mechanical SCCs with no transform labels still skip
extraction. Failed attempts and abandoned applied views contribute no
observations.

## 6. Candidate transaction and SCC control

The Python stage still accepts applied views first, skips the LLM/validator
when no transform region remains, and otherwise makes at most one initial LLM
generation plus ten repairs across fallback. It uses the same prompt context,
strict validator, canonical Crat replacement checks, and latest-failure
diagnostics. A rule-involved `cargo build --lib` failure rolls the whole SCC
back and retries baseline views; a mechanical build failure without a rule
remains fatal. Wrapper conversion errors for internal functions cannot occur
because no internal wrapper is requested.

Before installing a candidate, validate its sidecar and metadata against the
request, the accepted correspondence, and all three digests. Install only the
candidate library source into the accepted target project. Save the previous
source and restore it on nonzero build or builder exception, treating failed
restore as fatal. Cargo `target/` updates may remain after a source rollback,
as today. On success, retain the candidate, then extract observations from
the scratch source, promote only the successful sidecar rows, accepted
correspondence, views, and observation document. An extraction failure stops
the stage without publishing a usable final project. Maintain the existing
report, statistics, LLM usage, and output transaction semantics. Count the
initial partial build, every SCC attempt, and the final full build in Cargo
metrics; diagnostics should name the command actually run.

## 7. Finalization and project manifest

After all SCCs have compiled, invoke a thin Crat finalization command from
Python with the accepted project, the immutable normalized analysis source,
and the copied `proctor.toml` (or an equivalent request that carries its API
list and target kind). Crat owns AST-level function identity, signature
comparison, exact wrapper creation, export handling, and source output. The
command may parse a TOML file with other fields but must use only the needed
manifest fields and preserve the remaining fields when writing the manifest.
Stage an updated manifest separately from the copied original, changing only
the `wrappers` value; retain additional tables, comments, and formatting where
the selected TOML editing library permits. Reject a library API list containing
`main` before function selection; `main` is not an API target. For every other
API entry, select all original source functions whose Rust final identifier or explicit
`export_name` matches it, including same-name functions in different modules
as specified by `docs/proctor-spec.md`. An entry matching no function fails
clearly, as does an incompatible duplicate export-symbol conflict. Deduplicate
matched function identities across API entries so one function receives at
most one wrapper. Compare original and accepted signature types using the
existing `signature_types` rule. If unchanged, keep the implementation and
add no wrapper; if changed, call the existing
`allocate_wrapper_name`, `build_wrapper`, and export-transfer logic with no
conversion edits. A changed non-API function receives no wrapper. Generated
wrapper relationships use crate-relative full paths in deterministic source
order, and only those relationships enter `proctor.toml` `wrappers`.

Restore every excluded source-defined `main` at finalization for libraries and
executables, using `fixed_main_item` exactly where the existing two-argument
`main_0` logic requires it. Preserve original `main` otherwise, including the
supported zero-argument form. Do not modify the Cargo bin shim. A `main` that
was excluded from the SCC schedule is never sent to the LLM or observation
extractor. If an unchanged `main` calls a transformed callee using an obsolete
signature, the final full build fails and the stage aborts: no temporary
internal wrapper or implicit `main` rewrite is added to mask the mismatch.

Stage finalization transactionally: install the finalized source, update
`proctor.toml` as one paired attempt after preflighting and validating both
staged outputs. Keep the previous accepted source and manifest bytes available;
run the final ordinary `cargo build` with both staged files installed, and
restore both on failure or builder exception. Only after that build succeeds
may the pair be retained and published. A final wrapper, entry-point, or build
failure is terminal; do not reinterpret it as an SCC
repair. Do not publish an incomplete target or claim behavioral correctness.
Preserve the input manifest itself by working only on the copied project.

## 8. Failure boundaries and supportedness

Reject a missing/malformed `proctor.toml`, an invalid target kind/API list
(including a library API entry `main`), or a nonempty `wrappers` list before
mutating the accepted project. Project
and path validation continue to precede tool work. A function-dependent
global typically makes the initial `cargo build --lib` fail; report that
failure and abort. Preserve other initial build failures as terminal too.
Do not add synthetic declarations to compensate. A macro-hidden required
source-copy redirect fails during Crat assembly, before candidate acceptance.
Any unmatched API entry, wrapper conversion unsupported by existing Crat
logic, collision that the existing allocator cannot resolve, or final Cargo
failure aborts finalization. In each case emit the ordinary failure envelope
and no usable output project.

No incremental test-vector gate is added. The stage does not receive a
`test_package` under its manifest, and the generated observation set remains
compilation-accepted evidence only. Future whole-program test and repair work
may discard observations from subsequently modified functions, but that
policy has no implementation in this revision.

## 9. Implementation sequence and verification

1. Add Crat library routines for projection and additive SCC assembly, then
   thin CLI commands. Reuse canonicalization and sidecar production; isolate
   wrapper-only code from additive insertion. Keep old `replace` behavior for
   direct callers unless there is a proven reason to remove it.
2. Extend scratch observation construction and correspondence to support
   source-copy old-call stubs with no target wrapper. Verify typed extraction
   for changed internal signatures, recursion, foreign calls, and accepted
   callee identity.
3. Add final API/executable boundary generation using the unchanged wrapper
   implementation. Keep its result separate from SCC observation files.
4. Wire Python to retain the full analysis source, project the partial crate,
   build `--lib` initially and per SCC, finalize after the schedule, run full
   Cargo build, and update the copied manifest. Update command builders,
   metadata loaders, fake-tool tests, and metrics in lockstep.
5. Implement every updated and new case in
   [amendment-10-test-plan.md](amendment-10-test-plan.md), including exact
   input/output assertions and rollback/failure cases. Update the current
   [prototype description](prototype-desc.md) through the `prototype-desc`
   skill after code lands; update `docs/unsupported.md` only for actual changed
   supportedness. Do not revise historical plans.

Run `uv run pytest tests/test_local_transformation.py` from `proctor/`, then
the relevant default Python suite and static checks. Run focused Crat
`cargo test -p tools item_replacer::tests` and
`cargo test -p tools observation::tests`, expanding to `cargo test -p tools`
for shared APIs and `cargo test --workspace` when practical. Crat tests must
use in-memory library/compiler harnesses: they cannot invoke `crat-tool`,
mutate filesystem state, or live in a project-root `tests/` directory. After
modifying Crat Rust source, run `cargo fmt` and
`cargo clippy --workspace --all-targets` from `proctor/stages/crat/`, resolve
warnings, and use targeted Clippy allowances only where repository policy
permits. A real-toolchain smoke run should cover both a library API and an
executable with the checked-in local pipeline when prerequisites are present;
the default test suite must remain offline and API-key-free.

## 10. Completion criteria

- Before the first SCC, the accepted library contains retained non-function
  context and no source-defined function; its `cargo build --lib` succeeds.
- After each accepted SCC, exactly that SCC's transformed implementations
  have been added; no original caller, temporary wrapper, source copy, or
  stub appears in the accepted source. Every accepted candidate built with
  `cargo build --lib`; failed candidates restore the previous source.
- Changed non-API signatures never require wrapper conversion. Changed API
  signatures receive the same wrapper source/export behavior as the current
  generator, once, at finalization. Unchanged APIs receive none.
- The final complete Cargo project passes ordinary `cargo build`; its
  `proctor.toml` records precisely the permanent API wrappers. Eligible
  observations actually extracted from accepted SCCs and deterministic
  reports are retained.
- Test execution and semantic repair remain absent. No input project or
  prior historical section or detailed plan was revised.
