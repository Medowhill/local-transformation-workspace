# Amendment 3 Test Plan: Foreign-Function Guidance and Prompt Presentation

## 1. Purpose

This document specifies the complete focused regression suite for Amendment
Plan 3 in [amendment-3-plan.md](amendment-3-plan.md). It covers resolved
per-function foreign-name metadata, the final post-Amendment-2 skeleton-record
shape, strict Python loading, name-only context rendering, dependency
budgeting, and the final unreleased version-1 prompt presentation.

The historical phase and amendment test plans are not edited. Existing
implementation tests whose expected JSON fields, helper records, context
headings, character counts, prompt text, or prompt hashes conflict with
Amendment 3 are updated as specified here. No validator/replacer schema or
prompt version is incremented before the first release.

The suite contains 12 named cases:

| Area | Cases |
| --- | ---: |
| Existing regression updates | 2 |
| Crat foreign-reference collection | 2 |
| Python loading | 1 |
| Context and target rendering | 4 |
| Prompt and unchanged protocols | 3 |

## 2. Test execution and ownership policy

Rust semantic and JSON tests live beside the existing Crat skeleton tests and
run with:

```bash
cd proctor/stages/crat
cargo test --workspace
```

They use exact source strings with `run_compiler_on_str`; they create no
fixture files and invoke no nested Cargo process.

Python loading, rendering, budget, prompt, and protocol tests live in the
existing local-transformation test module and run with:

```bash
cd proctor
uv run pytest tests/test_local_transformation.py
```

They use immutable fake records originating from the exact Rust sources in
Section 3. They remain offline and do not invoke rustc, Cargo, Crat, or an LLM
provider.

After implementation, run the ordinary repository checks required for changed
Crat and PROCTOR code. No test edits `phase-1-test-plan.md`,
`phase-2-test-plan.md`, `phase-3-test-plan.md`, `phase-4-test-plan.md`, or
`amendment-2-test-plan.md`. No case changes a checked-in pipeline
configuration.

Existing implementation fixtures are updated by this explicit matrix:

- Every Rust and Python `Fn` record fixture adds required
  `foreign_function_names`; ordinary helpers default it to `[]`.
- Existing non-function records remain byte-for-byte unchanged except where an
  enclosing full-output golden necessarily changes because a function record
  gained the new key.
- Existing dependency and target rendering goldens remove item IDs and use the
  Amendment 3 final names and blank-line hierarchy. Duplicate dependency
  headings are accepted when records have the same kind and final name.
- Existing dependency-budget tests compute limits from the amended rendered
  entries. The section heading remains outside the entry budget.
- The complete prompt golden retains Amendment 2's paragraph, adds the one
  Amendment 3 advisory requirement, and replaces the two plain section labels
  with the exact conditional `##` hierarchy.
- Existing graph, scheduling, validation, replacement, repair, transaction,
  and metrics expectations otherwise remain unchanged.

## 3. Comparison policy and exact shared Rust inputs

JSON key order, list order, Markdown text, blank lines, prompt content hashes,
and character counts are exact where stated. Rust semantic collection uses
resolved HIR identities; Python never parses the Rust source.

When a case references a shared input below, it uses that exact source
byte-for-byte. Every named case references at least one exact shared Rust
input.

### A3-SRC-PLAIN

```rust
pub unsafe fn scalar(value: i32) -> i32 {
    value + 1
}
```

### A3-SRC-FOREIGN

```rust
#![feature(extern_types)]

unsafe extern "C" {
    fn strlen(text: *const core::ffi::c_char) -> usize;
    fn free(pointer: *mut core::ffi::c_void);
    fn transitive_foreign(value: i32) -> i32;
    fn unused_foreign(value: i32) -> i32;
    static FOREIGN_COUNTER: i32;
    type ForeignOpaque;
}

use strlen as c_strlen;

pub unsafe extern "C" fn local_abi(value: i32) -> i32 {
    transitive_foreign(value)
}

pub mod parser {
    pub unsafe fn scan(
        pointer: *mut core::ffi::c_void,
        text: *const core::ffi::c_char,
    ) -> usize {
        crate::free(pointer);
        let first = crate::c_strlen(text);
        let second = crate::strlen(text);
        let _ = crate::FOREIGN_COUNTER;
        let _: Option<*mut crate::ForeignOpaque> = None;
        let _ = crate::local_abi(first as i32);
        first + second + core::mem::size_of::<usize>()
    }

    pub unsafe fn release(pointer: *mut core::ffi::c_void) {
        crate::free(pointer);
        crate::free(pointer);
    }

    pub unsafe fn scalar(value: i32) -> i32 {
        crate::local_abi(value)
    }
}
```

### A3-SRC-DEPENDENCY-FOREIGN

```rust
extern crate libc;

pub unsafe fn dependency_free(pointer: *mut libc::c_void) {
    libc::free(pointer);
}
```

### A3-SRC-CANONICAL-NAME

```rust
unsafe extern "C" {
    #[link_name = "strlen"]
    fn c_strlen(text: *const core::ffi::c_char) -> usize;
}

pub unsafe fn length(text: *const core::ffi::c_char) -> usize {
    c_strlen(text)
}

pub unsafe fn hold_callable() {
    let callable = c_strlen;
    let _ = callable;
}
```

`hold_callable` is a focused robustness case outside the prototype's supported
no-function-pointer model.

### A3-SRC-KINDS

```rust
pub struct Point {
    pub value: i32,
}

pub static LIMIT: i32 = 10;
pub const STEP: i32 = 1;

pub unsafe fn helper(point: Point) -> i32 {
    point.value
}

pub unsafe fn target(point: Point) -> i32 {
    helper(point) + LIMIT + STEP
}
```

### A3-SRC-COLLISION

```rust
pub mod outer {
    pub mod left {
        pub unsafe fn parse(value: i32) -> i32 {
            value + 1
        }
    }

    pub mod right {
        pub unsafe fn parse(value: i32) -> i32 {
            value - 1
        }
    }
}

pub unsafe fn target(value: i32) -> i32 {
    outer::left::parse(value) + outer::right::parse(value)
}
```

## 4. Existing regression updates

### A3-UPDATE-01 `existing_function_records_and_helpers_adopt_the_final_shape`

Use exact inputs A3-SRC-PLAIN and A3-SRC-FOREIGN.

Update the existing Rust record-field, round-trip, deterministic-serialization,
and inline-JSON goldens. Update every Python function-record constructor and
literal used by loading, graph, context, prompt, and orchestration tests.
Ordinary helper calls default to:

```python
"foreign_function_names": []
```

The exact ordered keys of every final `Fn` record are:

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
foreign_function_names
signature_dependencies
dependencies
```

For A3-SRC-PLAIN, `foreign_function_names` is JSON `[]`. The field is absent
from all non-function records. Preserve Amendment 2's Boolean and label-array
values and invariants unchanged.

### A3-UPDATE-02 `existing_context_budget_and_prompt_goldens_use_final_presentation`

Use exact inputs A3-SRC-KINDS, A3-SRC-COLLISION, and A3-SRC-FOREIGN.

Update every existing implementation assertion that expects headings such as:

```text
### Function 1: callee
### Struct 20: Point
### Function 7: parser::scan
```

No amended ordinary context or target heading contains a skeleton item ID.
Ordinary noncolliding dependency headings use final names, target headings use
final function names, and colliding dependency headings deliberately repeat
the same final name without a path. Recompute exact dependency-entry lengths
after these text changes.

Update the complete version-1 prompt byte golden and content hash to the final
Amendment 3 text. The existing Amendment 2 paragraph still occurs exactly
once. Do not alter raw repair-diagnostic fixtures merely because their
validator JSON may contain an internal item ID.

## 5. Crat foreign-reference collection tests

### A3-FOREIGN-01 `collects_per_function_names_sorted_deduplicated_and_resolved`

Use exact input A3-SRC-FOREIGN.

Expected lists are:

```text
local_abi       ["transitive_foreign"]
parser::scan    ["free", "strlen"]
parser::release ["free"]
parser::scalar  []
```

`scan` proves all of the following in one compiling fixture:

- `free` sorts before `strlen`;
- the direct `strlen` reference and the `c_strlen` alias resolve to the same
  declaration identifier and deduplicate;
- the foreign static `FOREIGN_COUNTER` is excluded;
- the referenced foreign type `ForeignOpaque` is excluded;
- the source-defined `extern "C"` function `local_abi` is excluded;
- the non-foreign external function `core::mem::size_of` is excluded; and
- the declared but unreferenced `unused_foreign` is excluded.

`release` proves repeated calls deduplicate within one function, and
`local_abi` proves its own direct foreign reference is collected.
`parser::scalar` calls `local_abi` but remains empty, proving collection does
not traverse a local callee's body.

Ordinary dependency IDs and SCC edges remain exactly those produced without
the new metadata: foreign calls add neither.

Do not derive any expected name from rendered text or from the Amendment 2
transformation-label array.

### A3-FOREIGN-02 `handles_link_name_callable_reference_and_dependency_metadata`

Use exact inputs A3-SRC-CANONICAL-NAME and A3-SRC-DEPENDENCY-FOREIGN.

Expected:

```text
length        ["c_strlen"]
hold_callable ["c_strlen"]
dependency_free ["free"]
```

The external symbol string `"strlen"` is not emitted. The callable reference
in `hold_callable` is included even though it is not a direct call. Both lists
come from the resolved foreign `DefId` and declaration identifier. The
`dependency_free` list proves the same rule applies when the foreign
declaration is loaded from the existing `libc` dependency metadata rather than
declared in the source fixture. The existing Crat compiler-test harness already
provides that dependency; this case adds no nested Cargo invocation.

## 6. Python loading, context, and target-rendering tests

### A3-LOAD-01 `loader_requires_sorted_unique_nonempty_foreign_names`

Use function records originating from exact inputs A3-SRC-PLAIN and
A3-SRC-FOREIGN.

Accept and preserve:

```json
{"foreign_function_names": []}
```

and:

```json
{"foreign_function_names": ["free", "strlen"]}
```

Load the latter as the immutable sequence `("free", "strlen")`. Independently
reject:

- a missing field;
- a non-array field;
- a non-string entry;
- an empty string;
- `["free", "free"]`; and
- `["strlen", "free"]`.

Do not sort or deduplicate in Python. Confirm that non-function JSON records
neither require nor emit the field.

### A3-RENDER-01 `targets_use_final_names_and_omit_empty_foreign_line`

Use records originating from exact input A3-SRC-FOREIGN. Render
`parser::scan`, `parser::release`, and `parser::scalar` in ascending internal
item-ID order.

The exact leading portions are:

```text
### Function `scan`

Foreign function references: `free`, `strlen`

Source:
```

```text
### Function `release`

Foreign function references: `free`

Source:
```

```text
### Function `scalar`

Source:
```

Each entry then contains its exact annotated source and target skeleton in the
existing fences and labels. Entries are joined by exactly two newlines.
Neither `parser::` nor an item ID appears in a heading. The scalar entry
contains no `Foreign function references:` line and no additional blank line
in its place.

### A3-RENDER-02 `noncolliding_dependencies_use_kind_and_final_name`

Use exact input A3-SRC-KINDS and construct Dependency Context for `target`.
The selected entries retain ascending internal item-ID order and have these
headings:

```text
### Struct `Point`
### Static `LIMIT`
### Const `STEP`
### Function `helper`
```

Each heading is followed by one blank line and the existing kind-specific
definition, declaration, or signature rendering. No heading contains an item
ID or a module path. Dependency closure and selected item IDs are unchanged.

### A3-RENDER-03 `duplicate_dependency_names_remain_name_only`

Use exact input A3-SRC-COLLISION and construct Dependency Context for
`target`. The two direct function entries render exactly as:

```text
### Function `parse`
### Function `parse`
```

They deliberately have duplicate headings. Neither `left::`, `right::`, nor
`outer::` appears in a heading. The selected records and their final order
remain unchanged; the source signatures remain available beneath the
headings.

### A3-BUDGET-01 `budget_counts_final_name_only_entries_not_section_heading`

Use exact input A3-SRC-COLLISION. Let `entries` be the complete name-only
Dependency Context entry string for `target`, including both duplicate
function headings shown in A3-RENDER-03 but excluding the
`## Dependency Context` section heading.

With:

```python
limit = len(entries)
```

context construction succeeds and returns exactly `entries`. With:

```python
limit = len(entries) - 1
```

mandatory-context construction fails before an LLM call. Rendering the
successful value through the prompt adds `## Dependency Context` without
charging those heading characters to the entry budget.

## 7. Prompt and unchanged-protocol tests

### A3-PROMPT-01 `version_1_prompt_has_exact_hierarchy_and_advisory`

Use nonempty context originating from exact input A3-SRC-KINDS and a target
entry originating from exact input A3-SRC-FOREIGN.

Load `local_transformation`, version 1, with the existing description and
frontmatter variables. The exact Amendment 2 paragraph remains present once:

```text
Complete every generated `todo!()` hole. Preserve every complete labeled statement already present in the Target Skeleton exactly as provided.
```

The new exact requirement is present once:

```text
10. For each listed foreign-function reference, prefer a behavior-equivalent
    safe Rust function or method when one is available; otherwise preserve the
    foreign call.
```

The unsafe-block and fenced-output requirements are numbered 11 and 12.

The exact final rendered section hierarchy is:

```text
## Dependency Context

<exact nonempty dependency entries>

## Transformation Targets

<exact target entries>
```

There is no plain `Dependency Context:` or `Transformation Targets:` label.
Every item heading beneath the sections is `###`. Assert byte equality with
the final normative template, unchanged prompt ID/version/variables, and the
SHA-256 of the complete rendered text.

### A3-PROMPT-02 `empty_dependency_section_is_omitted_and_repair_text_is_unchanged`

Use exact input A3-SRC-PLAIN, whose nonrecursive singleton target has empty
Dependency Context.

The initial rendered prompt contains:

```text
## Transformation Targets
```

and contains no `## Dependency Context`, no plain `Dependency Context:`, and
no `Foreign function references:` line.

Render an independent repair with exact failed text:

```text
unsafe fn scalar() { broken }
```

and exact diagnostics:

```json
{"schema_version":1,"status":"invalid","failures":[{"id":0,"name":"scalar","failed_snippet":"unsafe fn scalar() { broken }","errors":[{"code":"missing_label","message":"Function `scalar` (item 0): label 0 is missing."}]}]}
```

The complete original prompt and Transformation Targets section are repeated,
the supplied failed transformation is present in the repair block, and the
complete diagnostics string is preserved byte-for-byte. The internal
diagnostic ID is not sanitized or reserialized. This distinguishes unchanged
raw repair diagnostics from the authored headings cleaned up by Amendment 3.

### A3-PROTO-01 `foreign_metadata_does_not_change_graph_or_tool_requests`

Use exact input A3-SRC-FOREIGN and its Crat-generated function records.

Assert that:

- `function_graph` uses only ordinary function-valued dependency IDs;
- foreign names add no node or edge;
- the version-1 structural-validation request contains the existing expected
  function metadata and Amendment 2 preservation metadata, but no
  `foreign_function_names`;
- the version-1 replacement request likewise contains its existing item and
  Amendment 2 metadata, but no `foreign_function_names`; and
- retaining `crate::free(pointer)` in a structurally valid returned
  `parser::release` body is not rejected merely because `free` is listed.

Use the exact generated annotated target skeleton for `parser::release` as the
expected function. The exact returned transformation is:

```rust
pub unsafe fn release(mut pointer: *mut core::ffi::c_void) {
    #[proctor(0)]
    crate::free(pointer);
    #[proctor(1)]
    crate::free(pointer);
}
```

This is a parser/structural validation case; it does not ask Python to parse
Rust and does not require the validator to resolve or remove either foreign
call.

## 8. Completion criteria

Amendment 3 is complete when:

- all 12 named cases are implemented and passing;
- every superseded implementation fixture and golden is updated without
  editing an existing test-plan file;
- Crat derives foreign names from resolved foreign function identities and
  emits one sorted unique list per function;
- Python validates but never semantically reconstructs the list;
- ordinary authored prompt headings expose no skeleton item ID;
- target and dependency headings never expose record paths, including when
  duplicate dependency headings result;
- dependency budgets use final name-only entry text while section headings
  remain outside the budget;
- empty foreign lines and empty Dependency Context sections are omitted;
- Amendment 2 statement preservation, statement labels, prompt paragraph, and
  tool metadata remain intact;
- foreign replacement remains advisory and unvalidated; and
- ordinary graph, scheduling, repair, replacement, transaction, compilation,
  usage, and reporting behavior is unchanged.
