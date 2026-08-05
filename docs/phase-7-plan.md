# Phase 7 Detailed Plan: Rule Application

This is a prospective implementation plan for the local-transformation
prototype. Reconcile it with current code and tests before implementation, and
update [prototype-desc.md](prototype-desc.md) only after the work is complete.

See the [historical-plan overview](prototype-plan.md#phase-7-rule-application).
The exhaustive verification contract is the
[Phase 7 test plan](phase-7-test-plan.md).

## Reconciled starting point

Implement this work against the current Phase 6 code, not against an older
plan snapshot. At the start of this work:

- Crat's `crates/tools/src/observation.rs` owns compiler-backed observation
  extraction and the Rust observation value types, but Python still owns the
  closed observation/rule validators, rule synthesis, final observation
  concatenation, and the standalone `extract_rules.py` command;
- `make-skeleton` accepts only the project and output path, and each function
  record has one `annotated_skeleton`, one Boolean `needs_transformation`, one
  sorted `statements_requiring_transformation` array, and matching statement
  metadata;
- validation and replacement requests repeat that flat, single-view shape;
- preservation metadata assumes a preserved parent cannot contain an open
  descendant;
- the stage parses every extracted observation document, accumulates
  observations in memory, and serializes the final document itself; and
- `stage.toml` leaves `rule_set` at the manifest default `unused` even though
  the framework contract and orchestrator already support an opaque
  `rule_set` input path.

The implementation must deliberately replace those surfaces. Do not retain a
second Python semantic implementation for compatibility: the prototype is
unreleased, and the revised version-1 documents and internal requests are the
only accepted shapes after this work.

## Goal and boundary

Phase 7 makes a fixed offline rule set optionally available to one
local-transformation run and applies its rules while Crat generates skeletons.
Rules remain fixed across every function and SCC in that run.

The stage continues to validate candidates by compilation only. Behavioral
tests and online rule synthesis remain out of scope. Do not add rule IDs,
provenance, rule-application reports, or new rule-application metrics. Existing
LLM usage and stage metrics retain their current meanings.

The stage manifest declares the existing framework `rule_set` input optional.
No PROCTOR envelope or artifact schema change is required. The stage treats the
rule file as immutable and includes it in its path-overlap checks.

## Ownership and tool boundary

Make `crates/tools` the sole authority for observation and rule documents:

- move the closed observation/rule data models, validation, canonical
  serialization, and Phase 6 synthesis from Python into Rust;
- keep observation extraction, observation merging, rule synthesis, matching,
  selection, and application in `crates/tools`;
- expose only filesystem and argument handling through the thin `crat-tool`
  binary; and
- remove Python parsing and semantic manipulation of observation and rule
  documents from the local-transformation stage.

Revise the unreleased version-1 observation and rule grammars in place to
require the assignment-side context field introduced below. Do not add a
compatibility default: a document using the earlier shape but omitting the
field is malformed. Apart from this context change, migrate the behavior
specified by the [Phase 6 plan](phase-6-plan.md) without redesigning its
anti-unification semantics, ordering, or canonicalization.

Add or extend these tool operations:

- `make-skeleton` accepts an optional rule-document path;
- `synthesize-rules` reads one or more version-1 observation documents and
  atomically writes the deterministic version-1 rule document; and
- `merge-observations` validates and deterministically concatenates per-SCC
  observation documents into the final document. With zero inputs it emits the
  canonical empty version-1 document; otherwise inputs are passed in accepted
  SCC schedule order and observations are copied verbatim with duplicates.

The stage retains only files from build-accepted SCC attempts and asks Crat to
merge them at final publication. Failed and superseded attempts never enter the
merge. The stage passes an optional rule path to `make-skeleton` and otherwise
treats both formats as black boxes.

Remove the superseded Python synthesis command/library and their observation
and rule wire validators after Rust parity is established. Keep Python models
for skeleton, validation, replacement, and other stage-owned protocols.

### Revised document grammar and canonical bytes

The Rust values use `serde` with unknown-field denial and explicit semantic
validation. Preserve every Phase 5/6 closed type, expression, identity,
variable, canonical-index, target-availability, anonymization, and
anti-unification invariant. Add exactly one field to each observation and rule:

```json
{
  "source_expression": {},
  "target_expression": {},
  "pointer_anchors": [],
  "lhs": false,
  "source_type": {},
  "source_adjusted_type": {},
  "target_type": {},
  "target_adjusted_type": {}
}
```

For a rule, the first two keys are `source_pattern` and `target_pattern`; the
remaining order is identical. `lhs` is a required JSON Boolean. It is not a
metavariable position and cannot be generalized. It participates in semantic
equality, hashing, pair compatibility, canonical rule identity,
deduplication, sorting, and pretty serialization. A missing, null, numeric, or
string value is malformed. The revision is intentionally incompatible with
the earlier version-1 shape and does not change `schema_version`.

Canonical pretty serialization uses two-space indentation, UTF-8 without
ASCII escaping, declared field order, and exactly one final newline. The empty
documents remain exactly:

```json
{
  "schema_version": 1,
  "observations": []
}
```

and

```json
{
  "schema_version": 1,
  "rules": []
}
```

Synthesis retains the complete Phase 6 algorithm: semantic duplicate
compression, deterministic unique-value sorting, distinct unordered pairs
plus repeated-value self-pairs, one coupled many-sorted identity environment,
source-first anti-unification, target lookup only, degenerate-source and
carrier rejection, first-occurrence variable canonicalization, exact canonical
deduplication, and final compact-canonical sorting. Compare `lhs` before
constructing the identity environment; unequal values are an ordinary
pair-local non-result. Context traversal for variable canonicalization is
anchors, the four type fields, source pattern, then target pattern; `lhs`
contains no variable and is visited between anchors and types only for the
semantic/canonical rule value.

`merge-observations` does not deserialize and reserialize individual
observations through a lossy intermediary. It fully validates each input,
creates one new canonical document, and appends cloned `Observation` values in
command-line input order and producer order. It preserves duplicates. With no
input paths it emits the canonical empty document.

### Thin-command boundary

Keep the binary a thin filesystem/argument adapter for these operations:

```text
crat-tool make-skeleton --output RECORDS [--rules RULES] PROJECT
crat-tool synthesize-rules --output RULES OBSERVATIONS...
crat-tool merge-observations --output MERGED [OBSERVATIONS...]
```

`synthesize-rules` delegates document validation and synthesis to
`crates/tools`; `merge-observations` delegates validation and ordered merge to
the same library. `make-skeleton` loads the optional rules once before
compiler-backed application. A malformed rule document is a fatal skeleton
generation error. The rule path is read-only and is never copied into the
output project. Omitting it is semantically the same as using a valid empty
rule document, including emission of explicit equal baseline and applied
views. Command-line parsing, path-node policy, and publication mechanics remain
thin binary concerns and are not part of this phase's tools-library test
contract.

## Definitions

- A **baseline view** is the current target skeleton without rule application.
- An **applied view** is generated from the same target type decisions after
  all accepted rule applications. Both views always exist; without accepted
  applications they are equal.
- An **anchor** is an eligible compiler-resolved local raw-pointer binding
  occurrence used to select a source expression region.
- A **region** is the expression root selected from one or more merged anchor
  occurrences by the shared observation/application selector.
- An **assignment-LHS region** is a region whose selected root is exactly the
  complete left-hand expression of a plain assignment. A region strictly
  nested inside that expression is not an assignment-LHS region.
- A rule is **applicable** when one well-sorted substitution satisfies its
  source pattern, anchor and type context, target-type requirement, carrier
  constraints, and target materialization checks.
- Pattern `P` is **at least as specific as** pattern `Q` when, after
  standardizing their variables apart, a well-sorted, domain-respecting
  substitution of `Q`'s variables produces `P` up to metavariable renaming.

Identity metavariables range only over compiler-resolved local identities in
their declared namespace. External and foreign identities remain rigid
constants. In particular, a local function or method metavariable cannot be
substituted with a fixed external function or method identity.

Every function record stores both views explicitly. Do not make the applied
view optional and do not preserve the old single-view skeleton JSON shape; the
prototype has not merged and may change this internal schema directly.

### Closed skeleton-view and request shapes

Introduce one closed `SkeletonView` value with this JSON shape:

```json
{
  "skeleton": "unsafe fn ...",
  "needs_transformation": true,
  "statement_dispositions": [
    {
      "label": 0,
      "disposition": "rule_applied",
      "children": [
        {"label": 1, "disposition": "transform", "children": []}
      ]
    }
  ],
  "statement_pair_metadata": []
}
```

Each function record replaces `annotated_skeleton`,
`needs_transformation`, `statements_requiring_transformation`, and the single
`statement_pair_metadata` array with required `baseline` and `applied`
`SkeletonView` objects. `annotated_source`, source and target signatures,
dependencies, foreign-function names, and all other item-record fields remain
top-level and unchanged. Non-function records are unchanged.

The recursive disposition forest contains every label in the expected
skeleton exactly once. Roots and children preserve lexical statement order;
flattening it depth first yields the skeleton's depth-first label order. Its
parent/child edges exactly match labeled-statement containment in the parsed
skeleton. Every `label` is a non-Boolean `u32`, unknown keys or disposition
strings are invalid, and the three strings are exactly `preserve`, `transform`,
and `rule_applied`.

Enforce these view invariants at Rust production, Python loading, validator
setup, and replacement setup:

- `needs_transformation` equals whether any node recursively has disposition
  `transform`;
- a `preserve` node has only `preserve` descendants;
- the baseline view contains no `rule_applied` node;
- baseline and applied skeletons have the same function identity, signature,
  declarations, label set, label topology, and control topology;
- a baseline `preserve` node remains `preserve` in the applied view;
- every applied `rule_applied` node corresponds to a baseline `transform`
  node, while other baseline `transform` nodes may remain `transform` or have
  independently classified descendants; and
- `statement_pair_metadata` has exactly one entry for every recursively
  `transform` label in depth-first order and no entry for `preserve` or
  `rule_applied`. Existing metadata contents and validation remain unchanged.

The applied skeleton is always serialized independently, even when its bytes,
forest, and metadata equal the baseline. Do not encode aliasing, null, a view
selector, rule IDs, or provenance in a record.

Revise validation and replacement request version 1 in place. Each expected
function/item carries the existing `id`, `name`, and replacement `path` fields
plus exactly one required `view: SkeletonView`; remove their flat `skeleton`,
`needs_transformation`, and `statements_requiring_transformation` fields.
Python projects either the complete baseline or complete applied view into
that field. Crat must not infer or switch views from request contents. This
makes one request internally consistent and keeps fallback an orchestration
decision. Request ordering, transformation text, accepted correspondence, and
response shapes otherwise remain unchanged.

## Recursive statement dispositions

Replace the current binary statement interpretation with view-specific
recursive dispositions:

- `preserve`: the complete statement subtree is canonical;
- `transform`: the statement's own payload is LLM-editable, while labeled
  descendants retain their independent dispositions; and
- `rule_applied`: the statement's rule-transformed own payload and structural
  shell are canonical, while labeled descendants retain their independent
  dispositions.

“Own payload” excludes nested labeled statement groups. This distinction is
required for a rule-applied condition, scrutinee, iterator, or other outer
payload to remain fixed while labeled statements inside the same control remain
open for the LLM.

Represent each view as a complete expected skeleton plus one recursive label
tree carrying these dispositions. Derive `needs_transformation` from the
presence of any `transform` disposition, not from whether an ancestor must be
traversed to reach an open descendant.

Update validation and replacement canonicalization to recurse by disposition:

1. Restore a `preserve` subtree completely from the expected view.
2. For `transform`, validate and retain the returned own payload, then recurse
   into labeled descendants.
3. For `rule_applied`, restore the expected structural shell and own payload,
   then splice recursively validated descendant groups into their labeled
   slots.

The validator and replacer must perform this independently through their shared
preservation machinery. This is narrower than arbitrary expression-span
preservation: rules commit the complete own payload of a labeled statement,
and labeled descendants are the only editable holes inside it.

For alignment, keep the current consecutive expansion-group rule: one returned
label may cover several sibling statements. A `transform` group may retain that
expansion behavior. `preserve` and `rule_applied` each restore exactly the one
canonical expected statement at that label; returned statements in that group
are discarded before descendants are spliced. A `rule_applied` statement must
therefore have the same supported control root and descendant slots as the
expected applied statement. Locate child slots by label topology, not by text
or source spans.

Implement canonicalization as one recursive operation shared by validator and
replacer but invoked independently:

1. Align the returned group for the current expected label and reject missing,
   malformed, reordered, duplicated, nonconsecutive, or wrong-level labels.
2. For `preserve`, clone the complete expected statement and stop; errors and
   generated declarations inside the discarded returned group do not leak.
3. For `transform`, retain the validated returned group's own statements and
   payload, check existing bindings and generated-temporary locality as today,
   then recursively canonicalize each labeled descendant slot.
4. For `rule_applied`, clone the expected applied statement, recursively
   canonicalize its child slots from the returned tree, and reject returned
   changes that alter, remove, duplicate, or make those slots unlocatable.
5. Run the existing complete declaration, supported-control, attribute,
   lifetime, signature, and temporary checks on the resulting function.

The fast path may skip recursive restoration only when every disposition in
the selected view is `transform`. It may not use the old comparison between
total labels and one flat transform-label array.

## Skeleton construction and shared region selection

Run pointer analysis once and create one target-typed, pre-hole function AST.
Apply the fixed target signature and recursive local target types before
producing either view, while retaining the source AST/HIR correspondence needed
for region matching.

Before pointer decisions, rule selection, or partial record construction,
reject a source-defined transformable free function that declares any type or
const generic parameter. Lifetime parameters retain their existing supported
behavior. This makes the prototype's existing generic exclusion explicit and
prevents rule application or direct-call inference from accidentally widening
it. Report one deterministic generation error naming the function and generic
kind; generate no skeleton records. Resolved external generic calls remain
eligible for the narrow rustc-substitution inference described below because
their generic declarations are not source-defined transformable records.

Derive the baseline view from the pre-hole AST using current preservation
classification and holes. Derive the applied view from a separate clone after
rule application. Never attempt to recover source regions from a baseline AST
whose payloads have already become `todo!()`.

Factor the source-side observation selector into shared Crat code. Both
observation extraction and application use exactly the same:

1. expression-tree and parent-role construction;
2. eligible-anchor filtering;
3. region promotion;
4. equal-root merging; and
5. strict ancestor/descendant overlap detection.

The selector accepts an eligible-binding catalog. Seed an anchor only when the
source local raw-pointer binding has a materialized selected target type.
Nested labeled statements are opaque while selecting the enclosing statement,
as they are during observation extraction.

After promotion and equal-root merging, annotate each selected region with a
required Boolean `lhs`. It is `true` exactly for an assignment-LHS region and
`false` everywhere else, including an assignment RHS and a region strictly
inside an LHS. Compute this role from compiler-resolved source AST identity,
not text, and use the same result for observation extraction and application.

Visit every baseline `transform` label recursively in deterministic depth-first
order. Selection for one label treats nested labeled descendants as opaque;
those descendants are then processed independently in their own visits.

Move the existing parent-role tree, anchor occurrence, promotion, merge, and
overlap logic behind one tools-private selector API. Its result for each region
must retain the source AST root identity, ordered merged anchor occurrences,
the compiler-resolved source expression/type information needed for dumping or
matching, and `lhs`. Keep source preorder stable: anchor occurrences are
visited left to right, equal-root merging retains first-root order, and merged
anchors retain first distinct source-binding occurrence.

Split the current undifferentiated assignment parent role into plain-assignment
left and right roles. `lhs` is true only when, after ignoring parentheses, the
selected root is identical to the assignment's complete left child and the
assignment is `ExprKind::Assign`. It is false for the complete RHS, a strict
descendant of either side, and both operands of compound assignment. Field
promotion may make a region become the complete LHS; classify only after final
promotion and equal-root merging. The same Boolean stored in the selector
result is serialized by observation extraction and consumed by application.

Application supplies an eligible-binding catalog built from the fixed initial
pointer decisions. A source binding is eligible only when its compiler type is
a raw pointer, it has a paired parameter/local target decision, and that
decision can be materialized as one semantic target type. Parameters are keyed
by resolved parameter identity; simple locals are keyed by their declaration
label/resolved binding correspondence. An unpaired raw local, an unmaterialized
decision, or a raw-pointer occurrence not resolving to a local never seeds an
anchor. Observation extraction builds the equivalent catalog from accepted
source/target binding correspondence. The two consumers must call the same
selection routine rather than duplicating parent-role policy.

Use a compiler-resolved internal normalized term for selection, matching, and
materialization. Keep resolved local binding identities and `DefId`-based
semantic identities until observation serialization anonymizes them. Do not
round-trip application through observation-local strings such as `<id0>`, and
do not match textual paths.

For each visited statement:

1. Select its own regions. Zero regions leaves that statement unchanged.
2. If selected regions overlap, leave the statement unchanged.
3. Find, select, and materialize one rule for every region.
4. Install all disjoint replacements simultaneously into a statement clone.
5. If any region fails or the complete statement is inadmissible, discard all
   tentative applications for that statement.
6. Otherwise mark the statement `rule_applied` in the applied view, regardless
   of whether labeled descendants remain `transform`.

“Complete statement is inadmissible” includes a failed source AST/HIR mapping,
a replacement root no longer occupying its recorded slot, target syntax that
cannot be constructed or parsed, a declaration/label/control shape rejected by
expected-skeleton construction, or an unsupported expression introduced by a
target. These are statement-local misses for a valid rule document: restore
the unmodified applied clone of that statement and continue to its nested
labels. Do not attempt HIR mapping or resolution for introduced target nodes.
Only global source setup/compiler inconsistency is a fatal generation error.

Apply disjoint replacements by AST node identity or immutable spans, not by
successive edits to shifting text. After all statements are processed,
validate the complete applied function using the same expected-skeleton,
declaration, label-tree, and supported-control checks used by validator and
replacement setup.

## Rule context, applicability, and target-type inference

Extend the Phase 6 context `C` with the region's required `lhs` Boolean. An
observation records the role selected from its source AST. Before
anti-unification, synthesis requires the two observations' `lhs` values to be
equal, just as it requires compatible anchors and root types. The synthesized
rule preserves that value, and canonicalization, equality, deduplication, and
serialization include it. Consequently, observations from complete assignment
LHS regions cannot synthesize rules with observations from any other role.

Match one rule with one many-sorted substitution environment shared across the
source pattern and its complete context. Require:

- exact variable sorts, identity domains, and equal substitutions for repeated
  variables;
- injective distinct identity variables within each semantic namespace;
- exact one-to-one ordered correspondence between rule and region anchors;
- exact anchor source and selected-target types;
- exact region source intrinsic and adjusted types;
- exact `lhs` equality;
- no expression-variable substitution containing a region anchor; and
- no non-anchor local identity split across multiple distinct carriers, where
  an explicit identity variable and each expression variable are carriers.

Anchor and ordinary-binding variables share the concrete local-binding
namespace and are injective across those two sorts. Other identity variables
match only local identities in their corresponding namespace. External and
foreign identities match only the same rigid identity; they cannot instantiate
local function, method, type, or member variables. Expression and integer
magnitude variables are not injective.

After matching, compute carrier sets directly. For each concrete non-anchor
local identity, add every explicit identity variable bound to it and every
distinct expression variable whose substituted subtree contains it. Reject a
match whose carrier set has more than one member, and reject immediately when
an expression-variable substitution contains an anchor.

Match rules structurally against the selected compiler-resolved source term.
Fixed constructors, operators, mutabilities, scalar leaves, external
identities, foreign symbols, list lengths and order must be equal. An integer
magnitude variable binds only the unsigned canonical magnitude leaf of an
integer literal; the literal constructor and primitive type remain fixed. An
expression variable binds one complete expression node. Identity variables
bind only the exact local semantic namespace declared by their sort. The
`anchor` and `binding` sorts both bind resolved local bindings, but anchors are
prebound by ordered region-anchor correspondence and ordinary bindings may not
reuse any anchor concrete identity.

Build one substitution environment in this order: pair ordered anchors and
their source/selected-target types, match the source intrinsic and adjusted
types, match `lhs`, then match the source pattern. Check repeated occurrences
against the same semantic value. Apply namespace injectivity across every
identity variable, with one shared concrete namespace for `anchor` and
`binding`; expression and magnitude variables remain non-injective. Match the
rule's target context only after contextual target inference is available:
for non-pointer-like source roots compare both target type trees, and for a
pointer-like source root compare only target adjusted type as specified below.
No target pattern node participates in source matching.

Compute the region's target-type requirement once and reuse it for every rule:

- when both source root types are non-pointer-like, require the rule's target
  intrinsic and adjusted types to equal the unchanged source pair; and
- when either source root type is pointer-like, infer one contextual target
  adjusted type, compare it with the rule's target adjusted type, and ignore
  the rule's target intrinsic type.

Use one `pointer_like` predicate. It is true exactly for raw-pointer and
reference type trees, the resolved language `Box` ADT, and the resolved
standard `Option` ADT whose contained type is a reference or `Box`. References
and boxes may contain slices; bare slices, arrays, tuples, and unrelated ADTs
are not pointer-like.

Recognize `Box` and `Option` by resolved language-item/diagnostic-item identity,
not spelling or imports. For `Box<T, A>`, the pointee is the first type
argument; retain and compare every normalized argument in rule context. For
`Option<R>`, require exactly the supported representation argument `R` after
normalization; `R` may be a reference or `Box`, including one whose pointee is
a slice. `Option<raw pointer>`, nested `Option`, `Vec`, and a user ADT named
`Box` or `Option` are not pointer-like.

Initially infer contextual target adjusted types only from:

- a simple local initializer's selected target binding type;
- an assignment-LHS region that is exactly a bare path to a pointer-type local
  binding, using that binding's selected target type;
- a plain-assignment RHS using the restricted target-place evaluator described
  below;
- a direct resolved call argument's selected target parameter type;
- the current function's selected return type for returns and body tails; and
- an instantiated struct or variant field type.

The assignment-LHS rule is intentionally narrow. It applies only when the
selected region is the complete LHS and its root resolves directly to the
local binding; it does not cover a dereference, field, index, other projection,
or a region nested inside the LHS. Never infer an LHS requirement from the RHS.
Unsupported LHS regions simply prevent complete rule coverage of that
statement.

For a plain-assignment RHS, evaluate the complete LHS recursively. A local path
uses its selected target binding type and parentheses recurse. A direct,
statically resolved function call uses its instantiated target return type. A
dereference takes the pointee of an evaluated raw pointer, reference, or `Box`;
for a supported `Option` containing a reference or `Box`, first project through
the `Option` and then take the contained representation's pointee. A field uses
the compiler-resolved field type instantiated with the evaluated base ADT
arguments, and an index takes the evaluated array, slice, or reference element
type.

This evaluator computes a type only. In particular, projecting through
`Option` does not choose or materialize an LHS rule. Every selected LHS and RHS
region must still obtain its own applicable rule, and statement atomicity
prevents a successful RHS inference from causing a partial application.
Any unsupported expression, failed resolution, or unsupported representation
makes the RHS requirement unavailable. Do not infer assignment evidence for
compound assignment.

The place evaluator is semantic and never rewrites the place. Parentheses
recurse. A local path consults the selected target-binding catalog; a resolved
direct call consults the direct-call resolver above. Dereference accepts only
raw pointers, references, and resolved `Box`, or resolved `Option` whose one
contained representation is a reference or `Box`; it first unwraps the
`Option` in the type domain and then returns the representation pointee. Field
projection requires a resolved field on a struct/enum ADT and substitutes the
evaluated base's ADT arguments into the field type. Index projection accepts
an array, slice, or reference to array/slice and returns its element. Do not
autoderef arbitrary ADTs, perform trait lookup, join branch types, or invent
runtime unwrapping syntax in this evaluator.

Use one direct-call resolver for call arguments and calls encountered by the
place evaluator. For a supported source-defined function, use its already
selected monomorphic target signature. Source-written type and const generics
remain outside the prototype and must not be admitted through this work. For a
direct resolved external Rust function, use its unchanged signature after
applying rustc's resolved generic arguments; this narrowly supports an
external call such as `consume::<i32>` without adding generic function records,
validation, or replacement. Foreign `extern "C"` signatures are unchanged and
non-generic. Indirect or unresolved calls, unsupported methods, ambiguous or
unrepresentable substitutions, overloaded syntax, branch joins, discarded
values, and unsupported places have no target-type requirement and therefore
admit no pointer-root rule.

Represent inferred requirements semantically and normalize them with the same
closed `TypeTree` conversion used by observations before comparison. Never
compare rendered Rust type text. Erase regions only where the existing
observation grammar does; preserve mutability, constructors, ADT identity and
arguments, tuple/list order, and array length.

The six context producers are exact:

- a labeled simple `let name [: T] = REGION` uses the selected target decision
  for that resolved binding, whether or not the source annotation was inferred;
- a complete bare-local assignment LHS uses that local's selected target type
  and only when the region has `lhs: true`;
- a complete plain-assignment RHS evaluates the complete source LHS under
  target decisions;
- a complete direct-call argument uses the parameter at the compiler-resolved
  argument index;
- a complete explicit return operand and the final expression statement of
  the current function body use the selected function return type; and
- a complete struct/variant field initializer uses the compiler-resolved,
  instantiated field type.

Parentheses are transparent for recognizing a complete region. A nested block,
branch, `if`, `match`, method receiver/argument, semicolon/discarded value, or
other expected-type flow does not gain evidence merely because rustc inferred
one in the source program.

## Rule selection

At rule-load time, alpha-normalize source patterns and precompute pairwise typed
subsumption after standardizing each pair's variables apart. Substitutions must
respect both grammar sorts and the local-only domain of identity metavariables;
a rigid external identity and a local identity metavariable are incomparable.
Equal-specificity groups are source-pattern alpha-equivalence classes. The
`lhs` role, types, and anchors filter applicability first but do not participate
in grouping or specificity.

Rule loading first enforces the closed Phase 6 invariants: canonical contiguous
first-occurrence indices per sort, distinct ordered anchors, no concrete local
IDs, every anchor use declared, and every target variable already available
from context or source. Standardizing variables apart for subsumption creates
internal identities only and does not change the loaded rule or its canonical
tie-break bytes. Subsumption is directional: `P` is at least as specific as
`Q` only if substituting variables of `Q` yields `P`; substitutions may contain
the already standardized variables of `P`, repeated variables must agree, and
identity substitutions obey namespace, local-only domain, and injectivity.

For one region:

1. Collect rules satisfying source/context matching.
2. Partition them into equal-specificity groups.
3. Remove every group with a strictly more-specific applicable group.
4. Among remaining groups, minimize the sum of normalized term sizes bound to
   each distinct source-pattern metavariable, counting each variable once and
   ignoring context-only variables.
5. Flatten the remaining groups and maximize normalized target-pattern node
   count.
6. Break ties by compact canonical target-pattern JSON and then compact
   canonical full-rule JSON.

Materialize the selected rule through a compiler-informed AST builder. It
reuses matched source syntax, retains source identity information, spells every
fixed target identity form admitted by the current rule grammar on a
best-effort basis, and uses the existing type speller for introduced types. A
rule with `lhs: true` must materialize a supported assignment-place expression;
otherwise it is inapplicable.

Expression variables clone their matched source AST subtree; magnitude
variables reuse the matched magnitude in the target literal constructor.
Anchor/binding/function/constant/static/method and nominal/member variables
reuse matched source syntax or a scope-aware spelling chosen from the retained
source identity. Fixed external and foreign identities use their canonical
identity as spelling guidance. Constructors retain their known ADT/variant
relationship, and method variables retain matched receiver/method syntax.
Introduced cast or aggregate types use the existing scope-aware semantic type
speller.

The materializer's acceptance boundary is AST-only: it must construct or parse
one syntactically valid expression and admit that expression under the current
expected-skeleton structural restrictions. Do not map new target nodes to HIR,
re-resolve their paths/methods/members, invoke type checking, or require fixed
and substituted identities to resolve after construction. Retained compiler
identities constrain matching and help choose syntax; candidate Cargo build is
the authority for name resolution and type correctness. A syntax-construction,
type-spelling, parse, supported-place, or supported-skeleton-shape failure is an
ordinary candidate miss.

For `lhs: true`, accept only a Rust place admitted by the current AST and
expected-skeleton restrictions: a local/static path, dereference, field, or
index recursively rooted in a supported place. Reject calls, method calls,
casts, literals, arithmetic, control, ranges, aggregates, and temporaries as
assignment places even if the parser accepts the syntax.

If the selected target cannot be spelled, parsed, or admitted by the current
expected-skeleton restrictions, remove that rule and rerun selection over the
remaining candidates. Do not let an unmaterializable preferred rule hide a
usable alternative. Do not add general opaque statement/control support;
target syntax outside current supported skeleton shapes remains inapplicable.

Use one recursive normalized-term size for both substitution cost and target
size. Count each grammar constructor, variable, and typed scalar leaf once;
recurse into compound children; count a scalar identity or integer magnitude as
one typed leaf; and do not count JSON keys or list containers.

## PROCTOR orchestration and fallback

Change `stage.toml` to declare `rule_set = "optional"` under `[requires]` and
leave products/configuration unchanged. Validate `inputs.rule_set`, when
present, as a regular nonsymlink file before tool building, copying, or stale
output clearing. Add it to the boundary map as a read-only input: it may not
equal, contain, or be contained by the input Rust project, work directory,
output Rust project, artifact directory, statement report, observations
artifact, configured Crat checkout, framework usage log, or another writable
boundary. The Crat checkout is included because tool building writes within
that tree; the usage log is included because the stage/provider appends to it.
Pass the input path to skeleton generation but never copy, rewrite, or digest
the rule file.

The framework-created `stage_input.json` is the audit record and may contain
the absolute `inputs.rule_set` path under the existing envelope contract.
Stage-produced logs, command renderings, output metadata, errors copied into
logs, and LLM exchange artifacts must not repeat that path: render a stable
`<rule-set>` placeholder wherever the optional argument would otherwise be
logged. Do not add the path or a digest to `StageOutput`. Omitting the input
selects the empty-rule behavior.

Update the Python `ItemRecord` loader with a frozen `SkeletonView` and recursive
`StatementDisposition` model. Reject unknown/missing keys and all cross-view
invariant failures described above. Prompt rendering, validation requests,
replacement requests, sidecar validation, observation metadata validation, and
statement-pair acceptance all receive an explicit per-SCC projection map.
Never mix a skeleton from one view with dispositions or metadata from the
other. Dependency context and signatures stay view-independent.

The initial attempt always projects the applied view for every SCC member; a
view with no application is simply equal to baseline. If no `transform`
disposition exists anywhere in the projected SCC, reuse the current mechanical
replacement/build path with no LLM completion or validator call.

If LLM work remains, prompt, validate, replace, check statement pairs, and
collect observations against the same applied projection. Formatting and
structural repairs continue using that projection until a candidate reaches
Cargo build.

Structural validation restores rule-applied AST payloads from the projected
applied skeleton; it does not resolve or type-check those restored nodes. Thus
a mixed SCC can be structurally valid even when a generated rule expression is
not name- or type-correct. Cargo build detects that failure, and the presence
of `rule_applied` in the projection triggers the SCC-wide fallback below.

When a candidate containing any rule-applied statement fails Cargo build:

1. Roll back the candidate transaction.
2. Keep the failed prompt-level transformation and Cargo diagnostics as repair
   context.
3. Switch every SCC member permanently to its baseline view.
4. Ask the LLM to fill the baseline view, allowing it to replace previously
   rule-applied statements.

Define “containing” from the selected applied projection, not from textual
inspection of the candidate: at least one disposition anywhere in any SCC
member is `rule_applied`. The transition is one-way and SCC-wide. Set the
selected projection for every member to baseline before rendering the next
prompt, and never return to applied for that SCC.

If the failed applied candidate followed an LLM generation, use that exact
extracted transformation plus Cargo stdout/stderr as the first baseline repair
context. If it was fully mechanical, use the concatenated applied skeletons in
member-ID order as the failed transformation plus Cargo diagnostics. In both
cases the first baseline completion is repair call 1: increment both existing
`llm_generation_calls` and `repair_calls`, write it in the existing
`generation-01` exchange directory, and allow at most calls 1 through 10.
There is no baseline “initial generation” and no budget reset. An applied LLM
generation before the failure remains generation 0 and does not itself count
as a repair.

Do not switch views for validator setup errors, malformed tool responses,
replacement failures, provider failures, builder exceptions, rollback
failures, or observation failures; retain their current fatal behavior. A
mechanical baseline candidate with no actual rule application also retains its
current fatal build-failure behavior.

Keep the current per-SCC LLM budget. An SCC with an initial LLM generation has
at most ten repair calls total across applied and baseline views, without a
reset. If a fully mechanical applied candidate fails, the first baseline LLM
call is repair call one and at most ten baseline calls are available. Do not
change eager LLM-client construction solely for rule-complete SCCs; consistency
requires only that no completion call is made.

Formatting or validator-invalid failures under the applied view consume repair
calls but do not cause fallback; subsequent repairs remain applied until a
candidate reaches Cargo. Once fallback occurs, all later formatting,
validation, and build repairs use baseline. A build failure with an applied
projection containing no `rule_applied` node is the existing ordinary failure:
repairable if it followed LLM work and fatal if the SCC was mechanical.

## Accepted observations and reporting

Rule-applied statements accepted without baseline fallback do not enter
ordinary statement pairs or new learning observations. If fallback reopens
them and an LLM-built baseline candidate succeeds, handle them as ordinary
transformed statements.

For each replacement request, derive its report/extraction label set from the
selected view's recursively `transform` dispositions. The replacement sidecar
and `current_items[*].transform_labels` must equal that depth-first set exactly.
Thus a `rule_applied` parent contributes neither a pair nor an observation,
while a nested `transform` child still does. The replacer builds its observation
source only for those transform labels and independently restores the selected
view's rule-applied shell around them.

Keep successful per-SCC observation files opaque in Python and merge them with
`crat-tool merge-observations` during final publication. Preserve existing
observation ordering and duplicate semantics.

Replace `RunState.observations` with an ordered list of accepted document
paths. After an LLM-built candidate succeeds, invoke `extract-observations`
only when the selected view has at least one transform label. Do not parse the
result in Python. Move the tool-validated output into a stage-owned
`accepted-observations/` directory under `framework.workdir` using
monotonically increasing schedule-order names, then commit that path together
with statement pairs and callable correspondence. Failed builds, failed
extraction, failed moves, and superseded attempts append nothing. A successful
zero-observation document is still retained if extraction ran.

Mechanical accepted SCCs and accepted SCCs with no transform labels produce no
per-SCC observation file. At finalization call `merge-observations --output`
with retained paths in accepted SCC schedule order. Zero paths must create the
empty document. Treat the merged result as an opaque regular file and include
it in the existing project/report/observation publication cleanup transaction.
Python no longer validates observation or rule JSON or serializes the final
observation document.

Do not publish rule provenance, rule IDs, rule-selection diagnostics,
rule-application reports, rule-file digests, or new rule metrics. Internal
application metadata required to validate the applied view is protocol state,
not reported provenance. Keep the existing statement-pair report and existing
stage/usage metrics otherwise unchanged.

After Rust parity, remove `rule_synthesis.py`, `extract_rules.py`,
`RuleDocument`, `load_rules`, `rules_to_json`, `ObservationDocument`,
`load_observations`, and the closed normalized-tree validators from the Python
stage. Retain Python replacement-metadata and skeleton/request models; they are
separate stage/tool protocols. Remove obsolete Python synthesis tests rather
than leaving two semantic authorities.

## Errors, determinism, and atomicity

A malformed rule document aborts skeleton generation. Failure of an individual
valid rule to match, infer a target type, materialize, or satisfy supported
skeleton structure is an ordinary miss. Failure to cover every selected region
leaves the complete statement on its baseline transformation path.

Rule choice must be independent of rule-document order after the specified
canonical tie-breaks. Observation merging and rule synthesis retain Phase 5/6
ordering, duplicate, canonicalization, and atomic-publication behavior.

Both views are generated once from the prepared project and remain immutable
while SCCs advance. Rule files and observation inputs are never modified.

## Implementation and verification handoff

Implement in this dependency order:

1. Move observation/rule models, serialization, merging, and Phase 6 synthesis
   into `crates/tools`, then add required `lhs` context to version-1 extraction
   and synthesis; add the thin command adapters and test deterministic formats
   through the tools-library API only.
2. Factor shared source-region selection and assignment-LHS classification.
3. Add domain-aware typed matching, LHS and RHS target-type inference,
   specificity selection, and compiler-aware target materialization.
4. Generate explicit baseline/applied views and implement recursive
   `preserve`/`transform`/`rule_applied` validation and canonical restoration.
5. Add optional rule-set stage plumbing and consistent active-view requests.
6. Add build-failure baseline fallback with the existing repair budget.
7. Make observation publication black-box through Crat, then remove the
   superseded Python observation/rule/synthesis surfaces.
8. Update `prototype-desc.md` cohesively after implementation.

Before implementation is considered complete, add focused Rust tests for the
revised version-1 wire grammar, `lhs` extraction and synthesis separation,
shared region selection, local-versus-external identity matching, carrier
rules, LHS and RHS target inference, supported `Option` and direct-call place
evaluation, ranking, LHS materialization, both skeleton views, recursive
dispositions, and validator/replacer restoration. Add focused PROCTOR tests
for optional input plumbing, consistent view selection, mechanical
rule-complete SCCs, bounded fallback and repair, fatal failures, opaque
observation publication, and the absence of new provenance/metrics.

Do not add tests for `crat-tool` command-line parsing, argv construction, help,
path/symlink/alias behavior, output publication, temporary files, rename, or
cleanup. Exercise Rust-owned document validation/serialization, synthesis,
in-memory merging, selection, application, and views directly through
`crates/tools`. Exercise PROCTOR orchestration with fake tool/state callbacks;
path-overlap coverage should target extracted stage boundary logic rather than
real filesystem nodes.

Run the focused tools suites, the full `tools` and PROCTOR local-transformation
tests, then the repository format, lint, type, Clippy, and broader unit checks
required by the affected code.
