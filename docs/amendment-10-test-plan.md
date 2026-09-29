# Amendment 10 Test Plan: Additive Function Assembly

## 1. Purpose and authority

This is the acceptance plan for [amendment-10-plan.md](amendment-10-plan.md).
The stage continues to analyze the complete prepared source but builds a second,
initially function-free library source. It adds one accepted leaf-first SCC at a
time, checks the growing library with `cargo build --lib`, and adds only final
executable or library API boundaries after every SCC. A final unrestricted
`cargo build` checks the deliverable. No test package is run and no semantic
correctness is claimed.

The user has fixed these limits: assume there are no wrappers from earlier
stages; a global that requires an omitted function is unsupported and an
initial build failure aborts; a call that cannot be redirected in the
observation-only source because it is hidden in macro input is unsupported;
and existing wrapper naming, conversions, and export behavior are retained
exactly. Library wrappers are generated only for API functions whose
signatures change. Every eligible observation actually extracted from an
accepted transform region is retained;
discarding observations after future test-based repair is outside this work.

Tests, schemas, and source remain authoritative if they expose a factual
conflict. A change to an outcome specified here or in the implementation plan
needs the user's decision; it must not be concealed as a test adjustment.
Identifiers such as `A10-*` and “Amendment 10” belong in planning documents
only, never in source, tests, fixtures, diagnostics, or configuration.

## 2. Test ownership and execution

Crat tests belong beside the pure implementation in
`proctor/stages/crat/crates/tools/src/`, especially `item_replacer.rs` and
`observation.rs`. They use in-memory sources and the existing compiler
harness where needed. They must not call the `crat-tool` CLI, modify a project
tree, or use a Crat-root `tests/` directory. A new thin CLI command can have
argument and serialization tests only where this can be done without calling
the binary or writing files.

Python protocol, fake-tool stage, Cargo-command, manifest, transaction, and
publication cases belong in `proctor/tests/test_local_transformation.py`.
Project-manifest model cases also belong in `proctor/tests/test_manifest.py` if
the model changes. Fake tools write only into `tmp_path` stage work and output
locations. The default suites need neither network nor a real model/toolchain.
Use a real Cargo/Crat flow only for optional `e2e` tests when prerequisites are
available; it cannot replace focused fake and pure tests.

Run after implementation, from `proctor/stages/crat`:

```bash
cargo test -p tools item_replacer::tests
cargo test -p tools observation::tests
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

## 3. Shared fixtures and comparison rules

The ordinary fixture is one root-level `[lib].path = "lib.rs"` Cargo project,
with `proctor.toml` and an optional explicit forwarding `[[bin]]` source.
Unless a case says otherwise, use a library manifest with
`target_kind = "library"`, `api_functions = []`, and `wrappers = []`. Its
prepared, normalized library source is:

```rust
#![allow(dead_code)]
use core::ffi::c_int;
pub struct Cell { pub value: c_int }
pub static SCALE: c_int = 2;
mod inner {
    use super::{Cell, SCALE};
    pub unsafe fn leaf(p: *const Cell) -> c_int { (*p).value * SCALE }
    pub unsafe fn caller(p: *const Cell) -> c_int { leaf(p) + 1 }
}
```

The skeleton schedule is `inner::leaf` followed by `inner::caller`. For
source-to-target signature examples, assume Crat's skeleton chooses
`*const Cell -> &Cell`; tests inject records or compile a fixture whose
pointer analysis actually makes that choice. For an SCC whose chosen view has
no transform labels, the skeleton itself is the mechanical candidate. A
returned LLM function always has the same validated labels and target
signature as its selected view. Source formatting may be compared as AST;
function presence, path, signature, body identity, wrapper/export status,
Cargo command arguments, manifest fields, sidecars, event order, and failure
status are exact.

In case descriptions, `P0` means the projected initial source and `P1`,
`P2` mean the accepted sources after the first and second SCC. `O1` means the
separate observation source for the first accepted SCC. The source analyzed
for skeletons is the full prepared/normalized input, even after `P0` is
installed. A successful build means exit code 0; a failed build means a
nonzero code with controlled diagnostics. Every source assertion checks the
whole relevant module, not just a substring. Every negative asserts both
failure class and that no later operation was invoked.

## 4. Updated existing regression cases

### A10-UPD-01 `initial_build_uses_projected_library`

Input: the shared two-function fixture, with a fake Cargo runner recording
arguments. Expected: skeleton generation sees both `inner::leaf` and
`inner::caller`; the first build sees `P0`, whose `inner` module contains its
`use` but neither function; the command is `cargo build --lib`. The old
full-source initial build is absent.

### A10-UPD-02 `incremental_builds_accept_only_added_sccs`

Input: the shared fixture, successful fake builds, and valid candidate
implementations. Expected: build sources are `P0`, `P1`, `P2`, followed by
the finalized source. `P1` contains `inner::leaf` and no `inner::caller`;
`P2` contains both. All three incremental commands include `--lib`; the
last command is unrestricted `cargo build`. `cargo_builds` is 4.

### A10-UPD-03 `old_wrapper_and_caller_rewrite_tests_change_scope`

Input: `leaf(p: *const Cell)` becomes `leaf(p: &Cell)` while `caller` has
not been transformed. Expected in `P1`: only the new `leaf`; no
`__proctor_wrapper_leaf`, source-form `caller`, or rewritten old call.
Expected in `P2`: validated `caller` calls `leaf` using `&Cell`; no temporary
wrapper. Any earlier test expecting a wrapper or old-call redirect in the
target must be updated to this result.

### A10-UPD-04 `accepted_reports_keep_their_existing_meaning`

Input: `leaf` has transform label 0 and `caller` has mechanical label 0.
Expected: `statement-pairs.md` contains both accepted canonical pairs in
item/label order; `observations.json` contains accepted `leaf` observations
and no observation for the mechanical `caller`; `statistics.json` counts the
same function/SCC, LLM, repair, and compilation events as the actual run.
Neither scratch source nor correspondence metadata is published.

## 5. Projection and additive source construction (Crat library)

### A10-PROJ-01 `preserves_non_function_crate_structure`

Input: the shared source plus `#[cfg(any())] const UNUSED: i32 = 1;`, a
type alias, an enum, a union, an `extern "C"` block, and a nested module
`mod deep { pub struct T; pub unsafe fn f() {} }`. Expected `P0` preserves
the crate attribute, imports, `Cell`, `SCALE`, the additional non-function
items, foreign declarations, module hierarchy, and `deep::T` in source order;
`inner::leaf`, `inner::caller`, `deep::f`, and any source-defined `main` are
absent. No empty module is collapsed if it carries attributes/imports.

### A10-PROJ-02 `same_named_functions_remain_path_distinct`

Input: `mod a { pub unsafe fn work() {} } mod b { pub unsafe fn work() {} }`.
Expected `P0` contains both modules without `work`; after adding `a::work`,
only `a::work` appears; adding `b::work` yields one function in each module.
Neither insertion overwrites the other or creates a wrapper.

### A10-PROJ-03 `source_and_target_are_separate`

Input: shared source and a successful `leaf` addition. Expected: the
immutable analysis source still contains both original functions and old
signatures, while `P1` contains only transformed `leaf`; subsequent
skeleton/prompt records retain the original `caller` body and target view.
Output copying contains `P1`/later target only, never the full analysis
source or a scratch observation source.

### A10-PROJ-04 `unrelated_compilable_non_function_items_survive`

Input: a root `const BASE: i32 = 3`, `static FLAG: bool = true`, a type
alias, and `fn f() -> i32 { BASE }`. Expected `P0` has the three non-function
items and no `f`; its initial library build succeeds; after `f` is added,
the candidate retains their identifiers and values unchanged.

### A10-PROJ-05 `function_dependent_global_aborts_at_initial_build`

Input: `static ANSWER: i32 = answer(); const fn answer() -> i32 { 42 }` in
the prepared source. Expected: projection keeps `ANSWER` and omits
`answer`; the initial `cargo build --lib` fails; stage status is failure,
with zero SCC additions, zero LLM requests, and no usable output project or
report. There is no deferred-global workaround.

### A10-PROJ-06 `unresolved_function_import_aborts_at_initial_build`

Input: `mod m { pub fn f() {} } use m::f;`. Expected `P0` retains the import
and omits `m::f`; an unresolved-import initial build aborts before any SCC.
The implementation must not silently discard the import to make the fixture
pass.

### A10-PROJ-07 `candidate_rejects_wrong_path_or_duplicate_installation`

Input: request to add `a::f` when the analysis source contains only
`b::f`, or request `a::f` when it already exists in the accepted target.
Expected: Crat returns a structured
target-resolution/rewrite failure, no partial candidate source is published,
no build runs, and the previously accepted target remains byte-for-byte
unchanged. Distinct `a::f` and `b::f` remain legal as in PROJ-02.

### A10-PROJ-08 `SCC_is_inserted_atomically`

Input: mutually recursive `even` and `odd` form one SCC, with a valid
two-function transformation; then repeat with only `even` returned or one
invalid target. Expected: success inserts both functions together before one
build; failure inserts neither, gives validator `invalid` or replacement
error as appropriate, and makes no half-SCC current source. Calls between
the two transformed functions use direct target signatures.

## 6. SCC scheduling, compilation, and recovery (Python fake protocol)

### A10-SCC-01 `leaf_first_order_and_tie_break_remain`

Input: `root -> left`, `root -> right`, with independent leaf IDs 5 and 2.
Expected additions/builds occur for ID 2, ID 5, then `root`, regardless of
source order, followed by one final build; accepted pairs and observation
documents retain that schedule order. Self-recursion remains one SCC.

### A10-SCC-02 `caller_is_not_required_for_callee_build`

Input: `caller(raw)` calls `leaf(raw)` in the original source; target
`leaf(&Cell)` compiles on its own, while old `caller(raw)` would not type
check against it. Expected first candidate `cargo build --lib` succeeds
because `caller` is absent. No wrapper is requested. The next candidate adds
the transformed `caller(&Cell)` and builds.

### A10-SCC-03 `non_api_unsupported_wrapper_conversion_is_irrelevant`

Input: an internal non-API function has old input `*mut i32`, target input
`Box<[i32]>`, and a transformed body valid in the target crate; its caller
is added later with a valid `Box<[i32]>` argument. Existing wrapper logic
rejects boxed-slice input conversion. Expected both incremental target
builds succeed and no wrapper conversion is attempted for this internal
function; neither target source nor manifest contains its wrapper. The
observation source may use a compiler-valid old-signature stub, never the
existing compatibility wrapper converter.

### A10-SCC-04 `one_build_failure_rolls_back_only_current_candidate`

Input: `leaf` accepted, then `caller` candidate produces a controlled Rust
type error. Expected transactional rollback restores exact `P1`, retains
earlier accepted pairs/observations, increments compilation failure once,
and supplies diagnostics to the next repair. No `caller`, wrapper, or scratch
observation source is installed in the current crate during repair.

### A10-SCC-05 `builder_exception_rolls_back`

Input: fake Cargo runner raises after candidate installation. Expected
previous library source is restored exactly; the stage emits failure if the
exception is terminal. A simulated restoration failure is separately fatal.
Cargo's `target/` cache need not roll back, as in the existing transaction.

### A10-SCC-06 `rule_fallback_reuses_the_same_additive_boundary`

Input: `leaf` applied view contains a rule and its candidate build fails;
baseline view then succeeds after one LLM repair. Expected first candidate
is rolled back, all members switch permanently to baseline, one repair
budget is consumed, the second candidate adds only baseline `leaf`, and
only the accepted baseline pairs/observations appear. No failed-rule
observation is retained.

### A10-SCC-07 `mechanical_failure_without_rules_is_fatal`

Input: a no-LLM SCC produces a mechanically complete but uncompilable
function and has no rule application. Expected one failed `cargo build
--lib`, exact source rollback, stage failure, no LLM request, no fallback,
and no extraction or finalization.

### A10-SCC-08 `validation_precedes_installation`

Input: LLM returns a function with a missing `#[proctor(0)]` label, then a
valid repair. Expected validator rejects the first text, no candidate is
installed/built for it, repair uses the unchanged partial crate, and the
second text can be accepted through replacement and `cargo build --lib`.
Preserved-region restoration remains the existing Crat behavior.

### A10-SCC-09 `no_functions_still_finalizes_and_builds`

Input: a library containing only `pub struct Cell;` with empty API list.
Expected empty schedule, one initial `cargo build --lib`, one final
`cargo build`, unchanged non-function library output, zero LLM requests,
empty version-1 observations document, and no wrappers.

## 7. Observation-only source and learning artifacts

### A10-OBS-01 `source_copy_compiles_off_target`

Input: first SCC changes `leaf(*const Cell)` to `leaf(&Cell)` and rewrites
`(*p).value` to `p.value`; original `caller(*const Cell)` remains unadded.
Expected target `P1` contains only transformed `leaf`. `O1` is a separate
compilable source containing the accepted target function, a source copy of
the old body, and no callee stub because this is the first SCC. The
correspondence maps the old and target regions to the same logical
`inner::leaf` and label 0. The extracted typed observation
contains the old dereference region and new field-access region. No stub or
source copy enters `P1` or the final project.

### A10-OBS-02 `recursion_and_previous_callees_resolve_in_scratch`

Input: accepted `leaf(p: &i32) -> i32`, originally
`leaf(p: *const i32) -> i32`, followed by one SCC containing
`even(p: *const i32, n: i32)` and `odd(p: *const i32, n: i32)`. Their
original bodies are respectively
`if n == 0 { *p + leaf(p) } else { odd(p, n - 1) }` and
`if n == 0 { *p } else { even(p, n - 1) }`; accepted signatures use `&i32`
and bodies retain the same control shape with dereferences and direct
target calls. Expected observation-source copies redirect `even` ↔ `odd`
to their crate-visible scratch source-copy paths and the old-signature
`leaf` call to one crate-visible scratch `leaf` stub. Accepted
implementations call the target functions directly. Extraction yields the
typed dereference-to-reference regions for
the accepted transform labels, and no stub is observed as a transformation.
The target crate contains only the three transformed functions and no
compatibility stubs.

### A10-OBS-02A `cross_module_old_callee_stub_is_visible_only_in_scratch`

Input: original source

```rust
mod callee {
    pub(crate) unsafe fn leaf(p: *const i32) -> i32 { *p }
}
mod caller {
    pub(crate) unsafe fn call(p: *const i32) -> i32 {
        crate::callee::leaf(p)
    }
}
```

After accepting `callee::leaf(p: &i32)`, add transformed
`caller::call(p: &i32)` with a direct target call to
`crate::callee::leaf(p)`. Expected the accepted target contains no old
`leaf` signature or stub. For the `caller` observation, the source copy
`caller::__proctor_source_call(p: *const i32)` is `pub(crate)` and calls
`crate::callee::__proctor_source_stub_leaf(p)`; that old-signature
`todo!()` stub is `pub(crate)` in the observation source and has the
correspondence ID of `callee::leaf`. Scratch source type checking and typed
observation extraction succeed. Neither helper appears in the accepted or
published target crate.

### A10-OBS-02B `cross_module_scc_source_copies_can_call_each_other`

Input: original source

```rust
mod a {
    pub(crate) unsafe fn even(p: *const i32, n: i32) -> i32 {
        if n == 0 { *p } else { crate::b::odd(p, n - 1) }
    }
}
mod b {
    pub(crate) unsafe fn odd(p: *const i32, n: i32) -> i32 {
        if n == 0 { *p } else { crate::a::even(p, n - 1) }
    }
}
```

Both target signatures take `&i32` and retain the same control/call
structure. Expected the one SCC addition inserts both target functions
and passes one `cargo build --lib`. Scratch source copies
`a::__proctor_source_even` and `b::__proctor_source_odd` are `pub(crate)`,
retain old `*const i32` signatures, and call each other through absolute
crate paths. No old-call stub is needed for these two current members.
Scratch type checking and typed dereference-region extraction succeed;
source copies and stubs are absent from the accepted and published crate.

### A10-OBS-03 `metadata_binds_candidate_pairs_and_scratch`

Input: valid sidecars, then mutate respectively one byte of candidate,
statement-pair JSON, or observation source before extraction. Expected each
tampered variant is rejected by the corresponding SHA-256 check; no typed
observation or final output is published. A duplicate, missing, or wrong
item/path correspondence is rejected before extraction as in the existing
protocol. No Python observation parser is introduced.

### A10-OBS-04 `accepted_only_and_empty_documents`

Input: one failed candidate, one accepted rule-complete SCC, and one
accepted transform SCC. Expected the failed candidate and rule-complete
SCC contribute no extraction; the accepted transform contributes exactly
one document; `merge-observations` receives that document once in schedule
order. With zero transform SCCs, merge receives an empty input list and
publishes `{"schema_version":1,"observations":[],"printf_observations":[]}`
modulo canonical whitespace.

### A10-OBS-05 `macro_hidden_old_call_is_unsupported`

Input: the prepared source has a module-level macro and two functions:

```rust
mod m {
    macro_rules! invoke { ($p:expr) => { leaf($p) } }
    pub(crate) unsafe fn leaf(p: *const i32) -> i32 { *p }
    pub(crate) unsafe fn caller(p: *const i32) -> i32 {
        invoke!(p)
    }
}
```

The generated skeleton labels the `invoke!(p)` statement as
`#[proctor(0)]`; the prepared input has no literal label. Accept
`leaf(p: &i32)` first,
then process `caller`; its source-copy macro invocation requires an
old-signature redirect hidden in the macro tokens. Expected Crat reports
the unsupported macro-token redirect during observation-source assembly,
before candidate installation or Cargo build. It does not report a
function-local-item error, publish an uncompilable scratch source, or
accept the `caller` SCC's observations or output.

## 8. Library API finalization and manifest

### A10-API-01 `final_wrapper_only_for_changed_api_signature`

Input: `api_functions = ["read"]`, source
`#[no_mangle] pub unsafe fn read(p: *const i32) -> i32 { *p }`, target
`read(p: &i32) -> i32 { *p }`. Expected all intermediate sources contain
only transformed `read(&i32)` and no wrapper. Final source keeps the same
implementation path, adds the existing collision-free same-module
`__proctor_wrapper_read` with the old `*const i32` signature and current
pointer-to-reference conversion, and moves the `no_mangle` export exactly
as current wrapper logic does. `proctor.toml` appends exactly
`{ wrapped = "read", wrapper = "__proctor_wrapper_read" }`.

### A10-API-02 `unchanged_api_signature_needs_no_wrapper`

Input: `api_functions = ["read"]`, source and target signatures both
`pub unsafe fn read(p: *const i32) -> i32`, but body changes. Expected final
source contains only `read`, retains existing export metadata on it, and
`wrappers = []`. There is no wrapper just because the function is public or
listed in the API manifest.

### A10-API-03 `changed_non_api_function_stays_unwrapped`

Input: `api_functions = ["entry"]`, where both `entry` and private `helper`
change their pointer signatures. Expected only `entry` gets a permanent
wrapper; `helper` remains only at its transformed path. The manifest has
one wrapper relation for `entry`, and target calls use new signatures.

### A10-API-04 `export_name_matches_api_even_when_rust_name_differs`

Input: `api_functions = ["public_read"]`, source
`#[export_name = "public_read"] pub unsafe fn internal_read(p: *const i32) -> i32 { *p }`,
with target `internal_read(p: &i32)`. Expected `internal_read` is selected,
receives its same-module existing-style wrapper at finalization, retains
the external `public_read` symbol on the wrapper, and records full
implementation/wrapper paths in `proctor.toml`.

### A10-API-05 `api_matching_uses_all_matching_rust_names`

Input: `api_functions = ["work"]`, with `a::work` and `b::work` both changed
from raw pointers to references. Expected both get distinct same-module
wrappers and two manifest relations ordered deterministically by source
item/path order. One arbitrary match or a cross-module wrapper is wrong.

### A10-API-06 `api_matching_combines_rust_and_export_names_without_duplicates`

Input: `api_functions = ["work", "public_work"]` and one function
`#[export_name = "public_work"] pub unsafe fn work(p: *const i32) -> i32`.
Expected one wrapper and one manifest relation for that function, even
though both criteria match. Duplicate API entries likewise do not create
duplicate wrappers.

### A10-API-06A `unmatched_api_name_fails_clearly`

Input: `api_functions = ["missing"]` with the only source function named
`present` and no `#[export_name = "missing"]`. Expected finalization fails
with an unresolved API identity diagnostic, no wrapper or final build is
performed, and no Rust project or artifacts are published. This differs
from multiple same-name Rust functions across modules: all of those are
matches as in API-05, subject to the final Cargo build's normal duplicate
external-symbol checks.

### A10-API-07 `wrapper_name_collision_uses_existing_allocator`

Input: API `f` needs a wrapper and a non-function item or existing sibling
uses `__proctor_wrapper_f` in the same value namespace. Expected the final
wrapper uses the existing next collision-free suffix (for example
`__proctor_wrapper_f_0` in the fixture), and the manifest's `wrapper` is
that actual full path. Internal module names do not force unrelated suffixes.

### A10-API-08 `wrapper_conversion_and_export_regressions_are_exact`

Input: API `f` has the following old/target signature pairs; compare the
generated wrapper AST with the existing `build_wrapper` result and require
the listed exact argument or result expression (use argument `p`, result
`__proctor_result`, and pointee `i32`):

| Old type | Target type | Expected wrapper conversion |
| --- | --- | --- |
| input `*const i32` | `&i32` | `&*(p as *const i32)` |
| input `*mut i32` | `&mut i32` | `&mut *(p as *mut i32)` |
| input `*const i32` | `Option<&i32>` | `(p as *const i32).as_ref()` |
| input `*mut i32` | `Option<&mut i32>` | `(p as *mut i32).as_mut()` |
| input `*const i32` | `&[i32]` | `if p.is_null() { &[] } else { std::slice::from_raw_parts(p as *const i32, 1_000_000) }` |
| input `*mut i32` | `&mut [i32]` | `if p.is_null() { &mut [] } else { std::slice::from_raw_parts_mut(p as *mut i32, 1_000_000) }` |
| input `*mut i32` | `Box<i32>` | `Box::from_raw(p as *mut i32)` |
| input `*mut i32` | `Option<Box<i32>>` | `if p.is_null() { None } else { Some(Box::from_raw(p as *mut i32)) }` |
| result `*const i32` | `&i32` | `__proctor_result as *const i32 as *const i32` |
| result `*mut i32` | `Option<&mut i32>` | `match __proctor_result { None => std::ptr::null_mut::<i32>() as *mut i32, Some(__proctor_result) => __proctor_result as *mut i32 as *mut i32 }` |
| result `*const i32` | `&[i32]` | `if __proctor_result.is_empty() { std::ptr::null::<i32>() as *const i32 } else { __proctor_result.as_ptr() as *const i32 }` |
| result `*mut i32` | `Box<i32>` | `Box::into_raw(__proctor_result) as *mut i32` |
| result `*mut i32` | `Option<Box<i32>>` | `match __proctor_result { None => std::ptr::null_mut::<i32>() as *mut i32, Some(__proctor_result) => Box::into_raw(__proctor_result) as *mut i32 }` |
| result `*mut i32` | `Box<[i32]>` | `if __proctor_result.is_empty() { drop(__proctor_result); std::ptr::null_mut::<i32>() as *mut i32 } else { Box::leak(__proctor_result).as_mut_ptr() as *mut i32 }` |

Also run one `#[no_mangle]` source, one
`#[export_name = "external_f"]` source, and one ordinary-metadata source.
Expected: the wrapper gets respectively `#[export_name = "f"]`, the
unchanged `#[export_name = "external_f"]`, or no export attribute; the
implementation retains only its non-export metadata. Conversion behavior,
including null handling and ownership assumptions, is unchanged.

### A10-API-09 `unsupported_api_conversion_fails_at_finalization`

Input: API function with old `*mut i32` input and target
`Box<[i32]>` input. Expected all SCC target additions can pass
`cargo build --lib`, but final wrapper creation reports the existing
`UnsupportedConversion` and stage fails without final output or a modified
published manifest. This contrasts with the non-API success in SCC-03.

### A10-API-10 `manifest_is_required_and_preserves_other_fields`

Input: valid `proctor.toml` with `target_kind = "library"`,
`target_name = "sample"`, `api_functions = ["read"]`, `wrappers = []`, plus
an additional unrelated TOML key; then omit the file or make
`api_functions` non-array. Expected valid input retains target identity,
API entries, and unrelated TOML information while appending only actual
final wrapper relations. Missing/malformed manifest fails before projection
or the initial Cargo build with a clear manifest error. No Python code
guesses API names from `pub` or `no_mangle` alone.

### A10-API-11 `preexisting_wrapper_relationship_is_rejected`

Input: otherwise valid library project with
`wrappers = [{ wrapped = "helper", wrapper = "helper_api" }]` in
`proctor.toml`. Expected stage reports this input unsupported before
project projection or the initial Cargo build, leaves the input project
unchanged, and publishes no output. Empty `wrappers = []` remains valid.

### A10-API-12 `library_main_cannot_be_an_api_entry`

Input: library manifest with `api_functions = ["main"]`, `wrappers = []`,
and source `pub fn main() {}`. Expected manifest validation rejects the
unsupported API entry before mutating the accepted project: no projection,
initial Cargo build, SCC additions, LLM calls, wrapper selection, final
build, or published project occur. The `main` function remains excluded
from SCC records; this case does not turn it into a transform target.

## 9. `main` and executable finalization

### A10-BIN-01 `main_boundary_is_absent_during_incremental_builds`

Input: executable manifest, original library with `main_0` and safe
forwarding `main`, plus explicit `[[bin]]` forwarding source. Expected `P0`
omits both source-defined functions, each SCC `cargo build --lib` sees only
accepted transformed functions, and no main wrapper/forwarding boundary is
installed before finalization. The explicit binary source is retained.

### A10-BIN-02 `two_argument_main_0_uses_existing_boundary_logic`

Input: `main_0(argc: i32, argv: *mut *mut i8) -> i32` whose target is
`main_0(argc: i32, argv: &mut [&mut [i8]]) -> i32`, plus original safe
`main`. Expected final source contains target `main_0` and the existing
mechanically rewritten safe `main` that constructs argument slices and
calls `main_0(argc, command_line_arg_slices.as_mut_slice())`; it contains no
ordinary `__proctor_wrapper_main_0`. Final `cargo build` builds the binary;
`proctor.toml` has no new wrapper relation.

### A10-BIN-03 `zero_argument_main_0_keeps_original_forwarder`

Input: `main_0() -> i32` and `pub fn main() { unsafe {
std::process::exit(main_0()) } }`. Expected finalization restores the
existing safe `main` with its original call shape, adds no compatibility
wrapper, and runs one final unrestricted `cargo build`. An unchanged
zero-argument `main_0` follows the same path.

### A10-BIN-04 `full_build_can_fail_after_lib_builds_pass`

Input: explicit binary source with a controlled compile error, while
projected library and all SCC candidates compile. Expected initial/SCC
`cargo build --lib` commands succeed; final `cargo build` fails; stage
reports failure and publishes no Rust project or report. There is no LLM
repair loop for a binary-only finalization failure.

### A10-MAIN-01 `ordinary_library_main_is_restored_unchanged`

Input: a library manifest with `api_functions = []` and source
`pub fn main() { println!("ready"); }` plus `pub unsafe fn helper() {}`.
Expected `P0` and the helper SCC candidate omit `main`; finalization adds
the original safe `main` with the same body and metadata. It is never
scheduled, sent to the LLM, observed, or entered in `proctor.toml` wrappers.
Final unrestricted `cargo build` succeeds. Target kind does not change this
rule.

### A10-MAIN-02 `library_two_argument_main_0_uses_same_fixed_rewrite`

Input: a library manifest with empty API list, original
`pub unsafe fn main_0(argc: i32, argv: *mut *mut i8) -> i32` and a safe
`pub fn main()` that forwards to it, with target
`main_0(argc: i32, argv: &mut [&mut [i8]]) -> i32`.
Expected `main_0` is added through its SCC; `main` is absent until
finalization, then receives the same existing argument-slice forwarding
body specified in BIN-02. No API wrapper or manifest relation is created;
final unrestricted library build succeeds.

### A10-MAIN-03 `library_zero_argument_main_0_keeps_original_main`

Input: a library manifest with `main_0() -> i32` and
`pub fn main() { unsafe { ::std::process::exit(main_0()) } }`.
Expected finalization restores that `main` body unchanged, adds no
`__proctor_wrapper_main_0`, and the full build succeeds. `main` is neither
an API function nor an SCC member.

### A10-MAIN-04 `unchanged_main_with_incompatible_callee_fails_final_build`

Input: library source
`pub unsafe fn helper(p: *const i32) -> i32 { *p }` and
`pub fn main() { let x = 1; unsafe { helper(&x); } }`, with target
`helper(p: &mut [i32]) -> i32 { p[0] }` that compiles alone and empty API
list.
Expected `P0` and the accepted helper candidate pass `cargo build --lib`;
finalization restores the original `main` call to `helper(&x)` verbatim,
with no helper wrapper. Final unrestricted `cargo build` fails on the
incompatible argument type, stage aborts, and no output is published. The
same rule applies to an executable manifest.

## 10. Protocol, publication, and regression

### A10-WIRE-01 `tool_commands_and_sidecars_are_lockstep`

Input: fake Crat and stage invocation with one SCC. Let `T` be the tool
binary, `A` the immutable analysis project, `C` the accepted partial
project, `M` the copied `proctor.toml`, `R` the version-1 replacement request,
and `I`, `X`, `S`, `O`, `D`, `F`, `FM` be distinct scratch files. Expected
commands, including argument order, are:

```text
T make-initial --output I A
T add-functions --request R --current-project C --output X --statement-pairs-output S --observation-source-output O --observation-metadata-output D A
T finalize-project --manifest M --current-project C --output F --manifest-output FM A
```

`A` is passed to compiler-resolved operations; `C` is passed where additive
insertion/finalization needs its accepted source. The `add-functions` request
remains replacement-request schema version 1, statement pairs remain version
1, and all four outputs are regular and pairwise distinct. `I`, `F`, and
`FM` cannot alias their inputs or each other. Finalization has no observation
output.

### A10-WIRE-01A `new_observation_metadata_is_closed_and_backward_readable`

Input: one accepted `leaf` with no previous callee, then a `caller` source
copy that needs an old-signature `leaf` stub. Expected first new metadata is
schema version 2 with `source_stubs = []`; second is version 2 with exactly
`[{"item_id": <leaf ID>, "path": "inner::__proctor_source_stub_leaf"}]`,
sorted by ID, and with
all existing candidate/pairs/source digests and correspondence fields.
The `path` is the actual collision-free crate-visible scratch stub path in
`O` (for this fixture, `inner::__proctor_source_stub_leaf`), not the
logical callee path.
Accepted and new correspondences have `wrapper_path: null`. Duplicate,
unsorted, unknown-item, or wrong-path stub entries are rejected before
extraction. If `O` contains an old-call stub but metadata omits its entry,
extraction fails as missing correspondence; if one stub identity maps to
two logical IDs, extraction fails as ambiguous correspondence. Existing
version-1 observation metadata remains accepted by
`extract-observations` for the old `replace` operation; unknown/newer
versions and extra fields are rejected.

### A10-WIRE-02 `missing_or_malformed_new_output_is_fatal`

Input: fake projection, addition, or finalization command returns zero but
does not write its declared regular output; repeat with a symlink where a
regular output is required. Expected stage rejects each before any build
that would consume it, removes temporary outputs, and does not publish a
partial project. A malformed replacement sidecar is rejected before
candidate installation as before.

### A10-PUB-01 `published_project_contains_only_final_target`

Input: successful two-SCC library run with API wrapper. Expected output
project contains the final transformed implementations and exactly the
permanent API wrapper; its `proctor.toml` records that wrapper. It contains
no original caller bodies, internal temporary wrappers, observation stubs,
source copies, scratch JSON, or Cargo `target/` directory. Input project
and optional rule-set file remain byte-for-byte unchanged.

### A10-PUB-02 `finalization_and_manifest_publication_are_atomic`

Input: inject a wrapper-generation failure, final-build failure, manifest
write failure, observation-merge failure, or final copy failure separately.
Expected every variant reports stage failure with no usable declared Rust
output; no final report, observations, or statistics artifact is partially
published. Earlier accepted SCCs may remain in stage work for diagnosis,
but the input project and rule set are unchanged.

### A10-PUB-03 `statistics_and_usage_account_for_new_builds`

Input: two SCCs, one rejected LLM candidate, one successful repair, then
successful finalization. Expected `cargo_builds` counts initial projection,
every installed candidate attempt, and final full build; validator-only
rejections do not increment it. `compilation_failures` counts only actual
failed builds; generation/repair counts and usage records follow existing
semantics. No test-execution metric is invented.

### A10-REG-01 `unchanged_stage_boundaries_stay_unchanged`

Input: stage with and without optional read-only rule set; bad Cargo
`[lib].path`; stale artifact files; output overlapping input. Expected
existing boundary validation, dependency normalization, preparation,
rule-set immutability, prompt/usage accounting, stale-artifact cleanup,
and failure envelope behavior remain as current tests specify. The new
projection does not change the standalone stage envelope or artifact kinds.

### A10-REG-02 `no_semantic_validation_is_claimed`

Input: fake candidate whose Rust compiles but whose body returns the wrong
value. Expected stage accepts it after compilation, publishes output and
observations, and does not invoke a test package or agent-based program
repair. A future semantic repair may invalidate observations from changed
functions, but this run discards none.

## 11. Optional real-toolchain scenarios and completion

### A10-E2E-01 `two_function_library_builds_incrementally`

Input: a small translated library with one changed callee signature, a
changed caller, and one named API entry; use deterministic rule-complete or
replay LLM responses. Expected observed Cargo sequence is initial `--lib`,
callee `--lib`, caller `--lib`, final unrestricted build; the output source
has both transformed functions plus only the final API wrapper; its manifest
records that wrapper; observations are extracted from accepted transform
labels.

### A10-E2E-02 `executable_final_boundary_builds`

Input: a translated executable with two-argument `main_0` and forwarding
binary. Expected all partial library builds succeed without a source-defined
`main`; finalization installs the existing argument-slice boundary and full
Cargo build succeeds. Do not use its runtime behavior as proof of semantic
equivalence in this amendment.

Completion requires the focused cases above, successful default unit/static
checks, no edits to older detailed plans or preexisting sections of
`prototype-plan.md`, and current-facing prototype
documentation updated when implementation lands. A test failure that reveals
a new unsupported boundary or requires changed wrapper behavior must be
reported for a plan decision rather than worked around silently.
