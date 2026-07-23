# Phase 2 Test Plan: Structural Validator

## 1. Purpose

This document specifies the complete automated test suite planned for Phase 2
of the local-transformation prototype. It is the hand-over contract for:

- the four Phase 1 skeleton-generation adjustments required during Phase 2;
- moving existing skeleton logic and tests out of `lib.rs`;
- the in-memory structural validator;
- validation request and response JSON; and
- deterministic, repair-oriented diagnostics.

Phase 1 is already complete. Do not modify `phase-1-test-plan.md` and do not
revisit any Phase 1 behavior except the four adjustments in Section 4 below.
Existing Phase 1 Rust tests must be updated as part of the Phase 2
implementation, but their historical planning document remains unchanged.

`unsupported.md` remains the conceptual input contract. Tests that exercise
easy parser, skeleton, or validator behavior outside that contract do not
expand the supported local-transformation input language.

The suite contains 69 named cases:

| Area | Cases |
| --- | ---: |
| Phase 1 generator adjustments | 6 |
| JSON contract and setup | 13 |
| Whole-result and function-set checks | 6 |
| Signatures and structural types | 8 |
| Labels and expansion groups | 9 |
| Controls and descendant placement | 10 |
| Existing declarations and target local types | 8 |
| Generated temporaries, unsafe blocks, and attributes | 6 |
| Diagnostics and integration | 3 |

Later bug fixes may add regression cases, but Phase 2 is not complete until
every case in this document is implemented and passing.

## 2. Test execution and filesystem policy

All tests are ordinary Rust tests under `crates/tools/src/` and run with:

```bash
cd crat
cargo test --workspace
```

Skeleton-adjustment tests use source strings with
`utils::compilation::run_compiler_on_str`, exactly like the completed Phase 1
tests. Validator tests are parser-only and call typed in-memory APIs such as:

```rust,ignore
validate(&ValidationRequest) -> ValidationResponse
validate_json(&str) -> String
```

No Phase 2 test may:

- create a file or directory;
- invoke `crat-tool` or another subprocess;
- test Clap;
- read a checked-in fixture or snapshot;
- construct a Cargo project;
- invoke `cargo` recursively;
- mutate an environment variable or other process-global setting; or
- modify any source, manifest, lockfile, or generated artifact.

The thin `crat-tool validate --input ... --output ...` filesystem wiring is
implemented in Phase 2 but is not tested here.

## 3. Test organization and comparison policy

Move the existing skeleton implementation into `skeleton.rs` and its existing
tests into `skeleton/tests.rs`. Put validator implementation in `validator.rs`
and the cases below in `validator/tests.rs`. Keep `lib.rs` minimal: it may
contain required crate-level feature attributes and `extern crate`
declarations, module declarations, and public re-exports, but no
skeleton/validator implementation or large test module.

Unless a case supplies multiple functions, the request entry is:

```text
id = 7
name = "f"
skeleton = the exact "Expected skeleton" Rust block
transformation = the exact "Transformation" Rust block
```

Every Rust block is complete parser input. Ellipses are never implicit Rust.
Whitespace is insignificant except in byte-determinism and JSON golden cases.
When a case expects `invalid`, it names the required stable error code and
important message fragments. Messages may contain additional parser detail but
must identify the function/item, label or structural role when applicable,
expected form, observed form, and corrective constraint.

Whole-result failures use one failure with `id = null`, `name = null`, and the
unchanged transformation as `failed_snippet`. Item failures use the expected
ID/name and the pretty-printed matched function. A parent association failure
suppresses dependent descendant errors as described by the individual cases.

Binding-location assertions use the structural-position model from
`prototype-plan.md` Section 13.1. In particular, compare the expected
declaration anchor and the complete pattern-child path, including tuple or
tuple-struct index, struct field, slice position, `@` binder/subpattern role,
`or` alternative, and reference/parenthesized wrapper. Generated sibling
statements do not renumber expected declaration anchors. Ignore only binding
`mut`; preserve by-value versus `ref`.

All setup checking, failure ordering, error ordering, and serialization must be
deterministic. Request `schema_version` and function `id` values are JSON
integers; the request version is exactly `1`, and each ID is in the Rust `u64`
range. Every response uses response `schema_version: 1`, even when the input is
malformed JSON or contains an unsupported request version.

## 4. Phase 1 adjustments performed during Phase 2

Only these four Phase 1 behaviors change during Phase 2:

> In `annotated_skeleton` and `target_signature`, recursively mark every
> binding identifier mutable. Leave `annotated_source` and `source_signature`
> unchanged.

> Make every target function header `unsafe fn`, while preserving source
> safety in `annotated_source` and `source_signature`.

> Emit no `Fn` record for a free function whose final identifier is exactly
> `main`, without inspecting its body.

> For the supported two-argument `main_0`, force the target `argv` parameter
> type to `&mut [&mut [i8]]` regardless of pointer analysis, while preserving
> the raw source type.

This includes parameters and bindings in simple/destructuring `let`,
`let-else`, `if let`, `while let`, `for`, and match-arm patterns. Wildcards
remain wildcards. It also includes bindings inside supported function-local
`const` and `static` initializer blocks. Preserve a binding's by-reference
mode; `ref x` becomes `ref mut x`.

Update existing Phase 1 Rust test oracles wherever they assert target
signatures or skeleton declarations. At minimum this includes:

- JSON embedded-text oracles containing skeleton locals;
- sanitized target-signature oracles containing parameters;
- skeleton-shape cases with locals or pattern bindings;
- TYPE target signatures and explicit local declarations;
- comprehensive fixtures that inspect target parameters, locals, or patterns;
  and
- inline JSON goldens if their skeleton contains a binding.

Do not change source-rendering oracles except where an existing whole-output
oracle must now omit `main`; source safety and the raw `main_0` source
signature remain unchanged. Add the regressions below rather than adding cases
to `phase-1-test-plan.md`.

The supported-input clarification that function bodies may contain only
function-local `const` and `static` item declarations requires no Phase 1
generator change. Phase 2 setup validation enforces it against each expected
skeleton. It is not a fifth Phase 1 change.

These newly decided generator behaviors add no new structural-validator rule.
The validator already ignores the returned function's safety qualifier;
excluded `main` functions never enter validation requests; and forced
`main_0` parameter types use the ordinary structural type comparison.
Whole-project safety normalization and mechanical replacement of the
two-argument executable `main` belong to Phase 3 and are not tested here.

### P2-MUT-01 `target_parameters_and_simple_locals_are_mutable`

Exact compiler input:

```rust
pub unsafe fn f(input: i32, mut existing: i32) -> i32 {
    let value = input;
    let mut total: i32 = existing;
    total += value;
    total
}
```

Expected target skeleton:

```rust
pub unsafe fn f(mut input: i32, mut existing: i32) -> i32 {
    #[proctor(0)]
    let mut value: i32 = todo!();
    #[proctor(1)]
    let mut total: i32 = todo!();
    #[proctor(2)]
    todo!();
    #[proctor(3)]
    todo!()
}
```

Expected source rendering retains `input` and `value` as nonmutable and
`existing` and `total` as mutable.

### P2-MUT-02 `wildcards_remain_wildcards_and_source_is_unchanged`

Exact compiler input:

```rust
pub unsafe fn f(pair: (i32, i32)) {
    let (_, value) = pair;
    let _ = value;
}
```

Expected target skeleton:

```rust
pub unsafe fn f(mut pair: (i32, i32)) {
    #[proctor(0)]
    let (_, mut value) = todo!();
    #[proctor(1)]
    let _ = todo!();
}
```

The source function remains exactly nonmutable at `pair` and `value`.

### P2-MUT-03 `all_nested_pattern_binding_kinds_become_mutable`

This is a component-level robustness regression. Its `ref` binding deliberately
exercises easy skeleton/validator support outside the prototype's conceptual
supported-input model; it does not make source `ref` bindings supported by the
local transformation.

Exact compiler input:

```rust
enum E {
    Pair(i32, i32),
    Struct { x: i32 },
    Unit,
}

enum Choice {
    Left(i32),
    Right(i32),
}

pub unsafe fn f(
    pair: (i32, i32),
    mut opt: Option<(i32, i32)>,
    values: [(i32, i32); 1],
    value: E,
    choice: Choice,
) {
    const LOCAL: i32 = {
        let inner_const = 1;
        inner_const
    };
    static STATE: i32 = {
        let inner_static = 2;
        inner_static
    };
    let ref borrowed = pair;
    let _ = borrowed;
    let whole @ (a, b) = pair;
    let Some((c, d)): Option<(i32, i32)> = opt else {
        return;
    };
    if let Some((e, f)) = opt {
        let _ = e + f;
    }
    while let Some((g, h)) = opt {
        opt = None;
        let _ = g + h;
    }
    for (i, j) in values {
        let _ = i + j;
    }
    match value {
        E::Pair(k, l) => {
            let _ = k + l;
        }
        E::Struct { x: m } => {
            let _ = m;
        }
        E::Unit => {}
    }
    match choice {
        Choice::Left(n) | Choice::Right(n) => {
            let _ = n;
        }
    }
    let _ = (whole, a, b, c, d);
}
```

Expected normalized target skeleton:

```rust
pub unsafe fn f(
    mut pair: (i32, i32),
    mut opt: Option<(i32, i32)>,
    mut values: [(i32, i32); 1],
    mut value: E,
    mut choice: Choice,
) {
    #[proctor(0)]
    const LOCAL: i32 = {
        #[proctor(1)]
        let mut inner_const = 1;
        #[proctor(2)]
        inner_const
    };
    #[proctor(3)]
    static STATE: i32 = {
        #[proctor(4)]
        let mut inner_static = 2;
        #[proctor(5)]
        inner_static
    };
    #[proctor(6)]
    let ref mut borrowed = todo!();
    #[proctor(7)]
    let _ = todo!();
    #[proctor(8)]
    let mut whole @ (mut a, mut b) = todo!();
    #[proctor(9)]
    let Some((mut c, mut d)): Option<(i32, i32)> = todo!() else {
        #[proctor(10)]
        return;
    };
    #[proctor(11)]
    if let Some((mut e, mut f)) = todo!() {
        #[proctor(12)]
        let _ = todo!();
    }
    #[proctor(13)]
    while let Some((mut g, mut h)) = todo!() {
        #[proctor(14)]
        todo!();
        #[proctor(15)]
        let _ = todo!();
    }
    #[proctor(16)]
    for (mut i, mut j) in todo!() {
        #[proctor(17)]
        let _ = todo!();
    }
    #[proctor(18)]
    match todo!() {
        E::Pair(mut k, mut l) => {
            #[proctor(19)]
            let _ = todo!();
        }
        E::Struct { x: mut m } => {
            #[proctor(20)]
            let _ = todo!();
        }
        E::Unit => {}
    }
    #[proctor(21)]
    match todo!() {
        Choice::Left(mut n) | Choice::Right(mut n) => {
            #[proctor(22)]
            let _ = todo!();
        }
    }
    #[proctor(23)]
    let _ = todo!();
}
```

The exact labels above are the expected depth-first Phase 1 labels. The
annotated source has the same labels but preserves every original binding mode:
in particular, `pair`, `value`, `choice`, `ref borrowed`, and the originally
nonmutable nested bindings—including `inner_const` and `inner_static`—remain
nonmutable there.

Also run this smaller source independently to cover a source binding that is
already `ref mut`:

```rust
pub unsafe fn ref_mut_source(mut value: i32) {
    let ref mut borrowed = value;
    let _ = borrowed;
}
```

Expected target skeleton:

```rust
pub unsafe fn ref_mut_source(mut value: i32) {
    #[proctor(0)]
    let ref mut borrowed = todo!();
    #[proctor(1)]
    let _ = todo!();
}
```

The target keeps `ref mut`, while the annotated source remains unchanged.

### P2-SAFETY-01 `safe_source_functions_get_unsafe_target_headers`

Exact compiler input:

```rust
pub fn safe(input: i32) -> i32 {
    let value = input;
    value
}

pub unsafe fn already_unsafe(input: i32) -> i32 {
    input
}
```

Expected target skeletons:

```rust
pub unsafe fn safe(mut input: i32) -> i32 {
    #[proctor(0)]
    let mut value: i32 = todo!();
    #[proctor(1)]
    todo!()
}
```

```rust
pub unsafe fn already_unsafe(mut input: i32) -> i32 {
    #[proctor(0)]
    todo!()
}
```

The exact source signatures remain:

```text
pub fn safe(input: i32) -> i32
pub unsafe fn already_unsafe(input: i32) -> i32
```

The exact target signatures are:

```text
pub unsafe fn safe(mut input: i32) -> i32
pub unsafe fn already_unsafe(mut input: i32) -> i32
```

The annotated source also preserves the safe/unsafe distinction and original
binding mutability. This is generation-only behavior. Existing validator cases
continue to accept either safety qualifier in an LLM result.

### P2-MAIN-01 `every_free_function_named_main_is_omitted_without_body_inspection`

This component test deliberately uses unsupported `main` bodies to prove that
the generator performs only the decided name-based exclusion. Exact compiler
input:

```rust
pub fn main() {
    let ignored = 1;
    core::mem::drop(ignored);
}

unsafe fn main_0() -> core::ffi::c_int {
    0
}

mod nested {
    pub fn main() {
        panic!("also omitted");
    }

    pub unsafe fn helper() -> i32 {
        1
    }
}

mod raw {
    pub fn r#main() {}

    pub unsafe fn helper() -> i32 {
        2
    }
}
```

Expected records are exactly:

```text
id=0 path="main_0" kind=Fn
id=1 path="nested::helper" kind=Fn
id=2 path="raw::helper" kind=Fn
```

Neither `main` receives an ID or record, at root or inline-module depth.
The raw spelling `r#main` is the same excluded identifier symbol. No omitted
body is inspected or diagnosed. The supported-input contract, not Phase 1,
restricts executable `main` to the two C2Rust forms.

### P2-MAIN-02 `supported_main_0_forms_are_generated_with_the_fixed_argv_override`

Run the two executable-library inputs independently.

Zero-argument form:

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

Expected output contains only the `main_0` function record. Its target
skeleton is:

```rust
unsafe fn main_0() -> core::ffi::c_int {
    #[proctor(0)]
    todo!()
}
```

The zero-argument form receives no special type override.

Two-argument form:

```rust
unsafe fn main_0(
    mut argc: core::ffi::c_int,
    mut argv: *mut *mut core::ffi::c_char,
) -> core::ffi::c_int {
    if argc > 0 {
        **argv as core::ffi::c_int
    } else {
        0
    }
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

Expected output again contains only the `main_0` function record. Its source
signature remains:

```text
unsafe fn main_0(mut argc: core::ffi::c_int, mut argv: *mut *mut core::ffi::c_char) -> core::ffi::c_int
```

Its normalized target skeleton is exactly:

```rust
unsafe fn main_0(
    mut argc: core::ffi::c_int,
    mut argv: &mut [&mut [i8]],
) -> core::ffi::c_int {
    #[proctor(0)]
    if todo!() {
        #[proctor(1)]
        todo!()
    } else {
        #[proctor(2)]
        todo!()
    }
}
```

The target `argv` type is forced even if the ordinary pointer-analysis decision
would keep it raw or choose another representation. `argc`, the return type,
the source signature, and every other pointer decision remain unchanged.
Phase 3—not this test—owns mechanical replacement of the excluded
two-argument `main`.

## 5. JSON contract and setup tests

### P2-JSON-01 `valid_response_has_explicit_status`

Expected skeleton:

```rust
unsafe fn f() {
    #[proctor(0)]
    todo!();
}
```

Transformation:

```rust
unsafe fn f() {
    #[proctor(0)]
    return;
}
```

Expected byte-exact response:

```json
{
  "schema_version": 1,
  "status": "valid"
}
```

### P2-JSON-02 `invalid_response_matches_schema_and_key_order`

Expected skeleton:

```rust
unsafe fn f() {
    #[proctor(0)]
    todo!();
}
```

Transformation:

```rust
unsafe fn f() {}
```

Expected response has keys `schema_version`, `status`, `failures`; the sole
failure has keys `id`, `name`, `failed_snippet`, `errors`; each error has keys
`code`, `message`. Status is `invalid`, the failure is `(7,"f")`, and the error
code is `missing_label`. No undefined field or trailing newline is emitted.

### P2-JSON-03 `setup_error_matches_schema_and_key_order`

Exact JSON input:

```json
{
  "schema_version": 2,
  "expected_functions": [],
  "transformation": ""
}
```

Expected response has exactly `schema_version`, `status`, `error`; status is
`setup_error`; the error has exactly `code`, `message`; code is
`unsupported_schema_version`.

### P2-JSON-04 `json_round_trip_preserves_embedded_rust`

Exact JSON input:

```json
{
  "schema_version": 1,
  "expected_functions": [
    {
      "id": 7,
      "name": "f",
      "skeleton": "unsafe fn f() -> &'static str {\n    #[proctor(0)]\n    todo!()\n}"
    }
  ],
  "transformation": "unsafe fn f() -> &'static str {\n    #[proctor(0)]\n    \"quote:\\\" slash:\\\\ line:\\n\"\n}"
}
```

Expected status is `valid`, and typed request deserialization preserves both
Rust strings byte-for-byte.

### P2-JSON-05 `response_serialization_is_byte_deterministic`

Expected skeleton:

```rust
unsafe fn f() {
    #[proctor(0)]
    todo!();
    #[proctor(1)]
    todo!();
}
```

Transformation:

```rust
unsafe fn f() {
    #[proctor(2)]
    return;
}
```

Two independent validations and serializations produce identical response
bytes, including failure and error ordering.

### P2-SETUP-01 `malformed_request_json_is_setup_error`

Exact input bytes:

```text
{"schema_version":1,"expected_functions":[
```

Expected code is `invalid_request_json`, status is `setup_error`, and the
process-level API does not panic. Also run each exact input independently; each
must produce the same code because schema versions and IDs must deserialize as
integers and IDs must fit `u64`:

```json
{"schema_version":1.0,"expected_functions":[],"transformation":""}
```

```json
{"schema_version":1,"expected_functions":[{"id":-1,"name":"f","skeleton":"unsafe fn f() {}"}],"transformation":""}
```

```json
{"schema_version":1,"expected_functions":[{"id":1.0,"name":"f","skeleton":"unsafe fn f() {}"}],"transformation":""}
```

```json
{"schema_version":1,"expected_functions":[{"id":18446744073709551616,"name":"f","skeleton":"unsafe fn f() {}"}],"transformation":""}
```

Every response in this case contains response `schema_version: 1`.

### P2-SETUP-02 `unknown_request_field_is_setup_error`

Exact JSON input:

```json
{
  "schema_version": 1,
  "expected_functions": [
    {
      "id": 7,
      "name": "f",
      "skeleton": "unsafe fn f() { #[proctor(0)] todo!(); }"
    }
  ],
  "transformation": "unsafe fn f() { #[proctor(0)] return; }",
  "extra": true
}
```

Expected code is `unknown_request_field`.

### P2-SETUP-03 `empty_expected_function_list_is_setup_error`

Exact JSON input:

```json
{
  "schema_version": 1,
  "expected_functions": [],
  "transformation": ""
}
```

Expected code is `empty_expected_functions`.

### P2-SETUP-04 `duplicate_expected_ids_are_setup_error`

Expected skeletons:

```rust
unsafe fn f() {
    #[proctor(0)]
    todo!();
}
```

```rust
unsafe fn g() {
    #[proctor(0)]
    todo!();
}
```

Use entries `(id=7,name="f")` and `(id=7,name="g")`. Transformation:

```rust
unsafe fn f() {
    #[proctor(0)]
    return;
}

unsafe fn g() {
    #[proctor(0)]
    return;
}
```

Expected code is `duplicate_expected_id`.

### P2-SETUP-05 `duplicate_expected_names_are_setup_error`

Expected skeleton entries are both named `f`, with IDs 7 and 8:

```rust
unsafe fn f() {
    #[proctor(0)]
    todo!();
}
```

```rust
unsafe fn f() {
    #[proctor(0)]
    todo!();
}
```

Transformation:

```rust
unsafe fn f() {
    #[proctor(0)]
    return;
}
```

Expected code is `duplicate_expected_name`.

### P2-SETUP-06 `expected_skeleton_must_parse_as_one_function`

Use these two malformed setup requests independently.

Expected skeleton A:

```rust
unsafe fn f( {
```

Transformation A:

```rust
unsafe fn f() {}
```

Expected skeleton B:

```rust
unsafe fn f() {}
unsafe fn g() {}
```

Transformation B:

```rust
unsafe fn f() {}
```

A produces `expected_skeleton_parse_error`; B produces
`expected_skeleton_item_count`.

### P2-SETUP-07 `expected_skeleton_name_must_match_metadata`

Use metadata `(id=7,name="f")`.

Expected skeleton:

```rust
unsafe fn g() {
    #[proctor(0)]
    todo!();
}
```

Transformation:

```rust
unsafe fn f() {
    #[proctor(0)]
    return;
}
```

Expected code is `expected_skeleton_name_mismatch`; the message names both
`f` and `g`.

### P2-SETUP-08 `invalid_expected_skeleton_is_setup_error`

Run these expected skeletons independently with metadata `(id=7,name="f")`.
First, an invalid label tree:

```rust
unsafe fn f() {
    #[proctor(0)]
    return;
    #[proctor(0)]
    return;
}
```

An expected skeleton with a noncanonical label token:

```rust
unsafe fn f() {
    #[proctor(00)]
    return;
}
```

Then representative prohibited function-local item declarations:

```rust
unsafe fn f() {
    #[proctor(0)]
    fn local() {}
}
```

```rust
unsafe fn f() {
    #[proctor(0)]
    const LOCAL: () = {
        #[proctor(1)]
        fn nested() {}
    };
}
```

```rust
unsafe fn f() {
    #[proctor(0)]
    type Local = i32;
}
```

```rust
unsafe fn f() {
    #[proctor(0)]
    struct Local(i32);
}
```

```rust
unsafe fn f() {
    #[proctor(0)]
    use core::mem;
}
```

```rust
unsafe fn f() {
    #[proctor(0)]
    macro_rules! local {
        () => {};
    }
}
```

Use the same transformation for every request:

```rust
unsafe fn f() {
    #[proctor(0)]
    return;
}
```

Every request produces `invalid_expected_skeleton`, and result validation does
not run. The noncanonical-label message gives the exact
`0|[1-9][0-9]*` grammar. For prohibited items, the message identifies `f`,
label 0, the observed item kind, and says that only function-local `const` and
`static` items are supported. The validator implements this as a
`const`/`static` whitelist over parsed item statements, so every other item
kind is rejected even when not enumerated above. The const-initializer case
proves that the whitelist scan is recursive. This setup check is Phase 2
behavior and does not require changing Phase 1 skeleton generation.

## 6. Whole-result and function-set tests

### P2-WHOLE-01 `result_parse_error_is_global`

Expected skeleton:

```rust
unsafe fn f() {
    #[proctor(0)]
    todo!();
}
```

Transformation:

```rust
unsafe fn f( {
```

Expected code is `result_parse_error`, the failure identity is null, and
`failed_snippet` is exactly the unchanged transformation.

### P2-WHOLE-02 `missing_expected_function_is_global`

Expected skeleton entries are `(7,"f")` and `(8,"g")`:

```rust
unsafe fn f() {
    #[proctor(0)]
    todo!();
}
```

```rust
unsafe fn g() {
    #[proctor(0)]
    todo!();
}
```

Transformation:

```rust
unsafe fn f() {
    #[proctor(0)]
    return;
}
```

Expected code is `missing_function`, the message identifies `g` and item 8,
and no item-specific validation of `f` is returned.

### P2-WHOLE-03 `unexpected_and_duplicate_functions_are_global`

Expected skeleton:

```rust
unsafe fn f() {
    #[proctor(0)]
    todo!();
}
```

Transformation:

```rust
unsafe fn f() {
    #[proctor(0)]
    return;
}

unsafe fn f() {
    #[proctor(0)]
    return;
}

unsafe fn extra() {}
```

The one global failure contains `duplicate_function` followed by
`unexpected_function`. Per-function checks are suppressed.

### P2-WHOLE-04 `every_nonfunction_top_level_item_is_unexpected`

Run each transformation independently against the same expected skeleton.

Expected skeleton:

```rust
unsafe fn f() {
    #[proctor(0)]
    todo!();
}
```

Transformations:

```rust
use core::ptr;
unsafe fn f() {
    #[proctor(0)]
    return;
}
```

```rust
extern crate core;
unsafe fn f() {
    #[proctor(0)]
    return;
}
```

```rust
const X: i32 = 1;
unsafe fn f() {
    #[proctor(0)]
    return;
}
```

```rust
mod m {}
unsafe fn f() {
    #[proctor(0)]
    return;
}
```

```rust
macro_rules! m { () => {} }
unsafe fn f() {
    #[proctor(0)]
    return;
}
```

```rust
static X: i32 = 1;
unsafe fn f() {
    #[proctor(0)]
    return;
}
```

```rust
type X = i32;
unsafe fn f() {
    #[proctor(0)]
    return;
}
```

```rust
struct X(i32);
unsafe fn f() {
    #[proctor(0)]
    return;
}
```

```rust
enum X {
    A,
}
unsafe fn f() {
    #[proctor(0)]
    return;
}
```

```rust
union X {
    value: i32,
}
unsafe fn f() {
    #[proctor(0)]
    return;
}
```

```rust
trait X {}
unsafe fn f() {
    #[proctor(0)]
    return;
}
```

```rust
impl X for i32 {}
unsafe fn f() {
    #[proctor(0)]
    return;
}
```

```rust
unsafe extern "C" {
    fn foreign();
}
unsafe fn f() {
    #[proctor(0)]
    return;
}
```

```rust
m!();
unsafe fn f() {
    #[proctor(0)]
    return;
}
```

Each result has one global `unexpected_item` error whose message names the
observed item kind.

### P2-WHOLE-05 `result_function_order_is_irrelevant`

Expected entries are ordered `(7,"f")`, `(8,"r#match")`:

```rust
unsafe fn f() {
    #[proctor(0)]
    todo!();
}
```

```rust
unsafe fn r#match() {
    #[proctor(0)]
    todo!();
}
```

Transformation:

```rust
unsafe fn r#match() {
    #[proctor(0)]
    return;
}

unsafe fn f() {
    #[proctor(0)]
    return;
}
```

Expected status is `valid`. Raw identifiers are matched by their complete Rust
spelling.

### P2-WHOLE-06 `function_header_attributes_do_not_add_items`

Expected skeleton:

```rust
unsafe fn f() {
    #[proctor(0)]
    todo!();
}
```

Transformation:

```rust
#[inline(always)]
pub const fn f() {
    #[proctor(0)]
    return;
}
```

Expected status is `valid`: function-header attributes, visibility, and
`const` are ignored. The candidate phase later emits its own header.

## 7. Signature and structural-type tests

### P2-SIG-01 `binding_mutability_and_ignored_header_properties_may_differ`

Expected skeleton:

```rust
pub unsafe extern "C" fn f(mut p: &mut [i32], mut n: usize) -> i32 {
    #[proctor(0)]
    todo!()
}
```

Transformation:

```rust
const fn f(p: &mut [i32], mut n: usize) -> i32 {
    #[proctor(0)]
    p[n]
}
```

Expected status is `valid`. Visibility, ABI, safety, constness, and parameter
binding `mut` are ignored; reference mutability inside `&mut [i32]` is not.

### P2-SIG-02 `parameter_count_and_names_are_exact`

Expected skeleton:

```rust
unsafe fn f(mut left: i32, mut right: i32) -> i32 {
    #[proctor(0)]
    todo!()
}
```

Run independently:

```rust
unsafe fn f(left: i32) -> i32 {
    #[proctor(0)]
    left
}
```

```rust
unsafe fn f(a: i32, right: i32) -> i32 {
    #[proctor(0)]
    a + right
}
```

Expected codes are `parameter_count_mismatch` and
`parameter_name_mismatch`, respectively.

### P2-SIG-03 `formatting_and_redundant_type_parentheses_are_ignored`

Expected skeleton:

```rust
unsafe fn f(mut p: Option<&'static mut [i32; 4]>) -> (i32, usize) {
    #[proctor(0)]
    todo!()
}
```

Transformation:

```rust
unsafe fn f(p: Option < & 'static mut ([i32; 4]) >) -> ((i32), usize) {
    #[proctor(0)]
    (0, p.unwrap().len())
}
```

Expected status is `valid`.

### P2-SIG-04 `path_qualification_is_structural`

Expected skeleton:

```rust
unsafe fn f(mut p: Option<&i32>) {
    #[proctor(0)]
    todo!();
}
```

Transformation:

```rust
unsafe fn f(p: core::option::Option<&i32>) {
    #[proctor(0)]
    return;
}
```

Expected code is `parameter_type_mismatch`; no name resolution or semantic
equivalence is attempted.

### P2-SIG-05 `pointer_and_reference_mutability_are_enforced`

Expected skeleton:

```rust
unsafe fn f(mut p: &mut i32, mut q: *const i32) {
    #[proctor(0)]
    todo!();
}
```

Transformation:

```rust
unsafe fn f(p: &i32, q: *mut i32) {
    #[proctor(0)]
    return;
}
```

Expected two `parameter_type_mismatch` errors in parameter order.

### P2-SIG-06 `array_lengths_and_generic_arguments_are_enforced`

Expected skeleton:

```rust
unsafe fn f(mut a: [u8; 4], mut p: Option<Box<[i32]>>) {
    #[proctor(0)]
    todo!();
}
```

Transformation:

```rust
unsafe fn f(a: [u8; 5], p: Option<Box<i32>>) {
    #[proctor(0)]
    return;
}
```

Expected two `parameter_type_mismatch` errors with rendered expected and
observed types.

### P2-SIG-07 `explicit_lifetime_names_in_types_are_enforced`

Expected skeleton:

```rust
unsafe fn f<'a>(mut x: &'a i32, mut y: &'a i32) -> &'a i32 {
    #[proctor(0)]
    todo!()
}
```

Transformation:

```rust
unsafe fn f<'b>(x: &'b i32, y: &'b i32) -> &'b i32 {
    #[proctor(0)]
    x
}
```

Expected parameter and return `*_type_mismatch` errors because explicit
lifetime names are structural. The generic declaration itself is not compared
separately.

### P2-SIG-08 `omitted_return_and_explicit_unit_are_distinct`

Expected skeleton:

```rust
unsafe fn f() {
    #[proctor(0)]
    todo!();
}
```

Transformation:

```rust
unsafe fn f() -> () {
    #[proctor(0)]
    return;
}
```

Expected code is `return_type_mismatch`.

## 8. Label and expansion-group tests

### P2-LABEL-01 `one_to_one_and_one_to_many_groups_are_valid`

Expected skeleton:

```rust
unsafe fn f(mut p: Option<&i32>) -> i32 {
    #[proctor(0)]
    let mut x: i32 = todo!();
    #[proctor(1)]
    todo!()
}
```

Transformation:

```rust
unsafe fn f(p: Option<&i32>) -> i32 {
    #[proctor(0)]
    let proctor_temp_var_0 = p.unwrap();
    #[proctor(0)]
    let x: i32 = *proctor_temp_var_0;
    #[proctor(1)]
    x
}
```

Expected status is `valid`.

### P2-LABEL-02 `every_expected_label_must_appear`

Expected skeleton:

```rust
unsafe fn f() {
    #[proctor(0)]
    todo!();
    #[proctor(1)]
    todo!();
}
```

Transformation:

```rust
unsafe fn f() {
    #[proctor(0)]
    return;
}
```

Expected code is `missing_label`; the message identifies label 1 and says it
must follow label 0.

### P2-LABEL-03 `new_numeric_label_is_rejected`

Expected skeleton:

```rust
unsafe fn f() {
    #[proctor(0)]
    todo!();
}
```

Transformation:

```rust
unsafe fn f() {
    #[proctor(0)]
    return;
    #[proctor(9)]
    return;
}
```

Expected code is `unexpected_label`.

### P2-LABEL-04 `groups_must_follow_expected_order`

Expected skeleton:

```rust
unsafe fn f() {
    #[proctor(0)]
    todo!();
    #[proctor(1)]
    todo!();
    #[proctor(2)]
    todo!();
}
```

Transformation:

```rust
unsafe fn f() {
    #[proctor(1)]
    return;
    #[proctor(0)]
    return;
    #[proctor(2)]
    return;
}
```

Expected code is `label_order_mismatch`, with expected sequence `0,1,2` and
observed group sequence `1,0,2`.

### P2-LABEL-05 `one_group_must_be_consecutive`

Expected skeleton:

```rust
unsafe fn f() {
    #[proctor(0)]
    todo!();
    #[proctor(1)]
    todo!();
}
```

Transformation:

```rust
unsafe fn f() {
    #[proctor(0)]
    return;
    #[proctor(1)]
    return;
    #[proctor(0)]
    return;
}
```

Expected code is `nonconsecutive_label`; the message says label 0 reappears
after label 1 begins.

### P2-LABEL-06 `unlabeled_sibling_at_expected_statement_level_is_rejected`

Expected skeleton:

```rust
unsafe fn f() {
    #[proctor(0)]
    todo!();
    #[proctor(1)]
    todo!();
}
```

Transformation:

```rust
unsafe fn f() {
    #[proctor(0)]
    return;
    consume();
    #[proctor(1)]
    return;
}
```

Expected code is `unlabeled_group_statement`. The message says the new
statement must join an expansion group or be nested inside one.

### P2-LABEL-07 `new_unlabeled_nested_code_inside_leaf_group_is_valid`

Expected skeleton:

```rust
unsafe fn f(mut flag: bool) -> i32 {
    #[proctor(0)]
    todo!()
}
```

Transformation:

```rust
unsafe fn f(flag: bool) -> i32 {
    #[proctor(0)]
    if flag {
        let proctor_temp_var_0 = 1;
        proctor_temp_var_0
    } else {
        0
    }
}
```

Expected status is `valid`: label 0 is a leaf in the skeleton, and all newly
introduced nested statements are unlabeled.

### P2-LABEL-08 `existing_label_may_not_repeat_in_nested_code`

Expected skeleton:

```rust
unsafe fn f(mut flag: bool) -> i32 {
    #[proctor(0)]
    todo!()
}
```

Transformation:

```rust
unsafe fn f(flag: bool) -> i32 {
    #[proctor(0)]
    if flag {
        #[proctor(0)]
        1
    } else {
        0
    }
}
```

Expected code is `nested_label_repetition`.

### P2-LABEL-09 `malformed_duplicate_and_misplaced_proctor_attributes_are_rejected`

Use these transformations independently against the same expected skeleton.

Expected skeleton:

```rust
unsafe fn f() {
    #[proctor(0)]
    todo!();
}
```

Transformations:

```rust
unsafe fn f() {
    #[proctor(x)]
    return;
}
```

```rust
unsafe fn f() {
    #[proctor(0, 1)]
    return;
}
```

```rust
unsafe fn f() {
    #[proctor()]
    return;
}
```

```rust
unsafe fn f() {
    #[proctor(4294967296)]
    return;
}
```

```rust
unsafe fn f() {
    #[other::proctor(0)]
    return;
}
```

```rust
unsafe fn f() {
    #[proctor(0)]
    #[proctor(0)]
    return;
}
```

```rust
unsafe fn f() {
    #[proctor(00)]
    return;
}
```

```rust
unsafe fn f() {
    #[proctor(1_0)]
    return;
}
```

```rust
unsafe fn f() {
    #[proctor(0u32)]
    return;
}
```

```rust
unsafe fn f() {
    #[proctor(0x0)]
    return;
}
```

```rust
unsafe fn f() {
    #[proctor(-1)]
    return;
}
```

```rust
unsafe fn f() {
    #[proctor(0)]
    return;
    #[proctor(4294967295)]
    return;
}
```

```rust
unsafe fn f() {
    #[proctor(0)]
    consume(#[proctor(0)] 1);
}
```

The first eleven produce `malformed_label`: respectively nonnumeric argument,
multiple arguments, zero arguments, `u32` overflow, nonexact attribute path,
duplicate attributes, a leading zero, a numeric separator, an integer suffix,
a nondecimal radix prefix, and a sign. The twelfth transformation proves the
inclusive upper boundary: `4294967295` is lexically and numerically valid, so
it produces `unexpected_label`, not `malformed_label`, against the label-0
skeleton. The accepted token grammar is exactly `0|[1-9][0-9]*` with a value
in the `u32` range. The final transformation produces `misplaced_label`.
Assert against original literal-token spelling, not only a normalized AST
integer value. None is reclassified as merely missing label 0.

## 9. Control and descendant-placement tests

### P2-CTRL-01 `all_phase_1_control_kinds_are_preserved`

Expected skeleton:

```rust
unsafe fn f(mut flag: bool, mut value: Option<i32>, mut xs: [i32; 1]) {
    #[proctor(0)]
    if todo!() {
        #[proctor(1)]
        todo!();
    } else {
        #[proctor(2)]
        todo!();
    }
    #[proctor(3)]
    if let Some(mut x) = todo!() {
        #[proctor(4)]
        todo!();
    }
    #[proctor(5)]
    while todo!() {
        #[proctor(6)]
        todo!();
    }
    #[proctor(7)]
    while let Some(mut x) = todo!() {
        #[proctor(8)]
        todo!();
    }
    #[proctor(9)]
    for mut x in todo!() {
        #[proctor(10)]
        todo!();
    }
    #[proctor(11)]
    loop {
        #[proctor(12)]
        break;
    }
    #[proctor(13)]
    match todo!() {
        Some(mut x) => {
            #[proctor(14)]
            todo!();
        }
        None => {
            #[proctor(15)]
            todo!();
        }
    }
    #[proctor(16)]
    {
        #[proctor(17)]
        todo!();
    }
}
```

Transformation:

```rust
unsafe fn f(flag: bool, value: Option<i32>, xs: [i32; 1]) {
    #[proctor(0)]
    if flag {
        #[proctor(1)]
        consume(1);
    } else {
        #[proctor(2)]
        consume(2);
    }
    #[proctor(3)]
    if let Some(x) = value {
        #[proctor(4)]
        consume(x);
    }
    #[proctor(5)]
    while flag {
        #[proctor(6)]
        break;
    }
    #[proctor(7)]
    while let Some(x) = value {
        #[proctor(8)]
        consume(x);
    }
    #[proctor(9)]
    for x in xs {
        #[proctor(10)]
        consume(x);
    }
    #[proctor(11)]
    loop {
        #[proctor(12)]
        break;
    }
    #[proctor(13)]
    match value {
        Some(x) => {
            #[proctor(14)]
            consume(x);
        }
        None => {
            #[proctor(15)]
            return;
        }
    }
    #[proctor(16)]
    {
        #[proctor(17)]
        return;
    }
}
```

Expected status is `valid`.

### P2-CTRL-02 `control_kinds_are_distinct`

Expected skeleton:

```rust
unsafe fn f(mut flag: bool) {
    #[proctor(0)]
    if todo!() {
        #[proctor(1)]
        todo!();
    }
}
```

Transformation:

```rust
unsafe fn f(flag: bool) {
    #[proctor(0)]
    while flag {
        #[proctor(1)]
        return;
    }
}
```

Expected code is `control_kind_mismatch`; the message says label 0 expected
`if` and observed `while`. No missing-label error for descendant 1 is emitted.

### P2-CTRL-03 `if_and_if_let_are_distinct`

Expected skeleton:

```rust
unsafe fn f(mut value: Option<i32>) {
    #[proctor(0)]
    if let Some(mut x) = todo!() {
        #[proctor(1)]
        todo!();
    }
}
```

Transformation:

```rust
unsafe fn f(value: Option<i32>) {
    #[proctor(0)]
    if value.is_some() {
        #[proctor(1)]
        return;
    }
}
```

Expected code is `control_kind_mismatch`, specifically `if let` versus `if`.

### P2-CTRL-04 `control_statement_role_is_preserved`

Use these transformations independently.

Expected skeleton:

```rust
unsafe fn f(mut flag: bool) -> i32 {
    #[proctor(0)]
    let mut x: i32 = if todo!() {
        #[proctor(1)]
        todo!()
    } else {
        #[proctor(2)]
        todo!()
    };
    #[proctor(3)]
    return if todo!() {
        #[proctor(4)]
        todo!()
    } else {
        #[proctor(5)]
        todo!()
    };
}
```

Transformation A:

```rust
unsafe fn f(flag: bool) -> i32 {
    #[proctor(0)]
    if flag {
        #[proctor(1)]
        1
    } else {
        #[proctor(2)]
        2
    }
    #[proctor(3)]
    return if flag {
        #[proctor(4)]
        3
    } else {
        #[proctor(5)]
        4
    };
}
```

Transformation B:

```rust
unsafe fn f(flag: bool) -> i32 {
    #[proctor(0)]
    let x: i32 = if flag {
        #[proctor(1)]
        1
    } else {
        #[proctor(2)]
        2
    };
    #[proctor(3)]
    if flag {
        #[proctor(4)]
        return 3;
    } else {
        #[proctor(5)]
        return 4;
    }
}
```

A reports `control_role_mismatch` for label 0 and B reports it for label 3.

Validate the direct `break`-value role with this independent request.

Expected skeleton:

```rust
unsafe fn g(mut flag: bool) -> i32 {
    #[proctor(0)]
    let mut result: i32 = loop {
        #[proctor(1)]
        break if todo!() {
            #[proctor(2)]
            todo!()
        } else {
            #[proctor(3)]
            todo!()
        };
    };
    #[proctor(4)]
    todo!()
}
```

Transformation:

```rust
unsafe fn g(flag: bool) -> i32 {
    #[proctor(0)]
    let result: i32 = loop {
        #[proctor(1)]
        if flag {
            #[proctor(2)]
            break 1;
        } else {
            #[proctor(3)]
            break 2;
        }
    };
    #[proctor(4)]
    result
}
```

Expected `control_role_mismatch` at label 1: the direct value of `break` was an
`if`; the returned transformation moved `break` inside the branches.

Validate the match-arm block-tail role with this independent request.

Expected skeleton:

```rust
unsafe fn h(mut value: i32) -> i32 {
    #[proctor(0)]
    match todo!() {
        0 => {
            #[proctor(1)]
            if todo!() {
                #[proctor(2)]
                todo!()
            } else {
                #[proctor(3)]
                todo!()
            }
        }
        _ => {
            #[proctor(4)]
            todo!()
        }
    }
}
```

Transformation:

```rust
unsafe fn h(value: i32) -> i32 {
    #[proctor(0)]
    match value {
        0 => {
            #[proctor(1)]
            return if value > 0 {
                #[proctor(2)]
                1
            } else {
                #[proctor(3)]
                2
            };
        }
        _ => {
            #[proctor(4)]
            0
        }
    }
}
```

Expected `control_role_mismatch` at label 1: the target control is the direct
tail expression statement of match arm 0, while the result moves it to the
direct value of `return`.

### P2-CTRL-05 `if_else_existence_and_recursive_else_if_shape_are_preserved`

Expected skeleton:

```rust
unsafe fn f(mut a: bool, mut b: bool) {
    #[proctor(0)]
    if todo!() {
        #[proctor(1)]
        todo!();
    } else if todo!() {
        #[proctor(2)]
        todo!();
    } else {
        #[proctor(3)]
        todo!();
    }
}
```

Transformation:

```rust
unsafe fn f(a: bool, b: bool) {
    #[proctor(0)]
    if a {
        #[proctor(1)]
        return;
    } else {
        #[proctor(2)]
        return;
        #[proctor(3)]
        return;
    }
}
```

Expected code is `branch_shape_mismatch`; descendants 2 and 3 are not
independently reported as moved because the missing `else if` is the parent
cause.

### P2-CTRL-06 `match_arm_count_order_and_guard_presence_are_preserved`

Expected skeleton:

```rust
unsafe fn f(mut value: i32) {
    #[proctor(0)]
    match todo!() {
        0 => {
            #[proctor(1)]
            todo!();
        }
        x if todo!() => {
            #[proctor(2)]
            todo!();
        }
        _ => {
            #[proctor(3)]
            todo!();
        }
    }
}
```

Run independently:

```rust
unsafe fn f(value: i32) {
    #[proctor(0)]
    match value {
        0 => {
            #[proctor(1)]
            return;
        }
        _ => {
            #[proctor(3)]
            return;
        }
    }
}
```

```rust
unsafe fn f(value: i32) {
    #[proctor(0)]
    match value {
        0 => {
            #[proctor(1)]
            return;
        }
        x => {
            #[proctor(2)]
            consume(x);
        }
        _ => {
            #[proctor(3)]
            return;
        }
    }
}
```

The first reports `match_arm_shape_mismatch`; the second reports
`match_guard_mismatch`.

### P2-CTRL-07 `labeled_descendant_cannot_move_between_branches`

Expected skeleton:

```rust
unsafe fn f(mut flag: bool) {
    #[proctor(0)]
    if todo!() {
        #[proctor(1)]
        todo!();
    } else {
        #[proctor(2)]
        todo!();
    }
}
```

Transformation:

```rust
unsafe fn f(flag: bool) {
    #[proctor(0)]
    if flag {
        #[proctor(2)]
        return;
    } else {
        #[proctor(1)]
        return;
    }
}
```

Expected `descendant_location_mismatch` errors identify labels 1 and 2 and
their expected/observed branches. A global preorder check alone is
insufficient.

### P2-CTRL-08 `control_group_has_exactly_one_preserved_control_root`

Expected skeleton:

```rust
unsafe fn f(mut flag: bool) {
    #[proctor(0)]
    if todo!() {
        #[proctor(1)]
        todo!();
    }
}
```

Valid transformation:

```rust
unsafe fn f(flag: bool) {
    #[proctor(0)]
    let proctor_temp_var_0 = flag;
    #[proctor(0)]
    if proctor_temp_var_0 {
        #[proctor(1)]
        return;
    }
    #[proctor(0)]
    consume(flag);
}
```

Invalid transformation:

```rust
unsafe fn f(flag: bool) {
    #[proctor(0)]
    if flag {
        #[proctor(1)]
        return;
    }
    #[proctor(0)]
    while flag {}
}
```

Also run this zero-root transformation independently:

```rust
unsafe fn f(flag: bool) {
    #[proctor(0)]
    consume(flag);
    #[proctor(0)]
    consume(!flag);
}
```

The first transformation is valid. The second reports
`multiple_control_roots`. The zero-root transformation reports
`missing_control_root`. Neither invalid result also reports descendant label 1
as missing because the required control-root association failed first.

### P2-CTRL-09 `plain_blocks_are_distinct_and_recursive`

Expected skeleton:

```rust
unsafe fn f() {
    #[proctor(0)]
    {
        #[proctor(1)]
        {
            #[proctor(2)]
            todo!();
        }
    }
}
```

Transformation:

```rust
unsafe fn f() {
    #[proctor(0)]
    {
        #[proctor(1)]
        loop {
            #[proctor(2)]
            break;
        }
    }
}
```

Expected `control_kind_mismatch` at label 1: plain block versus loop.

### P2-CTRL-10 `let_else_form_and_else_body_are_preserved`

Expected skeleton:

```rust
unsafe fn f(mut value: Option<i32>) -> i32 {
    #[proctor(0)]
    let Some(mut x): Option<i32> = todo!() else {
        #[proctor(1)]
        return todo!();
    };
    #[proctor(2)]
    todo!()
}
```

Transformation:

```rust
unsafe fn f(value: Option<i32>) -> i32 {
    #[proctor(0)]
    let x: i32 = value.unwrap();
    #[proctor(1)]
    if value.is_none() {
        return 0;
    }
    #[proctor(2)]
    x
}
```

Expected code is `let_else_shape_mismatch` at label 0. The moved label 1 error
is suppressed as derivative.

## 10. Existing-declaration and target-local-type tests

### P2-BIND-01 `binding_mutability_is_ignored_everywhere`

Expected skeleton:

```rust
unsafe fn f(mut pair: (i32, i32), mut value: Option<i32>) -> i32 {
    #[proctor(0)]
    let (mut a, mut b) = todo!();
    #[proctor(1)]
    if let Some(mut x) = todo!() {
        #[proctor(2)]
        return todo!();
    }
    #[proctor(3)]
    todo!()
}
```

Transformation:

```rust
unsafe fn f(mut pair: (i32, i32), value: Option<i32>) -> i32 {
    #[proctor(0)]
    let (a, mut b) = pair;
    #[proctor(1)]
    if let Some(x) = value {
        #[proctor(2)]
        return x + b;
    }
    #[proctor(3)]
    a
}
```

Expected status is `valid`.

### P2-BIND-02 `existing_binding_must_exist_exactly_once_in_its_group`

Expected skeleton:

```rust
unsafe fn f() -> i32 {
    #[proctor(0)]
    let mut x: i32 = todo!();
    #[proctor(1)]
    todo!()
}
```

Run independently:

```rust
unsafe fn f() -> i32 {
    #[proctor(0)]
    consume(1);
    #[proctor(1)]
    0
}
```

```rust
unsafe fn f() -> i32 {
    #[proctor(0)]
    let x: i32 = 1;
    #[proctor(0)]
    let x: i32 = 2;
    #[proctor(1)]
    x
}
```

Expected codes are `missing_existing_binding` and
`duplicate_existing_binding`.

### P2-BIND-03 `existing_binding_cannot_move_to_another_label`

Expected skeleton:

```rust
unsafe fn f() -> i32 {
    #[proctor(0)]
    let mut x: i32 = todo!();
    #[proctor(1)]
    todo!()
}
```

Transformation:

```rust
unsafe fn f() -> i32 {
    #[proctor(0)]
    consume(1);
    #[proctor(1)]
    let x: i32 = 1;
    #[proctor(1)]
    x
}
```

Expected code is `existing_binding_location_mismatch`, identifying expected
label 0 and observed label 1. Do not also report `x` as both missing and new.

### P2-BIND-04 `pattern_binding_names_stay_in_their_structural_roles`

Expected skeleton:

```rust
unsafe fn f(mut value: Option<(i32, i32)>) -> i32 {
    #[proctor(0)]
    match todo!() {
        Some((mut left, mut right)) => {
            #[proctor(1)]
            todo!()
        }
        None => {
            #[proctor(2)]
            todo!()
        }
    }
}
```

Transformation:

```rust
unsafe fn f(value: Option<(i32, i32)>) -> i32 {
    #[proctor(0)]
    match value {
        Some((right, left)) => {
            #[proctor(1)]
            left + right
        }
        None => {
            #[proctor(2)]
            0
        }
    }
}
```

Expected two `existing_binding_location_mismatch` errors because swapping
spellings swaps declaration identities within the tuple pattern.

Validate slice positions with this independent request.

Expected skeleton:

```rust
unsafe fn slice_positions(mut values: [i32; 3]) -> i32 {
    #[proctor(0)]
    let [mut head, .., mut tail] = todo!();
    #[proctor(1)]
    todo!()
}
```

Valid transformation:

```rust
unsafe fn slice_positions(values: [i32; 3]) -> i32 {
    #[proctor(0)]
    let [head, .., tail] = values;
    #[proctor(1)]
    head + tail
}
```

Invalid transformation:

```rust
unsafe fn slice_positions(values: [i32; 3]) -> i32 {
    #[proctor(0)]
    let [tail, .., head] = values;
    #[proctor(1)]
    head + tail
}
```

The invalid result has two `existing_binding_location_mismatch` errors for the
slice-prefix and slice-suffix declaration roles.

Validate `@` binder/subpattern roles with this independent request.

Expected skeleton:

```rust
unsafe fn at_roles(mut value: (i32, i32)) -> i32 {
    #[proctor(0)]
    let mut whole @ (mut inner, _) = todo!();
    #[proctor(1)]
    todo!()
}
```

Valid transformation:

```rust
unsafe fn at_roles(value: (i32, i32)) -> i32 {
    #[proctor(0)]
    let whole @ (inner, _) = value;
    #[proctor(1)]
    whole.0 + inner
}
```

Invalid transformation:

```rust
unsafe fn at_roles(value: (i32, i32)) -> i32 {
    #[proctor(0)]
    let (whole @ inner, _) = value;
    #[proctor(1)]
    whole + inner
}
```

The invalid result has two `existing_binding_location_mismatch` errors: an
`@` binder and its subpattern are distinct positions.

Validate struct fields and `or`-pattern declaration equivalence with this
independent request.

Expected skeleton:

```rust
unsafe fn or_fields(mut value: E) {
    #[proctor(0)]
    match todo!() {
        E::A {
            left: mut x,
            right: mut y,
        }
        | E::B {
            left: mut x,
            right: mut y,
        } => {
            #[proctor(1)]
            todo!();
        }
    }
}
```

Valid transformation:

```rust
unsafe fn or_fields(value: E) {
    #[proctor(0)]
    match value {
        E::A { left: x, right: y } | E::B { left: x, right: y } => {
            #[proctor(1)]
            consume(x + y);
        }
    }
}
```

Invalid transformation:

```rust
unsafe fn or_fields(value: E) {
    #[proctor(0)]
    match value {
        E::A { left: y, right: x } | E::B { left: y, right: x } => {
            #[proctor(1)]
            consume(x + y);
        }
    }
}
```

The valid result must not report either `x` or `y` as duplicated: occurrences
across alternatives of one valid `or` pattern form one declaration identity.
The invalid result has two `existing_binding_location_mismatch` errors for the
swapped struct-field roles, not four duplicate-binding errors.

Validate reference and parenthesized pattern wrappers with this independent
request.

Expected skeleton:

```rust
unsafe fn wrapper_paths(mut pair: &(i32, i32)) -> i32 {
    #[proctor(0)]
    let &(mut left, (mut right)) = todo!();
    #[proctor(1)]
    todo!()
}
```

Valid transformation:

```rust
unsafe fn wrapper_paths(pair: &(i32, i32)) -> i32 {
    #[proctor(0)]
    let &(left, (right)) = pair;
    #[proctor(1)]
    left + right
}
```

Invalid transformation:

```rust
unsafe fn wrapper_paths(pair: &(i32, i32)) -> i32 {
    #[proctor(0)]
    let (left, right) = *pair;
    #[proctor(1)]
    left + right
}
```

The invalid result reports `existing_binding_location_mismatch` for both
bindings because removing reference/parenthesized wrappers changes their
complete pattern-child paths.

Validate binding mode independently of position and `mut` with this request.
This parser/validator capability is intentionally tested even though source
programs containing `ref` bindings remain outside the prototype's conceptual
supported-input model.

Expected skeleton:

```rust
unsafe fn binding_modes(mut pair: (i32, i32)) -> i32 {
    #[proctor(0)]
    let (ref mut left, mut right) = todo!();
    #[proctor(1)]
    todo!()
}
```

Valid transformation:

```rust
unsafe fn binding_modes(pair: (i32, i32)) -> i32 {
    #[proctor(0)]
    let (ref left, right) = pair;
    #[proctor(1)]
    *left + right
}
```

Run independently:

```rust
unsafe fn binding_modes(pair: (i32, i32)) -> i32 {
    #[proctor(0)]
    let (left, right) = pair;
    #[proctor(1)]
    left + right
}
```

```rust
unsafe fn binding_modes(pair: (i32, i32)) -> i32 {
    #[proctor(0)]
    let (ref left, ref right) = pair;
    #[proctor(1)]
    *left + *right
}
```

Each invalid transformation produces one
`existing_binding_mode_mismatch`: the first removes required `ref` from
`left`, while the second adds `ref` to by-value binding `right`. Messages name
the binding and label 0, render expected and observed modes, and say that
binding `mut` may differ.

### P2-BIND-05 `target_local_type_is_structural_and_required`

Expected skeleton:

```rust
unsafe fn f() {
    #[proctor(0)]
    let mut p: Option<&mut [i32]> = todo!();
}
```

Run independently:

```rust
unsafe fn f() {
    #[proctor(0)]
    let p: Option<&[i32]> = None;
}
```

```rust
unsafe fn f() {
    #[proctor(0)]
    let p = None::<&mut [i32]>;
}
```

Expected codes are `local_type_mismatch` and
`local_type_presence_mismatch`. Both messages identify binding `p` and label
0.

### P2-BIND-06 `target_absence_of_local_type_is_preserved`

Expected skeleton:

```rust
unsafe fn f(mut pair: (i32, i32)) {
    #[proctor(0)]
    let (mut x, mut y) = todo!();
}
```

Transformation:

```rust
unsafe fn f(pair: (i32, i32)) {
    #[proctor(0)]
    let (x, y): (i32, i32) = pair;
}
```

Expected code is `local_type_presence_mismatch`: the target destructuring
declaration has no explicit local type.

### P2-BIND-07 `same_spelling_bindings_in_distinct_scopes_keep_identity`

Expected skeleton:

```rust
unsafe fn f(mut a: bool, mut b: bool) {
    #[proctor(0)]
    if todo!() {
        #[proctor(1)]
        let mut x: i32 = todo!();
    }
    #[proctor(2)]
    if todo!() {
        #[proctor(3)]
        let mut x: i32 = todo!();
    }
}
```

Transformation:

```rust
unsafe fn f(a: bool, b: bool) {
    #[proctor(0)]
    if a {
        #[proctor(1)]
        let x: i32 = 1;
    }
    #[proctor(2)]
    if b {
        #[proctor(3)]
        consume(2);
    }
}
```

Expected one `missing_existing_binding` for the identity at label 3. The
binding at label 1 does not satisfy it.

### P2-BIND-08 `nested_items_are_preserved_and_new_items_are_rejected`

Expected skeleton:

```rust
unsafe fn f() -> i32 {
    #[proctor(0)]
    const LOCAL: i32 = 1;
    #[proctor(1)]
    static mut STATE: i32 = 2;
    #[proctor(2)]
    todo!()
}
```

Valid transformation:

```rust
unsafe fn f() -> i32 {
    #[proctor(0)]
    const LOCAL: i32 = 1;
    #[proctor(1)]
    static mut STATE: i32 = 2;
    #[proctor(2)]
    LOCAL + STATE
}
```

The result is valid. After recursively removing `proctor` labels—which are
validated separately—and ignoring AST metadata and formatting, both local
items are structurally identical to their target declarations.

Also validate the global binding-mutability exception inside an exact local
item with this independent request.

Expected skeleton:

```rust
unsafe fn g() -> i32 {
    #[proctor(0)]
    const LOCAL: i32 = {
        #[proctor(1)]
        let mut inner = 1;
        #[proctor(2)]
        inner
    };
    #[proctor(3)]
    todo!()
}
```

Transformation:

```rust
unsafe fn g() -> i32 {
    #[proctor(0)]
    const LOCAL: i32 = {
        #[proctor(1)]
        let inner = 1;
        #[proctor(2)]
        inner
    };
    #[proctor(3)]
    LOCAL
}
```

This result is valid. Binding `mut` is ignored even inside a local-item
initializer; item-level `static mut` is not ignored.

Against the same `g` skeleton, this transformation is invalid:

```rust
unsafe fn g() -> i32 {
    #[proctor(0)]
    const LOCAL: i32 = {
        #[proctor(1)]
        let inner = 1;
        #[proctor(1)]
        let proctor_temp_var_0 = 2;
        #[proctor(2)]
        inner
    };
    #[proctor(3)]
    LOCAL
}
```

It produces `existing_item_structure_mismatch`. Although label 1 would
normally be an expansion group, exact local-item initialization overrides
expansion for labels nested inside the initializer. Do not also reject the
well-named generated binding independently.

Validate expected non-`proctor` item attributes with this independent request.

Expected skeleton:

```rust
unsafe fn h() -> i32 {
    #[proctor(0)]
    #[allow(dead_code)]
    const LOCAL: i32 = 1;
    #[proctor(1)]
    todo!()
}
```

This exact preservation is valid:

```rust
unsafe fn h() -> i32 {
    #[proctor(0)]
    #[allow(dead_code)]
    const LOCAL: i32 = 1;
    #[proctor(1)]
    LOCAL
}
```

Run these independently:

```rust
unsafe fn h() -> i32 {
    #[proctor(0)]
    const LOCAL: i32 = 1;
    #[proctor(1)]
    LOCAL
}
```

```rust
unsafe fn h() -> i32 {
    #[proctor(0)]
    #[allow(unused_variables)]
    const LOCAL: i32 = 1;
    #[proctor(1)]
    LOCAL
}
```

Both invalid transformations produce `existing_item_structure_mismatch`,
identifying a removed or changed expected item attribute. They do not produce
`unexpected_body_attribute`.

Run the following transformations independently.

Missing existing item:

```rust
unsafe fn f() -> i32 {
    #[proctor(0)]
    consume(1);
    #[proctor(1)]
    static mut STATE: i32 = 2;
    #[proctor(2)]
    STATE
}
```

Unexpected new local const:

```rust
unsafe fn f() -> i32 {
    #[proctor(0)]
    const LOCAL: i32 = 1;
    #[proctor(0)]
    const EXTRA: i32 = 3;
    #[proctor(1)]
    static mut STATE: i32 = 2;
    #[proctor(2)]
    LOCAL + EXTRA
}
```

Duplicate existing item:

```rust
unsafe fn f() -> i32 {
    #[proctor(0)]
    const LOCAL: i32 = 1;
    #[proctor(0)]
    const LOCAL: i32 = 2;
    #[proctor(1)]
    static mut STATE: i32 = 2;
    #[proctor(2)]
    LOCAL + STATE
}
```

Existing item moved to another expansion group:

```rust
unsafe fn f() -> i32 {
    #[proctor(0)]
    consume(1);
    #[proctor(1)]
    static mut STATE: i32 = 2;
    #[proctor(2)]
    const LOCAL: i32 = 1;
    #[proctor(2)]
    LOCAL + STATE
}
```

Changed const initializer:

```rust
unsafe fn f() -> i32 {
    #[proctor(0)]
    const LOCAL: i32 = 9;
    #[proctor(1)]
    static mut STATE: i32 = 2;
    #[proctor(2)]
    LOCAL + STATE
}
```

Changed const declared type:

```rust
unsafe fn f() -> i32 {
    #[proctor(0)]
    const LOCAL: i64 = 1;
    #[proctor(1)]
    static mut STATE: i32 = 2;
    #[proctor(2)]
    STATE
}
```

Changed static mutability:

```rust
unsafe fn f() -> i32 {
    #[proctor(0)]
    const LOCAL: i32 = 1;
    #[proctor(1)]
    static STATE: i32 = 2;
    #[proctor(2)]
    LOCAL + STATE
}
```

Changed static initializer:

```rust
unsafe fn f() -> i32 {
    #[proctor(0)]
    const LOCAL: i32 = 1;
    #[proctor(1)]
    static mut STATE: i32 = 7;
    #[proctor(2)]
    LOCAL + STATE
}
```

Changed static declared type:

```rust
unsafe fn f() -> i32 {
    #[proctor(0)]
    const LOCAL: i32 = 1;
    #[proctor(1)]
    static mut STATE: i64 = 2;
    #[proctor(2)]
    LOCAL
}
```

Changed item kind with the same declaration name:

```rust
unsafe fn f() -> i32 {
    #[proctor(0)]
    static LOCAL: i32 = 1;
    #[proctor(1)]
    static mut STATE: i32 = 2;
    #[proctor(2)]
    LOCAL + STATE
}
```

Added local-item attribute:

```rust
unsafe fn f() -> i32 {
    #[proctor(0)]
    #[allow(dead_code)]
    const LOCAL: i32 = 1;
    #[proctor(1)]
    static mut STATE: i32 = 2;
    #[proctor(2)]
    LOCAL + STATE
}
```

Unexpected new local function:

```rust
unsafe fn f() -> i32 {
    #[proctor(0)]
    const LOCAL: i32 = 1;
    #[proctor(1)]
    static mut STATE: i32 = 2;
    #[proctor(1)]
    fn extra() {}
    #[proctor(2)]
    LOCAL + STATE
}
```

Expected codes are, respectively:

```text
missing_existing_item
unexpected_nested_item
duplicate_existing_item
existing_item_location_mismatch
existing_item_structure_mismatch
existing_item_structure_mismatch
existing_item_structure_mismatch
existing_item_structure_mismatch
existing_item_structure_mismatch
existing_item_structure_mismatch
existing_item_structure_mismatch
unexpected_nested_item
```

The location mismatch identifies expected label 0 and observed label 2 and
suppresses derivative `missing_existing_item`/`unexpected_nested_item` errors
for `LOCAL`. Each structure-mismatch message identifies the item and label,
names the differing initializer, type, `static mut` status, item kind, or
attribute list, and prints the expected and observed forms. The same-named
`static LOCAL` is associated with expected `const LOCAL` before structure
comparison, so it does not become a missing-plus-unexpected pair. Complete
parsed structure is exact: an LLM may not modify an existing local `const` or
`static` initializer. Every new local item is rejected, including otherwise
supported `const` and `static` kinds and prohibited source-input kinds such as
a local function.

## 11. Generated-temporary, unsafe-block, and attribute tests

### P2-TEMP-01 `temporaries_are_local_to_one_expansion_group`

Expected skeleton:

```rust
unsafe fn f(mut p: Option<&i32>) -> i32 {
    #[proctor(0)]
    let mut x: i32 = todo!();
    #[proctor(1)]
    todo!()
}
```

Transformation:

```rust
unsafe fn f(p: Option<&i32>) -> i32 {
    #[proctor(0)]
    let proctor_temp_var_2 = p.unwrap();
    #[proctor(0)]
    let proctor_temp_var_9 = if *proctor_temp_var_2 > 0 {
        *proctor_temp_var_2
    } else {
        0
    };
    #[proctor(0)]
    let x: i32 = proctor_temp_var_9;
    #[proctor(1)]
    x
}
```

Expected status is `valid`. Suffixes need not start at zero or be contiguous.

### P2-TEMP-02 `new_binding_names_and_temporary_declarations_are_strict`

Expected skeleton:

```rust
unsafe fn f() {
    #[proctor(0)]
    todo!();
}
```

Run independently:

```rust
unsafe fn f() {
    #[proctor(0)]
    let helper = 1;
}
```

```rust
unsafe fn f() {
    #[proctor(0)]
    let proctor_temp_var_x = 1;
}
```

```rust
unsafe fn f() {
    #[proctor(0)]
    let proctor_temp_var_0 = 1;
    {
        let proctor_temp_var_0 = 2;
        consume(proctor_temp_var_0);
    }
}
```

```rust
unsafe fn f() {
    #[proctor(0)]
    if let Some(helper) = Some(1) {
        consume(helper);
    }
}
```

```rust
unsafe fn f() {
    #[proctor(0)]
    let (helper, proctor_temp_var_0) = (1, 2);
}
```

```rust
unsafe fn f() {
    #[proctor(0)]
    while let Some(helper) = None::<i32> {
        consume(helper);
    }
}
```

```rust
unsafe fn f() {
    #[proctor(0)]
    for helper in [1] {
        consume(helper);
    }
}
```

```rust
unsafe fn f() {
    #[proctor(0)]
    match Some(1) {
        Some(helper) => {
            consume(helper);
        }
        None => {}
    }
}
```

```rust
unsafe fn f() {
    #[proctor(0)]
    let proctor_temp_var_0 = |helper| helper;
}
```

Every transformation except the duplicate-shadowing transformation produces
`invalid_generated_binding_name`; the duplicate-shadowing transformation
produces `duplicate_generated_temporary`. The message for a bad name gives the
exact `proctor_temp_var_n` grammar. Together the cases cover simple and
destructuring `let`, `if let`, `while let`, `for`, match-arm, and closure
parameter bindings.

Also validate collision with an existing target binding using this independent
request.

Expected skeleton:

```rust
unsafe fn g() -> i32 {
    #[proctor(0)]
    let mut proctor_temp_var_0: i32 = todo!();
    #[proctor(1)]
    todo!()
}
```

Transformation:

```rust
unsafe fn g() -> i32 {
    #[proctor(0)]
    let proctor_temp_var_0: i32 = 1;
    #[proctor(1)]
    let proctor_temp_var_0 = 2;
    #[proctor(1)]
    proctor_temp_var_0
}
```

Expected code is `invalid_generated_binding_name`; the message explains that a
new binding may not reuse an existing target binding's spelling.

### P2-TEMP-03 `temporary_reference_cannot_cross_group_boundary`

Expected skeleton:

```rust
unsafe fn f() {
    #[proctor(0)]
    todo!();
    #[proctor(1)]
    todo!();
}
```

Transformation:

```rust
unsafe fn f() {
    #[proctor(0)]
    let proctor_temp_var_0 = 1;
    #[proctor(1)]
    consume(proctor_temp_var_0);
}
```

Expected code is `temporary_outside_expansion_group`, naming declaration label
0 and reference label 1.

Run this transformation independently against the same skeleton:

```rust
unsafe fn f() {
    #[proctor(0)]
    consume(proctor_temp_var_7);
    #[proctor(1)]
    return;
}
```

Expected code is `unresolved_generated_temporary`: the complete identifier
matches the generated-temporary spelling but resolves to neither an expected
existing local binding nor a generated-temporary declaration in `f`.

### P2-TEMP-04 `temporary_identifier_in_macro_tokens_is_rejected`

Expected skeleton:

```rust
unsafe fn f() {
    #[proctor(0)]
    todo!();
}
```

Transformation:

```rust
unsafe fn f() {
    #[proctor(0)]
    let proctor_temp_var_0 = 1;
    #[proctor(0)]
    println!("{}", proctor_temp_var_0);
}
```

Expected code is `temporary_in_macro`. The message instructs the LLM to use
ordinary Rust syntax outside the macro token tree. A macro without a generated
temporary remains allowed, as confirmed independently by:

```rust
unsafe fn f() {
    #[proctor(0)]
    println!("ok");
}
```

which has status `valid` against the same skeleton.

### P2-SAFE-01 `explicit_unsafe_block_is_rejected_at_any_depth`

Expected skeleton:

```rust
unsafe fn f(mut p: *const i32) -> i32 {
    #[proctor(0)]
    todo!()
}
```

Transformation:

```rust
unsafe fn f(p: *const i32) -> i32 {
    #[proctor(0)]
    if p.is_null() {
        0
    } else {
        unsafe {
            *p
        }
    }
}
```

Expected code is `explicit_unsafe_block`, even though the block is inside new
unlabeled code. The failure is item-specific with ID 7, name `f`, and the
complete pretty-printed matched function as `failed_snippet`. Its message
identifies function `f`, item 7, and label 0. Direct `*p` without an unsafe
block is structurally allowed.

### P2-SAFE-02 `new_statement_or_expression_attributes_are_rejected`

Expected skeleton:

```rust
unsafe fn f() {
    #[proctor(0)]
    todo!();
}
```

Run independently:

```rust
unsafe fn f() {
    #[proctor(0)]
    #[allow(unused_variables)]
    let proctor_temp_var_0 = 1;
}
```

```rust
unsafe fn f() {
    #[proctor(0)]
    {
        #[allow(unused_variables)]
        let proctor_temp_var_0 = 1;
    }
}
```

Both produce `unexpected_body_attribute`. The first identifies a labeled
statement; the second finds the attribute in newly introduced nested code.
This code applies to statement/expression attributes. A non-`proctor`
attribute added to or changed on an existing local `const`/`static` is instead
part of item-structure comparison and produces
`existing_item_structure_mismatch`, as covered by P2-BIND-08.

## 12. Diagnostic and integration tests

### P2-DIAG-01 `parent_failure_suppresses_dependent_cascade`

Expected skeleton:

```rust
unsafe fn f(mut flag: bool) {
    #[proctor(0)]
    if todo!() {
        #[proctor(1)]
        let mut x: i32 = todo!();
    } else {
        #[proctor(2)]
        let mut y: i32 = todo!();
    }
}
```

Transformation:

```rust
unsafe fn f(flag: bool) {
    #[proctor(0)]
    loop {
        #[proctor(2)]
        let y: u32 = 1;
    }
}
```

Return only the reliable parent `control_kind_mismatch` for label 0. Do not
also report missing label 1, moved label 2, missing `x`, or wrong type for `y`,
because the result no longer has trustworthy branch association.

### P2-DIAG-02 `independent_errors_are_aggregated_in_stable_order`

Expected skeleton:

```rust
unsafe fn f(mut p: &i32) -> i32 {
    #[proctor(0)]
    let mut x: i32 = todo!();
    #[proctor(1)]
    todo!()
}
```

Transformation:

```rust
unsafe fn f(p: *const i32) -> i32 {
    #[proctor(0)]
    let x: u32 = 1;
    #[proctor(2)]
    0
}
```

Expected item-error order is:

```text
parameter_type_mismatch
local_type_mismatch
missing_label
unexpected_label
```

Messages contain `f`, item 7, and the relevant parameter, binding, or label.
Repeated runs produce byte-identical output.

### P2-DIAG-03 `deep_multi_function_request_reports_only_failing_items`

Expected entry `(7,"leaf")`:

```rust
unsafe fn leaf(mut p: Option<&i32>) -> i32 {
    #[proctor(0)]
    let mut x: i32 = todo!();
    #[proctor(1)]
    todo!()
}
```

Expected entry `(8,"driver")`:

```rust
unsafe fn driver(mut flag: bool, mut p: Option<&i32>) -> i32 {
    #[proctor(0)]
    let mut result: i32 = if todo!() {
        #[proctor(1)]
        todo!()
    } else {
        #[proctor(2)]
        todo!()
    };
    #[proctor(3)]
    match todo!() {
        0 => {
            #[proctor(4)]
            todo!()
        }
        _ => {
            #[proctor(5)]
            todo!()
        }
    }
}
```

Transformation:

```rust
unsafe fn driver(flag: bool, p: Option<&i32>) -> i32 {
    #[proctor(0)]
    let result: i32 = if flag {
        #[proctor(2)]
        0
    } else {
        #[proctor(1)]
        leaf(p)
    };
    #[proctor(3)]
    match result {
        0 => {
            #[proctor(4)]
            0
        }
        _ => {
            #[proctor(5)]
            result
        }
    }
}

unsafe fn leaf(p: Option<&i32>) -> i32 {
    #[proctor(0)]
    let x: i32 = *p.unwrap();
    #[proctor(1)]
    x
}
```

Only `driver` appears in `failures`, with descendant-location errors for
labels 1 and 2. `leaf` is valid despite result order and removed binding
mutability. Failure ordering follows request order, not result order.

## 13. Stable initial error-code set

The initial implementation and tests use these codes without repurposing:

```text
invalid_request_json
unsupported_schema_version
unknown_request_field
empty_expected_functions
duplicate_expected_id
duplicate_expected_name
expected_skeleton_parse_error
expected_skeleton_item_count
expected_skeleton_name_mismatch
invalid_expected_skeleton
result_parse_error
missing_function
duplicate_function
unexpected_function
unexpected_item
parameter_count_mismatch
parameter_name_mismatch
parameter_type_mismatch
return_type_mismatch
missing_label
unexpected_label
label_order_mismatch
nonconsecutive_label
unlabeled_group_statement
nested_label_repetition
malformed_label
misplaced_label
control_kind_mismatch
control_role_mismatch
branch_shape_mismatch
match_arm_shape_mismatch
match_guard_mismatch
descendant_location_mismatch
missing_control_root
multiple_control_roots
let_else_shape_mismatch
missing_existing_binding
duplicate_existing_binding
existing_binding_location_mismatch
existing_binding_mode_mismatch
local_type_mismatch
local_type_presence_mismatch
missing_existing_item
duplicate_existing_item
existing_item_location_mismatch
existing_item_structure_mismatch
unexpected_nested_item
invalid_generated_binding_name
duplicate_generated_temporary
unresolved_generated_temporary
temporary_outside_expansion_group
temporary_in_macro
explicit_unsafe_block
unexpected_body_attribute
```

`descendant_location_mismatch` includes a known label found in the wrong
branch, match arm, `let-else` body, plain-block body, loop body, or other
preserved structural statement list. It replaces a derivative
missing-plus-unexpected-label pair. Same-category ordering and cascade
suppression follow the table in `prototype-plan.md` Section 14.8.
`existing_item_structure_mismatch` is used only after a unique local `const` or
`static` has been associated with its expected location; it reports a
structural difference without also treating that declaration as missing or
new.

## 14. Completion criteria

Phase 2 is complete when:

- the completed Phase 1 implementation and tests have been moved into the
  `skeleton` module without behavior changes other than Section 4;
- `lib.rs` contains only required crate-level attributes/`extern crate`
  declarations, module declarations, and public re-exports, with no
  implementation or large tests;
- all four Phase 1 generator adjustments and all affected existing Rust
  oracles are implemented;
- safe source functions retain safe source renderings but receive unsafe target
  headers;
- every free function named `main` is omitted without body inspection;
- the supported two-argument `main_0` receives exactly the forced
  `&mut [&mut [i8]]` target `argv` type;
- all 69 cases in this document pass;
- no Phase 2 test performs filesystem I/O or subprocess invocation;
- validator tests require no `TyCtxt` and do not type-check returned snippets;
- no Phase 2 validator behavior is added specifically for function safety or
  executable entry-point handling;
- expected-skeleton setup accepts only function-local `const`/`static` items,
  without another Phase 1 generator change;
- existing local `const`/`static` items are structurally exact and every new
  local item is rejected;
- labels use only canonical `0|[1-9][0-9]*` tokens in the `u32` range;
- explicit unsafe blocks produce item-specific failures;
- setup, global, and item failures follow the specified precedence and cascade
  suppression;
- messages are actionable and include all available identity and structural
  context;
- response serialization is deterministic and byte-stable;
- `crat-tool validate` implements the documented request/response and exit
  behavior, although its filesystem wiring is not tested here; and
- from `crat/`, `cargo fmt`, `cargo clippy --workspace --all-targets`, and
  `cargo test --workspace` pass.
