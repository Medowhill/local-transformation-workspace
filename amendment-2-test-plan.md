# Amendment 2 Test Plan: Conservative Statement Preservation

## 1. Purpose

This document specifies the complete regression suite for Amendment Plan 2 in
`prototype-plan.md`. It covers conservative statement classification,
preservation-aware skeleton JSON, canonical validation and replacement,
the final unreleased version-1 protocols and prompt text, and SCCs that require
no LLM request.

The historical phase test plans are not edited. Existing implementation tests
whose old expected holes or earlier unreleased version-1 protocol shapes
conflict with Amendment 2 are updated as specified here, and the new cases
below are added. No schema or prompt version is incremented before the first
release.

The suite contains 45 named cases:

| Area | Cases |
| --- | ---: |
| Existing regression updates and JSON | 5 |
| Preservation classification and skeletons | 18 |
| Validator canonicalization | 8 |
| Replacement and integration | 6 |
| Python orchestration and prompt | 8 |

## 2. Test execution and ownership policy

Rust semantic tests live beside the existing Crat tools modules and run with:

```bash
cd proctor/stages/crat
cargo test --workspace
```

Skeleton classification tests use exact source strings with
`run_compiler_on_str`. Validator and request-shape tests remain parser-only
where rustc resolution is unnecessary. Replacement tests use the existing
in-memory compiler helper. They create no fixture files and invoke no nested
Cargo process.

Python integration tests live with the existing local-transformation stage
tests and run through the stage's `uv` environment. They use temporary
stage-owned directories, fake Crat tools, fake builds, and fake LLM clients.
They remain offline and never invoke the real Crat toolchain, Cargo, or a
provider API.

No test edits any historical test-plan file. No case changes
`configs/full_pipeline.toml`.

Existing implementation fixtures are updated by this explicit matrix:

- Rust skeleton goldens replace blanket holes only where the classifier proves
  preservation and add both metadata fields to `Fn` records.
- Validator and replacer request fixtures add the final version-1
  preservation metadata; response fixtures retain their version-1 shape.
- Python skeleton-record helpers require the new fields. The general
  `fn_record` helper defaults to `needs_transformation = true`,
  `statements_requiring_transformation = [0]`, and its existing one-label hole
  skeleton, so historical orchestration tests continue exercising the LLM
  path unless a test explicitly opts into mechanical preservation.
- The version-1 prompt golden changes only by the Amendment 2 paragraph.
- Existing event and metrics fixtures keep their transforming defaults; only
  the Amendment 2 cases below assert the no-LLM path.

## 3. Comparison policy and exact shared Rust inputs

Structured AST comparison ignores formatting, spans, node IDs, token caches,
and presentation-only binding `mut` unless a case explicitly tests rendered
text. Labels, statement dispositions, types, item identity, control roles, and
ordering are exact.

When a case references a shared input below, it uses that exact source
byte-for-byte. Every other case prints its complete Rust input locally.

### A2-SRC-SCALAR

```rust
pub unsafe fn scalar(mut x: i32, y: i32, z: i32) -> i32 {
    let sum = y + z;
    x = sum * 2;
    return x;
}
```

### A2-SRC-MIXED-CONTROL

```rust
pub unsafe fn mixed(mut p: *mut i32, flag: bool, y: i32, z: i32) -> i32 {
    if flag {
        let sum = y + z;
        *p = sum;
    } else {
        let difference = y - z;
        return difference;
    }
    return y + z;
}
```

### A2-SRC-LOCAL-ADTS

```rust
pub struct Leaf {
    pub pointer: *mut i32,
}
pub struct Middle {
    pub leaf: Leaf,
}
pub enum Choice {
    Empty,
    Value(Middle),
}
pub union Storage {
    pub leaf: core::mem::ManuallyDrop<Leaf>,
    pub integer: i64,
}
pub struct Link {
    pub next: Option<Box<Link>>,
    pub leaf: Leaf,
}
pub type Alias = Choice;

pub unsafe fn move_values(
    mut a: Middle,
    b: Middle,
    mut c: Alias,
    d: Alias,
    mut s: Storage,
    t: Storage,
    mut link: Link,
    other: Link,
) {
    a = b;
    c = d;
    s = t;
    link = other;
}
```

### A2-SRC-SIGNATURE-SCC

```rust
pub unsafe fn callee(_pointer: *mut i32, value: i32) -> i32 {
    value + 1
}

pub unsafe fn caller(pointer: *mut i32, value: i32) -> i32 {
    callee(pointer, value)
}
```

### A2-SRC-PRESERVED-WRAPPER

```rust
#[no_mangle]
pub unsafe extern "C" fn unused_pointer(pointer: *mut i32, value: i32) -> i32 {
    let doubled = value * 2;
    doubled + 1
}
```

### A2-SRC-RESTRICTED-CONDITIONAL

```rust
pub unsafe fn conditional(mut x: i32, flag: bool, pointer: *mut i32) -> i32 {
    x = 1 + if flag { 2 } else { 3 };
    x = 1 + if flag { *pointer } else { 3 };
    x
}
```

### A2-SRC-VALIDATOR

```rust
pub unsafe fn validate_me(flag: bool, mut pointer: *mut i32) -> i32 {
    let scalar = 1 + 2;
    if flag {
        let nested = 3 + 4;
        *pointer = nested;
    } else {
        return scalar;
    }
    scalar
}
```

## 4. Existing regression updates and JSON contract

### A2-UPDATE-01 `existing_scalar_hole_expectations_become_canonical_statements`

Use exact input A2-SRC-SCALAR. Update existing skeleton-shape assertions that
previously expected all scalar payloads to be `todo!()`. The exact amended
target skeleton is structurally:

```rust
pub unsafe fn scalar(mut x: i32, mut y: i32, mut z: i32) -> i32 {
    #[proctor(0)]
    let mut sum: i32 = y + z;
    #[proctor(1)]
    x = sum * 2;
    #[proctor(2)]
    return x;
}
```

Expected metadata:

```text
needs_transformation = false
statements_requiring_transformation = []
```

The annotated source still uses its ordinary source signature, subject to the
existing presentation mutability normalization.

### A2-UPDATE-02 `existing_control_tests_preserve_only_proven_complete_subtrees`

Use exact input A2-SRC-MIXED-CONTROL. Update old blanket-hole expectations
without weakening their label-order and control-shape assertions. Expected
preorder labels are:

```text
0 outer if
1 then let sum
2 pointer write
3 else let difference
4 else return
5 final return
```

Labels `1`, `3`, `4`, and `5` render their canonical complete statements.
Labels `0` and `2` require transformation: label 2 directly uses a raw
pointer, and label 0 is not preservable because its complete subtree contains
label 2. The outer `if` retains its existing control skeleton and condition
hole.

Also update payloadless-control regressions with this exact input:

```rust
pub unsafe fn empty() {}

pub unsafe fn flow(flag: bool) {
    if flag {
        return;
    }
    loop {
        break;
    }
}
```

Both records have `needs_transformation = false` and an empty transformation
label array. The empty function has no labels. Every `flow` label is
preserved, including payloadless `return;` and `break;`.

### A2-JSON-01 `function_record_has_exact_amended_key_order`

Use exact input A2-SRC-SCALAR. The exact ordered function-record keys are:

```text
id
path
kind
name
annotated_source
annotated_skeleton
source_signature
target_signature
needs_transformation
statements_requiring_transformation
signature_dependencies
dependencies
```

The Boolean is JSON `false`, the statement array is `[]`, and neither field is
emitted for non-function records.

### A2-JSON-02 `transformation_labels_are_sorted_unique_and_boolean_is_derived`

Use exact input A2-SRC-MIXED-CONTROL. Expected:

```json
{
  "needs_transformation": true,
  "statements_requiring_transformation": [0, 2]
}
```

Generation produces strictly increasing unique `u32` labels and
`needs_transformation == !statements_requiring_transformation.is_empty()`.

### A2-JSON-03 `every_label_has_one_disposition_and_preserved_parent_is_closed`

Use exact input A2-SRC-MIXED-CONTROL. Parse `annotated_skeleton`, collect every
label, and prove that the transform-label set is a subset of it. Every label
not in the set is preserved. No preserved label has a transformed descendant.
Repeat with A2-SRC-SCALAR, where every label is preserved.

## 5. Preservation classification and skeleton tests

### A2-PRES-01 `preserves_scalar_lvalues_rvalues_casts_and_aggregates`

Exact Rust input:

```rust
pub unsafe fn arithmetic(mut x: i32, y: i32, z: i32) -> (i64, [i32; 2]) {
    x = y + z;
    x += 1;
    let wide = x as i64;
    let pair = (wide, [y, z]);
    return pair;
}
```

Every label is preserved. The skeleton contains the complete assignment,
compound assignment, cast, tuple, array, and return expressions and contains
no generated hole.

### A2-PRES-02 `preserves_safe_local_and_safe_nonlocal_calls`

Exact Rust input:

```rust
pub fn local(value: i32) -> i32 {
    value + 1
}

pub unsafe fn caller(values: &[i32], left: i32, right: i32) -> i32 {
    let a = local(left);
    let b = std::cmp::max(a, right);
    let c = values.len() as i32;
    let d = values[0];
    b + c + d
}
```

All statements in both functions are preserved. The local safe call and the
safe non-local `std::cmp::max`, slice-method, and resolved indexing operations
are statically resolved, have unchanged pointer-free signatures, and are not
rejected merely because a callee or trait implementation is non-local.

### A2-PRES-03 `local_unsafe_call_is_allowed_when_signature_is_unchanged`

Exact Rust input:

```rust
pub unsafe fn helper(value: i32) -> i32 {
    value + 1
}

pub unsafe fn caller(value: i32) -> i32 {
    helper(value)
}
```

Every statement is preserved. Target-safety normalization is not treated as a
signature change, and the local unsafe call is not subject to the non-local
unsafe-call prohibition.

### A2-PRES-04 `foreign_call_requires_transformation_without_pointer_types`

Exact Rust input:

```rust
unsafe extern "C" {
    fn foreign_abs(value: i32) -> i32;
}

pub unsafe fn caller(value: i32) -> i32 {
    foreign_abs(value)
}
```

The function-tail call label requires transformation even though every
argument and result is `i32`.

### A2-PRES-05 `unsafe_nonlocal_function_and_method_calls_require_transformation`

Exact Rust input:

```rust
pub unsafe fn caller(values: &[i32]) -> (i32, char) {
    let value = *values.get_unchecked(0);
    let character = char::from_u32_unchecked(65);
    (value, character)
}
```

The two `let` labels require transformation. The final tuple label is
preserved. Resolution may identify the definitions in `core`; surface
spelling through `std` is irrelevant.

### A2-PRES-06 `raw_pointer_in_lvalue_rvalue_and_adjustment_requires_transformation`

Exact Rust input:

```rust
pub unsafe fn pointer_uses(mut left: *mut i32, right: *const i32) -> i32 {
    *left = *right;
    let alias: *const i32 = left;
    let value = *alias;
    value
}
```

The first three labels require transformation. Check unadjusted types,
adjusted types, intermediate pointer-mutability coercion for `left`, and both
place and value expressions. In this exact fixture the binding `value` has
unchanged type `i32`, so the final `value` label is preserved.

Using exact input A2-SRC-SCALAR in a classifier-unit subcase, make the injected
AST-to-HIR lookup omit the `y` expression under label 0. Label 0 must become
`transform`; the classifier may never treat a missing mapping as evidence of
pointer-freedom.

### A2-PRES-07 `declaration_without_initializer_checks_target_binding_type`

Exact Rust input:

```rust
pub unsafe fn declarations() {
    let scalar: i32;
    let pointer: *mut i32;
    scalar = 1;
    pointer = core::ptr::null_mut();
    let _ = scalar;
    let _ = pointer;
}
```

The scalar declaration and scalar-only uses are preserved. The raw-pointer
declaration requires transformation despite having no initializer. Every
statement referencing `pointer`, including its assignment and wildcard use,
requires transformation.

### A2-PRES-08 `nested_generic_type_components_are_opened`

Exact Rust input:

```rust
pub unsafe fn nested(
    mut a: Option<Box<(*mut i32, [usize; 2])>>,
    b: Option<Box<(*mut i32, [usize; 2])>>,
) {
    a = b;
}
```

The assignment requires transformation because recursive generic, box, tuple,
and array traversal reaches `*mut i32`.

Type-only generic arguments use this exact input:

```rust
pub unsafe fn type_arguments() -> usize {
    let marker = core::marker::PhantomData::<*mut i32>;
    let size = core::mem::size_of::<*mut i32>();
    let _ = marker;
    size
}
```

The first three labels require transformation because the explicit generic
argument is raw-pointer-typed even where the runtime result is zero-sized,
`usize`, or discarded. The final `usize` tail is preserved.

### A2-PRES-09 `project_local_struct_enum_union_alias_and_recursive_fields_are_opened`

Use exact input A2-SRC-LOCAL-ADTS. All four assignments require
transformation. The checks reach:

- `Middle -> Leaf -> *mut i32`;
- alias `Alias -> Choice -> Middle`;
- union field `Storage::leaf -> ManuallyDrop<Leaf> -> Leaf`;
- recursive `Link::next` without looping, plus `Link::leaf`.

The external `ManuallyDrop` and `Box` definitions remain opaque, but their
generic arguments are inspected.

Add this low-level type-sensitivity-helper input:

```rust
pub struct Wrap<T> {
    pub value: T,
}

pub struct Recursive<T> {
    pub next: Option<Box<Recursive<T>>>,
    pub value: T,
}

pub unsafe fn generic_values(
    mut scalar: Wrap<i32>,
    other_scalar: Wrap<i32>,
    mut pointer: Wrap<*mut i32>,
    other_pointer: Wrap<*mut i32>,
    recursive_scalar: Recursive<i32>,
    recursive_pointer: Recursive<*mut i32>,
) {
    scalar = other_scalar;
    pointer = other_pointer;
    let _ = recursive_scalar;
    let _ = recursive_pointer;
}
```

Invoke only the compiler-backed type-sensitivity helper, not end-to-end
skeleton generation: source-written generic items remain outside the
prototype's supported-input contract. Require `Wrap<i32>` and
`Recursive<i32>` to be insensitive, `Wrap<*mut i32>` and
`Recursive<*mut i32>` to be sensitive, correct substitution of `T`, and
termination of the instantiated recursive cycle.

Use the same helper-only policy for this exact opaque-projection input:

```rust
pub trait HasItem {
    type Item;
}

pub unsafe fn projection<T: HasItem>(value: T::Item) {
    let _ = value;
}
```

If `T::Item` cannot be normalized to a concrete insensitive type in the
available parameter environment, require transformation. This is another
defensive unit test and does not expand end-to-end support for source-written
generics.

### A2-PRES-10 `external_nominal_representation_is_opaque_but_arguments_are_checked`

Exact Rust input:

```rust
pub unsafe fn vectors(
    mut integers: Vec<i32>,
    other_integers: Vec<i32>,
    mut pointers: Vec<*mut i32>,
    other_pointers: Vec<*mut i32>,
) {
    integers = other_integers;
    pointers = other_pointers;
}
```

The first assignment is preserved even though `Vec` internally uses pointers.
The second requires transformation because the explicit generic argument is a
raw pointer.

### A2-PRES-11 `future_field_type_change_marks_containing_values_sensitive`

Exact Rust input:

```rust
#[derive(Clone, Copy)]
pub struct FutureField {
    pub value: i32,
}

pub unsafe fn move_future(mut left: FutureField, right: FutureField) -> i32 {
    left = right;
    left.value + right.value
}
```

Exercise the classifier through its in-memory field-decision input twice. With
no field change, preserve both labels. With the exact synthetic planned field
change `FutureField::value: i32 -> String`, classify `FutureField` as
transformation-sensitive and require both the assignment and the tail field
uses to transform. This specifically prevents a preserved source-level copy
from surviving when a future field change removes `Copy`. The test fixes the
forward-compatible query boundary; it does not change the production
prototype's current empty field-decision set.

### A2-PRES-12 `changed_binding_decision_requires_transformation_even_through_safe_result`

Exact Rust input:

```rust
pub unsafe fn observe(pointer: *mut i32) -> bool {
    let is_null = pointer.is_null();
    is_null
}
```

Inject the exact target parameter decision
`pointer: *mut i32 -> &mut i32`. The `let` label requires transformation
because the referenced receiver binding changes and its source receiver
expression is raw-pointer-typed, even though the statement result is `bool`.
The final `is_null` label is preserved because `is_null` is an unchanged
Boolean binding.

### A2-PRES-13 `changed_local_callee_signature_requires_call_transformation`

Exact Rust input:

```rust
pub unsafe fn scalar_callee(value: i32) -> i32 {
    value + 1
}

pub unsafe fn scalar_caller(value: i32) -> i32 {
    scalar_callee(value)
}
```

Exercise the classifier through its in-memory signature-decision input. With
source and target callee signatures both exactly
`unsafe fn scalar_callee(i32) -> i32`, preserve the call. In a separate
classifier-only subcase, supply the exact synthetic target signature
`unsafe fn scalar_callee(String) -> i32`; require the call statement to
transform solely because the resolved local callee signature differs. This
synthetic decision tests the forward-compatible query and is not passed to
the current pointer-only replacer.

### A2-PRES-14 `indirect_calls_and_callable_values_are_never_preserved`

Exact Rust input:

```rust
pub unsafe fn invoke(callback: unsafe fn(i32) -> i32, value: i32) -> i32 {
    callback(value)
}
```

Although all visible parameter and result types are raw-pointer-free, the
indirect call requires transformation. This is a conservative robustness test
for input outside the prototype's supported no-function-pointer model.

Run the same conservative surface classifier on these two exact inputs:

```rust
pub unsafe fn local(value: i32) -> i32 {
    value + 1
}

pub unsafe fn hold_callable() {
    let callable = local;
    let _ = callable;
}
```

```rust
pub unsafe fn closure(value: i32) -> i32 {
    let add = |other: i32| value + other;
    add(1)
}
```

The callable-reference statement and every closure-containing or
closure-calling statement require transformation. These remain robustness
tests outside the supported no-function-pointer/no-closure input contract.

Finally classify this exact inline-assembly input:

```rust
pub unsafe fn assembly(mut value: u64) -> u64 {
    core::arch::asm!("/* {0} */", inout(reg) value);
    value
}
```

The inline-assembly statement requires transformation even though its operand
is pointer-free; the final scalar tail is preserved.

Ambiguously desugared syntax uses this exact input:

```rust
pub unsafe fn question(value: Option<i32>) -> Option<i32> {
    let inner = value?;
    Some(inner + 1)
}
```

The `let` containing `?` requires transformation under the conservative
desugared-syntax boundary. The final pointer-free variant construction is
preserved.

### A2-PRES-15 `macros_are_never_preserved`

Exact Rust input:

```rust
pub unsafe fn macros(value: i32) {
    println!("{value}");
    assert!(value > 0);
    let nested = 1 + dbg!(value);
    let _ = nested;
}
```

The first three statements, including the macro nested inside an initializer,
require transformation. The final scalar wildcard use is preserved. The
classifier does not use expanded implementation calls or inspect the apparent
token-tree types to claim preservation.

### A2-PRES-16 `patterns_and_guards_check_all_binding_types`

Exact Rust input:

```rust
pub unsafe fn patterns(
    pair: (i32, i32),
    pointer_pair: Option<(*mut i32, i32)>,
) -> i32 {
    let (left, right) = pair;
    let Some((pointer, value)) = pointer_pair else {
        return left + right;
    };
    match value {
        n if n > 0 => { let copy = n; copy }
        _ => { let _ = pointer; 0 }
    }
}
```

The first destructuring `let` is preserved. The `let-else` parent requires
transformation because its initializer and `pointer` pattern binding are
transformation-sensitive; its pointer-free `else` return is preserved. The
match parent requires transformation because its subtree contains the pointer
use in the fallback arm. The guarded positive arm statements are preserved;
the fallback pointer-use statement requires transformation.

Also classify this exact control-pattern input:

```rust
pub unsafe fn control_patterns(
    mut current: Option<*mut i32>,
    values: [*mut i32; 1],
) {
    if let Some(pointer) = current {
        let _ = pointer;
    }
    while let Some(pointer) = current.take() {
        let _ = pointer;
    }
    for pointer in values {
        let _ = pointer;
    }
}
```

Each `if let`, `while let`, and `for` parent requires transformation, as does
each nested pointer-binding use. This verifies that desugaring does not hide
the initializer, iterator item, or pattern binding type.

### A2-PRES-17 `fully_safe_control_subtree_is_preserved_as_one_canonical_tree`

Exact Rust input:

```rust
pub unsafe fn control(flag: bool, left: i32, right: i32) -> i32 {
    if flag {
        let value = left + right;
        return value;
    } else {
        return left - right;
    }
}
```

Every label is preserved. The complete outer `if`, including all nested
labels, appears unchanged in the canonical target skeleton apart from target
types and presentation mutability. The outer label is not listed for
transformation merely because the statement is a control form.

Pointer-free unsafe storage operations use this exact input:

```rust
pub static mut GLOBAL: i32 = 0;

pub union Scalar {
    pub signed: i32,
    pub unsigned: u32,
}

pub unsafe fn storage(value: Scalar) -> i32 {
    GLOBAL = 1;
    let local = value.signed;
    local + GLOBAL
}
```

All three labels require transformation. Labels 0 and 2 access `static mut`;
label 1 reads a union field. Neither operation is a preservation candidate
even though every involved type is pointer-free. Also classify this exact
raw-pointer-containing union input:

```rust
pub union Mixed {
    pub pointer: *mut i32,
    pub integer: i32,
}

pub unsafe fn mixed_storage(value: Mixed) -> i32 {
    value.integer
}
```

The tail requires transformation both because union-field access is prohibited
from preservation and because opening the project-local union finds the
raw-pointer field even though the selected field is scalar.

### A2-PRES-18 `amendment_1_conditionals_preserve_or_remain_opaque`

Use exact input A2-SRC-RESTRICTED-CONDITIONAL. The first assignment's
Amendment-1 restricted conditional is pointer-free, so the complete assignment
is preserved with its one enclosing label and no branch labels. The second
assignment requires transformation; it retains Amendment 1's one opaque
`todo!()` payload and no branch labels because one branch dereferences a raw
pointer. The final `x` label is preserved.

## 6. Preservation-aware validator tests

All cases in this section use the exact annotated skeleton generated from
A2-SRC-VALIDATOR and its generated disposition metadata unless a case gives a
different complete input. The parser-only fixtures hold the target signature
equal to the printed source signature, so every returned function below has
that exact signature; pointer target-decision integration is tested in
Section 7.

### A2-VAL-01 `version_1_request_and_response_have_exact_final_shapes`

The exact expected-function object contains:

```json
{
  "id": 0,
  "name": "validate_me",
  "skeleton": "<exact generated annotated_skeleton>",
  "needs_transformation": true,
  "statements_requiring_transformation": [1, 3]
}
```

Here labels are `0 let scalar`, `1 outer if`, `2 nested let`, `3 pointer
write`, `4 else return`, and `5 final scalar`; the recursive parent rule makes
labels 1 and 3 transformed. A schema version other than 1, an omitted field,
an unknown field, an inconsistent Boolean, a duplicate/out-of-order label, a
nonexistent label, or a preserved label with a transformed descendant is a
deterministic `setup_error`.

Every serialized response continues to use `"schema_version": 1`. Re-run the
exact existing `valid`, `invalid`, and `setup_error` response serialization
goldens with the amended request. Their status-specific keys and key order are
unchanged. An unsupported request version such as 2 is still a setup error;
there is no version-2 request or response.

### A2-VAL-02 `changed_preserved_leaf_is_accepted_and_canonicalized`

Exact returned transformation:

```rust
unsafe fn validate_me(flag: bool, pointer: *mut i32) -> i32 {
    #[proctor(0)]
    let scalar: i32 = 999;
    #[proctor(1)]
    if flag {
        #[proctor(2)]
        let nested: i32 = -100;
        #[proctor(3)]
        *pointer = nested;
    } else {
        #[proctor(4)]
        return -200;
    }
    #[proctor(5)]
    300
}
```

After canonicalization, labels 0, 2, 4, and 5 equal the target-skeleton
statements, while transformed labels 1 and 3 retain returned payloads. The
preserved-statement differences produce no validation error.

### A2-VAL-03 `preserved_expansion_group_is_discarded_as_a_whole`

Exact Rust input:

```rust
pub unsafe fn f(value: i32) -> i32 {
    value + 1
}
```

Exact returned transformation:

```rust
unsafe fn f(value: i32) -> i32 {
    #[proctor(0)]
    let proctor_temp_var_0 = value * 100;
    #[proctor(0)]
    proctor_temp_var_0
}
```

The complete two-statement group is discarded and replaced by the one
canonical `value + 1` statement. Validation is `valid`.

### A2-VAL-04 `preserved_parent_is_opaque_after_outer_alignment`

Exact Rust input:

```rust
pub unsafe fn f(flag: bool) -> i32 {
    if flag {
        let value = 1;
        value
    } else {
        2
    }
}
```

Exact returned transformation:

```rust
unsafe fn f(flag: bool) -> i32 {
    #[proctor(0)]
    return 999;
}
```

The outer label is in the expected function-body position. Its returned
statement, changed control kind, and missing descendant labels are ignored;
the complete canonical `if` subtree is restored and validation is `valid`.

### A2-VAL-05 `preserved_child_must_be_locatable_under_transformed_parent`

Use A2-SRC-VALIDATOR. The exact successful transformation is:

```rust
unsafe fn validate_me(flag: bool, pointer: *mut i32) -> i32 {
    #[proctor(0)]
    let scalar: i32 = 999;
    #[proctor(1)]
    let proctor_temp_var_0 = flag;
    #[proctor(1)]
    if proctor_temp_var_0 {
        #[proctor(2)]
        let nested: i32 = -100;
        #[proctor(3)]
        *pointer = nested;
    } else {
        #[proctor(4)]
        return -200;
    }
    #[proctor(1)]
    let proctor_temp_var_1 = 0;
    #[proctor(5)]
    300
}
```

The label-1 expansion has exactly one Phase-2 structural anchor: the `if`.
The same-label statements before and after it remain ordinary expansion
siblings. Canonicalization recurses only through that anchor, locates
preserved labels 2 and 4 in their expected roles, and accepts the result.

For the misplaced-child subcase, replace the successful anchor with this
exact group:

```rust
#[proctor(1)]
if proctor_temp_var_0 {
    #[proctor(3)]
    *pointer = 7;
} else {
    #[proctor(2)]
    let nested: i32 = -100;
    #[proctor(4)]
    return -200;
}
```

Expect `descendant_location_mismatch`. For zero- and multiple-anchor
subcases, replace the complete successful label-1 group respectively with:

```rust
#[proctor(1)]
let proctor_temp_var_0 = flag;
#[proctor(1)]
let proctor_temp_var_1 = 0;
```

```rust
#[proctor(1)]
if flag {
    #[proctor(2)]
    let nested: i32 = -100;
    #[proctor(3)]
    *pointer = nested;
} else {
    #[proctor(4)]
    return -200;
}
#[proctor(1)]
if flag {
    #[proctor(2)]
    let nested: i32 = -101;
    #[proctor(3)]
    *pointer = nested;
} else {
    #[proctor(4)]
    return -201;
}
```

Expect the stable “missing control anchor” and “multiple control anchors”
structural diagnostics. A preserved child becomes opaque only after its
unique correct structural location is established.

### A2-VAL-06 `missing_malformed_reordered_or_nonconsecutive_outer_group_is_invalid`

Exact Rust input:

```rust
pub unsafe fn f(value: i32) -> i32 {
    let first = value + 1;
    first + 2
}
```

Run these exact returned transformations:

```rust
unsafe fn f(value: i32) -> i32 {
    #[proctor(1)]
    value + 2
}
```

```rust
unsafe fn f(value: i32) -> i32 {
    #[proctor(00)]
    let first = value + 1;
    #[proctor(1)]
    first + 2
}
```

```rust
unsafe fn f(value: i32) -> i32 {
    #[proctor(1)]
    value + 2;
    #[proctor(0)]
    let first = value + 1;
}
```

```rust
unsafe fn f(value: i32) -> i32 {
    #[proctor(0)]
    let first = value + 1;
    #[proctor(1)]
    first + 2;
    #[proctor(0)]
    let second = value + 3;
}
```

Preserve the existing stable error codes for missing, malformed, reordered,
and nonconsecutive labels, respectively. Discarding contents does not weaken
outer alignment.

### A2-VAL-07 `discarded_body_errors_do_not_leak_but_external_uses_are_checked`

Exact Rust input:

```rust
pub unsafe fn f(value: i32, pointer: *mut i32) -> i32 {
    let scalar = value + 1;
    *pointer = scalar;
    scalar
}
```

The first exact returned transformation is:

```rust
unsafe fn f(value: i32, pointer: *mut i32) -> i32 {
    #[proctor(0)]
    #[allow(unused_variables)]
    unsafe {
        const LOCAL: i32 = 100;
        let wrong_name = value + LOCAL;
        wrong_name
    };
    #[proctor(1)]
    *pointer = value + 1;
    #[proctor(2)]
    999
}
```

It is valid: the explicit unsafe block, local `const`, expression attribute,
and invalid replacement for the existing `scalar` declaration are all inside
discarded label 0. Preserved label 2 is restored as `scalar`.

The second exact returned transformation is:

```rust
unsafe fn f(value: i32, pointer: *mut i32) -> i32 {
    #[proctor(0)]
    let proctor_temp_var_0 = value + 1;
    #[proctor(1)]
    *pointer = proctor_temp_var_0;
    #[proctor(2)]
    999
}
```

Canonicalization removes the temporary declaration, so
`temporary_reference_cannot_cross_group_boundary` rejects its use in
transformed label 1.

### A2-VAL-08 `ordinary_transformed_group_validation_is_unchanged`

Exact Rust input:

```rust
pub unsafe fn f(pointer: *mut i32) {
    *pointer = 1;
}
```

The sole label requires transformation. Re-run representative existing
validator fixtures exactly from P2-LABEL-01 (one-to-many), P2-LABEL-06
(unlabeled sibling), P2-BIND-02 (wrong existing binding), P2-TEMP-03
(cross-group temporary), and P2-SAFE-01 (explicit unsafe block), changing only
the request object to include Amendment 2's final version-1 metadata. Their
transformations, results, and stable error codes are otherwise byte-for-byte
unchanged because no group is preserved.

## 7. Replacement and integration tests

### A2-REP-01 `version_1_replacement_request_has_exact_final_shape`

Use exact input A2-SRC-VALIDATOR. The requested item contains:

```json
{
  "id": 0,
  "path": "validate_me",
  "name": "validate_me",
  "skeleton": "<exact generated annotated_skeleton>",
  "needs_transformation": true,
  "statements_requiring_transformation": [1, 3]
}
```

The top-level request has `schema_version: 1`, the ordered item array, and one
complete transformation string. Version 2, missing preservation metadata,
inconsistent metadata, and unknown fields are rejected before rewriting.

### A2-REP-02 `replacement_discards_every_preserved_llm_group`

Use exact current source A2-SRC-VALIDATOR and exact transformation from
A2-VAL-02. The emitted implementation contains canonical expressions
`1 + 2`, `3 + 4`, `return scalar`, and final `scalar`; it contains none of
`999`, `-100`, `-200`, or `300`. The transformed pointer-write group comes
from the returned transformation. All `proctor` labels are removed.

### A2-REP-03 `replacement_recanonicalizes_without_prior_validator`

Use exact Rust input and transformation from A2-VAL-03. Call `replace_items`
directly without calling `validate`. The emitted function body is exactly the
canonical `value + 1` expression after label removal. This proves the replacer
does not trust orchestration order.

### A2-REP-04 `fully_preserved_body_can_still_change_signature_and_create_wrapper`

Use exact input A2-SRC-PRESERVED-WRAPPER and inject this exact target skeleton:

```rust
unsafe fn unused_pointer(mut pointer: &mut i32, mut value: i32) -> i32 {
    #[proctor(0)]
    let mut doubled: i32 = value * 2;
    #[proctor(1)]
    doubled + 1
}
```

This constructed replacer fixture fixes the pointer target decision to
`pointer: &mut i32`; it does not depend on whether the current analysis would
select that decision for an unused parameter. The record has
`needs_transformation = false` and
`statements_requiring_transformation = []`. Replacement uses the exact target
signature and canonical complete body. Existing wrapper rules move
`#[no_mangle]` and `extern "C"` to a same-module compatibility wrapper. The
implementation, wrapper, and conversion from `*mut i32` to `&mut i32` compile
under the existing in-memory replacement check.

### A2-REP-05 `preserved_calls_cannot_overwrite_required_wrapper_redirection`

Use exact input A2-SRC-SIGNATURE-SCC. Inject these exact target signatures:

```rust
unsafe fn callee(mut _pointer: &mut i32, mut value: i32) -> i32
unsafe fn caller(mut pointer: &mut i32, mut value: i32) -> i32
```

The caller's call label is transformed because its source expressions contain
raw pointers and the resolved callee signature changes. Replace `callee`
first and assert that the still-source-form `caller` is redirected to the
generated raw-pointer compatibility wrapper. Later replace `caller` with this
exact transformed group under its generated call label:

```rust
#[proctor(0)]
callee(pointer, value)
```

The emitted target-form caller invokes the transformed implementation
directly. Because that call group is transformed, canonical preservation can
neither restore the obsolete raw-pointer call nor undo the earlier wrapper
redirection.

For the unchanged-signature contrast, use the exact scalar
`scalar_callee`/`scalar_caller` source from A2-PRES-13 with both target
signatures equal to their printed source signatures. Its caller label is
preserved and no wrapper or call-site redirection is generated.

### A2-REP-06 `metadata_or_canonicalization_failure_is_atomic`

Use exact input A2-SRC-VALIDATOR. Independently mutate the valid metadata to
`statements_requiring_transformation = [1, 3, 99]`, mutate it to `[3]` while
label 1 remains a preserved ancestor, and use the exact misplaced-child
transformation from A2-VAL-05. Keep `needs_transformation = true` in all three
requests. Each request returns no source string and makes no partial wrapper,
call, main, or function-body edit.

## 8. Python loading, prompt, and SCC orchestration tests

Every case uses an exact originating Rust input named below. Fake skeleton JSON
contains the exact Crat-generated fields for that input; Python never parses
the Rust.

### A2-PY-01 `loader_requires_and_preserves_amended_function_fields`

Use exact originating input A2-SRC-SCALAR. Load its function record with
`needs_transformation: false` and an empty transformation-label array.
Independently reject a missing Boolean, a non-Boolean, a missing/non-array
label field, Boolean integers, labels outside `u32`, duplicate or descending
labels, and Boolean/array inconsistency. Preserve Rust strings exactly.

### A2-PY-02 `version_1_prompt_has_exact_final_minimal_insertion`

Use exact originating input A2-SRC-MIXED-CONTROL. Load prompt
`local_transformation`, version 1. Construct the expected text by taking the
complete historical Section 10 body and inserting exactly the Amendment 2
paragraph at the specified boundary. Assert byte equality, content hash,
frontmatter variables, and request metadata version 1. The exact insertion is:

```text
Complete every generated `todo!()` hole. Preserve every complete labeled statement already present in the Target Skeleton exactly as provided.
```

The complete-statement and hole instructions each occur once. No second prompt
file or version-2 metadata is introduced.

### A2-PY-03 `all_preserved_singleton_skips_the_llm_request_path`

Use exact originating input A2-SRC-SCALAR. Queue a successful normalized
initial build, mechanical replacement candidate, and candidate build. Exact
SCC events after initialization are:

```text
replace with concatenated annotated_skeleton for SCC 0
cargo_build
```

There is no dependency-context render, prompt render, LLM call, extraction,
validation request, or validator call. This test does not assert which
process-level provider-adjacent resources were initialized.

### A2-PY-04 `entirely_mechanical_run_makes_zero_llm_calls`

Exact Rust input:

```rust
pub unsafe fn first(value: i32) -> i32 {
    value + 1
}

pub unsafe fn second(value: i32) -> i32 {
    first(value) * 2
}
```

Both singleton SCCs are all-preserved. Supply the ordinary fake LLM
configuration and make its generation method fail the test if invoked. Both
SCCs replace and build in leaf order. Final reporting includes:

```text
function_count = 2
scc_count = 2
llm_generation_calls = 0
repair_calls = 0
structural_failures = 0
compilation_failures = 0
cargo_builds = 3
```

The three builds are the normalized initial build plus two mechanical
candidate builds. Do not add assertions about initialization of prompt
libraries, indexes, trackers, clients, or exchange directories.

### A2-PY-05 `mixed_scc_makes_one_llm_call_and_restores_preserved_member`

Exact Rust input:

```rust
pub unsafe fn even(value: u32) -> char {
    if value == 0 {
        return 'E';
    }
    odd(value - 1)
}

pub unsafe fn odd(value: u32) -> char {
    if value == 0 {
        return char::from_u32_unchecked(79);
    }
    even(value - 1)
}
```

The mutually recursive pair is one SCC. `even` is genuinely all-preserved:
its metadata is `false, []`. `odd` has metadata `true, [0, 1]`: its outer
`if` and nested return require transformation because
`char::from_u32_unchecked` is an unsafe non-local call, while its final local
recursive call is preserved. The LLM request still contains both functions.
Return this exact response:

```rust
unsafe fn even(mut value: u32) -> char {
    #[proctor(0)]
    return '?';
    #[proctor(2)]
    odd(value - 99)
}

unsafe fn odd(mut value: u32) -> char {
    #[proctor(0)]
    if value == 0 {
        #[proctor(1)]
        return char::from_u32(79).unwrap();
    }
    #[proctor(2)]
    even(value - 99)
}
```

Canonicalization restores the complete original `even` body, including its
omitted nested label 1, and restores `odd`'s final `even(value - 1)` call. It
retains the transformed safe replacement for label 1. Assert one logical
generation, one validation, one replacement transaction, and one candidate
build.

### A2-PY-06 `mechanical_and_llm_sccs_share_one_deterministic_schedule`

Exact Rust input:

```rust
pub unsafe fn scalar_leaf(value: i32) -> i32 {
    value + 1
}

pub unsafe fn pointer_leaf(pointer: *mut i32) -> i32 {
    *pointer
}

pub unsafe fn root(pointer: *mut i32, value: i32) -> i32 {
    scalar_leaf(value) + pointer_leaf(pointer)
}
```

The leaf schedule processes `scalar_leaf` mechanically, then
`pointer_leaf` through the LLM, then `root` through the LLM. No generation
request is made for `scalar_leaf`; the first generation request is for
`pointer_leaf`. Later replacements see the promoted current source while
every prompt still uses immutable skeleton records.

### A2-PY-07 `mechanical_signature_change_still_runs_replacer_and_build`

Use exact originating input A2-SRC-PRESERVED-WRAPPER and the exact synthetic
target skeleton and metadata from A2-REP-04. Even though
`needs_transformation` is false, assert one replacement and one SCC candidate
build after the normalized build. The stage passes the skeleton itself as the
transformation; it does not copy the input directly or skip integration. The
fake replacer asserts the exact `&mut i32` target signature and returns a
candidate containing the compatibility wrapper.

### A2-PY-08 `mechanical_failure_is_fatal_and_not_repairable`

Use exact originating input A2-SRC-SCALAR. Run independent subcases where the
mechanical replacer fails and where its candidate build fails. Each aborts
with zero LLM calls and zero repairs. A failed build restores the prior source
through the existing transaction, and no output project is claimed.

## 9. Completion criteria

Amendment 2 is complete when:

- all 45 named cases are implemented and passing;
- every existing implementation test superseded by these expectations is
  updated without editing historical test-plan files;
- preservation is a Crat proof over mapped HIR and target decisions, never a
  Python text heuristic;
- project-local field traversal is recursive and future field-type changes use
  the same sensitivity query;
- validator and replacer use one canonicalization implementation;
- no LLM-written preserved statement reaches emitted source;
- an all-preserved SCC performs replacement and compilation without
  issuing an LLM request; and
- ordinary transforming statements retain the completed Phase 1--4 structural,
  wrapper, repair, transaction, and reporting behavior.
