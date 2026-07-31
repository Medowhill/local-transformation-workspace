# Amendment 5 Test Plan: Before/After Statement-Pair Dump

## 1. Purpose

This document specifies the focused regression suite for Amendment Plan 5 in
[amendment-5-plan.md](amendment-5-plan.md). It covers compiler-resolved source
binding facts, canonical one-to-many after groups, the replacement sidecar,
build-acceptance filtering, deterministic Markdown rendering, and artifact
publication.

The suite contains 26 named cases:

| Area | Cases |
| --- | ---: |
| Existing regression updates | 2 |
| Skeleton binding facts and before statements | 7 |
| Canonical replacement library | 6 |
| CLI serialization | 1 |
| Python protocol, acceptance, and Markdown artifacts | 10 |

## 2. Test execution and ownership policy

Rust semantic, serialization, canonicalization, and CLI-library tests live
beside the existing Crat tools modules. Run:

```bash
cd proctor/stages/crat
cargo test -p tools skeleton::tests
cargo test -p tools item_replacer::tests
cargo test -p tools
cargo test --bin crat-tool
cargo test --workspace
```

Use `run_compiler_on_str` for binding identity and type facts. Parser-only
tests remain parser-only. Do not nest compiler callbacks: own any generated
records or source before starting another callback.

Python protocol, orchestration, transaction, renderer, and artifact tests live
in `proctor/tests/test_local_transformation.py` and use temporary directories,
fake Crat tools, fake Cargo builds, and fake LLM responses. Run:

```bash
cd proctor
uv run pytest tests/test_local_transformation.py
```

They remain offline and do not invoke a real compiler, Cargo, or provider.
One optional real-tool integration regression may reuse an existing e2e
fixture, but it is not a substitute for the deterministic unit cases.

After changing Rust source, run from `proctor/stages/crat`:

```bash
cargo fmt
cargo clippy --workspace --all-targets
cargo test --workspace
```

Resolve all Clippy warnings. Use source `#[allow(clippy::...)]` only for
`len_without_is_empty`, `too_many_arguments`, or `type_complexity`, as allowed
by the repository instructions.

After changing Python source, run from `proctor`:

```bash
uv run pytest tests/test_local_transformation.py
uv run ruff check stages/local-transformation tests/test_local_transformation.py
uv run ruff format --check stages/local-transformation tests/test_local_transformation.py
```

No case edits historical phase/amendment test plans or a checked-in pipeline
configuration.

## 3. Comparison policy and shared Rust inputs

JSON keys, list order, sidecar keys, Markdown text, blank lines, and command
arguments are exact where stated. Binding inclusion and deduplication are
semantic and use `HirId`, never rendered identifier equality. No JSON or
Markdown oracle contains a `HirId`.

The expected type spelling begins with the same source-valid syntax placed in
the prompt-facing source or target skeleton. Markdown table oracles then
collapse each maximal run of ASCII space, tab, carriage return, line feed, or
form feed to one ASCII space and trim the ends. They do not normalize aliases
or otherwise change type syntax semantics unless Amendment 4's inferred-type
path necessarily renders a semantic type.

Every before and after snippet oracle retains its complete canonical
`#[proctor(N)]` attributes. Fence lengths and every heading/table code value
follow the exact dynamic-fence and HTML/entity algorithm in A5.7.

### A5-SRC-BINDINGS

```rust
pub unsafe fn update(mut pointer: *mut i32, mut amount: i32) -> i32 {
    let mut alias = pointer;
    *alias += amount;
    *pointer + *alias
}
```

This input supplies one raw-pointer parameter, one inferred simple
raw-pointer local, one scalar parameter, repeated references, and statements
that may share the same bindings. Tests first assert the actual initial
pointer decisions used by their exact expected target strings.

### A5-SRC-NESTED

```rust
pub unsafe fn choose(mut pointer: *mut i32, mut flag: bool) -> i32 {
    if flag {
        *pointer += 1;
        *pointer
    } else {
        0
    }
}
```

Use the existing preservation override/test helper where necessary to require
both the control parent and its pointer-using descendants to have disposition
`transform`. The case is about overlapping complete subtrees, not incidental
analysis changes.

### A5-SRC-SHADOW

```rust
pub unsafe fn shadow(mut pointer: *mut i32) -> i32 {
    if pointer.is_null() {
        0
    } else {
        let mut value = pointer;
        {
            let mut value = pointer;
            *value += 1;
        }
        *value
    }
}
```

The two locals named `value` are distinct compiler bindings. A parent subtree
that contains both has two rows even though their rendered names match;
repeated uses of either binding do not add rows.

### A5-SRC-EXCLUSIONS

```rust
pub struct Holder {
    pub pointer: *mut i32,
}

pub static mut GLOBAL: *mut i32 = core::ptr::null_mut();

pub unsafe fn exclusions(mut holder: Holder, mut pointer: *mut i32) -> i32 {
    let mut copied = holder.pointer;
    let mut tuple = (pointer, 1_i32);
    let mut scalar = *pointer;
    scalar += *GLOBAL;
    scalar + *tuple.1 + *copied
}
```

`pointer` and inferred raw-pointer local `copied` are eligible where they
occur. `holder`, `tuple`, and `scalar` are excluded because their outer types
are not raw pointers. `holder.pointer`, `GLOBAL`, and dereference expressions
are not variable rows of their own.

### A5-SRC-EXPANSION

```rust
pub unsafe fn expansion(mut pointer: *mut i32) -> i32 {
    *pointer
}
```

The replacement test returns label 0 as several consecutive statements,
including a fresh `proctor_temp_var_0`, and verifies that the sidecar contains
the complete canonical group while the generated temporary never enters the
source variable table.

### A5-SRC-INCOMPLETE

```rust
macro_rules! bump {
    ($pointer:expr) => {{
        *$pointer += 1;
    }};
}

pub unsafe fn incomplete(mut pointer: *mut i32) -> i32 {
    if !pointer.is_null() {
        bump!(pointer);
    }
    *pointer
}
```

The `if` statement has a directly resolvable `pointer` occurrence and a
surface macro invocation. Its metadata is therefore incomplete while
retaining the parameter row. Use a collector-level missing-map fixture for an
entirely unresolvable row-list subcase rather than depending on whether rustc
maps a particular expanded macro statement in a pinned compiler build.

## 4. Existing regression updates

### A5-UPDATE-01 `function_records_and_python_helpers_add_statement_pair_metadata`

Update every exact Rust `FunctionRecord` JSON golden and Python `fn_record`
helper. The exact function key order is:

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
statement_pair_metadata
foreign_function_names
signature_dependencies
dependencies
```

An all-preserved helper defaults `statement_pair_metadata` to `[]`. A
one-label transforming helper must provide matching before metadata, its
`pointer_variables_complete` Boolean, and typed origins rather than allowing
the Python loader to synthesize them. Non-function records remain unchanged.

### A5-UPDATE-02 `replacement_api_cli_and_fake_tools_require_the_sidecar_output`

Update existing replacement library assertions to read
`ReplacementOutput.source`. Update exact CLI argv to contain
`--statement-pairs-output` and its path. Update `FakeTools.replace` to create
both candidate and sidecar files and to record both paths. Existing validation
request and replacement request JSON assertions remain byte-for-byte
unchanged.

## 5. Skeleton binding facts and before-statement tests

### A5-RUST-01 `records_parameter_and_inferred_local_types_from_source_and_target_asts`

Use A5-SRC-BINDINGS. Assert that every transformed statement has one metadata
entry and that eligible rows report the exact source and generated target
syntax. `pointer` has `before_type_is_inferred == false`; `alias` has
`before_type_is_inferred == true`; scalar `amount` never appears. Assert
parameter target lifetimes and local target types exactly as emitted in
`annotated_skeleton`, including a raw fallback if that is the asserted initial
decision. All ordinary mapped entries set `pointer_variables_complete` to
`true`.

Include a long source-valid type whose pretty rendering wraps across lines.
Assert skeleton metadata retains the rendered type rather than rejecting the
function; its single-line report oracle is covered by A5-PY-07.

### A5-RUST-02 `deduplicates_by_binding_identity_in_first_occurrence_order`

Use A5-SRC-BINDINGS and A5-SRC-SHADOW. Repeated references to one binding
produce one row. The parent control statement containing both shadowed
`value` bindings contains two rows in source first-occurrence order. Do not
deduplicate by name/type text or a rendered statement scan. Assert exact
origins:
`{"kind":"parameter","index":0}` for `pointer` and distinct local declaration
labels for the shadowed rows. Assert no serialized or rendered compiler ID.

### A5-RUST-03 `before_statement_is_the_complete_prompt_facing_source_subtree`

Use A5-SRC-NESTED. Assert that the parent before snippet includes its complete
then/else subtree and descendant statements, while each transformed child has
its own smaller snippet. Assert presentation-mutability and pretty-printing
agree with `annotated_source`, not the byte input.

Assert every canonical `#[proctor(...)]` attribute remains in the snippet,
including descendant labels in the complete parent subtree.

### A5-RUST-04 `includes_only_outer_raw_pointer_parameters_and_simple_locals`

Use A5-SRC-EXCLUSIONS. Assert the exact eligible identities described below
that input. In particular, do not add rows for a pointer field, mutable
static, aggregate containing a pointer, scalar loaded through a pointer, or
an expression. Add a parser-only generated-temporary example and prove it
cannot enter skeleton source metadata.

### A5-RUST-05 `metadata_labels_exactly_match_transformation_dispositions`

Use one all-preserved function and A5-SRC-NESTED. Assert metadata labels are
strictly increasing and exactly equal
`statements_requiring_transformation`. Assert empty arrays for preserved
functions, empty `pointer_variables` when a transformed statement has no
eligible binding, `pointer_variables_complete == true` for a completely mapped
empty result, byte-deterministic repeated JSON, exact field/origin key order,
and no metadata field on non-function records.

### A5-RUST-06 `main_override_aliases_and_raw_fallbacks_report_actual_skeleton_types`

Use focused existing skeleton fixtures for:

- the two-argument `main_0` `argv` override;
- an explicit source alias hiding or naming a raw pointer; and
- a decision retaining a raw pointer.

For every variable occurrence, assert before syntax comes from the
prompt-facing source rule and selected target syntax comes from the final
target AST. Do not recompute type text from `PtrKind` inside the report
collector.

### A5-RUST-07 `unmappable_statement_emits_resolvable_rows_and_incomplete_flag`

Use A5-SRC-INCOMPLETE. Assert skeleton generation succeeds, the `if` entry has
`pointer_variables_complete == false`, and its resolvable `pointer` row retains
the parameter-0 origin and exact source/target types. In a collector-level
missing-map subcase, assert an entirely unresolvable statement emits an empty
row array with the same false flag. Neither subcase silently marks partial
metadata complete.

## 6. Canonical replacement sidecar tests

### A5-REP-01 `replacement_output_has_exact_source_and_sorted_sidecar_shape`

Use two requested functions supplied in reverse item-ID order. Assert
`ReplacementOutput.source` equals the existing replacement oracle and
typed `ReplacementStatementPair` values are ordered by item ID then label.
Assert a fully preserved request produces an empty typed pair vector. JSON
serialization belongs only to A5-CLI-01.

### A5-REP-02 `one_source_statement_reports_the_complete_canonical_expansion_group`

Use A5-SRC-EXPANSION. Return a valid label-0 group with multiple consecutive
statements. Assert `after_statement` contains every statement exactly once in
order and is extracted before labels are removed from the final candidate.
Assert all outer and descendant `#[proctor(N)]` attributes remain in the exact
snippet oracle.

### A5-REP-03 `canonical_after_restores_preserved_descendants_before_capture`

Use a transformed control parent with a preserved child. Deliberately alter
the preserved child in returned code. Assert the sidecar's parent after
snippet contains the immutable canonical preserved statement, while the
emitted candidate retains the existing canonical replacement behavior. The
raw altered text must not occur in the sidecar.

### A5-REP-04 `overlapping_parent_and_descendant_labels_each_get_one_entry`

Use A5-SRC-NESTED with transformed parent and descendant labels. Assert one
sidecar entry per transformed label. The parent after group contains the
descendant subtree; the descendant entry contains only its own canonical
group. No interval subtraction or child-hole replacement occurs.

### A5-REP-05 `sidecar_excludes_preserved_labels_and_generated_variable_type_rows`

Use a mixed preserved/transformed function and A5-SRC-EXPANSION. Assert only
labels in `statements_requiring_transformation` are emitted. The generated
temporary may occur in after Rust text but no sidecar type schema exists for
it and no source metadata row names it.

### A5-REP-06 `replacement_library_failures_return_no_typed_output`

In `item_replacer::tests`, run independent malformed-label,
target-resolution, unsupported-conversion, call-rewrite, and source-rewrite
subcases. Each returns no successful `ReplacementOutput` and no partial
sidecar data. No case performs filesystem assertions or weakens independent
replacement canonicalization.

## 7. CLI serialization test

### A5-CLI-01 `replace_cli_helper_serializes_both_outputs_exactly`

Give the pure serialization helper used by `crat-tool replace` a typed
`ReplacementOutput` with out-of-order-looking paths but already canonical
item-ID/label order. Under `cargo test --bin crat-tool`, assert the candidate
bytes and exact pretty sidecar JSON, schema version, object/key order, string
escaping, absence of an appended newline, and empty-list shape. Exercise the
Clap parser directly and assert replace argv without
`--statement-pairs-output` is rejected while the complete form stores both
distinct output paths. As a subcase, pass the same candidate and sidecar path
and assert rejection at the CLI/parser boundary before compilation or any
write.

Keep this test free of project compilation. Actual destination clearing,
regular-file requirements, and partial filesystem outputs belong to
A5-PY-03, not the tools library or this pure serialization test.

## 8. Python protocol, acceptance, and Markdown artifact tests

### A5-PY-01 `skeleton_loader_requires_exact_matching_statement_metadata`

Accept valid empty and populated metadata. Reject a missing or unknown nested
key, wrong container/object/scalar types, an out-of-range label,
missing/extra/reordered labels, empty names/types, a trailing statement
newline, a non-Boolean completeness or inference flag, malformed origin
variants, out-of-range origin values, duplicate origins within one statement,
and malformed variable entries. Reject a carriage return or newline in a name
but accept a source-valid type string containing pretty-printer line wrapping.
Assert the loader preserves wrapped types, incomplete entries, and producer
order without sorting.

### A5-PY-02 `replacement_sidecar_loader_is_strict_and_cross_checks_the_scc`

Accept the exact A5.5 sidecar. Reject unsupported versions, unknown/missing
keys, invalid integer ranges, empty path/snippet, duplicate or unsorted keys,
a path with a carriage return or newline, an after snippet with a carriage
return or trailing newline, an item/path outside the SCC, and any missing or
extra expected transformed label. A fully preserved SCC accepts only an empty
list.

### A5-PY-03 `crat_tools_replace_clears_and_requires_both_scratch_outputs`

At the Python `CratTools` filesystem boundary, precreate stale candidate and
sidecar files and assert `replace` clears exactly both before invocation.
Exercise independent missing, symlink, directory, and other-nonregular output
subcases for each destination. Stale regular files and symlinks are unlinked;
stale directories and other nodes are rejected without invoking the command
or recursively deleting them. A zero exit is usable only when both newly
created paths are regular non-symlink files. A command failure or unusable
second output never returns a usable candidate. Do not test these filesystem
properties in the tools-library or CLI serializer cases.

### A5-PY-04 `failed_build_attempt_is_discarded_and_repair_success_is_kept_once`

Provide two valid replacement sidecars with visibly different after snippets.
Make the first candidate build fail and the repaired candidate build succeed.
Assert the accumulator and final Markdown contain only the second canonical
snippet. Existing compilation-failure, generation, repair, and Cargo-build
metrics remain unchanged.

### A5-PY-05 `malformed_sidecar_is_fatal_before_candidate_installation`

Return a valid candidate and malformed or mismatched sidecar. Assert the
library source is never installed, Cargo is not invoked for that candidate,
the failure is not sent to the LLM, and no final report or Rust-project output
is claimed.

### A5-PY-06 `accepted_pairs_sort_by_item_and_label_not_scc_schedule`

Process several SCCs in leaf-first order that differs from item-ID order.
Give one function multiple transformed labels. Assert the final report groups
by ascending item ID and ascending label and emits each item heading once.
Earlier accepted SCC pairs remain intact while later SCCs run.

### A5-PY-07 `markdown_renders_origins_completeness_and_escaping_exactly`

Test complete Markdown bytes for parameter and local origins, explicit and
inferred types, repeated shadowed names, a complete empty row list, an
incomplete list with resolvable rows, and an incomplete empty list. Assert the
exact warning and both distinct no-row sentences.

Independently feed the single-line code renderer synthetic values containing
`&`, `<`, `>`, `|`, backticks, and backslashes. Assert the exact ordered
entities and `<code>...</code>` wrapper, including that generated entity
ampersands are not escaped twice. Reject CR/LF in non-type values. Give the
snippet renderer inputs whose longest backtick runs have lengths 0, 3, and 5
and assert fence lengths 3, 4, and 6, exact `rust` info strings, preserved
snippet bytes, and one boundary newline. Assert headings, table columns
including `Origin`, `yes`/`no`, blank lines, and canonical labels exactly.

For type cells, add a long realistic pretty-wrapped source-valid type oracle
whose input is the escaped string
`" \tOption<\r\n    &'a mut [core::mem::MaybeUninit<i32>;\f32],\n> "`.
Assert its normalized spelling is exactly
`"Option< &'a mut [core::mem::MaybeUninit<i32>; 32], >"` and its final cell is
exactly
`<code>Option&lt; &amp;'a mut [core::mem::MaybeUninit&lt;i32&gt;; 32], &gt;</code>`.
This covers maximal runs of spaces, tabs, CR, LF, and form feeds, trimming,
ordered escaping after normalization, and the rule that multiline type
rendering never rejects a translation. Keep the existing CR/LF rejection for
non-type single-line values.

### A5-PY-08 `markdown_uses_complete_before_and_canonical_after_snippets`

Render one overlapping parent/child example and one one-to-many expansion.
Assert complete snippets, no raw rejected LLM text, and no final-project
reparse. Assert all outer and descendant canonical `#[proctor(N)]` attributes
are retained symmetrically in before and after snippets.

### A5-PY-09 `report_destination_fallback_empty_report_and_stale_path_handling`

Run successful independent subcases with and without
`outputs.artifacts_dir`. Assert `statement-pairs.md` is under the artifact
directory in the first case and `framework.workdir` in the second, never under
the Rust project. Precreate a stale regular report and a stale report symlink;
assert the exact path is unlinked at invocation start and replaced only by the
new successful report. A later stage failure after stale removal leaves the
final report path absent. A directory or other node at the exact report path
is rejected without recursive removal, and siblings are untouched.

Before those stale-path subcases, exercise report paths equal to, ancestors
of, and descendants of `outputs.rust_project`. Assert initial boundary
validation rejects every overlap before stale report clearing, final-project
copying, or any output mutation.

An all-preserved/empty schedule writes the exact explanatory empty report.
Assert `config_used`, `stage.toml`, prompt metadata, metrics, and
`StageOutput.logs` retain their existing shapes.

### A5-PY-10 `final_copy_and_report_publication_cleanup_is_exact`

Use instrumentation around final project copy and atomic report replacement.
Assert the report is rendered in memory, its sibling temporary is created only
after every SCC and `_copy_final` succeed, and publication uses atomic
replacement in the destination directory.

Run independent later-fatal-SCC, exhausted-repair, partial-final-copy,
Markdown-temp-write, and atomic-publication failure subcases. Assert:

- no final report or report temporary exists after failure;
- a partial or complete `outputs.rust_project` created by this invocation is
  absent after final-copy or report-publication failure;
- the exact stale report removed at start is never restored;
- unrelated artifact siblings, work files, and parent directories remain;
- cleanup errors are included with the primary stage failure and no successful
  output is claimed; and
- no per-attempt JSON sidecar appears under `artifacts_dir`.

On success, assert atomic publication is the last transformation-output
filesystem mutation inside `run_stage` and both the Rust output and final
report exist. The outer `main` may then write the required
`stage_output.json`; that protocol write is not treated as another
transformation-output mutation.

## 9. Completion criteria

Amendment 5 is complete when:

- all 26 named cases are implemented and passing;
- each transformed metadata label has one prompt-facing before statement and
  one build-accepted canonical after expansion group;
- every statement marks pointer-variable completeness strictly, retains
  resolvable rows when incomplete, and renders the exact warning;
- pointer-variable rows carry typed stable origins, are compiler-identity
  deduplicated, and remain limited to the agreed source binding scope without
  serializing `HirId`;
- all canonical statement labels remain visible in both report snippets;
- failed and superseded repair attempts contribute no report data;
- the final Markdown is deterministic, unconditional, human-readable, and
  outside the Rust project, with wrapped source-valid types normalized to one
  line only for report presentation;
- stale-report, partial-copy, report-write, and publication failures clean up
  only exact current-invocation outputs;
- no rule extraction/application, prompt, validator, replacement-request,
  pointer-decision, metric, or framework artifact-kind behavior changes;
- `docs/prototype-desc.md` and the Amendment 5 overview accurately describe
  the implemented behavior; and
- the focused checks in Section 2, Crat formatting, Clippy, and relevant full
  test suites pass.
