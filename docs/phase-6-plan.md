# Phase 6 Detailed Plan: Rule Synthesis

This is a prospective implementation plan for the local-transformation
prototype. Reconcile it with the current implementation and tests when work
begins, and do not describe it as implemented in
[prototype-desc.md](prototype-desc.md) until the work is complete.

See the [historical-plan overview](prototype-plan.md#phase-6-rule-synthesis).

## Goal and boundary

Phase 6 synthesizes a deterministic set of reusable candidate rules from one
or more schema-version-1 observation documents. A rule has the judgment form

```text
C |- source_pattern => target_pattern
```

where `C` is the ordered pointer-anchor list plus the source, adjusted source,
target, and adjusted target root types. The rule preserves `C`; only the two
expression trees are generalized.

The initial operation is offline. It reads ordinary `observations.json` files
and writes one rule document. It does not run Cargo or rustc, change a Rust
project, invoke an LLM, apply a rule, rank rules, validate behavior, or join the
PROCTOR artifact pipeline. Its output is a set of syntactically justified but
semantically unvalidated candidates.

Keep the synthesis core callable as a pure Python function so the existing
local-transformation stage can reuse it for online synthesis later. Online
integration, rule application, fallback behavior, and application-time
validation are deferred.

## Ownership and implementation surfaces

Keep compiler-dependent observation production in Crat and put rule synthesis
with the Python local-transformation implementation:

- adjust `proctor/stages/crat/crates/tools/src/observation.rs` only for the
  field-region correction described below;
- add a pure synthesis module under
  `proctor/stages/local-transformation/`, alongside the existing strict
  observation model;
- add a thin standalone Python command in the same directory for input/output
  handling; and
- extend focused Crat and Python tests, with the detailed cases to be specified
  in a later test plan.

Do not add a `crat-tool` rule-synthesis command. The input is already a closed,
resolved JSON tree, so synthesis needs no compiler service. Do not add a
PROCTOR stage: observation sets are not a framework artifact kind, and
cross-run offline extraction does not fit the one-case stage contract. A
future stage wrapper may call the same Python library if the artifact contract
later gains an appropriate input.

The standalone interface is conceptually:

```text
python extract_rules.py --output rules.json OBSERVATIONS...
```

Require at least one input. Each path contains exactly one ordinary observation
document; concatenated JSON and JSONL are unsupported. Reject repeated resolved
input paths and an output path that aliases an input. Parse and validate every
input before atomically publishing the output.

## Observation producer correction for fields

The current region selector stops before a non-pointer-like field whose base
contains the selected pointer operation. Consequently, an accepted enclosing
change such as a dereference removed beneath a field access records only the
base rewrite and omits the resolved field identity.

Change field-base selection to include exactly the immediate enclosing field
expression and then stop. Do not turn the existing field-base boundary into
ordinary recursive growth: that would absorb arbitrary field chains and could
continue through other allowed parents.

Because selected roots are otherwise opaque during source/target alignment,
guard a promoted field root before accepting it:

1. Require both promoted roots to be field expressions.
2. Resolve both fields and require the same field identity under the existing
   source/target correspondence.
3. Promote one immediate field parent only, then perform overlap checks using
   the promoted roots.
4. Dump the complete promoted expressions through the existing observation
   serializer.

The existing field identity already embeds its owning local ADT, and the rule
must preserve that structural owner relationship. Nested fields retain only
the immediate promoted field; an unchanged outer field remains alignment
context.

This changes producer semantics without changing the observation wire shape.
Keep observation schema version 1 for this prototype; existing version-1
observation files remain valid inputs. Update `prototype-desc.md` after
implementation; retain the older detailed plans as history.

## Input preparation and pair enumeration

Use the existing Python observation loader as the authoritative input
validator. Flatten the validated documents in command-line, document, and
observation order.

Group structurally equal parsed observations before synthesis. For each unique
value retain only whether it appeared once or at least twice; exact
multiplicity, weights, provenance, and confidence are unnecessary.

Enumerate:

- each unordered pair of different observation values exactly once; and
- one self-pair for a value marked repeated.

Never self-pair a singleton. This produces the same deduplicated candidate set
as enumerating all distinct observation occurrences, while avoiding repeated
work. Any two equal observation occurrences establish the repeated flag
regardless of their input files. The final rule records no support count.

## Rule terms and variable sorts

Treat normalized expressions and types as a closed, many-sorted term grammar,
not as arbitrary JSON. Constructors, tagged variants, operators, mutability,
list order, list arity, type constructors, primitive types, and external
resolved identities remain structural values.

Rules may contain these variable classes:

- anchor binding identities, written conceptually as `A`;
- ordinary local binding identities, written as `B`;
- opaque local semantic identities, including nominal ADTs and their owned
  fields, written with namespace-specific sorts such as `S` and `F`;
- complete normalized expressions, written as `E`; and
- unsigned integer-literal magnitudes, written as `N`.

Use the corresponding identity sort for the remaining existing local identity
namespaces without adding separate synthesis behavior for each. A local field
term structurally contains both its owner ADT variable and field variable; do
not express ownership as a detached side condition.

All local IDs in observations are observation-local alpha names. Align them as
identity variables rather than treating equal strings from different
observations as one global entity. Repeated use of one identity variable
requires the same resolved entity. Distinct identity variables of the same
namespace must match distinct entities; this injectivity is global rule
semantics and is not serialized as per-rule inequality constraints.

Expression and magnitude variables are not injective. Repeated occurrences of
the same variable require equal substitutions, while two different `E` or `N`
variables may receive equal substitutions.

External identities remain rigid resolved constants. They never become
identity variables. A larger complete-expression disagreement may hide an
external identity only when the complete expression is generalized under the
ordinary expression-variable rules.

## Context compatibility and identity alignment

Before expression anti-unification, compare the two contexts structurally and
build one namespace-aware identity correspondence environment:

1. Require equal anchor counts and preserve anchor-list order.
2. Pair anchor binding IDs by list position and assign anchor identity
   variables.
3. Compare both types of every anchor and the four root type trees using equal
   constructors, mutability, primitive names, arity, and ordering.
4. When corresponding type positions contain local nominal identities, extend
   or check a namespace-preserving bijection and assign identity variables.
5. Require external identities to be exactly equal.

If any structural fact conflicts, skip the pair. Retain the resulting identity
environment for both expressions. This is one correspondence applied
consistently to the complete transformation; do not compare contexts under an
independent alpha-renaming that is discarded before expression synthesis.

## Coupled anti-unification

Synthesize the source first and the target second. Maintain an ordered
disagreement table whose conceptual key is:

```text
(variable_sort, value_from_first_observation, value_from_second_observation)
```

The ordered values matter: a reversed pair is a different disagreement.
Reusing a key reuses its variable and thereby preserves repeated-subterm
relationships.

### Source pattern

Traverse the source expressions together:

1. Preserve equal structural values.
2. Recurse through matching expression constructors and corresponding fields.
3. At corresponding local identity leaves, reuse an exact context-established
   pair or introduce/reuse a namespace-specific identity variable keyed by the
   ordered identity pair. Defer one seed identity participating in several
   pairs to the carrier check instead of silently merging the pairs.
4. For two integer literals that differ only in unsigned magnitude, keep each
   side's fixed literal kind and primitive type and introduce or reuse an `N`
   variable for the ordered magnitude pair. A sign remains ordinary unary
   syntax.
5. For another supported disagreement, generalize the nearest complete
   expression and introduce or reuse an `E` variable for the ordered subtree
   pair.

Do not introduce variables for operators, mutability, types, external
identities, or list fragments. A difference in such a child may be hidden only
by generalizing a complete enclosing expression. Variable-length sequence
patterns are not part of this phase.

### Target pattern

Traverse the target expressions using the completed source disagreement table
and identity environment in lookup-only mode:

1. Preserve equal structure and recurse through matching constructors.
2. Reuse an `E` or `N` variable only when the target has the same ordered,
   same-sort concrete disagreement already recorded by the source.
3. Reuse a local identity variable only when it was already bound by the
   context or source pattern.
4. Never allocate a new target variable or identity correspondence.

If a target disagreement has no source entry, or an explicit target local
identity is unbound, synthesis for that pair produces no rule. Source variables
unused by the target are allowed because a rewrite may discard source syntax.

## Post-synthesis acceptance

After both patterns exist, apply these syntactic acceptance checks:

1. Reject a rule whose complete source pattern is one unconstrained `E`
   variable.
2. For each seed, inspect which rule variable carries every non-anchor local
   identity in the context and source pattern. An explicit identity variable
   established by the context or source is one carrier; each expression
   variable whose concrete seed substitution contains that identity is another
   carrier.
3. Reject when one identity has more than one distinct carrier. Repeated
   occurrences of the same carrier are valid. Thus one identity may not be
   split between an identity variable and an expression variable, between two
   identity variables, or between two expression variables.
4. In particular, reject any expression-variable substitution containing an
   anchor ID. The anchor set comes directly from `pointer_anchors`; no separate
   context-carrier comparison is needed for this special case.

An identity wholly contained in one expression-variable substitution is
allowed when it has no other carrier. The complete expression transports that
identity without losing a cross-boundary relationship.

Do not perform a reconstruction pass in production. Exact reconstruction of
both seed transformations is a correctness property of the coupled
anti-unification implementation and belongs in focused tests.

Retain concrete rules, including those produced by two equal observation
occurrences. Retain rules whose source and target expression patterns are
equal, because their anchor and root target types may still describe a
meaningful pointer-representation change.

## Canonicalization and deduplication

Canonicalize every accepted rule before comparing it with another rule:

1. Traverse `C` in wire order, then the source pattern, then the target pattern.
2. Rename variables independently within each sort by first occurrence.
3. Preserve meaningful list order and every fixed constructor and identity
   relationship.
4. Serialize the canonical semantic rule core deterministically.

Deduplicate only exact canonical rules. Do not perform subsumption, semantic
equivalence, majority selection, or conflict resolution. A general rule and a
more specific rule both remain. Rules with the same context and source pattern
but different targets also remain as separate candidates. Sort the final rules
by their canonical semantic representation so input ordering cannot affect the
output.

## Rule document

Define one closed schema-version-1 rule document containing `schema_version`
and an ordered `rules` array. Each rule stores the source and target pattern,
the ordered pointer anchors, and the four root type trees. Extend the existing
closed observation grammar with explicitly tagged variable nodes so each
variable's sort is unambiguous; a separate variable-declaration list is
unnecessary.

Do not include rule IDs, hashes, input paths, provenance, witnesses,
multiplicity, pair counts, confidence, ranking, status, or synthesis statistics.
Schema version 1 defines both the wire shape and synthesis semantics. Reject
unknown fields and unsupported newer versions. A valid invocation with no
accepted rules succeeds and emits an empty array.

## Errors and determinism

Treat unreadable inputs, malformed JSON, invalid or unsupported observation
documents, repeated/aliased paths, arithmetic/resource failures, and output
publication failures as fatal command errors. A context mismatch, unsupported
pair disagreement, target lookup failure, degenerate source, or carrier
violation is an ordinary pair non-result.

Read and validate all inputs before writing. Publish through a sibling
temporary and rename so failure does not leave a usable partial rule file.
Given the same multiset of validated observation values, output bytes must be
independent of input-file and observation order.

## Correctness claims and deferred work

This phase guarantees only synthesis validity: each emitted pattern is a
closed result of the stated coupled construction, and every target variable is
available from its context or source match. It does not guarantee that applying
the rule preserves behavior.

The current observations come from structurally accepted, build-successful
transformations, not behavioral validation. They omit facts such as inner-node
types, trait properties, callee signatures, aliasing, bounds, and effects.
Compilation alone can accept a semantically wrong transformation, so future
validation and LLM fallback do not retroactively make these candidates sound.

Defer rule matching and application, held-out evaluation, ranking, confidence,
subsumption, online synthesis, PROCTOR `rule_set` integration, compilation or
test validation, rollback, blacklisting, and LLM fallback. A later evaluation
should compare generalized-rule coverage with a concrete ground-rule baseline
rather than treating duplicated observations as independent evidence.

## Documentation and verification handoff

Before implementation, write a separate Phase 6 test plan that derives focused
cases from every invariant above. At minimum, verification must cover strict
input parsing, context identity alignment, each variable sort, ordered
disagreement reuse, lookup-only target synthesis, carrier rejection, field
promotion, exact canonical deduplication, duplicate compression, deterministic
output, and fatal-versus-pair-local failure behavior.

After implementation, run the focused Python local-transformation tests and
the focused Crat observation tests, then the format, lint, type, Clippy, and
broader unit checks required by the affected repositories. Update
`prototype-desc.md` to describe the implemented boundary accurately.
