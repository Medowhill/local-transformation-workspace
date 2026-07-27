# Amendment 3 Detailed Plan

This is a historical implementation plan. Its substantive text was moved
verbatim from the former consolidated `prototype-plan.md`; imperative and
future-tense wording describes the work assigned at the time. New navigation
notes identify where later work changed an earlier component.

See the [historical overview](prototype-plan.md#amendment-3).
See the [Amendment 3 test plan](amendment-3-test-plan.md).

## Amendment Plan 3: Foreign-function guidance and prompt presentation

This amendment gives the LLM explicit, per-function notice of resolved foreign
function references and removes skeleton-internal presentation details from
ordinary prompt context. It applies after completed Phases 1--4 and Amendment
Plans 1 and 2. It supersedes only the conflicting foreign-function metadata,
function-record key order, skeleton-loading, context-rendering, prompt-
presentation, and prompt-text requirements in this document. All statement-
preservation, SCC scheduling, validation, replacement, repair, transaction,
and compilation behavior remains unchanged.

`amendment-3-test-plan.md` is the exhaustive executable contract for this
amendment. Do not edit any historical phase or amendment test-plan file.

### A3.1 Resolved foreign-function references

For every source function emitted as a `Fn` skeleton record, collect the Rust
declaration identifiers of all foreign functions referenced directly by that
function's compiled body.

Collection is semantic, not textual:

- resolve every function path through rustc;
- include a resolved function when `TyCtxt::is_foreign_item` identifies its
  `DefId` as a foreign item;
- collect the resolved foreign declaration's Rust identifier, even when the
  source reference uses a `use` alias;
- use the Rust declaration identifier rather than an external symbol named by
  `#[link_name]`;
- include a resolved callable reference even when it is not the callee of a
  direct call;
- include references to foreign functions defined through dependency-crate
  metadata as well as foreign declarations in the current crate; and
- traverse the function body only, not the bodies of local callees.

Deduplicate identifiers by their exact string and serialize them in ascending
lexicographic order. Two distinct foreign declarations with the same Rust
identifier therefore produce one name. Repeated calls or references likewise
produce one name.

Do not include:

- foreign statics or foreign types;
- non-foreign functions from `core`, `std`, `alloc`, compiler support crates,
  or third-party crates;
- a source-defined function merely because its header has an explicit ABI such
  as `extern "C"`; or
- a foreign function that is declared but never referenced by the function.

This collection uses the same resolved compiling program that already owns
dependency analysis. Do not infer foreign functions from identifier spellings,
maintain a libc allowlist, parse rendered Rust in Python, or derive the list
from Amendment 2 statement dispositions.

The existing ordinary dependency behavior is unchanged. A foreign reference
does not create an item record, dependency ID, function-graph edge, SCC edge,
or dependency-context declaration. The new name list is guidance attached
only to the referring function.

### A3.2 Function-record JSON

Add this required field to every `Fn` skeleton record:

```json
{
  "foreign_function_names": ["free", "strlen"]
}
```

Emit `foreign_function_names: []` when the function has no resolved foreign
function reference. Do not emit this field for `Static`, `Const`, `TyAlias`,
`Enum`, `Struct`, or `Union` records.

The final function-record serialization key order after Amendment 2 is:

```text
id, path, kind, name, annotated_source, annotated_skeleton,
source_signature, target_signature, needs_transformation,
statements_requiring_transformation, foreign_function_names,
signature_dependencies, dependencies
```

The top-level skeleton JSON remains an unversioned internal artifact produced
and consumed within one local-transformation stage invocation. This additive
pre-release change does not alter a validator or replacement protocol version.

The foreign-name list is independent of Amendment 2 metadata. In particular,
do not enforce an additional Boolean/list invariant between
`foreign_function_names`, `needs_transformation`, and
`statements_requiring_transformation`. Amendment 2 continues to classify a
foreign call or callable reference using its own conservative rules.

### A3.3 Python loading

The Python skeleton model stores `foreign_function_names` as an immutable
sequence of strings. Require the field on every `Fn` record and require:

- a JSON array;
- only nonempty strings;
- no duplicates; and
- strict ascending lexicographic order.

Reject malformed producer output rather than sorting or deduplicating it in
Python. Preserve every name exactly as decoded. Existing item-ID, path,
dependency, statement-disposition, and Rust-text validation remains unchanged.

The field is presentation metadata only. Do not copy it into structural-
validation or item-replacement requests, and do not use it in function-graph,
SCC, or dependency-closure construction.

### A3.4 Prompt item headings and target rendering

Skeleton item IDs remain internal to JSON loading, graph construction,
deterministic ordering, tool requests, usage metadata, and diagnostics. Do not
put an item ID in an ordinary Dependency Context or Transformation Targets
heading.

This does not remove numeric `#[proctor(N)]` statement labels. Those labels are
part of the structural transformation protocol and remain in source,
skeleton, LLM output, validation, and replacement exactly as required by the
completed implementation.

Render every transformation target in ascending member-item-ID order, but
identify it to the LLM only by its final function name:

````text
### Function `<name>`

Foreign function references: `<first>`, `<second>`

Source:
```rust
<annotated_source>
```
Target skeleton:
```rust
<annotated_skeleton>
```
````

The foreign-function line uses the record's existing sorted order and wraps
each identifier in one Markdown code span. Omit the complete line, including
its following blank line, when `foreign_function_names` is empty.

Within-SCC final function names are already required to be unique before an
LLM request, so a target heading never needs a path fallback. Keep the full
record path internally for diagnostics and replacement, but do not render it
in a target heading.

### A3.5 Dependency entry names

Ordinary Dependency Context entries also omit item IDs. Their headings use
the record kind and the final component of the record's crate-relative path:

````text
### Function `<name>`

Source signature:
```rust
<source_signature>
```
Target signature:
```rust
<target_signature>
```
````

````text
### <Static-or-Const> `<name>`

```rust
<declaration>
```
````

````text
### <TyAlias-or-Enum-or-Struct-or-Union> `<name>`

```rust
<definition>
```
````

Always use only that final name. Never put an item ID or any preceding path
component in an ordinary dependency heading. If two dependency records have
the same kind and final name, deliberately render duplicate headings. For
example, records with paths `outer::left::parse` and
`outer::right::parse` both render as:

```text
### Function `parse`
```

Do not add a path-disambiguation algorithm in this prototype. The original
source and the rendered signatures or definitions remain the available
context. If duplicate name-only headings cause a concrete ambiguity in the
future, revise both the prompt context and the corresponding source
presentation together rather than adding paths only to headings.

Continue to order entries internally by item ID and join them with exactly two
newlines. Apply the existing character budget directly to these final
name-only entry strings. The visible omission of IDs and paths does not alter
dependency closure or deterministic ordering.

### A3.6 Prompt sections and exact advisory instruction

The unreleased prompt remains `local_transformation`, version 1, with the same
frontmatter variables and description. Amendment 2's exact complete-statement
paragraph remains in place.

Add this exact requirement after the completed requirement about preserving a
control root and before the prohibition on explicit unsafe blocks:

```text
10. For each listed foreign-function reference, prefer a behavior-equivalent
    safe Rust function or method when one is available; otherwise preserve the
    foreign call.
```

Renumber the existing unsafe-block requirement to 11 and the fenced-output
requirement to 12. Do not add a libc allowlist, mandate replacement, or claim
that every foreign function has a safe equivalent.

Replace the plain `Dependency Context:` and `Transformation Targets:` labels at
the end of the template with Markdown section headings. The exact final
template fragment is:

```jinja
{% if dependency_context %}## Dependency Context

{{ dependency_context }}

{% endif %}## Transformation Targets

{{ transformation_targets }}

{{ repair_context -}}
```

When `dependency_context` is the exact empty string, omit the complete
Dependency Context section. Transformation Targets is always present for an
LLM-backed SCC. The `##` section headings own the `###` item headings described
above. The section headings are outside the 100,000-character dependency-entry
budget.

The prompt still receives the complete original target material on every
repair. Keep Amendment 2's prompt instruction and the existing raw structural
validator and compiler diagnostic behavior unchanged. In particular, this
amendment removes item IDs from authored context headings; it does not sanitize
or reserialize raw validator response text used by a repair.

### A3.7 Advisory behavior and non-enforcement

The foreign-name list is guidance, not a new correctness condition. The LLM
may keep any foreign reference when no behavior-equivalent safe Rust operation
is available. Structural validation, canonical statement restoration, item
replacement, and compilation do not require a listed foreign function to
disappear and do not reject a new or retained foreign call solely because it
is foreign.

This is intentional. Foreign APIs can differ from superficially similar Rust
APIs in allocation ownership, errno behavior, locale behavior, overlap rules,
null-pointer preconditions, integer conversions, or other semantics. The
existing exact-behavior requirement remains stronger than the advisory
preference for safe Rust.

### A3.8 Required regression scope

Implement every case in `amendment-3-test-plan.md`. Update existing Rust and
Python implementation tests whose expected function-record fields, exact JSON,
record helpers, context entries, budgets, target headings, prompt text, or
content hashes are superseded. Do not edit `phase-1-test-plan.md`,
`phase-2-test-plan.md`, `phase-3-test-plan.md`, `phase-4-test-plan.md`, or
`amendment-2-test-plan.md`.

The focused regression suite must cover:

- per-function foreign-name collection, deduplication, and sorting;
- resolved aliases, `#[link_name]`, callable references, and foreign
  declarations loaded from dependency metadata;
- exclusion of foreign statics, foreign types, non-foreign external functions,
  source-defined ABI functions, and unused declarations;
- exact final function-record shape and field ownership;
- strict Python loading of the new string list;
- target rendering with names, no item IDs or paths, and an omitted empty
  foreign-reference line;
- dependency rendering with final names only, including deliberate duplicate
  headings when kinds and names coincide;
- character budgeting over final name-only entry rendering;
- exact `##`/`###` prompt hierarchy and omission of an empty Dependency
  Context section;
- preservation of Amendment 2's prompt paragraph, version 1 metadata, repair
  diagnostics, and statement labels; and
- advisory-only behavior with no validator or replacer protocol change.
