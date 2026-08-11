# Local-Transformation Prototype

## Purpose and authority

This document describes the local-transformation prototype that is currently
implemented in PROCTOR and Crat. It is the entry point for understanding what
the prototype does now.

Current tests, schemas, manifests, and implementation are authoritative when
they disagree with this description. The broader [research plan](research-plan.md)
and [component specification](proctor-spec.md) describe the intended research
system, including behavior that this prototype does not implement.

Keep this document as a cohesive description of the current prototype, not a
changelog. When behavior changes, revise the existing organization and remove
obsolete prose rather than appending milestone-specific details. Keep the tone
and level of detail consistent across sections, without giving recent changes
disproportionate coverage.

The [historical prototype plan](prototype-plan.md) explains how the
implementation was built and later amended. Read it for rationale and task
provenance, then follow its links to a detailed plan only when that history is
relevant.

The main implementation surfaces are:

- the PROCTOR
  [local-transformation stage](../proctor/stages/local-transformation/);
- the stage's focused
  [Python tests](../proctor/tests/test_local_transformation.py);
- Crat's [local-transformation tools](../proctor/stages/crat/crates/tools/src/);
  and
- the thin [`crat-tool` CLI](../proctor/stages/crat/src/bin/crat-tool.rs).

## Prototype boundary

The prototype is a standalone PROCTOR stage that transforms one Cargo project
at a time. Its stage envelope requires one `rust_project`, accepts an optional
read-only `rule_set`, and declares one `rust_project` output. Inputs are
read-only; all work occurs in the stage work directory, and the output
directory must not already exist. A rule set must be a regular nonsymlink file
whose path does not overlap any input, output, work, artifact, usage-log, or
Crat boundary.

The input Cargo manifest must declare `[lib].path` as one regular, root-level
source file. Absolute paths, paths containing `..`, and nested library paths
are rejected. The project may also contain an explicit forwarding binary.

The stage recognizes two configuration keys:

- `crat_dir`, the Crat checkout to build and invoke; and
- `dump_llm_exchanges`, which retains each rendered prompt and provider
  response as a stage artifact.

The stage does not:

- run a test package or establish semantic correctness;
- produce a rule set as a framework output;
- interpret or update `proctor.toml` or its wrapper metadata;
- transform pointer-containing named types or global variable types;
- remove compatibility wrappers after all functions are transformed; or
- checkpoint and resume individual function groups within one stage
  invocation.

Separately from the stage contract, `crat-tool` can synthesize rules from one
or more observation documents and merge multiple observation documents. These
are not PROCTOR stages and do not consume or produce framework artifacts.

An existing `proctor.toml` is copied with the project but otherwise ignored.
The output is required to build, not to pass a behavioral test suite.

## End-to-end flow

For one stage invocation, the implementation:

1. validates the stage envelope, configuration, Cargo layout, and
   non-overlapping filesystem boundaries;
2. builds or reuses release `crat` and `crat-tool` binaries;
3. copies the complete input project to `work/current`;
4. runs ordinary Crat `expand` followed immediately by `unexpand`, with
   `--unexpand-use-print`;
5. validates the optional rule document in Crat and generates immutable dual-
   view skeleton records, applying matching rules when supplied;
6. normalizes target-function safety in the current library source;
7. requires the normalized project to pass `cargo build`;
8. constructs the function graph and a deterministic leaf-first SCC schedule;
9. processes each SCC from its applied view mechanically or with an LLM;
10. structurally validates LLM output when an LLM was used;
11. asks Crat to produce a complete candidate library source and a canonical
    statement-pair sidecar, plus a separate labeled observation source and
    digest-bound correspondence metadata;
12. validates, installs, and builds the candidate transactionally, retaining
    its source and statement pairs only on success;
13. extracts typed expression observations from the labeled source only after
    the candidate build succeeds;
14. asks Crat to merge the accepted observation documents without Python
    parsing or reserialization; and
15. copies the final project to the declared output and publishes the
    statement-pair report and merged `observations.json`.

The initial input copy retains an existing `target/`. Candidate builds also
retain their changes to `target/` after a source rollback. Earlier successful
SCCs remain promoted while later SCCs are processed.

## Crat preparation and tool boundary

Ordinary `crat` owns the initial `expand,unexpand` preparation. Expand cleanup
preserves explicitly declared `[[bin]].path` sources, the root `build.rs`, and
the root `target/`; it removes obsolete Rust source files before writing the
expanded library source. Explicit bin paths are lexically normalized and may
not be absolute or escape the crate root.

`crat-tool` exposes eight local-transformation operations:

- `make-skeleton` compiles the prepared library, optionally loads and applies
  a rule document, and writes JSON item records;
- `validate` parses a validation request and returned Rust snippets without
  compiling a project;
- `normalize-safety` rewrites one Rust source file;
- `replace` compiles the current project for name resolution and writes one
  complete candidate source file, one canonical statement-pair sidecar, a
  separate labeled observation source, and digest-bound correspondence
  metadata; and
- `extract-observations` compiles only the labeled observation source and
  writes a versioned closed observation document;
- `synthesize-rules` validates one or more observation documents and writes a
  closed, canonical rule document;
- `pretty-print-rules` validates one rule document and writes a human-readable
  Markdown bullet list;
- `merge-observations` validates and concatenates ordinary observation
  documents in argument and member order.

Filesystem I/O and command dispatch remain thin CLI responsibilities.
Skeleton generation, validation, preservation, and replacement are in the
Crat `tools` library. SCC scheduling, LLM use, sidecar acceptance, build
transactions, repair, and report rendering remain in the Python stage.

## Skeleton and analysis model

Crat reuses its whole-program pointer analysis to choose target types for
function parameters, returns, and simple local bindings. Skeleton generation
uses the initial analysis decisions before ordinary pointer-rewrite fallback
or demotion.

Tools mode assumes upstream processing has removed real negative pointer
offsets. It skips offset-sign analysis and therefore does not select
slice-cursor representations for conservative negative-offset handling.

Possible target representations include raw pointers, references, optional
references, slices, boxes, optional boxes, boxed slices, and optional boxed
slices. The special two-argument `main_0` target always uses
`argv: &mut [&mut [i8]]`.

Records are emitted in deterministic recursive source order for:

- source-defined free functions other than every function named `main`;
- statics and constants;
- type aliases;
- enums;
- structs; and
- unions.

Each record has a numeric ID, item kind, and crate-relative path. Function
records additionally contain a final name, annotated source, source and target
signatures, direct dependencies, signature dependencies, resolved
foreign-function names, and two complete skeleton views. The `baseline` view
contains the ordinary analysis result. The `applied` view has the same label
topology and signature but includes every statement that was completely fixed
by selected rules. Each view carries its own skeleton, transformation flag,
recursive statement-disposition forest, and statement-pair metadata for its
remaining transform labels.

Dependencies are compiler-resolved, direct rather than transitive, sorted,
and deduplicated. Foreign functions do not become transformable records or
graph dependencies. Their resolved Rust declaration names are recorded only
as advisory prompt context for the function that refers to them.

Prompt-facing function headers omit explicit ABI syntax and `#[no_mangle]`.
Every target function is shown as unsafe. Non-`ref` bindings are displayed as
mutable in both source and target presentations; `ref` and `ref mut` binding
modes remain distinct.

## Synthesized type spelling

Every type syntax synthesized by skeleton generation is rendered for the
module containing the target function.

When a changed explicit pointer type already contains valid source syntax,
Crat retains and shortens that syntax by resolved identity where possible.
Inferred types use semantic rendering. The preferred result is an in-scope
one-segment name; otherwise Crat uses an accessible `crate::...` path for a
local type or a visible absolute path for an external type.

Introduced bare `Option` and `Box` constructors are used only when the ordinary
prelude is enabled and those names resolve to the standard language items.
Shadowing, an unavailable path, or another unspellable synthesized type aborts
the complete skeleton generation operation rather than emitting invalid Rust.

## Rule application

Observation and rule documents are closed, canonical version-1 formats owned
by the Crat tools library. Skeleton generation optionally loads one fixed rule
set and uses the same compiler-resolved expression-region selection as
observation extraction.

Matching uses normalized expressions, pointer anchors, source and target
types, assignment-side context, and resolved identities. Applicable rules are
ranked deterministically, and replacements are materialized with retained
source syntax and scope-aware type spelling. Every selected region in one
statement must be covered; otherwise that statement remains unmodified.

Target-context inference is deliberately narrow and limited to supported
declarations, assignments, direct calls, returns, body tails, and aggregate
fields. Candidates that cannot currently be bound, spelled, parsed, or
admitted structurally are skipped without invalidating the rule document.
Materialization checks syntax and statement shape; the candidate Cargo build
remains authoritative for name resolution and type correctness.

## Statement labels, preservation, and holes

Source and target statements receive matching depth-first numeric
`#[proctor(N)]` labels. A statement disposition is one of:

- `preserve`, meaning Crat has proved that the complete statement subtree
  remains valid under the selected target types;
- `preserve_shell`, meaning Crat has proved the statement's own declaration or
  control shell while nested labeled statement groups remain independently
  classified;
- `transform`, meaning its payload remains work for an LLM; or
- `rule_applied`, meaning rule application replaced the complete statement's
  selected regions atomically and its canonical applied payload must be
  retained.

Preservation is deliberately conservative. Missing AST/HIR mappings, sensitive
pointer-containing types, changed call signatures, unsafe or unresolved
callables, macros, mutable statics, unions, closures, inline assembly, and
other uncertain constructs require transformation. For `preserve_shell`, this
proof applies to the statement's own payload while nested labeled statements
remain independently classified.

A fully preserved parent cannot contain any non-preserved descendant.
Preserved statements retain the canonical view subtree. Preserved-shell and
rule-applied statements retain their canonical payload while recursively
exposing independently classified descendants. Transformed statements retain
their structural role and control shape but replace relevant payloads with
parseable `todo!()` holes. Only `transform` requires LLM work. The two views
must have identical statement labels, parent/child topology, and function
signatures; the baseline cannot contain `rule_applied`.

An ordinary `if` may occur beneath a non-control expression wrapper only in a
restricted C-conditional-like form: it must have an `else`, each branch must
contain exactly one tail expression, and any nested control must recursively
have the same conditional form. Such an expression is opaque to internal
labeling and remains part of its enclosing statement region.

Generation fails atomically for unsupported shapes such as empty statements,
function-local items, non-block match arms, invalid nested control, an AST/HIR
mapping mismatch, or an unspellable synthesized type.

## Structural validation

One LLM response must contain exactly the requested top-level functions and no
other top-level items. Validation compares structure rather than formatting.
It enforces:

- the complete lifetime declaration and its order;
- parameter count, names, and structural types;
- return-type presence and structural type;
- existing binding identity, pattern role, `ref` mode, and explicit local
  types;
- label order, placement, grouping, and nested control structure;
- canonical fully preserved regions and preserved statement shells;
- exact control roles, branch and arm structure, guards, and descendant
  placement; and
- lexical locality of generated temporaries.

One labeled statement may expand to several consecutive statements. New
bindings must use a fresh `proctor_temp_var_<n>` name and remain inside their
expansion group.

Returned functions may not introduce function-local items, explicit unsafe
blocks, unexpected statement or expression attributes, malformed labels, or
temporaries that escape their permitted region.

Malformed requests and inconsistent expected skeletons produce a
`setup_error`. Repairable returned-code violations produce `invalid` with
deterministic, function-oriented diagnostics. A conforming result produces
`valid`.

Preserved output is not trusted. The validator restores preserved groups and
shells from the immutable target skeleton before checking the result, and the
replacer independently performs the same canonical restoration.

## Replacement and compatibility

Before SCC processing, safety normalization recursively makes every
source-defined free function except `main` unsafe. The operation is
idempotent.

Replacement resolves each target by its full crate-relative path. It requires
the exact requested function set and a previously normalized current target.
The accepted implementation keeps the current function's visibility and
non-export metadata while adopting the validated target lifetime declaration,
parameters, return type, and canonicalized body. PROCTOR statement labels are
removed from emitted code.

When parameter or return types change, Crat normally leaves the transformed
implementation at its original path and creates a collision-free same-module
compatibility wrapper with the old signature. Compiler-resolved callers
outside the current SCC are redirected to the wrapper; calls within the SCC,
including recursion, remain direct.

Wrapper generation supports the implemented raw-pointer conversions to and
from references, optional references, slices, boxes, optional boxes, and
selected boxed-slice returns. Unlisted conversions, including boxed-slice
inputs, fail the whole replacement. Slice inputs use the prototype's fixed
length of `1_000_000` and map null to an empty slice; slice returns map empty
to null and nonempty slices to their data pointer.

Export responsibility moves to the wrapper when required. An implementation
`#[no_mangle]` becomes the corresponding wrapper export name, an explicit
`#[export_name]` moves unchanged, and unrelated attributes stay with the
implementation.

A two-argument `main_0` does not receive an ordinary compatibility wrapper.
Instead, Crat mechanically replaces its sibling safe `main` with a forwarding
boundary that constructs mutable argument slices and calls the transformed
`main_0`. A zero-argument `main_0` leaves the existing `main` unchanged.

Replacement is atomic: target resolution, wrapper conversion, call rewriting,
macro safety, and all other checks complete before one candidate source string
is returned. A required call redirect hidden in macro token input is rejected.

## Function scheduling and prompt context

The function graph contains only function records and function-valued direct
dependencies, including direct self edges. Tarjan SCCs are scheduled
leaf-first so callees are transformed before callers; ties are resolved by the
smallest item ID. Duplicate final function names are permitted across SCCs but
are fatal within one SCC.

Every SCC starts from its applied views. A rule-complete SCC proceeds
mechanically; otherwise one LLM request covers every member while rule-applied
regions remain immutable. If a rule-involved candidate fails its Cargo build,
the stage rolls it back, switches the whole SCC permanently to its baseline
views, and continues within the existing repair budget. A mechanical failure
without a rule application remains fatal.

The prompt contains:

- each member's annotated source and target skeleton, ordered by item ID;
- resolved foreign-function names for each member when present;
- direct dependency entries;
- every SCC member signature for a recursive SCC; and
- a breadth-first closure through signature dependencies for value items and
  ordinary dependencies for type items.

Dependency entries are rendered in item-ID order using kind and final name.
Only complete breadth-first depths are admitted. Mandatory context exceeding
100,000 Python characters aborts before an LLM request; optional closure stops
before exceeding the limit.

The chosen views remain immutable prompt input even after earlier SCCs change
the promoted source. Prompting, validation, replacement, and reporting use the
same view for every member.

## LLM response and repair

The stage uses version 1 of the `local_transformation` prompt through PROCTOR's
shared LLM client, prompt library, pricing, and usage tracker. Framework
`context_overflow` behavior is forced to `error`.

The response extractor considers complete line-oriented triple-backtick
blocks, selects the longest block, and chooses the first on equal length.
Missing fenced code is a repairable formatting failure.

The stage permits one initial generation and at most ten repair generations
across applied processing and any baseline fallback. Each repair is a fresh
request containing only the latest failed transformation and diagnostics. A
fallback never returns to the applied views for that SCC.

Missing fenced code, structurally invalid returned Rust, and candidate build
failures are repairable. A build failure involving applied rules first causes
the one-way baseline fallback described above.

Setup or protocol errors, malformed validator output, tool failures,
replacement failures, provider terminal errors, context overflow, initial
normalization/build failure, and mechanical SCC failures without rule
application abort the stage.

## Build transaction and reporting

Crat writes a scratch candidate source. The stage copies the current library
source to a rollback file, atomically installs the candidate, and runs
`cargo build`.

On success, the candidate remains current and the rollback is removed. On a
build failure or builder exception, the previous library source is restored.
Failure to restore is fatal. Only the root library source is rolled back;
Cargo build artifacts remain. Each replacement attempt also writes a scratch
sidecar. The stage validates it before candidate installation and retains its
canonical groups only after a successful build.

The ordinary candidate and statement-pair sidecar are unchanged by observation
collection. After an accepted SCC with remaining transform labels, Crat uses a
separate labeled source and callable correspondence to extract a typed
observation document. PROCTOR retains accepted documents opaquely; failed,
superseded, and rule-complete attempts contribute none.

Success copies the final current project and publishes deterministic
`statement-pairs.md`, `observations.json`, and `statistics.json` artifacts under
`outputs.artifacts_dir`, or under `framework.workdir` when no artifact directory
is present. Ordered by item and statement, the report pairs prompt-facing
source statements with accepted canonical replacements and relevant source and
target pointer types; it warns when binding metadata is incomplete. Stale
artifact files or symlinks are cleared at invocation start, and final copying
and publication are one cleanup transaction. Failure reports no usable
outputs. The report is diagnostic and does not extract or apply reusable rules.

Before final publication, Crat validates and merges the accepted documents in
schedule order, preserving duplicates. The artifact is published even when it
is empty; a merge failure prevents all final outputs. Scratch observation
sources, metadata, and per-SCC documents never enter the output project.

The stage output reports:

- resolved stage configuration;
- distinct provider/model pairs;
- aggregate per-attempt usage and cost when available;
- the prompt ID and version when an LLM was used;
- function and SCC counts;
- logical generation and repair counts;
- structural and compilation failure counts;
- Cargo build count; and
- the local-transformation log.

Provider retries contribute separate usage attempts but do not increment the
logical generation count. Usage accumulated before a terminal failure is
still reported. Optional exchange artifacts retain every prompt and response
by SCC and generation.

## Offline rule synthesis

The Crat tools library derives deterministic candidate expression rules from
compatible pairs in one or more closed version-1 observation documents. It
generalizes normalized source and target trees together while preserving
pointer, type, LHS, and local-identity relationships, then canonicalizes,
deduplicates, and sorts the candidates. The thin `crat-tool synthesize-rules`
command validates its inputs and atomically writes the result; valid inputs
with no candidate rules produce an empty document. Python contains no
observation or rule parser, validator, merger, or synthesizer.

Normalized integer magnitudes use canonical ASCII digits. Rules that are valid
but need target context unavailable to current application remain in the rule
set and are skipped until that context can be inferred.

## Supportedness and further reading

The prototype intentionally supports a restricted Rust input model. Important
excluded constructs include methods and traits, closures, source-written
type/const generics, function pointers and callbacks, function-local items,
explicit unsafe blocks outside the managed executable boundary, and pointer
transformations that require changing named types or globals.

See [unsupported.md](unsupported.md) for the consolidated conceptual input
contract. Current Crat generation, validation, and replacement checks remain
authoritative for mechanically enforced restrictions.

For historical implementation details, continue with
[prototype-plan.md](prototype-plan.md). That overview links the detailed work
plans and their exhaustive historical test plans.
