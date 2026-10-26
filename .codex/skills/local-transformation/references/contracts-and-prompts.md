# Contracts and Prompts

Paths are relative to the standalone PROCTOR root. Follow the general `$proctor`
skill for envelope evolution and the `$crat` skill for Rust-side schemas and
compiler algorithms. Short Python filenames below refer to
`stages/local-transformation/`.

## Stage envelope and configuration

The stage manifest ID is `local_transformation`. Inputs require `rust_project`
and optionally accept one `rule_set`; the only declared project product is
`rust_project`. `framework.workdir` is required. The input project, rule set,
and framework-owned inputs remain read-only. The project destination must be
nonexistent; report/usage/scratch paths must satisfy `stage.py`'s non-overlap
checks. A rule set must be a regular nonsymlink file outside all input,
output, work, artifact, usage, and configured Crat boundaries.

| Configuration | Current behavior |
| --- | --- |
| `workers` | Positive integer, default 4; controls long-lived SCC workers. One serializes jobs; booleans are rejected. |
| `crat_dir` | Nonempty string, default `../crat`, resolved relative to the stage directory. |
| `dump_llm_exchanges` | Boolean, default `false`; retain rendered prompt/provider response for every generation. |

Unknown keys are errors. Stage settings are distinct from the shared
`framework.llm` provider/model/retry/rate/pricing configuration. Workers force
`context_overflow` to `error` and share one request-rate limiter. The checked-in
`configs/c2rust_crat_local.toml` overrides `workers` and exchange dumping; do not
mistake example settings for manifest defaults.

`ProjectManifest` in `proctor/contracts/manifest.py` owns `proctor.toml`'s general
shape. This stage additionally requires explicit `api_functions` and `wrappers`,
an empty wrapper list, nonempty API names, and no library API named `main`.
The stage preserves the manifest rather than storing accepted-function state
or learned rules in it. Its current API-name checks do not prove ABI retention.

## Frozen records and target decisions

`make-skeleton` produces an ordered JSON array of records. `model.py` strictly
loads the current shape and validates IDs, Rust namespaces/paths, sorted
resolved dependency lists, and function/view consistency. Do not introduce
compatibility fallbacks for earlier within-run record variants.

Records represent source-defined free functions other than `main`, plus statics,
constants, aliases, enums, structs, and unions. Function records carry
annotated source, source/target signatures, direct and signature dependencies,
foreign-function/static names, exact consuming `printf` specifiers, resolved
`proctor_libc` calls, and `baseline`/`applied` views. Foreign declarations are
advisory context rather than graph nodes or transformation targets.

Crat selects parameter, return, and simple local pointer types from the
initial whole-program analysis decisions, before ordinary rewrite demotion.
Tools mode skips offset-sign analysis on the assumption that non-local
processing removed real negative offsets; it never chooses `SliceCursor`.
Possible representations include raw pointers, references/optional references,
slices, boxes/optional boxes, and boxed slices/optional boxed slices. Named
types and globals remain unchanged. Prompt-facing headers omit explicit ABI
and `no_mangle`; actual accepted items retain original metadata. Target headers
are `unsafe` and may introduce named lifetimes; non-`ref` bindings are displayed
as mutable without changing binding identity or `ref`/`ref mut` mode.

The frozen bundle is created in the same compiler session as these records.
The CLI's independent-worker request sends empty `accepted_correspondence`;
other function interfaces are dependencies, not accepted transformations.
Full compiler-side freezing/insertion/assembly contracts belong to `$crat`.

## Labels and dual views

Both views have identical signatures, labels, statement topology, and
`mechanical` payloads. `baseline` contains the analysis result; `applied` adds
complete selected rule replacements. Neither Python nor the LLM may mutate
the immutable view records.

| Disposition | Meaning for the stage |
| --- | --- |
| `preserve` | Complete subtree is canonical and unchanged. |
| `preserve_shell` | Own declaration/control shell is canonical; nested groups retain independent dispositions. |
| `transform` | Payload contains work for the LLM. |
| `rule_applied` | Selected regions were replaced atomically; canonical `applied` payload is retained. |
| `mechanical` | Fixed conversion needs neither rule nor LLM, currently an admitted `printf` statement with no consuming arguments. |

Only `transform` requires generation. A fully preserved parent cannot contain
non-preserved descendants; `baseline` cannot contain `rule_applied`. The Python
loader checks structural cross-view consistency rather than trusting matching
text lengths or labels alone. Source and target statements use depth-first
numeric `#[proctor(N)]` labels, and one original statement may expand into consecutive
siblings carrying its label.

Ordinary expression regions may include the narrowly admitted conditional
expression form. Closures are opaque regions: their internal statements,
parameters, and control receive no independent labels or target local types.
Captured outer bindings remain subject to the surrounding skeleton and
temporary-scope rules. See the supportedness reference for input restrictions.

## Prompt context and output

`protocol.py` renders numeric version 1 of the `local_transformation` prompt.
Every member appears in item-ID order with annotated source and its selected
target skeleton. Context includes all direct dependencies and all member
signatures for a recursive SCC. Further context follows signature dependencies
for value items and ordinary dependencies for type items, breadth-first.
Entries are sorted by item ID and only complete depths are admitted. Mandatory
context over 100,000 Python characters aborts; optional depths stop before the
limit. This is a dependency-context budget, not the provider token limit.

Only current SCC members activate foreign-name, `printf`-format, and resolved
`proctor_libc` guidance. Dependencies rendered as context do not activate it.
`libc_guidance.py` supplies selected safe equivalents; `printf_guidance.py` selects
format-family and adapter advice; scanning references select `xj_scanf` guidance;
foreign `stdin`/`stdout`/`stderr` references select Rust-I/O advice. Resolved
`proctor_libc` advice retains the same callee identity while allowing argument
adaptation. These preferences are prompt guidance, not structural checks.
Do not copy their signature catalogs into this skill.

The prompt requests source behavior on defined inputs, including apparent
bugs and preconditions, exact target signatures/local types, calls through
target dependency signatures, and fully qualified helper paths. Existing
outer binding names/pattern roles remain stable. New bindings outside opaque
closures use fresh `proctor_temp_var_<n>` names and may not escape their one
statement expansion group, including through captures. Output must contain
exactly the requested functions, no other items, no explicit `unsafe` blocks,
and no unexpected statement/expression attributes.

The extractor accepts complete line-oriented triple-backtick blocks, selects
the longest content, and uses the first on a tie. A missing fence is repairable.
Each repair rerenders all member functions and unchanged guidance, including
only the latest failed text and diagnostics. The one-way `applied`-to-`baseline`
switch changes selected views without resetting guidance or the repair budget.

## Structural validation and sidecar acceptance

Crat's parser-only validator checks requested functions, target lifetimes and
types, existing bindings, label order/grouping, exact control roles and nested
structure, preservation, and lexical temporary scope. It restores canonical
preserved groups/shells before checking the returned result. It does not
compile or prove name resolution, typing, behavior, or whole-program safety.
Insertion independently restores canonical payloads before producing source.

Validator status `valid` continues, `invalid` supplies repair diagnostics, and
`setup_error` aborts. Python validates response fields/status relationships
strictly; malformed or inconsistent tool output is not an LLM repair problem.
Mechanical jobs bypass this parser-only validator but still go through Crat
insertion, sidecar checks, candidate installation, and Cargo compilation.

Replacement outputs comprise canonical candidate source, statement-pair JSON,
separate labeled observation source, and replacement metadata. The stage
cross-checks item IDs/full paths, selected-view labels, callable/dependency
correspondence, source stubs, `unfilled_stub_items`, and SHA-256 identities for
the candidate, pairs, observation source, and frozen bundle. Dependency stubs
must not appear as accepted implementations or produce reported observations.
Outputs must be distinct and outside the input projects/frozen inputs; stale,
missing, or nonregular outputs cannot satisfy the operation.

Digest checks cover worker candidate installation/build and retained source
before assembly. A worker Cargo/build script that changes accepted source
bytes causes failure. The final full build has no post-build source-digest
check.

## Observations, rules, and format lifetimes

Crat extracts typed aligned expression observations from accepted `transform`
regions. Scratch source contains labeled transformed implementations,
original-function copies, and original/target-signature dependency stubs;
compiler-resolved correspondence keeps their identities separate. Source
copies/stubs serve observation extraction only and never enter accepted output.
Print observations form a separate family keyed to consuming source formats.
Recovery failures may suppress a statement's print observations as a group.

Python forwards accepted documents unchanged to `merge-observations`. It does
not validate their semantic contents, concatenate JSON arrays, synthesize
rules, or rewrite the input rule set. Offline `synthesize-rules` derives
deterministic candidates from compatible observation pairs; `make-skeleton`
applies one fixed rule set before LLM work. Current matches are provisional
until the candidate builds and may trigger whole-SCC `baseline` fallback.

Keep current formats straight: framework envelopes, reusable observation/rule
documents retain schema version 1, and prompts retain numeric versions for
provenance. Skeleton/frozen/validation/replacement/assembly data exchanged
within one checkout/run use strict unversioned shapes; human diagnostic
reports/statistics do not need format-version fields. Do not add obsolete
variant acceptance to current private protocols.
