# Phase 7 Test Plan: Deterministic Rule Application

## 1. Purpose and test discipline

This is the exhaustive verification contract for the
[Phase 7 rule-application plan](phase-7-plan.md). It covers the Rust migration
of observation/rule ownership, the revised version-1 wire grammar, shared
region selection and `lhs`, typed matching and carrier checks, target-type
inference, specificity/ranking, target materialization, dual skeleton views,
recursive canonical restoration, optional PROCTOR input plumbing, fallback,
and opaque observation publication.

Tests must exercise the smallest owning layer. Pure wire, synthesis, matching,
ranking, and term-size tests do not invoke rustc. Compiler-backed selector,
type-inference, materialization, skeleton, validator, and replacer tests use
the existing in-memory source-string harness under `crates/tools`; they do not
invoke `crat-tool`, change filesystem state, or create a project-root `tests/`
directory. Python tests use fake tools/projects/providers and remain offline.
No test invokes `crat-tool` or tests its command-line parsing, argv, path-node
policy, publication, temporary-file, rename, or cleanup behavior. Rust tests
exercise `crates/tools` values and pure/in-memory operations directly. PROCTOR
tests exercise stage logic through fakes without adding CLI or real-filesystem
coverage merely for this work.

Every Rust-like pattern below is test notation for the corresponding closed
normalized term. `[A0]`, `[B0]`, `[E0]`, `[N0]`, `[F0]`, and `[M0]` mean
`anchor`, `binding`, `expression`, `integer_magnitude`, `function`, and
`method` variable nodes with index zero. Tests compare semantic terms, not
these display strings.

Case IDs such as `P7-MATCH-01` are planning-document references only. Use the
descriptive backtick name (or an equally behavioral name) for implementation
tests; do not put planning labels in Rust/Python identifiers, fixtures,
diagnostics, or file names.

## 2. Execution and comparison policy

Run from `proctor/stages/crat`:

```bash
cargo test -p tools observation::tests
cargo test -p tools rule::tests
cargo test -p tools rule_synthesis::tests
cargo test -p tools rule_application::tests
cargo test -p tools skeleton::tests
cargo test -p tools validator::tests
cargo test -p tools item_replacer::tests
cargo test -p tools
cargo test --workspace
cargo fmt
cargo clippy --workspace --all-targets
```

Module names may follow the final surgical layout, but preserve these focused
test boundaries. Run from `proctor`:

```bash
uv run pytest tests/test_local_transformation.py
uv run pytest
uv run ruff check stages/local-transformation tests/test_local_transformation.py
uv run ruff format --check stages/local-transformation tests/test_local_transformation.py
uv run mypy proctor
uv run mypy stages/local-transformation
```

Wire tests compare exact pretty JSON including key order and one terminal
newline. Determinism tests compare complete bytes. Compiler-backed application
tests assert baseline skeleton, applied skeleton, both disposition forests,
remaining statement metadata, and the exact parseable replacement AST.
Orchestration tests assert exact fake-tool event order, projected request JSON,
metrics, LLM calls, accepted files, and published artifacts.

## 3. Common exact fixtures

Use the existing closed normalized expression/type constructors. Extend every
observation/rule constructor with a required keyword-only `lhs: bool`
and insert that key after `pointer_anchors`. The minimal observation is:

```json
{
  "source_expression": {"kind":"unary","operator":"deref","operand":{"kind":"path","value":{"kind":"binding","id":"<id0>"}}},
  "target_expression": {"kind":"unary","operator":"deref","operand":{"kind":"path","value":{"kind":"binding","id":"<id0>"}}},
  "pointer_anchors": [{
    "id":"<id0>",
    "source_type":{"kind":"raw_pointer","mutability":"const","pointee":{"kind":"primitive","name":"i32"}},
    "target_type":{"kind":"reference","mutability":"shared","pointee":{"kind":"primitive","name":"i32"}}
  }],
  "lhs": false,
  "source_type":{"kind":"primitive","name":"i32"},
  "source_adjusted_type":{"kind":"primitive","name":"i32"},
  "target_type":{"kind":"primitive","name":"i32"},
  "target_adjusted_type":{"kind":"primitive","name":"i32"}
}
```

The minimal rule replaces observation-local IDs with canonical variable nodes
and uses `source_pattern`/`target_pattern`. Unless a case overrides context,
use one `A0` anchor from `*const i32` to `&i32`, `lhs: false`, and `i32` for all
four root types. “Applicable” always means this complete context also matches.

For compiler-backed inference cases, declare source raw-pointer types that make
the displayed source compile and inject the listed selected target decisions
through the skeleton test harness. The context-side target declarations below
are decisions, not edits to the source fixture.

## 4. Rust ownership migration and revised documents

### P7-WIRE-01 `lhs_is_required_and_canonically_ordered`

Input: the minimal observation and minimal rule with `lhs: false`, wrapped in
their version-1 documents. Expected: both load; serialization places `lhs`
after `pointer_anchors` and before `source_type`, uses two-space indentation,
and ends with one newline. Change each value to `true`; expected: same success
and exact Boolean preservation.

### P7-WIRE-02 `missing_or_nonboolean_lhs_rejects`

Input: independently remove `lhs`, or replace it with `null`, `0`, `1`, `"false"`,
`[]`, and `{}` in each minimal document. Expected: every observation/rule load
fails and returns no partial value.

### P7-WIRE-03 `earlier_version_one_shape_is_not_accepted`

Input: a previously valid version-1 observation document and rule document
whose entries omit only `lhs`. Expected: both fail as malformed; there is no
default to `false` and no schema-version conversion.

### P7-WIRE-04 `empty_documents_have_exact_bytes`

Input: empty Rust `ObservationDocument` and `RuleDocument`. Expected bytes:

```json
{
  "schema_version": 1,
  "observations": []
}
```

and the analogous document with `"rules": []`, each with one final newline.

### P7-WIRE-05 `phase_six_closed_grammar_moves_without_semantic_drift`

Input: port every current Python closed-model case to Rust, adding
`lhs: false`: every variable sort/position; integer magnitude; local ADT,
field and variant ownership; unknown keys/tags/versions; canonical indices;
target-only variables; concrete local IDs; every expression/type/literal
constructor; foreign identities; and canonical member ordering. Expected: the
same accept/reject result and same canonical patterns as the executable Python
tests, with only the required `lhs` key added. Delete the Python semantic tests
only after this parity test passes.

### P7-SYN-01 `lhs_must_match_before_synthesis`

Input: two otherwise compatible observations whose only difference is
`lhs: false` versus `lhs: true`. Expected: no candidate and the internal
pair-local reason is context mismatch; no identity or disagreement variable is
allocated.

### P7-SYN-02 `synthesized_rule_preserves_lhs`

Input: the two canonical offset observations used to synthesize
`*[A0].offset([N0]) => [A0][[N0]]`, first with both `lhs: false`, then with both
`lhs: true`. Expected: one rule in each run with the matching Boolean. Combining
all four observations yields two rules, not one deduplicated rule.

### P7-SYN-03 `rust_synthesis_retains_all_existing_semantics`

Input: port the currently executable coupled-synthesis corpus: magnitude and
expression disagreements, repeated disagreements, reordered identities, exact
self-pairs, equal source/target, all identity sorts, rigid external/foreign
identities, target lookup failures, degenerate source, anchor hiding, carrier
splits, equality-partition conflict, constructor ownership, every grammar
constructor, duplicate compression, input permutations, and nonmutation.
Add `lhs: false` to every seed. Expected: byte-identical canonical rule cores
apart from `lhs: false`, identical pair-local accept/reject results, exact seed
reconstruction, and no Python synthesis dependency.

### P7-MERGE-01 `merge_zero_one_and_many_documents`

Call the in-memory tools-library merge with input A: no documents. Expected:
canonical empty observation value. Input B: one document `[o0, o0]`. Expected
observations `[o0, o0]`. Input C: three document values `[o0]`, `[]`,
`[o1, o0]` in that order. Expected `[o0, o1, o0]`; duplicates and `lhs` values
are unchanged. Serialize the returned values through the canonical Rust
serializer; do not invoke the binary or touch files.

## 5. Shared selection and assignment-side context

### P7-SELECT-01 `complete_plain_assignment_lhs_is_true`

Input:

```rust
#[proctor(0)] p = q;
```

where `p` is an eligible raw-pointer local and its selected region is the bare
path `p`. Expected: one region rooted at the complete left child with
`lhs: true`; observation extraction serializes `true` and application sees the
same root/anchor/role.

### P7-SELECT-02 `parenthesized_complete_lhs_is_true`

Input `#[proctor(0)] (p) = q;`. Expected: parentheses are transparent, region
root resolves to `p`, and `lhs: true`.

### P7-SELECT-03 `nested_lhs_region_is_false`

Input `#[proctor(0)] slots[*index] = value;` where `slots` is a non-pointer
slice place and `index: *const usize` is eligible. The `index` anchor promotes
through dereference and stops at the index-operand boundary. Expected region
`*index`, which is strictly inside the complete LHS `slots[*index]`, with
`lhs: false`.

### P7-SELECT-04 `rhs_and_compound_assignment_are_false`

Inputs `#[proctor(0)] q = p;`, `#[proctor(0)] p += 1;`, and
`#[proctor(0)] q += *p;`. Expected: every selected region has `lhs: false`; no
compound-assignment operand is assignment-LHS context.

### P7-SELECT-05 `field_promotion_can_form_complete_lhs`

Input `#[proctor(0)] (*holder).ptr = q;` where selection promotes through the
immediate resolved `.ptr` field and stops. Expected: the promoted field root is
the complete LHS and has `lhs: true`; its resolved owner/member is retained.

### P7-SELECT-06 `eligible_catalog_filters_anchors`

Input: one statement uses parameter `p`, paired simple local `q`, unpaired raw
local `r`, raw static `S`, and local `u` whose target decision cannot be
materialized. Expected anchors: occurrences of `p` and `q` only, in source
preorder. `r`, `S`, and `u` do not seed regions and do not disturb valid
disjoint anchors.

### P7-SELECT-07 `nested_labels_are_opaque_and_then_independent`

Input: label 0 is an `if *p != 0` containing label 1 `*q = 1;`. Expected visit
order `[0, 1]`; label 0 selects only the condition region and treats label 1 as
opaque; label 1 independently selects its own region. Equal-root merging and
strict overlap apply within one label, never across the two visits.

### P7-SELECT-08 `observation_and_application_selector_are_identical`

Input: the existing field-promotion, raw-method, call-argument, two-disjoint-
binary, overlap, unsupported-edge, and rejected-anchor source fixtures.
Expected: for the same eligible catalog, both consumers return identical root
AST identities, anchor order, promotion flags, overlap result, and `lhs`.

## 6. Source-pattern applicability and carriers

Each case uses a rule whose fixed context matches the region unless the stated
reason rejects it.

### P7-MATCH-01 `expression_variable_matches_one_local`

Pattern `*[A0].offset([E0] as isize)`; region `*p.offset(i as isize)`.
Expected applicable substitution: `A0 = p`, `E0 = i`.

### P7-MATCH-02 `expression_variable_matches_compound_expression`

Same pattern; region `*p.offset((i + 1) as isize)`. Expected applicable:
`A0 = p`, `E0 = i + 1`.

### P7-MATCH-03 `repeated_expression_variable_agrees`

Pattern `*[A0].offset(([E0] + [E0]) as isize)`; region
`*p.offset((i + i) as isize)`. Expected applicable with `E0 = i`.

### P7-MATCH-04 `repeated_expression_variable_mismatch_rejects`

Same pattern; region `*p.offset((i + j) as isize)`. Expected inapplicable
because the two `E0` occurrences differ.

### P7-MATCH-05 `expression_variable_may_not_contain_anchor`

Pattern `*[A0].offset([E0])`; region `*p.offset(p as isize)`. Expected
inapplicable because the `E0` subtree contains anchor `p`.

### P7-MATCH-06 `anchor_and_binding_are_jointly_injective`

Pattern `*[A0].offset([B0] as isize)`; region `*p.offset(p as isize)`.
Expected inapplicable because distinct `A0` and `B0` bind the same local.

### P7-MATCH-07 `explicit_and_expression_carriers_may_not_split`

Pattern `*[A0].offset(if [B0] { [E0] } else { 0 })`; region
`*p.offset(if flag { flag as isize } else { 0 })`. Expected inapplicable:
`flag` has carriers `B0` and `E0`.

### P7-MATCH-08 `two_expression_carriers_may_not_split_local`

Pattern `*[A0].offset([E0] + [E1])`; region
`*p.offset((i as isize) + (i as isize))`. Expected inapplicable because local
`i` occurs in two distinct carriers.

### P7-MATCH-09 `nonidentity_expressions_need_not_be_injective`

Same pattern; region `*p.offset(1 + 1)`. Expected applicable with
`E0 = 1`, `E1 = 1` because no local identity is split.

### P7-MATCH-10 `integer_magnitude_variable_matches_leaf`

Pattern `*[A0].offset([N0])`; region `*p.offset(4)`. Expected applicable with
`A0 = p`, `N0 = 4`; the fixed literal type must also match.

### P7-MATCH-11 `fixed_integer_magnitude_is_rigid`

Pattern `*[A0].offset(1)`; region `*p.offset(2)`. Expected inapplicable.

### P7-MATCH-12 `fixed_method_identity_is_rigid`

Pattern `*[A0].offset([E0])`; region `*p.add(i)`. Expected inapplicable because
resolved `offset` and `add` differ.

### P7-MATCH-13 `repeated_anchor_may_repeat`

Pattern `*[A0] == *[A0]`; region `*p == *p`. Expected applicable with one
ordered anchor `A0 = p`.

### P7-MATCH-14 `repeated_anchor_must_agree`

Same pattern; region `*p == *q`. Expected inapplicable because one variable
would require two locals (and the region anchor lists do not correspond).

### P7-MATCH-15 `distinct_binding_variables_are_injective`

Pattern `*[A0].offset(([B0] - [B1]) as isize)`; region
`*p.offset((i - i) as isize)`. Expected inapplicable because `B0` and `B1`
would bind the same local.

### P7-MATCH-16 `one_expression_carrier_may_contain_local`

Pattern `*[A0].offset([E0])`; region `*p.offset((i + 1) as isize)`.
Expected applicable: non-anchor local `i` has only carrier `E0`.

### P7-MATCH-17 `local_identity_variable_cannot_match_external_identity`

Pattern `[F0]([A0])`; region `libc::free(p)` where the callee resolves to an
external or foreign function. Expected inapplicable. A fixed matching external
identity pattern is applicable; a local source-defined function is applicable
to `F0`.

### P7-MATCH-18 `lhs_and_all_type_context_are_exact`

Input: one otherwise matching rule; independently change anchor count/order,
anchor source type, anchor target type, source intrinsic type, source adjusted
type, required target type, or `lhs`. Expected: each mismatch makes the rule
inapplicable before target materialization.

## 7. Typed source-pattern specificity

For every case call the directional subsumption predicate both ways and assert
the named relation after standardizing variables apart.

### P7-SPEC-01

Left `*[A0].offset(1)`, right `*[A0].offset([N0])`. Expected: left strictly
more specific.

### P7-SPEC-02

Left `*[A0].offset([B0] as isize)`, right
`*[A0].offset([E0])`. Expected: left strictly more specific.

### P7-SPEC-03

Left `*[A0].offset([E0] + [E0])`, right
`*[A0].offset([E0] + [E1])`. Expected: left strictly more specific.

### P7-SPEC-04

Left `*[A0].offset([E0] + 1)`, right `*[A0].offset([E0])`.
Expected: left strictly more specific.

### P7-SPEC-05

Left `[A0].[M0]([B0] as isize)`, right `[A0].[M0]([E0])`.
Expected: left strictly more specific.

### P7-SPEC-06

Left `[F0]([A0], [E0] + 1)`, right `[F0]([A0], [E0])`.
Expected: left strictly more specific.

### P7-SPEC-07

Both `*[A0].offset([E0])`. Expected: equal specificity and one alpha class.

### P7-SPEC-08

Left `*[A0].offset([E0] - [E1])`, right
`*[A0].offset([E1] - [E0])`. Expected: equal after alpha-renaming.

### P7-SPEC-09

`*[A0].offset(1)` versus `*[A0].offset(2)`. Expected incomparable.

### P7-SPEC-10

`*[A0].offset([E0] + 1)` versus `*[A0].offset(1 + [E0])`.
Expected incomparable.

### P7-SPEC-11

`*[A0].offset([E0])` versus `*[A0].add([E0])`. Expected incomparable because
fixed resolved methods differ.

### P7-SPEC-12

`*[A0].offset([E0] + [E0])` versus `*[A0].offset([E0] + 1)`.
Expected incomparable.

### P7-SPEC-13

`(*[A0]).field` versus `*[A0][[E0]]`. Expected incomparable.

### P7-SPEC-14

`[A0].is_null()` versus `[A0].[M0]()`. Expected incomparable because local-only
`M0` cannot instantiate the fixed external `is_null` method identity.

### P7-SPEC-15

`libc::free([A0])` versus `[F0]([A0])`. Expected incomparable because local
function variable `F0` cannot instantiate the rigid external/foreign callee.

### P7-SPEC-16 `context_does_not_change_specificity_groups`

Input: alpha-equal source patterns with different `lhs`, anchor target type,
and root target type. Expected: one source alpha-equivalence class at load
time, but context filtering may leave only one applicable rule for a region.

## 8. Contextual target-type inference

Each case selects region `□`, makes its source root pointer-like, and asserts
the exact normalized adjusted target requirement. Where “unavailable” is
expected, no pointer-root rule is applicable even if its target type happens
to resemble a source type.

### P7-TYPE-00 `source_type_and_const_generics_reject_before_application`

Call tools-library skeleton generation directly on separate sources containing
`unsafe fn f<T>(p: *mut T) { ... }` and
`unsafe fn g<const N: usize>(p: *mut [i32; N]) { ... }`, with no rules and with
an empty rule value. Expected: the same deterministic unsupported-generic
generation error naming `f`/type and `g`/const before pointer decisions, rule
matching, or any record is returned. A function with lifetime parameters only
retains its existing result. This is a library test, not a command test.

### P7-TYPE-01 `simple_local_initializer`

Source context `let q: *mut i32 = □;`; selected `q: Option<&mut i32>`.
Expected `Option<&mut i32>`.

### P7-TYPE-02 `bare_local_assignment_rhs`

`q = □;`; selected `q: &mut [i32]`. Expected `&mut [i32]`.

### P7-TYPE-03 `dereferenced_reference_place`

`*out = □;`; selected `out: &mut Option<&i32>`. Expected `Option<&i32>`.

### P7-TYPE-04 `dereferenced_field_place`

`(*holder).ptr = □;`; selected `holder: &mut Holder`, resolved
`Holder::ptr: *mut i32`. Expected `*mut i32`.

### P7-TYPE-05 `resolved_direct_call_argument`

`consume(□);`; resolved unchanged/selected signature
`fn consume(_: &mut [i32])`. Expected `&mut [i32]`.

### P7-TYPE-06 `foreign_direct_call_argument`

`ffi_consume(□);`; resolved `unsafe extern "C" fn ffi_consume(_: *mut i32)`.
Expected `*mut i32`.

### P7-TYPE-07 `explicit_return`

`return □;`; selected current return `Option<&i32>`. Expected
`Option<&i32>`.

### P7-TYPE-08 `function_body_tail`

Current function body `{ update(); □ }`; selected return `Box<[i32]>`.
Expected `Box<[i32]>`.

### P7-TYPE-09 `resolved_struct_field_initializer`

`Holder { ptr: □ }`; resolved `Holder::ptr: *mut i32`. Expected `*mut i32`.

### P7-TYPE-10 `resolved_external_generic_call_argument`

`consume::<i32>(□);`; external resolved signature `fn consume<T>(_: &mut [T])`.
Expected `&mut [i32]` after rustc substitution. Negative subcase: make
`consume` source-defined generic; expect the early whole-generation rejection
from P7-TYPE-00, not an inference attempt.

### P7-TYPE-11 `indirect_call_is_unavailable`

`fp(□);` with `fp: fn(*mut i32)`. Expected unavailable.

### P7-TYPE-12 `direct_call_place_then_dereference`

`*get_slot() = □;`; resolved direct
`fn get_slot() -> &'static mut Option<&'static i32>`. Expected
`Option<&'static i32>`.

### P7-TYPE-13 `method_argument_is_unavailable`

`sink.consume(□);` with `sink: Sink`. Expected unavailable.

### P7-TYPE-14 `discarded_value_is_unavailable`

`□;` as a semicolon expression. Expected unavailable.

### P7-TYPE-15 `branch_join_is_unavailable`

`let r = if flag { □ } else { q };` with selected
`q: Option<&i32>`. Expected unavailable; do not infer from the sibling branch.

### P7-TYPE-16 `indexed_slice_place`

`slots[i] = □;`; selected `slots: &mut [Option<&i32>]`. Expected
`Option<&i32>`.

### P7-TYPE-17 `parenthesized_local_place`

`(q) = □;`; selected `q: Box<i32>`. Expected `Box<i32>`.

### P7-TYPE-18 `boxed_place_dereference`

`*slot = □;`; selected `slot: Box<Option<&i32>>`. Expected `Option<&i32>`.

### P7-TYPE-19 `optional_reference_place_dereference`

`*out = □;`; selected `out: Option<&mut Option<&i32>>`. Expected
`Option<&i32>` after type-domain `Option` projection and dereference.

### P7-TYPE-20 `assignment_lhs_inference_is_narrow`

Inputs where `□` itself is the complete LHS: bare raw local `q`, `*q`,
`q.field`, `q[i]`, and a strict region inside any of them. Selected target for
`q` is `&mut [i32]`. Expected: only bare `q` with `lhs: true` yields
`&mut [i32]`; every projection/dereference/nested case is unavailable from the
LHS-specific rule. The place evaluator may still use those complete places to
infer a separate RHS.

### P7-TYPE-21 `pointer_like_is_resolved_and_closed`

Inputs: raw pointer, reference, `Box<i32>`, `Box<[i32]>`,
`Option<&i32>`, `Option<Box<[i32]>>`, bare slice, array, tuple,
`Option<*mut i32>`, `Option<Option<&i32>>`, `Vec<i32>`, and user ADTs named
`Box`/`Option`. Expected pointer-like only for the first six.

### P7-TYPE-22 `nonpointer_roots_ignore_contextual_inference`

Input: source intrinsic/adjusted root both `i32`, rule target roots both `i32`,
inside an unavailable/discarded context. Expected rule may apply because the
unchanged source pair is the requirement. Change either rule target root to
`usize`; expected inapplicable.

### P7-TYPE-23 `assignment_statement_application_is_atomic`

Input `*out = rhs;` where the RHS obtains an applicable pointer rule through
the place evaluator but the selected pointer-like LHS region has no narrow LHS
requirement/rule. Expected: discard the successful RHS tentative rewrite and
leave the complete statement baseline `transform`.

## 9. Target materialization and application results

For each case provide matching anchor/source/root context and a target context
that admits the displayed result. Parse the output and compare its structural
AST; do not map/re-resolve introduced nodes, type-check them, or compare pretty
text alone. Candidate Cargo-build orchestration tests own resolution/type
failures.

### P7-APPLY-01

Source `*[A0].offset([E0] as isize)`, target `[A0][[E0] as usize]`, region
`*p.offset(i as isize)`. Expected `p[i as usize]`.

### P7-APPLY-02

Source `*[A0].add([E0])`, target `[A0][[E0]]`, region `*p.add(i)`.
Expected `p[i]`.

### P7-APPLY-03

Source `[A0].is_null()`, target `[A0].is_none()`, region `p.is_null()`.
Expected `p.is_none()`.

### P7-APPLY-04

Source `![A0].is_null()`, target `[A0].is_some()`, region `!p.is_null()`.
Expected `p.is_some()`.

### P7-APPLY-05

Source `&mut *[A0]`, target `[A0]`, region `&mut *p`. Expected the path AST
`p`, cloned from the matched original local syntax.

### P7-APPLY-06

Source `&*[A0]`, target `[A0]`, region `&*p`. Expected `p`.

### P7-APPLY-07

Source `*[A0]`, target `*[A0].as_deref().unwrap()`, region `*p`.
Expected the parseable AST for `*p.as_deref().unwrap()`; do not resolve the
introduced methods in this test.

### P7-APPLY-08

Source `*[A0]`, target `*[A0].as_deref_mut().unwrap()`, region `*p`.
Expected `*p.as_deref_mut().unwrap()`.

### P7-APPLY-09

Source `(*[A0]).value`, target `[A0].value`, region `(*p).value`.
Expected the parseable field AST `p.value`, using retained member spelling
without re-resolving the new field node.

### P7-APPLY-10

Source `[F0]([A0])`, target `[F0]([A0].as_deref().unwrap())`, region
`read(p)` where `read` is source-defined. Expected
`read(p.as_deref().unwrap())`; the callee syntax is cloned from the matched
source and is not re-resolved.

### P7-APPLY-11

Source `[A0].offset([E0] as isize)`, target
`&[A0][[E0] as usize]`, region `p.offset(i as isize)`. Expected
`&p[i as usize]`.

### P7-APPLY-12

Source `*[A0].offset([E0] as isize)`, target
`[A0].as_deref().unwrap()[[E0] as usize]`, region
`*p.offset(i as isize)`. Expected `p.as_deref().unwrap()[i as usize]`.

### P7-APPLY-13

Source `*[A0].offset(([E0] + [N0]) as isize)`, target
`[A0][[E0] + [N0]]`, region `*p.offset((i + 1) as isize)`.
Expected `p[i + 1]`; `N0` is materialized as the matched magnitude.

### P7-APPLY-14 `lhs_target_must_be_a_supported_place`

Input: `lhs: true` rules materializing respectively `p`, `*p`, `p.field`,
`p[i]`, `f()`, `p + 1`, `p as *mut i32`, and `if c { p } else { q }`.
Expected first four remain structurally admissible candidates; last four are
removed as unsupported assignment places. No identity or type check is run on
the constructed target nodes.

### P7-APPLY-15 `unmaterializable_preferred_rule_does_not_mask_fallback_rule`

Input: two applicable rules in the winning specificity/cost class. Canonical
tie-break selects a target with an unspellable fixed identity; the other target
is `p[i]`. Expected: remove only the first, rerun ranking, and materialize
`p[i]`. If all candidates fail, the region is uncovered and the statement is
unchanged.

### P7-APPLY-16 `all_regions_install_simultaneously`

Input: one statement with two disjoint eligible roots and rules whose rendered
lengths differ from the sources. Expected: both replacements use original AST
identities/spans and install correctly; reversing rule-document order gives
identical applied skeleton bytes. Make either region fail; expected neither
replacement is retained.

### P7-APPLY-17 `ordinary_misses_do_not_abort_generation`

Input one valid rule at a time that fails source matching, anchor context,
target inference, carrier validation, spelling, parsing, place validation, or
supported-skeleton admission. Target identity resolution is deliberately not
attempted. Expected: every listed construction/shape failure is an ordinary
miss; the statement remains
baseline and other functions/statements continue. Independently inject a
corrupt loaded-rule invariant or global source AST/HIR mapping inconsistency.
Expected: fatal whole-generation failure with no record value.

## 10. Ranking and normalized size

### P7-RANK-01 `strictly_more_specific_group_wins`

Input region `*p.offset(1)` with applicable source groups
`*[A0].offset(1)`, `*[A0].offset([N0])`, and `*[A0].offset([E0])`, each with a
different target. Expected: fixed-`1` group alone survives specificity.

### P7-RANK-02 `substitution_cost_counts_distinct_source_variables_once`

Input equal-maximal groups where rule A repeats `E0` twice and binds it to
`i + 1`, while rule B uses `E0`,`E1` bound to `i` and `1`. Compute normalized
sizes using constructors/scalar leaves and count repeated `E0` once. Expected:
the smaller summed distinct-variable cost wins; context-only variables add
zero.

### P7-RANK-03 `larger_target_then_canonical_json_break_ties`

Input applicable equal-specificity/equal-cost rules with target node counts
3 and 5. Expected count 5 wins. Equalize counts; expected lexicographically
smallest compact canonical target JSON, then full-rule JSON. Permute document
order; expected identical winner.

### P7-RANK-04 `term_size_counts_terms_not_json_containers`

Input exact normalized terms for a path identity, unary path, binary of two
paths, call with two arguments, integer literal, and variable. Expected size is
one per grammar constructor/variable/typed scalar leaf, recursively; JSON keys
and argument arrays contribute zero. Assert the same function drives
substitution cost and target-size ranking.

## 11. Dual views and recursive dispositions

### P7-VIEW-01 `record_has_two_explicit_equal_views_without_rules`

Input: generate skeletons without `--rules` and with an empty rule document.
Expected byte-equivalent records: each function has required `baseline` and
`applied` objects; the objects are equal values but independently serialized;
old flat view keys are absent.

### P7-VIEW-WIRE-01 `projected_requests_have_exact_closed_shape`

Let the selected view value be exactly:

```json
{
  "skeleton": "unsafe fn read(mut p: &[i32]) -> i32 { #[proctor(0)] p[0] }",
  "needs_transformation": false,
  "statement_dispositions": [
    {"label": 0, "disposition": "rule_applied", "children": []}
  ],
  "statement_pair_metadata": []
}
```

For member 7, transformation `returned`, and no accepted correspondence, the
validation request is exactly:

```json
{
  "schema_version": 1,
  "expected_functions": [
    {
      "id": 7,
      "name": "read",
      "view": {
        "skeleton": "unsafe fn read(mut p: &[i32]) -> i32 { #[proctor(0)] p[0] }",
        "needs_transformation": false,
        "statement_dispositions": [
          {"label": 0, "disposition": "rule_applied", "children": []}
        ],
        "statement_pair_metadata": []
      }
    }
  ],
  "transformation": "returned"
}
```

The replacement request is exactly:

```json
{
  "schema_version": 1,
  "items": [
    {
      "id": 7,
      "path": "module::read",
      "name": "read",
      "view": {
        "skeleton": "unsafe fn read(mut p: &[i32]) -> i32 { #[proctor(0)] p[0] }",
        "needs_transformation": false,
        "statement_dispositions": [
          {"label": 0, "disposition": "rule_applied", "children": []}
        ],
        "statement_pair_metadata": []
      }
    }
  ],
  "transformation": "returned",
  "accepted_correspondence": []
}
```

Expected: no old flat `skeleton`,
`needs_transformation`, or transformation-label fields occur beside `view`;
unknown/missing request or view keys reject. Repeat with the baseline view and
assert only the complete `view` value changes.

### P7-VIEW-02 `one_rule_complete_statement`

Input function:

```rust
unsafe fn read(mut p: *const i32, i: usize) -> i32 {
    #[proctor(0)] *p.add(i)
}
```

with selected `p: &[i32]` and rule `*[A0].add([E0]) => [A0][[E0]]`.
Expected baseline label 0 `transform` with a hole; applied label 0
`rule_applied` containing `p[i]`, `needs_transformation: false`, and empty
applied statement metadata. Baseline metadata still contains label 0.

### P7-VIEW-03 `rule_applied_outer_with_transform_child`

Input an `if` whose raw-pointer condition has a rule but whose labeled body
statement has no rule. Expected applied forest root 0 `rule_applied` with child
1 `transform`; applied skeleton canonically fixes the condition/control shell
and leaves the child hole; `needs_transformation: true`; metadata `[1]`.

### P7-VIEW-04 `transform_outer_with_rule_applied_child`

Input an outer control payload with no applicable rule and a nested labeled
pointer statement with full rule coverage. Expected root 0 `transform`, child
1 `rule_applied`; outer own payload stays editable and child is canonical;
metadata `[0]` only.

### P7-VIEW-05 `partial_region_coverage_discards_statement`

Input one label with two disjoint regions and a rule for only one. Expected
applied statement equals baseline, disposition remains `transform`, and
metadata remains present. A nested label may still apply independently.

### P7-VIEW-06 `invalid_single_view_invariants_reject_every_boundary`

Mutate a valid view with missing/extra/duplicate/out-of-order labels, wrong
parent, non-`u32` label, unknown disposition/key, baseline `rule_applied`,
preserve parent with open descendant, wrong `needs_transformation`, skeleton
versus forest topology mismatch, and metadata not equal to DFS transform
labels. Expected Rust setup, Python loading, validator, and replacer each
reject before producing a candidate.

### P7-VIEW-07 `cross_view_invariants_reject_independently`

Start from one valid baseline/applied record. Independently mutate only the
applied view to change function signature, declaration/binding topology, label
set/order/parent, or supported control topology; change a baseline `preserve`
node to applied `transform`; or mark a baseline-preserved node
`rule_applied`. Expected the Python record loader rejects each cross-view
inconsistency even though both individual views are internally valid. Positive
input: a baseline `transform` parent/child may become any of
`transform/transform`, `rule_applied/transform`, or
`transform/rule_applied` when each complete applied skeleton matches its
forest.

### P7-VAL-01 `rule_applied_shell_is_restored_around_returned_child`

Input expected applied root 0 `rule_applied`/child 1 `transform`. Returned code
changes the fixed condition and supplies a valid child expansion. Expected
validator result `valid`; canonical result restores the expected condition and
shell but retains the child expansion. Replacer independently emits the same
canonical result without relying on prior validator output.

### P7-VAL-02 `transform_outer_retains_payload_and_restores_child`

Input root 0 `transform`/child 1 `rule_applied`. Returned code changes the
outer condition and corrupts child 1. Expected valid if the outer change obeys
structure; canonical result retains the returned condition and restores child
1 exactly.

### P7-VAL-03 `descendant_slot_must_remain_locatable`

For both mixed topologies, independently remove, duplicate, reorder, move, or
make a child label nonconsecutive/wrong-level. Expected deterministic
`invalid` from validator and atomic replacement failure; a fixed parent does
not make descendant alignment optional.

### P7-VAL-04 `preserve_transform_and_rule_expansion_groups_differ`

Input returned multi-statement groups for labels of each disposition.
Expected: transform may retain a valid expansion; preserve and rule-applied
discard returned own statements and restore one canonical expected statement,
then splice only valid labeled descendants. Generated temporaries in discarded
groups do not escape or produce false errors.

### P7-VAL-05 `restored_rule_ast_defers_resolution_to_build`

Input a mixed applied view whose `rule_applied` expression is parseable and
structurally supported but names an unavailable value or has an incompatible
type; another label remains `transform` and receives valid returned code.
Expected tools structural validation succeeds and restores the exact rule AST
without HIR mapping, resolution, or type checking. In the fake stage flow,
Cargo build fails and the projection's `rule_applied` node triggers the normal
whole-SCC baseline fallback and shared repair budget.

### P7-REP-01 `pairs_and_observation_metadata_include_only_transform`

Input selected forest root 0 `rule_applied`, child 1 `transform`, sibling 2
`preserve`. Expected replacement sidecar and
`current_items.transform_labels` contain exactly `[1]`; statement pairs and
extracted observations may arise only from label 1.

## 12. PROCTOR input, view projection, fallback, and publication

### P7-PY-01 `manifest_and_make_skeleton_optional_rule_path`

Input manifest inspection and two calls to the stage's fake tools interface.
Without `inputs.rule_set`, expected `[requires].rule_set == "optional"` and the
skeleton-generation method receives `None`. With a normalized opaque rule-path
token, expected the method receives that token unchanged. Do not inspect or
construct `crat-tool` argv in this test.

### P7-PY-02 `rule_path_overlap_map_covers_every_mutable_boundary`

Exercise the extracted pure boundary-relation logic with normalized path
tokens, not real nodes. Make the rule path equal to, an ancestor of, and a
descendant of each of: Rust input, workdir, Rust output, artifacts directory,
statement report, observations artifact, configured Crat checkout, and
framework usage log. Expected every relation rejects before the fake tool-build
event. A disjoint token succeeds. Regular-file/symlink and real path-node
behavior are not tested here.

### P7-PY-03 `rule_path_is_redacted_from_stage_outputs`

Input one fake run with rule-path token `/sensitive/rules.json`. The framework
envelope fixture may serialize that value in `stage_input.json`. Expected every
stage-owned command-log event uses `<rule-set>` instead; stage output, metrics,
metadata, errors copied to its log, and exchange artifacts do not contain the
token or a digest. The fake skeleton-generation call still receives the opaque
token. This tests rendered stage data only, not a command line or filesystem.

### P7-PY-04 `every_protocol_uses_one_consistent_projection`

Input one applied mixed SCC, then a baseline fallback. Expected applied prompt,
validation request, replacement request, sidecar expected labels, metadata
expected labels, and accepted-pair metadata all use applied view together.
After switch, all use baseline together. Inject one mixed view in a helper;
expected focused unit assertion/error rather than malformed tool invocation.

### P7-PY-05 `rule_complete_scc_is_mechanical`

Input applied views with `needs_transformation: false` and at least one
`rule_applied`; fake replacement/build succeeds. Expected no prompt, validator,
or completion call; one replacement/build; no pairs/observation extraction for
rule-only labels; existing function/SCC/build metrics only.

### P7-PY-06 `mixed_applied_candidate_succeeds`

Input applied view with one rule-applied and one transform label. LLM returns a
valid completion and build succeeds. Expected generation 0 uses applied,
extracts observations only for the transform label, retains one opaque
per-SCC document, and never reports the rule-applied label.

### P7-FALLBACK-01 `llm_applied_build_failure_switches_whole_scc_once`

Input two-member SCC, applied view has a rule only in member A and open labels
in B. Generation 0 validates/replaces but Cargo returns stdout `out0`, stderr
`err0`. Expected rollback, both members project baseline, repair call 1 prompt
contains the exact generation-0 transformation and diagnostics, and all later
requests remain baseline. Metrics after baseline success:

```python
{
    "function_count": 2,
    "scc_count": 1,
    "llm_generation_calls": 2,
    "repair_calls": 1,
    "structural_failures": 0,
    "compilation_failures": 1,
    "cargo_builds": 3,
}
```

### P7-FALLBACK-02 `mechanical_applied_failure_starts_at_repair_one`

Input fully rule-complete applied SCC; mechanical replacement succeeds but
build fails, then first baseline completion succeeds. Expected failed context
is concatenated applied skeletons in member-ID order, the first logical exchange
event is tagged `generation-01`, no `generation-00` LLM exchange event exists,
and final metrics
are exactly:

```python
{
    "function_count": 1,
    "scc_count": 1,
    "llm_generation_calls": 1,
    "repair_calls": 1,
    "structural_failures": 0,
    "compilation_failures": 1,
    "cargo_builds": 3,
}
```

### P7-FALLBACK-03 `existing_repairs_before_fallback_share_budget`

Input generation 0 missing fence, applied repair 1 validator-invalid, applied
repair 2 reaches Cargo and fails with a rule-applied projection. Expected next
baseline call is repair 3, not 1; at most calls through repair 10. Assert exact
prompt/request view sequence. Let baseline repair 3 succeed; expected metrics:

```python
{
    "function_count": 1,
    "scc_count": 1,
    "llm_generation_calls": 4,
    "repair_calls": 3,
    "structural_failures": 2,
    "compilation_failures": 1,
    "cargo_builds": 3,
}
```

### P7-FALLBACK-04 `ten_total_repairs_are_the_limit`

Input failures through repair 10 spanning both views. Expected exactly eleven
LLM calls only when there was generation 0, or ten calls after a mechanical
failure; then SCC exhaustion. No reset occurs at fallback and no failed attempt
contributes pairs, correspondence, or observation files. Make every candidate
in this fixture reach Cargo and fail. The generation-0 variant's final metrics
are exactly:

```python
{
    "function_count": 1,
    "scc_count": 1,
    "llm_generation_calls": 11,
    "repair_calls": 10,
    "structural_failures": 0,
    "compilation_failures": 11,
    "cargo_builds": 12,
}
```

The mechanical-entry variant has the same dictionary except
`llm_generation_calls` is `10`; its mechanical applied build plus ten baseline
builds still gives `compilation_failures: 11` and `cargo_builds: 12`.

### P7-FALLBACK-05 `only_rule_involved_cargo_failure_switches`

Independently inject validator setup error, malformed validator response,
replacement failure, provider failure, builder exception, rollback failure,
and observation failure while applied. Expected current fatal behavior and no
switch. A mechanical applied view equal to baseline with no `rule_applied`
whose build fails is fatal. An LLM candidate with no `rule_applied` whose build
fails uses ordinary same-view repair.

### P7-PUB-01 `accepted_observation_files_are_opaque_and_ordered`

Input accepted SCC documents containing `[o0,o0]`, then `[]`, then `[o1]`;
include different `lhs` values. Use opaque accepted-document tokens and a fake
merge callback. Expected Python never parses them, passes tokens to merge in
SCC acceptance order, and the callback observes `[o0,o0,o1]`. Superseded and
failed-attempt tokens are absent from the accepted state and merge call.

### P7-PUB-02 `accepted_document_commit_is_one_state_transition`

Input one successful build with fake extracted-document token `d0`. Make the
stage's injected retain/move callback first succeed, then raise. Expected on
success: the accepted-document list appends `d0` together with statement pairs
and callable correspondence. On failure: none of those three state components
changes and the stage fails; do not exercise a real move, path, or file.

### P7-PUB-03 `zero_extractions_still_merge_an_empty_value`

Input all-mechanical preserved/rule-complete stage state. Expected no extraction
and a fake merge call with an empty accepted-document tuple; its returned
canonical empty observation value is handed to the existing publisher. Do not
test command argv or publication paths here.

### P7-PUB-04 `opaque_merge_logic_failure_is_fatal`

Input a fake merge callback that raises before returning a merged-document
token. Expected stage failure, no publisher call, and no attempt to parse or
reconstruct JSON in Python. Do not inject command exit, node, rename, cleanup,
or other filesystem failures.

### P7-PUB-05 `no_new_provenance_or_metrics`

Input successful rule-complete and fallback runs. Expected stage output,
Markdown, logs, metrics, metadata, and artifacts contain no rule ID, rule path,
digest, provenance, selection diagnostic, coverage count, rule metric, or
application report. Existing metrics retain their meanings and only the
ordinary statement report plus `observations.json` are published.

## 13. Completion criteria

The work is complete only when all cases above pass; every preexisting tools
and local-transformation regression is updated to the two-view/revised-document
shape; rule order cannot affect output; statement application is atomic;
validator and replacer independently restore recursive dispositions; the
fallback budget is demonstrated at both boundaries; Python treats accepted
observation documents as opaque; and format, lint, type, Clippy, focused, and
workspace checks succeed.
