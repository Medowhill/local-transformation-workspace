# Phase 3 Test Plan: Item Replacement and Integration

## 1. Purpose

This document specifies the complete automated test suite planned for Phase 3
of the local-transformation prototype. It is the hand-over contract for:

- the Phase 2 lifetime-generic validator amendment;
- the Phase 1/2 amendment that rejects every function-local item;
- one-time target-safety normalization;
- the versioned in-memory replacement request;
- validated function replacement by full Rust path;
- recursive label removal and exact header composition;
- same-module compatibility wrappers;
- ABI and export-attribute movement;
- wrapper input and output conversions;
- HIR-resolved external call rewriting;
- executable `main_0` migration; and
- atomic, structured integration errors.

Phase 1 and the original Phase 2 implementation are complete. Do not modify
`phase-1-test-plan.md` or `phase-2-test-plan.md`. Section 4 below specifies the
required updates to existing Phase 1 and Phase 2 Rust tests and the additional
regressions. Those updates implement the amendments in `prototype-plan.md`
without rewriting either historical test plan.

`unsupported.md` remains the conceptual input contract. Tests explicitly
identified as component-level robustness or rejection cases do not expand that
contract.

The suite contains 48 named cases:

| Area | Cases |
| --- | ---: |
| Phase 1/2 amendments | 8 |
| Target-safety normalization | 2 |
| Replacement request and setup | 6 |
| Function replacement and header composition | 6 |
| Wrapper placement, visibility, ABI, and exports | 7 |
| Wrapper input conversions | 5 |
| Wrapper output conversions | 6 |
| Call rewriting and SCC boundaries | 5 |
| Executable migration and atomic integration | 3 |

Later bug fixes may add regressions, but Phase 3 is not complete until every
case in this document is implemented and passing.

## 2. Test execution and filesystem policy

All tests are ordinary Rust tests under `crates/tools/src/` and run with:

```bash
cd crat
cargo test --workspace
```

Put the Phase 1 local-item amendment case in `skeleton/tests.rs`. Put Phase 2
amendment cases in `validator/tests.rs`. Put every Phase 3 case in
`item_replacer/tests.rs`. Keep `lib.rs` limited to required crate-level
attributes and `extern crate` declarations, module declarations, and public
re-exports.

Safety-normalization tests call the parser-only in-memory API directly.
Request JSON tests call
`replacement_request_from_json(&str) -> Result<ReplacementRequest,
ReplacementError>` directly. Replacement tests compile the exact
current-source string with
`utils::compilation::run_compiler_on_str` and call:

```rust,ignore
replace_items(
    source: &str,
    request: &ReplacementRequest,
    tcx: TyCtxt<'_>,
) -> Result<String, ReplacementError>
```

inside the compiler callback. Where a case expects successful, compilable
output, return or store the owned output string from the first compiler
callback, allow that callback and its `TyCtxt` to end, and only then compile the
string with a separate second `run_compiler_on_str` invocation. Never nest the
second compiler run inside the first callback. This is still in-memory
processing and does not construct a Cargo project.

No Phase 3 test may:

- create, copy, read, or write a file or directory;
- invoke `crat-tool`, Cargo, or another subprocess;
- test Clap or orchestrator-owned project-copying code;
- read a checked-in fixture or snapshot;
- mutate an environment variable or other process-global setting; or
- modify any source, manifest, lockfile, or generated artifact.

The `normalize-safety` and `replace` CLI wiring that writes one `.rs` output
file is implemented but is not tested in Phase 3. Project copying, library
source overwrite, build, and promotion belong to the orchestrator and are
also outside this test suite.

## 3. Test helpers and comparison policy

Unless a case says otherwise, a replacement request has:

```text
schema_version = 1
items = [{ id: 7, path: "f", name: "f" }]
transformation = the exact Transformation Rust block
```

Every “Current compiler input,” “Parser input,” “Expected skeleton,” and
“Transformation” block is complete input. Ellipses are never implicit Rust.
When a case has multiple independent inputs, run each block as a separate
subcase.

Use AST/HIR-aware assertions for semantic structure and normalized
pretty-printed fragments for generated expressions. Do not require the
pretty-printer to preserve original whitespace. For every successful
replacement, assert:

- the requested implementation occurs exactly once at its original full path;
- every inserted wrapper occurs exactly once in the implementation's module;
- no `proctor` label remains in a replacement implementation or wrapper;
- unrelated items and functions remain structurally unchanged;
- each rewritten caller resolves syntactically to the documented absolute
  wrapper path; and
- re-parsing succeeds.

Where the expected output should compile, compile it in-memory as described in
Section 2. Cases involving deliberately conflicting source declarations,
unsupported source syntax, or a transformed body whose only purpose is
parser-level header testing need only assert the documented parser or
replacement result.

All failure cases assert one of the six coarse `ReplacementErrorKind` variants
from `prototype-plan.md` Section 16.2 plus item ID/path/name when available.
Messages identify the concrete observed construct for debugging. Do not test
fine-grained subcodes or impose a validator-style total precedence. Repeated
runs over the same input return the same coarse kind, item context, and
message. Because `replace_items` returns
`Result<String, ReplacementError>`, an error has no partial source output.

Use the coarse kinds consistently:

| Kind | Failures |
| --- | --- |
| `InvalidRequest` | JSON/schema/item metadata |
| `InvalidTransformation` | transformation parse, function-set, and unsupported returned-header failures |
| `TargetResolution` | missing/wrong/unsupported current targets and missing safety normalization |
| `UnsupportedConversion` | unsupported source/target wrapper conversion pairs |
| `UnsupportedCallRewrite` | a required call redirect inside a macro token input |
| `RewriteFailure` | unexpected source parse, AST/HIR mapping, executable migration, or emission failures |

## 4. Phase 1/2 amendments

### 4.1 Phase 2 lifetime-generic amendment

Update `validate_signature` so the complete supported lifetime-generic
declaration is checked before parameter count, names, types, and return type.
The declaration must contain exactly the target skeleton's named lifetime
parameters in the same order, with no bounds, type parameters, const
parameters, lifetime-parameter attributes, or syntactically present `where`
clause, including an empty one. A difference emits exactly one
`generic_parameter_mismatch` for that function and does not suppress
independent parameter or return-type errors.

Add `generic_parameter_mismatch` immediately before
`parameter_count_mismatch` in the stable Phase 2 signature-code ordering.
Update every affected asserted code sequence; there is no requirement to add
an otherwise unrelated exhaustive enumeration test for all error-code strings.

Extend the existing `invalid_expected_skeleton_is_setup_error` input table with
these two exact expected skeletons:

```rust
unsafe fn f<#[allow(unused)] 'a>(x: &'a i32) {
    #[proctor(0)]
    return;
}
```

```rust
unsafe fn f<'a>(x: &'a i32)
where
{
    #[proctor(0)]
    return;
}
```

Each remains a Phase 2 `invalid_expected_skeleton` setup error because Phase 1
never generates a lifetime-parameter attribute or any syntactic `where`
clause.

Update the existing
`explicit_lifetime_names_in_types_are_enforced` Rust test. Its exact skeleton
and transformation remain:

```rust
unsafe fn f<'a>(mut x: &'a i32, mut y: &'a i32) -> &'a i32 {
    #[proctor(0)]
    todo!()
}
```

```rust
unsafe fn f<'b>(x: &'b i32, y: &'b i32) -> &'b i32 {
    #[proctor(0)]
    x
}
```

Its expected code sequence changes from three entries to:

```text
generic_parameter_mismatch
parameter_type_mismatch
parameter_type_mismatch
return_type_mismatch
```

No other existing Phase 2 expected result changes unless it returned a
different generic declaration from its expected skeleton.

#### P3-P2-LT-01 `matching_lifetime_generic_declaration_is_valid`

Expected skeleton:

```rust
unsafe fn f<'input, 'output>(
    mut input: &'input i32,
    mut fallback: &'output i32,
) -> &'input i32 {
    #[proctor(0)]
    todo!()
}
```

Transformation:

```rust
pub extern "C" fn f<'input, 'output>(
    input: &'input i32,
    fallback: &'output i32,
) -> &'input i32 {
    #[proctor(0)]
    let _ = fallback;
    #[proctor(0)]
    input
}
```

The result is valid. Visibility, ABI, safety, and binding mutability remain
ignored; the exact lifetime declaration is accepted.

#### P3-P2-LT-02 `omitted_lifetime_declaration_is_rejected_even_when_types_match`

Expected skeleton:

```rust
unsafe fn f<'a>(mut input: &'a i32) -> &'a i32 {
    #[proctor(0)]
    todo!()
}
```

Transformation:

```rust
unsafe fn f(input: &'a i32) -> &'a i32 {
    #[proctor(0)]
    input
}
```

Return exactly `generic_parameter_mismatch`. The parser does not need to
resolve the undeclared lifetime. Parameter and return type syntax still match
structurally, so no type mismatch is added.

#### P3-P2-LT-03 `added_or_renamed_lifetime_is_rejected`

Run independently.

Expected skeleton:

```rust
unsafe fn f<'a>(mut input: &'a i32) -> &'a i32 {
    #[proctor(0)]
    todo!()
}
```

Transformation with an added lifetime:

```rust
unsafe fn f<'a, 'unused>(input: &'a i32) -> &'a i32 {
    #[proctor(0)]
    input
}
```

Transformation with a renamed declaration but unchanged type spelling:

```rust
unsafe fn f<'b>(input: &'a i32) -> &'a i32 {
    #[proctor(0)]
    input
}
```

Each subcase returns exactly `generic_parameter_mismatch`.

#### P3-P2-LT-04 `lifetime_parameter_order_is_exact`

Expected skeleton:

```rust
unsafe fn f<'a, 'b>() {
    #[proctor(0)]
    return;
}
```

Transformation:

```rust
unsafe fn f<'b, 'a>() {
    #[proctor(0)]
    return;
}
```

Return exactly `generic_parameter_mismatch`. This isolates declaration order
from type comparison because neither lifetime occurs in a type.

#### P3-P2-LT-05 `attributes_bounds_type_const_and_where_generics_are_rejected`

Use this expected skeleton for every independent subcase:

```rust
unsafe fn f<'a>(mut input: &'a i32) -> &'a i32 {
    #[proctor(0)]
    todo!()
}
```

Lifetime bound:

```rust
unsafe fn f<'a: 'static>(input: &'a i32) -> &'a i32 {
    #[proctor(0)]
    input
}
```

Lifetime-parameter attribute:

```rust
unsafe fn f<#[allow(unused)] 'a>(input: &'a i32) -> &'a i32 {
    #[proctor(0)]
    input
}
```

Type parameter:

```rust
unsafe fn f<'a, T>(input: &'a i32) -> &'a i32 {
    #[proctor(0)]
    input
}
```

Const parameter:

```rust
unsafe fn f<'a, const N: usize>(input: &'a i32) -> &'a i32 {
    #[proctor(0)]
    input
}
```

Where predicate:

```rust
unsafe fn f<'a>(input: &'a i32) -> &'a i32
where
    'a: 'static,
{
    #[proctor(0)]
    input
}
```

Syntactically present empty `where` clause:

```rust
unsafe fn f<'a>(input: &'a i32) -> &'a i32
where
{
    #[proctor(0)]
    input
}
```

Every subcase returns exactly one `generic_parameter_mismatch`. It remains a
result-validation error, not `invalid_expected_skeleton`.

#### P3-P2-LT-06 `generic_mismatch_aggregates_in_request_order`

Expected entry `(7, "first")`:

```rust
unsafe fn first<'a>(mut input: &'a i32) -> &'a i32 {
    #[proctor(0)]
    todo!()
}
```

Expected entry `(8, "second")`:

```rust
unsafe fn second<'x, 'y>(mut input: &'x i32, mut fallback: &'y i32) -> &'x i32 {
    #[proctor(0)]
    todo!()
}
```

Transformation:

```rust
unsafe fn second<'y, 'x>(input: &'x i32, fallback: &'y i32) -> &'x i32 {
    #[proctor(0)]
    let _ = fallback;
    #[proctor(0)]
    input
}

unsafe fn first(input: &'a i32) -> &'a i32 {
    #[proctor(0)]
    input
}
```

There are two item failures in request order, `first` then `second`, each
starting with exactly one `generic_parameter_mismatch`. Result function order
does not affect failure order. Repeated JSON serialization is byte-identical.

### 4.2 Complete function-local-item rejection amendment

This amendment supersedes the historical Phase 1/2 support for function-local
`const` and `static` items. Do not edit either historical test-plan document;
change the existing Rust implementation and tests.

Remove the Phase 1 test
`labels_item_statements_inside_function_bodies`. In
`all_nested_pattern_binding_kinds_become_mutable`, remove the local `const`
and `static` declarations and their initializer bindings while retaining all
other pattern coverage.

Remove the Phase 2 test
`nested_items_are_preserved_and_new_items_are_rejected` and every assertion
for `missing_existing_item`, `duplicate_existing_item`,
`existing_item_location_mismatch`, or
`existing_item_structure_mismatch`. Remove the local-item subcase from any
mixed diagnostic-aggregation test while preserving its unrelated binding,
label, control, temporary, unsafe-block, and attribute assertions. Retain
`unexpected_nested_item` for an item introduced by a returned transformation.

#### P3-P1-ITEM-01 `phase_1_rejects_local_const_and_static_recursively`

Run these exact source inputs independently through skeleton generation.

Direct local const:

```rust
pub unsafe fn f() -> i32 {
    const LOCAL: i32 = {
        let inner = 1;
        inner
    };
    LOCAL
}
```

Nested local static:

```rust
pub unsafe fn f(flag: bool) -> i32 {
    if flag {
        static mut STATE: i32 = {
            let inner = 1;
            inner
        };
        STATE
    } else {
        0
    }
}
```

Each returns `GenerationErrorKind::FunctionLocalItem`, with function path `f`
and a message identifying `const` or `static`, respectively. No `ItemRecord`
is returned. The rejected item receives no `#[proctor(...)]` label, and
skeleton generation does not descend into its initializer. Keep the existing
representative rejection coverage for all other local item kinds.

#### P3-P2-ITEM-01 `phase_2_rejects_every_local_item`

Extend `invalid_expected_skeleton_is_setup_error` with these independent exact
expected skeletons.

Local const:

```rust
unsafe fn f() {
    #[proctor(0)]
    const LOCAL: i32 = 1;
}
```

Nested local static:

```rust
unsafe fn f() {
    #[proctor(0)]
    {
        #[proctor(1)]
        static mut STATE: i32 = 1;
    }
}
```

Each returns `invalid_expected_skeleton`, identifies the enclosing function
and observed item kind, and states that every function-local item is
unsupported. This check is recursive through ordinary supported
statement-list structure.

Then use this valid expected skeleton:

```rust
unsafe fn f() {
    #[proctor(0)]
    return;
}
```

Run these returned transformations independently.

Introduced const:

```rust
unsafe fn f() {
    #[proctor(0)]
    const LOCAL: i32 = {
        let ordinary_name = 1;
        ordinary_name
    };
}
```

Introduced static:

```rust
unsafe fn f() {
    #[proctor(0)]
    static mut STATE: i32 = 1;
}
```

Each returns exactly `unexpected_nested_item` for `f`; it does not attempt
expected-item identity, location, duplication, or structural comparison and
does not descend into the rejected initializer to produce derivative label,
binding, temporary, attribute, or unsafe-block diagnostics.

### 4.3 Arity-only executable recognition update

Update the existing Phase 1 Rust test
`supported_main_0_forms_are_generated_with_the_fixed_argv_override`. Retain
its supported zero- and two-argument executable inputs, but remove any
assertion that recognition depends on normalized parameter or return-type
syntax. Add this exact component-level input:

```rust
unsafe fn main_0(first: usize, second: *const u8) -> bool {
    let _ = (first, second);
    false
}
```

Although this is not a supported executable signature, its name and arity are
the only recognition inputs. Its target signature therefore preserves
`first: usize` and `-> bool` while forcing only the second parameter to
`&mut [&mut [i8]]`. Do not inspect safety, parameter names or types, return
type, visibility, ABI, attributes, or body. The existing zero-argument subcase
continues to receive no override. The Phase 3 executable tests in Section 12
apply the same name-and-arity-only classification when deciding whether to
replace the co-located sibling `main`.

## 5. Target-safety normalization tests

### P3-NORM-01 `normalizes_every_non_main_free_function_and_is_idempotent`

Exact parser input:

```rust
#![allow(dead_code)]

#[inline]
pub extern "C" fn root(value: i32) -> i32 {
    value
}

pub unsafe fn already(value: i32) -> i32 {
    value + 1
}

#[no_mangle]
pub extern "C" fn exported(value: i32) -> i32 {
    value + 2
}

#[export_name = "renamed_alias"]
pub fn alias(value: i32) -> i32 {
    value + 3
}

pub fn r#type() -> i32 {
    4
}

pub fn main() {}

mod outer {
    pub(crate) fn nested(value: i32) -> i32 {
        value + 5
    }

    pub unsafe extern "C" fn already_unsafe(value: i32) -> i32 {
        value + 6
    }

    pub fn r#main() {}
}

extern "C" {
    fn foreign(value: *const i32) -> i32;
}
```

Call `normalize_target_safety` with only this source string. `root`,
`exported`, `alias`, `r#type`, and `outer::nested` become unsafe.
`already` and `outer::already_unsafe` contain exactly one `unsafe`.
Root `main` and `outer::r#main` remain safe because the comparison uses the
final identifier symbol. The foreign declaration remains unchanged. Every
attribute, visibility, ABI, parameter, return type, and body is structurally
unchanged.

Run normalization again on its output. The second result is structurally
identical and does not duplicate `unsafe`. This test supplies no skeleton JSON
or function-path list.

### P3-NORM-02 `whole_program_normalization_preserves_safe_main_and_compiles`

Exact parser and compiler input:

```rust
pub fn callee(value: i32) -> i32 {
    value + 1
}

pub fn caller(value: i32) -> i32 {
    callee(value)
}

unsafe fn main_0() -> core::ffi::c_int {
    caller(1)
}

pub fn main() {
    unsafe {
        ::std::process::exit(main_0() as i32)
    }
}
```

Call `normalize_target_safety` with only this source string. `callee`,
`caller`, and `main_0` are unsafe; `main` remains safe. The direct calls remain
`callee(value)`, `caller(1)`, and `main_0()`. Compile the complete normalized
output in a separate in-memory compiler invocation. This pins the
incremental-compilation invariant without target-path resolution.

## 6. Replacement request and setup tests

### P3-REQ-01 `versioned_request_json_round_trip_preserves_rust`

Exact current compiler input:

```rust
pub unsafe fn f(value: i32) -> i32 {
    value
}
```

Exact request JSON:

```json
{
  "schema_version": 1,
  "items": [
    {
      "id": 7,
      "path": "f",
      "name": "f"
    }
  ],
  "transformation": "unsafe fn f(value: i32) -> i32 {\n    #[proctor(0)]\n    value + 1\n}"
}
```

Deserialization and reserialization preserve the integer fields and embedded
Rust content. The typed request replaces `f` successfully.

### P3-REQ-02 `request_json_rejects_unknown_fields_and_non_u64_numbers`

Exact current compiler input:

```rust
pub unsafe fn f() {}
```

Run these exact JSON inputs independently:

```json
{"schema_version":1,"items":[{"id":7,"path":"f","name":"f"}],"transformation":"unsafe fn f() {}","extra":true}
```

```json
{"schema_version":1.0,"items":[{"id":7,"path":"f","name":"f"}],"transformation":"unsafe fn f() {}"}
```

```json
{"schema_version":1,"items":[{"id":-1,"path":"f","name":"f"}],"transformation":"unsafe fn f() {}"}
```

```json
{"schema_version":1,"items":[{"id":18446744073709551616,"path":"f","name":"f"}],"transformation":"unsafe fn f() {}"}
```

The first is an unknown-field error. The other three are invalid request JSON
number errors. None invokes replacement.

### P3-REQ-03 `unsupported_version_and_empty_items_are_rejected`

Exact current compiler input:

```rust
pub unsafe fn f() {}
```

Exact typed requests:

```text
ReplacementRequest { schema_version: 2, items: [{ id: 7, path: "f", name: "f" }], transformation: "unsafe fn f() {}" }
ReplacementRequest { schema_version: 1, items: [], transformation: "" }
```

The first reports unsupported schema version before parsing the
transformation. The second reports an empty item list.

### P3-REQ-04 `duplicate_ids_paths_and_names_are_rejected_deterministically`

Exact current compiler input:

```rust
pub unsafe fn f() {}
pub unsafe fn g() {}
```

Run independently:

```text
items = [{ id: 7, path: "f", name: "f" }, { id: 7, path: "g", name: "g" }]
items = [{ id: 7, path: "f", name: "f" }, { id: 8, path: "f", name: "g" }]
items = [{ id: 7, path: "f", name: "f" }, { id: 8, path: "g", name: "f" }]
```

Use this exact transformation in every subcase:

```rust
unsafe fn f() {}
unsafe fn g() {}
```

Errors are duplicate ID, duplicate path, and duplicate name respectively, in
that setup precedence. No source output is returned.

### P3-REQ-05 `path_name_disagreement_and_invalid_paths_are_rejected`

Exact current compiler input:

```rust
mod m {
    pub unsafe fn f() {}
}
```

Use this exact transformation:

```rust
unsafe fn f() {}
```

Run these item entries independently:

```text
{ id: 7, path: "m::f", name: "g" }
{ id: 7, path: "", name: "f" }
{ id: 7, path: "m::::f", name: "f" }
```

The first reports final-name disagreement. The other two report invalid full
paths. Request validation occurs before source mutation.

### P3-REQ-06 `transformation_must_be_exact_supported_requested_function_set`

Exact current compiler input:

```rust
pub unsafe fn f() {}
pub unsafe fn g() {}
```

Requested items:

```text
[(7, "f", "f"), (8, "g", "g")]
```

Run independently with these exact transformations:

Parse failure:

```rust
unsafe fn f( {
```

Missing function:

```rust
unsafe fn f() {}
```

Duplicate function:

```rust
unsafe fn f() {}
unsafe fn f() {}
unsafe fn g() {}
```

Unexpected function:

```rust
unsafe fn f() {}
unsafe fn g() {}
unsafe fn h() {}
```

Unexpected nonfunction item:

```rust
unsafe fn f() {}
unsafe fn g() {}
const EXTRA: i32 = 1;
```

Unsupported parameter count:

```rust
unsafe fn f(value: i32) {
    let _ = value;
}
unsafe fn g() {}
```

Unsupported async result:

```rust
async unsafe fn f() {}
unsafe fn g() {}
```

Unsupported variadic result:

```rust
unsafe extern "C" fn f(mut count: i32, mut args: ...) {
    let _ = count;
}
unsafe fn g() {}
```

Each returns `InvalidTransformation` with the concrete reason in its message
and no replacement output.
The last three defensively recheck assumptions guaranteed by the Phase 2
validator rather than attempting partial replacement.

Finally, run this exact current compiler input independently:

```rust
pub unsafe fn f(value: (i32, i32)) -> i32 {
    value.0 + value.1
}
```

with:

```rust
unsafe fn f((left, right): (i32, i32)) -> i32 {
    #[proctor(0)]
    left + right
}
```

Return the unsupported non-identifier parameter-pattern error and no output.

## 7. Function replacement and header-composition tests

### P3-REP-01 `replaces_body_and_recursively_removes_only_proctor_labels`

Exact current compiler input:

```rust
#![allow(dead_code)]

pub unsafe fn f(mut value: i32) -> i32 {
    value += 1;
    value
}

pub unsafe fn untouched() -> i32 {
    9
}
```

Transformation:

```rust
unsafe fn f(value: i32) -> i32 {
    #[proctor(0)]
    let result: i32 = if value > 0 {
        #[proctor(1)]
        value * 2
    } else {
        #[proctor(2)]
        {
            #[proctor(3)]
            0
        }
    };
    #[proctor(4)]
    result
}
```

The output implementation is `pub unsafe fn f(value: i32) -> i32` with the
transformation body and no `proctor` attributes at any nesting depth.
`untouched` is structurally unchanged. No wrapper is generated because
binding mutability is not a type change.

### P3-REP-02 `preserves_current_header_properties_and_ignores_llm_header`

Exact current compiler input:

```rust
#![allow(dead_code)]

#[inline(never)]
pub(crate) unsafe extern "C" fn f(mut value: i32) -> i32 {
    value
}
```

Transformation:

```rust
#[cold]
pub const extern "system" fn f(value: i32) -> i32 {
    #[proctor(0)]
    value + 1
}
```

The output remains `#[inline(never)] pub(crate) unsafe extern "C" fn f` and
does not contain `#[cold]`, `const`, or `extern "system"`. It uses the
transformation's parameter pattern/type, return type, and body. This proves the
returned `const` qualifier is ignored rather than copied; source `const fn`
remains unsupported. No wrapper is generated because the parameter and return
types are unchanged.

### P3-REP-03 `replaces_exact_nested_full_path_without_touching_same_name`

Exact current compiler input:

```rust
pub mod left {
    pub unsafe fn f(value: i32) -> i32 {
        value + 1
    }
}

pub mod right {
    pub unsafe fn f(value: i32) -> i32 {
        value + 2
    }
}
```

Request item:

```text
{ id: 7, path: "right::f", name: "f" }
```

Transformation:

```rust
unsafe fn f(value: i32) -> i32 {
    #[proctor(0)]
    value + 20
}
```

Only `right::f` receives `value + 20`; `left::f` retains `value + 1`.

Also run this raw-identifier replacement independently.

Exact current compiler input:

```rust
pub mod r#type {
    pub unsafe fn r#match(value: *const i32) -> i32 {
        *value
    }

    pub unsafe fn caller(value: *const i32) -> i32 {
        r#match(value)
    }
}
```

Request item:

```text
{ id: 7, path: "r#type::r#match", name: "r#match" }
```

Transformation:

```rust
unsafe fn r#match(value: &i32) -> i32 {
    #[proctor(0)]
    *value + 1
}
```

The request path and name retain Phase 1's exact raw spelling. The
implementation remains `r#type::r#match`; the wrapper uses identifier symbols
to form `__proctor_wrapper_match`, and the caller becomes
`crate::r#type::__proctor_wrapper_match(value)`.

### P3-REP-04 `multiple_functions_match_by_name_and_replace_in_request_order`

Exact current compiler input:

```rust
pub unsafe fn first(value: i32) -> i32 {
    value
}

pub unsafe fn second(value: i32) -> i32 {
    value
}
```

Request items:

```text
[(7, "first", "first"), (8, "second", "second")]
```

Transformation, deliberately reversed:

```rust
unsafe fn second(value: i32) -> i32 {
    #[proctor(0)]
    value + 2
}

unsafe fn first(value: i32) -> i32 {
    #[proctor(0)]
    value + 1
}
```

Both original item positions remain stable, each receives its name-matched
body, and no wrapper is generated.

### P3-REP-05 `copies_validated_lifetime_generics_parameters_and_return`

Exact current compiler input:

```rust
pub unsafe fn choose(
    first: *const i32,
    second: *const i32,
    take_first: bool,
) -> *const i32 {
    if take_first { first } else { second }
}

pub unsafe fn caller(
    first: *const i32,
    second: *const i32,
) -> *const i32 {
    choose(first, second, true)
}
```

Transformation:

```rust
unsafe fn choose<'a, 'b>(
    first: &'a i32,
    second: &'b i32,
    take_first: bool,
) -> &'a i32 {
    #[proctor(0)]
    if take_first {
        first
    } else {
        let _ = second;
        first
    }
}
```

The implementation declares exactly `<'a, 'b>` and uses the exact transformed
parameter and return types. A same-module raw-signature wrapper is generated,
and `caller` is redirected to it. The test proves replacement does not try to
reconstruct generated lifetimes independently of the validated snippet.

### P3-REP-06 `source_target_resolution_and_normalized_safety_fail_atomically`

Run independently.

Missing path source:

```rust
pub unsafe fn f() {}
```

Request item and transformation:

```text
{ id: 7, path: "missing", name: "missing" }
```

```rust
unsafe fn missing() {}
```

Not-normalized source:

```rust
pub fn f() {}
```

Request item and transformation:

```text
{ id: 7, path: "f", name: "f" }
```

```rust
unsafe fn f() {}
```

The first reports a missing current target with item identity. The second
reports that the current target must already be unsafe. Neither returns
partial source.

## 8. Wrapper placement, visibility, ABI, and export tests

### P3-WRAP-01 `private_wrapper_is_a_same_module_sibling`

Exact current compiler input:

```rust
mod m {
    unsafe fn f(f: *const i32) -> i32 {
        *f
    }

    pub unsafe fn caller(value: *const i32) -> i32 {
        f(value)
    }
}
```

Request item:

```text
{ id: 7, path: "m::f", name: "f" }
```

Transformation:

```rust
unsafe fn f(f: &i32) -> i32 {
    #[proctor(0)]
    *f
}
```

The implementation remains private at `m::f`. A private sibling named from
`__proctor_wrapper_f` is inserted in `m`, not in a generated wrapper module.
Its source-signature parameter is also named `f`, so the wrapper must delegate
with the absolute `crate::m::f(...)` path rather than the shadowed unqualified
name. `m::caller` calls `crate::m::__proctor_wrapper_f(value)`. Both functions
are unsafe and the output compiles.

### P3-WRAP-02 `wrapper_preserves_restricted_visibility_in_nested_module`

Exact current compiler input:

```rust
mod outer {
    pub(super) unsafe fn f(value: *mut i32) -> i32 {
        *value
    }
}

pub unsafe fn caller(value: *mut i32) -> i32 {
    outer::f(value)
}
```

Request item:

```text
{ id: 7, path: "outer::f", name: "f" }
```

Transformation:

```rust
unsafe fn f(value: &mut i32) -> i32 {
    #[proctor(0)]
    *value += 1;
    #[proctor(1)]
    *value
}
```

Both `outer::f` and its sibling wrapper are exactly `pub(super)`, not `pub` or
`pub(crate)`. `caller` uses
`crate::outer::__proctor_wrapper_f(value)`.

### P3-WRAP-03 `wrapper_name_collision_is_resolved_deterministically`

Exact current compiler input:

```rust
mod m {
    pub unsafe fn __proctor_wrapper_f(value: *const i32) -> i32 {
        *value + 10
    }

    pub unsafe fn __proctor_wrapper_f_0(value: *const i32) -> i32 {
        *value + 20
    }

    pub unsafe fn f(value: *const i32) -> i32 {
        *value
    }

    pub unsafe fn caller(value: *const i32) -> i32 {
        f(value)
    }
}
```

Request item and transformation:

```text
{ id: 7, path: "m::f", name: "f" }
```

```rust
unsafe fn f(value: &i32) -> i32 {
    #[proctor(0)]
    *value
}
```

The two existing functions remain unchanged. The generated wrapper is exactly
`__proctor_wrapper_f_1`, and `caller` uses exactly that identifier. Two runs
produce byte-identical output.

Also run this multi-allocation collision independently:

```rust
mod m {
    pub unsafe fn __proctor_wrapper_f(value: *const i32) -> i32 {
        *value + 10
    }

    pub unsafe fn f(value: *const i32) -> i32 {
        *value
    }

    pub unsafe fn f_0(value: *const i32) -> i32 {
        *value
    }

    pub unsafe fn caller(value: *const i32) -> i32 {
        f(value) + f_0(value)
    }
}
```

Request both `(7, "m::f", "f")` and `(8, "m::f_0", "f_0")` with:

```rust
unsafe fn f(value: &i32) -> i32 {
    #[proctor(0)]
    *value
}

unsafe fn f_0(value: &i32) -> i32 {
    #[proctor(0)]
    *value
}
```

Allocate both names against the original module and against names already
reserved in request order for this transaction. The wrapper for `f` is exactly
`__proctor_wrapper_f_0`; that reservation occupies the base name for `f_0`, so
the wrapper for `f_0` is exactly `__proctor_wrapper_f_0_0`. Each caller
occurrence uses the wrapper allocated to its resolved callee.

Finally, prove that occupancy is conservative across Rust namespaces:

```rust
mod m {
    pub type __proctor_wrapper_g = i32;

    pub unsafe fn g(value: *const i32) -> i32 {
        *value
    }

    pub unsafe fn caller(value: *const i32) -> i32 {
        g(value)
    }
}
```

Request:

```text
{ id: 7, path: "m::g", name: "g" }
```

Transformation:

```rust
unsafe fn g(value: &i32) -> i32 {
    #[proctor(0)]
    *value
}
```

Even though the type alias does not occupy the value namespace, the wrapper is
exactly `__proctor_wrapper_g_0` because every sibling item name is treated as
occupied.

### P3-WRAP-04 `no_mangle_moves_to_wrapper_as_original_export_name`

Exact current compiler input:

```rust
#[no_mangle]
pub unsafe extern "C" fn exported(value: *const i32) -> i32 {
    *value
}
```

Transformation:

```rust
unsafe fn exported(value: &i32) -> i32 {
    #[proctor(0)]
    *value
}
```

The transformed `exported` implementation has Rust ABI and neither
`no_mangle` nor `export_name`. Its same-module wrapper is
`pub unsafe extern "C" fn __proctor_wrapper_exported` and carries exactly:

```rust
#[export_name = "exported"]
```

It does not carry `#[no_mangle]`, because that would export the wrapper's Rust
identifier instead of the original symbol.

Also run this unchanged-signature source independently:

```rust
#[no_mangle]
pub unsafe extern "C" fn exported(value: *const i32) -> i32 {
    *value
}
```

with:

```rust
unsafe fn exported(value: *const i32) -> i32 {
    #[proctor(0)]
    *value + 1
}
```

No wrapper is generated; the implementation retains both `#[no_mangle]` and
`extern "C"`.

### P3-WRAP-05 `explicit_export_name_moves_exactly_to_wrapper`

Exact current compiler input:

```rust
#[export_name = "c_api_entry_v1"]
pub unsafe extern "C" fn internal_name(value: *mut i32) -> i32 {
    *value
}
```

Transformation:

```rust
unsafe fn internal_name(value: &mut i32) -> i32 {
    #[proctor(0)]
    *value += 1;
    #[proctor(1)]
    *value
}
```

The exact `#[export_name = "c_api_entry_v1"]` attribute and `extern "C"` ABI
move to the wrapper. Neither remains on the implementation. The wrapper's
Rust identifier does not affect the external symbol.

Also run an unchanged-signature transformation:

```rust
unsafe fn internal_name(value: *mut i32) -> i32 {
    #[proctor(0)]
    *value + 1
}
```

against the same current compiler input. No wrapper is generated; the
implementation retains the exact export name and ABI.

### P3-WRAP-06 `explicit_abi_moves_even_without_export_attribute`

Exact current compiler input:

```rust
pub(crate) unsafe extern "C" fn f(value: *const i32) -> i32 {
    *value
}
```

Transformation:

```rust
unsafe fn f(value: &i32) -> i32 {
    #[proctor(0)]
    *value
}
```

The implementation becomes `pub(crate) unsafe fn f`. The wrapper is
`pub(crate) unsafe extern "C" fn __proctor_wrapper_f`. No export attribute is
invented.

### P3-WRAP-07 `nonexport_attributes_stay_only_on_implementation`

Exact current compiler input:

```rust
#![allow(dead_code)]

#[inline(never)]
#[cold]
pub unsafe fn f(value: *const i32) -> i32 {
    *value
}
```

Transformation:

```rust
#[allow(unused_variables)]
unsafe fn f(value: &i32) -> i32 {
    #[proctor(0)]
    *value
}
```

The implementation retains exactly `#[inline(never)]` and `#[cold]`. The
wrapper receives neither. The transformation's
`#[allow(unused_variables)]` is ignored. Visibility remains `pub` on both.

## 9. Wrapper input-conversion tests

### P3-IN-01 `raw_inputs_convert_to_shared_and_mutable_references_unchecked`

Exact current compiler input:

```rust
pub unsafe fn combine(left: *const i32, right: *mut i32) -> i32 {
    *right += *left;
    *right
}

pub unsafe fn caller(left: *const i32, right: *mut i32) -> i32 {
    combine(left, right)
}
```

Transformation:

```rust
unsafe fn combine(left: &i32, right: &mut i32) -> i32 {
    #[proctor(0)]
    *right += *left;
    #[proctor(1)]
    *right
}
```

The wrapper call arguments contain the exact conversions:

```rust
&*(left as *const i32)
&mut *(right as *mut i32)
```

There is no null check, `Option`, or explicit unsafe block in the already
unsafe wrapper.

### P3-IN-02 `raw_inputs_convert_to_optional_references_by_nullity`

Exact current compiler input:

```rust
pub unsafe fn choose(left: *const i32, right: *mut i32) -> i32 {
    if left.is_null() {
        0
    } else if right.is_null() {
        *left
    } else {
        *left + *right
    }
}

pub unsafe fn caller(left: *const i32, right: *mut i32) -> i32 {
    choose(left, right)
}
```

Transformation:

```rust
unsafe fn choose(left: Option<&i32>, right: Option<&mut i32>) -> i32 {
    #[proctor(0)]
    left.copied().unwrap_or(0) + right.map(|value| *value).unwrap_or(0)
}
```

The wrapper passes:

```rust
(left as *const i32).as_ref()
(right as *mut i32).as_mut()
```

and does not manufacture nonoptional references.

### P3-IN-03 `raw_inputs_convert_to_slices_with_null_empty_and_fixed_bound`

Exact current compiler input:

```rust
pub unsafe fn sum(first: *const i32, second: *mut i32) -> i32 {
    let left = if first.is_null() { 0 } else { *first };
    let right = if second.is_null() { 0 } else { *second };
    left + right
}

pub unsafe fn caller(first: *const i32, second: *mut i32) -> i32 {
    sum(first, second)
}
```

Transformation:

```rust
unsafe fn sum(first: &[i32], second: &mut [i32]) -> i32 {
    #[proctor(0)]
    first.first().copied().unwrap_or(0) + second.first().copied().unwrap_or(0)
}
```

The first wrapper argument is an `if first.is_null()` whose null branch is
`&[]` and whose other branch calls
`std::slice::from_raw_parts(first as *const i32, 1_000_000)`. The second is
the mutable analogue using `&mut []` and `from_raw_parts_mut`. Assert the
literal is exactly `1_000_000`.

### P3-IN-04 `raw_inputs_convert_to_box_and_optional_box`

Exact current compiler input:

```rust
pub unsafe fn consume(owned: *mut i32, optional: *mut i32) -> i32 {
    let first = *owned;
    let second = if optional.is_null() { 0 } else { *optional };
    first + second
}

pub unsafe fn caller(owned: *mut i32, optional: *mut i32) -> i32 {
    consume(owned, optional)
}
```

Transformation:

```rust
unsafe fn consume(owned: Box<i32>, optional: Option<Box<i32>>) -> i32 {
    #[proctor(0)]
    *owned + optional.map(|value| *value).unwrap_or(0)
}
```

The first argument is exactly `Box::from_raw(owned as *mut i32)` with no null
check. The optional argument checks `optional.is_null()` and otherwise uses
`Some(Box::from_raw(optional as *mut i32))`.

### P3-IN-05 `raw_cast_passthrough_and_unsupported_input_pairs`

Successful exact current compiler input:

```rust
pub unsafe fn f(pointer: *mut i32, count: usize) -> usize {
    if pointer.is_null() { 0 } else { count }
}

pub unsafe fn caller(pointer: *mut i32, count: usize) -> usize {
    f(pointer, count)
}
```

Successful transformation:

```rust
unsafe fn f(pointer: *const i32, count: usize) -> usize {
    #[proctor(0)]
    if pointer.is_null() { 0 } else { count }
}
```

The wrapper casts `pointer` to `*const i32` and passes `count` unchanged.

Then run these failure transformations independently against:

```rust
pub unsafe fn f(pointer: *mut i32) {}
```

Boxed slice:

```rust
unsafe fn f(pointer: Box<[i32]>) {
    #[proctor(0)]
    drop(pointer);
}
```

Optional boxed slice:

```rust
unsafe fn f(pointer: Option<Box<[i32]>>) {
    #[proctor(0)]
    drop(pointer);
}
```

Incompatible nonpointer:

```rust
unsafe fn f(pointer: usize) {
    #[proctor(0)]
    let _ = pointer;
}
```

The first two report unsupported boxed-slice input conversion. The third
reports an unsupported source/target pair. None returns output.

## 10. Wrapper output-conversion tests

### P3-OUT-01 `reference_outputs_cast_to_exact_raw_pointer_type`

Exact current compiler input:

```rust
pub unsafe fn identity(value: *mut i32) -> *mut i32 {
    value
}

pub unsafe fn caller(value: *mut i32) -> *mut i32 {
    identity(value)
}
```

Transformation:

```rust
unsafe fn identity<'a>(value: &'a mut i32) -> &'a mut i32 {
    #[proctor(0)]
    value
}
```

The wrapper stores the implementation result once, casts it to exact
`*mut i32`, and returns it. The implementation call occurs exactly once in the
wrapper AST.

Run these cross-mutability forms independently.

Shared reference returned to mutable raw source:

```rust
pub unsafe fn identity(value: *mut i32) -> *mut i32 {
    value
}
```

```rust
unsafe fn identity<'a>(value: &'a i32) -> &'a i32 {
    #[proctor(0)]
    value
}
```

The wrapper converts the result as `value as *const i32 as *mut i32`; it never
attempts the invalid direct cast `&i32 as *mut i32`.

Mutable reference returned to const raw source:

```rust
pub unsafe fn identity(value: *const i32) -> *const i32 {
    value
}
```

```rust
unsafe fn identity<'a>(value: &'a mut i32) -> &'a mut i32 {
    #[proctor(0)]
    value
}
```

The wrapper first obtains `*mut i32` from the result and then casts that raw
pointer to exact `*const i32`. Both outputs compile in separate in-memory
compiler runs.

### P3-OUT-02 `optional_reference_outputs_map_none_to_typed_null`

Run the shared and mutable cases independently.

Shared current compiler input:

```rust
pub unsafe fn maybe(value: *const i32, present: bool) -> *const i32 {
    if present { value } else { core::ptr::null() }
}
```

Shared transformation:

```rust
unsafe fn maybe<'a>(value: &'a i32, present: bool) -> Option<&'a i32> {
    #[proctor(0)]
    if present { Some(value) } else { None }
}
```

Mutable current compiler input:

```rust
pub unsafe fn maybe(value: *mut i32, present: bool) -> *mut i32 {
    if present { value } else { core::ptr::null_mut() }
}
```

Mutable transformation:

```rust
unsafe fn maybe<'a>(value: &'a mut i32, present: bool) -> Option<&'a mut i32> {
    #[proctor(0)]
    if present { Some(value) } else { None }
}
```

The shared wrapper maps `None` to `std::ptr::null()` cast to `*const i32`; the
mutable wrapper uses `std::ptr::null_mut()` cast to `*mut i32`. `Some`
casts the contained reference.

Also run these exact cross-mutability pairs:

```rust
pub unsafe fn maybe(value: *mut i32, present: bool) -> *mut i32 {
    if present { value } else { core::ptr::null_mut() }
}
```

```rust
unsafe fn maybe<'a>(value: &'a i32, present: bool) -> Option<&'a i32> {
    #[proctor(0)]
    if present { Some(value) } else { None }
}
```

and:

```rust
pub unsafe fn maybe(value: *const i32, present: bool) -> *const i32 {
    if present { value } else { core::ptr::null() }
}
```

```rust
unsafe fn maybe<'a>(value: &'a mut i32, present: bool) -> Option<&'a mut i32> {
    #[proctor(0)]
    if present { Some(value) } else { None }
}
```

The first `None` uses `null_mut` and converts `Some(&T)` through `*const T`
before `*mut T`; the second `None` uses `null` and converts `Some(&mut T)`
through `*mut T` before `*const T`.

### P3-OUT-03 `slice_outputs_map_empty_to_null_and_nonempty_to_data_pointer`

Run shared and mutable independently.

Shared current compiler input:

```rust
pub unsafe fn prefix(value: *const i32) -> *const i32 {
    value
}
```

Shared transformation:

```rust
unsafe fn prefix<'a>(value: &'a [i32]) -> &'a [i32] {
    #[proctor(0)]
    if value.is_empty() { &value[..0] } else { value }
}
```

Mutable current compiler input:

```rust
pub unsafe fn prefix(value: *mut i32) -> *mut i32 {
    value
}
```

Mutable transformation:

```rust
unsafe fn prefix<'a>(value: &'a mut [i32]) -> &'a mut [i32] {
    #[proctor(0)]
    value
}
```

The shared output checks `is_empty()`, returns typed immutable null when
empty, and otherwise casts `as_ptr()`. The mutable output uses typed mutable
null and `as_mut_ptr()`. The input side simultaneously uses the fixed
slice-bound conversion.

Repeat the shared transformation against:

```rust
pub unsafe fn prefix(value: *mut i32) -> *mut i32 {
    value
}
```

and the mutable transformation against:

```rust
pub unsafe fn prefix(value: *const i32) -> *const i32 {
    value
}
```

The shared/`*mut` empty branch uses `null_mut`, while its nonempty branch casts
the `*const i32` from `as_ptr()` to `*mut i32`. The mutable/`*const` empty
branch uses `null`, while its nonempty branch casts the `*mut i32` from
`as_mut_ptr()` to `*const i32`.

### P3-OUT-04 `box_and_optional_box_outputs_use_into_raw`

Run independently.

Box current compiler input:

```rust
pub unsafe fn make() -> *mut i32 {
    Box::into_raw(Box::new(1))
}
```

Box transformation:

```rust
unsafe fn make() -> Box<i32> {
    #[proctor(0)]
    Box::new(2)
}
```

Optional box current compiler input:

```rust
pub unsafe fn make(present: bool) -> *mut i32 {
    if present {
        Box::into_raw(Box::new(1))
    } else {
        core::ptr::null_mut()
    }
}
```

Optional box transformation:

```rust
unsafe fn make(present: bool) -> Option<Box<i32>> {
    #[proctor(0)]
    if present { Some(Box::new(2)) } else { None }
}
```

The first wrapper returns `Box::into_raw(value)` cast to `*mut i32`. The
second maps `None` to typed mutable null and `Some(value)` through
`Box::into_raw`.

Repeat the two transformations against these exact const-pointer sources:

```rust
pub unsafe fn make() -> *const i32 {
    Box::into_raw(Box::new(1)) as *const i32
}
```

```rust
pub unsafe fn make(present: bool) -> *const i32 {
    if present {
        Box::into_raw(Box::new(1)) as *const i32
    } else {
        core::ptr::null()
    }
}
```

The owning pointer first comes from `Box::into_raw` as `*mut i32` and is then
cast to exact `*const i32`. The optional `None` branch uses `null`, not
`null_mut`.

### P3-OUT-05 `boxed_slice_outputs_drop_empty_and_leak_nonempty`

Run independently.

Boxed-slice current compiler input:

```rust
pub unsafe fn make(empty: bool) -> *mut i32 {
    if empty { core::ptr::null_mut() } else { core::ptr::null_mut() }
}
```

Boxed-slice transformation:

```rust
unsafe fn make(empty: bool) -> Box<[i32]> {
    #[proctor(0)]
    if empty {
        Vec::<i32>::new().into_boxed_slice()
    } else {
        vec![1, 2].into_boxed_slice()
    }
}
```

Optional boxed-slice current compiler input:

```rust
pub unsafe fn make(kind: i32) -> *mut i32 {
    let _ = kind;
    core::ptr::null_mut()
}
```

Optional boxed-slice transformation:

```rust
unsafe fn make(kind: i32) -> Option<Box<[i32]>> {
    #[proctor(0)]
    match kind {
        0 => None,
        1 => Some(Vec::<i32>::new().into_boxed_slice()),
        _ => Some(vec![1, 2].into_boxed_slice()),
    }
}
```

For `Box<[i32]>`, the wrapper tests emptiness: it drops the empty box and
returns typed mutable null, or returns
`Box::leak(value).as_mut_ptr()` cast to `*mut i32`. For the optional form,
`None` and `Some(empty)` return null, the empty box is dropped, and only
`Some(nonempty)` is leaked.

Repeat both transformations against these exact const-pointer sources:

```rust
pub unsafe fn make(empty: bool) -> *const i32 {
    let _ = empty;
    core::ptr::null()
}
```

```rust
pub unsafe fn make(kind: i32) -> *const i32 {
    let _ = kind;
    core::ptr::null()
}
```

Every None/empty branch uses `null`; every nonempty branch obtains `*mut i32`
from `Box::leak(...).as_mut_ptr()` before casting to exact `*const i32`.

### P3-OUT-06 `raw_nonpointer_unit_and_single_evaluation_outputs`

Run independently.

Raw cast current compiler input:

```rust
pub unsafe fn raw(value: *mut i32) -> *mut i32 {
    value
}
```

Raw cast transformation:

```rust
unsafe fn raw(value: *const i32) -> *const i32 {
    #[proctor(0)]
    value
}
```

Nonpointer current compiler input:

```rust
pub unsafe fn count(value: i32) -> i32 {
    value
}
```

Nonpointer transformation:

```rust
unsafe fn count(value: i32) -> i32 {
    #[proctor(0)]
    value + 1
}
```

Unit current compiler input:

```rust
pub unsafe fn touch(value: *const i32) {
    let _ = *value;
}
```

Unit transformation:

```rust
unsafe fn touch(value: &i32) {
    #[proctor(0)]
    let _ = *value;
}
```

The raw wrapper casts to and from the exact pointer types. `count` gets no
wrapper. `touch` gets an input-converting wrapper with no artificial return
temporary. Across all wrapper-return cases P3-OUT-01 through P3-OUT-05, assert
the implementation call occurs exactly once.

## 11. Call-rewriting and SCC-boundary tests

### P3-CALL-01 `aliases_multiple_calls_and_nested_expressions_rewrite_by_resolution`

Exact current compiler input:

```rust
mod m {
    pub(crate) unsafe fn f(value: *const i32) -> i32 {
        *value
    }
}

use m::f as alias;

pub unsafe fn caller(value: *const i32, flag: bool) -> i32 {
    let first = alias(value);
    if flag {
        first + m::f(value)
    } else {
        core::cmp::max(alias(value), 0)
    }
}
```

Request item:

```text
{ id: 7, path: "m::f", name: "f" }
```

Transformation:

```rust
unsafe fn f(value: &i32) -> i32 {
    #[proctor(0)]
    *value
}
```

All three calls resolve to `m::f` and become
`crate::m::__proctor_wrapper_f(value)`. The `use` item remains unchanged; path
spelling does not control call identity.

### P3-CALL-02 `self_super_crate_and_fully_qualified_calls_rewrite`

Exact current compiler input:

```rust
pub(crate) mod outer {
    pub(crate) mod inner {
        pub(crate) unsafe fn f(value: *const i32) -> i32 {
            *value
        }

        pub(crate) unsafe fn via_self(value: *const i32) -> i32 {
            self::f(value)
        }
    }

    pub(crate) unsafe fn via_child(value: *const i32) -> i32 {
        inner::f(value)
    }

    pub(crate) mod sibling {
        pub(crate) unsafe fn via_super(value: *const i32) -> i32 {
            super::inner::f(value)
        }
    }
}

pub unsafe fn via_crate(value: *const i32) -> i32 {
    crate::outer::inner::f(value)
}
```

Request item:

```text
{ id: 7, path: "outer::inner::f", name: "f" }
```

Transformation:

```rust
unsafe fn f(value: &i32) -> i32 {
    #[proctor(0)]
    *value
}
```

Every external call becomes
`crate::outer::inner::__proctor_wrapper_f(value)`.

### P3-CALL-03 `mutually_recursive_scc_calls_stay_direct_while_external_calls_redirect`

Exact current compiler input:

```rust
pub unsafe fn even(value: *const i32, n: i32) -> i32 {
    if n == 0 { *value } else { odd(value, n - 1) }
}

pub unsafe fn odd(value: *const i32, n: i32) -> i32 {
    if n == 0 { *value } else { even(value, n - 1) }
}

pub unsafe fn caller(value: *const i32) -> i32 {
    even(value, 4) + odd(value, 3)
}
```

Request items:

```text
[(7, "even", "even"), (8, "odd", "odd")]
```

Transformation:

```rust
unsafe fn odd(value: &i32, n: i32) -> i32 {
    #[proctor(0)]
    if n == 0 { *value } else { even(value, n - 1) }
}

unsafe fn even(value: &i32, n: i32) -> i32 {
    #[proctor(0)]
    if n == 0 { *value } else { odd(value, n - 1) }
}
```

The transformed implementations call `even`/`odd` directly with target
arguments. `caller` uses the two absolute wrapper paths. Each wrapper calls its
own implementation directly, never itself or the other wrapper.

Also run this exact two-transaction progression independently.

Initial current compiler input:

```rust
pub unsafe fn callee(value: *const i32) -> i32 {
    *value
}

pub unsafe fn caller(value: *const i32) -> i32 {
    callee(value)
}

pub unsafe fn top(value: *const i32) -> i32 {
    caller(value)
}
```

First request:

```text
{ id: 7, path: "callee", name: "callee" }
```

First transformation:

```rust
unsafe fn callee(value: &i32) -> i32 {
    #[proctor(0)]
    *value
}
```

After the first callback ends, compile its owned output in memory. It contains
the callee wrapper, and `caller` temporarily calls that wrapper. Then compile
that exact first output again to obtain a new matching `TyCtxt` and apply:

```text
{ id: 8, path: "caller", name: "caller" }
```

```rust
unsafe fn caller(value: &i32) -> i32 {
    #[proctor(0)]
    callee(value)
}
```

The second output preserves the earlier callee wrapper, replaces `caller` with
a target-signature body that calls `callee(value)` directly, creates a caller
wrapper, and redirects only `top` to the caller wrapper. No stale callee-wrapper
call remains in the new caller body. Compile the final output in a separate
in-memory compiler run. This proves each `TyCtxt` corresponds to the exact
current source for its transaction.

### P3-CALL-04 `direct_recursion_stays_direct_and_wrapper_call_is_not_rewritten`

Exact current compiler input:

```rust
pub unsafe fn recurse(value: *const i32, n: i32) -> i32 {
    if n == 0 { *value } else { recurse(value, n - 1) }
}

pub unsafe fn caller(value: *const i32) -> i32 {
    recurse(value, 2)
}
```

Transformation:

```rust
unsafe fn recurse(value: &i32, n: i32) -> i32 {
    #[proctor(0)]
    if n == 0 { *value } else { recurse(value, n - 1) }
}
```

The recursive implementation call remains `recurse(value, n - 1)`.
`caller` redirects to the wrapper. The newly inserted wrapper contains exactly
one direct call to `recurse`, proving calls were snapshotted before insertion.

### P3-CALL-05 `unchanged_signature_needs_no_rewrite_and_macro_input_call_errors`

First run this unchanged-signature case.

Current compiler input:

```rust
pub unsafe fn f(value: *const i32) -> i32 {
    *value
}

pub unsafe fn caller(value: *const i32) -> i32 {
    dbg!(f(value))
}
```

Transformation:

```rust
unsafe fn f(value: *const i32) -> i32 {
    #[proctor(0)]
    *value + 1
}
```

No wrapper is generated and the macro token input remains `f(value)`, because
no compatibility redirection is needed.

Then run the same exact current compiler input with:

```rust
unsafe fn f(value: &i32) -> i32 {
    #[proctor(0)]
    *value + 1
}
```

This requires a wrapper, so replacement returns the specific macro-call
rewrite error and no output. The test confirms expanded-HIR dependency/call
resolution may observe the call, while the unexpanded surface rewriter refuses
to leave it stale. The first subcase is component-level robustness for a
signature-unchanged replacement; it does not make source calls inside macro
inputs part of the supported local-transformation contract.

## 12. Executable migration and atomic-integration tests

### P3-MAIN-01 `zero_argument_main_0_leaves_excluded_main_unchanged`

Exact current compiler input:

```rust
unsafe fn main_0() -> core::ffi::c_int {
    0
}

pub fn main() {
    unsafe {
        ::std::process::exit(main_0() as i32)
    }
}
```

Transformation:

```rust
unsafe fn main_0() -> core::ffi::c_int {
    #[proctor(0)]
    1
}
```

`main_0` receives the new body. The exact parsed `main` item is structurally
unchanged, and no wrapper is generated. Classification checks only the final
identifier symbol `main_0` and arity zero. The sibling is found by the final
identifier symbol `main`; neither body nor either signature is inspected.

### P3-MAIN-02 `two_argument_main_0_uses_fixed_main_and_never_wraps`

Exact current compiler input:

```rust
unsafe fn main_0(
    argc: core::ffi::c_int,
    argv: *mut *mut core::ffi::c_char,
) -> core::ffi::c_int {
    let _ = argv;
    argc
}

pub fn main() {
    let mut command_line_args: Vec<*mut core::ffi::c_char> = Vec::new();
    for arg in ::std::env::args() {
        command_line_args.push(
            ::std::ffi::CString::new(arg)
                .expect("Failed to convert argument into CString.")
                .into_raw(),
        );
    }
    command_line_args.push(::core::ptr::null_mut());
    unsafe {
        ::std::process::exit(
            main_0(
                (command_line_args.len() - 1) as core::ffi::c_int,
                command_line_args.as_mut_ptr() as *mut *mut core::ffi::c_char,
            ) as i32,
        )
    }
}
```

Transformation:

```rust
unsafe fn main_0(
    argc: core::ffi::c_int,
    argv: &mut [&mut [i8]],
) -> core::ffi::c_int {
    #[proctor(0)]
    let _ = argv;
    #[proctor(1)]
    argc
}
```

Despite the parameter-type change, no `__proctor_wrapper_main_0` is generated.
The transformed `main_0` uses the target slice signature. Replace `main`
exactly with:

```rust
pub fn main() {
    let mut command_line_arg_storage: Vec<Vec<i8>> = ::std::env::args()
        .map(|arg| {
            ::std::ffi::CString::new(arg)
                .expect("Failed to convert argument into CString.")
                .into_bytes_with_nul()
                .into_iter()
                .map(|byte| byte as i8)
                .collect()
        })
        .collect();

    let argc = command_line_arg_storage.len() as core::ffi::c_int;
    let mut command_line_arg_slices: Vec<&mut [i8]> = command_line_arg_storage
        .iter_mut()
        .map(|arg| arg.as_mut_slice())
        .collect();

    let mut argv_terminator: [i8; 0] = [];
    command_line_arg_slices.push(&mut argv_terminator);

    unsafe {
        ::std::process::exit(
            main_0(argc, command_line_arg_slices.as_mut_slice()) as i32,
        )
    }
}
```

Assert `argc` excludes the appended empty sentinel, the inner vectors retain
their trailing NUL through `into_bytes_with_nul`, and the output compiles
in-memory.

Also run this nested-module form independently:

```rust
pub mod app {
    pub(crate) unsafe fn main_0(
        mut argc: core::ffi::c_int,
        mut argv: *mut *mut core::ffi::c_char,
    ) -> core::ffi::c_int {
        let _ = argv;
        argc
    }

    pub fn main() {
        let mut command_line_args: Vec<*mut core::ffi::c_char> = Vec::new();
        for arg in ::std::env::args() {
            command_line_args.push(
                ::std::ffi::CString::new(arg)
                    .expect("Failed to convert argument into CString.")
                    .into_raw(),
            );
        }
        command_line_args.push(::core::ptr::null_mut());
        unsafe {
            ::std::process::exit(
                main_0(
                    (command_line_args.len() - 1) as core::ffi::c_int,
                    command_line_args.as_mut_ptr() as *mut *mut core::ffi::c_char,
                ) as i32,
            )
        }
    }
}

pub mod distractor {
    pub unsafe fn main_0() -> core::ffi::c_int {
        9
    }

    pub fn main() {
        unsafe {
            ::std::process::exit(main_0() as i32)
        }
    }
}
```

Request only:

```text
{ id: 7, path: "app::main_0", name: "main_0" }
```

Transformation:

```rust
unsafe fn main_0(
    mut argc: core::ffi::c_int,
    mut argv: &mut [&mut [i8]],
) -> core::ffi::c_int {
    #[proctor(0)]
    let _ = argv;
    #[proctor(1)]
    argc
}
```

Recognize the two-argument form solely from the final identifier symbol
`main_0` and arity two, despite `pub(crate)` and binding `mut`; do not inspect
parameter types, return type, safety, ABI, attributes, or body.
Replace only sibling `app::main` with the same fixed function body shown above,
inserted inside `app` so its unqualified `main_0` resolves there. Preserve
`distractor::main_0` and `distractor::main` exactly, and generate no wrapper.

### P3-ATOMIC-01 `one_unsupported_item_aborts_multi_item_transaction`

Exact current compiler input:

```rust
pub unsafe fn good(value: *const i32) -> i32 {
    *value
}

pub unsafe fn bad(value: *mut i32) {
    let _ = value;
}

pub unsafe fn caller(value: *mut i32) -> i32 {
    good(value) + {
        bad(value);
        0
    }
}
```

Request items:

```text
[(7, "good", "good"), (8, "bad", "bad")]
```

Transformation:

```rust
unsafe fn good(value: &i32) -> i32 {
    #[proctor(0)]
    *value
}

unsafe fn bad(value: Box<[i32]>) {
    #[proctor(0)]
    drop(value);
}
```

`good` alone would be replaceable, but `bad` requires the unsupported boxed
slice input conversion. The complete call returns the `bad` conversion error
with item 8/path/name. It returns no source containing a replaced `good`, no
wrapper, and no redirected caller. Reversing transformation function order
produces the same error.

## 13. Completion criteria

Phase 3 is complete when:

- the existing lifetime-name test oracle is updated exactly as Section 4
  specifies;
- all six lifetime-generic amendment cases pass and
  `generic_parameter_mismatch` has stable signature-category precedence;
- the Phase 1/2 local-item implementation and Rust tests are updated exactly
  as Section 4.2 specifies, including removal of obsolete expected-item
  diagnostics;
- executable recognition uses only final identifier symbols and `main_0`
  arity as Section 4.3 specifies;
- `item_replacer` owns the pure safety normalizer, replacement request/error
  types, and `replace_items`;
- all 48 named cases in this document pass;
- no test performs filesystem I/O, constructs a Cargo project, or invokes a
  subprocess or CLI;
- successful replacement output is parser-valid and is compiled in-memory
  whenever the case is intended to be a compilable project state;
- replacement preserves current visibility and normalized safety without
  trusting LLM header attributes, ABI, visibility, safety, or constness;
- wrappers are same-module siblings with preserved visibility, absolute
  implementation calls, and the exact base/`_0`/`_1` collision policy;
- ABI, `no_mangle`, and `export_name` follow Section 8 exactly;
- simultaneous source `no_mangle` and `export_name` is rejected;
- every supported conversion uses the exact null, empty-slice, ownership, and
  fixed-bound behavior in Sections 9 and 10;
- boxed-slice input and every unsupported conversion fail before output;
- direct external calls rewrite by HIR identity, while current-SCC calls stay
  direct and macro-token calls that need wrappers fail explicitly;
- zero- and two-argument executable migrations match Section 12;
- every failure is atomic and uses the six coarse debugging kinds with
  deterministic item context and an actionable message, without fine-grained
  subcodes or validator-style precedence;
- thin `normalize-safety` and `replace` CLI wiring that writes only one `.rs`
  output file is implemented but untested here; orchestrator-owned project
  copying is not part of either command; and
- from `crat/`, `cargo fmt`, `cargo clippy --workspace --all-targets`, and
  `cargo test --workspace` pass.
