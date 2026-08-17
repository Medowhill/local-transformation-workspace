# Amendment 6 Test Plan: Maximal Regions and Foreign-Call Rules

## 1. Purpose and test discipline

This plan is the exhaustive acceptance specification for
[Amendment 6](amendment-6-plan.md). It covers the changed Crat tools behavior
at four boundaries:

1. compiler-resolved expression-region selection and observation extraction;
2. version-1 observation/rule validation and serialization;
3. coupled rule synthesis, including scan-format literal rigidity; and
4. rule matching, ranking, materialization, and atomic statement application.

Tests belong in the existing module-local Crat test modules:

- compiler/extraction cases in `crates/tools/src/observation.rs`;
- document, synthesis, matching, and ranking cases in
  `crates/tools/src/rule.rs` or its existing child test module;
- applied-view and materialization cases in
  `crates/tools/src/skeleton/tests.rs` or the existing emitted-view test
  module; and
- Markdown cases in `crates/tools/src/rule/markdown.rs` when direct coverage is
  needed.
- SCC scheduling/fallback/publication and opaque-document integration cases in
  `proctor/tests/test_local_transformation.py`, using its existing temporary
  fake projects, fake provider/tool callbacks, and saved-output assertions.

Do not place **Crat** tests in a Crat-root `tests/` directory. Crat tests must
call library functions directly, must not invoke `crat-tool`, and must not
create or mutate filesystem state. Use the existing source-string compiler
harness for compiler-resolved cases and direct normalized values for pure
synthesis cases. The filesystem restriction does not apply to PROCTOR's
existing pytest ownership: its required cases may use `tmp_path` and the
established fake stage/tool behavior, but must remain offline and must not run
a real Crat CLI.

Names such as `A6-SELECT-01` below are planning references only. Do not put
`A6`, `Amendment 6`, `phase`, or `amendment` in implementation test names,
fixtures, diagnostics, or source comments. Suggested implementation test names
are the descriptive backticked phrases.

## 2. Execution and exact-comparison policy

For normalized document cases, compare complete Rust values or complete
pretty canonical JSON, not substrings. For compiler cases, compare exact
selected root structure, observation count/order, anchors/order, `lhs`, all
four root types, and source/target expressions. For applied-view cases, compare
the complete applied statement and its disposition; parse it and inspect the
AST where pretty-printer parentheses are not contractual.

Determinism cases must repeat with reversed rule-document order, permuted
observation-document order where synthesis promises multiset determinism, and
at least two compiler runs where source traversal is involved.

Run from `proctor/stages/crat`:

```bash
cargo test -p tools observation::tests
cargo test -p tools rule::tests
cargo test -p tools skeleton::tests
cargo test -p tools
cargo test --workspace
cargo fmt
cargo clippy --workspace --all-targets
```

Run the required orchestration regressions from `proctor`:

```bash
uv run pytest tests/test_local_transformation.py
```

This focused pytest command is mandatory because Sections 12 and 13 include
required orchestration-owned cases, even though production Python and the stage
protocol are expected to remain unchanged.

## 3. Shared exact notation

The following JSON fragments are definitions used by later cases. They are
literal expansions, not a proposal to add test-only production syntax.

```json
{
  "I32": {"kind": "primitive", "name": "i32"},
  "U8": {"kind": "primitive", "name": "u8"},
  "I8": {"kind": "primitive", "name": "i8"},
  "CONST_U8": {
    "kind": "raw_pointer",
    "mutability": "const",
    "pointee": {"kind": "primitive", "name": "u8"}
  },
  "CONST_I8": {
    "kind": "raw_pointer",
    "mutability": "const",
    "pointee": {"kind": "primitive", "name": "i8"}
  },
  "MUT_I32": {
    "kind": "raw_pointer",
    "mutability": "mut",
    "pointee": {"kind": "primitive", "name": "i32"}
  },
  "B0": {"kind": "path", "value": {"kind": "binding", "id": "<id0>"}},
  "SCANF": {
    "kind": "path",
    "value": {"kind": "foreign_function", "symbol": "scanf"}
  },
  "XJ_SCANF": {
    "kind": "path",
    "value": {
      "kind": "external",
      "crate": "xj_scanf",
      "path": ["legacy", "scanf"]
    }
  },
  "XJ_BRSCANF": {
    "kind": "path",
    "value": {
      "kind": "external",
      "crate": "xj_scanf",
      "path": ["legacy", "brscanf"]
    }
  },
  "XJ_BSCANF": {
    "kind": "path",
    "value": {
      "kind": "external",
      "crate": "xj_scanf",
      "path": ["legacy", "bscanf"]
    }
  }
}
```

`SCAN_SOURCE` expands exactly to:

```json
{
  "kind": "call",
  "callee": {
    "kind": "path",
    "value": {"kind": "foreign_function", "symbol": "scanf"}
  },
  "arguments": [
    {
      "kind": "cast",
      "expression": {
        "kind": "cast",
        "expression": {
          "kind": "literal",
          "value": {"kind": "byte_string", "value": [37, 100, 0]}
        },
        "type": {
          "kind": "raw_pointer",
          "mutability": "const",
          "pointee": {"kind": "primitive", "name": "u8"}
        }
      },
      "type": {
        "kind": "raw_pointer",
        "mutability": "const",
        "pointee": {"kind": "primitive", "name": "i8"}
      }
    },
    {
      "kind": "cast",
      "expression": {
        "kind": "address_of",
        "borrow": "reference",
        "mutability": "mut",
        "expression": {
          "kind": "path",
          "value": {"kind": "binding", "id": "<id0>"}
        }
      },
      "type": {
        "kind": "raw_pointer",
        "mutability": "mut",
        "pointee": {"kind": "primitive", "name": "i32"}
      }
    }
  ]
}
```

`SCAN_TARGET` expands exactly to:

```json
{
  "kind": "call",
  "callee": {
    "kind": "path",
    "value": {
      "kind": "external",
      "crate": "xj_scanf",
      "path": ["legacy", "scanf"]
    }
  },
  "arguments": [
    {
      "kind": "literal",
      "value": {"kind": "string", "value": "%d"}
    },
    {
      "kind": "address_of",
      "borrow": "reference",
      "mutability": "mut",
      "expression": {
        "kind": "array",
        "elements": [
          {
            "kind": "address_of",
            "borrow": "reference",
            "mutability": "mut",
            "expression": {
              "kind": "path",
              "value": {"kind": "binding", "id": "<id0>"}
            }
          }
        ]
      }
    }
  ]
}
```

`SCAN_OBSERVATION` is the exact value:

```json
{
  "source_expression": "SCAN_SOURCE",
  "target_expression": "SCAN_TARGET",
  "pointer_anchors": [],
  "lhs": false,
  "source_type": {"kind": "primitive", "name": "i32"},
  "source_adjusted_type": {"kind": "primitive", "name": "i32"},
  "target_type": {"kind": "primitive", "name": "i32"},
  "target_adjusted_type": {"kind": "primitive", "name": "i32"}
}
```

In an actual expected value, replace the quoted `"SCAN_SOURCE"` and
`"SCAN_TARGET"` placeholders with the complete objects above. This shorthand
is used only to keep later cases readable.

For synthesis-only cases, `obs(S, T)` means the complete observation:

```json
{
  "source_expression": S,
  "target_expression": T,
  "pointer_anchors": [],
  "lhs": false,
  "source_type": {"kind": "primitive", "name": "i32"},
  "source_adjusted_type": {"kind": "primitive", "name": "i32"},
  "target_type": {"kind": "primitive", "name": "i32"},
  "target_adjusted_type": {"kind": "primitive", "name": "i32"}
}
```

`doc(O1, O2)` means
`{"schema_version":1,"observations":[O1,O2]}`. A rule's corresponding four
context types are the rule grammar's concrete primitive `i32` nodes.

## 4. Updated existing regression cases

### A6-UPDATE-01 `strict_ancestry_keeps_the_maximal_region`

Update the current `strict_ancestry_overlap_skips_complete_statement` case.

Input:

```rust
unsafe fn source_copy(mut base: *const i32, mut other: *const i32) -> i32 {
    #[proctor(0)]
    *base.offset(other.offset_from(base))
}
unsafe fn target(mut base: &[i32], mut other: &[i32]) -> i32 {
    #[proctor(0)]
    base[other.as_ptr().offset_from(base.as_ptr()) as usize]
}
```

Expected: exactly one observation, not zero. Its source root is the complete
`*base.offset(other.offset_from(base))`; its target root is the complete index
expression. Its anchors are exactly `<id0>` for `base`, then `<id1>` for
`other`; the second `base` occurrence is deduplicated. `lhs` is false. Repeating
the extraction produces byte-identical canonical JSON.

### A6-UPDATE-02 `promoted_overlap_keeps_the_outer_field`

Update the current `promoted_regions_participate_in_overlap_check` case.

Input:

```rust
struct Pair { value: i32 }
unsafe fn source_copy(mut pointer: *const Pair, mut index: *const isize) -> i32 {
    #[proctor(0)]
    (*pointer.offset(*index)).value
}
unsafe fn target(mut pointer: &[Pair], mut index: &isize) -> i32 {
    #[proctor(0)]
    pointer[*index as usize].value
}
```

Expected: exactly one observation rooted at the complete field expression.
The field remains the same resolved `Pair::value`; anchors are `pointer` then
`index`; the retained root is marked internally as promoted for alignment;
`lhs` is false. No descendant promotion flag is copied to a different root.

### A6-UPDATE-03 `empty_anchor_documents_are_valid`

Update every loader test whose sole expected error is
`pointer_anchors must be nonempty`. The exact anchorless observation/rule
documents in Section 7 must now round-trip. Keep tests for duplicate,
misordered, unused, wrong-sort, and wrong-type nonempty anchors unchanged.

### A6-UPDATE-04 `literal_conflicts_remain_generalizable_outside_formats`

Retain the historical behavior represented by the current noninteger-literal
anti-unification coverage. Input two non-scan calls:

```text
pair('x', *A0) -> pair('x', A0)
pair('y', *A0) -> pair('y', A0)
```

Expected: the character-literal position becomes one expression variable and
is reused in the target. This regression prevents scan-format rigidity from
making all string-like or noninteger literals globally rigid.

### A6-UPDATE-05 `link_name_metadata_retains_rust_name_and_adds_symbol`

Update the existing
`handles_link_name_callable_reference_and_dependency_metadata` Crat test and
the focused PROCTOR guidance tests with this exact matrix:

1. For local exact-C-ABI
   `#[link_name = "strlen"] fn c_strlen(...)`, both a direct call and a
   function-value reference emit `foreign_function_names` exactly
   `["c_strlen", "strlen"]`.
2. For local exact-C-ABI
   `#[link_name = "scanf"] fn rust_scanf(...)`, a function that references it
   emits exactly `["rust_scanf", "scanf"]`. Loading that unchanged list in
   Python makes `_uses_xj_scanf_guidance` return `true`; rendered transformation
   targets contain exactly
   ``Foreign function references: `rust_scanf`, `scanf` `` in canonical order
   and the prompt contains the existing xj-scanf guidance once.
3. For a direct local exact-C-ABI declaration named `scanf` without
   `link_name`, emit exactly `["scanf"]`; declaration name and semantic symbol
   do not create a duplicate.
4. For `#[link_name = "scanf"] fn rust_scanf(...)` in `extern "C-unwind"`,
   retain only `["rust_scanf"]`; Python guidance remains disabled. The linked
   symbol is not added because the declaration is outside the shared supported-
   foreign predicate.
5. Preserve the dependency-owned `libc::free` expectation exactly as
   `["free"]`. Also add this GNU-host-gated case to the same test, using the
   already available pinned `libc` dependency through the existing read-only
   compiler harness:

   ```rust
   extern crate libc;

   pub unsafe fn dependency_renamed(
       path: *mut libc::c_char,
   ) -> *mut libc::c_char {
       libc::posix_basename(path)
   }
   ```

   In the pinned GNU `libc`, the dependency-owned Rust declaration
   `posix_basename` has `#[link_name = "__xpg_basename"]`. Expect
   `foreign_function_names` exactly `["posix_basename"]` and explicitly
   assert that `"__xpg_basename"` is absent. Gate this assertion with
   `#[cfg(all(target_os = "linux", target_env = "gnu"))]`; do not compile a
   temporary dependency or write fixture state during the Crat test.
6. Preserve the Python exact-name negatives: `["rust_scanf"]`, `["c_scanf"]`,
   `["vscanf"]`, and `["__isoc99_scanf"]` do not activate guidance. A sorted
   list containing a retained Rust name plus exact linked `scanf`, `fscanf`, or
   `sscanf` does activate it.

Compare the complete sorted vectors and complete rendered guidance state. Do
not add a record field, relax Python's closed record keys, or change the prompt
template/version.

## 5. Maximal-region selection and anchor transfer

### A6-SELECT-01 `identical_roots_merge_anchors_once`

Use the coalescer's focused internal test fixture with two candidate
`SelectedRegion` values that have the same real AST root identity. Give the
first anchors `(p at preorder 2, q at preorder 5)` and the second anchors
`(p at preorder 7, r at preorder 9)`, with consistent paired target bindings.
Expected: one retained region at that root, anchors exactly `p`, `q`, `r` in
preorder `2, 5, 9`; the later `p` occurrence is removed. If either candidate
was promoted to this same root, the merged root's `promoted_field` is true.

### A6-SELECT-02 `all_disjoint_maximal_roots_survive`

Input is the existing binary case:

```rust
unsafe fn source_copy(mut left: *const i32, mut right: *const i32) -> i32 {
    #[proctor(0)] *left + *right
}
unsafe fn target(mut left: &i32, mut right: &i32) -> i32 {
    #[proctor(0)] *left + *right
}
```

Expected: two observations in source order, first `*left`, then `*right`.
Each observation has one anchor anonymized as `<id0>` within that observation.
No globally largest region is chosen.

### A6-SELECT-03 `ancestor_absorbs_multiple_descendants`

Input:

```rust
unsafe fn source_copy(
    mut base: *const i32,
    mut first: *const isize,
    mut second: *const isize,
) -> i32 {
    #[proctor(0)]
    *base.offset((*first + *second) as isize)
}
unsafe fn target(mut base: &[i32], mut first: &isize, mut second: &isize) -> i32 {
    #[proctor(0)]
    base[(*first + *second) as usize]
}
```

Expected: one maximal dereference observation. Anchors are `base`, `first`,
`second` in first source-occurrence order. Neither descendant region emits an
observation.

### A6-SELECT-04 `dedup_uses_binding_identity_not_text`

Use a focused coalescer fixture with resolved bindings `parameter_p` and
`shadow_p`, both displayed as `p`. Candidate descendants contribute occurrences
`parameter_p@3`, `parameter_p@6`, and `shadow_p@8` to one retained root.
Expected anchors are exactly `parameter_p@3`, `shadow_p@8`; the parameter's
second occurrence is removed. Deduplication uses resolved identity, not text.

### A6-SELECT-05 `retained_root_recomputes_lhs`

Input:

```rust
unsafe extern "C" {
    fn get_out(p: *mut *mut i32) -> *mut *mut i32;
}
unsafe fn source_copy(mut p: *mut *mut i32) {
    #[proctor(0)]
    *get_out(p) = core::ptr::null_mut();
}
unsafe fn target(mut p: &mut *mut i32) {
    #[proctor(0)]
    *get_out(p as *mut *mut i32) = core::ptr::null_mut();
}
```

Expected: the foreign seed grows through dereference to the complete assignment
LHS and contains the nested `p` anchor candidate. Coalescing retains that
dereference, transfers anchor `p`, and computes `lhs: true` from the final root.
The RHS has no seed.

### A6-SELECT-06 `descendant_lhs_does_not_taint_ancestor`

Use the coalescer/final-metadata internal fixture with a discarded child whose
pre-coalescing test value has `lhs: true` and a retained parent whose actual
parent edge is not `AssignLeft`. Expected: final metadata recomputation makes
the retained root `lhs: false`. This directly asserts that coalescing does not
OR stale flags; compiler-backed A6-SELECT-05 covers the realizable assignment
shape.

### A6-SELECT-07 `field_promotion_is_root_local`

Use the promoted-field source from A6-UPDATE-02 and inspect the coalesced
selector result. Expected: the retained field root has `promoted_field: true`.
Add a second internal tree where a promoted descendant is contained by a
different, larger foreign-call-derived root. Expected: the retained call root
has `promoted_field: false`; the descendant flag is not transferred.

### A6-SELECT-08 `source_order_is_independent_of_seed_kind`

Input one statement with, in order, a pointer-only region, an anchorless
foreign call, and another pointer-only region:

```rust
unsafe extern "C" { fn ping(value: i32) -> i32; }
unsafe fn source_copy(mut p: *const i32, mut q: *const i32) -> i32 {
    #[proctor(0)] *p + ping(7) + *q
}
unsafe fn target(mut p: &i32, mut q: &i32) -> i32 {
    #[proctor(0)] *p + ping(8) + *q
}
```

Expected observations are ordered `*p`, `ping(7)`, `*q`; the middle
observation is anchorless. Repeated compiler runs have identical order.

### A6-SELECT-09 `rejected_seed_does_not_remove_valid_maxima`

Combine one indirect-call pointer argument that is rejected by the existing
parent policy with one disjoint supported foreign call and one valid pointer
dereference. Expected: the rejected seed contributes nothing; the other two
maximal regions remain in source order.

### A6-SELECT-10 `nested_labels_remain_opaque`

Input an outer labeled control containing a nested labeled statement whose
foreign call and pointer anchor would otherwise be descendants of an outer
candidate. Expected: outer selection treats the nested expression as opaque;
the inner label is processed independently and its call/anchors do not transfer
to an outer region.

## 6. Foreign-call seed discovery and observation extraction

### A6-FOREIGN-01 `direct_local_c_abi_call_seeds_without_anchors`

Input:

```rust
unsafe extern "C" { fn ping(value: i32) -> i32; }
unsafe fn source_copy() -> i32 {
    #[proctor(0)] ping(1)
}
unsafe fn target() -> i32 {
    #[proctor(0)] ping(2)
}
```

Expected: one observation rooted at the complete call, with
`pointer_anchors: []`, `lhs: false`, and all four root types equal to primitive
`i32`. Source and target callees both normalize to foreign symbol `ping`; the
integer arguments are respectively `1` and `2`.

### A6-FOREIGN-02 `scanf_run_emits_the_exact_anchorless_observation`

Use the accepted transformation from the recorded B01 run:

```rust
unsafe extern "C" {
    fn scanf(format: *const i8, ...) -> i32;
}
unsafe fn source_copy() -> i32 {
    #[proctor(0)]
    let mut x: i32 = 0;
    #[proctor(1)]
    scanf(b"%d\0" as *const u8 as *const i8, &mut x as *mut i32)
}
unsafe fn target() -> i32 {
    #[proctor(0)]
    let mut x: i32 = 0;
    #[proctor(1)]
    xj_scanf::legacy::scanf("%d", &mut [&mut x])
}
```

Run this exact source through the existing in-memory `run_compiler_on_str`
harness after the normal `deps_crate` prerequisite has provided the real
`xj_scanf` rlib and `utils::compilation::make_config` has registered it as an
extern. Call `extract_case(source, "source_copy", "target", vec![1])`.
Expected: the complete observation is exactly `SCAN_OBSERVATION` from Section
3. Assert the target callee's compiler-normalized identity is external crate
`xj_scanf`, path `legacy::scanf`; a textual or local fake is not acceptable.
`x` is `<id0>` in both trees but is not a pointer anchor. This case is
mandatory, not environment-conditional.

### A6-FOREIGN-03 `foreign_call_absorbs_pointer_argument_anchor`

Input:

```rust
unsafe extern "C" { fn read_one(p: *const i32) -> i32; }
unsafe fn source_copy(mut p: *const i32) -> i32 {
    #[proctor(0)] read_one(p)
}
unsafe fn target(mut p: &i32) -> i32 {
    #[proctor(0)] read_one(p as *const i32)
}
```

Expected: one call-root observation, not a call region plus a child path
region. Its sole anchor is `p`; its source pattern contains the same `<id0>` in
the call argument; its root type is `i32`.

### A6-FOREIGN-04 `foreign_call_absorbs_two_arguments_in_source_order`

Input direct `compare(left, right)` where both arguments are eligible raw
pointer bindings and the target casts references back to raw pointers.
Expected: one call-root observation with anchors `left`, `right`, in argument
order. Reverse the parameter declaration order without changing argument order;
anchor order remains expression occurrence order.

### A6-FOREIGN-05 `dereference_grows_from_pointer_returning_call`

Input:

```rust
unsafe extern "C" { fn strchr(s: *const i8, c: i32) -> *mut i8; }
unsafe fn source_copy(mut s: *const i8) -> i8 {
    #[proctor(0)] *strchr(s, 97)
}
unsafe fn target(mut s: &[i8]) -> i8 {
    #[proctor(0)] s[0]
}
```

Expected: one observation rooted at the complete dereference, not at the call.
The nested `s` anchor transfers to it. Source intrinsic/adjusted type is `i8`;
target intrinsic/adjusted type is `i8`; `lhs` is false.

### A6-FOREIGN-06 `pointer_returning_call_can_be_anchorless`

Input:

```rust
unsafe extern "C" { fn allocate() -> *mut i32; }
unsafe fn source_copy() -> i32 {
    #[proctor(0)] *allocate()
}
unsafe fn target() -> i32 {
    #[proctor(0)] 0
}
```

Expected: one dereference-root observation with no anchors and scalar `i32`
root types. This is extraction only; it does not imply an applicable rule in a
context where target type inference is unavailable.

### A6-FOREIGN-07 `nested_foreign_calls_keep_only_the_outer_call`

Input:

```rust
unsafe extern "C" {
    fn inner() -> i32;
    fn outer(value: i32) -> i32;
}
unsafe fn source_copy() -> i32 {
    #[proctor(0)] outer(inner())
}
unsafe fn target() -> i32 {
    #[proctor(0)] outer(inner() + 1)
}
```

Expected: one observation rooted at `outer(inner())`. The `inner()` seed is a
strict descendant and emits no second observation.

### A6-FOREIGN-08 `disjoint_foreign_calls_remain_separate`

Input `left() + right()` for two locally declared C-ABI functions and target
`1 + 2`. Expected: two anchorless observations in left, right source order,
mapping respectively to literals `1` and `2`. Independently change the target
parent operator to subtraction; expected: the complete statement is rejected
because selected roots do not make their nonregion parent arbitrary.

### A6-FOREIGN-09 `link_name_is_the_semantic_identity`

Input:

```rust
unsafe extern "C" {
    #[link_name = "scanf"]
    fn rust_scan(format: *const i8, ...) -> i32;
}
unsafe fn source_copy() -> i32 {
    #[proctor(0)] rust_scan(core::ptr::null())
}
```

Expected: the call is a seed and its callee normalizes to
`{"kind":"foreign_function","symbol":"scanf"}`, not `rust_scan`. The
containing function record has `foreign_function_names` exactly
`["rust_scan", "scanf"]`; loading that record in PROCTOR activates the existing
xj-scanf prompt guidance without changing the prompt template.

### A6-FOREIGN-10 `defined_extern_c_function_is_not_a_seed`

Input:

```rust
unsafe extern "C" fn defined(value: i32) -> i32 { value }
unsafe fn source_copy() -> i32 {
    #[proctor(0)] defined(1)
}
unsafe fn target() -> i32 {
    #[proctor(0)] defined(2)
}
```

Expected: zero observations when there is no pointer anchor. A function
definition with C ABI is source-defined, not a foreign item declaration.

### A6-FOREIGN-11 `other_foreign_abis_are_not_seeds`

Use separate compiling fixtures for `extern "C-unwind"` and a platform-
supported non-C ABI such as `extern "system"`. Each contains one direct call
and no pointer anchor. Expected: zero observations. Gate the platform-specific
ABI fixture exactly as existing rustc tests do; the exact C-unwind negative is
mandatory on all supported test platforms.

### A6-FOREIGN-12 `indirect_foreign_function_pointer_is_not_a_seed`

Input:

```rust
unsafe fn source_copy(mut f: unsafe extern "C" fn(i32) -> i32) -> i32 {
    #[proctor(0)] f(1)
}
unsafe fn target(mut f: unsafe extern "C" fn(i32) -> i32) -> i32 {
    #[proctor(0)] f(2)
}
```

Expected: zero observations. The function pointer type does not make a direct
foreign item resolution.

### A6-FOREIGN-13 `dependency_owned_foreign_item_is_not_a_seed`

In a compiler fixture that resolves `libc::free`, call it with no eligible
local raw-pointer anchor (for example a constant null expression). Expected:
zero observations. Assert the resolved `DefId` is nonlocal so the test does not
accidentally cover a local shim.

### A6-FOREIGN-14 `ordinary_external_rust_call_is_not_a_seed`

Input a direct `std::cmp::max(1, 2)` transformation with no pointer anchor.
Expected: zero observations. External Rust functions and foreign declarations
are distinct.

### A6-FOREIGN-15 `macros_still_skip_the_complete_label`

Input one labeled statement containing a supported foreign call plus any macro
call in another argument or nested opaque subtree. Expected: the label's
`macro_skip` remains true and no observation is emitted. Independently verify
that a macro-free call transformed to an ordinary `xj_scanf` function remains
eligible.

### A6-FOREIGN-16 `multi_statement_target_group_still_skips`

Input one foreign-call source statement whose accepted target label expands to
two consecutive statements. Expected: no observation, even though the source
call is a valid seed. Foreign seeding does not relax the one-target-expression
alignment requirement.

## 7. Version-1 validation, serialization, and Markdown

### A6-WIRE-01 `anchorless_observation_round_trips_canonically`

Input the exact document:

```json
{
  "schema_version": 1,
  "observations": [
    {
      "source_expression": {
        "kind": "call",
        "callee": {"kind":"path","value":{"kind":"foreign_function","symbol":"ping"}},
        "arguments": []
      },
      "target_expression": {
        "kind": "literal",
        "value": {"kind": "integer", "value": "0", "type": "i32"}
      },
      "pointer_anchors": [],
      "lhs": false,
      "source_type": {"kind":"primitive","name":"i32"},
      "source_adjusted_type": {"kind":"primitive","name":"i32"},
      "target_type": {"kind":"primitive","name":"i32"},
      "target_adjusted_type": {"kind":"primitive","name":"i32"}
    }
  ]
}
```

Expected: validation succeeds, deserialization/serialization is stable, and
pretty canonical JSON retains `"pointer_anchors": []` in the existing member
position. `schema_version` remains 1.

### A6-WIRE-02 `anchorless_rule_round_trips_canonically`

Input the exact rule document:

```json
{
  "schema_version": 1,
  "rules": [
    {
      "source_pattern": {
        "kind":"call",
        "callee":{"kind":"path","value":{"kind":"foreign_function","symbol":"ping"}},
        "arguments":[{"kind":"variable","sort":"expression","index":0}]
      },
      "target_pattern": {"kind":"variable","sort":"expression","index":0},
      "pointer_anchors": [],
      "lhs": false,
      "source_type": {"kind":"primitive","name":"i32"},
      "source_adjusted_type": {"kind":"primitive","name":"i32"},
      "target_type": {"kind":"primitive","name":"i32"},
      "target_adjusted_type": {"kind":"primitive","name":"i32"}
    }
  ]
}
```

Expected: validation and round-trip succeed. The target `E0` is available from
the source pattern. No dummy anchor variable is introduced.

### A6-WIRE-03 `empty_anchor_list_does_not_declare_anchor_variables`

Starting from A6-WIRE-02, replace the source expression variable with
a path whose value identity is
`{"kind":"variable","sort":"anchor","index":0}` while leaving
`pointer_anchors: []`. Expected: validation rejects because the anchor variable
is undeclared. Permitting an empty list does not implicitly declare `A0`.

### A6-WIRE-04 `ordinary_bindings_are_valid_without_anchors`

Input a rule with empty anchors and a source/target path using
`{"kind":"variable","sort":"binding","index":0}`. Expected:
validation succeeds when `B0` first occurs in the source pattern and is reused
in the target. This is the shape needed for `scanf(..., &mut x)`.

### A6-WIRE-05 `nonempty_anchor_invariants_remain_closed`

For one otherwise valid nonempty observation/rule, independently inject:

- duplicate anchor IDs/variables;
- noncanonical anchor index `1` without `0`;
- observation anchors in reverse source occurrence order;
- an observation anchor absent from the source expression;
- a rule anchor of sort `binding` rather than `anchor`; and
- a non-raw source anchor type.

Expected: each document is rejected with the existing deterministic category.
The exact error wording may change only where the removed nonempty check made a
previous case unreachable.

### A6-WIRE-06 `empty_anchor_merge_and_canonical_sort_are_stable`

Merge two valid observation documents containing anchorless values, including
duplicates. Expected: merge preserves document/member order and duplicates as
before. Synthesize with input-document and observation order permutations;
expected canonical rule bytes are identical.

### A6-WIRE-07 `markdown_renders_anchorless_rules`

Input A6-WIRE-02 to `rule_document_to_markdown`. Expected: one ordinary rule
bullet with the source/target expression and type context, no panic, no phantom
anchor constraint, and no omission of the rule. Compare the full Markdown
string according to the existing renderer's exact format.

### A6-WIRE-08 `old_nonempty_documents_keep_exact_bytes`

Serialize representative existing one-anchor and two-anchor observation/rule
fixtures before and after the implementation. Expected: byte-for-byte equality
with their checked-in canonical strings. Only newly valid empty-anchor inputs
widen behavior.

## 8. Scan-family literal normalization and recognition

### A6-LITERAL-01 `string_syntax_normalizes_to_semantic_value`

Input expressions `"%d"`, `r"%d"`, and an escape-equivalent string literal.
Expected: each dumps as
`{"kind":"string","value":"%d"}`. Token spelling is not retained.

### A6-LITERAL-02 `byte_string_retains_explicit_zero`

Input `b"%d\0"`. Expected exact dump:

```json
{"kind":"byte_string","value":[37,100,0]}
```

An escape-equivalent byte spelling produces the same bytes.

### A6-LITERAL-03 `c_string_omits_only_the_implicit_terminator`

Input `c"%d"`. Expected exact dump:

```json
{"kind":"c_string","value":[37,100]}
```

### A6-LITERAL-04 `literal_kinds_remain_distinct`

Compare normalized `"%d"`, `b"%d"`, and `c"%d"`. Expected kinds are
respectively `string`, `byte_string`, and `c_string`; equal decoded bytes do
not make cross-kind nodes equal.

### A6-LITERAL-05 `source_scan_family_positions_are_exact`

Use normalized calls with unique string literals in every argument:

- foreign `scanf(format, target)` protects argument 0 only;
- foreign `fscanf(stream, format, target)` protects argument 1 only; and
- foreign `sscanf(input, format, target)` protects argument 1 only.

Expected: changing only the listed format rejects synthesis; changing an
unlisted string-like argument follows ordinary generalization.

### A6-LITERAL-06 `target_scan_family_positions_are_exact`

Use resolved normalized target calls:

- `xj_scanf::legacy::scanf(format, targets)` protects argument 0;
- `xj_scanf::legacy::brscanf(reader, format, targets)` protects argument 1;
- `xj_scanf::legacy::bscanf(input, format, targets)` protects argument 1.

Expected: target mismatches at those positions reject. In particular, the
`bscanf` input at argument 0 is not protected.

### A6-LITERAL-07 `recognition_uses_identity_not_spelling`

Resolve a local foreign declaration with Rust name `rust_scan` and
`#[link_name="scanf"]`, then call it through an alias. Expected: source format
argument 0 is protected. Conversely, a source-defined function named `scanf`
and an external path whose last component is `scanf` but whose resolved
identity is not the foreign symbol are not protected.

Add exact foreign-symbol near misses `vscanf` and `__isoc99_scanf`, each with
two otherwise compatible observations whose source and target calls are the
same near-miss function and whose argument-0 strings differ between `%d` and
`%u`. Expected for each pair: one nondegenerate rule, one `expression` variable
at the format literal reused in the target, and no rejection. Both calls are
valid foreign seeds, but neither is suffix-normalized to protected `scanf`.

### A6-LITERAL-08 `target_recognition_requires_exact_crate_and_path`

Normalized negative inputs:

- crate `other`, path `legacy::scanf`;
- crate `xj_scanf`, path `scanf`;
- crate `xj_scanf`, path `other::scanf`; and
- crate `xj_scanf`, path `legacy::sscanf`; and
- local function identity printed as `xj_scanf::legacy::scanf`.

Expected: none protects a format. Only the exact resolved external identities
in Section 3 do. For each normalized negative, pair two observations whose
source and target use that same negative callee and whose candidate format
strings are `%d` and `%u`. Expected: synthesis emits one rule with one
`expression` variable reused at that literal and does not return
`PairRejection::Source` or `PairRejection::TargetLookup`. In particular,
`xj_scanf::legacy::sscanf` is not treated as one of the prompt-directed target
functions.

### A6-LITERAL-09 `nested_scan_calls_protect_each_format_only`

Input a normalized outer `scanf("%d", nested_sscanf("input", "%u", x))`.
Expected: the outer argument-0 string and nested `sscanf` argument-1 string are
protected; nested input string is not. A mismatch in either protected literal
rejects; a mismatch only in nested input may generalize.

## 9. Coupled synthesis with rigid format literals

### A6-SYN-01 `identical_scan_observations_produce_one_exact_rule`

Input `doc(SCAN_OBSERVATION, SCAN_OBSERVATION)` with independent deep copies.
Expected: one anchorless rule. Its source and target patterns retain the exact
byte-string and string literals, and ordinary `<id0>` becomes one `binding`
variable reused in both patterns. No `anchor`, `expression`, or
`integer_magnitude` variable is allocated merely for the format.

### A6-SYN-02 `equal_normalized_format_spellings_are_compatible`

Extract two observations whose source format tokens differ (`b"%d\0"` versus
an escape-equivalent byte string) but normalize to `[37,100,0]`; target tokens
`"%d"` and `r"%d"` normalize to the same string. Expected: synthesis accepts
and retains concrete normalized literals.

### A6-SYN-03 `scanf_source_format_mismatch_rejects`

Input two `SCAN_OBSERVATION` values, changing only the second source byte-string
payload from `[37,100,0]` (`%d\0`) to `[37,117,0]` (`%u\0`). Keep both target
formats equal to `%d` so the source cause is isolated. Expected: no rule;
pair rejection is `Source`; no expression variable is recorded for the literal
or an enclosing cast/call.

### A6-SYN-04 `scanf_target_format_mismatch_rejects`

Input two observations with identical `SCAN_SOURCE`, but target string values
`%d` and `%u`. Expected: `PairSynthesis.rule == None`,
`PairSynthesis.rejection == Some(PairRejection::TargetLookup)`, and
`substitutions` contains no `expression` entry for the target literal, call, or
any enclosing wrapper. Source traversal does not authorize target format
generalization.

### A6-SYN-05 `byte_string_kind_mismatch_rejects`

At the protected source format position, compare
`ByteString([37,100,0])` with `CString([37,100])`. Expected: rejection even
though both semantically describe a C-compatible `%d` sequence. Kind equality
is required.

### A6-SYN-06 `string_cstring_and_byte_string_each_stay_rigid`

For each literal kind independently, synthesize two scan observations with
equal values and then unequal values. Expected: equal pairs retain a concrete
literal; unequal pairs reject. For `CString`, compare stored payloads without
an implicit terminal zero.

### A6-SYN-07 `source_and_target_literal_kinds_need_not_match_each_other`

Input two copies of the normal transformation whose source format is
`ByteString([37,100,0])` and target format is `String("%d")`. Expected: one
rule. There is no cross-side equality requirement.

### A6-SYN-08 `sscanf_input_literal_remains_generalizable`

Input two pure normalized observations with source expressions:

```text
sscanf(b"12", b"%d\0", &mut x as *mut i32)
sscanf(b"34", b"%d\0", &mut x as *mut i32)
```

and corresponding targets:

```text
xj_scanf::legacy::bscanf(b"12", "%d", &mut [&mut x])
xj_scanf::legacy::bscanf(b"34", "%d", &mut [&mut x])
```

Expected: synthesis accepts. The source input byte-string at argument 0 and
target `bscanf` input at argument 0 become/reuse the appropriate source-derived
expression disagreement because the ordered normalized disagreement pair is
identical on both sides; both format arguments remain concrete. There is no
format-related rejection. This is a pure anti-unification fixture, not a claim
that a C `sscanf` source call omits its required C-string terminator.

### A6-SYN-09 `fscanf_stream_is_not_format_rigid`

Use two normalized anchorless observations with primitive `i32` context. Their
source expressions are exactly:

```text
Call(ForeignFunction("fscanf"), [String("left"), String("%d"), Binding("<id0>")])
Call(ForeignFunction("fscanf"), [String("right"), String("%d"), Binding("<id0>")])
```

Their corresponding target expressions are exactly:

```text
Call(External("xj_scanf", "legacy::brscanf"), [String("left"), String("%d"), Binding("<id0>")])
Call(External("xj_scanf", "legacy::brscanf"), [String("right"), String("%d"), Binding("<id0>")])
```

Expected: synthesis returns one rule and no rejection. The complete source
pattern is
`Call(ForeignFunction("fscanf"), [E0, String("%d"), B0])`; the complete target
pattern is
`Call(External("xj_scanf", "legacy::brscanf"), [E0, String("%d"), B0])`.
There is exactly one expression variable and one binding variable. Argument 0
reuses `E0` across source and target, while each argument-1 format remains the
concrete `String("%d")`.

### A6-SYN-10 `other_scan_arguments_keep_current_generalization`

Run three independent normalized anchorless observation pairs with primitive
`i32` context. In each pair, source and target use the same ordered `L`, `R`
literal pair from one row:

| Literal kind | `L` | `R` |
| --- | --- | --- |
| string | `String("left")` | `String("right")` |
| byte string | `ByteString([108, 101, 102, 116])` | `ByteString([114, 105, 103, 104, 116])` |
| C string | `CString([108, 101, 102, 116])` | `CString([114, 105, 103, 104, 116])` |

For each row the two source expressions are exactly
`Call(ForeignFunction("sscanf"), [L, String("%d"), Binding("<id0>")])` and the
same call with `R`; the corresponding targets are exactly
`Call(External("xj_scanf", "legacy::bscanf"), [L, String("%d"),
Binding("<id0>")])` and the same call with `R`.

Expected for every row: synthesis returns one rule and no rejection. Its source
pattern is exactly
`Call(ForeignFunction("sscanf"), [E0, String("%d"), B0])`, and its target
pattern is exactly
`Call(External("xj_scanf", "legacy::bscanf"), [E0, String("%d"), B0])`.
There is exactly one expression variable and one binding variable; input
argument 0 reuses `E0`, and the protected argument-1 formats stay concrete.

### A6-SYN-11 `non_scan_string_literals_still_generalize`

Input:

```text
log("left", value) -> consume("left", value)
log("right", value) -> consume("right", value)
```

using the same rigid external callees in both observations. Expected: one `E0`
at the string-literal position, reused in the target. Repeat for unequal byte
strings and C strings.

### A6-SYN-12 `protected_mismatch_propagates_through_casts`

Use A6-SYN-03's unequal source byte strings below the two cast nodes in
`SCAN_SOURCE`. Expected: `Reject` propagates through both casts, the argument
list, and the call. The root call does not become `E0`.

### A6-SYN-13 `protected_mismatch_propagates_through_nested_expression`

Define these four exact normalized calls:

- `S_d` and `S_u` are identical foreign-symbol `scanf` calls except argument 0
  is respectively `String("%d")` and `String("%u")`;
- `T_d` and `T_u` are identical external
  `xj_scanf::legacy::scanf` calls except argument 0 is respectively
  `String("%d")` and `String("%u")`.

All other arguments are the same empty target array expression, and every root
context type is primitive `i32`. For every wrapper `W` in the table below run
both exact pairs:

```text
source rejection: obs(W(S_d), W(T_d)) with obs(W(S_u), W(T_d))
target rejection: obs(W(S_d), W(T_d)) with obs(W(S_d), W(T_u))
```

The first pair must return `rule: None`,
`rejection: Some(PairRejection::Source)`; the second must return `rule: None`,
`rejection: Some(PairRejection::TargetLookup)`. In both, substitutions contain
no `expression` entry for the protected literal, scan call, wrapper, or any
ancestor.

| Propagation site | Exact normalized wrapper `W(x)` |
| --- | --- |
| array expression list | `Array { elements: [x] }` |
| tuple expression list | `Tuple { elements: [Integer(0), x] }` |
| call callee | `Call { callee: x, arguments: [] }` |
| call argument list | `Call { callee: External(fixture::keep), arguments: [x] }` |
| method receiver | `MethodCall { receiver: x, method: External(fixture::Trait::keep), arguments: [] }` |
| method argument list | `MethodCall { receiver: Integer(0), method: External(fixture::Trait::keep), arguments: [x] }` |
| binary left | `Binary(Add, x, Integer(0))` |
| binary right | `Binary(Add, Integer(0), x)` |
| assignment left | `Assign { left: x, right: Integer(0) }` |
| assignment right | `Assign { left: Binding(<id0>), right: x }` |
| assignment-operator left | `AssignOp(Add, x, Integer(0))` |
| assignment-operator right | `AssignOp(Add, Binding(<id0>), x)` |
| unary operand | `Unary(Not, x)` |
| cast operand | `Cast { expression: x, type: Primitive(i32) }` |
| field base | `Field { base: x, field: External(fixture::Record::field) }` |
| index base | `Index { base: x, index: Integer(0) }` |
| index index | `Index { base: Binding(<id0>), index: x }` |
| range optional start | `Range { start: Some(x), end: Some(Integer(1)), half_open }` |
| range optional end | `Range { start: Some(Integer(0)), end: Some(x), half_open }` |
| if condition | `If { condition: x, then: block_tail(Integer(0)), else: Some(Integer(1)) }` |
| if then block | `If { condition: Bool(true), then: block_tail(x), else: Some(Integer(1)) }` |
| if optional else | `If { condition: Bool(true), then: block_tail(Integer(0)), else: Some(x) }` |
| while condition | `While { condition: x, body: empty_block }` |
| while body block | `While { condition: Bool(true), body: block_semicolon(x) }` |
| loop body block | `Loop { body: block_semicolon(x) }` |
| struct field value/list | `Struct(fixture::Record) { field: x, other: Integer(0), rest: None }` |
| struct optional rest | `Struct(fixture::Record) { field: Integer(0), rest: Some(x) }` |
| address operand | `AddressOf(reference, const, x)` |
| optional return value | `Return { value: Some(x) }` |
| optional break value | `Break { value: Some(x) }` |
| repeat value | `Repeat { value: x, count: Integer(1, usize) }` |
| repeat count | `Repeat { value: Integer(0), count: x }` |
| explicit block | `Block { block: block_semicolon(x) }` |
| block expression statement | `Block { block: [Expression { expression: x, semicolon: true }] }` |
| block let initializer | `Block { block: [Let { pattern: Binding(<id0>), type: None, initializer: Some(x) }] }` |

`block_tail(x)` is exactly one normalized expression statement with
`semicolon: false`; `block_semicolon(x)` uses `semicolon: true`; `empty_block`
has `statements: []`. Fixed local IDs and external identities are identical in
the paired terms. This matrix covers every current child-bearing
`expression_inner`, expression-list, optional-expression, `struct_expression`,
and normalized-block route. `Path`, `Literal`, and `Continue` are leaves; the
protected literal's direct rejection is covered by A6-SYN-03--06.

Add list/optional topology subcases in which only the left term contains a
protected call: `Array([S_d])` versus `Array([])`, `Return(Some(S_d))` versus
`Return(None)`, `Break(Some(S_d))` versus `Break(None)`,
`Range(Some(S_d), None)` versus `Range(None, None)`, an `if` with
`else: Some(S_d)` versus `else: None`, and a struct with `rest: Some(S_d)`
versus `rest: None`. Each must preflight-reject as
`PairRejection::Source` with no enclosing `E`, rather than use the existing
list-arity or optional-presence `Generalize` result.

### A6-SYN-14 `block_expression_statement_preserves_reject`

Input source patterns whose differing scan calls occur in a normalized block:

```json
{
  "kind":"block",
  "block":{"statements":[{
    "kind":"expression",
    "expression":"SCAN_CALL_WITH_FORMAT",
    "semicolon":true
  }]}
}
```

Compare `%d` with `%u`. Expected: no rule. This specifically catches the old
`block` expression-statement branch that converted every non-`Ok` child into
`Generalize`. Run the source pair and target pair from A6-SYN-13 and require
respectively exact `PairRejection::Source` and
`PairRejection::TargetLookup`, with no enclosing `E`.

### A6-SYN-15 `block_generalization_behavior_is_otherwise_unchanged`

In the same block shape, change an ordinary nonprotected expression in the
statement. Expected: `Generalize` may still bubble to the permitted enclosing
expression exactly as before. The block fix must distinguish `Reject` from
other non-`Ok` outcomes rather than making blocks universally rigid.

### A6-SYN-16 `let_initializer_preserves_reject`

Put the differing scan call in a normalized `let` initializer inside a block,
using equal binding pattern/type on both sides. Expected: the pair rejects and
does not generalize the `let`, block, or enclosing expression.

### A6-SYN-17 `target_block_mismatch_cannot_reuse_source_disagreement`

Use equal protected source formats but unequal protected target formats inside
normalized expression-statement blocks. Expected: target synthesis rejects;
`PairSynthesis.rejection` is exactly `Some(PairRejection::TargetLookup)`, and it
cannot use an unrelated source `E` disagreement elsewhere in the source
pattern. Assert no `expression` substitution equals either protected target
literal, scan call, or block.

### A6-SYN-17A `one_sided_protection_rejects_before_descent`

Source-side fixture: pair a foreign `scanf("%d", ...)` call with a structurally
corresponding foreign `vscanf("%d", ...)` call. Target-side fixture: keep source
expressions identical and pair external `xj_scanf::legacy::scanf("%d", ...)`
with external `xj_scanf::legacy::sscanf("%d", ...)`. In each pair exactly one
protected-location map marks the candidate format position.

Expected source result is exactly `PairRejection::Source`; expected target
result is exactly `PairRejection::TargetLookup`. Both have `rule: None` and no
new `expression` substitution, even though callee traversal would otherwise be
able to generalize an enclosing call before reaching the argument. Repeat the
same two one-sided cases inside `Block { block: block_semicolon(x) }`; outcomes
are unchanged.

### A6-SYN-18 `different_scan_family_callees_do_not_cross_synthesize`

Pair a foreign `scanf` observation with a foreign `sscanf` observation, even
if both protected format values equal `%d`. Expected: existing rigid foreign
callee identity semantics prevent a useful nondegenerate rule; format policy
does not merge function families.

### A6-SYN-19 `anchorless_binding_carriers_remain_sound`

Use two anchorless scan observations whose `x` binding appears explicitly in
both source and target. Expected: one ordinary binding carrier. Add a second
case where the same binding is split between an explicit binding variable and
an expression-variable substitution. Expected: existing carrier validation
rejects even though the anchor set is empty.

### A6-SYN-20 `mixed_empty_and_nonempty_anchor_contexts_do_not_pair`

Pair two otherwise equal observations, one with `pointer_anchors: []` and one
with one valid anchor. Expected: context rejection and no rule. Empty anchors
are a real exact context, not a wildcard.

## 10. Matching, specificity, and ranking with empty anchors

### A6-MATCH-01 `anchorless_rule_matches_only_anchorless_region`

Input A6-WIRE-02 and a `RuleMatchInput` for `ping(7)` with empty anchors and
all scalar `i32` context. Expected: match succeeds and binds `E0` to literal
`7`. Add one region anchor without changing the expression; expected: match
fails due to exact anchor count.

### A6-MATCH-02 `anchored_rule_does_not_match_anchorless_region`

Use an existing one-anchor dereference rule against the same source expression
but with an empty region anchor list. Expected: no match; there is no implicit
anchor binding.

### A6-MATCH-03 `ordinary_binding_variables_match_anchorless_calls`

Rule source `scanf(FORMAT, &mut B0)` and target `safe(FORMAT, &mut B0)`, empty
anchors. Region contains concrete binding `<id0>`. Expected: `B0` binds and is
substituted in the target. A second distinct binding variable remains
injective under existing namespace rules.

### A6-MATCH-04 `empty_anchor_context_does_not_change_specificity_groups`

Create an exact-call rule and a more general call-with-`E0` rule, both empty-
anchor and same types/lhs. Expected: source-pattern subsumption and alpha groups
are computed solely as before; the exact applicable group wins.

### A6-RANK-01 `anchorless_rules_use_the_existing_ranking_pipeline`

Use `pointer_anchors: []`, `lhs: false`, and matching context types in each of
these exact independent subcases. Reverse the listed rule order in every
subcase and require the same winner.

1. **Specificity.** Concrete source is `ping(1_i32)`. Rule G is
   `ping(E0) -> 20_i32`; rule S is `ping(1_i32) -> 10_i32`. Expected target is
   exactly `10_i32` from S.
2. **Distinct source-substitution cost.** Concrete source is tuple
   `((b + 1_i32), (b + 1_i32), 0_i32)`. Rule R is
   `(E0, E0, 0_i32) -> 7_i32`; rule D is
   `((E0 + E1), (E0 + E1), E2) -> 9_i32`. Use the current normalized term-size
   function and assert R cost `8`, D cost `10`, neither pattern subsumes the
   other, and exact target `7_i32` wins.
3. **Target size.** Both rules have exact source `ping(1_i32)`. Rule Small
   targets `1_i32`; rule Large targets `(1_i32 + 2_i32)`. Expected target is
   exactly `(1_i32 + 2_i32)`, and the reported target size equals
   `normalized_term_size(Large.target_pattern)`.
4. **Canonical target JSON.** Both rules have exact source `ping(1_i32)` and
   one-node targets `1_i32` and `2_i32`. Expected target is exactly `1_i32`.
5. **Canonical full-rule JSON.** Use an anchorless pointer-like concrete source
   `allocate()` with identical source pattern, target pattern, source types,
   target adjusted type, and target expression in two rules. Let only the
   dormant target intrinsic types differ as primitive `i32` versus primitive
   `usize`; pointer-root matching ignores that intrinsic position. Both are
   applicable and all earlier ranks tie. Expected selected full rule is the
   one with primitive `i32`, the lexicographically smaller canonical full-rule
   JSON; inspect the selected rule index/value because target expressions are
   equal.

No subcase may acquire an implicit anchor or a different winner merely because
the anchor list is empty.

### A6-RANK-02 `unmaterializable_foreign_winner_falls_back`

Use this compiler input, which deliberately has no declaration linked as
`c_ping`:

```rust
unsafe extern "C" {
    fn source_ping(value: i32) -> i32;
}
unsafe fn f() -> i32 {
    #[proctor(0)] source_ping(1)
}
```

Both rules have `pointer_anchors: []`, `lhs: false`, and all four context types
equal to primitive `i32`:

- rule U has exact source
  `Call(ForeignFunction("source_ping"), [Integer(1, i32)])` and target
  `Call(ForeignFunction("c_ping"), [Integer(1, i32)])`;
- rule F has source
  `Call(ForeignFunction("source_ping"), [E0])` and target `Integer(7, i32)`.

Run with both rule-document orders. In each order, test-internal directional
specificity reports U at least as specific as F and F not at least as specific
as U, so their alpha groups differ and U's group dominates F's. The initial
`select_with_exclusions` result has U's `rule_index`,
`alpha_group == loaded.alpha_group(U_index)`, `substitution_cost == 0`,
`target_size == 7`, and exact target `c_ping(1_i32)`. Materialization then
rejects only U because no accessible declaration resolves linked symbol
`c_ping`; it emits no bare callee spelling.

After inserting exactly U's index into the exclusion set, the rerun selects F
with F's `rule_index`, `alpha_group == loaded.alpha_group(F_index)`,
`substitution_cost == 4`, `target_size == 4`, and exact normalized target
`7_i32`. Applied output is exactly `7_i32` at label 0 with disposition
`rule_applied`. Reversing document order changes numeric indices but none of
these rule identities, keys, selections, or output.

## 11. Foreign spelling and materialization

### A6-SPELL-01 `link_name_reuses_the_matched_rust_name`

Compiler input:

```rust
unsafe extern "C" {
    #[link_name = "c_ping"]
    fn rust_ping(value: i32) -> i32;
}
unsafe fn f() -> i32 {
    #[proctor(0)] rust_ping(1)
}
```

Apply an anchorless rule that retains foreign symbol `c_ping` but changes the
argument to `2`. Expected applied statement calls `rust_ping(2)`, not
`c_ping(2)`. The normalized match key remains `c_ping`.

### A6-SPELL-02 `module_qualified_foreign_path_is_retained`

Compiler input:

```rust
mod ffi {
    unsafe extern "C" {
        #[link_name = "c_ping"]
        pub fn rust_ping(value: i32) -> i32;
    }
}
unsafe fn f() -> i32 {
    #[proctor(0)] ffi::rust_ping(1)
}
```

Expected replacement calls `ffi::rust_ping(2)`. Bare `c_ping` and bare
`rust_ping` are not emitted in this scope.

### A6-SPELL-03 `import_alias_is_retained`

Add `use ffi::rust_ping as alias;` and make the source call `alias(1)`.
Expected replacement uses `alias(2)`. Matching still compares the linked
symbol, not alias text.

### A6-SPELL-04 `target_only_foreign_identity_uses_unique_accessible_declaration`

Compiler input:

```rust
mod ffi {
    unsafe extern "C" {
        #[link_name = "source_ping"]
        pub fn source_rust(value: i32) -> i32;
        #[link_name = "c_ping"]
        pub fn target_rust(value: i32) -> i32;
    }
}
unsafe fn f() -> i32 {
    #[proctor(0)] ffi::source_rust(1)
}
```

Apply an anchorless rule with exact source
`ForeignFunction("source_ping")(1_i32)` and target
`ForeignFunction("c_ping")(1_i32)`. The target identity has no matched source
occurrence. Expected: the unique accessible declaration resolver chooses the
target declaration `DefId`, scope-aware spelling emits
`ffi::target_rust(1)`, parsing and shape admission succeed, and label 0 becomes
`rule_applied`. It must not emit `c_ping(1)` or `source_rust(1)`.

### A6-SPELL-05 `missing_or_ambiguous_foreign_declaration_is_a_candidate_miss`

Use the exact source declaration/call for `source_ping` from A6-SPELL-04 and a
preferred rule targeting `ForeignFunction("c_ping")(1_i32)`. Add a less-ranked
materializable fallback rule targeting literal `7_i32`; arrange canonical
ordering so the foreign rule is selected first, then excluded on spelling
failure. Run these complete compiler fixtures independently:

1. **Missing:** omit every declaration with linked symbol `c_ping`.
2. **Ambiguous:** add both
   `left::left_rust` and `right::right_rust` as public local exact-C-ABI
   declarations with `#[link_name = "c_ping"]`; neither occurs in the matched
   source expression.
3. **Inaccessible:** put the sole `#[link_name = "c_ping"] fn hidden_rust`
   declaration in a sibling module without visibility from `f`'s module.

Expected in each fixture: the preferred rule is an ordinary candidate miss,
is added to the exclusion set, the complete ranking pipeline reruns, the
fallback produces exact applied expression `7_i32`, and label 0 is
`rule_applied`. With the fallback removed, the original
`ffi::source_rust(1)` remains unchanged and the label remains `transform`.
The rule document still validates; no candidate emits bare `c_ping`.

### A6-SPELL-06 `plain_declaration_name_still_materializes`

Input a declaration `fn ping` without `link_name` and a source call `ping(1)`.
Expected replacement `ping(2)`. This prevents the spelling fix from regressing
the simple case.

### A6-SPELL-07 `foreign_static_behavior_is_not_accidentally_changed`

Use a direct renderer fixture for
`ValueIdentity::ForeignStatic { symbol: "ERRNO" }` with no syntax override.
Expected output remains exactly the bare spelling `ERRNO`, and the existing
foreign-static normalized matching/validation tests remain byte-identical.
Foreign statics do not become selection seeds, do not use the new occurrence-
specific function provenance, and receive no `link_name`/module/alias widening
in this amendment.

### A6-SPELL-08 `same_symbol_provenance_is_occurrence_specific`

Compiler input:

```rust
mod left {
    unsafe extern "C" {
        #[link_name = "c_ping"]
        pub fn left_rust(value: i32) -> i32;
    }
}
mod right {
    unsafe extern "C" {
        #[link_name = "c_ping"]
        pub fn right_rust(value: i32) -> i32;
    }
}
unsafe extern "C" { fn combine(left: i32, right: i32) -> i32; }
use left::left_rust as first;
use right::right_rust as second;
unsafe fn f() -> i32 {
    #[proctor(0)] combine(first(1), second(2))
}
```

Both nested callees normalize to `ForeignFunction("c_ping")`, but their source
expression ordinals, declaration `DefId`s, and spellings differ. Apply an
anchorless rule whose exact source is the complete `combine(first(1),
second(2))` normalized tree and whose target is the uniquely matching retained
subtree `ForeignFunction("c_ping")(2_i32)`. Expected applied expression is
exactly `second(2)`, never `first(2)` and never `c_ping(2)`. This must pass when
rule order is reversed with an irrelevant rule and proves provenance is not a
single symbol-to-spelling map.

## 12. Applied-view behavior and atomicity

### A6-APPLY-01 `anchorless_foreign_rule_marks_statement_applied`

Input source statement `rust_ping(1)` from A6-SPELL-01 and one matching
anchorless rule to `rust_ping(2)`. Expected applied view contains the complete
replacement and marks the statement `rule_applied`; baseline view is unchanged.
No LLM hole remains for a rule-complete statement.

### A6-APPLY-02 `foreign_parent_rule_supersedes_child_rule`

Input `read_one(p)` with both:

- a rule matching the maximal call region; and
- a rule that would match the nested pointer path/cast if selected separately.

Expected: only the call rule is considered and installed. Removing the call
rule leaves the maximal call region uncovered and the statement unchanged; the
child rule is not attempted as fallback.

### A6-APPLY-03 `maximal_parent_rule_receives_transferred_anchor_context`

Input A6-FOREIGN-03 and a call-root rule declaring one anchor. Expected: the
rule matches `p` through the transferred anchor, verifies its source/selected
target types, and materializes the call replacement. A zero-anchor otherwise
identical rule does not match this anchored maximal region.

### A6-APPLY-04 `all_disjoint_maxima_install_simultaneously`

Input one statement containing two disjoint foreign calls and one pointer-only
region, with one applicable rule per root and replacements of different
rendered lengths. Expected: all three install from original AST identities in
one clone, independent of rule-document order.

### A6-APPLY-05 `one_uncovered_maximal_region_rolls_back_the_statement`

Starting from A6-APPLY-04, remove or make unmaterializable any one rule.
Expected: none of the other two replacements is retained and the statement's
applied view stays baseline/transform. Nested labels continue to be considered
independently according to existing traversal.

### A6-APPLY-06 `pairwise_disjoint_postcondition_prevents_installer_overlap`

Instrument or directly inspect selected replacement roots for a statement with
several ancestor/descendant candidates. Expected: after coalescing, no retained
root is an ancestor of another. `RuleExpressionInstaller` receives only
pairwise-disjoint keys and never depends on traversal order to resolve overlap.

### A6-APPLY-07 `pointer_root_still_requires_target_context`

Input an anchorless pointer-valued foreign call used as a discarded semicolon
statement and an applicable-looking pointer-root rule. Expected: no application
because contextual target adjusted type is unavailable. Put the same root in a
supported return or direct-call-argument context with a matching target type;
expected: it becomes applicable. Foreign seeding does not bypass target
context.

### A6-APPLY-08 `scalar_foreign_root_needs_no_new_context_producer`

Input scalar-returning `ping(1)` with all scalar source/target rule context
equal to `i32`. Expected: anchorless rule application succeeds using the
existing non-pointer-root comparison to unchanged source types. No contextual
target inference is requested.

### A6-APPLY-09 `lhs_is_checked_after_maximalization`

Provide one maximal retained plain-assignment LHS region and rules identical
except `lhs`. Expected: only `lhs: true` matches. For a retained non-LHS parent
that absorbed an LHS-descendant candidate, only `lhs: false` matches.

### A6-APPLY-10 `macro_and_shape_misses_remain_local`

Use this compiler fixture:

```rust
unsafe extern "C" {
    fn ping() -> i32;
}
unsafe fn f() {
    #[proctor(0)]
    let value: i32 = ping();
    #[proctor(1)]
    let keep: i32 = 1;
}
```

Provide two matching anchorless rules. The more highly ranked rule has source
`Call(ForeignFunction("ping"), [])` and target
`Array([While { condition: Bool(true), body: empty_block }])`, rendered as
`[while true {}]`; the fallback has the same source and target literal `7_i32`.
The larger first target wins the initial target-size tie-break, but structural
admission rejects it because it nests a control expression under a non-control
array. Expected: that rule alone becomes an ordinary candidate miss, ranking
reruns, label 0 becomes exactly `let value: i32 = 7_i32;` and is
`rule_applied`, and label 1 remains exactly `let keep: i32 = 1;` with
disposition `preserve` in both baseline and applied views. With the fallback
removed, annotated source retains exactly `let value: i32 = ping();`, while
both `baseline.skeleton` and the unapplied `applied.skeleton` contain the
canonical `todo!()` transform hole for label 0 with disposition `transform`.
Label 1 remains exactly `let keep: i32 = 1;` with disposition `preserve` in
both views, and processing still succeeds.

Independently, observation extraction still skips source/target macro labels as
specified; the closed rule grammar itself has no macro constructor.

### A6-APPLY-11 `observation_and_application_select_identical_maxima`

Extend the existing shared-selector parity fixture with:

- one strict pointer ancestry overlap;
- one foreign-call ancestor with nested pointer anchors;
- one promoted-field descendant under a foreign-call root; and
- two disjoint roots of different seed kinds.

Expected: observation extraction and `select_rule_regions` return the same
root identities/order, anchor binding/order, `lhs`, and root-local promotion
state before each consumer adds target-only data.

### A6-APPLY-12 `rule_application_does_not_change_scheduling_protocol`

Implement this regression in
`proctor/tests/test_local_transformation.py`, using its existing `tmp_path`,
`FakeTools`, `FakeClient`, `apply_rules`, and `run_fake` harness. Supply a
schema-version-1 input rule set with `pointer_anchors: []`; have `FakeTools`
return an applied-view skeleton whose matching label is `rule_applied`, as it
already does for opaque rule-set tests. Run these exact subcases:

1. a rule-complete singleton SCC has no LLM or validation calls, performs no
   observation extraction, and installs its mechanical candidate;
2. a mixed SCC's first applied-view candidate fails Cargo, after which every
   SCC member uses its baseline view for the sole fallback path and shares the
   existing repair budget; and
3. accepted SCCs are published in scheduler order with the existing metrics
   keys/values and no field derived from the empty anchor list.

Expected event order, selected views, call counts, budgets, final project
contents, and statistics are byte-for-byte the same as the corresponding
existing non-anchorless tests. No new Python field, metric, prompt input, or
schema is introduced.

## 13. End-to-end and regression matrix

### A6-E2E-01 `recorded_scanf_transformation_synthesizes_when_repeated`

Produce two build-accepted observations matching the recorded B01 shape, with
different ordinary binding identities before anonymization but the same
normalized `%d` formats. Expected merged observations each contain one
anchorless call region; offline synthesis emits one canonical anchorless rule;
pretty printing succeeds. Applying the rule to a third matching call produces
the `xj_scanf::legacy::scanf("%d", &mut [&mut x])` expression when the external
dependency is resolvable.

### A6-E2E-02 `different_scan_formats_do_not_share_a_rule`

Repeat A6-E2E-01 with `%d` and `%u`. Expected: both observations are valid and
published, but their pair yields no generalized rule. Two equal `%d`
observations may still yield a `%d` rule and two equal `%u` observations may
yield a `%u` rule through normal pair enumeration/deduplication.

### A6-E2E-03 `existing_pointer_rule_corpus_is_stable_without_foreign_calls`

Run the current observation, synthesis, matching, and applied-view fixtures
whose selected statements contain no supported foreign call and no strict
overlap. Expected: canonical documents, selected rules, applied skeletons, and
ordering are byte-identical. This guards the shared-selector refactor.

### A6-E2E-04 `foreign_seed_does_not_make_preserved_statements_observations`

Place an unchanged scalar foreign call in a label classified `preserve` rather
than `transform`. Expected: no observation and no rule application attempt at
that label. Seeding changes region eligibility, not statement-disposition
ownership.

### A6-E2E-05 `failed_or_superseded_attempts_still_publish_nothing`

Implement this regression in
`proctor/tests/test_local_transformation.py` with the established `tmp_path`,
`FakeTools`, `FakeClient`, and fake observation documents. Exercise a
structurally invalid attempt, then a Cargo-failing attempt, then a successful
repair. Configure the one post-acceptance extraction result as a
schema-version-1 document whose only region has `pointer_anchors: []` and
sentinel `"producer": "accepted"`. Expected: there is exactly one
`extract_observations` event, after the last Cargo build; merge receives only
that extracted path; and the published document contains only the accepted
sentinel. Thus neither rejected attempt can create an observation at all. In a
second rule-complete mechanical subcase, expect no `extract_observations`
event; final empty-observation publication may still invoke the normal merge
with an empty input tuple, exactly as the existing stage protocol does.

### A6-E2E-06 `version_one_protocols_remain_opaque_to_python`

Implement this regression in
`proctor/tests/test_local_transformation.py`. Make the fake extraction tool
write a schema-version-1 document containing one otherwise opaque region with
`pointer_anchors: []` and a sentinel member nested inside its expression. Make
the fake merge tool assert that the input bytes retain both the empty list and
sentinel, then return a distinct canonical document. Expected: Python passes
the extracted path to merge without decoding, validating, rewriting, or
dropping region members and publishes the fake merge result byte-for-byte.
Stage input/output envelopes, event order, and statistics equal the existing
nonempty-anchor fixture.

## 14. Completion matrix

Implementation is not complete until the test suite demonstrates every row:

| Requirement | Required cases |
| --- | --- |
| inclusion-maximal coalescing | A6-UPDATE-01--02, A6-SELECT-01--10 |
| deterministic foreign seeds | A6-FOREIGN-01--09, A6-FOREIGN-14 |
| exclusion boundary | A6-FOREIGN-10--13 |
| macro/alignment boundary | A6-FOREIGN-15--16 |
| anchorless v1 documents | A6-WIRE-01--08 |
| literal normalization | A6-LITERAL-01--04 |
| exact scan positions/identities | A6-LITERAL-05--09 |
| rigid paired formats | A6-SYN-01--07, A6-SYN-12--18, A6-SYN-17A |
| generalization elsewhere | A6-UPDATE-04, A6-SYN-08--11, A6-SYN-15 |
| empty-anchor carrier/matching | A6-SYN-19--20, A6-MATCH-01--04 |
| ranking | A6-RANK-01--02 |
| foreign Rust spelling | A6-SPELL-01--08 |
| atomic application and context | A6-APPLY-01--12 |
| whole-flow regressions | A6-E2E-01--06 |
| linked-symbol prompt guidance | A6-UPDATE-05, A6-FOREIGN-09 |

The final verification report must list every exact command run and its result.
All required focused cases above are mandatory; do not omit a case merely
because it needs the existing compiler/dependency or `tmp_path` fake harness.
The report must not claim semantic correctness from compilation alone: the
prototype still does not run a behavioral test package during local
transformation.
