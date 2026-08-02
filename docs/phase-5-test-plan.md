# Phase 5 Test Plan: Validated Expression Observation Collection

## 1. Purpose and test discipline

This is the exhaustive regression plan for
[Phase 5](phase-5-plan.md). It covers deterministic source relabeling, ordinary-candidate
invariance, observation-source construction, correspondence, the unexpanded-AST
runner, regions and alignment, every closed expression/type wire variant,
strict Python protocols, post-build accumulation, and transactional
publication.

Every named case below contains its exact Rust input. Implementation tests may
factor harness code, but must keep each Rust fixture local to its case. Do not
move the Rust snippets to a file-wide fixture or combine independent cases to
save compiler invocations. Inputs are intentionally minimal; a larger input is
used only for a genuinely multi-function or nested-control interaction.

Every case has a local `Expected result` oracle. For extraction cases it names
the selected label/unit, each eligible anchor occurrence in source order, the
selected source root or Reject reason, merge/overlap handling, and the emitted
observation or skip. It also gives a mechanical, label-by-label alignment list:
every final deduplicated source region and its target expression are quoted as
exact Rust fragments, and every label without a mapping says `no alignment`
with the terse reason.
Unless a case explicitly exercises adjustments, an emitted expression's
adjusted type equals its ordinary type. Each emitted anchor row records the
paired binding types stated by the source and target signatures. Protocol and
orchestration cases instead state the exact generated fields, paths, bytes,
ordering, error, or cleanup state; anchors and regions do not apply to them.

The planned suite contains 74 named cases:

| Area | Cases |
| --- | ---: |
| Replacement, observation source, and regressions | 10 |
| Specialized runner and correspondence | 8 |
| Statement selection, regions, and alignment | 26 |
| Closed trees, types, and anonymization | 17 |
| Python protocols, acceptance, and publication | 13 |

All assertions reflect the settled Phase 5 contracts.

## 2. Execution and comparison policy

Rust tests belong beside `skeleton`, `item_replacer`, and the new `observation`
module, with thin CLI tests in `src/bin/crat-tool.rs`. Never nest rustc compiler
callbacks; return owned source, metadata, or wire values first. Run from
`proctor/stages/crat`:

```bash
cargo test -p tools skeleton::tests
cargo test -p tools item_replacer::tests
cargo test -p tools observation::tests
cargo test -p tools
cargo test --bin crat-tool
cargo test --workspace
cargo fmt
cargo clippy --workspace --all-targets
```

Python tests remain offline in `proctor/tests/test_local_transformation.py` and
use fake Crat/Cargo/LLM boundaries. Run from `proctor`:

```bash
uv run pytest tests/test_local_transformation.py
uv run ruff check stages/local-transformation tests/test_local_transformation.py
uv run ruff format --check stages/local-transformation tests/test_local_transformation.py
```

Candidate and statement-pair regressions compare exact existing bytes.
Observation-source tests parse AST unless formatting is the contract. Every
observation source compiles after PROCTOR labels are removed. Extraction tests
assert exact roots, skip/fatal class, anchor order, all six type locations, and
closed typed wire values. Representative documents compare exact pretty JSON;
variant tests compare exact serde values and reject unknown keys. Repeat runs
must be byte-identical.

## 3. Replacement, observation source, and regressions

### P5-REP-01 `ordinary_candidate_and_statement_pairs_remain_exact`

Current source:

```rust
pub unsafe fn read(mut pointer: *const i32) -> i32 {
    *pointer
}
```

Accepted target:

```rust
pub unsafe fn read<'a>(mut pointer: &'a i32) -> i32 {
    #[proctor(0)]
    *pointer
}
```

Exact request:

```json
{
  "schema_version": 1,
  "items": [{"id": 7, "path": "read", "name": "read", "skeleton": "pub unsafe fn read<'a>(mut pointer: &'a i32) -> i32 { #[proctor(0)] *pointer }", "needs_transformation": true, "statements_requiring_transformation": [0]}],
  "transformation": "pub unsafe fn read<'a>(mut pointer: &'a i32) -> i32 { #[proctor(0)] *pointer }",
  "accepted_correspondence": []
}
```

Expected result:

- Output: the old and extended calculations produce byte-identical candidate
  and schema-version-1 statement-pair files. The extended calculation also
  produces `replacement-observation.rs` and the exact metadata collections
  shown locally below.
- Invariants: all four output paths are distinct; each digest equals SHA-256 of
  the exact companion bytes. This replacement-only case performs no region
  extraction.

Expected metadata fields other than the three content digests:

```json
{
  "schema_version": 1,
  "accepted_correspondence": [],
  "new_correspondence": [{"item_id": 7, "logical_path": "read", "implementation_path": "read", "wrapper_path": "__proctor_wrapper_read"}],
  "current_items": [{"item_id": 7, "logical_path": "read", "source_copy_path": "__proctor_source_read", "implementation_path": "read", "wrapper_path": "__proctor_wrapper_read", "transform_labels": [0]}]
}
```

The complete object inserts `candidate_sha256`, `statement_pairs_sha256`, and
`observation_source_sha256` immediately after `schema_version`; each exact value
is the lowercase SHA-256 of the corresponding exact output bytes from this
fixture.

### P5-REP-02 `source_copy_names_avoid_module_collisions_without_moving_wrappers`

Current source:

```rust
mod a {
    fn __proctor_source_read() {}
    fn __proctor_source_read_0() {}
    fn __proctor_wrapper_read() {}

    pub unsafe fn read(mut pointer: *const i32) -> i32 { *pointer }
}

mod b {
    pub unsafe fn read(mut pointer: *const i32) -> i32 { *pointer }
}
```

Accepted targets:

```rust
unsafe fn read(mut pointer: &i32) -> i32 {
    #[proctor(0)]
    *pointer
}
```

Expected result for the two-module input:

- Generated paths: `a::read` gets source copy
  `a::__proctor_source_read_1` and wrapper
  `a::__proctor_wrapper_read_0`; `b::read` gets
  `b::__proctor_source_read` and `b::__proctor_wrapper_read`.
- Ordering/invariance: allocation is module-local and deterministic. Wrapper
  names are frozen before source-copy allocation, so adding observation output
  changes neither candidate wrapper names nor candidate bytes.

Exact raw-identifier subcase:

```rust
pub unsafe fn r#type(mut pointer: *const i32) -> i32 { *pointer }
```

Accepted target:

```rust
unsafe fn r#type(mut pointer: &i32) -> i32 { #[proctor(0)] *pointer }
```

Expected result for the raw-identifier subcase:

- Generated path: the copy is `__proctor_source_type` (plus only the ordinary
  numeric collision suffix if occupied), never a name containing `r#`.

### P5-REP-03 `observation_functions_strip_outer_attributes_but_keep_statement_labels`

Current source:

```rust
#[inline(never)]
#[no_mangle]
pub unsafe extern "C" fn read(mut pointer: *const i32) -> i32 {
    *pointer
}
```

Accepted target:

```rust
pub unsafe fn read(mut pointer: &i32) -> i32 {
    #[proctor(0)]
    *pointer
}
```

Expected result:

- Candidate: current export relocation semantics and bytes remain unchanged.
- Observation source: source copy, implementation, and wrapper have no outer
  attributes. The private, ordinary-Rust-ABI source copy contains `*pointer`;
  both source copy and implementation retain `#[proctor(0)]` on that statement.
- Compilation: removing PROCTOR labels yields a compiling crate. No pass
  inspects which stripped outer attributes were present.

### P5-REP-04 `deterministic_relabeler_labels_real_source_after_earlier_call_redirect`

Current source after an earlier accepted callee rewrite:

```rust
unsafe fn callee(mut pointer: &i32) -> i32 { *pointer }
unsafe fn __proctor_wrapper_callee(mut pointer: *const i32) -> i32 {
    callee(&*pointer)
}
pub unsafe fn caller(mut pointer: *const i32) -> i32 {
    __proctor_wrapper_callee(pointer)
}
```

Accepted target:

```rust
pub unsafe fn caller(mut pointer: &i32) -> i32 {
    #[proctor(0)]
    callee(pointer)
}
```

Exact request correspondence/order fields:

```json
{
  "items": [{"id": 7, "path": "caller", "name": "caller", "needs_transformation": true, "statements_requiring_transformation": [0]}],
  "accepted_correspondence": [{"item_id": 3, "logical_path": "callee", "implementation_path": "callee", "wrapper_path": "__proctor_wrapper_callee"}]
}
```

Expected result:

- Source copy: its only body statement is exactly the current
  `__proctor_wrapper_callee(pointer)` call and receives `#[proctor(0)]` from the
  depth-first relabeler.
- Target: label 0 remains on `callee(pointer)`. No stale pre-redirection source
  spelling appears in the copy.

### P5-REP-05 `source_clone_relabeling_needs_no_expected_label_sidecar`

Current source:

```rust
pub unsafe fn read(mut pointer: *const i32) -> i32 {
    let value = 1;
    *pointer + value
}
```

Accepted target:

```rust
pub unsafe fn read(mut pointer: &i32) -> i32 {
    #[proctor(0)]
    let value = 1;
    #[proctor(1)]
    *pointer + value
}
```

Expected result:

- Source/target labels: the real source copy is relabeled with 0 on
  `let value = 1` and 1 on `*pointer + value`; target labels stay 0 and 1.
- Metadata: `current_items[0].transform_labels` is exactly `[1]`; there is no
  locator or expected-label sidecar field.
- Selection: extraction pairs the two materialized label-1 statements directly;
  label 0 is not selected.
- Anchors/regions: label 1 has one `pointer` anchor selecting `*pointer`; no
  merge or overlap occurs.
- Exact alignment:
  - label 0: no alignment (not selected)
  - label 1: `*pointer` -> `*pointer`
- Observation: label 1 emits one record with anchor `<id0>` (`*const i32` ->
  `&i32`) and four `i32` expression types.

### P5-REP-06 `source_copy_recursion_uses_absolute_copy_path`

Current source:

```rust
mod nested {
    pub unsafe fn sum(mut pointer: *const i32, mut count: usize) -> i32 {
        if count == 0 { 0 } else { *pointer + sum(pointer.add(1), count - 1) }
    }
}
```

Accepted target:

```rust
unsafe fn sum(mut pointer: &[i32], mut count: usize) -> i32 {
    #[proctor(0)]
    if count == 0 {
        #[proctor(1)]
        0
    } else {
        #[proctor(2)]
        pointer[0] + sum(&pointer[1..], count - 1)
    }
}
```

Expected result:

- Generated source: the allocated copy path is exactly
  `nested::__proctor_source_sum`, and only its recursive call becomes
  `crate::nested::__proctor_source_sum(pointer.add(1), count - 1)`. The target
  retains the supplied spelling `sum(&pointer[1..], count - 1)`; it resolves to
  implementation path `nested::sum` and is not textually absolutized.
- Metadata/compilation: current correspondence relates those callables through
  logical path `nested::sum`; the label-free observation source compiles.

### P5-REP-07 `mutual_scc_copies_call_each_other_in_item_order`

Current source:

```rust
pub unsafe fn even(mut pointer: *const i32, mut n: usize) -> i32 {
    if n == 0 { *pointer } else { odd(pointer, n - 1) }
}
pub unsafe fn odd(mut pointer: *const i32, mut n: usize) -> i32 {
    if n == 0 { *pointer } else { even(pointer, n - 1) }
}
```

Accepted targets:

```rust
unsafe fn even(mut pointer: &i32, mut n: usize) -> i32 {
    #[proctor(0)]
    if n == 0 {
        #[proctor(1)] *pointer
    } else {
        #[proctor(2)] odd(pointer, n - 1)
    }
}
unsafe fn odd(mut pointer: &i32, mut n: usize) -> i32 {
    #[proctor(0)]
    if n == 0 {
        #[proctor(1)] *pointer
    } else {
        #[proctor(2)] even(pointer, n - 1)
    }
}
```

Exact request identity/order fields (the complete skeleton/transformation
strings are the accepted functions immediately above):

```json
{
  "schema_version": 1,
  "items": [
    {"id": 7, "path": "odd", "name": "odd", "needs_transformation": true, "statements_requiring_transformation": [0, 1, 2]},
    {"id": 3, "path": "even", "name": "even", "needs_transformation": true, "statements_requiring_transformation": [0, 1, 2]}
  ],
  "accepted_correspondence": []
}
```

Expected result:

- Ordering: source-copy allocation, wrapper allocation, `current_items`, and
  `new_correspondence` all preserve the deliberately reversed request order.
  Observation records later use item-ID order independently.
- Generated calls: each source copy calls the other absolute source-copy path;
  each implementation calls the other implementation path. Both pairs are
  related by their current correspondence records.

Expected ordered metadata collections:

```json
{
  "accepted_correspondence": [],
  "new_correspondence": [
    {"item_id": 7, "logical_path": "odd", "implementation_path": "odd", "wrapper_path": "__proctor_wrapper_odd"},
    {"item_id": 3, "logical_path": "even", "implementation_path": "even", "wrapper_path": "__proctor_wrapper_even"}
  ],
  "current_items": [
    {"item_id": 7, "logical_path": "odd", "source_copy_path": "__proctor_source_odd", "implementation_path": "odd", "wrapper_path": "__proctor_wrapper_odd", "transform_labels": [0, 1, 2]},
    {"item_id": 3, "logical_path": "even", "source_copy_path": "__proctor_source_even", "implementation_path": "even", "wrapper_path": "__proctor_wrapper_even", "transform_labels": [0, 1, 2]}
  ]
}
```

The source-copy calls are exactly `crate::__proctor_source_even(pointer,
n - 1)` from the odd copy and `crate::__proctor_source_odd(pointer, n - 1)`
from the even copy. Target calls retain `even(pointer, n - 1)` and
`odd(pointer, n - 1)` as supplied.

### P5-REP-08 `accepted_wrapper_correspondence_is_echoed_and_used`

Current source:

```rust
unsafe fn callee(mut pointer: &i32) -> i32 { *pointer }
unsafe fn __proctor_wrapper_callee(mut pointer: *const i32) -> i32 {
    callee(&*pointer)
}
pub unsafe fn caller(mut pointer: *const i32) -> i32 {
    __proctor_wrapper_callee(pointer)
}
```

Accepted target:

```rust
pub unsafe fn caller(mut pointer: &i32) -> i32 {
    #[proctor(0)]
    callee(pointer)
}
```

Expected result:

- Metadata: `accepted_correspondence` echoes the supplied `callee` record
  value-for-value and order-for-order; `new_correspondence` contains only the
  `caller` record.
- Exact alignment:
  - label 0: `pointer` -> `pointer`
- The source pointer anchor selects `pointer` as the direct-call argument. The
  non-region callee identities compare as one logical `<fn0>` only through correspondence.
  The observation is emitted with anchor `<id0>` (`*const i32` -> `&i32`) and
  expression types/adjusted types `*const i32` -> `&i32`.
- Errors/skips: malformed, contradictory, dangling, reordered, or altered
  retained records are fatal. With the relationship omitted, the well-formed
  callee mismatch skips label 0 and emits zero observations; names/bodies are
  never reverse-engineered.
- Exact alignment with the relationship omitted:
  - label 0: no alignment (non-region callee mismatch)

Expected metadata collections:

```json
{
  "accepted_correspondence": [{"item_id": 3, "logical_path": "callee", "implementation_path": "callee", "wrapper_path": "__proctor_wrapper_callee"}],
  "new_correspondence": [{"item_id": 7, "logical_path": "caller", "implementation_path": "caller", "wrapper_path": "__proctor_wrapper_caller"}],
  "current_items": [{"item_id": 7, "logical_path": "caller", "source_copy_path": "__proctor_source_caller", "implementation_path": "caller", "wrapper_path": "__proctor_wrapper_caller", "transform_labels": [0]}]
}
```

### P5-REP-09 `two_argument_main_boundary_has_no_logical_copy`

Current source:

```rust
pub unsafe fn main_0(mut argc: i32, mut argv: *mut *mut i8) -> i32 {
    argc + (**argv != 0) as i32
}
pub fn main() {
    unsafe { std::process::exit(main_0(0, std::ptr::null_mut())) }
}
```

Accepted target:

```rust
pub unsafe fn main_0(mut argc: i32, mut argv: &mut [&mut [i8]]) -> i32 {
    #[proctor(0)]
    argc + (argv[0][0] != 0) as i32
}
```

Expected result:

- Generated source/metadata: candidate and observation source contain the same
  generated safe `main` boundary and absolute `main_0` call. Only `main_0`
  appears in `current_items`/`new_correspondence` and has a source copy; `main`
  appears in neither collection.
- Compilation: the observation source compiles after label removal.

### P5-REP-10 `macro_hidden_copy_call_rewrite_is_fatal`

Current source:

```rust
macro_rules! recurse { ($p:expr) => { read($p) }; }
pub unsafe fn read(mut pointer: *const i32) -> i32 {
    if pointer.is_null() { 0 } else { recurse!(pointer) }
}
```

Accepted target:

```rust
pub unsafe fn read(mut pointer: Option<&i32>) -> i32 {
    #[proctor(0)]
    if pointer.is_none() {
        #[proctor(1)] 0
    } else {
        #[proctor(2)] read(pointer)
    }
}
```

Expected result:

- Failure: replacement detects that the resolved recursive call requiring a
  source-copy redirect is hidden in macro tokens and returns fatal
  `UnsupportedCallRewrite`.
- Files/state: none of the four outputs is usable (any owned temporary or
  earlier final is removed), and Python does not classify the error as an LLM
  repair opportunity.

## 4. Specialized runner and correspondence

### P5-RUN-01 `labels_are_recorded_removed_and_recovered_on_unexpanded_ast`

Observation source:

```rust
unsafe fn source_copy(mut pointer: *const i32) -> i32 {
    #[proctor(0)]
    if pointer.is_null() {
        #[proctor(1)]
        0
    } else {
        #[proctor(2)]
        *pointer
    }
}
unsafe fn target(mut pointer: Option<&i32>) -> i32 {
    #[proctor(0)]
    if pointer.is_none() {
        #[proctor(1)]
        0
    } else {
        #[proctor(2)]
        *pointer.unwrap()
    }
}
```

Expected result:

- Recovery: labels 0, 1, and 2 are each found once under both metadata-declared
  functions and retain their unexpanded statement node/span identities.
- Compiler input/output: all six attributes are removed before expansion;
  every recorded macro-free node maps exactly once to HIR/type results, and no
  PROCTOR attribute reaches ordinary attribute checking.

### P5-RUN-02 `macro_anywhere_in_selected_statement_skips_before_expansion`

Observation source:

```rust
macro_rules! one { () => { 1_i32 }; }
unsafe fn source_copy(mut pointer: *const i32) -> i32 {
    #[proctor(0)]
    if true {
        #[proctor(1)]
        *pointer
    } else {
        #[proctor(2)] one!()
    }
}
unsafe fn target(mut pointer: &i32) -> i32 {
    #[proctor(0)]
    if true {
        #[proctor(1)]
        *pointer
    } else {
        #[proctor(2)] one!()
    }
}
```

Expected result:

- Selection: labels 0, 1, and 2 are independent units. Nested labels 1 and 2
  are opaque to label 0, whose condition/shell contains no macro and no eligible
  anchor. Label 2 skips because its own statement contains `one!()`.
- Exact alignment:
  - label 0: no alignment (no eligible anchor in the outer condition/shell)
  - label 1: `*pointer` -> `*pointer`
  - label 2: no alignment (macro call in the selected statement)
- Label 1 emits one observation with anchor `<id0>` (`*const i32` ->
  `&i32`) and four expression types all `i32`. No expansion-origin recovery is
  attempted.

### P5-CORR-01 `parameters_pair_by_index_and_symbol`

Observation source:

```rust
unsafe fn source_copy(mut left: *const i32, mut right: *const i32) -> i32 {
    #[proctor(0)] *left + *right
}
unsafe fn target(mut left: &i32, mut right: &i32) -> i32 {
    #[proctor(0)] *left + *right
}
```

Expected result:

- Correspondence: source/target parameters pair as index 0 `left` and index 1
  `right`, independent of compiler-local IDs. Swapping target symbols is fatal.
- Anchors/regions: label 0 has anchors `left`, then `right`; they select
  disjoint roots `*left` and `*right`. No merge or overlap occurs.
- Exact alignment:
  - label 0:
    - `*left` -> `*left`
    - `*right` -> `*right`
- Observations: two are emitted in source order. Each has one anchor (`<id0>`
  within its observation), binding types `*const i32` -> `&i32`, and source/
  source-adjusted/target/target-adjusted expression types all `i32`.

### P5-CORR-02 `simple_locals_pair_by_declaration_label_and_symbol`

Observation source:

```rust
unsafe fn source_copy(mut pointer: *mut i32) -> i32 {
    #[proctor(0)] let mut alias: *mut i32 = pointer;
    #[proctor(1)] *alias
}
unsafe fn target(mut pointer: &mut i32) -> i32 {
    #[proctor(0)] let mut alias: &mut i32 = pointer;
    #[proctor(1)] *alias
}
```

Expected result:

- Correspondence: declaration label 0 and symbol pair `alias`; source annotation
  normalizes to `*mut i32`, target annotation to `&mut i32`, and the differing
  annotations are accepted per-side compiler facts. Pairing uses the source and
  target declaration labels, symbols, resolved identities, and resolved types;
  observation metadata carries no skeleton sidecar.
- Anchors/regions: label 0 has one `pointer` anchor selecting `pointer`; label 1
  has one `alias` anchor selecting `*alias`. Neither label has a merge or
  overlap.
- Exact alignment:
  - label 0: `pointer` -> `pointer`
  - label 1: `*alias` -> `*alias`
- Observations: label 0 emits one record whose anchor `<id0>` has binding types
  `*mut i32` -> `&mut i32`; its source/source-adjusted expression types are
  `*mut i32`, and its target/target-adjusted expression types are `&mut i32`.
  Label 1 emits one record whose anchor `<id0>` has the same binding types and
  whose four expression types are `i32`.
- Fatal mutations: changed declaration label/symbol, one-sided annotation, or
  either annotation disagreeing with its own compiler-resolved binding type.

### P5-CORR-03 `shadowed_locals_remain_distinct`

Observation source:

```rust
unsafe fn source_copy(mut pointer: *mut i32) -> i32 {
    #[proctor(0)] let mut alias: *mut i32 = pointer;
    #[proctor(1)]
    {
        #[proctor(2)] let mut alias: *mut i32 = pointer;
        #[proctor(3)] *alias += 1;
    }
    #[proctor(4)] *alias
}
unsafe fn target(mut pointer: &mut i32) -> i32 {
    #[proctor(0)] let mut alias: &mut i32 = pointer;
    #[proctor(1)]
    {
        #[proctor(2)] let mut alias: &mut i32 = alias;
        #[proctor(3)] *alias += 1;
    }
    #[proctor(4)] *alias
}
```

Expected result:

- Correspondence: declaration label 0 pairs the outer `alias`; label 2 pairs
  the shadowing inner `alias`. They remain distinct resolved bindings.
- Anchors/regions: label 0's `pointer` anchor selects `pointer`; label 2's
  `pointer` anchor also selects `pointer`; label 3's inner `alias` anchor
  selects `*alias`; and label 4's outer `alias` anchor selects `*alias`.
  Each label has one region, so no merge or overlap occurs. Label 1 is a
  preserved outer block and is not a candidate.
- Exact alignment:
  - label 0: `pointer` -> `pointer`
  - label 1: no alignment (preserved label)
  - label 2: `pointer` -> `alias` (the target path resolves to the outer
    label-0 local)
  - label 3: `*alias` -> `*alias`
  - label 4: `*alias` -> `*alias`
- Observations: labels 0 and 2 each emit one record whose `pointer` anchor has
  binding types `*mut i32` -> `&mut i32`; its source/source-adjusted expression
  types are `*mut i32`, and its target/target-adjusted expression types are
  `&mut i32`. Labels 3 and 4 each emit one record whose `alias` anchor has the
  same binding-type pair and whose four expression types are `i32`. In every
  observation the sole anchor is `<id0>`; the inner and outer `alias` bindings
  remain distinct compiler identities even though observation-local numbering
  resets for each record.

### P5-CORR-04 `source_only_reference_is_allowed`

Observation source:

```rust
unsafe fn source_copy(
    mut pointer: *const i32,
    mut extra: usize,
    mut index: usize,
) -> i32 {
    #[proctor(0)] *pointer.add(extra + index)
}
unsafe fn target(mut pointer: &[i32], mut extra: usize, mut index: usize) -> i32 {
    #[proctor(0)] pointer[index]
}
```

Expected result:

- Selection: the statement contains exactly one eligible `pointer` occurrence,
  inside `*pointer.add(extra + index)`. It grows through `add` and dereference,
  selecting that complete source expression; there is no merge/overlap.
- Exact alignment:
  - label 0: `*pointer.add(extra + index)` -> `pointer[index]`
- The non-region source-only `extra` is permitted inside the wildcard source
  tree.
- Observation: binding IDs from source preorder are `pointer` `<id0>`, `extra`
  `<id1>`, and `index` `<id2>`; target reuses `<id0>`/`<id2>`. Anchor `<id0>` has
  `*const i32` -> `&[i32]`; all four expression types are `i32`.

### P5-CORR-05 `target_only_user_function_discards_observation`

Observation source:

```rust
unsafe fn helper(mut value: i32) -> i32 { value }
unsafe fn source_copy(mut pointer: *const i32) -> i32 {
    #[proctor(0)] *pointer
}
unsafe fn target(mut pointer: &i32) -> i32 {
    #[proctor(0)] helper(*pointer)
}
```

Expected result:

- Anchors/regions: source anchor `pointer` selects `*pointer`; no merge or
  overlap occurs.
- Exact alignment:
  - label 0: `*pointer` -> `helper(*pointer)`
- Observation: target dumping encounters source-defined `helper` with no source
  correspondent, discards this observation, emits zero records, and never
  allocates a target-only `<fnN>`.

### P5-CORR-06 `inferred_raw_local_pairs_with_materialized_target_type`

Observation source:

```rust
unsafe fn source_copy(mut pointer: *mut i32) -> i32 {
    #[proctor(0)] let mut alias = pointer;
    #[proctor(1)] *alias
}
unsafe fn target(mut pointer: &mut i32) -> i32 {
    #[proctor(0)] let mut alias: &mut i32 = pointer;
    #[proctor(1)] *alias
}
```

Expected result:

- Correspondence: label 0/symbol pair `alias`; compiler source binding type is
  `*mut i32`, target annotation and selected type are `&mut i32`. Source
  `type:null` versus target `reference(mutable,i32)` is an accepted per-side
  compiler fact recovered without skeleton metadata.
- Anchors/regions: label 0 has one `pointer` anchor selecting `pointer`; label 1
  has one inferred raw-local `alias` anchor selecting `*alias`. Neither label
  has a merge or overlap.
- Exact alignment:
  - label 0: `pointer` -> `pointer`
  - label 1: `*alias` -> `*alias`
- Observations: label 0 emits one record whose anchor `<id0>` records
  `*mut i32` -> `&mut i32`; its source/source-adjusted expression types are
  `*mut i32`, and its target/target-adjusted expression types are `&mut i32`.
  Label 1 emits one record with the same anchor binding types and four `i32`
  expression type fields.
- Fatal mutations: absent/wrong target annotation or any unexpected source
  annotation/type combination.

## 5. Statement selection, regions, and alignment

### P5-STMT-01 `preserved_labels_never_emit`

```rust
unsafe fn source_copy(mut pointer: *const i32) -> i32 {
    #[proctor(0)] let value = 1;
    #[proctor(1)] value + *pointer
}
unsafe fn target(mut pointer: &i32) -> i32 {
    #[proctor(0)] let value = 1;
    #[proctor(1)] value + *pointer
}
```

Expected result:

- Selection: only label 1 is an alignment unit; preserved label 0 is never
  visited for anchors regardless of its Rust text.
- Anchors/regions: label 1 has one `pointer` occurrence selecting `*pointer`;
  no merge/overlap.
- Exact alignment:
  - label 0: no alignment (preserved label)
  - label 1: `*pointer` -> `*pointer`
- Observation: emit one
  observation with anchor `<id0>` (`*const i32` -> `&i32`) and all four
  expression types `i32`.

### P5-STMT-02 `multi_statement_target_group_skips`

```rust
unsafe fn source_copy(mut pointer: *const i32) -> i32 {
    #[proctor(0)] *pointer
}
unsafe fn target(mut pointer: &i32) -> i32 {
    #[proctor(0)] let proctor_temp_var_0 = *pointer;
    #[proctor(0)] proctor_temp_var_0
}
```

Expected result:

- Selection: label 0 is recovered as one valid consecutive two-statement target
  group, then skipped before anchor collection because the target unit is not a
  single statement.
- Exact alignment:
  - label 0: no alignment (multi-statement target group)
- Output: successful extraction with zero observations; neither target
  statement is aligned independently.

### P5-STMT-03 `outer_control_keeps_conditions_and_opaque_nested_labels`

```rust
unsafe fn source_copy(mut pointer: *const i32) -> i32 {
    #[proctor(0)]
    if pointer.is_null() {
        #[proctor(1)] 0
    } else {
        #[proctor(2)] *pointer
    }
}
unsafe fn target(mut pointer: Option<&i32>) -> i32 {
    #[proctor(0)]
    if pointer.is_none() {
        #[proctor(1)] 0
    } else {
        #[proctor(2)] *pointer.unwrap()
    }
}
```

Expected result:

- Anchors/regions: nested labels 1/2 are opaque to label 0. Its sole source anchor is
  condition occurrence `pointer`; allowlisted `is_null` selects
  `pointer.is_null()`.
- Exact alignment:
  - label 0: `pointer.is_null()` -> `pointer.is_none()`
  - label 1: no alignment (preserved label with no eligible anchor)
  - label 2: `*pointer` -> `*pointer.unwrap()`
- Observations: label 0 emits one
  bool observation with anchor `<id0>` (`*const i32` -> `Option<&i32>`); all
  four expression types are `bool`.
- Nested units: preserved label 1 emits nothing. Label 2 independently emits an `i32` observation
  with the same binding-type pair. No outer/nested roots are compared for
  overlap because they belong to different alignment units.

### P5-STMT-04 `match_arms_use_canonical_labeled_blocks`

```rust
unsafe fn source_copy(mut pointer: *const i32) -> i32 {
    #[proctor(0)]
    match *pointer {
        0 => { #[proctor(1)] *pointer.add(1) }
        _ => { #[proctor(2)] 2 }
    }
}
unsafe fn target(mut pointer: &[i32]) -> i32 {
    #[proctor(0)]
    match pointer[0] {
        0 => { #[proctor(1)] pointer[1] }
        _ => { #[proctor(2)] 2 }
    }
}
```

Expected result:

- Labeling/topology: arms remain blocks containing statement labels 1 and 2;
  label 0 sees both arm blocks as opaque leaves.
- Anchors/regions: label 0's only mineable occurrence is scrutinee `pointer`,
  which selects `*pointer`. Label 1 independently selects `*pointer.add(1)`.
  Label 2 has no anchor.
- Exact alignment:
  - label 0: `*pointer` -> `pointer[0]`
  - label 1: `*pointer.add(1)` -> `pointer[1]`
  - label 2: no alignment (no eligible anchor)
- Observations: label 0 emits one `i32` observation with anchor `<id0>` (`*const i32` ->
  `&[i32]`) and four expression types `i32`.
- Label 1 emits one observation with the same anchor binding types and four `i32`
  expression types. Label 2 has no anchor and emits nothing.

### P5-STMT-05 `outer_control_keeps_disjoint_condition_regions`

```rust
unsafe fn source_copy(mut pointer: *const i32, mut other: *const i32) -> i32 {
    #[proctor(0)]
    if pointer.is_null() || other.is_null() {
        #[proctor(1)] 0
    } else {
        #[proctor(2)] *pointer
    }
}
unsafe fn target(mut pointer: Option<&i32>, mut other: Option<&i32>) -> i32 {
    #[proctor(0)]
    if pointer.is_none() || other.is_none() {
        #[proctor(1)] 0
    } else {
        #[proctor(2)] *pointer.unwrap()
    }
}
```

Expected result:

- Anchors/regions: label 0 has source-order anchors `pointer` and `other`.
  They select the disjoint roots `pointer.is_null()` and `other.is_null()`;
  neither root encloses opaque labels 1/2, and no merge or overlap occurs.
  Label 2 independently selects `*pointer`.
- Exact alignment:
  - label 0:
    - `pointer.is_null()` -> `pointer.is_none()`
    - `other.is_null()` -> `other.is_none()`
  - label 1: no alignment (preserved label with no eligible anchor)
  - label 2: `*pointer` -> `*pointer.unwrap()`
- Output: label 0 emits two observations in source order. Their sole anchors
  are respectively `pointer` and `other`, numbered `<id0>` within each record;
  both have binding types `*const i32` -> `Option<&i32>` and four `bool`
  expression types. Label 2 emits one observation with the `pointer` anchor,
  the same binding-type pair, and four `i32` expression types. Label 1 emits
  nothing.

### P5-EDGE-01 `builtin_binary_produces_two_disjoint_regions`

```rust
unsafe fn source_copy(mut left: *const i32, mut right: *const i32) -> i32 {
    #[proctor(0)] *left + *right
}
unsafe fn target(mut left: &i32, mut right: &i32) -> i32 {
    #[proctor(0)] *left + *right
}
```

Expected result:

- Selection: label 0 anchors are `left`, then `right`. Dereference grows each to
  roots `*left` and `*right`; builtin `+` finishes both. Roots are disjoint, so
  there is no merge or overlap skip.
- Exact alignment:
  - label 0:
    - `*left` -> `*left`
    - `*right` -> `*right`
- Observations: emit two in left-to-right order. Each has one `<id0>` anchor,
  binding type `*const i32` -> `&i32`, and four expression type fields `i32`.

### P5-EDGE-02 `raw_receiver_allowlist_grows_through_deref`

```rust
unsafe fn source_copy(mut pointer: *const i32, mut index: usize) -> i32 {
    #[proctor(0)] *pointer.add(index)
}
unsafe fn target(mut pointer: &[i32], mut index: usize) -> i32 {
    #[proctor(0)] pointer[index]
}
```

Expected result for the labeled `add` input:

- Selection: the only anchor grows through allowlisted receiver `add`, then
  through dereference, selecting `*pointer.add(index)`; no merge/overlap.
- Exact alignment:
  - label 0: `*pointer.add(index)` -> `pointer[index]`
- Observation: emit one record with
  anchor `<id0>` (`*const i32` -> `&[i32]`) and all four expression types `i32`.

The exact classifier input for those variants is:

```rust
unsafe fn methods(mut p: *const i32, mut q: *const i32, mut n: usize) {
    let _ = p.offset(n as isize);
    let _ = p.sub(n);
    let _ = p.wrapping_offset(n as isize);
    let _ = p.wrapping_add(n);
    let _ = p.wrapping_sub(n);
    let _ = p.offset_from(q);
    let _ = p.is_null();
    let _ = p.read();
}
```

Expected result for the classifier matrix:

- Receivers of `offset`, `add`, `sub`, `wrapping_offset`, `wrapping_add`,
  `wrapping_sub`, `offset_from`, and `is_null` return Grow. Receiver `read`
  returns Reject. Arguments obey their independent direct-method argument
  roles; this lower-level matrix does not itself emit observations.

### P5-EDGE-03 `array_tuple_repeat_values_finish_nonpointer_elements`

```rust
unsafe fn source_copy(mut pointer: *const i32) -> ([i32; 1], (i32,), [i32; 2]) {
    #[proctor(0)] ([*pointer], (*pointer,), [*pointer; 2])
}
unsafe fn target(mut pointer: &i32) -> ([i32; 1], (i32,), [i32; 2]) {
    #[proctor(0)] ([*pointer], (*pointer,), [*pointer; 2])
}
```

Expected result:

- Selection: three `pointer` occurrences select `*pointer` in array element,
  tuple element, and repeat value order. Each finishes before its non-pointer
  aggregate parent; roots are disjoint and neither merge nor overlap.
- Exact alignment:
  - label 0:
    - `*pointer` -> `*pointer` (array element)
    - `*pointer` -> `*pointer` (tuple element)
    - `*pointer` -> `*pointer` (repeat value)
- Observations: emit three ordered records. Each anchor is `<id0>` with
  `*const i32` -> `&i32`, and every expression/adjusted type is `i32`.

### P5-EDGE-04 `all_resolved_direct_call_arguments_finish`

Observation source:

```rust
unsafe fn local_read(mut pointer: *const i32) -> i32 { *pointer }
unsafe extern "C" { fn foreign_read(pointer: *const i32) -> i32; }
unsafe fn source_copy(mut pointer: *const i32) -> i32 {
    #[proctor(0)]
    local_read(pointer)
        + std::ptr::read(pointer)
        + foreign_read(pointer)
}
unsafe fn target(mut pointer: &i32) -> i32 {
    #[proctor(0)]
    local_read(pointer as *const i32)
        + std::ptr::read(pointer as *const i32)
        + foreign_read(pointer as *const i32)
}
```

No test-only crate or Rust dependency file is added under Crat. The standard
library supplies the external-Rust definition needed by this case.

Expected result:

- Selection: label 0 contains three source-order `pointer` anchors. Each is a
  direct argument and finishes at source root `pointer`; callee paths never
  seed. The three roots are distinct/disjoint, with no merge or overlap.
- Exact alignment:
  - label 0:
    - `pointer` -> `pointer as *const i32` (`local_read` argument)
    - `pointer` -> `pointer as *const i32` (`std::ptr::read` argument)
    - `pointer` -> `pointer as *const i32` (`foreign_read` argument)
- Non-region callees resolve respectively as the same
  source-defined function, canonical standard-library function, and same
  `extern "C"` symbol.
- Observations: emit three ordered records. Each anchor `<id0>` has binding type
  `*const i32` -> `&i32`; source expression/adjusted types are `*const i32`, and
  target expression/adjusted types are `*const i32` because of the cast.

### P5-EDGE-05 `indirect_calls_reject`

```rust
unsafe fn source_copy(
    mut function: unsafe fn(*const i32) -> i32,
    mut pointer: *const i32,
) -> i32 {
    #[proctor(0)] function(pointer)
}
unsafe fn target(
    mut function: unsafe fn(*const i32) -> i32,
    mut pointer: &i32,
) -> i32 {
    #[proctor(0)] function(pointer as *const i32)
}
```

Expected result:

- Selection: label 0's `pointer` occurrence reaches the direct-call argument
  boundary, but resolution says the callee is the function-pointer parameter;
  that anchor Rejects and selects no region.
- Exact alignment:
  - label 0: no alignment (indirect call rejected)
- Output: successful extraction with zero observations.

### P5-EDGE-06 `method_receiver_and_argument_roles_are_closed`

```rust
struct Sink;
impl Sink { fn take(&self, value: u32) -> u32 { value } }
unsafe fn source_copy(mut pointer: *const u32, mut sink: Sink) -> u32 {
    #[proctor(0)] (*pointer).count_ones();
    #[proctor(1)] sink.take(*pointer)
}
unsafe fn target(mut pointer: &u32, mut sink: Sink) -> u32 {
    #[proctor(0)] pointer.count_ones();
    #[proctor(1)] sink.take(*pointer)
}
```

Expected result:

- Anchors/regions: label 0's `pointer` anchor grows to `*pointer`, then finishes
  before the non-pointer `u32` receiver method. Label 1's `pointer` anchor
  grows to `*pointer`, then finishes as the resolved `take` argument.
- Exact alignment:
  - label 0: `*pointer` -> `pointer`
  - label 1: `*pointer` -> `*pointer`
- Observations: label 0 emits one observation with anchor `<id0>`
  (`*const u32` -> `&u32`): source/source-adjusted types are `u32`; target type
  is `&u32` and target adjusted type is `u32` after receiver autoderef.
- Label 1 emits one observation with the same
  anchor binding types and all four expression types `u32`. Roots are
  single/disjoint in both labels.

### P5-EDGE-07 `conditions_and_guards_finish_without_required_type_inference`

```rust
unsafe fn source_copy(mut pointer: *const i32) -> i32 {
    #[proctor(0)]
    match 0 {
        _ if pointer.is_null() => { #[proctor(1)] 0 }
        _ => { #[proctor(2)] 1 }
    }
}
unsafe fn target(mut pointer: Option<&i32>) -> i32 {
    #[proctor(0)]
    match 0 {
        _ if pointer.is_none() => { #[proctor(1)] 0 }
        _ => { #[proctor(2)] 1 }
    }
}
```

Expected result for the match input:

- Selection: label 0 treats arm labels as opaque. Its sole pointer anchor is in
  the guard and selects `pointer.is_null()`. There is no merge/overlap.
- Exact alignment:
  - label 0: `pointer.is_null()` -> `pointer.is_none()`
  - label 1: no alignment (no eligible anchor)
  - label 2: no alignment (no eligible anchor)
- Observation: emit one record with anchor `<id0>` (`*const i32` ->
  `Option<&i32>`) and all four expression types `bool`; no required-type field.

Exact `if`/`while` input is:

```rust
unsafe fn source_copy(mut pointer: *const i32) {
    #[proctor(0)] if pointer.is_null() { #[proctor(1)] return; }
    #[proctor(2)] while pointer.is_null() { #[proctor(3)] return; }
}
unsafe fn target(mut pointer: Option<&i32>) {
    #[proctor(0)] if pointer.is_none() { #[proctor(1)] return; }
    #[proctor(2)] while pointer.is_none() { #[proctor(3)] return; }
}
```

Expected result for the `if`/`while` subcase:

- Exact alignment:
  - label 0: `pointer.is_null()` -> `pointer.is_none()`
  - label 1: no alignment (no eligible anchor)
  - label 2: `pointer.is_null()` -> `pointer.is_none()`
  - label 3: no alignment (no eligible anchor)
- Labels 0 and 2 each emit the same bool
  observation: anchor `<id0>` records `*const i32` -> `Option<&i32>`, and all
  four expression types are `bool`. Nested return labels contain no anchors.

### P5-EDGE-08 `let_assignment_return_and_root_semicolon_finish`

```rust
unsafe fn source_copy(mut pointer: *const i32) -> i32 {
    #[proctor(0)] let mut value: i32 = *pointer;
    #[proctor(1)] value = *pointer.add(1);
    #[proctor(2)] pointer.add(2);
    #[proctor(3)] return *pointer.add(3);
}
unsafe fn target(mut pointer: &[i32]) -> i32 {
    #[proctor(0)] let mut value: i32 = pointer[0];
    #[proctor(1)] value = pointer[1];
    #[proctor(2)] pointer.get(2);
    #[proctor(3)] return pointer[3];
}
```

Expected result:

- Exact alignment:
  - label 0: `*pointer` -> `pointer[0]`
  - label 1: `*pointer.add(1)` -> `pointer[1]`
  - label 2: `pointer.add(2)` -> `pointer.get(2)`
  - label 3: `*pointer.add(3)` -> `pointer[3]`
- Each label has one anchor, no merge/overlap, and emits one observation. Labels
  0/1/3 have four expression types `i32`; label 2 records source expression and
  adjusted type `*const i32`, while target expression/adjusted type is
  `Option<&i32>`. All anchors record binding `*const i32` -> `&[i32]`. The
  semicolon root is not skipped for lack of a context-required type.

### P5-EDGE-09 `field_base_allows_deref_removal_and_anonymizes_field`

```rust
struct Pair { value: i32 }
unsafe fn source_copy(mut pointer: *const Pair) -> i32 {
    #[proctor(0)] (*pointer).value
}
unsafe fn target(mut pointer: &Pair) -> i32 {
    #[proctor(0)] pointer.value
}
```

Expected result:

- Selection: sole anchor grows through dereference to source root `*pointer`;
  field-base policy finishes before `.value`. No merge/overlap.
- Exact alignment:
  - label 0: `*pointer` -> `pointer`
- Observation: emit one
  anchor `<id0>` (`raw_pointer(const,<struct0>)` ->
  `reference(shared,<struct0>)`). Source/source-adjusted types are `<struct0>`;
  target type is `reference(shared,<struct0>)` and target adjusted type is
  `<struct0>` after field-base autoderef. The selected expressions contain no
  field node; a separate dumper assertion obtains the enclosing concrete field
  expression from this Rust and assigns it `<field0>`. Neither `Pair` nor
  `value` appears in JSON.

### P5-EDGE-10 `index_index_and_struct_field_value_finish`

```rust
struct Pair { value: i32 }
unsafe fn source_copy(mut pointer: *const usize, mut values: &[i32]) -> Pair {
    #[proctor(0)] Pair { value: values[*pointer] }
}
unsafe fn target(mut pointer: &usize, mut values: &[i32]) -> Pair {
    #[proctor(0)] Pair { value: values[*pointer] }
}
```

Expected result:

- Selection: the only anchor grows to `*pointer`, then finishes as the builtin
  scalar index operand. It does not grow through `values[...]`, the struct field
  value, or the constructor. No merge/overlap.
- Exact alignment:
  - label 0: `*pointer` -> `*pointer`
- Observation: emit one
  observation with anchor `*const usize` -> `&usize` and all four expression
  types `usize`.

### P5-EDGE-11 `address_and_cast_grow_transparently_through_parentheses`

```rust
unsafe fn source_copy(mut pointer: *const i32) -> usize {
    #[proctor(0)] ((&*pointer) as *const i32) as usize
}
unsafe fn target(mut pointer: &i32) -> usize {
    #[proctor(0)] (pointer as *const i32) as usize
}
```

Expected result:

- Selection: anchor `pointer` grows through dereference, address-of, the inner
  raw-pointer cast, and the outer scalar cast; transparent parentheses add no
  node. Source root is `((&*pointer) as *const i32) as usize`; no merge/overlap.
- Exact alignment:
  - label 0: `((&*pointer) as *const i32) as usize` -> `(pointer as *const i32) as usize`
- Observation: emit
  one anchor (`*const i32` -> `&i32`) with all four expression types `usize`.
  Dumped inner cast targets are normalized `raw_pointer(const,i32)` trees, not
  source spelling.

### P5-EDGE-12 `pointerlike_aggregate_roles_reject`

Exact semantic-type motivating input for the classifier matrix:

```rust
struct Contains(*const i32);
type Raw = *mut i32;
type Ref<'a> = &'a mut i32;
type Slice<'a> = &'a mut [i32];
type OptionalRef<'a> = Option<&'a mut i32>;
type Owned = Box<i32>;
type OptionalOwned = Option<Box<i32>>;
type OwnedSlice = Box<[i32]>;
type OptionalOwnedSlice = Option<Box<[i32]>>;
```

Resolve each alias in the classifier unit. `Contains` supplies the concrete
pointer-containing-ADT negative input.

Exact aggregate input is:

```rust
unsafe fn source_copy(mut pointer: *const i32) -> ([*const i32; 1], (*const i32,)) {
    #[proctor(0)] ([pointer], (pointer,))
}
unsafe fn target<'a>(mut pointer: Option<&'a i32>)
    -> ([Option<&'a i32>; 1], (Option<&'a i32>,))
{
    #[proctor(0)] ([pointer], (pointer,))
}
```

Expected result for the type-classifier and aggregate inputs:

- Classifier: resolved `Raw`, `Ref`, `Slice`, `OptionalRef`, `Owned`,
  `OptionalOwned`, `OwnedSlice`, and `OptionalOwnedSlice` are pointer-like;
  `Contains` is not.
- Aggregate: both source `pointer` occurrences Reject at their array/tuple
  element roles because their expression is pointer-like. Each anchor has the
  binding-type pair `*const i32` -> `Option<&i32>`. No regions survive and the
  label emits zero observations.
- Exact alignment for the aggregate input:
  - label 0: no alignment (both pointer-like aggregate roles rejected)

### P5-EDGE-13 `overloaded_operations_reject`

```rust
use std::ops::Add;
struct Wrap(*const i32);
impl Add<*const i32> for Wrap {
    type Output = i32;
    fn add(self, _other: *const i32) -> i32 { 0 }
}
unsafe fn source_copy(mut pointer: *const i32) -> i32 {
    #[proctor(0)] Wrap(core::ptr::null()) + pointer
}
unsafe fn target(mut pointer: &i32) -> i32 {
    #[proctor(0)] Wrap(core::ptr::null()) + (pointer as *const i32)
}
```

Expected result for the overloaded-binary input:

- Selection: label 0's source `pointer` anchor is directly the right operand of
  the resolved overloaded `Add`. That parent edge Rejects the anchor, so no
  root or observation is produced.
- Exact alignment:
  - label 0: no alignment (overloaded binary edge rejected)

Exact overloaded assign-op input:

```rust
use std::ops::AddAssign;
struct Wrap;
impl AddAssign<*const i32> for Wrap {
    fn add_assign(&mut self, _other: *const i32) {}
}
unsafe fn source_copy(mut pointer: *const i32, mut value: Wrap) {
    #[proctor(0)] value += pointer;
}
unsafe fn target(mut pointer: &i32, mut value: Wrap) {
    #[proctor(0)] value += pointer as *const i32;
}
```

Expected result for the overloaded-assign-op input:

- Selection: label 0's source `pointer` anchor is directly the right operand of
  the resolved overloaded `AddAssign`. That parent edge Rejects the anchor, so
  no root or observation is produced.
- Exact alignment:
  - label 0: no alignment (overloaded assign-op edge rejected)

### P5-EDGE-14 `unsupported_control_and_expression_variants_reject_without_panics`

```rust
unsafe fn source_copy(mut pointer: *const Result<i32, i32>) -> Result<i32, i32> {
    #[proctor(0)] Ok((*pointer)?)
}
unsafe fn target(mut pointer: &Result<i32, i32>) -> Result<i32, i32> {
    #[proctor(0)] Ok((*pointer)?)
}
```

Expected result for the labeled input:

- Anchor `pointer` grows to `*pointer`, then Rejects at the `Try` parent. No
  root survives and extraction succeeds with zero observations.
- Exact alignment:
  - label 0: no alignment (`Try` parent rejected)

The exact reachable parser fixtures are:

```rust
unsafe fn closure(mut pointer: *const i32) { let _ = || *pointer; }
unsafe fn loop_break(mut pointer: *const i32) {
    let _ = loop { break *pointer; };
}
```

Expected result for the variant matrix:

- The closure-boundary and value-bearing-break edges obtained from the displayed
  parsed and type-checked Rust each Reject their `pointer` occurrence without
  panic. These classifier checks have no PROCTOR labels and do not run alignment
  or emit observations.

### P5-EDGE-15 `index_base_and_struct_rest_reject_independently`

Index-base input:

```rust
unsafe fn source_copy(mut pointer: *const [i32], mut index: usize) -> i32 {
    #[proctor(0)] (*pointer)[index]
}
unsafe fn target(mut pointer: &[i32], mut index: usize) -> i32 {
    #[proctor(0)] pointer[index]
}
```

Struct-rest input:

```rust
#[derive(Copy, Clone)] struct Pair { value: i32 }
unsafe fn source_copy(mut pointer: *const Pair) -> Pair {
    #[proctor(0)] Pair { value: 1, ..*pointer }
}
unsafe fn target(mut pointer: &Pair) -> Pair {
    #[proctor(0)] Pair { value: 1, ..*pointer }
}
```

Expected result:

- Index-base input: anchor `pointer` grows to `*pointer`, then Rejects as the
  index base; no observation.
- Struct-rest input: anchor grows to `*pointer`, then Rejects as functional
  update base; no observation. The inputs are run independently so neither
  rejection masks the other.
- Exact alignment for each independent input:
  - index-base input:
    - label 0: no alignment (index-base role rejected)
  - struct-rest input:
    - label 0: no alignment (struct-rest role rejected)

### P5-EDGE-16 `builtin_assignop_and_other_unary_finish`

```rust
unsafe fn source_copy(mut pointer: *mut bool) {
    #[proctor(0)] *pointer &= true;
    #[proctor(1)] let value = !*pointer;
}
unsafe fn target(mut pointer: &mut bool) {
    #[proctor(0)] *pointer &= true;
    #[proctor(1)] let value = !*pointer;
}
```

Expected result:

- Label 0's anchor grows to `*pointer` and finishes before builtin non-pointer
  `&=`. Label 1's anchor grows to `*pointer` and finishes before builtin `!`,
  not at the complete unary node.
- Exact alignment:
  - label 0: `*pointer` -> `*pointer`
  - label 1: `*pointer` -> `*pointer`
- Observations: one per label, no merge/overlap, anchor binding `*mut bool` ->
  `&mut bool`, and all four expression types `bool`.

### P5-EDGE-17 `repeated_binding_occurrences_remain_distinct`

Observation source:

```rust
unsafe fn source_copy(mut pointer: *const i32) -> i32 {
    #[proctor(0)] *pointer + *pointer
}
unsafe fn target(mut pointer: &i32) -> i32 {
    #[proctor(0)] *pointer + *pointer
}
```

Expected result:

- The two `pointer` occurrences select distinct roots `*pointer`, ordered left
  then right; no merge or overlap.
- Exact alignment:
  - label 0:
    - `*pointer` -> `*pointer` (left operand)
    - `*pointer` -> `*pointer` (right operand)
- Emit two byte-identical observations, each with `<id0>`, binding
  `*const i32` -> `&i32`, and four `i32` expression types. Duplicate evidence
  is retained.

### P5-EDGE-18 `strict_ancestry_overlap_skips_complete_statement`

```rust
unsafe fn source_copy(mut base: *const i32, mut other: *const i32) -> i32 {
    #[proctor(0)] *base.offset(other.offset_from(base))
}
unsafe fn target(mut base: &[i32], mut other: &[i32]) -> i32 {
    #[proctor(0)] base[other.as_ptr().offset_from(base.as_ptr()) as usize]
}
```

Expected result:

- Selection: source-order anchors are outer receiver `base`, inner receiver
  `other`, then inner argument `base`. They select, respectively, outer
  `*base.offset(other.offset_from(base))`, nested
  `other.offset_from(base)`, and the innermost `base` argument path. After
  identical-root merging, these roots are related by strict ancestry.
- Output: overlap skips complete label 0; neither outer nor inner observation
  is emitted.
- Exact alignment:
  - label 0: no alignment (strictly overlapping final regions)

### P5-EDGE-19 `nonregion_operator_or_child_role_change_rejects_statement`

```rust
unsafe fn source_copy(mut pointer: *const i32, mut scalar: i32) -> i32 {
    #[proctor(0)] ((*pointer)) + scalar
}
unsafe fn target(mut pointer: &i32, mut scalar: i32) -> i32 {
    #[proctor(0)] scalar - (((*pointer)))
}
```

Expected result:

- Selection: source anchor selects `*pointer`; no merge/overlap.
- Exact alignment for the displayed target:
  - label 0: no alignment (non-region operator/child-role mismatch)
- Although a wildcard could map `*pointer` to target `*pointer`, the
  surrounding source `add(left=wildcard,right=scalar)` does not match target
  `subtract(left=scalar,right=wildcard)`. Skip the complete label and emit zero.

### P5-EDGE-20 `logical_callee_identity_aligns_wrapper_and_implementation`

```rust
unsafe fn index(mut value: i32) -> usize { value as usize }
unsafe fn __proctor_wrapper_index(mut value: i32) -> usize { index(value) }
unsafe fn source_copy(mut pointer: *const i32) -> i32 {
    #[proctor(0)] *pointer.add(__proctor_wrapper_index(1))
}
unsafe fn target(mut pointer: &[i32]) -> i32 {
    #[proctor(0)] pointer[index(1)]
}
```

Expected result:

- Selection: sole anchor grows through `add` and dereference, selecting
  `*pointer.add(__proctor_wrapper_index(1))`; no merge/overlap.
- Exact alignment with correspondence:
  - label 0: `*pointer.add(__proctor_wrapper_index(1))` -> `pointer[index(1)]`
- With correspondence, wrapper and
  implementation dump as one `<fn0>`. Emit one `i32` observation with anchor
  `<id0>` (`*const i32` -> `&[i32]`) and four expression types `i32`.
- Without correspondence: non-region callee mismatch skips label 0 and emits
  zero. Malformed, contradictory, or dangling records are fatal instead.
- Exact alignment without correspondence:
  - label 0: no alignment (non-region callee mismatch)

### P5-EDGE-21 `rejected_anchor_does_not_block_disjoint_valid_anchor`

```rust
unsafe fn source_copy(
    mut good: *const i32,
    mut rejected: *const [i32],
) -> i32 {
    #[proctor(0)] *good + (*rejected)[0]
}
unsafe fn target(mut good: &i32, mut rejected: *const [i32]) -> i32 {
    #[proctor(0)] *good + (*rejected)[0]
}
```

Expected result:

- Anchors/regions: source order is `good`, then `rejected`. `good` grows to
  `*good` and finishes before `+`; `rejected` grows to `*rejected` then Rejects
  at index-base role. Only one root survives; no merge/overlap.
- Exact alignment:
  - label 0: `*good` -> `*good`
- Observation: emit one record
  with anchor `<id0>` (`*const i32` -> `&i32`) and all four expression types
  `i32`. Rejection is per occurrence.

## 6. Closed trees, normalized types, and anonymization

Serializer tests in this section obtain the corresponding supported node from
the exact Rust snippet and compare the complete closed serde value.
For brevity, nested leaves shown as compact JSON are still literal expected
objects, not shared aliases. Every case also mutates one key/tag and asserts
strict rejection.

### P5-WIRE-TYPE-01 `aliases_and_cast_syntax_normalize_to_one_tree`

```rust
type Int = i32;
type Pointer = *const Int;
unsafe fn source_copy(mut pointer: Pointer) -> usize {
    #[proctor(0)] pointer as usize
}
unsafe fn target(mut pointer: *const i32) -> usize {
    #[proctor(0)] pointer as usize
}
```

Expected result:

- Type conversion: every binding/cast-derived semantic type contains no
  `Int`/`Pointer`. The exact raw-pointer tree is
`{"kind":"raw_pointer","mutability":"const","pointee":{"kind":"primitive","name":"i32"}}`
and the exact cast target is `{"kind":"primitive","name":"usize"}`.
- Selection: label 0 anchor `pointer` grows through cast to the complete
  `pointer as usize`.
- Exact alignment:
  - label 0: `pointer as usize` -> `pointer as usize`
- Output: emit one
  observation. Anchor binding types are
  `raw_pointer(const,primitive(i32))` on both sides; all four expression types
  are primitive `usize`.

### P5-WIRE-TYPE-02 `slice_array_raw_ref_tuple_and_primitives_have_exact_variants`

```rust
type Types<'a> = (&'a mut [i16], [u8; 3], *mut bool, (char, f64, str));
unsafe fn source_copy(mut pointer: *const i32) -> i32 {
    #[proctor(0)] *pointer
}
unsafe fn target(mut pointer: &i32) -> i32 {
    #[proctor(0)] *pointer
}
```

Expected result:

- Type-converter output for resolved `Types` is a tuple whose ordered elements
  are exactly `reference(mutable,slice(primitive(i16)))`,
  `array(primitive(u8),length=3)`, `raw_pointer(mut,primitive(bool))`, and
  `tuple[primitive(char),primitive(f64),primitive(str)]`; no lifetime field is
  present. Unit is `{"kind":"tuple","elements":[]}` and never is
  `{"kind":"primitive","name":"never"}`.
- Exact alignment for the labeled extraction control:
  - label 0: `*pointer` -> `*pointer`
- The control emits one record with `*const i32` -> `&i32` anchor types and four `i32`
  expression types. The converter assertions are independent owned-value tests.

The exact `Types` value is:

```json
{
  "kind": "tuple",
  "elements": [
    {"kind": "reference", "mutability": "mutable", "pointee": {"kind": "slice", "element": {"kind": "primitive", "name": "i16"}}},
    {"kind": "array", "element": {"kind": "primitive", "name": "u8"}, "length": 3},
    {"kind": "raw_pointer", "mutability": "mut", "pointee": {"kind": "primitive", "name": "bool"}},
    {"kind": "tuple", "elements": [{"kind": "primitive", "name": "char"}, {"kind": "primitive", "name": "f64"}, {"kind": "primitive", "name": "str"}]}
  ]
}
```

### P5-WIRE-TYPE-03 `external_generic_adt_uses_defining_identity_and_type_arguments`

```rust
use std::boxed::Box as Heap;
use std::option::Option as Maybe;
unsafe fn source_copy(mut pointer: *const Maybe<Heap<i32>>) -> i32 {
    #[proctor(0)] 0
}
unsafe fn target(mut pointer: &Maybe<Heap<i32>>) -> i32 {
    #[proctor(0)] 0
}
```

Expected result for the aliased form:

- Type converter returns nested `adt` nodes: outer `Option` has
  `adt_kind:"enum"`, canonical defining crate/path, and one argument; that
  argument is `Box` with `adt_kind:"struct"`, canonical defining crate/path,
  and primitive `i32`. No alias/import spelling occurs. The function's label
  has no pointer occurrence in its expression, so it emits zero observations.
- Exact alignment:
  - label 0: no alignment (no eligible anchor)

The exact normalized ADT subtree is:

```json
{
  "kind": "adt",
  "adt_kind": "enum",
  "identity": {"kind": "external", "crate": "core", "path": ["option", "Option"]},
  "arguments": [
    {
      "kind": "adt",
      "adt_kind": "struct",
      "identity": {"kind": "external", "crate": "alloc", "path": ["boxed", "Box"]},
      "arguments": [{"kind": "primitive", "name": "i32"}]
    }
  ]
}
```

Exact absolute-spelling input is:

```rust
unsafe fn source_copy(
    mut pointer: *const ::std::option::Option<::std::boxed::Box<i32>>,
) -> i32 {
    #[proctor(0)] 0
}
unsafe fn target(
    mut pointer: &::std::option::Option<::std::boxed::Box<i32>>,
) -> i32 {
    #[proctor(0)] 0
}
```

Expected result for the absolute-spelling form:

- Its normalized pointer/ADT trees are byte-for-value equal to the aliased
  form. It likewise emits zero observations; no new crate or test dependency is
  added because `std` is supplied by the toolchain.
- Exact alignment:
  - label 0: no alignment (no eligible anchor)

### P5-WIRE-TYPE-04 `local_adts_fields_and_variants_are_anonymized_consistently`

```rust
struct Node { value: i32 }
union Cell { value: i32 }
enum Choice { Node(Node), Cell(Cell) }
unsafe fn source_copy(mut pointer: *const Choice) -> i32 {
    match &*pointer { Choice::Node(node) => { node.value }, _ => { 0 } }
}
unsafe fn target(mut pointer: &Choice) -> i32 {
    match pointer { Choice::Node(node) => { node.value }, _ => { 0 } }
}
```

Expected result:

- Type/identity compiler unit allocates `<enum0>`, `<struct0>`, `<union0>`,
  `<variant0>`, and `<field0>` in first-occurrence order and reuses each resolved
  definition across source/target. The owner on local field/variant identities
  is the corresponding anonymized ADT. JSON contains none of `Node`, `Cell`,
  `Choice`, or `value`.
- This fixture has no PROCTOR labels and is a converter/identity unit; anchor,
  region, alignment, and observation emission are not applicable.

Exact owned identity/type values asserted by the unit:

```text
{"kind":"adt","adt_kind":"enum","identity":{"kind":"local","id":"<enum0>"},"arguments":[]}
{"kind":"adt","adt_kind":"struct","identity":{"kind":"local","id":"<struct0>"},"arguments":[]}
{"kind":"adt","adt_kind":"union","identity":{"kind":"local","id":"<union0>"},"arguments":[]}
{"kind":"local","owner":{"kind":"local","id":"<enum0>"},"id":"<variant0>"}
{"kind":"local","owner":{"kind":"local","id":"<struct0>"},"id":"<field0>"}
```

### P5-WIRE-TYPE-05 `unrepresentable_recorded_type_discards_one_observation`

```rust
unsafe fn source_copy(
    mut unrepresentable: *const fn(i32) -> i32,
    mut pointer: *const i32,
) {
    #[proctor(0)] unrepresentable;
    #[proctor(1)] *pointer;
}
unsafe fn target(
    mut unrepresentable: &fn(i32) -> i32,
    mut pointer: &i32,
) {
    #[proctor(0)] unrepresentable;
    #[proctor(1)] *pointer;
}
```

Expected result:

- Anchors/regions: label 0's raw-pointer `unrepresentable` anchor selects the
  path `unrepresentable`; label 1's `pointer` anchor selects `*pointer`. Each
  label has one region and no merge or overlap.
- Exact alignment:
  - label 0: `unrepresentable` -> `unrepresentable`
  - label 1: `*pointer` -> `*pointer`
- Label 0 reaches observation dumping normally. Its source expression and
  anchor type require `raw_pointer(const,function(i32 -> i32))`, while the
  target requires `reference(shared,function(i32 -> i32))`. Because function
  types are not representable by `TypeTree`, only this mapped observation is
  discarded.
- Label 1 emits one record with anchor `<id0>` (`*const i32` -> `&i32`) and
  four `i32` expression types. Extraction succeeds with exactly that record.

### P5-WIRE-TYPE-06 `four_expression_types_and_two_anchor_types_are_all_recorded`

```rust
unsafe fn take_ref(mut value: &i32) -> i32 { *value }
unsafe fn __proctor_wrapper_take_ref(mut value: *const i32) -> i32 {
    take_ref(&*value)
}
unsafe fn source_copy(mut pointer: *mut i32) -> i32 {
    #[proctor(0)] __proctor_wrapper_take_ref(pointer)
}
unsafe fn target(mut pointer: &mut i32) -> i32 {
    #[proctor(0)] take_ref(pointer)
}
```

Expected result:

- `accepted_correspondence` is exactly
  `[{"item_id":5,"logical_path":"take_ref","implementation_path":"take_ref",
  "wrapper_path":"__proctor_wrapper_take_ref"}]`. It maps the source-side
  wrapper to implementation `take_ref`; both paths denote the same logical
  callable for alignment.
- Selection: label 0's `pointer` anchor finishes at the resolved direct-call
  argument, selecting `pointer`; there is no merge or overlap.
- Exact alignment:
  - label 0: `pointer` -> `pointer`
- Observation facts: source type `raw_pointer(mut,i32)`, source adjusted
  type `raw_pointer(const,i32)`, target type `reference(mutable,i32)`, target
  adjusted type `reference(shared,i32)`. Anchor `<id0>` records binding types
  `raw_pointer(mut,i32)` -> `reference(mutable,i32)`. No context-required type
  key exists.

Exact observation:

```json
{
  "source_expression":{"kind":"path","value":{"kind":"binding","id":"<id0>"}},
  "target_expression":{"kind":"path","value":{"kind":"binding","id":"<id0>"}},
  "pointer_anchors":[{"id":"<id0>","source_type":{"kind":"raw_pointer","mutability":"mut","pointee":{"kind":"primitive","name":"i32"}},"target_type":{"kind":"reference","mutability":"mutable","pointee":{"kind":"primitive","name":"i32"}}}],
  "source_type":{"kind":"raw_pointer","mutability":"mut","pointee":{"kind":"primitive","name":"i32"}},
  "source_adjusted_type":{"kind":"raw_pointer","mutability":"const","pointee":{"kind":"primitive","name":"i32"}},
  "target_type":{"kind":"reference","mutability":"mutable","pointee":{"kind":"primitive","name":"i32"}},
  "target_adjusted_type":{"kind":"reference","mutability":"shared","pointee":{"kind":"primitive","name":"i32"}}
}
```

### P5-WIRE-DUMP-01 `array_tuple_repeat_and_block_variants_are_exact`

```rust
fn values() { let _ = ([1_i32], (2_i32,), [3_i32; 2], { 4_i32 }); }
```

Expected result:

- Dumper output is, in tuple order, `array {elements:[integer(1,i32)]}`,
  `tuple {elements:[integer(2,i32)]}`, `repeat {value:integer(3,i32),
  count:integer(2,usize)}`, and `block {block:{statements:[expression {
  expression:integer(4,i32),semicolon:false}]}}`. The exact serde objects use
  only the schema keys named here and in the Phase 5 plan; parentheses add no
  node.
- This is a dumper unit with no raw-pointer anchor or observation extraction.

The four exact relevant expression values are:

```text
{"kind":"array","elements":[{"kind":"literal","value":{"kind":"integer","value":"1","type":"i32"}}]}
{"kind":"tuple","elements":[{"kind":"literal","value":{"kind":"integer","value":"2","type":"i32"}}]}
{"kind":"repeat","value":{"kind":"literal","value":{"kind":"integer","value":"3","type":"i32"}},"count":{"kind":"literal","value":{"kind":"integer","value":"2","type":"usize"}}}
{"kind":"block","block":{"statements":[{"kind":"expression","expression":{"kind":"literal","value":{"kind":"integer","value":"4","type":"i32"}},"semicolon":false}]}}
```

### P5-WIRE-DUMP-02 `call_method_and_path_variants_are_exact`

```rust
fn helper(mut value: i32) -> i32 { value }
struct Sink;
impl Sink { fn take(&self, value: i32) -> i32 { value } }
fn calls(mut sink: Sink, mut value: i32) -> i32 { sink.take(helper(value)) }
enum Choice { Value(i32) }
fn constructor() -> Choice { Choice::Value(1) }
```

Expected result for the local/standard input:

- The complete `calls` expression and constructor expression are, respectively:

```json
{"kind":"method_call","receiver":{"kind":"path","value":{"kind":"binding","id":"<id0>"}},"method":{"kind":"method","id":"<method0>"},"arguments":[{"kind":"call","callee":{"kind":"path","value":{"kind":"function","id":"<fn0>"}},"arguments":[{"kind":"path","value":{"kind":"binding","id":"<id1>"}}]}]}
```

```json
{"kind":"call","callee":{"kind":"path","value":{"kind":"constructor","adt":{"kind":"local","id":"<enum0>"},"variant":{"kind":"local","owner":{"kind":"local","id":"<enum0>"},"id":"<variant0>"}}},"arguments":[{"kind":"literal","value":{"kind":"integer","value":"1","type":"i32"}}]}
```

  Thus `sink`/`value` are `<id0>`/`<id1>`, `take` is `<method0>`, and `helper`
  is `<fn0>`; no textual local identity occurs.

Exact foreign value input is:

```rust
unsafe extern "C" {
    fn foreign_read(pointer: *const i32) -> i32;
    static FOREIGN_VALUE: i32;
}
unsafe fn values(mut pointer: *const i32) -> i32 {
    foreign_read(pointer) + FOREIGN_VALUE + std::ptr::read(pointer)
}
```

Expected result for the foreign-value input:

- Exact path values are `{"kind":"foreign_function","symbol":"foreign_read"}`
  and `{"kind":"foreign_static","symbol":"FOREIGN_VALUE"}`, with no ABI
  key. `std::ptr::read` is canonical external identity. The `pointer` call
  arguments can seed in extraction, but this subcase directly tests owned dump
  values rather than pairing a source/target unit.

The three complete path/call values are:

```text
{"kind":"call","callee":{"kind":"path","value":{"kind":"foreign_function","symbol":"foreign_read"}},"arguments":[{"kind":"path","value":{"kind":"binding","id":"<id0>"}}]}
{"kind":"path","value":{"kind":"foreign_static","symbol":"FOREIGN_VALUE"}}
{"kind":"call","callee":{"kind":"path","value":{"kind":"external","crate":"core","path":["ptr","read"]}},"arguments":[{"kind":"path","value":{"kind":"binding","id":"<id0>"}}]}
```

Exact rejected ABI input is:

```rust
unsafe extern "system" { fn foreign_read(pointer: *const i32) -> i32; }
unsafe fn values(mut pointer: *const i32) -> i32 { foreign_read(pointer) }
```

Expected result for the rejected-ABI input:

- Resolving `foreign_read` as `extern "system"` makes the containing dump
  unsupported; it is discarded and never serialized as a C foreign function.

### P5-WIRE-DUMP-03 `binary_unary_cast_and_operator_enums_are_exact`

```rust
fn ops(mut value: i32) -> i64 { (!(value == 0) as i32 + -value) as i64 }
```

Expected result for `ops`:

- Exact nesting is outer `cast(i64)` around `binary(add)` whose left is
  `cast(i32)` of `unary(not)` of `binary(equal)` and whose right is
  `unary(negate)`. Operator strings are exactly `equal`, `not`, `add`, and
  `negate`; binding IDs follow source preorder.

Complete `ops` value:

```json
{
  "kind": "cast",
  "expression": {
    "kind": "binary",
    "operator": "add",
    "left": {"kind":"cast","expression":{"kind":"unary","operator":"not","operand":{"kind":"binary","operator":"equal","left":{"kind":"path","value":{"kind":"binding","id":"<id0>"}},"right":{"kind":"literal","value":{"kind":"integer","value":"0","type":"i32"}}}},"type":{"kind":"primitive","name":"i32"}},
    "right": {"kind":"unary","operator":"negate","operand":{"kind":"path","value":{"kind":"binding","id":"<id0>"}}}
  },
  "type": {"kind":"primitive","name":"i64"}
}
```

Table-driven serializer values cover every other closed binary operator using
this exact local input; no unknown operator falls through:

```rust
fn all_ops(mut a: i32, mut b: i32, mut x: bool, mut y: bool) {
    let _ = (a - b, a * b, a / b, a % b, x && y, x || y);
    let _ = (a ^ b, a & b, a | b, a << b, a >> b);
    let _ = (a != b, a < b, a <= b, a > b, a >= b);
}
```

Expected result for `all_ops`:

- In source order, exact binary operator strings are `subtract`, `multiply`,
  `divide`, `remainder`, `and`, `or`, `bit_xor`, `bit_and`, `bit_or`,
  `shift_left`, `shift_right`, `not_equal`, `less`, `less_equal`, `greater`, and
  `greater_equal`.

Each independently allocated operator value has this exact complete shape;
the operator field takes the following values in fixture order:

```text
{"kind":"binary","operator":"subtract","left":{"kind":"path","value":{"kind":"binding","id":"<id0>"}},"right":{"kind":"path","value":{"kind":"binding","id":"<id1>"}}}
{"kind":"binary","operator":"multiply","left":{"kind":"path","value":{"kind":"binding","id":"<id0>"}},"right":{"kind":"path","value":{"kind":"binding","id":"<id1>"}}}
{"kind":"binary","operator":"divide","left":{"kind":"path","value":{"kind":"binding","id":"<id0>"}},"right":{"kind":"path","value":{"kind":"binding","id":"<id1>"}}}
{"kind":"binary","operator":"remainder","left":{"kind":"path","value":{"kind":"binding","id":"<id0>"}},"right":{"kind":"path","value":{"kind":"binding","id":"<id1>"}}}
{"kind":"binary","operator":"and","left":{"kind":"path","value":{"kind":"binding","id":"<id0>"}},"right":{"kind":"path","value":{"kind":"binding","id":"<id1>"}}}
{"kind":"binary","operator":"or","left":{"kind":"path","value":{"kind":"binding","id":"<id0>"}},"right":{"kind":"path","value":{"kind":"binding","id":"<id1>"}}}
{"kind":"binary","operator":"bit_xor","left":{"kind":"path","value":{"kind":"binding","id":"<id0>"}},"right":{"kind":"path","value":{"kind":"binding","id":"<id1>"}}}
{"kind":"binary","operator":"bit_and","left":{"kind":"path","value":{"kind":"binding","id":"<id0>"}},"right":{"kind":"path","value":{"kind":"binding","id":"<id1>"}}}
{"kind":"binary","operator":"bit_or","left":{"kind":"path","value":{"kind":"binding","id":"<id0>"}},"right":{"kind":"path","value":{"kind":"binding","id":"<id1>"}}}
{"kind":"binary","operator":"shift_left","left":{"kind":"path","value":{"kind":"binding","id":"<id0>"}},"right":{"kind":"path","value":{"kind":"binding","id":"<id1>"}}}
{"kind":"binary","operator":"shift_right","left":{"kind":"path","value":{"kind":"binding","id":"<id0>"}},"right":{"kind":"path","value":{"kind":"binding","id":"<id1>"}}}
{"kind":"binary","operator":"not_equal","left":{"kind":"path","value":{"kind":"binding","id":"<id0>"}},"right":{"kind":"path","value":{"kind":"binding","id":"<id1>"}}}
{"kind":"binary","operator":"less","left":{"kind":"path","value":{"kind":"binding","id":"<id0>"}},"right":{"kind":"path","value":{"kind":"binding","id":"<id1>"}}}
{"kind":"binary","operator":"less_equal","left":{"kind":"path","value":{"kind":"binding","id":"<id0>"}},"right":{"kind":"path","value":{"kind":"binding","id":"<id1>"}}}
{"kind":"binary","operator":"greater","left":{"kind":"path","value":{"kind":"binding","id":"<id0>"}},"right":{"kind":"path","value":{"kind":"binding","id":"<id1>"}}}
{"kind":"binary","operator":"greater_equal","left":{"kind":"path","value":{"kind":"binding","id":"<id0>"}},"right":{"kind":"path","value":{"kind":"binding","id":"<id1>"}}}
```

### P5-WIRE-DUMP-04 `unlabeled_if_while_loop_break_continue_are_exact`

```rust
fn controls(mut flag: bool) {
    loop {
        while flag { if flag { break; } else { continue; } }
        break;
    }
    let _value = loop { break 1_i32; };
}
```

Expected result for `controls`:

- Dumper returns nested exact `loop`, `while`, and `if` nodes; `break` without a
  value is `{"kind":"break","value":null}`, `continue` is
  `{"kind":"continue"}`, and the value loop contains `break` with integer
  `1_i32`. No node contains a loop-label field.

Complete outer-loop value (the repeated `flag` path is binding `<id0>`):

```json
{
  "kind": "loop",
  "body": {"statements":[
    {"kind":"expression","semicolon":false,"expression":{"kind":"while","condition":{"kind":"path","value":{"kind":"binding","id":"<id0>"}},"body":{"statements":[{"kind":"expression","semicolon":false,"expression":{"kind":"if","condition":{"kind":"path","value":{"kind":"binding","id":"<id0>"}},"then":{"statements":[{"kind":"expression","semicolon":true,"expression":{"kind":"break","value":null}}]},"else":{"kind":"block","block":{"statements":[{"kind":"expression","semicolon":true,"expression":{"kind":"continue"}}]}}}}]}}},
    {"kind":"expression","semicolon":true,"expression":{"kind":"break","value":null}}
  ]}
}
```

Complete value-loop initializer:

```json
{"kind":"loop","body":{"statements":[{"kind":"expression","semicolon":true,"expression":{"kind":"break","value":{"kind":"literal","value":{"kind":"integer","value":"1","type":"i32"}}}}]}}
```

The explicit-label rejection input is:

```rust
fn labeled() { 'outer: loop { break 'outer; } }
```

Expected result for `labeled`:

- Encountering explicit `'outer` makes the containing dump unsupported; the
  observation is discarded rather than erasing or serializing the label.

### P5-WIRE-DUMP-05 `assign_assignop_field_index_and_struct_are_exact`

```rust
struct Pair { value: i32 }
enum Choice { Pair { value: i32 } }
fn update(mut pair: Pair, mut values: [i32; 1]) -> Pair {
    pair.value = values[0];
    pair.value += 1;
    let _ = Choice::Pair { value: pair.value };
    Pair { value: pair.value, ..pair }
}
```

Expected result:

- First statement dumps the complete `assign` value below; the second dumps the
  complete `assign_op` value below. Constructors dump `struct` with ordered
  `fields`, anonymized local
  ADT/field identities, enum `variant` non-null for `Choice::Pair`, and `rest`
  non-null only for the final `Pair` update. No textual local names survive.
- This is a dumper unit; anchors/region alignment are not applicable.

Complete values for the four expressions, with each line an independently
allocated dump value:

```text
{"kind":"assign","left":{"kind":"field","base":{"kind":"path","value":{"kind":"binding","id":"<id0>"}},"field":{"kind":"local","owner":{"kind":"local","id":"<struct0>"},"id":"<field0>"}},"right":{"kind":"index","base":{"kind":"path","value":{"kind":"binding","id":"<id1>"}},"index":{"kind":"literal","value":{"kind":"integer","value":"0","type":"usize"}}}}
{"kind":"assign_op","operator":"add","left":{"kind":"field","base":{"kind":"path","value":{"kind":"binding","id":"<id0>"}},"field":{"kind":"local","owner":{"kind":"local","id":"<struct0>"},"id":"<field0>"}},"right":{"kind":"literal","value":{"kind":"integer","value":"1","type":"i32"}}}
{"kind":"struct","adt":{"kind":"local","id":"<enum0>"},"variant":{"kind":"local","owner":{"kind":"local","id":"<enum0>"},"id":"<variant0>"},"fields":[{"field":{"kind":"local","owner":{"kind":"local","id":"<enum0>"},"id":"<field0>"},"value":{"kind":"field","base":{"kind":"path","value":{"kind":"binding","id":"<id0>"}},"field":{"kind":"local","owner":{"kind":"local","id":"<struct0>"},"id":"<field1>"}}}],"rest":null}
{"kind":"struct","adt":{"kind":"local","id":"<struct0>"},"variant":null,"fields":[{"field":{"kind":"local","owner":{"kind":"local","id":"<struct0>"},"id":"<field0>"},"value":{"kind":"field","base":{"kind":"path","value":{"kind":"binding","id":"<id0>"}},"field":{"kind":"local","owner":{"kind":"local","id":"<struct0>"},"id":"<field0>"}}}],"rest":{"kind":"path","value":{"kind":"binding","id":"<id0>"}}}
```

### P5-WIRE-DUMP-06 `range_address_of_return_and_repeat_are_exact`

```rust
fn misc(mut value: i32) -> *const i32 {
    let _ = 0..=value;
    let _ = [value; 2];
    let _ = &value;
    let _ = &mut value;
    return &raw const value;
}
```

Expected result for `misc`:

- Dumper returns range `{limits:"closed",start:integer(0),end:binding(value)}`,
  repeat with integer count 2, shared and mutable `address_of` nodes, and
  `return` containing `address_of {borrow:"raw",mutability:"const"}`. All use
  only exact closed-schema keys.

Complete values, independently allocated:

```text
{"kind":"range","start":{"kind":"literal","value":{"kind":"integer","value":"0","type":"i32"}},"end":{"kind":"path","value":{"kind":"binding","id":"<id0>"}},"limits":"closed"}
{"kind":"repeat","value":{"kind":"path","value":{"kind":"binding","id":"<id0>"}},"count":{"kind":"literal","value":{"kind":"integer","value":"2","type":"usize"}}}
{"kind":"address_of","borrow":"reference","mutability":"const","expression":{"kind":"path","value":{"kind":"binding","id":"<id0>"}}}
{"kind":"address_of","borrow":"reference","mutability":"mut","expression":{"kind":"path","value":{"kind":"binding","id":"<id0>"}}}
{"kind":"return","value":{"kind":"address_of","borrow":"raw","mutability":"const","expression":{"kind":"path","value":{"kind":"binding","id":"<id0>"}}}}
```

Exact half-open/null-endpoint input is:

```rust
fn ranges(mut value: i32) { let _ = 0..value; let _ = ..; }
```

Expected result for `ranges`:

- `0..value` has `limits:"half_open"` and both endpoints non-null; `..` has
  `limits:"half_open"` and both endpoints null.

```text
{"kind":"range","start":{"kind":"literal","value":{"kind":"integer","value":"0","type":"i32"}},"end":{"kind":"path","value":{"kind":"binding","id":"<id0>"}},"limits":"half_open"}
{"kind":"range","start":null,"end":null,"limits":"half_open"}
```

### P5-WIRE-DUMP-07 `literal_variants_are_semantic_and_exact`

```rust
fn literals() {
    let _ = (true, 'x', b'x', "x", b"xy", c"xy", 12_u16, 1.5_f32);
}
```

Expected result for `literals`:

- Tuple elements are exact literal objects in order: bool `true`, char `"x"`,
  byte 120, string `"x"`, byte-string `[120,121]`, C-string `[120,121]` without
  terminator, integer `{value:"12",type:"u16"}`, and float
  `{bits:"3fc00000",type:"f32"}`.

Complete tuple value:

```json
{"kind":"tuple","elements":[{"kind":"literal","value":{"kind":"bool","value":true}},{"kind":"literal","value":{"kind":"char","value":"x"}},{"kind":"literal","value":{"kind":"byte","value":120}},{"kind":"literal","value":{"kind":"string","value":"x"}},{"kind":"literal","value":{"kind":"byte_string","value":[120,121]}},{"kind":"literal","value":{"kind":"c_string","value":[120,121]}},{"kind":"literal","value":{"kind":"integer","value":"12","type":"u16"}},{"kind":"literal","value":{"kind":"float","bits":"3fc00000","type":"f32"}}]}
```

Exact alternate-spelling input is:

```rust
fn alternate() { let _ = (r#"x"#, 0x0c_u16, 1.50_f32, -12_i32); }
```

Expected result for `alternate`:

- Raw string, hexadecimal integer, and alternate float spelling serialize to
  string `"x"`, integer `{value:"12",type:"u16"}`, and float
  `{bits:"3fc00000",type:"f32"}`. `-12_i32` is
  `unary {operator:"negate"}` around unsigned-magnitude integer
  `{value:"12",type:"i32"}`.

```json
{"kind":"tuple","elements":[{"kind":"literal","value":{"kind":"string","value":"x"}},{"kind":"literal","value":{"kind":"integer","value":"12","type":"u16"}},{"kind":"literal","value":{"kind":"float","bits":"3fc00000","type":"f32"}},{"kind":"unary","operator":"negate","operand":{"kind":"literal","value":{"kind":"integer","value":"12","type":"i32"}}}]}
```

### P5-WIRE-DUMP-08 `let_and_expression_statements_and_patterns_are_exact`

```rust
fn statements(mut value: i32) -> i32 {
    let ref mut alias: i32 = value;
    let _ = *alias;
    *alias;
    *alias
}
```

Expected result for `statements`:

- First statement is `let` with binding `<id0>`, mutability `immutable`,
  `by_ref:"mutable"`, primitive-i32 `type`, and binding-value initializer.
  Second is a wildcard `let`; the final two expression statements distinguish
  `semicolon:true` and `semicolon:false`. No let-else key exists.

Complete block value (`value` is `<id1>`):

```json
{
  "statements": [
    {"kind":"let","pattern":{"kind":"binding","id":"<id0>","mutability":"immutable","by_ref":"mutable"},"type":{"kind":"primitive","name":"i32"},"initializer":{"kind":"path","value":{"kind":"binding","id":"<id1>"}}},
    {"kind":"let","pattern":{"kind":"wildcard"},"type":null,"initializer":{"kind":"unary","operator":"deref","operand":{"kind":"path","value":{"kind":"binding","id":"<id0>"}}}},
    {"kind":"expression","expression":{"kind":"unary","operator":"deref","operand":{"kind":"path","value":{"kind":"binding","id":"<id0>"}}},"semicolon":true},
    {"kind":"expression","expression":{"kind":"unary","operator":"deref","operand":{"kind":"path","value":{"kind":"binding","id":"<id0>"}}},"semicolon":false}
  ]
}
```

Exact rejected inputs are:

```rust
fn let_else(mut value: Option<i32>) {
    let Some(value) = value else { return; };
    let _ = value;
}
fn destructure(mut pair: (i32, i32)) { let (left, right) = pair; let _ = left + right; }
```

Expected result for rejected inputs:

- Let-else and tuple-destructuring patterns each make their containing dump
  unsupported; the containing observation is discarded with no partial tree.

### P5-WIRE-DUMP-09 `unsupported_dump_node_discards_only_mapped_observation`

```rust
unsafe fn source_copy(mut left: *const i32, mut right: *const i32) -> i32 {
    #[proctor(0)] *left + *right
}
unsafe fn target(mut left: &i32, mut right: &i32) -> i32 {
    #[proctor(0)] (|| *left)() + *right
}
```

Expected result:

- Selection: source anchors `left`, then `right` select roots `*left` and
  `*right`. They are disjoint, with no merge or overlap. During simultaneous
  source-guided alignment, the selected left root acts as the wildcard for the
  target left operand.
- Exact alignment:
  - label 0:
    - `*left` -> `(|| *left)()`
    - `*right` -> `*right`
- Output: closure dumping discards only the first observation. Emit the second
  with anchor `<id0>` (`*const i32` -> `&i32`) and all four expression types
  `i32`; extraction succeeds.

### P5-WIRE-ANON-01 `namespaces_follow_source_then_target_first_occurrence`

```rust
struct Node { value: i32 }
struct Offset { index: usize }
unsafe fn helper(mut value: usize) -> usize { value }
unsafe fn source_copy(mut left: *const Node, mut offset: Offset) -> i32 {
    #[proctor(0)] (*left.add(helper(offset.index))).value
}
unsafe fn target(mut left: &[Node], mut offset: Offset) -> i32 {
    #[proctor(0)] left[helper(offset.index)].value
}
```

Expected result:

- Selection: label 0's `left` anchor grows through allowlisted `add` and
  dereference, selecting `*left.add(helper(offset.index))`. Field-base policy
  finishes at that base, before the outer `.value`; there is one region and no
  merge or overlap.
- Exact alignment:
  - label 0: `*left.add(helper(offset.index))` -> `left[helper(offset.index)]`
- IDs in the mapped source tree are `left` `<id0>`, `helper` `<fn0>`, and
  `offset` `<id1>`. Resolving `offset.index` assigns local `Offset` `<struct0>`
  and its field `<field0>`; the anchor and expression types then assign local
  `Node` `<struct1>`. Target dumping reuses every identity. The outer
  `Node::value` field is a matching non-region leaf and is not part of either
  mapped expression. Original names, compiler IDs, and spans are absent.
- Observation facts: anchor `<id0>` records `*const Node` -> `&[Node]`; the
  source/source-adjusted and target/target-adjusted expression types are all
  local `Node` (`<struct1>`).

### P5-WIRE-ANON-02 `crate_local_constant_static_and_method_policy`

Rust input:

```rust
static OFFSET: usize = 0;
const STEP: usize = 1;
struct Index;
impl Index {
    const BASE: usize = 2;
    fn get(&self) -> usize { 0 }
    fn base() -> usize { 0 }
}
unsafe fn source_copy(mut pointer: *const i32, mut index: Index) -> i32 {
    #[proctor(0)]
    *pointer.add(index.get() + Index::base() + OFFSET + STEP + Index::BASE)
}
unsafe fn target(mut pointer: &[i32], mut index: Index) -> i32 {
    #[proctor(0)]
    pointer[index.get() + Index::base() + OFFSET + STEP + Index::BASE]
}
```

Expected result for the inherent-item input:

- Selection: sole anchor grows through `add` and dereference, selecting the
  complete source dereference.
- Exact alignment:
  - label 0: `*pointer.add(index.get() + Index::base() + OFFSET + STEP + Index::BASE)` -> `pointer[index.get() + Index::base() + OFFSET + STEP + Index::BASE]`
- Output: emit one
  `i32` observation with anchor `*const i32` -> `&[i32]`.
- IDs in source-tree order: `index.get` is `<method0>`, `Index::base` is
  `<method1>`, `OFFSET` is `<static0>`, `STEP` is `<const0>`, and `Index::BASE`
  is `<const1>`. Original names are absent; external identities would remain
  canonical.

Exact trait-method rejection input:

```rust
trait IndexValue { fn get(&self) -> usize; }
struct Index;
impl IndexValue for Index { fn get(&self) -> usize { 0 } }
unsafe fn source_copy(mut pointer: *const i32, mut index: Index) -> i32 {
    #[proctor(0)] *pointer.add(index.get())
}
unsafe fn target(mut pointer: &[i32], mut index: Index) -> i32 {
    #[proctor(0)] pointer[index.get()]
}
```

Expected result for the trait-method input:

- Selection reaches the same source/target roots, but dumping resolved local
  trait method `get` is unsupported.
- Exact alignment before dumping:
  - label 0: `*pointer.add(index.get())` -> `pointer[index.get()]`
- Discard the one observation and emit zero.

## 7. Python protocols, acceptance, and publication

### P5-PY-01 `command_builders_use_exact_four_output_and_extract_argv`

```rust
pub unsafe fn read(mut pointer: *const i32) -> i32 { *pointer }
```

Expected result:

- Replace argv is exactly `crat-tool replace --request
  replacement-request.json --output candidate.rs --statement-pairs-output
  replacement-statement-pairs.json --observation-source-output
  replacement-observation.rs --observation-metadata-output
  replacement-observation-metadata.json current-project`.
- Extraction argv is exactly `crat-tool extract-observations --metadata
  replacement-observation-metadata.json --output extracted-observations.json
  replacement-observation.rs`. Flag order is fixed; the four replacement paths
  and three extraction paths are pairwise distinct within their commands.

### P5-PY-02 `tooling_clears_requires_and_cleans_every_exact_output`

```rust
pub unsafe fn read(mut pointer: *const i32) -> i32 { *pointer }
```

Expected result:

- Before invocation, each stale exact regular file or symlink is removed;
  a directory or other nonregular node at an exact destination fails without
  touching unrelated paths.
- Zero exit missing any required output and nonzero exit after any partial write
  are failures. Every owned temporary and earlier renamed final is absent after
  each failure; success leaves exactly the required regular nonsymlink files.
  The test asserts cleanup-transactional state, not cross-file atomic visibility.

### P5-PY-03 `metadata_digests_and_cross_file_contract_are_strict`

```rust
pub unsafe fn read(mut pointer: *const i32) -> i32 { *pointer }
```

Exact companion bytes are candidate
`pub unsafe fn read(mut pointer: &i32) -> i32 { *pointer }\n`, statement sidecar
`{"schema_version":1,"statements":[{"item_id":7,"path":"read","label":0,"after_statement":"*pointer"}]}\n`, and observation source
`unsafe fn source_copy(mut pointer: *const i32) -> i32 { #[proctor(0)] *pointer }\nunsafe fn read(mut pointer: &i32) -> i32 { #[proctor(0)] *pointer }\n`.
The request has item 7/path `read`, transform labels `[0]`, and accepted
correspondence `[]`. Exact valid metadata:

```json
{
  "schema_version": 1,
  "candidate_sha256": "6c3ea56d9debffcf25243e9a41d58805af269772d266088c83d19053f7ccebf1",
  "statement_pairs_sha256": "2b8e6af47f728734179fa6d023e74d812a888e65d4f111e8ff4a6c01f75c823b",
  "observation_source_sha256": "9f0b4479a714b852af82426ac4665d1159fddd22472555b988430253c14cf499",
  "accepted_correspondence": [],
  "new_correspondence": [{"item_id":7,"logical_path":"read","implementation_path":"read","wrapper_path":"__proctor_wrapper_read"}],
  "current_items": [{"item_id":7,"logical_path":"read","source_copy_path":"__proctor_source_read","implementation_path":"read","wrapper_path":"__proctor_wrapper_read","transform_labels":[0]}]
}
```

Expected result:

- Valid fixture: all three recomputed SHA-256 values match exact bytes;
  accepted echo equals the request tuple; `current_items` and
  `new_correspondence` agree and remain in request order.
- Each digest/echo/shared-field/order/label mutation is `StageFailure` before
  candidate installation. Uniqueness is checked independently for item IDs,
  logical paths, implementation paths, wrappers, and source copies, plus the
  forbidden cross-category collisions. Same-record logical path equal to
  implementation path succeeds; prohibited cross-record/cross-field equality
  fails. Merely absent callable correspondence remains loader-valid and later
  causes a conservative alignment skip.

Exact `StageFailure` message oracles are
`replacement metadata <digest-field> does not match <companion-name> bytes`,
`replacement metadata accepted_correspondence does not equal the request`,
`replacement metadata current_items[0].<field> disagrees with new_correspondence[0].<field>`,
`replacement metadata records do not preserve request order`, and
`replacement metadata current_items[0].transform_labels does not equal [0]`.
Uniqueness failures use
`replacement metadata has duplicate <item_id|logical_path|implementation_path|wrapper_path|source_copy_path> <value>`;
cross-category failures use
`replacement metadata path <value> is used as both <first-category> and <second-category>`.
The test substitutes the exact mutated field/value/category into these fixed
templates and compares the complete message.

### P5-PY-04 `real_cli_rejects_versions_paths_and_partial_outputs`

```rust
pub unsafe fn read(mut pointer: *const i32) -> i32 { *pointer }
```

Use schema-version-1 item 7/path `read`, empty accepted correspondence, and the
exact three digests/current/new records displayed locally here:

The digest inputs are exactly candidate
`pub unsafe fn read(mut pointer: &i32) -> i32 { *pointer }\n`, statement sidecar
`{"schema_version":1,"statements":[{"item_id":7,"path":"read","label":0,"after_statement":"*pointer"}]}\n`,
and observation source
`unsafe fn source_copy(mut pointer: *const i32) -> i32 { #[proctor(0)] *pointer }\nunsafe fn read(mut pointer: &i32) -> i32 { #[proctor(0)] *pointer }\n`.

```json
{"schema_version":1,"items":[{"id":7,"path":"read","name":"read","skeleton":"unsafe fn read(mut pointer: &i32) -> i32 { #[proctor(0)] *pointer }","needs_transformation":true,"statements_requiring_transformation":[0]}],"transformation":"unsafe fn read(mut pointer: &i32) -> i32 { #[proctor(0)] *pointer }","accepted_correspondence":[]}
```

```json
{"schema_version":1,"candidate_sha256":"6c3ea56d9debffcf25243e9a41d58805af269772d266088c83d19053f7ccebf1","statement_pairs_sha256":"2b8e6af47f728734179fa6d023e74d812a888e65d4f111e8ff4a6c01f75c823b","observation_source_sha256":"9f0b4479a714b852af82426ac4665d1159fddd22472555b988430253c14cf499","accepted_correspondence":[],"new_correspondence":[{"item_id":7,"logical_path":"read","implementation_path":"read","wrapper_path":"__proctor_wrapper_read"}],"current_items":[{"item_id":7,"logical_path":"read","source_copy_path":"__proctor_source_read","implementation_path":"read","wrapper_path":"__proctor_wrapper_read","transform_labels":[0]}]}
```

Expected result:

- Real CLI: every failure exits 1 and writes
  `crat-tool: <code>: <message>\n`. Exact pairs are:
  `unsupported_schema_version` / `unsupported schema_version 2`;
  `output_path_collision` / `output paths must be pairwise distinct`;
  `metadata_io` / `failed to read replacement-observation-metadata.json`;
  `observation_source_digest_mismatch` / `observation source SHA-256 does not match metadata`;
  `malformed_proctor_label` / `source_copy contains a malformed proctor statement label`;
  and `output_io` / `failed to create extraction output temporary` for an
  unwritable destination. Schema 1 with the request and metadata embedded in
  this case succeeds.
- Filesystem matrix: failure at each of four replacement temporary writes/final
  renames and at extraction temporary/final rename removes all owned
  temporaries and every earlier final. Cleanup failure is appended
  deterministically to the primary error. This is Rust CLI coverage, not a
  duplicate assertion inferred from the Python tooling test.
  Each simulated write/rename failure uses code `output_io` and exact message
  `failed to write <path>.tmp` or `failed to rename <path>.tmp to <path>` with
  the concrete matrix path substituted; cleanup failure appends
  `; cleanup failed: <cleanup-message>` to that line.

### P5-PY-05 `extraction_runs_only_after_successful_build`

```rust
pub unsafe fn read(mut pointer: *const i32) -> i32 { *pointer }
```

Expected result:

- Call sequence is replace-1, install/build-1(fail), repair, replace-2,
  install/build-2(success), extract-2. There is no extract-1 call.
- State contains only attempt 2's statement pairs, observations, and new
  correspondence. Exact counters are `function_count=1`, `scc_count=1`,
  `llm_generation_calls=2`, `repair_calls=1`, `structural_failures=0`,
  `compilation_failures=1`, and `cargo_builds=3` (initial normalized build plus
  two candidate builds). Tool-call counts are replace=2, validate=2, build=3,
  extract=1; extraction has no stage metric and does not change those counters.

### P5-PY-06 `post_acceptance_extraction_failure_is_fatal_not_repairable`

```rust
pub unsafe fn read(mut pointer: *const i32) -> i32 { *pointer }
```

Expected result:

- Either extraction process failure or malformed result is fatal immediately
  after the accepted build; there is no repair/LLM call.
- No new correspondence, statement pairs, or observations are committed, and
  no final project, Markdown, or JSON artifact is usable. The internal accepted
  current source need not be rolled back solely to retry an already accepted
  transformation.

### P5-PY-07 `strict_observation_loader_uses_exact_valid_base_document`

```rust
pub unsafe fn read(mut pointer: *const i32) -> i32 { *pointer }
```

Valid base response:

```json
{
  "schema_version": 1,
  "observations": [
    {
      "source_expression": {"kind": "path", "value": {"kind": "binding", "id": "<id0>"}},
      "target_expression": {"kind": "path", "value": {"kind": "binding", "id": "<id0>"}},
      "pointer_anchors": [{"id": "<id0>", "source_type": {"kind": "raw_pointer", "mutability": "const", "pointee": {"kind": "primitive", "name": "i32"}}, "target_type": {"kind": "reference", "mutability": "shared", "pointee": {"kind": "primitive", "name": "i32"}}}],
      "source_type": {"kind": "raw_pointer", "mutability": "const", "pointee": {"kind": "primitive", "name": "i32"}},
      "source_adjusted_type": {"kind": "raw_pointer", "mutability": "const", "pointee": {"kind": "primitive", "name": "i32"}},
      "target_type": {"kind": "reference", "mutability": "shared", "pointee": {"kind": "primitive", "name": "i32"}},
      "target_adjusted_type": {"kind": "reference", "mutability": "shared", "pointee": {"kind": "primitive", "name": "i32"}}
    }
  ]
}
```

Expected result:

- The displayed document loads to exactly one ordered observation. Its root
  expressions are binding `<id0>`; anchor types are `*const i32` -> `&i32`;
  expression and adjusted types are the same respective pair.
- Each version/key/Boolean-integer/tag/enum/ID/target-only-ID/anchor-order/
  nested-type/trailing-content mutation raises `StageFailure`. The loader
  neither repairs nor sorts any value.

### P5-PY-08 `accepted_correspondence_promotes_after_extraction`

```rust
pub unsafe fn leaf(mut pointer: *const i32) -> i32 { *pointer }
pub unsafe fn root(mut pointer: *const i32) -> i32 { leaf(pointer) }
```

Expected result:

- Success sequence: leaf build, leaf extraction, promote leaf record, then root
  replacement. Root request carries leaf's record value-for-value in retained
  tuple position 0; root metadata echoes it unchanged and proposes only root's
  new record.
- Reordered/altered echo is fatal. If leaf extraction fails, leaf is not
  promoted, root is never processed, and retained correspondence stays empty.

### P5-PY-09 `mechanical_scc_promotes_correspondence_without_extracting`

```rust
pub unsafe fn leaf(mut pointer: *const i32) -> i32 { *pointer }
pub unsafe fn root(mut pointer: *const i32) -> i32 { leaf(pointer) }
```

Expected result:

- Successful mechanical leaf validates all four replacement outputs and its
  build, makes zero extraction calls because every `transform_labels` list is
  empty, promotes the leaf wrapper record, and passes it to root.
- Failed mechanical build makes zero extraction calls and promotes neither
  correspondence nor statement pairs.

### P5-PY-10 `observations_retain_schedule_producer_and_duplicate_order`

```rust
pub unsafe fn leaf(mut pointer: *const i32) -> i32 { *pointer }
pub unsafe fn root(mut pointer: *const i32) -> i32 { leaf(pointer) + *pointer }
```

Expected result:

- Given producer list `[leaf0, leaf0]` followed by `[root0, root1]`, final
  `observations` is exactly `[leaf0, leaf0, root0, root1]`. Neither duplicate
  removal nor sorting occurs.
- Two runs produce byte-identical pretty JSON; Python dict/set iteration cannot
  affect the recorded SCC/producer order.

### P5-PY-11 `empty_document_is_always_published_beside_statement_pairs`

```rust
pub unsafe fn scalar(mut value: i32) -> i32 { value + 1 }
```

Expected result:

- Both artifact-directory and workdir-fallback runs publish
  `observations.json` beside `statement-pairs.md` with exact bytes:

```json
{
  "schema_version": 1,
  "observations": []
}
```

  plus one terminal newline.
- The JSON is absent from the output Rust project and `StageOutput.logs`;
  statement-pair Markdown bytes are unchanged.

### P5-PY-12 `nonempty_artifact_is_pretty_deterministic_and_data_only`

```rust
pub unsafe fn read(mut pointer: *const i32) -> i32 { *pointer }
```

Local extracted document:

```json
{
  "schema_version": 1,
  "observations": [
    {
      "source_expression": {"kind": "path", "value": {"kind": "binding", "id": "<id0>"}},
      "target_expression": {"kind": "path", "value": {"kind": "binding", "id": "<id0>"}},
      "pointer_anchors": [{"id": "<id0>", "source_type": {"kind": "raw_pointer", "mutability": "const", "pointee": {"kind": "primitive", "name": "i32"}}, "target_type": {"kind": "reference", "mutability": "shared", "pointee": {"kind": "primitive", "name": "i32"}}}],
      "source_type": {"kind": "raw_pointer", "mutability": "const", "pointee": {"kind": "primitive", "name": "i32"}},
      "source_adjusted_type": {"kind": "raw_pointer", "mutability": "const", "pointee": {"kind": "primitive", "name": "i32"}},
      "target_type": {"kind": "reference", "mutability": "shared", "pointee": {"kind": "primitive", "name": "i32"}},
      "target_adjusted_type": {"kind": "reference", "mutability": "shared", "pointee": {"kind": "primitive", "name": "i32"}}
    }
  ]
}
```

Expected result:

- Publish this exact document with schema key order, two-space indentation, and
  one terminal newline. Repeated runs are byte-identical.
- No observation source, PROCTOR label, or metadata file enters the output Rust
  project; the JSON is not a `StageOutput.logs` entry.

### P5-PY-13 `project_markdown_json_publish_as_one_cleanup_transaction`

```rust
pub unsafe fn read(mut pointer: *const i32) -> i32 { *pointer }
```

Expected result:

- Success publishes exactly final project, `statement-pairs.md`, and
  `observations.json` in that order after preparing artifact temporaries.
- Every simulated preparation/copy/project-rename/Markdown-rename/JSON-rename
  failure removes all owned temporaries and every earlier final from this
  invocation, preserves unrelated/pre-existing paths, and reports cleanup
  failures after the primary error. Cross-file visibility is not claimed.
- Invocation start removes stale exact regular files/symlinks, rejects exact
  directories, and rejects either artifact path overlapping input, current, or
  output project boundaries.

## 8. Completion criteria

Phase 5 is not complete until:

- all named cases are implemented with their Rust input local to the case;
- the ordinary candidate and statement-pair sidecar remain byte-for-byte;
- every closed expression/type/operator/literal variant has exact serde tests;
- real CLI process/error/cleanup and schema-version tests pass;
- focused/full Rust and Python tests pass;
- `cargo fmt` and `cargo clippy --workspace --all-targets` pass under the
  repository warning policy; and
- `prototype-desc.md` is reorganized to describe the implementation without
  appending planning history.
