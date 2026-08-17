# Amendment 7 Detailed Plan: `printf` Argument Rules

## 1. Purpose and authority

This amendment adds one deliberately special local transformation: a supported
whole-statement C `printf` call receives a canonical `::std::print!` skeleton,
and reusable rules are learned and applied independently to its consuming
format arguments. It does not add general macro observation or rewriting.

This file is self-contained implementation guidance for a fresh session.
Current tests and implementation remain authoritative if they reveal a factual
discrepancy. Amendment 7 is planning-document terminology only; do not put
planning labels in PROCTOR or Crat code, tests, fixtures, diagnostics,
configuration, commands, or file names.

The exhaustive cases are in
[amendment-7-test-plan.md](amendment-7-test-plan.md). After implementation,
update [prototype-desc.md](prototype-desc.md) through its required documentation
workflow. Do not revise historical plan files or historical sections.

## 2. Goal, ownership, and non-goals

The feature belongs primarily to Crat's `tools` library. Crat owns compiler
identity, format conversion, skeleton construction, macro-template validation,
replacement enforcement, typed observation extraction, synthesis, matching,
and materialization. The Python local-transformation stage continues to treat
observation and rule documents opaquely, but must understand the new statement
disposition for view validation, reporting, and statistics.

In scope:

- only `printf`, identified by compiler identity and a supported prototype;
- only a call that is the complete payload of one labeled semicolon statement;
- one conservative, total C-to-Rust format converter local to `crates/tools`;
- one canonical `::std::print!` template with implicit-order value arguments;
- one printf-specific observation/rule family per consuming argument; and
- one `mechanical` disposition for converted calls with no consuming values.

Out of scope:

- `fprintf`, `sprintf`, `snprintf`, `vprintf`, wide printing, or other I/O;
- a general macro normalizer, macro rule language, or arbitrary macro
  observation support;
- formatter helpers, argument adapters supplied by the converter, positional
  or named fields, dynamic width/precision, or reordered arguments;
- preserving or learning the ignored C return value;
- nonliteral formats, concatenated literals, statics, `concat!`, or data-flow
  recovery of format strings;
- semantic validation beyond the stated supported domain and the existing
  Cargo build; and
- a stage-envelope, artifact-kind, prompt, or public PROCTOR contract change.

## 3. Settled semantic domain

Equivalence is claimed only when all of these assumptions hold:

- the `printf` return value is ignored and stdout writes succeed; differences
  between C error returns and Rust output failure/panic behavior are excluded;
- C formatting uses the C/POSIX locale;
- the source C execution has no undefined behavior, and every vararg has the
  C-defined promoted type required by its conversion specifier;
- every runtime `%s` value consumed by an accepted rule is valid UTF-8;
- supported floating values are finite and C and Rust use compatible rounding;
- every conversion has an explicitly proven standard-Rust mapping; no
  approximate mapping is allowed; and
- if any byte or conversion is unsupported, the whole printf special case is
  rejected, not partially converted.

The converter must not introduce helper types, helper functions, extra value
arguments, or transformations of argument expressions. Argument conversion is
learned separately from accepted LLM output.

Do not add a second C default-argument-promotion validator. Rust compiler types
on the already translated call are recorded as they exist. The semantic-domain
assumption above, exact source context, structural checks, and candidate Cargo
build are the boundary.

## 4. Source-call eligibility

### 4.1 Statement and callable shape

A candidate must satisfy every check below.

1. It is one labeled `StmtKind::Semi` whose complete expression, after only
   parentheses around the complete expression, is one direct call. A tail
   expression, `let` initializer, assignment, return, condition, call argument,
   arithmetic use, or nested call is not eligible.
2. The direct callee resolves to a locally owned foreign item in an exact
   non-unwinding C foreign module. Its effective linked symbol, honoring
   `#[link_name]`, is exactly `printf`.
3. The declaration is C-variadic, has exactly one fixed argument whose
   compiler type is `*const c_char`, and returns compiler type `c_int`.
   Textual spelling, imports, aliases, and `link_name` do not substitute for
   compiler identity; compare normalized target C types, not one platform's
   source aliases.
4. The call has exactly one fixed format argument plus exactly the number of
   value arguments consumed by the parsed format. Extra or missing varargs
   reject the complete special case.
5. After ignoring parentheses, the format argument is a byte-string literal
   followed by one or more casts. The first cast target is exactly
   `*const u8` or the resolved fixed-parameter type `*const c_char`. Every
   later cast is a raw-const-pointer cast whose pointee is only `u8` or the
   resolved `c_char`, and the terminal type is exactly the declaration's fixed
   parameter type. Redundant casts are allowed. Reject raw-mutable pointers,
   every other pointee, pointer/integer or reference casts, `.as_ptr()`,
   arithmetic, transmute, and every other expression.
6. The decoded byte literal has exactly one terminal NUL and no interior NUL.
   The bytes before it must be valid UTF-8 so they can be emitted as one Rust
   string literal.

These checks are shared by skeleton generation, observation extraction, and
printf-rule application. Independently, every subtree whose direct call
resolves to the exact supported printf declaration is opaque to the ordinary
region selector, even when its return is used or its statement/literal/format
is not special-case eligible. Ordinary observation extraction and ordinary
rule application may select neither that call, any descendant, nor any
ancestor containing it. Disjoint sibling regions remain eligible and region
growth stops before the opaque ancestor. Only an eligible whole semicolon
statement enters printf-specific extraction/application. Other foreign calls,
and functions merely spelled `printf` that fail compiler identity/prototype
checks, retain current behavior.

### 4.2 Format parse result

Implement a tools-private total parser/converter, not a reuse of the current
`io_replacer` converter unless the reused code is first made to satisfy every
restriction here. Its result is either one closed value or `Unsupported`; it
must never panic on arbitrary bytes, overflow a decimal accumulator, or leave
an unconsumed suffix.

The successful value contains:

```text
rust_format: String
conversions: [
  {
    source_specifier: String,  // exact decoded `%...conversion` substring
    conversion: semantic class,
  }, ...
]
```

`conversions` contains one entry per consuming conversion in source order.
Literal text and `%%` consume no value. The emitted Rust format has exactly one
implicit-order placeholder per entry and contains no numeric, named, width-
argument, precision-argument, or reordered reference.

Escape literal `{` and `}` as `{{` and `}}`, and render the result through one
deterministic Rust string-literal emitter. The semantic string, rather than its
source token spelling, is authoritative.

## 5. Conservative format conversion

### 5.1 Grammar and universal rejections

Always support ordinary UTF-8 literal text and exact `%%`. Parse the general
shape `%[flags][width][.precision][length]conversion`, then accept only the
matrix below.

Reject the whole format for:

- incomplete or unknown `%` sequences;
- positional syntax such as `%2$d`;
- dynamic width or precision (`*`, including positional `*m$`);
- any mapping requiring a Rust positional or named field;
- a numeric width/precision greater than `i32::MAX` (parse with checked
  arithmetic; accept exactly `2147483647` and reject `2147483648` or a longer
  overflowing digit sequence);
- the space-sign or apostrophe/grouping flag;
- `%n`, `%p`, `%c`, wide strings/chars, and every unlisted conversion; or
- any flag/length/conversion combination without defined C semantics or an
  exact standard-Rust representation in the supported domain.

Repeated accepted flags are semantically redundant and are normalized.
`-` overrides `0`, as in C. Do not preserve an ignored flag in Rust if doing
so changes alignment or sign behavior. Accepted `-` or `0` without a width is
a no-op and is omitted; `+` still controls the sign.

### 5.2 Integer conversions

Accept exactly:

```text
%(hh|h|l|ll|j|z|t)?[diuoxX]
```

with any combination of proven flags/fixed width from the table below, but no
precision. The length modifier remains part of `source_specifier`; it does not
change the Rust placeholder because the learned target argument expression
must perform any needed narrowing or signedness conversion.

| C property | Rust form | Policy |
| --- | --- | --- |
| `d`, `i`, `u` | `{}` | accept |
| `o` | `{:o}` | accept |
| `x`, `X` | `{:x}`, `{:X}` | accept |
| fixed width | width component | accept |
| `-` | explicit `<` alignment when width exists | accept |
| `+` | `+` sign | accept only for signed `d`/`i`; reject for `u`/`o`/`x`/`X` |
| `0` | sign-aware zero padding when width exists | accept |
| `-` with `0` | left alignment, omit zero padding | accept |
| `#` | none | reject: prefixes differ, notably octal and uppercase hex |
| any precision | none | reject: C integer precision is minimum digits |

In the accepted matrix, `-` and `0` without a field width are C no-ops and are
omitted from the Rust field. Focused pure tests must cover that normalization.

### 5.3 String conversion

Accept bare `%s` only: no length modifier, flag, width, or precision. It maps
to `{}`. Width and precision remain unsupported because C counts bytes while
Rust string formatting counts Unicode characters; valid UTF-8 alone does not
make those quantities equal. `%ls` is a wide-string conversion and is rejected.

### 5.4 Floating conversions

Parse `f`, `F`, `e`, `E`, `g`, `G`, `a`, and `A`, with flags, fixed width,
fixed precision, and their possible length modifiers. Accept only `f`, `F`,
`lf`, and `lF` combinations representable by standard Rust formatting; `l` is
a no-op for C printf's double argument:

- no precision means an explicit Rust precision of `.6`;
- fixed precision `.N` maps to Rust `.N`;
- fixed width, `-`, `+`, and `0` map as for finite numeric formatting;
- `-` overrides `0`;
- `#` may be ignored only when the effective precision is greater than zero,
  because a decimal point is then already mandatory; reject `#` with effective
  precision zero; and
- space-sign and apostrophe always reject.

`f` and `F` share one Rust rendering because their C difference is observable
only for nonfinite spellings, which are outside the accepted domain. Reject
`e`/`E` because Rust does not guarantee C's exponent sign and minimum digit
count, `g`/`G` because Rust has no exact C significant-digit/selection rule,
and `a`/`A` because standard Rust formatting has no hexadecimal-float form.
Parse but reject `Lf` and `LF`: the pinned `f128::f128` only implements a
quadmath `%Qg`-based `Display`, not C fixed-decimal long-double formatting, and
its GCC `__float128` representation is not generally the target C ABI's
`long double`. Reject all other floating length modifiers.

Adding a newly proven mapping later requires converter tests for edge values;
it must not be inferred from superficially similar punctuation.

## 6. Skeleton construction and dispositions

### 6.1 Baseline template

Run printf recognition after labels and compiler mappings exist but before the
ordinary transform payload is replaced by `todo!()`. For an eligible converted
call consuming `N > 0` values, emit this complete labeled semicolon statement:

```rust
#[proctor(L)]
::std::print!("<converted>", todo!(), /* exactly N entries */);
```

The baseline disposition remains `transform`; `needs_transformation` follows
the existing recursive rule. This template is not a promise that each `todo!`
has the source argument's type: it is a syntactic slot whose accepted target
expression is validated and then compiled.

An ineligible or unconvertible whole-statement call keeps the current
whole-statement skeleton with one ordinary `todo!()` payload. Its exact printf
subtree is still opaque to ordinary rules. Never emit a partially converted
format or a mixture of known and unknown format arguments.

### 6.2 Mechanical output

An eligible converted call with `N == 0` becomes exactly:

```rust
#[proctor(L)]
::std::print!("<converted>");
```

Add `StatementDispositionKind::Mechanical`, serialized as `mechanical`. It is
canonical and immutable like a fully fixed payload, appears identically in
baseline and applied views, does not set `needs_transformation`, does not
involve a rule, is never sent as LLM work, produces no observation, and cannot
cause applied-to-baseline rule fallback. It is nevertheless included in final
statement statistics and in `statement-pairs.md` with the original printf
statement as Before and canonical print statement as After.

`mechanical` must be threaded through `view.rs`,
`preservation::make_disposition_forest`, `validate_skeleton_view`, cross-view
transition checks, canonical restoration, validator, replacer, Python's strict
loader/projection, report metadata, fake records, and statistics defaults.
Only `transform` counts as LLM work. A fully `mechanical` SCC follows the
existing no-LLM mechanical candidate path; a build failure without a rule
application remains fatal.

### 6.3 Applied view and atomic argument rules

For an eligible call consuming values, attempt printf rules independently for
each source vararg in source order. Match only printf rules whose exact
specifier and exact intrinsic/adjusted source type context agree. Preserve the
implicit argument order.

Commit at statement granularity:

- if every argument has one applicable, materializable rule, emit the same
  canonical macro and format literal with all target expressions installed and
  mark the complete statement `rule_applied`;
- if any argument lacks a rule, is ambiguous only after existing deterministic
  ranking, cannot materialize, or fails structural admission, discard every
  tentative target argument and leave the applied statement byte-equivalent to
  its baseline template with disposition `transform`.

There is no partial macro with a mix of learned expressions and `todo!()`.
Nested labels remain independently governed by existing topology, though an
eligible printf format itself cannot contain a labeled descendant.

The existing whole-SCC behavior remains: applied views are tried first; a
Cargo failure involving any `rule_applied` statement rolls back and switches
the complete SCC permanently to baseline views within the shared repair budget.
`mechanical` alone never activates that fallback.

## 7. Exact structural validation and independent replacement defense

Do not generally allow macros in observation rules or transform regions. Add
one dedicated expected-template path for an outer statement macro generated by
this feature.

For a transform label whose expected skeleton is a printf template, both the
validator and, independently, the replacer must require the returned expansion
group to contain exactly one semicolon statement with:

- an outer macro invocation, not a function call or block wrapper;
- absolute macro path exactly `::std::print`;
- parenthesized macro input;
- an ordinary Rust string literal whose semantic value exactly equals the
  expected converted format;
- only implicit-order placeholders in that literal; and
- exactly the expected number of comma-separated value expressions, with no
  named arguments or extra tokens. Accept and ignore one optional trailing
  comma, which does not change macro semantics.

Use a dedicated token/parser helper capable of parsing top-level Rust
expressions; do not count commas textually. Argument expressions remain LLM
work and may contain nested calls, tuples, blocks, or macros. Macro path,
delimiter, format semantics, and count are fixed. A nested macro in an
argument may pass structural validation and Cargo, but it makes the complete
printf observation ineligible.

`mechanical` and `rule_applied` macro statements are never trusted from
returned text: canonical restoration replaces them from the immutable expected
skeleton. Extend the shared canonicalization sets so validator and replacer
make the same decision, while retaining the replacer's independent check of
any transform-template macro it adopts.

## 8. Printf-specific observation extraction

### 8.1 Source/target recovery

Only build-accepted attempts with remaining transform labels are considered,
as today. In `observation.rs`, give a statement recognized as supported
`printf` exclusive ownership: it either emits one complete printf observation
per consuming argument or emits nothing; it never falls back to ordinary
expression-region extraction.

Re-run the shared eligibility/converter on the labeled source-copy statement.
Require the accepted target group to be exactly the one validated outer
`::std::print!` statement, with the same converted semantic format and the same
argument count. Although structural validation already established this, the
extractor must fail closed rather than align the wrong expansion.

Before compiling the separate observation source, rewrite only that source's
validated target print arguments from `arg` to `{ arg }`. These one-tail block
wrappers are anti-fold barriers: they keep literal and simple user arguments as
distinct compiler-mapped values instead of allowing `format_args!` lowering to
fold them into formatting internals. Do not change the accepted candidate,
canonical statement-pair sidecar, prompt-facing skeleton, or final project.

Use both compiler views deliberately:

- the unexpanded AST identifies the outer statement macro, exact path, format
  token, source label, and source/target statement correspondence;
- clone rustc's expanded AST and map it with the existing expanded-AST/HIR
  support to find the `FormatArgs` expansion belonging to that exact outer
  macro callsite;
- recover the ordered anti-fold blocks from `FormatArgs.arguments`, require
  each to contain exactly its one user tail expression, map that inner
  expression to HIR, and obtain intrinsic and adjusted types; and
- reject zero/multiple unrelated `FormatArgs`, a callsite mismatch, reordered,
  captured, named, or non-user format arguments, or any uncertain mapping.

This is outer-print-only handling. Do not teach the normalized expression AST
or ordinary alignment walker a general macro constructor.

### 8.2 Argument pairing and atomic failure

Pair consuming conversion `i`, source vararg `i`, and expanded target format
argument `i`. The source format parser guarantees implicit order. Require all
three lists to have identical lengths.

For each pair, normalize the complete source and target argument expressions
through the existing closed `Expression` machinery and record the source
intrinsic and adjusted types. Do not record or require a target root/contextual
type: a format slot supplies no such type, and valid transformations may change
the root type (for example pointer-to-`str` `%s` and promoted-`c_int` to narrow
integer `%hhd`). Discover every eligible
raw-pointer anchor occurring anywhere in the complete source argument, in
source occurrence order, and retain the usual source/target anchor types and
binding correspondence. Anchors remain structured metadata, not part of the
exact printf grouping key.

Compare source `TypeTree`s structurally and exactly. If a source context
contains a nominal identity that cannot be compared stably (in particular an
observation-local nominal identity), printf-specific observation/rule handling
is ineligible; do not relate it to a coincidentally numbered identity.

Treat each observation-source-only `{ arg }` block as an alignment boundary:
normalize the inner user expression, not the artificial block and not
compiler-generated formatting wrappers, borrows, match scaffolding, or
expansion internals.

If any source or target argument cannot be mapped, typed, normalized, aligned,
or anchored; if a nested macro occurs in any target argument; or if any
printf-specific invariant fails, emit no printf observations for that labeled
statement. Other labels and functions continue. This statement-atomic rule
prevents learning a misleading subset of a format call.

## 9. Version-1 document model

Keep every schema-version constant at `1`, but make the new arrays required.
Do not use `serde(default)`. Consequently, older version-1 observation and rule
documents without the new arrays are rejected intentionally; no version 2 is
introduced because the current formats have not reached the master branch.

The closed shapes are:

```text
ObservationDocument {
  schema_version: 1,
  observations: Vec<Observation>,
  printf_observations: Vec<PrintfObservation>,
}

RuleDocument {
  schema_version: 1,
  rules: Vec<Rule>,
  printf_rules: Vec<PrintfRule>,
}
```

Each `PrintfObservation` contains:

- `format_specifier`: the exact decoded ASCII `%...conversion` substring;
- complete normalized source and target argument expressions;
- `pointer_anchors` in the existing observation shape/order;
- exact intrinsic and adjusted source `TypeTree`s.

It has no `lhs`: a printf vararg is never assignment-side context. Each
`PrintfRule` mirrors the source/target patterns, usual rule anchors, exact
specifier, and exact intrinsic/adjusted source type context. Pointer anchors
retain both their usual source and target anchor types. There is no target
root or contextual type field. Use the ordinary closed expression/type
grammars; do not add macro or format-expression constructors.

Canonical member order is `format_specifier`, `source_expression`,
`target_expression`, `pointer_anchors`, `source_type`,
`source_adjusted_type` for an observation. A rule uses the same order with
`source_pattern` and `target_pattern` replacing the expression names. The
rule's two exact source fields use closed `TypeTree`, not `RuleTypeTree`, so
local type identities cannot be abstracted out of the exact
grouping/application key. Anchor type fields retain the ordinary rule type
grammar where anti-unification is required.

Validation must enforce the existing expression, identity, anchor, carrier,
type, canonical-variable, ordering, and closed-object invariants for both
families. Additionally, a printf specifier must parse as exactly one consuming
supported conversion with no literal prefix/suffix, and its exact source type
fields must be valid closed `TypeTree`s. Printf rule validation must reject any
target-pattern or target-anchor-type variable not already bound on the source
side; this check applies equally to synthesized and deserialized rules.
Canonical JSON writes
`schema_version`, the ordinary array, then the printf array; empty documents
contain both empty arrays. Merge concatenates each family independently in
input-document/member order and preserves duplicates.

Update `rule/markdown.rs` so pretty printing distinguishes printf argument
rules and shows the exact specifier and exact source type context. Python must
remain opaque to both arrays; only fake bytes/defaults must use the new valid
closed document shape.

## 10. Synthesis, matching, and materialization

Ordinary observations synthesize only ordinary rules with unchanged semantics.
Printf observations synthesize only printf rules; never pair across families.

For each unordered printf-observation pair:

1. require exact string equality of `format_specifier` and exact structural
   `TypeTree` equality of `source_type` and `source_adjusted_type` before
   anti-unification;
2. if that exact context differs, emit no candidate;
3. otherwise use the existing coupled source/target expression
   anti-unification, anchor relations, identity/carrier constraints, canonical
   variable allocation, validation, deduplication, and sort;
4. retain pointer anchors in their usual separate structured semantics; and
5. reject synthesis unless every variable occurring in the target pattern or
   target anchor types is already bound by the source pattern, exact source
   context, resolved identities, or source-side anchor structure; then emit
   only a printf rule.

This is intentionally stricter than ordinary context synthesis: do not
structurally abstract or generalize the printf source type grouping key.

Load ordinary and printf rules into separate indexes. A printf argument is
matched only against the printf index for its exact specifier and exact
intrinsic/adjusted source types. Within that bucket, retain the existing
pattern matching, anchor injectivity/carriers, specificity, substitution cost,
target size, canonical JSON tie-break, target spelling, parse, and structural
admission rules. A candidate miss permits the next ranked printf rule for that
same argument. Statement atomicity is applied only after every argument has a
winner. Materialization does not request a target root/contextual type; all
target variables are already bound, pointer anchors retain their target anchor
types, and the candidate Cargo build checks the actual target expression's
typing and formatting-trait compatibility.

## 11. Reporting and Python orchestration

The stage contract and artifact names remain unchanged.

Update Python `model.py` so the disposition set includes `mechanical`, the
recursive forest/cross-view rules accept only valid transitions, and
`transform_labels` still returns only `transform`. Add a separate report-label
derivation for `transform` plus `mechanical`; do not overload observation
metadata's `transform_labels`.

Update skeleton `statement_pair_metadata`, replacement sidecar generation,
`_load_replacement_statement_pairs`, and `_accept_statement_pairs` so final
reporting covers:

- build-accepted `transform` groups as today; and
- canonical `mechanical` groups even though they were never LLM work.

Observation metadata and extraction continue to list only accepted transform
labels. A mechanical label must never enter `CurrentObservationItem`'s
`transform_labels` or trigger `extract-observations`.

Add `mechanical` to `statistics.json` under `statements`; `total` includes it.
Update `_statistics_json`, `_publish_final_outputs`' exact default JSON, fake
records/tools, artifact transaction tests, and any complete-key assertions.
No new metric, artifact, prompt input, stage configuration, or output-envelope
field is introduced.

## 12. Concrete implementation surfaces

At minimum inspect and update these current owners:

- `proctor/stages/crat/crates/tools/src/skeleton.rs`: eligibility integration,
  baseline macro templates, mechanical classification, per-argument
  application, metadata, target materialization, and focused unit tests;
- `.../tools/src/observation.rs`: shared printf identity helper, source
  eligibility, expanded `FormatArgs` recovery, anti-fold roots, typed
  printf observations, and extraction tests;
- `.../tools/src/rule.rs`: required document arrays, closed validation,
  canonical JSON/merge, family-separated synthesis, matching/indexing/ranking,
  and tests;
- `.../tools/src/rule/markdown.rs`: printf-rule presentation;
- `.../tools/src/view.rs`: `Mechanical`, transform/report label helpers;
- `.../tools/src/preservation.rs`: forest validation, cross-view invariants,
  and canonical restoration;
- `.../tools/src/validator.rs`: dedicated expected print-template parser and
  validation, with tests in `validator/tests.rs`;
- `.../tools/src/item_replacer.rs`: independent template enforcement,
  mechanical restoration, report sidecars, observation-label isolation, and
  tests in `item_replacer/tests.rs`;
- `.../tools/src/lib.rs` and `proctor/stages/crat/src/bin/crat-tool.rs`: expose
  changed library values without adding CLI operations;
- `proctor/stages/local-transformation/model.py`, `protocol.py`, and `stage.py`:
  strict disposition/view projection, report acceptance, valid empty
  observation bytes, and statistics;
- `proctor/tests/test_local_transformation.py`: fake records/tools, protocol,
  scheduling/fallback, publication, report, and statistics regressions; and
- `docs/prototype-desc.md` and the current Amendment 7 entry in
  `docs/prototype-plan.md` after implementation.

Put the converter in one small tools-private module if that is simpler than
keeping it in `skeleton.rs`; do not depend on `io_replacer` helper formatters or
add a cross-crate abstraction used only once.

## 13. Implementation order

1. Add the total parser/converter and its exhaustive pure table tests.
2. Consolidate one compiler-resolved printf/prototype and literal-chain helper
   used by skeleton, extraction, and application; make its exact-prototype
   calls opaque in the shared ordinary region selector before adding any
   printf-specific path.
3. Extend the disposition model, forest validation/restoration, report-label
   metadata, Python loader/projection, statistics, defaults, and fakes.
4. Generate baseline/mechanical templates and enforce dual-view invariants.
5. Add the dedicated validator parser/checks, then duplicate the defense in
   replacement through a shared pure helper where possible without trusting a
   prior validator result.
6. Add required version-1 printf arrays, values, validation, canonical JSON,
   merge, Markdown, and backwards-rejection tests.
7. Recover expanded `FormatArgs`, install anti-fold boundaries, and extract
   statement-atomic typed printf observations with anchors.
8. Add family-separated synthesis, bound-target-variable validation, and
   exact-context indexing without target root types.
9. Apply rules to every argument atomically and integrate the existing SCC
   fallback without new orchestration state.
10. Update reporting, current documentation, and all focused/integration
    regressions in the companion test plan.

## 14. Error, determinism, and compatibility policy

Normal unsupportedness is local and conservative:

- format/call/literal ineligibility leaves the current whole-statement
  transform hole;
- absent or unmaterializable printf rules leave the baseline macro template;
- printf observation ineligibility emits nothing for that statement; and
- an unsupported nested macro does not enable general macro processing.

Malformed closed documents, contradictory skeleton metadata, contradictory
trusted compiler correspondences that current policy treats as fatal,
impossible trusted-template states, validator setup errors, replacement
failures, and post-acceptance extraction failures retain their current
fatal/protocol behavior. An uncertain printf-specific `FormatArgs` recovery is
instead statement-local observation ineligibility as specified in Section 8.
Never silently downgrade corrupted metadata to an ordinary miss.

Determinism follows source item/label/conversion/argument order, existing
canonical expression IDs and rule ordering, input-document/member order for
merge, and sorted item/label order for reports. Hash maps must not define
serialized or selection order.

Compatibility is intentionally limited:

- observation and rule schema versions remain 1;
- older version-1 documents missing either required printf array are rejected;
- stage input/output envelopes, manifest, artifact names, prompt ID/version,
  rule-set artifact handling, and command set do not change; and
- existing valid documents must be regenerated with an empty required printf
  array before use.

## 15. Verification and completion

From `proctor/stages/crat` after Rust source changes:

```bash
cargo fmt
cargo clippy --workspace --all-targets
cargo test -p tools
cargo test --workspace
```

Use targeted `#[allow(clippy::...)]` only for `len_without_is_empty`,
`too_many_arguments`, or `type_complexity` when necessary. Crat tests stay
inside crate modules, do not invoke `crat-tool`, and do not mutate filesystem
state.

From `proctor`:

```bash
uv run pytest tests/test_local_transformation.py
uv run pytest
uv run ruff check .
uv run ruff format --check .
uv run mypy proctor
```

Implementation is complete only when every case in the companion test plan
passes, `docs/prototype-desc.md` describes current behavior proportionately,
and `docs/prototype-plan.md` links both Amendment 7 files without changing
historical summaries.

## 16. Final fixed choices

The required version-1 members are `printf_observations` and `printf_rules`.
Supported fixed-decimal floating conversions use either no length modifier or
`l`; `Lf` and `LF` are parsed but rejected. The exact declaration predicate is
local non-unwinding C ABI, effective linked symbol `printf`, C-variadic,
exactly one fixed `*const c_char` argument, and `c_int` return. These are
implementation requirements, not extension points. Printf rules contain exact
source root TypeTrees and no target root/context type; target variables must
already be source-bound. Exact-prototype printf subtrees are opaque to ordinary
extraction and application. The format-literal cast grammar and `i32::MAX`
numeric cap are closed as specified above. The accepted semantic domain assumes C-defined,
conversion-compatible promoted varargs and no C undefined behavior; no new
promotion validator is part of this work.
