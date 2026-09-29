# Amendment 12 Test Plan: Compilable Function Placeholders

## 1. Purpose and authority

This is the acceptance plan for [amendment-12-plan.md](amendment-12-plan.md).
The initial partial library retains source imports and reexports and puts a
temporary declaration at each pending non-`main` function's original path.
Its exact shape is `[original visibility] fn [original name]() {}`: no original
parameters, return type, safety, ABI, attributes, or body. An SCC replaces
its pending declarations with accepted implementations at those same paths.
Tests must distinguish this temporary library source from the complete,
unchanged analysis source used for skeletons and source-copy observations.

The accepted boundary assumes retained non-function items do not need a
pending function's signature or value, apart from name resolution through
imports and reexports. The tests do not add a placeholder-body identity check,
a finalization completeness check, or an RPIT-specific check. Earlier plan
files remain historical. Planning labels below must not appear in source,
test names, fixtures, configuration, or diagnostics.

## 2. Test ownership and execution

Crat coverage belongs in the existing in-memory tests in
`proctor/stages/crat/crates/tools/src/item_replacer/tests.rs`. These tests
must not invoke the `crat-tool` CLI, change filesystem state, or use a Crat
root `tests/` directory. Compare parsed functions by full crate-relative
path, visibility, signature, body, and attributes; ignore only Rust printer
whitespace. Where stated, compile the in-memory source using the existing
test harness.

Python stage protocol, fake Cargo builds, rollback, and publication coverage
belongs in `proctor/tests/test_local_transformation.py`. The fake tool and
runner may write into `tmp_path`; default tests need no real toolchain,
network, or API key. An optional real-toolchain regression may live in the
existing `e2e` suite. There is no schema or manifest-model change, so no new
schema or manifest test is required.

After implementation, run from `proctor/stages/crat`:

```bash
cargo test -p tools item_replacer::tests
cargo test -p tools
cargo fmt
cargo clippy --workspace --all-targets
```

From `proctor`:

```bash
uv run pytest tests/test_local_transformation.py
uv run pytest
uv run ruff check .
uv run ruff format --check .
uv run mypy proctor
uv run proctor validate -c configs/c2rust_crat_local.toml
```

## 3. Shared fixture and comparison rules

Use this complete, normalized analysis source for the ordinary two-SCC
case. The private module and its imports remain in all partial sources:

```rust
mod inner {
    pub unsafe fn leaf(p: *mut i32) -> i32 { *p }
    pub unsafe fn caller(p: *mut i32) -> i32 { leaf(p) }
}
mod consumer { use crate::inner::{leaf as aliased_leaf, caller}; }
```

The initial source `P0` has `inner::leaf` and `inner::caller` as
`pub fn leaf() {}` and `pub fn caller() {}`. It retains the grouped aliased
import unchanged. `P1` has accepted `inner::leaf` and pending
`inner::caller`; `P2` has both accepted implementations. The first SCC is
`leaf`, then `caller`. Accepted implementations in examples below are
`pub unsafe fn leaf(p: Box<[i32]>) -> i32 { p[0] }` and
`pub unsafe fn caller(p: Box<[i32]>) -> i32 { leaf(p) }` (with normal source
labels supplied to Crat where required). Each source has one definition at
each original path. A successful fake Cargo build returns 0; a controlled
failure returns 101 and diagnostic text. Terminal-failure cases assert that
no later tool, LLM, or publication step occurs; repairable candidate failures
continue through the existing repair or rule-fallback path.

## 4. Update existing Crat tests

### A12-CRAT-01 Initial projection

Update `initial_projection_keeps_context_and_omits_all_free_functions`.
Input: its existing `c_int` import, constant, nested `inner::leaf`, nested
`inner::deep::main`, and root `main`, plus a `use crate::inner::leaf;` in a
separate module. Expected: all non-function items and imports remain
unchanged; `inner::leaf` is represented once at its original path by a
zero-argument, unit-returning, safe Rust declaration with original `pub`
visibility and empty body; its original signature/body is absent. Both
`main` functions remain absent. The initial library source compiles.

### A12-CRAT-02 Existing two-SCC replacement and observations

Update `additive_internal_signature_change_has_no_wrapper_and_uses_old_call_stub`.
Input: its existing `inner::leaf(p: *mut i32)` and
`inner::caller(p: *mut i32)` source, with `P0` from the new projection and
the existing accepted `Box<[i32]>` transformations. Expected: `P1` contains
exactly one `inner::leaf` with the accepted signature/body and one pending
`inner::caller`; `P1` compiles. `P2` contains exactly one accepted function
at each path, no pending declarations, and compiles. In the second SCC's
observation source, `caller` is replaced by its labeled accepted version;
the original `caller` source copy and the old-signature `leaf` call stub are
separate scratch items. That source compiles and typed extraction succeeds.
No source copy or stub enters `P2` or the finalized project.

### A12-CRAT-03 Recursive group and same names

Update `additive_cross_module_recursive_group_uses_visible_source_copies`,
`finalization_selects_all_same_named_functions_and_deduplicates_api_entries`,
and `additive_rejects_wrong_path_and_repeated_installation`. Input: the
existing `a::even(n: u32) -> bool` and `b::odd(n: u32) -> bool` functions,
each calling the other; separately, `a::work(p: *const i32) -> i32` and
`b::work(p: *const i32) -> i32` in distinct modules. Expected: `P0` has one pending declaration at
each original path; replacing the two members of one SCC yields one accepted
function at each path in both candidate and observation source, plus exactly
one separately named source copy per member in the latter. Same final names
in different modules remain path-distinct. A request for nonexistent `c::f`
still fails target resolution, and an already accepted path cannot be
requested again through the stage's accepted correspondence. No template
body comparison is introduced.

### A12-CRAT-04 Finalization retains accepted declarations

Update `finalization_keeps_changed_apis_at_original_paths` and the existing
`main` finalization tests. Input: `#[no_mangle] pub unsafe extern "C" fn
first(p: *const i32) -> i32`, private `middle(p: *const i32) -> i32`, and
`#[export_name = "public_last"] pub unsafe extern "C" fn
last(p: *const i32) -> i32`, accepted with reference signatures; also use
the existing ordinary and forwarding `main` fixtures. Expected: finalized output has exactly one function
at each original path, with the accepted signature/body, original export
attributes, and existing `main` behavior; no placeholder remains. This is a
positive-path assertion, not a new finalizer completeness precondition.

## 5. New Crat regression cases

### A12-CRAT-05 Single import across modules (B01 failure pattern)

Input: `mod defs { pub unsafe fn foo(p: i32) -> i32 { p } }` and
`mod imports { use crate::defs::foo; }`. Expected `P0` retains the exact
`use` and `pub fn foo() {}` in `defs`, with no `unsafe`, parameter, or return
type. It compiles as a library. After replacing `defs::foo`, the candidate
contains the accepted function once, retains the import, and compiles.

### A12-CRAT-06 Grouped imports, aliases, and reexports (P01 failure pattern)

Input: `mod defs { pub unsafe fn f(p: i32) -> i32 { p } pub unsafe fn g() {} }`,
`mod imported { use crate::defs::{f as renamed, g}; }`, and
`mod api { pub use crate::defs::{f as public_f, g}; }`. Expected `P0` retains
both import trees structurally, allowing Rust printer whitespace, and has
`pub fn f() {}` and `pub fn g() {}` in
`defs`. The initial library build succeeds. Replacing `f` leaves `g` pending;
replacing `g` retains both aliases and both reexports. No import is deleted
or reconstructed.

### A12-CRAT-07 Visibility, nested modules, and glob imports

Input: the complete normalized analysis source
`mod a { pub unsafe fn chosen() {} }`,
`mod b { unsafe fn chosen() {} pub(crate) mod deep { pub(crate) unsafe fn nested() {} pub(super) unsafe fn restricted() {} } }`,
and `mod consumer { use crate::a::*; use crate::b::*; use crate::b::deep::nested; }`.
Expected `P0` retains the three import statements and has `pub fn chosen()`
at `a::chosen`, private `fn chosen()` at `b::chosen`, and
`pub(crate) fn nested()` and `pub(super) fn restricted()` at their original
deep paths. All have empty bodies, and
the library compiles. Replacing `b::chosen` changes only that path; no
visibility is widened, so a glob cannot newly expose it to `consumer`.

### A12-CRAT-08 Distinguish pending and accepted paths through correspondence

Input: the shared fixture and first accepted SCC, with correspondence for
`inner::leaf` only. Expected the second add operation accepts `P1` even
though `P1` contains both function paths; it replaces `inner::caller` and
returns `accepted_correspondence = [inner::leaf]` and
`new_correspondence = [inner::caller]`. Their combination names both
accepted functions. The existing check that rejected
any occupied requested path is no longer applied to a pending declaration.
The existing `InvalidRequest` check rejects overlap between accepted
correspondence and requested paths; no check
compares a pending function body with a placeholder template.

### A12-CRAT-09 `main` remains excluded

Input: `pub fn main() {}` and `mod inner { pub fn main() {} }`, with no imports
of either function. Expected `P0` has no `main` declaration at either path;
finalization restores both original `main` items once and unchanged. In a
separate boundary fixture, `pub fn main() {}` plus
`mod consumer { use crate::main; }` retains the import but has no placeholder,
so its initial library compile fails with unresolved import. This remains
outside Amendment 12's supported import pattern.

### A12-CRAT-10 Function metadata and foreign declarations

Input: `#[no_mangle] pub unsafe extern "C" fn exported(p: *const i32)
-> i32 { *p }` and `extern "C" { fn foreign(p: *const i8) -> i32; }`.
Expected `P0` contains exactly `pub fn exported() {}` for the source-defined
function and retains the foreign block and prototype unchanged. It contains
no copied `no_mangle`, `unsafe`, or `extern "C"` on the placeholder. After
accepting `exported(p: &i32) -> i32`, the candidate has its original
`no_mangle`, visibility, safety, and ABI plus the accepted signature/body;
`foreign` remains a declaration, not an SCC placeholder.

### A12-CRAT-11 Raw identifier survives placeholder replacement

Input: normalized source `mod api { pub unsafe fn r#type(x: i32) -> i32 { x } }`
and `mod imported { use crate::api::r#type as chosen; }`. Expected `P0`
retains the import and has exactly `pub fn r#type() {}` at `api::r#type`;
it compiles. A replacement request for that full path produces exactly one
accepted `pub unsafe fn r#type(x: i32) -> i32 { x }` at the same path, keeps
the import, and compiles. Do not normalize the raw identifier into a
different source spelling.

## 6. Python stage transaction and publication

### A12-STAGE-01 Initial and incremental build sources

Update `test_additive_builds_keep_complete_analysis_and_grow_target_by_scc`.
Input: its existing `inner::leaf`/`inner::caller` fake-tool fixture, with
`P0`, `P1`, and `P2` changed to the source shapes in section 3. Expected:
the analysis file remains the complete original source throughout; fake
Cargo sees `P0`, `P1`, `P2`, and finalized `P2` in that order. Build modes
are `--lib`, `--lib`, `--lib`, then unrestricted. The stage reports success,
publishes `P2`, and counts four builds. Its first build no longer fails
because a retained import names an omitted function.

### A12-STAGE-02 Candidate build failure restores pending declaration

Update `test_failed_candidate_build_restores_then_repairs` and the builder
exception test. Input: `P1` from the shared fixture; the first `caller`
candidate has a controlled type error and the second is valid. Expected:
after the failed candidate build, current source is byte-identical to `P1`
with pending `caller`, accepted `leaf`, and its import. The repair installs
valid `P2`; only the accepted attempt contributes statement pairs and
observations. If the builder raises instead, the same rollback occurs and
the existing exception policy remains in force.

### A12-STAGE-03 True initial failure still aborts

Update `test_normalized_initial_build_failure_aborts_without_llm`. Input:
`P0` with `use crate::missing::foo;`, which has no matching source module,
and a fake initial Cargo result 101 with unresolved-import diagnostics.
Expected stage failure before any SCC replacement, LLM request, or final
publication. This case does not treat an import of a source-defined pending
function as an initial-build failure.

### A12-STAGE-04 Later failure does not publish placeholders

Retain `test_final_build_failure_restores_source_and_manifest_together`'s
existing opaque `"accepted partial\n"` fixture: a controlled final build
failure must restore that exact source and the original manifest, with no
published output. In `test_project_markdown_json_publish_as_one_cleanup_transaction`,
replace the opaque `"accepted"` source literal with
`"pub unsafe fn accepted(p: i32) -> i32 { p }\n"` and assert that exact
source at the destination on successful publication. Keep its controlled
publication failure and cleanup assertions. Together with `A12-STAGE-01`,
these positive paths verify that the stage publishes its accepted current
source and does not publish a pending declaration from that fixture.
`A12-CRAT-02` verifies at the source level that scratch observation copies
and old-call stubs do not enter the accepted or finalized project.

### A12-STAGE-05 Mechanical and rule-fallback paths replace placeholders

Update `test_mechanical_only_scc_skips_llm_validation_and_observation` and
`test_printf_rule_build_failure_uses_whole_scc_baseline_once`. In the first
test, set `P0` to `fn target() {}\n` and the fake candidate to
`unsafe fn target() { ::std::print!("fixed"); }\n`. Expected the candidate
replaces `target` once, builds, and needs no LLM or observation extraction.
In the second test, the existing `printf_member` and `ordinary_member` are
one two-member SCC. Set `P0` to
`fn printf_member() {}\nfn ordinary_member() {}\n`. Set the fake applied
candidate to
`unsafe fn printf_member() { rule_value(); }\nunsafe fn ordinary_member() {}\n`
and return Cargo 101. Assert that the next `add_functions` invocation sees
the exact `P0` bytes after rollback and requests both baseline views. Set
the fake baseline candidate to
`unsafe fn printf_member() { ::std::print!("fixed"); }\nunsafe fn ordinary_member() {}\n`
and return Cargo 0. The accepted candidate has one implementation at each
path, builds, and only the accepted baseline view
contributes reports and observations. No duplicate original-path function
appears in either candidate.

## 7. Optional real-toolchain regression and completion

One optional `e2e` fixture may exercise both observed import patterns with
a real Crat tool and Cargo: a single cross-module `use`, and grouped aliases
and reexports. Expected initial `cargo build --lib` succeeds, each accepted
SCC build succeeds, and the final project builds. This fixture need not
replay the large archived B01 and P01 run directories or call a model.

An optional preparation regression may use source with
`#[cfg(feature = "active")] fn choice() {}` and
`#[cfg(not(feature = "active"))] fn choice() {}` with the feature disabled.
After the existing expand/unexpand preparation, the analysis source and `P0`
should contain exactly one `choice`/placeholder path, respectively. This
checks the existing preparation boundary; it does not add `cfg` attributes to
the placeholder.

Complete this amendment when the updated and new focused tests pass, both
reported unresolved-import patterns are represented, and the Crat/Python
verification commands above pass. The tests establish name resolution,
replacement, rollback, and accepted-source publication; they do not claim
runtime equivalence or support for retained non-function items that require
pending function types or values.
