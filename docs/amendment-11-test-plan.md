# Amendment 11 Test Plan: Direct Additive Finalization

## 1. Purpose and authority

This is the acceptance plan for [amendment-11-plan.md](amendment-11-plan.md).
The active stage keeps additive SCC assembly, observation extraction,
`main` restoration, and the final Cargo build. Finalization keeps every
accepted function at its original path with its transformed signature and
original export metadata. It neither compares API signatures for ABI
compatibility nor creates wrappers. The unused general Crat `replace`
operation is removed. The ordinary Crat `interface` pass is neither changed
nor run after local transformation.

The user accepts that this stage no longer promises the original C ABI. An
exported Rust slice reference or boxed slice may be wide where the original
C parameter was one raw pointer. Cargo build success is not an ABI proof;
these tests must not claim that old C callers can use such exports.

Update and retain the existing tests below. **No new Crat test function or
exhaustive type matrix is required.** The wrapper behavior has been removed,
so representative existing finalization tests and their changed assertions
cover it. Keep earlier plans and their historical sections unchanged.
Planning labels such as `A11-*` belong only in plan files, never in code,
tests, fixtures, diagnostics, or configuration.

## 2. Test layers and verification

Crat tests stay beside the implementation under
`proctor/stages/crat/crates/tools/src/`. Use the existing in-memory compiler
harness; Crat tests must not invoke the `crat-tool` CLI, change filesystem
state, or use a Crat-root `tests/` directory. A source assertion checks
crate-relative function paths/count, accepted signature/body, visibility,
calling convention, and export attributes, ignoring only printer whitespace.
The retained `finalize` test helper compiles the analysis source in memory.

Python protocol, fake-tool stage, final Cargo-command, rollback, manifest,
and publication coverage stays in `proctor/tests/test_local_transformation.py`.
Its fake tool/runner may write to `tmp_path`; the default suite does not need
a real Crat/C2Rust binary, network, or API key. The existing Python tests
check unchanged `wrappers=[]` in the final manifest and the unrestricted
final Cargo build. A real Cargo fixture is optional `e2e` coverage, not a
required test for each API type.

Run after implementation, from `proctor/stages/crat`:

```bash
cargo test -p tools item_replacer::tests
cargo test -p tools skeleton::tests
cargo test -p tools
cargo fmt
cargo clippy --workspace --all-targets
```

Run from `proctor`:

```bash
uv run pytest tests/test_local_transformation.py tests/test_manifest.py
uv run pytest
uv run ruff check .
uv run ruff format --check .
uv run mypy proctor
uv run proctor validate -c configs/c2rust_crat_local.toml
```

## 3. Update existing Crat additive and finalization tests

The following are existing functions in `item_replacer/tests.rs`. Rename
ones whose old name promises wrappers. Use their current in-memory fixtures;
only the specified input variant or expectation changes. The finalization
result no longer has a `wrappers` field. No case invokes general `replace`.

| Existing test to update | Concrete input | Expected output |
| --- | --- | --- |
| `additive_internal_signature_change_has_no_wrapper_and_uses_old_call_stub` | Existing `inner::leaf(p:*mut i32)->i32` and `inner::caller`, with accepted `leaf(p:Box<[i32]>)->i32` and accepted caller. Finalize the accepted project with API list `leaf`. | Keep existing first/second SCC and observation-only old-call-stub assertions. Finalization now succeeds: exactly one direct `inner::leaf` has `Box<[i32]>`; no generated wrapper/conversion. The old `UnsupportedConversion` assertion is removed. |
| `finalization_wraps_only_changed_api_in_source_order` | Existing `#[no_mangle] extern "C" first(*const i32)`, private `middle(*const i32)`, and `#[export_name="public_last"] extern "C" last(*const i32)` plus `main`; accept `first(&i32)`, `middle(&i32)`, and change the existing `last` accepted signature to `last(&[i32])` with body `p[0]`. API list remains `public_last,first`. | One accepted function at each original path; `main` restored; `first` retains `#[no_mangle]`, `last` retains its one `export_name`, and both retain `extern "C"`. No new `__proctor_wrapper_*`, `from_raw_parts`, or unsafe conversion appears. The accepted slice signature is direct despite its C ABI limitation. |
| `finalization_restores_original_main_and_fixed_two_argument_main` | Existing ordinary `main`, zero-argument `main_0` plus sibling `main`, and two-argument `main_0(argc:i32, argv:*mut *mut i8)` accepted as `main_0(argc:i32, argv:&mut [&mut [i8]])`. | Ordinary and zero-argument `main` restoration remains as before; the two-argument case keeps its fixed sibling `main` argument-slice forwarding body. Existing in-memory compile checks remain. Remove only the obsolete final-wrapper result assertion. |
| `finalization_requires_sibling_main_only_for_two_argument_main_0` | Existing two-argument `main_0` without sibling `main`; separately zero-argument `main_0` without sibling. | Two-argument case still returns `RewriteFailure` with the exact message `` two-argument `main_0` requires exactly one sibling `main`, found 0 ``; zero-argument case finalizes without creating `main`. Delete the comparison against the removed general `replace` operation. |
| `finalization_selects_all_same_named_functions_and_deduplicates_api_entries` | Existing `a::work(*const i32)` and `b::work(*const i32)` each accepted as `work(&i32)`; API list `work,work`; separately API `missing`. | One direct accepted `work` in each module, no generated sibling. Duplicate API entries add nothing. `missing` still yields `TargetResolution` naming the missing entry. |
| `finalization_deduplicates_rust_and_export_names_for_one_function` | Existing `#[export_name="public_work"] extern "C" work(*const i32)` accepted as `work(&i32)`; API list `work,public_work,work`. | Exactly one direct `work(&i32)` retains one `#[export_name="public_work"]` and `extern "C"`; no wrapper or duplicate export. |

The first, second, and last rows cover boxed slice, ordinary reference,
slice reference, `no_mangle`, and explicit export without a new matrix. The
type-independent finalizer does not need separate tests for every other
pointer spelling. The existing unmatched-API check remains tested by
`finalization_selects_all_same_named_functions_and_deduplicates_api_entries`.
Keep the production accepted-set and non-function-context guards; the user
has not requested new tests for those unchanged checks.

## 4. Remove general `replace` tests while retaining shared checks

- Delete the `replace`, `replace_output`, and `replace_extended` test helpers
  in `item_replacer/tests.rs`. Delete tests dedicated only to general
  replacement wrappers, raw-pointer conversions, old-caller redirects, and
  its observation-only copy path. Do not delete tests by a `replace*` name
  alone: `ReplacementRequest`, shared preservation, statement pairs, and
  additive source-copy behavior still exist.
- In existing `versioned_request_json_round_trip_preserves_rust`, keep the
  literal schema-version-1 request and its JSON/label assertions; remove its
  trailing general `replace` call. Keep
  `request_json_rejects_unknown_fields_and_non_u64_numbers` unchanged.
  For existing request-validation functions such as
  `unsupported_version_and_empty_items_are_rejected`,
  `duplicate_ids_paths_and_names_are_rejected_deterministically`,
  `path_name_disagreement_and_invalid_paths_are_rejected`, and
  `transformation_must_be_exact_supported_requested_function_set`, call
  `add` on an initial function-free projection of the same concrete source
  instead of `replace`; retain each current invalid-request or
  invalid-transformation result and deterministic diagnostic assertion.
- Migrate existing preserved-group, rule-applied, canonical restoration,
  and statement-pair tests to `add` only where they cover logic shared by
  `prepare_transformations` or additive output and no equivalent retained
  test already covers it. Keep their original source/returned transformation
  and expected canonical statement or sidecar; remove assertions specific
  to temporary general-replace wrappers/call redirects. Delete duplicate
  legacy-only tests after confirming retained preservation/validator/additive
  coverage. This is migration of existing cases, not creation of a new test
  family.
- In `skeleton/tests.rs`, retain the request JSON and skeleton assertions
  in `existing_pointer_and_protocol_regressions_change_only_rendered_tools_types`
  but remove its trailing `replace_items` smoke invocation. In
  `generated_local_name_validates_replaces_and_compiles_in_original_module`,
  retain the same nested-module generated-local-name fixture and validator
  checks. Replace its `replace_items` tail with `make_initial_source` plus
  `add_functions_with_observations` on the same request, then assert the
  accepted source still contains `let mut init: cb_rgb`, contains
  `pub unsafe fn cb_remove_gamma_rgb`, has no `#[proctor(...)]` labels, and
  type-checks in memory. No Crat test invokes the CLI or writes files.
- Remove the legacy version-1 observation metadata emitter if deleting
  general `replace` makes it unused. Preserve version-1 metadata **reading**
  for old artifacts and keep the current version-2 additive emitter/schema
  tests. The retained additive correspondence still has
  `wrapper_path:null`; old-call stubs exist only in observation scratch
  source, never in accepted/final source.

## 5. Update existing Python stage/tooling tests

| Existing test | Concrete change and expected result |
| --- | --- |
| `test_additive_command_builders_keep_analysis_and_current_roles_distinct` | Remove only the assertion constructing `replace_command`. Keep exact `make-initial`, `add-functions`, and `finalize-project` argument/path assertions. |
| `test_final_build_failure_restores_source_and_manifest_together` | Existing fake finalizer writes direct accepted `f(&[i32])` source and an unchanged input manifest with `wrappers=[]`; its fake unrestricted Cargo build returns 9 with `final error`. Expect failure envelope, no usable output, and restoration of both prior stage-work source and manifest. Remove the fake `final with wrapper` and nonempty-wrapper manifest. |
| `test_additive_builds_keep_complete_analysis_and_grow_target_by_scc` | Keep the existing leaf/caller fixture, initial `cargo build --lib`, one `--lib` build per accepted SCC, and final unrestricted build. Accepted/final target contains no wrappers, and final manifest remains `wrappers=[]`; analysis source stays complete. |
| `test_invalid_project_manifest_fails_before_tool_work` | Keep its existing nonempty-wrapper input row. Expect failure before tool work and no output; input manifest unchanged. |
| Existing successful publication tests, including `test_final_output_excludes_only_root_target` | Keep final unrestricted build, success envelope, published reports, and no scratch source copies/stubs in the final project. Ensure fake finalization leaves `wrappers=[]`. |

Delete `test_crat_tools_replace_clears_and_requires_both_scratch_outputs`
and other tests/branches dedicated solely to `CratTools.replace`. The
existing `FakeTools.add_functions` delegates to `FakeTools.replace`: rewrite
that existing fake to write its candidate and sidecars directly before
removing the fake `replace` method. Remove old command-only error/cleanup rows;
keep all `add_functions`, `finalize_project`, output-collision, build
rollback, observation, and publication rows. No new Python test function is
needed if these existing cases retain the stated assertions.

## 6. Completion and known limit

After edits, searching implementation surfaces finds no general
`crat-tool replace` command, exported `replace_items` API, Python
`replace_command`, or `CratTools.replace` method. Check this by source review
and compilation; do not add a new CLI-invoking or Clap-parse test solely to
prove absence. Keep the checked-in prototype config ending in
`local_transformation`, with no post-local `interface` invocation.

For the updated `last(*const i32)` to `last(&[i32])` fixture, final source
intentionally retains the latter signature and original export. The test
checks direct Rust output, not C-call compatibility. Leave historical test
plans unchanged and keep `proctor/` independent of enclosing workspace docs.
