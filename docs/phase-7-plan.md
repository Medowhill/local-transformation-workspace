# Phase 7 Detailed Plan: Rule Application

This is a prospective implementation plan for the local-transformation
prototype. Reconcile it with current code and tests before implementation, and
update [prototype-desc.md](prototype-desc.md) only after the work is complete.

See the [historical-plan overview](prototype-plan.md#phase-7-rule-application).
No separate Phase 7 test-plan document exists yet.

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

## Skeleton construction and shared region selection

Run pointer analysis once and create one target-typed, pre-hole function AST.
Apply the fixed target signature and recursive local target types before
producing either view, while retaining the source AST/HIR correspondence needed
for region matching.

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

For each visited statement:

1. Select its own regions. Zero regions leaves that statement unchanged.
2. If selected regions overlap, leave the statement unchanged.
3. Find, select, and materialize one rule for every region.
4. Install all disjoint replacements simultaneously into a statement clone.
5. If any region fails or the complete statement is inadmissible, discard all
   tentative applications for that statement.
6. Otherwise mark the statement `rule_applied` in the applied view, regardless
   of whether labeled descendants remain `transform`.

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

Use one direct-call resolver for call arguments and calls encountered by the
place evaluator. For a source-defined function, select its target signature
and apply compiler-resolved generic substitution before reading the parameter
or return type. External signatures remain unchanged. Indirect or unresolved
calls, unsupported methods, ambiguous generics, overloaded syntax, branch
joins, discarded values, and unsupported places have no target-type
requirement and therefore admit no pointer-root rule.

## Rule selection

At rule-load time, alpha-normalize source patterns and precompute pairwise typed
subsumption after standardizing each pair's variables apart. Substitutions must
respect both grammar sorts and the local-only domain of identity metavariables;
a rigid external identity and a local identity metavariable are incomparable.
Equal-specificity groups are source-pattern alpha-equivalence classes. The
`lhs` role, types, and anchors filter applicability first but do not participate
in grouping or specificity.

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

Materialize the selected rule through a compiler-aware AST builder. It reuses
matched source syntax, preserves resolved local identities, resolves and spells
every fixed target identity form admitted by the current rule grammar on a
best-effort basis, and uses the existing type speller for introduced types. A
rule with `lhs: true` must materialize a supported assignment-place expression;
otherwise it is inapplicable.

If the selected target cannot be resolved, spelled, parsed, or admitted by the
current expected-skeleton restrictions, remove that rule and rerun selection
over the remaining candidates. Do not let an unmaterializable preferred rule
hide a usable alternative. Do not add general opaque statement/control support;
target syntax outside current supported skeleton shapes remains inapplicable.

Use one recursive normalized-term size for both substitution cost and target
size. Count each grammar constructor, variable, and typed scalar leaf once;
recurse into compound children; count a scalar identity or integer magnitude as
one typed leaf; and do not count JSON keys or list containers.

## PROCTOR orchestration and fallback

The initial attempt always projects the applied view for every SCC member; a
view with no application is simply equal to baseline. If no `transform`
disposition exists anywhere in the projected SCC, reuse the current mechanical
replacement/build path with no LLM completion or validator call.

If LLM work remains, prompt, validate, replace, check statement pairs, and
collect observations against the same applied projection. Formatting and
structural repairs continue using that projection until a candidate reaches
Cargo build.

When a candidate containing any rule-applied statement fails Cargo build:

1. Roll back the candidate transaction.
2. Keep the failed prompt-level transformation and Cargo diagnostics as repair
   context.
3. Switch every SCC member permanently to its baseline view.
4. Ask the LLM to fill the baseline view, allowing it to replace previously
   rule-applied statements.

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

## Accepted observations and reporting

Rule-applied statements accepted without baseline fallback do not enter
ordinary statement pairs or new learning observations. If fallback reopens
them and an LLM-built baseline candidate succeeds, handle them as ordinary
transformed statements.

Keep successful per-SCC observation files opaque in Python and merge them with
`crat-tool merge-observations` during final publication. Preserve existing
observation ordering and duplicate semantics.

Do not publish rule provenance, rule IDs, rule-selection diagnostics,
rule-application reports, rule-file digests, or new rule metrics. Internal
application metadata required to validate the applied view is protocol state,
not reported provenance. Keep the existing statement-pair report and existing
stage/usage metrics otherwise unchanged.

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
   and synthesis; add thin tool commands and deterministic-format tests.
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

Run the focused tools suites, the full `tools` and PROCTOR local-transformation
tests, then the repository format, lint, type, Clippy, and broader unit checks
required by the affected code.
