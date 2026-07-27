# Phase 4 Test Plan: Local-Transformation Orchestration

## 1. Purpose

This document specifies the complete automated test suite planned for Phase 4
of the local-transformation prototype. It is the hand-over contract for:

- the standalone PROCTOR `local_transformation` stage;
- strict Python loading of immutable skeleton records;
- function-graph construction, SCC computation, and deterministic leaf
  scheduling;
- dependency-context closure, rendering, and character budgeting;
- versioned prompt construction through PROCTOR's prompt library;
- LLM response extraction;
- Crat protocol orchestration without reimplementing Crat semantics;
- bounded structural and compiler repair;
- transactional library-source installation and rollback;
- persistent incremental Cargo target reuse;
- final artifact creation and stage reporting; and
- mandatory abort-on-context-overflow behavior.

Phases 1, 2, and 3 are complete. Do not duplicate their semantic assertions
at the Python level. In particular, Phase 4 tests do not inspect whether Crat
generated correct skeleton Rust, found correct dependencies, validated labels
correctly, normalized the correct functions, generated correct wrappers, or
rewrote correct call sites. Those behaviors remain owned by the existing Rust
test suites.

Phase 4 also amends Crat's skeleton presentation: every non-`ref` binding is
shown as `mut` in both source and target renderings, while `ref` and `ref mut`
are preserved exactly. This document specifies updates to existing in-memory
Crat skeleton tests for that amendment. It does not add a new Crat test
function or a filesystem-changing Crat test.

All amendments to earlier implementation belong to Phase 4. Do not edit
`phase-1-test-plan.md`, `phase-2-test-plan.md`, or
`phase-3-test-plan.md`.

The Python suite contains 66 named cases:

| Area | Cases |
| --- | ---: |
| Skeleton loading | 8 |
| Graphs, SCCs, and scheduling | 8 |
| Dependency context and budgets | 12 |
| Prompt construction, tool protocols, and response extraction | 12 |
| Attempt and repair orchestration | 11 |
| Source transactions, artifacts, usage, and stage contract | 15 |

Later bug fixes may add regressions, but Phase 4 is not complete until every
case in this document is implemented and passing.

## 2. Test execution and ownership policy

Put Python stage implementation under `proctor/stages/local-transformation/`
and importable unit-testable logic in ordinary `.py` modules beside
`main.py`. Put its Python tests under `proctor/tests/`, using a focused file
such as `test_local_transformation.py`. Run them from `proctor/` with:

```bash
uv run pytest tests/test_local_transformation.py
```

After implementation, also run the normal repository checks described by the
PROCTOR contributor guidance. Update and run the existing in-memory Crat
skeleton tests with:

```bash
cd stages/crat
cargo test -p tools skeleton::tests
```

Do not add a filesystem or CLI test for Crat's Expand cleanup. This document
adds no new Crat test function and no live-provider or real-toolchain
end-to-end test. Real-program evaluation follows a separate validation plan
and is outside this document.

Every default Python test is offline and deterministic. It may use
`tmp_path`, read/write temporary project files, and invoke pure stage
functions. It must not:

- invoke the real `crat`, `crat-tool`, Cargo, or rustc;
- make an HTTP request or require an API key;
- retest the semantic content of any Crat-produced Rust or diagnostic;
- parse Rust in Python;
- depend on timing, filesystem enumeration order, Python hash order, or a
  particular provider SDK; or
- modify checked-in fixtures or process-global environment variables.

Use injected fakes for the tool runner, Cargo builder, and LLM client. The
stage's production runner remains subprocess-based, but its command assembly
and response handling must be callable without launching the command.

## 3. Exact common test data and helpers

Every case below either supplies a complete literal input or names one of the
exact fixtures in this section. Ellipses are never implicit input.

### 3.1 Exact record constructors

Use these test-only constructors exactly:

```python
def fn_record(
    item_id: int,
    path: str,
    name: str,
    dependencies: list[int],
    signature_dependencies: list[int] | None = None,
) -> dict[str, object]:
    sig_deps = [] if signature_dependencies is None else signature_dependencies
    return {
        "id": item_id,
        "path": path,
        "kind": "Fn",
        "name": name,
        "annotated_source": (
            f"unsafe fn {name}() {{\n"
            "    #[proctor(0)]\n"
            "    ()\n"
            "}"
        ),
        "annotated_skeleton": (
            f"unsafe fn {name}() {{\n"
            "    #[proctor(0)]\n"
            "    todo!()\n"
            "}"
        ),
        "source_signature": f"unsafe fn {name}()",
        "target_signature": f"unsafe fn {name}()",
        "signature_dependencies": sig_deps,
        "dependencies": dependencies,
    }


def type_record(
    item_id: int,
    path: str,
    kind: str,
    definition: str,
    dependencies: list[int],
) -> dict[str, object]:
    return {
        "id": item_id,
        "path": path,
        "kind": kind,
        "definition": definition,
        "dependencies": dependencies,
    }


def value_record(
    item_id: int,
    path: str,
    kind: str,
    declaration: str,
    dependencies: list[int],
    signature_dependencies: list[int],
) -> dict[str, object]:
    return {
        "id": item_id,
        "path": path,
        "kind": kind,
        "declaration": declaration,
        "signature_dependencies": signature_dependencies,
        "dependencies": dependencies,
    }
```

The constructors preserve the list order supplied by the case. They do not
sort, deduplicate, or validate.

### 3.2 Exact graph fixture

`GRAPH_RECORDS` is this exact list:

```python
GRAPH_RECORDS = [
    fn_record(0, "leaf", "leaf", []),
    fn_record(1, "left", "left", [0, 100]),
    fn_record(2, "right", "right", [0]),
    fn_record(3, "cycle::a", "a", [4]),
    fn_record(4, "cycle::b", "b", [1, 3]),
    fn_record(5, "self_rec", "self_rec", [2, 5]),
    fn_record(6, "root", "root", [3, 5]),
    fn_record(7, "isolated", "isolated", []),
    fn_record(8, "same_a::same", "same", [9]),
    fn_record(9, "same_b::same", "same", [8]),
    type_record(100, "Context", "Struct", "struct Context;", []),
]
```

The exact function graph is:

```text
0 -> {}
1 -> {0}
2 -> {0}
3 -> {4}
4 -> {1, 3}
5 -> {2, 5}
6 -> {3, 5}
7 -> {}
8 -> {9}
9 -> {8}
```

Dependency `1 -> 100` is not a graph edge because item 100 is not a function.

### 3.3 Exact dependency-context fixture

`CONTEXT_RECORDS` is this exact list:

```python
CONTEXT_RECORDS = [
    {
        **fn_record(0, "target", "target", [1, 2, 20], [20]),
        "annotated_source": (
            "unsafe fn target(mut p: *const S) -> i32 {\n"
            "    #[proctor(0)]\n"
            "    callee(p.cast()) + GLOBAL\n"
            "}"
        ),
        "annotated_skeleton": (
            "unsafe fn target(mut p: &S) -> i32 {\n"
            "    #[proctor(0)]\n"
            "    todo!()\n"
            "}"
        ),
        "source_signature": "unsafe fn target(mut p: *const S) -> i32",
        "target_signature": "unsafe fn target(mut p: &S) -> i32",
    },
    {
        **fn_record(1, "callee", "callee", [21, 22], [21]),
        "source_signature": "unsafe fn callee(mut p: *const T) -> i32",
        "target_signature": "unsafe fn callee(mut p: &T) -> i32",
    },
    value_record(
        2,
        "GLOBAL",
        "Static",
        "static GLOBAL: i32;",
        [23, 24],
        [23],
    ),
    {
        **fn_record(3, "peer", "peer", [0, 2], []),
        "source_signature": "unsafe fn peer() -> i32",
        "target_signature": "unsafe fn peer() -> i32",
    },
    type_record(20, "S", "Struct", "struct S { value: *const i32 }", [25]),
    type_record(21, "T", "TyAlias", "type T = U;", [26]),
    fn_record(22, "body_only", "body_only", []),
    value_record(23, "N", "Const", "const N: usize;", [27, 28], [27]),
    type_record(24, "BodyOnly", "Struct", "struct BodyOnly;", []),
    type_record(25, "V", "Union", "union V { value: i32 }", [26, 28]),
    type_record(26, "U", "Struct", "struct U { e: E }", [29]),
    type_record(27, "W", "Enum", "enum W { A }", []),
    type_record(28, "X", "Struct", "struct X;", []),
    type_record(29, "E", "Enum", "enum E { A }", []),
]
```

For target 0 as a nonrecursive singleton:

```text
mandatory direct IDs = [1, 2, 20]
depth 1 = [21, 23, 25]
depth 2 = [26, 27, 28]
depth 3 = [29]
never reached through dependency closure = [22, 24]
```

IDs 22 and 24 are body-only dependencies of a function/static dependency.

### 3.4 Exact fake results

Use these exact values:

```python
VALID_TRANSFORMATION = """unsafe fn target() {
    #[proctor(0)]
    ()
}"""

VALID_RESPONSE_TEXT = f"```rust\n{VALID_TRANSFORMATION}\n```"

INVALID_TRANSFORMATION = """unsafe fn target() {
    ()
}"""

INVALID_RESPONSE_TEXT = f"```rust\n{INVALID_TRANSFORMATION}\n```"

VALIDATOR_VALID = {"schema_version": 1, "status": "valid"}

VALIDATOR_INVALID = {
    "schema_version": 1,
    "status": "invalid",
    "failures": [
        {
            "id": 0,
            "name": "target",
            "failed_snippet": INVALID_TRANSFORMATION,
            "errors": [
                {
                    "code": "missing_label",
                    "message": "Function `target` (item 0): label 0 is missing.",
                }
            ],
        }
    ],
}

VALIDATOR_INVALID_RAW = """{
  "schema_version": 1,
  "status": "invalid",
  "failures": [
    {
      "id": 0,
      "name": "target",
      "failed_snippet": "unsafe fn target() {\\n    ()\\n}",
      "errors": [
        {
          "code": "missing_label",
          "message": "Function `target` (item 0): label 0 is missing."
        }
      ]
    }
  ]
}"""

VALIDATOR_SETUP_ERROR = {
    "schema_version": 1,
    "status": "setup_error",
    "error": {
        "code": "duplicate_expected_name",
        "message": "Expected function name `target` appears twice.",
    },
}

BUILD_OK = {
    "returncode": 0,
    "stdout": "build stdout\n",
    "stderr": "build stderr\n",
}

BUILD_FAIL = {
    "returncode": 101,
    "stdout": "checking local_project\n",
    "stderr": "error[E0308]: mismatched types\n",
}

TOOL_OK = {
    "returncode": 0,
    "stdout": "",
    "stderr": "",
}
```

The fake LLM response reports provider `replay`, model `phase4-test`, finish
reason `stop`, latency `0.25`, and usage:

```text
input_tokens = 100
cached_input_tokens = 20
output_tokens = 30
reasoning_tokens = 4
```

### 3.5 Fake runner event vocabulary

The injected fake runner records these event tuples without executing them:

```text
("build_tools", crat_dir)
("prepare", current_project, ("expand", "unexpand"), use_print)
("make_skeleton", current_project, skeleton_json)
("normalize", library_source, normalized_source)
("cargo_build", current_project)
("validate", validation_request, validation_response)
("replace", current_project, replacement_request, candidate_source)
```

Tests enqueue exact file contents and result dictionaries before calling stage
logic. An unexpected event fails the test immediately.

### 3.6 Exact stage-envelope fixture

Unless a stage-contract case gives an explicit field mutation, its exact typed
input is:

```python
BASE_STAGE_INPUT = StageInput(
    schema_version=1,
    run_id="phase4-run",
    stage_id="local_transformation",
    stage_index=0,
    item=None,
    inputs=InputArtifacts(
        c_project=None,
        rust_project=tmp_path / "input",
        test_package=None,
        rule_set=None,
    ),
    outputs=OutputDestinations(
        rust_project=tmp_path / "output",
        rule_set=None,
        artifacts_dir=tmp_path / "artifacts",
    ),
    config={},
    framework=FrameworkSettings(
        llm={
            "provider": "replay",
            "model": "phase4-test",
            "context_overflow": "error",
            "max_retries": 1,
            "pricing": {
                "replay/phase4-test": {
                    "input": 0.0,
                    "cached_input": 0.0,
                    "output": 0.0,
                }
            },
        },
        usage_log=tmp_path / "work" / "usage.jsonl",
        prompt_library=None,
        workdir=tmp_path / "work",
        budget=Budget(),
        timeout_s=3600,
    ),
)
```

The case creates `input/`, `artifacts/`, and `work/` before invocation unless
it is specifically testing a missing path. The output path is absent.
Field-mutation cases construct a new frozen dataclass value; they never mutate
this fixture or its nested LLM dictionary.

## 4. Existing Crat in-memory regression updates

Do not add a Crat test function for Phase 4. Update the existing unit tests in
`crates/tools/src/skeleton/tests.rs` so that the same in-memory compiler
fixtures enforce the new shared source/target presentation rule. Keep all
existing semantic coverage and make these exact changes.

In the existing
`target_parameters_and_simple_locals_are_mutable` test, use its current exact
input:

```rust
pub unsafe fn f(input: i32, mut existing: i32) -> i32 {
    let value = input;
    let mut total: i32 = existing;
    total += value;
    total
}
```

Rename the test to describe both presentations. Require
`source_signature` and `target_signature` to be exactly:

```rust
pub unsafe fn f(mut input: i32, mut existing: i32) -> i32
```

Require `annotated_source` to retain the original expressions while containing
`let mut value = input;` and `let mut total: i32 = existing;`. Keep the
existing target-skeleton assertion, including `mut input`, `mut existing`,
`let mut value: i32 = todo!();`, and `let mut total: i32 = todo!();`. Remove
the obsolete assertion that source `value` is non-`mut`.

In the existing `wildcards_remain_wildcards_and_source_is_unchanged` test, use
its current exact input:

```rust
pub unsafe fn f(pair: (i32, i32)) {
    let (_, value) = pair;
    let _ = value;
}
```

Rename the test to remove the obsolete “source is unchanged” claim. Require
both signatures to contain `mut pair`. Require `annotated_source` to contain
`let (_, mut value) = pair;` and `let _ = value;`; require the target to
contain `let (_, mut value) = todo!();` and `let _ = todo!();`. The two `_`
patterns remain wildcards in both presentations and never become bindings.

In the existing `all_nested_pattern_binding_kinds_become_mutable` test, retain
this complete input:

```rust
enum E { Pair(i32, i32), Struct { x: i32 }, Unit }
enum Choice { Left(i32), Right(i32) }
pub unsafe fn f(
    pair: (i32, i32),
    mut opt: Option<(i32, i32)>,
    values: [(i32, i32); 1],
    value: E,
    choice: Choice,
) {
    let ref borrowed = pair;
    let _ = borrowed;
    let whole @ (a, b) = pair;
    let Some((c, d)): Option<(i32, i32)> = opt else { return; };
    if let Some((e, f)) = opt { let _ = e + f; }
    while let Some((g, h)) = opt { opt = None; let _ = g + h; }
    for (i, j) in values { let _ = i + j; }
    match value {
        E::Pair(k, l) => { let _ = k + l; }
        E::Struct { x: m } => { let _ = m; }
        E::Unit => {}
    }
    match choice {
        Choice::Left(n) | Choice::Right(n) => { let _ = n; }
    }
    let _ = (whole, a, b, c, d);
}
```

Rename the test to state that non-`ref` bindings are normalized and reference
binding modes are preserved. In both `annotated_source` and
`annotated_skeleton`, require `mut` on all five by-value parameters and these
exact binding fragments:

```text
let ref borrowed
let mut whole @ (mut a, mut b)
Some((mut c, mut d))
Some((mut e, mut f))
Some((mut g, mut h))
for (mut i, mut j)
E::Pair(mut k, mut l)
x: mut m
Choice::Left(mut n) | Choice::Right(mut n)
```

Explicitly assert that neither rendering contains `let ref mut borrowed`.
Retain the existing independent auxiliary input:

```rust
pub unsafe fn ref_mut_source(mut value: i32) {
    let ref mut borrowed = value;
    let _ = borrowed;
}
```

and require `let ref mut borrowed` in both renderings. Thus a source `ref`
stays `ref`, a source `ref mut` stays `ref mut`, and only by-value identifier
bindings are forced to `mut`.

In the existing `safe_source_functions_get_unsafe_target_headers` test, retain
this exact input:

```rust
pub fn safe(input: i32) -> i32 { let value = input; value }
pub unsafe fn already_unsafe(input: i32) -> i32 { input }
```

Require the safe function's exact source signature to be
`pub fn safe(mut input: i32) -> i32` and its exact target signature to be
`pub unsafe fn safe(mut input: i32) -> i32`. Its annotated source remains
safe but presents both `input` and `value` as `mut`; its target becomes unsafe
and presents the same bindings as `mut`. Require both signatures of
`already_unsafe` to remain unsafe and present `mut input`.

Update the exact source-signature assertions in these two existing tests as
mechanical consequences of the same rule; do not add replacement tests:

- `function_records_sanitize_prompt_only_header_tokens_and_split_signatures`
  keeps its exact input
  `#[no_mangle] pub unsafe extern "system" fn add(x: i32, y: i32) -> i32 { x + y }`
  and now expects
  `pub unsafe fn add(mut x: i32, mut y: i32) -> i32` for both signatures.
- `keeps_non_pointer_signature_types_unchanged` keeps its exact input
  `struct S { x: i32 } pub unsafe fn f(a: i32, b: (u8, bool), c: [i16; 2], s: S) -> (S, usize) { (s, a as usize + b.0 as usize + c[0] as usize) }`
  and now expects its source signature, like its target signature, to present
  `mut a`, `mut b`, `mut c`, and `mut s`, without changing any type.

Update the existing skeleton test helper to apply the shared presentation
binding normalizer, including the new preserve-`ref` behavior. Retain the
existing independent parse check for both annotated snippets. No test in this
section creates a Cargo project, traverses a project directory, invokes the
Crat CLI, or tests Expand's file cleanup.

## 5. Skeleton-loading tests

### P4-LOAD-01 `valid_records_load_without_reordering_or_text_changes`

Exact input is:

```python
json.dumps(CONTEXT_RECORDS, ensure_ascii=False)
```

Load it. Assert record order is the exact listed order, dependencies retain
their listed order, and every Rust text field is byte-for-byte equal to the
decoded fixture value. Re-serializing is not required; Phase 4 owns no
skeleton JSON format.

### P4-LOAD-02 `top_level_and_kind_shapes_fail_clearly`

Run these exact JSON strings independently:

```text
[
```

```json
{}
```

```json
[{"id": 0, "path": "f", "kind": "Module"}]
```

```json
[{"id": 0, "path": "f", "kind": "Fn", "name": "f"}]
```

The truncated text reports a JSON decode failure. The object reports that the
top level must be an array. The `Module` record reports its unknown kind. The
incomplete `Fn` record reports all missing required fields in a deterministic
message. No graph is returned.

### P4-LOAD-03 `ids_and_same_namespace_paths_are_valid_and_unique`

Start from:

```python
[fn_record(0, "a::f", "f", []), fn_record(1, "b::g", "g", [])]
```

Run these exact independent mutations:

```text
first id = true
first id = -1
second id = 0
second path = "a::f"
first path = ""
```

Each input fails loading. Diagnostics distinguish invalid IDs, duplicate ID
0, duplicate value-namespace path `a::f`, and empty path. Boolean `true` is
not accepted as integer 1 or 0.

### P4-LOAD-04 `dependency_lists_must_be_sorted_unique_and_resolved`

Base input:

```python
[
    fn_record(0, "f", "f", [1, 2], [1]),
    type_record(1, "A", "Struct", "struct A;", []),
    type_record(2, "B", "Struct", "struct B;", []),
]
```

Run these exact independent changes:

```text
f.dependencies = [2, 1]
f.dependencies = [1, 1]
f.dependencies = [1, true]
f.dependencies = [1, 99]
```

Reject each input without sorting or deduplicating it. Identify the field,
record ID, and offending value/order.

### P4-LOAD-05 `signature_dependencies_must_be_dependency_subset`

Exact input:

```python
[
    fn_record(0, "f", "f", [], [1]),
    type_record(1, "A", "Struct", "struct A;", []),
]
```

Reject it because item 1 is in `signature_dependencies` but not
`dependencies`. Run these exact inputs independently and require the same
rule:

```python
[
    value_record(0, "S", "Static", "static S: A;", [], [1]),
    type_record(1, "A", "Struct", "struct A;", []),
]
```

```python
[
    value_record(0, "C", "Const", "const C: A;", [], [1]),
    type_record(1, "A", "Struct", "struct A;", []),
]
```

### P4-LOAD-06 `kind_specific_fields_are_not_interchangeable`

Run these exact records independently as one-element arrays:

```json
{
  "id": 0,
  "path": "S",
  "kind": "Struct",
  "declaration": "struct S;",
  "dependencies": []
}
```

```json
{
  "id": 0,
  "path": "N",
  "kind": "Const",
  "definition": "const N: usize = 1;",
  "signature_dependencies": [],
  "dependencies": []
}
```

```json
{
  "id": 0,
  "path": "f",
  "kind": "Fn",
  "name": "f",
  "annotated_source": 4,
  "annotated_skeleton": "unsafe fn f() {}",
  "source_signature": "unsafe fn f()",
  "target_signature": "unsafe fn f()",
  "signature_dependencies": [],
  "dependencies": []
}
```

Require `definition` for the struct, `declaration` for the const, and a string
for the function's `annotated_source`. Do not attempt Rust validation.

### P4-LOAD-07 `same_display_path_across_rust_namespaces_is_accepted`

Exact input, matching the Phase 1 namespace regression:

```python
[
    type_record(0, "X", "TyAlias", "type X = i32;", []),
    value_record(1, "X", "Const", "const X: i32;", [], []),
    fn_record(2, "f", "f", [0, 1], [0]),
]
```

Loading succeeds. IDs 0 and 1 both retain path `"X"`; the dependency IDs,
not path text, distinguish them. Independently replace item 1 with:

```python
fn_record(1, "X", "X", [])
```

Loading still succeeds because it remains one type-namespace record and one
value-namespace record. By contrast, the two-`Fn` duplicate-path mutation in
P4-LOAD-03 remains invalid.

### P4-LOAD-08 `ids_accept_exact_u64_range_and_reject_overflow`

Run these exact one-record inputs independently:

```python
[fn_record(18446744073709551615, "max", "max", [])]
```

```python
[fn_record(18446744073709551616, "overflow", "overflow", [])]
```

The first loads and preserves its exact ID. The second fails with an ID
out-of-range diagnostic. Together with P4-LOAD-03's `-1` and boolean cases,
this fixes the accepted ID domain to the inclusive Rust `u64` range.

## 6. Function-graph, SCC, and scheduling tests

### P4-GRAPH-01 `graph_uses_only_function_valued_dependencies`

Exact input is `GRAPH_RECORDS`. Assert the exact graph in Section 3.2.
Specifically, item 100 is absent as a node and edge target.

### P4-GRAPH-02 `direct_self_edges_are_retained`

Exact input:

```python
[fn_record(5, "self_rec", "self_rec", [5])]
```

Expected graph is `5 -> {5}`. Its one-member SCC is classified recursive.

### P4-GRAPH-03 `nonrecursive_singleton_has_no_synthetic_self_edge`

Exact input:

```python
[fn_record(5, "plain", "plain", [])]
```

Expected graph is `5 -> {}`. Its SCC is nonrecursive. Do not infer recursion
from its singleton size.

### P4-SCC-01 `tarjan_partition_matches_nested_cycles_and_chains`

Exact input graph is the graph in Section 3.2. Expected SCC member tuples,
shown sorted by minimum member ID, are:

```text
(0,)
(1,)
(2,)
(3, 4)
(5,)
(6,)
(7,)
(8, 9)
```

The algorithm may discover them internally in another order, but its public
normalized result must use increasing member IDs and deterministic SCC order.

### P4-SCC-02 `leaf_schedule_is_exact_for_graph_fixture`

Exact input is `GRAPH_RECORDS`. Ignoring the duplicate-name check, the exact
leaf-first schedule is:

```text
(0,), (1,), (2,), (3, 4), (5,), (6,), (7,), (8, 9)
```

After each selected SCC, assert that it had no outgoing edge to an
unprocessed SCC at that moment.

### P4-SCC-03 `schedule_is_independent_of_record_and_dependency_order`

Use these two exact logical inputs:

```python
first = GRAPH_RECORDS
second = list(reversed(GRAPH_RECORDS))
```

For `second`, also reverse each function's dependency list before constructing
the already-loaded record model. Graph construction must sort traversal
inputs, and both schedules must equal P4-SCC-02. This bypasses loader ordering
checks deliberately to test graph determinism in isolation.

### P4-SCC-04 `duplicate_names_abort_only_inside_one_scc`

Exact input:

```python
[
    fn_record(8, "same_a::same", "same", [9]),
    fn_record(9, "same_b::same", "same", [8]),
]
```

Immediately before the first LLM request, abort with both IDs and paths in
member-ID order. Exercise the SCC-processing loop after the normalized initial
project build has already succeeded. Assert zero LLM, validator, replacement,
and SCC-candidate Cargo calls; the one initialization build is outside this
unit's call counter.

### P4-SCC-05 `duplicate_names_in_distinct_sccs_are_allowed`

Exact input:

```python
[
    fn_record(1, "a::same", "same", []),
    fn_record(2, "b::same", "same", [1]),
]
```

Expected SCC schedule is `(1,), (2,)`; both SCCs pass the uniqueness check.
Do not perform a global name-uniqueness check.

## 7. Dependency-context and budget tests

### P4-CTX-01 `nonrecursive_singleton_omits_own_signature`

Exact input is target SCC `(0,)` over `CONTEXT_RECORDS`. The selected mandatory
IDs are exactly `[1, 2, 20]`. Item 0 has no dependency-context entry, although
its source and skeleton occur in Transformation Targets.

### P4-CTX-02 `direct_recursive_singleton_includes_own_signatures_once`

Exact records:

```python
[
    {
        **fn_record(0, "f", "f", [0, 1]),
        "source_signature": "unsafe fn f(mut p: *const i32)",
        "target_signature": "unsafe fn f(mut p: &i32)",
    },
    type_record(1, "T", "Struct", "struct T;", []),
]
```

For SCC `(0,)`, context entries are IDs `[0, 1]` in that order. Item 0 occurs
once, as mandatory SCC signatures, not again as a direct dependency.

### P4-CTX-03 `multi_function_scc_includes_every_member_signature_once`

Exact records:

```python
[
    fn_record(0, "m::a", "a", [1, 2]),
    fn_record(1, "m::b", "b", [0, 3]),
    type_record(2, "A", "Struct", "struct A;", []),
    value_record(3, "N", "Const", "const N: usize;", [], []),
]
```

For SCC `(0, 1)`, mandatory entries are IDs `[0, 1, 2, 3]`. Member functions
0 and 1 each render exactly once. Direct outside-SCC records 2 and 3 follow in
item-ID order.

### P4-CTX-04 `direct_function_static_and_type_entries_render_by_kind`

Exact input is SCC `(0,)` with `CONTEXT_RECORDS` and a limit larger than the
complete context. Assert direct entries 1, 2, and 20 use respectively:

- both exact function signatures;
- the exact static declaration only; and
- the exact struct definition only.

Do not include function bodies, annotated skeletons, or initializers.

### P4-CTX-05 `closure_follows_signature_edges_but_not_body_only_edges`

Exact input is SCC `(0,)` with `CONTEXT_RECORDS`. The breadth-first depths must
equal Section 3.3. IDs 22 and 24 never appear. This proves Python follows
record fields rather than treating every dependency list transitively.

### P4-CTX-06 `closure_deduplicates_at_shortest_union_depth`

Exact input is `CONTEXT_RECORDS`. Item 26 is reached through both item 21 and
item 25 at depth 2, and item 28 through items 23 and 25 at depth 2. Assert each
renders once at depth 2. Item 29 remains depth 3.

### P4-CTX-07 `entries_are_finally_sorted_by_item_id_not_discovery_parent`

Exact input:

```python
[
    fn_record(0, "f", "f", [10, 30]),
    type_record(10, "A", "Struct", "struct A;", [40]),
    type_record(20, "B", "Struct", "struct B;", []),
    type_record(30, "C", "Struct", "struct C;", [20]),
    type_record(40, "D", "Struct", "struct D;", []),
]
```

Depth zero is `[10, 30]`, depth one is `[20, 40]`, and final rendering order is
`[10, 20, 30, 40]`, not breadth-first discovery order `[10, 30, 20, 40]`.

### P4-CTX-08 `empty_context_is_exact_empty_string`

Exact input:

```python
[fn_record(0, "f", "f", [])]
```

For nonrecursive SCC `(0,)`, selected IDs are empty and rendered context is
exactly `""`, with no newline or placeholder text.

### P4-CTX-09 `rendering_matches_exact_golden`

Exact input is item 1 from `CONTEXT_RECORDS`. Its complete rendered entry is:

````text
### Function 1: callee
Source signature:
```rust
unsafe fn callee(mut p: *const T) -> i32
```
Target signature:
```rust
unsafe fn callee(mut p: &T) -> i32
```
````

Assert byte equality. Independently render items 2 and 20 and assert the exact
kind heading, Rust fence, source text, and absence of a trailing newline.
Joining `[1, 2, 20]` inserts exactly `"\n\n"` between entries.

### P4-BUDGET-01 `exact_limit_is_accepted`

Exact input is SCC `(0,)` with `CONTEXT_RECORDS`, but stop the candidate
closure after mandatory IDs `[1, 2, 20]`. Let `mandatory` be their exact
Section 9 rendering and set:

```python
limit = len(mandatory)
```

Context construction succeeds and returns exactly `mandatory`.

### P4-BUDGET-02 `whole_depth_is_rejected_without_partial_entries`

Exact input is SCC `(0,)` with `CONTEXT_RECORDS`. Let:

```python
mandatory = render([1, 2, 20])
with_depth_1 = render([1, 2, 20, 21, 23, 25])
limit = len(with_depth_1) - 1
```

The returned context is exactly `mandatory`. None of IDs 21, 23, or 25
appears, even if one entry alone would fit. Do not consider depths 2 or 3.

### P4-BUDGET-03 `mandatory_overflow_aborts_before_llm`

Exact input is SCC `(0,)` with `CONTEXT_RECORDS` and:

```python
limit = len(render([1, 2, 20])) - 1
```

Abort with SCC ID 0, actual mandatory character count, and limit. Assert zero
LLM calls and no validator/replacement/build attempt for the SCC.

## 8. Prompt and response-extraction tests

### P4-PROMPT-01 `transformation_targets_render_in_member_id_order`

Exact SCC members are supplied in order `(3, 0)` from `CONTEXT_RECORDS`.
Render them as normalized SCC `(0, 3)`. Expected headings are:

```text
### Function 0: target
### Function 3: peer
```

Each entry contains its exact `annotated_source` and `annotated_skeleton`,
joined by exactly two newlines. It contains no standalone source/target
signature section.

### P4-PROMPT-02 `initial_prompt_uses_versioned_template_and_empty_repair`

Exact logical renderer input:

```python
PromptRenderInput(
    dependency_context="DEPENDENCY\n",
    transformation_targets="TARGETS\n",
    failed_transformation=None,
    diagnostics=None,
)
```

Load the stage-local prompt through `PromptLibrary`. The stage must load prompt
ID `local_transformation`, version `1`, description
`Transform one Rust function SCC against Crat skeletons.`, and exact declared
variable tuple
`("dependency_context", "transformation_targets", "repair_context")`. Render
its declared variables as:

```python
{
    "dependency_context": "DEPENDENCY\n",
    "transformation_targets": "TARGETS\n",
    "repair_context": "",
}
```

Assert the rendered text contains the complete normative instructions once,
`DEPENDENCY\n` once, `TARGETS\n` once, and no
`"The previous transformation failed."`. Assert metadata contains prompt ID
`local_transformation`, version `1`, and the SHA-256 of the complete rendered
text.

For run `"phase4-run"`, stage `"local_transformation"`, and SCC members
`(0, 3)`, assert the exact logical request is:

```python
Request(
    messages=(Message(role="user", content=rendered.text),),
    system=None,
    model=None,
    max_tokens=None,
    temperature=None,
    metadata=RequestMetadata(
        run_id="phase4-run",
        stage="local_transformation",
        item="0,3",
        prompt_id="local_transformation",
        prompt_version=1,
        prompt_hash=rendered.content_hash,
    ),
)
```

### P4-PROMPT-03 `repair_prompt_contains_only_latest_failure`

First failure:

```text
failed text = "first bad code"
diagnostics = "first diagnostics"
```

Second failure:

```text
failed text = "second bad code"
diagnostics = "second diagnostics"
```

Render the second repair. It contains the complete original dependency and
target sections plus the second pair exactly once. It contains neither first
string. This proves no conversation history or accumulated diagnostics.

### P4-PROMPT-04 `initial_prompt_matches_complete_normative_golden`

Use the exact `PromptRenderInput` from P4-PROMPT-02. Construct the expected
string from the complete normative prompt body printed after the frontmatter
in [phase-4-plan.md](phase-4-plan.md#10-initial-llm-prompt-template) Section 10
by replacing:

```text
{{ dependency_context }} -> DEPENDENCY\n
{{ transformation_targets }} -> TARGETS\n
{{ repair_context }} -> empty string
```

No other byte, including blank lines, indentation, punctuation, or
final-newline state, may differ from the normative Section 10 text. Assert:

```python
rendered.text == expected
rendered.content_hash == hashlib.sha256(expected.encode("utf-8")).hexdigest()
```

This is a complete byte-equality golden assertion, not a substring or
instruction-presence test. The exact expected input text is the normative
template in the implementation plan, so the two hand-over documents cannot
silently drift.

### P4-EXTRACT-01 `single_fence_ignores_surrounding_prose_and_preserves_interior`

Exact response:

````text
Here is the result.

```rust
unsafe fn f() {
    #[proctor(0)]
    ()
}
```

I hope this helps.
````

Expected selected text is exactly:

```text
unsafe fn f() {
    #[proctor(0)]
    ()
}
```

There is no leading or trailing newline in the selected interior, and internal
indentation is unchanged.

Run this exact response independently:

````text
```rust
first

```
````

Expected selected text is exactly `"first\n"`. Extraction removes only the
single structural newline immediately before the closing fence; it does not
apply `.strip()` or remove the content's additional final newline.

Run this exact response independently:

```python
"```rust\r\nfirst\r\nsecond\r\n```"
```

Expected selected text is exactly `"first\r\nsecond"`. The two structural
CRLF sequences are removed, while the interior CRLF is preserved.

### P4-EXTRACT-02 `longest_block_wins_and_first_breaks_tie`

Run these exact responses independently:

````text
```rust
short
```
```rust
this block is longer
```
````

Expected is `"this block is longer"`.

````text
```rust
first
```
```text
later
```
````

Both interiors have length five; expected is `"first"`. Language tags do not
contribute to length.

Run this exact response independently:

``````text
~~~~rust
tilde
~~~~
````rust
four
````
```rust
triple
```
``````

Only the exact triple-backtick block is recognized, so the expected result is
`"triple"`. Tilde fences and four-or-more-backtick fences are not candidates.

### P4-EXTRACT-03 `missing_fence_is_repairable_with_raw_failed_text`

Exact response:

```text
I cannot provide that transformation.
```

Extraction returns no candidate, failed text equal to the complete response,
and exact diagnostic:

```text
The LLM response contained no triple-backtick fenced code block; return exactly one triple-backtick fenced Rust code block.
```

It does not call the validator on the raw prose.

Run these exact responses independently and require the same no-candidate
result and diagnostic, with each complete response preserved as failed text:

````text
~~~~rust
not a triple-backtick block
~~~~
````

`````text
````rust
not an exact triple-backtick block
````
`````

````text
prefix ```rust
inline opening
```
````

````text
  ```rust
indented opening
```
````

````text
```rust extra
tag contains whitespace
```
````

````text
```rust
closing has trailing space
``` 
````

````text
```rust
unclosed
````

### P4-PROTO-01 `validation_request_is_exact_and_member_ordered`

Exact synthetic SCC member input is `(3, 0)` from `CONTEXT_RECORDS`,
normalized to member order `(0, 3)`. Exact transformation is:

```rust
unsafe fn target(p: &S) -> i32 {
    #[proctor(0)]
    p.value as i32
}
unsafe fn peer() {
    #[proctor(0)]
    ()
}
```

Expected request:

```json
{
  "schema_version": 1,
  "expected_functions": [
    {
      "id": 0,
      "name": "target",
      "skeleton": "unsafe fn target(mut p: &S) -> i32 {\n    #[proctor(0)]\n    todo!()\n}"
    },
    {
      "id": 3,
      "name": "peer",
      "skeleton": "unsafe fn peer() {\n    #[proctor(0)]\n    todo!()\n}"
    }
  ],
  "transformation": "unsafe fn target(p: &S) -> i32 {\n    #[proctor(0)]\n    p.value as i32\n}\nunsafe fn peer() {\n    #[proctor(0)]\n    ()\n}"
}
```

Assert exact object equality and member-ID order. Python copies skeleton text
unchanged and does not split or inspect the transformation.

### P4-PROTO-02 `replacement_request_is_exact_and_member_ordered`

Use the same normalized synthetic SCC `(0, 3)` and transformation as
P4-PROTO-01.
Expected request:

```json
{
  "schema_version": 1,
  "items": [
    {"id": 0, "path": "target", "name": "target"},
    {"id": 3, "path": "peer", "name": "peer"}
  ],
  "transformation": "unsafe fn target(p: &S) -> i32 {\n    #[proctor(0)]\n    p.value as i32\n}\nunsafe fn peer() {\n    #[proctor(0)]\n    ()\n}"
}
```

Assert exact object equality. Phase 4 does not add wrapper metadata, source
signatures, or project paths to this request.

### P4-CMD-01 `crat_tool_argv_is_exact_for_all_four_operations`

Use these exact resolved paths:

```text
crat_tool = /tools/crat-tool
current_project = /work/current
library_source = /work/current/lib.rs
skeleton_output = /work/skeletons.json
normalized_output = /work/normalized.rs
validation_request = /work/validation-request.json
validation_response = /work/validation-response.json
replacement_request = /work/replacement-request.json
candidate_output = /work/candidate.rs
```

Call each command-assembly helper and assert exact argument-vector equality:

```python
[
    "/tools/crat-tool",
    "make-skeleton",
    "--output",
    "/work/skeletons.json",
    "/work/current",
]
```

```python
[
    "/tools/crat-tool",
    "normalize-safety",
    "--output",
    "/work/normalized.rs",
    "/work/current/lib.rs",
]
```

```python
[
    "/tools/crat-tool",
    "validate",
    "--input",
    "/work/validation-request.json",
    "--output",
    "/work/validation-response.json",
]
```

```python
[
    "/tools/crat-tool",
    "replace",
    "--request",
    "/work/replacement-request.json",
    "--output",
    "/work/candidate.rs",
    "/work/current",
]
```

Each is passed directly to the subprocess runner without shell joining.

### P4-CMD-02 `nonzero_build_preparation_or_crat_tool_exit_is_fatal`

Parameterize over these exact operations:

```text
deps_crate cargo build
release cargo build --bin crat
release cargo build --bin crat-tool
ordinary Crat prepare
crat-tool make-skeleton
crat-tool normalize-safety
crat-tool validate
crat-tool replace
```

For the selected operation, the fake runner returns:

```python
{
    "returncode": 7,
    "stdout": "partial stdout\n",
    "stderr": "tool failed\n",
}
```

Use the exact project tree from P4-FLOW-01 and exact records
`[fn_record(0, "target", "target", [])]`. Successful earlier build/tool
operations return `TOOL_OK`. Successful output-producing operations create
these exact files:

```text
skeletons.json = json.dumps([fn_record(0, "target", "target", [])])
normalized.rs = "unsafe fn target() {}\n"
validation-response.json = json.dumps(VALIDATOR_VALID)
candidate.rs = "unsafe fn target() {}\n"
```

For the validate and replace failure subcases, the preceding logical LLM
generation returns `VALID_RESPONSE_TEXT` with the exact normal fake response
metadata and usage from Section 3.4. For the replace failure subcase, the
preceding validator returns `VALIDATOR_VALID`.

No later operation is called. Every subcase aborts with operation name, exit
code 7, and both captured streams. A validation or replacement command
failure is not converted into an SCC repair.

### P4-CMD-03 `created_outputs_cannot_be_missing_nonregular_or_stale`

Create `work = tmp_path / "work"` and parameterize over these exact
output-producing operations and paths beneath it:

```text
make-skeleton    -> work / "skeletons.json"
normalize-safety -> work / "normalized.rs"
validate         -> work / "validation-response.json"
replace          -> work / "candidate.rs"
```

For each operation, first create its output path as a regular file containing
exact bytes `"stale\n"`. The fake runner asserts the path has been removed
before launch, returns code 0, and creates nothing. The operation fails for a
missing newly created regular output; it must not consume `"stale\n"`.

Run a second exact subcase in which the output again starts as `"stale\n"`;
after asserting its removal, the fake runner creates a directory at that
path and returns code 0. The operation fails because the new output is not a
regular file. Neither subcase parses a response, installs a source, or starts
an SCC repair.

## 9. Attempt and repair orchestration tests

### P4-FLOW-01 `preparation_and_initialization_event_order_is_exact`

Exact project input:

```text
input/Cargo.toml = "[package]\nname = \"p\"\nversion = \"0.1.0\"\nedition = \"2021\"\n\n[lib]\nname = \"p\"\npath = \"lib.rs\"\n\n[[bin]]\nname = \"p-bin\"\npath = \"driver.rs\"\n"
input/config.toml = ""
input/lib.rs = "pub struct S;\n"
input/driver.rs = "fn main() {}\n"
input/proctor.toml = "wrappers = [{ wrapped = \"a\", wrapper = \"b\" }]\n"
input/target/cache = "warm\n"
```

Queue a prepared skeleton `[]`, normalized source `"pub struct S;\n"`, and
`BUILD_OK`. The exact event prefix is:

```text
build_tools
prepare(current, ("expand", "unexpand"), True)
make_skeleton
normalize(current/lib.rs)
cargo_build(current)
```

Because the skeleton is empty, the stage then finalizes successfully without
an LLM, validator, or replacement call. Its exact count metrics are:

```text
function_count = 0
scc_count = 0
llm_generation_calls = 0
repair_calls = 0
structural_failures = 0
compilation_failures = 0
cargo_builds = 1
```

It reports `usage = null`, `models = []`, and `prompts = []`.
The final output preserves the exact nonempty `proctor.toml` without parsing
it and preserves `driver.rs`; its root `target/` is absent.

Use these exact resolved paths:

```text
crat = /tools/crat
current = /work/current
config = /work/current/config.toml
```

The exact preparation argument vector is:

```python
[
    "/tools/crat",
    "--inplace",
    "--config",
    "/work/current/config.toml",
    "--pass",
    "expand,unexpand",
    "--unexpand-use-print",
    "/work/current",
]
```

It includes neither `split` nor `bin`. This tests command construction only,
not Crat behavior.

Run an exact second subcase with the same project except that `config.toml` is
absent. Its exact vector is:

```python
[
    "/tools/crat",
    "--inplace",
    "--pass",
    "expand,unexpand",
    "--unexpand-use-print",
    "/work/current",
]
```

This uses Crat's defaults and otherwise has the same event order.
Every `cargo_build(current)` event in this plan means exact argv
`["cargo", "build"]` with `/work/current` as the working directory.

Run an exact third subcase from the first project: the fake preparation
returns `TOOL_OK` but removes `/work/current/lib.rs`. The stage aborts before
`make_skeleton` because the previously validated library path is no longer a
regular file.

### P4-FLOW-02 `normalized_initial_build_failure_aborts_without_llm`

Use the exact P4-FLOW-01 input, one-function skeleton
`[fn_record(0, "target", "target", [])]`, normalized source
`"unsafe fn target() {}\n"`, and `BUILD_FAIL` for the initial build.
The stage fails with both build streams. Assert zero LLM, validation, and
replacement calls and no output project.

### P4-FLOW-03 `valid_initial_generation_validates_replaces_and_builds_once`

Exact records:

```python
[fn_record(0, "target", "target", [])]
```

Queue `VALID_RESPONSE_TEXT`, `VALIDATOR_VALID`, candidate source
`"candidate-one\n"`, and `BUILD_OK`. After the already-successful normalized
initial build, the exact SCC event order is:

```text
LLM request for SCC "0"
validate
replace
cargo_build
```

The current library becomes `"candidate-one\n"`. Counts are one generation,
zero repairs, one validation, one replacement, and one SCC candidate build.

### P4-FLOW-04 `validator_invalid_consumes_one_repair_then_succeeds`

Use the P4-FLOW-03 records. Queue:

```text
LLM #1 -> INVALID_RESPONSE_TEXT
validator #1 -> VALIDATOR_INVALID
LLM #2 -> VALID_RESPONSE_TEXT
validator #2 -> VALIDATOR_VALID
replace -> "candidate-two\n"
build -> BUILD_OK
```

The fake validator writes the exact `VALIDATOR_INVALID_RAW` bytes and also
returns the parsed `VALIDATOR_INVALID` object. The second prompt includes
`VALIDATOR_INVALID_RAW` unchanged—not a Python reserialization—and the
selected transformation. Counts are two generation calls, one repair, two
validations, one replacement, and one candidate build.

### P4-FLOW-05 `missing_fence_consumes_repair_without_validator_call`

Use the P4-FLOW-03 records. Queue:

```text
LLM #1 -> "no code here"
LLM #2 -> VALID_RESPONSE_TEXT
validator -> VALIDATOR_VALID
replace -> "candidate\n"
build -> BUILD_OK
```

Assert one repair, one validator call total, and the second prompt contains
the exact P4-EXTRACT-03 raw text and diagnostic.

### P4-FLOW-06 `failed_candidate_build_restores_then_repairs`

Use the exact records `[fn_record(0, "target", "target", [])]`.
Initial current library is `"normalized-current\n"`. Queue:

```text
LLM #1 -> VALID_RESPONSE_TEXT
validator #1 -> VALIDATOR_VALID
replace #1 -> "bad-candidate\n"
build #1 -> BUILD_FAIL
LLM #2 -> VALID_RESPONSE_TEXT
validator #2 -> VALIDATOR_VALID
replace #2 -> "good-candidate\n"
build #2 -> BUILD_OK
```

Immediately before LLM #2 and replacement #2, the current library must again
equal `"normalized-current\n"`. The second repair diagnostics are exactly:

```text
cargo build stdout:
checking local_project

cargo build stderr:
error[E0308]: mismatched types
```

After success it equals `"good-candidate\n"`. `target/` is never copied or
removed during either attempt.

### P4-FLOW-07 `ten_failed_repairs_allow_exactly_eleven_generations`

Use the exact records `[fn_record(0, "target", "target", [])]`. Queue eleven
fenced responses, each followed by
`VALIDATOR_INVALID`; every response is exactly `INVALID_RESPONSE_TEXT`. The
initial call and repairs 1 through 10 occur. After the eleventh invalid
response, abort. Assert:

```text
LLM generations = 11
repairs = 10
validator calls = 11
replacement calls = 0
candidate builds = 0
```

The failure identifies SCC 0 and repair exhaustion. A twelfth response left
in the fake queue is untouched.

### P4-FLOW-08 `validator_setup_or_protocol_failure_aborts_without_repair`

Use the exact records `[fn_record(0, "target", "target", [])]` and exact LLM
response `VALID_RESPONSE_TEXT`. Run these exact independent validator
outcomes:

```python
VALIDATOR_SETUP_ERROR
```

```text
process return code = 2
response file absent
stderr = "validator crashed\n"
```

```json
{"schema_version": 1, "status": "mystery"}
```

```json
{"schema_version": 2, "status": "valid"}
```

```text
{not-json
```

Each aborts immediately after one LLM call. Assert zero repairs, replacements,
and candidate builds. Diagnostics distinguish setup error, process failure,
unknown status, unsupported response version, and malformed JSON protocol.

### P4-FLOW-09 `replacement_failure_is_not_sent_to_llm`

Use the exact records `[fn_record(0, "target", "target", [])]`. Queue
`VALID_RESPONSE_TEXT`, `VALIDATOR_VALID`, then:

```text
replace return code = 1
candidate file absent
stderr = "TargetResolution: missing current function target\n"
```

Abort after one LLM call with no repair and no source installation/build.

### P4-FLOW-10 `context_overflow_is_forced_to_error_and_aborts`

Use the exact P4-FLOW-01 project initialization with skeleton records
`[fn_record(0, "target", "target", [])]` and a successful normalized initial
build.
Exact framework LLM settings:

```python
{
    "provider": "replay",
    "model": "phase4-test",
    "context_overflow": "truncate_middle",
    "max_retries": 5,
}
```

Construct the stage client through an injectable factory that returns a real
`LlmClient` with a real temporary-file `UsageTracker` and a fake provider
whose `name == "replay"` and whose first `complete` raises:

```python
ContextLimitExceeded("too long", needed_tokens=12000, limit_tokens=10000)
```

Assert the factory received a new settings dictionary with
`context_overflow == "error"`, while the exact input dictionary remains
unchanged with `"truncate_middle"`. Abort the complete stage after the shared
tracker writes one record with attempt 1, provider `"replay"`, model
`"phase4-test"`, integer token fields `input_tokens`,
`cached_input_tokens`, and `output_tokens` equal to zero,
`reasoning_tokens is null`, finish reason `"error"`, and an error beginning
`"ContextLimitExceeded: too long"`. The failure `StageOutput.usage.calls` is
1; its input, cached-input, and output totals are zero, and its optional
reasoning-token total is null; `cost_usd == 0.0`. It reports the replay
provider/model and prompt ID/version once. Assert no SCC repair, validation,
replacement, or candidate build.

### P4-FLOW-11 `later_scc_uses_promoted_source_but_immutable_skeleton_prompt`

Exact records:

```python
[
    fn_record(0, "callee", "callee", []),
    fn_record(1, "caller", "caller", [0]),
]
```

Initial current source is `"normalized\n"`. Queue successful callee
and caller attempts as:

```text
LLM #1 -> "```rust\nunsafe fn callee() {\n    #[proctor(0)]\n    ()\n}\n```"
validator #1 -> VALIDATOR_VALID
replace #1 -> "after-callee\n"
build #1 -> BUILD_OK
LLM #2 -> "```rust\nunsafe fn caller() {\n    #[proctor(0)]\n    ()\n}\n```"
validator #2 -> VALIDATOR_VALID
replace #2 -> "after-caller\n"
build #2 -> BUILD_OK
```

Assert:

- schedule is `(0,), (1,)`;
- the caller's `replace` event receives the current project whose library is
  `"after-callee\n"`;
- the caller prompt still uses item 1's immutable original
  `annotated_source` and `annotated_skeleton`; and
- final source is `"after-caller\n"`.

Python maintains no transformed-function registry and never regenerates
skeletons after promotion.

## 10. Source transactions, artifacts, and stage-contract tests

### P4-TXN-01 `successful_candidate_keeps_source_and_deletes_rollback`

Exact temporary files:

```text
work/current/lib.rs = "old\n"
work/candidate.rs = "new\n"
work/rollback/ does not exist
```

Fake build returns `BUILD_OK`. Run the source transaction. Expected:

```text
work/current/lib.rs = "new\n"
work/candidate.rs no longer exists after atomic installation
no rollback file remains
```

The build observes `"new\n"` at `current/lib.rs`.

### P4-TXN-02 `failed_build_restores_source_but_retains_target_updates`

Exact initial tree:

```text
work/current/lib.rs = "old\n"
work/current/target/cache = "before\n"
work/candidate.rs = "bad\n"
```

During the fake failed build, assert `lib.rs == "bad\n"` and overwrite
`target/cache` with `"after-failure\n"`, then return `BUILD_FAIL`. After the
transaction:

```text
lib.rs = "old\n"
target/cache = "after-failure\n"
```

The candidate is not promoted, while Cargo cache changes persist.

### P4-TXN-03 `exception_after_installation_restores_in_finally`

Use the P4-TXN-02 source files, but make the builder raise:

```python
RuntimeError("builder transport failed")
```

The exception propagates after `lib.rs` is restored to `"old\n"`. No rollback
file remains.

### P4-TXN-04 `rollback_failure_is_fatal_and_not_repairable`

Install `"bad\n"` over `"old\n"`, return `BUILD_FAIL`, then inject:

```python
OSError("restore denied")
```

for the atomic restore operation. The transaction raises a fatal restoration
error containing both the build failure and `"restore denied"`. The SCC loop
does not issue another LLM request.

### P4-ART-01 `input_is_copied_once_and_never_mutated`

Exact input tree:

```text
input/Cargo.toml = "[package]\nname = \"p\"\nversion = \"0.1.0\"\nedition = \"2021\"\n\n[lib]\npath = \"lib.rs\"\n"
input/lib.rs = "pub unsafe fn target() {}\n"
input/proctor.toml = "wrappers = [{ wrapped = \"a\", wrapper = \"b\" }]\n"
input/target/cache = "warm\n"
```

Run the exact successful one-SCC flow from P4-FLOW-03 after the normal
P4-FLOW-01-style preparation, with its replacement candidate changed to
`"pub unsafe fn target() {}\n"`.
Assert all four input files retain exact bytes. Instrument project copying and
assert one input-to-current copy and one current-to-output copy; there is no
per-SCC project copy.

### P4-ART-02 `final_output_excludes_only_root_target`

Exact current tree before finalization:

```text
current/Cargo.toml = "cargo\n"
current/lib.rs = "final\n"
current/proctor.toml = "wrappers = [{ wrapped = \"a\", wrapper = \"b\" }]\n"
current/driver.rs = "fn main() {}\n"
current/target/cache = "large\n"
current/assets/target = "ordinary-file\n"
```

Finalize to an absent output directory. Expected output contains every exact
file except `current/target/cache`; `assets/target` remains because only the
root `target/` directory is excluded. `proctor.toml` and `driver.rs` are
byte-identical.

### P4-ART-03 `failure_creates_no_claimed_output`

Use the P4-ART-01 input and an absent output path. Use the exact
P4-FLOW-07 eleven-generation exhaustion.
Assert:

- the output path does not exist;
- `StageOutput.status == "failure"`;
- reported `outputs.rust_project` is null; and
- the private current project may remain only under `framework.workdir`.

### P4-ART-04 `existing_output_is_refused_before_work_or_tool_calls`

Exact paths:

```text
input/ is the P4-ART-01 input
output/existing.txt = "owned by caller\n"
work/ is empty
```

Run the stage with `outputs.rust_project = output`. It fails before building
tools or copying input, leaves `existing.txt` unchanged, and reports that the
destination already exists.

### P4-CONTRACT-01 `stage_manifest_declares_exact_artifacts_and_warmup`

Read the checked-in `stage.toml`. Assert:

```text
id = "local_transformation"
version = "0.1.0"
exec = ["python3", "main.py"]
warmup = ["python3", "main.py", "--build-only"]
requires = { rust_project = "required" }
produces = { rust_project = true }
no c_project, test_package, or rule_set requirement/product
config key set = {"crat_dir"}
crat_dir default = "../crat"
```

Also read the stage's `pyproject.toml` and require a `proctor` dependency whose
local source is exactly `../..`; require the checked-in `uv.lock` to exist.
This is a packaging/manifest contract test, not a toolchain test.

### P4-CONTRACT-02 `successful_output_reports_llm_reproducibility`

Use the one-SCC P4-FLOW-03 success and exact fake usage from Section 3.4.
Expected output has:

```text
status = success
outputs.rust_project = declared output path
outputs.rule_set = null
models = [{provider: "replay", model: "phase4-test"}]
usage.calls = 1
usage.input_tokens = 100
usage.cached_input_tokens = 20
usage.output_tokens = 30
usage.reasoning_tokens = 4
usage.cost_usd = 0.0
prompts = [{id: "local_transformation", version: 1}]
logs contains the stage command/diagnostic log
error = null
```

`config_used` contains only the effective `crat_dir`, resolved relative to the
stage directory using the same path-resolution convention as the existing
Crat adapter:

```python
{
    "crat_dir": str((Path(main_module.__file__).parent / "../crat").resolve())
}
```

For that run, set `outputs.artifacts_dir` to an existing
`artifacts/` directory and assert every reported log path is relative to it.
Run an exact second subcase with `outputs.artifacts_dir = null`: the diagnostic
log is retained under `framework.workdir`, but `StageOutput.logs == ()`
because no artifacts-directory-relative log can be reported.

Run an exact third subcase from `BASE_STAGE_INPUT` with
`framework.usage_log = null`. The tracker writes
`work/usage.jsonl`, and the successful output has the same exact usage,
model, and prompt summaries shown above.

Run an exact fourth subcase with the `"pricing"` key removed from the copied
LLM settings. The token totals are unchanged, but
`StageOutput.usage.cost_usd is null`; unknown pricing is not reported as free.

Metrics contain these exact entries:

```text
function_count = 1
scc_count = 1
llm_generation_calls = 1
repair_calls = 0
structural_failures = 0
compilation_failures = 0
cargo_builds = 2
```

The two Cargo builds are the normalized initial build and the candidate build.

### P4-CONTRACT-03 `failure_after_llm_still_reports_accumulated_usage`

Use one LLM response with the Section 3.4 usage, then
`VALIDATOR_SETUP_ERROR`. Expected failure output retains the same model,
prompt, and usage summary as P4-CONTRACT-02, has a nonempty error, reports no
Rust output, and returns process exit code 1.

### P4-USAGE-01 `provider_retry_success_counts_attempts_but_one_generation`

Use a real `LlmClient`, real temporary-file `UsageTracker`, no-op injected
sleep, and these exact settings:

```python
{
    "provider": "replay",
    "model": "phase4-test",
    "context_overflow": "error",
    "max_retries": 1,
}
```

The fake provider has `name = "replay"` and returns this exact queue:

```python
[
    ProviderError("temporary failure", status=503),
    Response(
        text=VALID_RESPONSE_TEXT,
        finish_reason="stop",
        provider="replay",
        model="phase4-test",
        latency_s=0.25,
        usage=Usage(
            input_tokens=100,
            cached_input_tokens=20,
            output_tokens=30,
            reasoning_tokens=4,
        ),
    ),
]
```

Complete the otherwise successful P4-FLOW-03. The usage JSONL has two records
in order: attempt 1 is an error with its three integer token counts zero and
`reasoning_tokens is null`; attempt 2 is the successful exact usage above.
Expected aggregate:

```text
usage.calls = 2
usage.input_tokens = 100
usage.cached_input_tokens = 20
usage.output_tokens = 30
usage.reasoning_tokens = 4
usage.cost_usd = 0.0
models = [{provider: "replay", model: "phase4-test"}]
metrics.llm_generation_calls = 1
metrics.repair_calls = 0
```

The stage must aggregate tracker attempts, not infer one usage call from the
single successful `Response`.

### P4-USAGE-02 `terminal_llm_errors_preserve_every_failed_attempt`

Use the same real-client/tracker setup and settings as P4-USAGE-01. Run these
exact independent provider queues:

```python
[
    ProviderError("upstream unavailable", status=503),
    ProviderError("still unavailable", status=503),
]
```

```python
[AuthError("bad key")]
```

The exhausted-provider subcase aborts after two provider attempts; the auth
subcase aborts after one and is not retried. Their failure outputs report,
respectively:

```text
usage.calls = 2
usage.input_tokens = 0
usage.cached_input_tokens = 0
usage.output_tokens = 0
usage.reasoning_tokens = null
usage.cost_usd = 0.0
metrics.llm_generation_calls = 1
metrics.repair_calls = 0
```

```text
usage.calls = 1
usage.input_tokens = 0
usage.cached_input_tokens = 0
usage.output_tokens = 0
usage.reasoning_tokens = null
usage.cost_usd = 0.0
metrics.llm_generation_calls = 1
metrics.repair_calls = 0
```

Each JSONL record has finish reason `"error"`, its one-based attempt number,
provider/model attribution, prompt metadata, zero integer token fields,
`reasoning_tokens is null`, and the exact exception class and message. Neither
error becomes an SCC structural/compiler repair, and no validator,
replacement, or candidate build occurs.
Both failure outputs report
`models = [{provider: "replay", model: "phase4-test"}]` and
`prompts = [{id: "local_transformation", version: 1}]`.

### P4-CONTRACT-04 `build_only_warms_both_crat_binaries_without_stage_io`

Invoke `main(["--build-only"])` with an injected build helper. Exact expected
build targets are:

```text
deps_crate default target
root release binary "crat"
root release binary "crat-tool"
```

It reads no stage envelope, creates no project copy, and prints the resolved
binary paths. A fake build failure returns nonzero and identifies the failed
target.

### P4-CONTRACT-05 `missing_required_paths_fail_before_side_effects`

Run these exact `StageInput` changes independently from `BASE_STAGE_INPUT`:

```text
inputs.rust_project = null
outputs.rust_project = null
framework.workdir = null
input Cargo.toml absent
input Cargo.toml = "[package]\nname = \"p\"\n"
input Cargo.toml = "[lib]\npath = 7\n"
input Cargo.toml = "[lib]\npath = \"src/lib.rs\"\n" with input/src/lib.rs present
input Cargo.toml = "[lib]\npath = \"src/../lib.rs\"\n" with input/src/ present
input Cargo.toml = "[lib]\npath = \"/outside/lib.rs\"\n"
input Cargo.toml = "[lib]\npath = \"../lib.rs\"\n"
config = {"crat_dir": ""}
config = {"crat_dir": 7}
config = {"crat_dir": "../crat", "unknown": true}
```

Each produces a failure envelope and zero build-tool, project-copy, LLM, or
Cargo events. Diagnostics identify the exact missing/invalid boundary rather
than leaking `AssertionError`, `KeyError`, or `TypeError`. The three library
path classes establish that only a root-level library source is supported and
that every `..` component is rejected even if it would return to the root.
Independently, this exact input succeeds past validation and resolves its
library path to `input/lib.rs`:

```text
input Cargo.toml = "[lib]\npath = \"./lib.rs\"\n"
input/lib.rs exists
```

Also independently, `config = {}` succeeds past validation and resolves the
manifest default `../crat`.

## 11. Completion criteria

Phase 4 is complete when:

- every case above is implemented and passing;
- no Python test duplicates Crat's Rust semantic assertions;
- graph, SCC, context, budget, response, repair, and transaction logic is
  exercised through pure or fake-driven tests;
- the stage uses PROCTOR's typed envelopes, shared LLM client, prompt library,
  usage tracker, and pricing;
- context overflow cannot select a truncation path;
- no failed attempt mutates the input artifact or leaks a claimed output;
- successful finalization preserves non-library files and excludes only the
  root `target/`;
- the amended existing in-memory Crat skeleton tests pass without adding a
  filesystem-changing Crat test; and
- `configs/full_pipeline.toml` remains unchanged.
