# Amendment 5 Detailed Plan

This is a prospective implementation plan for the local-transformation
prototype. It must be reconciled with the current implementation and tests
when the work begins. Do not treat behavior described here as implemented
until [prototype-desc.md](prototype-desc.md) says that it is.

See the [prototype-plan overview](prototype-plan.md#amendment-5-planned).
See the [Amendment 5 test plan](amendment-5-test-plan.md).

## Amendment Plan 5: Human-readable before/after statement-pair dump

This amendment adds one diagnostic artifact that helps a researcher inspect
the local transformations already performed by the prototype. For every
statement whose Amendment 2 disposition is `transform`, it records the
prompt-facing source statement, the canonical accepted replacement expansion
group, and the source and selected target types of relevant existing
raw-pointer bindings.

The report is observational. It does not extract, represent, validate, load,
or apply reusable transformation rules. It does not affect the prompt,
validation criteria, replacement semantics, SCC schedule, repair budget,
pointer decisions, or generated Rust project.

### A5.1 Scope and terminology

A *reported statement* is a source statement label listed in a function
record's `statements_requiring_transformation`. Textual `todo!()` detection is
never used. A transformed declaration without a hole is still reported, and a
literal `todo!()` outside this metadata has no reporting significance.

The *before statement* is the complete source `rustc_ast::Stmt` subtree for
that label in the same immutable, sanitized, presentation-normalized function
used to produce `annotated_source`. It therefore reflects the exact source
presentation given to the LLM:

- explicit ABI syntax and `#[no_mangle]` are absent from the function
  presentation;
- non-`ref` bindings are presentation-mutable;
- the complete statement subtree, including nested labeled statements, is
  present; and
- the source is Crat's parsed and pretty-printed presentation, not a
  byte-for-byte slice of the input file.

The *after statement* is the complete canonical replacement expansion group
for the same transformed label. One source statement may correspond to
several consecutive returned statements carrying that label. Canonical
replacement has already restored every preserved group, checked the unique
control anchor and nested structural roles, and rejected malformed alignment.
The after statement is not taken from the raw LLM response and is not
reconstructed from the final unlabeled project.

Nested transformed statements deliberately produce overlapping entries. A
transformed parent entry contains its complete canonical child subtrees, and
each transformed descendant also receives its own entry. This amendment does
not turn child regions into holes in the parent or attempt to derive
non-overlapping rule candidates.

The *pointer variables* for one entry are existing source bindings that meet
all of these conditions:

1. the binding is a function parameter or a simple local identifier binding;
2. its compiler-resolved source semantic outer type is a raw pointer;
3. its declaration or a compiler-resolved reference to it occurs in the
   complete source statement subtree; and
4. it existed in the immutable source function.

“Simple local identifier binding” has the same boundary as current target
local materialization: a direct identifier pattern without a subpattern.
Compiler binding identity, not identifier text, defines equality. Repeated
uses of one binding produce one row, while shadowed bindings with the same
rendered name remain distinct rows.

The following are outside the report:

- generated `proctor_temp_var_<n>` bindings, because they have no source
  counterpart;
- destructuring and control-pattern bindings for which the prototype does not
  materialize an independent selected target type;
- statics, constants, fields, and arbitrary pointer-valued expressions;
- a binding whose outer semantic type is not a raw pointer, even if a named
  aggregate or generic component recursively contains one; and
- return types, which are not variables appearing in a statement.

Raw-pointer fallback decisions remain in scope. A row is emitted even when the
selected target type is still a raw pointer or is textually equal to the
source type.

### A5.2 Binding facts and type presentation

Collect binding facts while skeleton generation owns the source AST/HIR
mapping, `TyCtxt`, initial pointer decisions, scope-aware type speller, and
both the source and completed target-skeleton ASTs. Python must never infer
bindings or types by parsing rendered Rust.

Build one per-function catalog keyed internally by the binding's `HirId`. It
contains only the eligible parameters and simple locals from A5.1. For every
catalog entry record:

- the identifier as displayed in the prompt-facing source;
- the before type;
- the selected target type; and
- whether the before type was inferred in the source.

For an explicitly typed parameter or local, render the before type from its
prompt-facing source AST type. This intentionally preserves a valid source
alias or relative spelling, including an alias whose normalized semantic type
is a raw pointer. For an inferred local, render the compiler type through
Amendment 4's source-valid, containing-module type speller. Do not use
`Ty::to_string()` as source syntax.

Render the selected target type from the corresponding completed target
skeleton AST:

- parameter types come from the target signature after ordinary initial
  decisions, lifetimes, scope-aware spelling, and the two-argument `main_0`
  override;
- local types come from the type materialized by `Skeletonizer`; and
- an absent or raw-fallback decision is represented by the target skeleton's
  actual retained type rather than by inventing a safer type.

This makes the report agree with what validation and replacement require.
Types are report presentation, not a second pointer-decision format.
Keep these strings as the source-valid AST/type renderings even when the
pretty-printer wraps a long type across lines. A5.7 performs the deterministic
single-line normalization only when it renders a report table. That
normalization is report-only spelling normalization: it does not change the
source or target AST, reselect a type, or cause an otherwise valid translation
to be rejected.

For each transformed source statement, visit these occurrence-bearing surface
constructs in depth-first source order:

- every direct identifier binding pattern; and
- every expression path.

Map each construct through the existing AST/HIR map and resolve local
references through rustc. Add eligible identities in first-occurrence order.
When the statement itself maps to a HIR root, also traverse that complete HIR
subtree and append any additional eligible identity at its first HIR
occurrence; this retains resolvable identities exposed only by lowering or
macro expansion. Deduplicate the combined sequence by `HirId`. The complete
parent traversal includes nested statements, which is required by the
overlapping-entry policy.

Initialize `pointer_variables_complete` to `true` and set it to `false` under
any of these exact conditions:

- the statement has no mapped HIR root;
- a visited direct identifier binding pattern has no corresponding HIR
  pattern;
- a visited expression path has no corresponding HIR expression or cannot be
  resolved; or
- the surface subtree contains a macro invocation, because token references
  cannot be correlated completely with compiler identities.

Other resolved nonlocal paths and resolved local bindings outside the eligible
catalog do not make the list incomplete; they have been proved irrelevant to
the agreed row scope. Continue collecting every binding that can be resolved
when the flag becomes false. Do not fail skeleton generation and do not
silently present the partial list as complete.

Give every catalog entry a stable human origin without serializing compiler
identity:

- a parameter origin contains its zero-based parameter index; and
- a local origin contains the numeric label of its simple `let` declaration
  statement.

The declaration label comes from the immutable annotated source function and
may identify a preserved statement nested beneath a transformed parent.
`HirId` remains an in-memory deduplication key only and must never appear in
JSON or Markdown.

### A5.3 Skeleton-record metadata

Add these serializable Rust structures in the Crat tools skeleton module:

```rust,ignore
pub struct StatementPairMetadata {
    pub label: u32,
    pub before_statement: String,
    pub pointer_variables_complete: bool,
    pub pointer_variables: Vec<PointerVariableMetadata>,
}

pub struct PointerVariableMetadata {
    pub name: String,
    pub origin: PointerVariableOrigin,
    pub before_type: String,
    pub selected_target_type: String,
    pub before_type_is_inferred: bool,
}

#[serde(tag = "kind", rename_all = "snake_case")]
pub enum PointerVariableOrigin {
    Parameter { index: u32 },
    Local { declaration_label: u32 },
}
```

Add the required field below to every `FunctionRecord`, immediately after
`statements_requiring_transformation`:

```rust,ignore
pub statement_pair_metadata: Vec<StatementPairMetadata>,
```

The amended function-record serialization order is:

```text
id, path, kind, name, annotated_source, annotated_skeleton,
source_signature, target_signature, needs_transformation,
statements_requiring_transformation, statement_pair_metadata,
foreign_function_names, signature_dependencies, dependencies
```

`statement_pair_metadata` is ordered by increasing label. Its labels must
equal `statements_requiring_transformation` exactly: no missing, extra,
duplicate, or reordered labels. A function with no transformed statements
uses an empty array. Variable rows use the deterministic first-occurrence
order from A5.2 and may be empty. `pointer_variables_complete` is `true` only
under A5.2's proof and is otherwise `false` while retaining all resolvable
rows.

The exact serialized origin variants are:

```json
{"kind": "parameter", "index": 0}
{"kind": "local", "declaration_label": 3}
```

Each statement metadata object serializes keys in this order:

```text
label, before_statement, pointer_variables_complete, pointer_variables
```

Each pointer-variable object serializes keys in this order:

```text
name, origin, before_type, selected_target_type, before_type_is_inferred
```

The keys occur in the shown order. Parameter indices and declaration labels
are u32 values. Within one statement entry, no two rows may have the same
origin; compiler identity establishes this invariant before serialization.
Crat construction also requires a parameter origin to match that binding's
actual signature position and a local origin to match the canonical label on
that binding's direct `let` declaration. Python validates the closed variant
shape, range, and per-statement uniqueness but does not parse Rust to repeat
those semantic checks.

The before statement is rendered from the already annotated source AST
statement rather than located by parsing `annotated_source`. Retain every
canonical `#[proctor(N)]` attribute in the snippet, including descendant
labels in an overlapping parent. Do not run the final-project label remover on
report snippets.

Extend PROCTOR's immutable Python `ItemRecord` model with matching nested
immutable values. The loader requires the new function field and validates:

- exact statement, variable, and origin object key sets and field types;
- u32 labels;
- exact increasing-label equality with
  `statements_requiring_transformation`;
- nonempty before statements, binding names, and type strings;
- no trailing carriage return or newline in a before statement;
- no carriage return or newline in a binding name;
- Boolean completeness and inference flags;
- exact typed origin objects with u32 values and no duplicate origins within
  one statement; and
- deterministic producer order without sorting, deduplicating, or repairing
  malformed input.

The top-level skeleton JSON remains an internal, unversioned artifact. No
framework stage-envelope schema changes.

Do not copy this metadata into validation requests. Validation has no reporting
responsibility, and its version-1 protocol remains unchanged.

### A5.4 Canonical after-group extraction

Extend Crat's replacement library result so one successful, atomic replacement
calculation returns both:

```rust,ignore
pub struct ReplacementOutput {
    pub source: String,
    pub statement_pairs: Vec<ReplacementStatementPair>,
}

pub struct ReplacementStatementPair {
    pub item_id: u64,
    pub path: String,
    pub label: u32,
    pub after_statement: String,
}
```

The exact Rust names may follow existing module conventions, but there must be
one typed library result rather than separate parsing logic in the CLI.

For each requested item, first perform the existing strict replacement
canonicalization. From that labeled canonical function, locate the complete
consecutive expansion group for every label in
`statements_requiring_transformation`. Reuse the shared preservation grouping
and label helpers; do not implement a looser text or regex alignment path.
Render the group only after preserved descendants have been restored and
before the implementation's labels are removed. Retain every canonical
`#[proctor(N)]` attribute in the rendered after snippet, symmetrically with
the before snippet.

After collection, continue the existing composition, wrapper generation,
call-site rewriting, executable handling, and recursive label removal to
produce `ReplacementOutput.source`. Reporting must not affect the emitted
source.

Return one sidecar entry per transformed label, ordered by item ID and then
label, regardless of replacement-request iteration or hash-map order. Do not
emit preserved labels. A request containing only fully preserved functions
returns an empty sidecar list.

All current replacement failures remain atomic. A malformed label group,
canonicalization failure, target-resolution failure, unsupported conversion,
call-rewrite failure, or source rewrite failure returns no successful
`ReplacementOutput`. The report path must not weaken replacer's independent
canonicalization or make it trust prior validation.

### A5.5 Thin CLI and per-attempt sidecar

Amend `crat-tool replace` to require a second output path:

```text
crat-tool replace \
  --request <replacement-request.json> \
  --output <candidate.rs> \
  --statement-pairs-output <replacement-statement-pairs.json> \
  <current-project>
```

The CLI remains responsible only for project/source I/O, compiler invocation,
and serialization. It calls the typed replacement API once, writes
`ReplacementOutput.source` to the candidate path, and writes this exact
version-1 sidecar shape:

```json
{
  "schema_version": 1,
  "statements": [
    {
      "item_id": 7,
      "path": "module::function",
      "label": 2,
      "after_statement": "..."
    }
  ]
}
```

Every object has exactly the shown keys. The list uses item-ID/label order.
`path` is the same crate-relative function path as the replacement request.
The sidecar contains no before statements or type facts; Python joins it to
the immutable skeleton metadata by `(item_id, label)`.

Put construction of the schema-version wrapper and pretty JSON in one pure
CLI serialization helper used by the command. It returns the candidate source
bytes unchanged and `serde_json::to_string_pretty` sidecar text without an
appended newline. Unit-test that helper under the `crat-tool` binary target;
do not duplicate the sidecar object shape in command dispatch.

At the `crat-tool` CLI/parser boundary, reject a replace invocation when
`--output` and `--statement-pairs-output` name the same path. Perform this
check before project compilation or either output write.

The two files are scratch outputs from one command, not independently
publishable artifacts. Before invoking the command, PROCTOR's tooling wrapper
clears each exact destination by unlinking a stale regular file or symlink. It
rejects a directory or other node rather than recursively deleting it. After
a zero exit it requires both outputs to be regular, non-symlink files. A
missing or malformed sidecar makes the tool invocation unusable even if a
candidate file exists. Cross-file filesystem atomicity is not assumed.

Update the Python command builder and `CratTools.replace` boundary to accept
both outputs. The replacement request's version-1 JSON shape is unchanged;
the reporting sidecar is an output protocol, not a request field.

### A5.6 Python acceptance and accumulation

Use one scratch sidecar path in `framework.workdir`, overwritten for every
replacement attempt. Add a strict parser for the sidecar that:

- requires schema version 1 and exact object keys;
- requires valid u64 item IDs, u32 labels, nonempty single-line paths and
  nonempty after snippets;
- rejects carriage returns or a trailing newline in an after snippet;
- requires strict item-ID/label ordering and no duplicate keys;
- verifies that every item ID/path belongs to the current SCC replacement
  request; and
- verifies exact key equality with the transformed labels expected for those
  members.

Parse and validate the sidecar after `crat-tool replace` succeeds but before
installing its candidate. A malformed sidecar is a fatal Crat protocol error,
not an LLM repair opportunity, and the candidate is not installed.

Keep the parsed entries local to the current attempt. Install and build the
candidate through the existing source transaction. Only when `cargo build`
succeeds may the stage join those entries to immutable
`statement_pair_metadata` and add them to the run accumulator.

On a failed candidate build:

- restore the previous source exactly as today;
- discard that attempt's parsed sidecar entries;
- retain no before/after data from the failed generation; and
- continue the existing LLM repair loop when the failure is repairable.

Thus a repaired SCC contributes only its final build-accepted canonical
groups. A mechanical SCC still invokes replacement and produces a validated
empty sidecar. Previously accepted SCC entries remain in memory while later
SCCs run, but no final report is published until the whole stage succeeds.

Reject a duplicate `(item_id, label)` in the run accumulator as an internal
protocol failure. Do not use last-write-wins behavior.

### A5.7 Final Markdown artifact

The report is unconditional. Do not add a stage configuration key or alter
`stage.toml`.

Its destination is:

```text
<outputs.artifacts_dir>/statement-pairs.md
```

when `outputs.artifacts_dir` is present, and:

```text
<framework.workdir>/statement-pairs.md
```

for a direct stage invocation without an artifact directory. It is never
placed in the Rust project and is not copied by `_copy_final`.

Resolve this exact final report path during initial boundary validation. Using
the validated absolute output paths, require the report path and
`outputs.rust_project` to be disjoint: they must not be equal, and neither may
be an ancestor or descendant of the other. Reject an overlap before stale
report clearing, output copying, or any other output mutation.

After that validation, clear a stale node only at the exact report path at the
start of the invocation:

- unlink it when it is a regular file or symlink;
- proceed when it does not exist; and
- reject it when it is a directory or another node type.

Do not recursively remove a stale report directory and do not delete any
sibling artifact. This makes a failed rerun unable to expose an earlier
invocation's report as current output.

Render entries after all SCCs have succeeded and sort globally by item ID,
then label. This deliberately differs from leaf-first SCC processing order.
Do not include timestamps, provider responses, generation numbers, or other
nondeterministic run data.

Use this document structure:

````markdown
# Before/After Statement Pairs

This report contains build-accepted local-transformation statement pairs.

## Item 7: <code>module::function</code>

### Statement 2

#### Before

```rust
#[proctor(2)]
<complete before statement>
```

#### After

```rust
#[proctor(2)]
<first canonical after statement>
#[proctor(2)]
<second canonical after statement>
```

#### Pointer variables

| Variable | Origin | Before type | Selected target type | Before type inferred |
| --- | --- | --- | --- | --- |
| <code>pointer</code> | <code>parameter 0</code> | <code>*mut i32</code> | <code>&amp;mut i32</code> | no |
````

Repeat the item heading only when the item ID changes. Use `yes` and `no` for
the inference column. Render parameter origins as `parameter <index>` and
local origins as `local statement <declaration_label>`.

Before rendering a before type or selected target type in a table cell, start
with its source-valid AST/type rendering and collapse each maximal run of
ASCII space, tab, carriage return, line feed, or form feed to one ASCII space,
then trim ASCII spaces from both ends. This is the only single-line type
normalization. It is a report-only spelling operation after type rendering:
it must not change pointer decisions, source or target Rust, or reject a
translation merely because the Rust pretty-printer wrapped a long type.

Use one exact code-value renderer for the normalized types and every other
single-line code value in item headings and table cells: function path,
variable name, and origin. Reject a non-type value containing `\r` or `\n`.
Transform characters in this order:

1. `&` to `&amp;`;
2. `<` to `&lt;`;
3. `>` to `&gt;`;
4. `|` to `&#124;`;
5. `` ` `` to `&#96;`; and
6. `\` to `&#92;`.

Because ampersands are handled first, the ampersands introduced by later
numeric entities are not escaped again. Wrap the result exactly in
`<code>...</code>`. Do not use variable-length inline Markdown code spans.

For each Rust snippet, choose a backtick fence length equal to the greater of
three and one plus the longest consecutive run of backticks anywhere in the
snippet. The opening is exactly that many backticks followed immediately by
`rust`; the closing is exactly that many backticks with no info string. Emit
the nonempty snippet bytes between them, adding exactly one newline after the
opening and one before the closing. Producers and loaders reject a snippet
with a trailing carriage return or newline, so the renderer never introduces
an extra blank line. Rust code fences have no Markdown indentation.

Retain all canonical `#[proctor(N)]` attributes in both before and after
snippets. The visible statement heading identifies the outer label; nested
attributes preserve the overlapping alignment tree.

When `pointer_variables_complete` is false, put this exact warning immediately
after one blank line following the `#### Pointer variables` heading:

```markdown
> **Warning:** Pointer-variable metadata is incomplete because Crat could not
> resolve every possible binding occurrence in this source statement.
```

After the warning, render every resolvable row normally. If the incomplete
entry has rows, put one blank line between the warning and the table. If the
incomplete entry has no rows, put one blank line between the warning and:

```markdown
_No eligible pointer-variable binding could be resolved for this statement._
```

When a complete entry has no in-scope pointer variables, replace the table
with:

```markdown
_No existing source raw-pointer parameter or simple local binding appears in
this statement._
```

When there are no transformed statements in the complete translation, write
this exact complete file:

````markdown
# Before/After Statement Pairs

This report contains build-accepted local-transformation statement pairs.

_No statements required local transformation._
````

Join headings, prose, fences, warnings, and tables with exactly the blank
lines shown above. Put one empty line between consecutive statement entries
and between item groups. End every report, including the empty report, with
exactly one newline.

Render the complete Markdown in memory. Only after all SCCs succeed, copy the
final Rust project and then write a uniquely named sibling temporary file in
the final report directory. Publish it with one atomic replacement of the
exact report path. Keeping the temporary beside the destination is required
because `framework.workdir` and `outputs.artifacts_dir` may be on different
filesystems. Create the sibling temporary with an exclusive standard-library
temporary-file operation, retain its exact returned path, and never derive it
from a broad glob.

Treat final project copy and report publication as one stage-owned cleanup
transaction:

- the Rust output destination is known not to exist before `_copy_final`;
- if `_copy_final` fails after creating any part of that exact destination,
  remove only that newly created destination tree;
- track the exact report temporary created by this invocation;
- if report writing or publication fails, remove that temporary if present,
  remove the exact final report file or symlink if this invocation created
  it, and remove the newly copied Rust output destination; and
- report any cleanup failure together with the primary failure, claim no
  successful output, and never broaden deletion to a parent or sibling path.

For report cleanup, unlink only the recorded temporary and an exact regular
file or symlink at `statement-pairs.md`; treat an unexpected directory or
other node as a cleanup failure rather than deleting it recursively. For the
Rust output, recursively remove only the exact new nonsymlink directory tree
created from the previously nonexistent destination; unlink that exact path
instead if it is a regular file or symlink, without following a symlink.
Treat another node type as a cleanup failure.

After successful atomic report replacement, perform no further fallible
transformation-output mutation inside `run_stage`; it is the last
stage-owned filesystem mutation of the Rust project and report outputs. The
outer stage `main` still writes `stage_output.json` afterward as required by
the stage protocol. The stale report removed at invocation start is not
restored after failure. Do not expose the per-attempt JSON sidecar under
`artifacts_dir`.

The report is an ordinary persistent artifact, not a log. Keep
`StageOutput.logs` limited to actual log files; no stage-envelope or framework
artifact-kind change is needed.

### A5.8 Ownership, non-goals, and documentation

Keep responsibilities at their existing boundaries:

- `crates/tools/src/skeleton.rs` owns compiler-resolved binding identities,
  source/target types, and before-statement metadata;
- the shared preservation/replacement code owns canonical expansion groups;
- the `crat-tool` binary owns only CLI arguments and file serialization;
- local-transformation Python owns strict sidecar loading, build-acceptance
  filtering, cross-SCC accumulation, Markdown rendering, and artifact
  publication; and
- the orchestrator remains unaware of the report's internal format.

Do not:

- implement rule extraction, a rule schema, rule application, or a `rule_set`
  artifact;
- compile the accepted candidate a second time merely to discover inferred
  after-side types;
- include generated bindings or inspect arbitrary after-side expressions;
- parse Rust in Python;
- change `statements_requiring_transformation` or preservation classification;
- change the LLM prompt, prompt version, validator protocol, replacement
  request schema, metrics, repair counts, or Cargo build counts;
- make report generation optional; or
- put diagnostics into the generated Cargo project.

When implementation is complete, reorganize
[prototype-desc.md](prototype-desc.md) so it accurately describes the
statement-pair artifact, its source/target type scope, accepted-attempt
filtering, destination fallback, and explicit absence of rule
extraction/application. Specifically update:

- **End-to-end flow** to include replacement sidecar creation, strict
  pre-install validation, build-success acceptance, final report rendering,
  final project copy, and report publication;
- **Crat preparation and tool boundary** to describe `crat-tool replace`'s
  required second JSON output and the typed replacement result; and
- **Build transaction and reporting** to describe failed-attempt discard,
  completeness warnings, stale-report handling, the artifact/workdir
  destination, deterministic ordering, and final cleanup transaction.

Update the Amendment 5 overview entry from “planned” to completed/historical
wording. Do not merely append a stale implementation note.

### A5.9 Required regression scope

Implement every case in
[amendment-5-test-plan.md](amendment-5-test-plan.md). Update existing exact
function-record JSON, Python helper, `crat-tool` argv, replacement API, fake
tool, and successful-stage artifact expectations superseded by this
amendment.

Do not edit older phase or amendment test-plan documents. The implementation
is complete only when the focused Rust and Python suites, Crat formatting and
Clippy checks, and the full relevant workspaces pass.
