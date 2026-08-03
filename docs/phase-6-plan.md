# Phase 6 Detailed Plan: Rule Synthesis

This is a prospective implementation plan for the local-transformation
prototype. Reconcile it with the current implementation and tests when work
begins, and do not describe it as implemented in
[prototype-desc.md](prototype-desc.md) until the work is complete.

See the [historical-plan overview](prototype-plan.md#phase-6-rule-synthesis)
and the exhaustive [Phase 6 test plan](phase-6-test-plan.md).

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
- add the pure synthesis module
  `proctor/stages/local-transformation/rule_synthesis.py`, alongside the
  existing strict observation model;
- add the thin standalone command
  `proctor/stages/local-transformation/extract_rules.py`; and
- extend the focused Crat and Python tests specified in the Phase 6 test plan.

Extend `model.py` only with the closed rule wire validators and immutable
`RuleDocument` value needed at the Python boundary. Keep pair enumeration,
anti-unification, carrier analysis, canonicalization, and rule sorting in
`rule_synthesis.py`. The core public entry point is:

```python
def synthesize_rules(
    documents: Sequence[ObservationDocument],
) -> RuleDocument:
    ...
```

It performs no filesystem access and does not mutate either its inputs or
their nested dictionaries. Also provide `load_rules(text: str) -> RuleDocument`
and `rules_to_json(document: RuleDocument) -> str` in `model.py`. The loader
makes the closed output grammar and version check executable. The serializer
is the single byte-format authority for the command and tests.

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

`extract_rules.main(argv: list[str] | None = None) -> int` uses `argparse`,
constructing `ArgumentParser(prog="extract_rules.py")` so imported tests and
direct execution have the same help text. Define required
`--output OUTPUT` and positional `OBSERVATIONS` with `nargs="+"`. It
prints exactly one `extract_rules: <message>` line to standard error, returns
`1` for every argument, operational, or validation failure, and returns `0`
only after publication succeeds. Failure prints nothing to standard output
and does not print argparse usage or a traceback. Use a narrow parser error
override (or equivalent) so malformed arguments do not escape as exit status
2. `main(["--help"])` returns `0`, prints argparse's help text to standard
output, prints nothing to standard error, and performs no filesystem access.
A successful synthesis prints nothing to either stream. Resolve every input
and the output with `Path.resolve(strict=False)` for alias comparison before
opening or creating anything. Require every input to be an existing regular,
nonsymlink file, checking the supplied path itself with `lstat` before using
its resolved comparison key. Reject duplicate resolved inputs and any resolved
input/output equality. Do not attempt inode-level hard-link detection in this
prototype.

Read inputs as UTF-8 and call `load_observations` on each complete file. JSONL,
concatenated JSON, trailing non-whitespace, and a UTF-8 decoding failure are
fatal. Use the validated `ObservationDocument` values directly; do not add a
second permissive parser.

## Observation producer correction for fields

The current region selector stops before a non-pointer-like field whose base
contains the selected pointer operation. Consequently, an accepted enclosing
change such as a dereference removed beneath a field access records only the
base rewrite and omits the resolved field identity.

Change `select_region` so a selected root reaching `ParentRole::FieldBase`
includes exactly the immediate enclosing field expression and then stops.
Return both the selected root and a `promoted_field` Boolean. Do not turn the
existing field-base boundary into
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

Implement the guard at the selected-root fast path in `align_expression` (or
an equivalently narrow helper): pass the set of promoted source root IDs in
addition to the set of all opaque selected roots. Before recording the opaque
source-to-target mapping for one of those IDs, require `ExprKind::Field` on
both sides and compare `resolved_field` results with `same_resolved`. An absent
resolution is not equal. Ordinary selected roots retain the existing opaque
mapping behavior. Merge anchors by the promoted root, and run
`regions_overlap` only after promotion and merge. This ordering ensures two
roots that overlap only because one was promoted are skipped.

For `(*pointer).value`, the emitted source expression therefore becomes the
complete `field(unary(deref, pointer), owner::field)` tree and the target is
`field(pointer, same owner::field)`. For
`(*pointer).inner.outer`, selection stops at `.inner`; `.outer` remains
alignment context and is not serialized into that observation. A source and
target field with different resolved owners or members causes the labeled
unit to be skipped, not a fatal extraction error.

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

Use a recursive frozen semantic value (tagged tuples for objects and arrays,
with object members in the grammar's declared order) as the equality and
hashing key. Do not compare `repr(dict)`, Python object identity, or input JSON
bytes. Sort the unique values by a compact, `sort_keys=True` JSON rendering of
their validated semantic value before pair enumeration. This deterministic
orientation is required because disagreement keys are ordered and prevents
command-line or document order from deciding which seed is “first.”

Enumerate:

- each unordered pair of different observation values exactly once; and
- one self-pair for a value marked repeated.

Never self-pair a singleton. This produces the same deduplicated candidate set
as enumerating all distinct observation occurrences, while avoiding repeated
work. Any two equal observation occurrences establish the repeated flag
regardless of their input files. The final rule records no support count.

The exact enumeration over the sorted unique values is `(i, j)` for every
`i < j`, followed by `(i, i)` exactly when value `i` is marked repeated. Pair
iteration order is not itself observable because accepted rules are
canonicalized, deduplicated, and sorted before output, but keeping this order
makes focused tests reproducible.

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

### Exact variable wire form

Every variable is the same closed tagged node:

```json
{"kind": "variable", "sort": "expression", "index": 0}
```

`index` is a non-Boolean unsigned 64-bit JSON integer. `sort` is exactly one
of:

```text
anchor
binding
function
struct
enum
union
field
variant
constant
static
method
expression
integer_magnitude
```

The first eleven sorts are injective identity sorts. `anchor` and `binding`
both correspond to observation `binding` identities, but remain separate
because the ordered `pointer_anchors` list establishes anchors before ordinary
source traversal. `expression` and `integer_magnitude` are non-injective.

The variable node replaces the complete grammar value at its sort's position:

- an `expression` variable is an `Expression` node;
- an `anchor` or `binding` variable is a `ValueIdentity` node when replacing a
  path identity, and replaces the `id` value of a binding pattern when that
  definition is traversed inside a block expression;
- a `function`, `constant`, `static`, or `method` variable is a
  `ValueIdentity` node;
- a `struct`, `enum`, or `union` variable is an `AdtIdentity` node;
- an `integer_magnitude` variable replaces the decimal-string `value` of an
  integer literal while `kind: integer` and `type` stay fixed; and
- a `field` or `variant` variable replaces only the `id` member of a local
  member identity. The surrounding `{"kind":"local","owner":...}` and its
  owner ADT identity remain explicit.

The `id` of every output pointer-anchor entry is an `anchor` variable node.
No concrete observation-local ID such as `<id0>` or `<struct0>` may remain in
an emitted rule. External identities, foreign symbols, primitive names,
operators, mutability values, constructors, member ownership, and list shape
remain ordinary concrete grammar values.

For example, the normalized fragment conceptually written
`value.<Struct0::field0>` becomes:

```json
{
  "kind": "field",
  "base": {"kind": "path", "value": {"kind": "variable", "sort": "binding", "index": 0}},
  "field": {
    "kind": "local",
    "owner": {"kind": "variable", "sort": "struct", "index": 0},
    "id": {"kind": "variable", "sort": "field", "index": 0}
  }
}
```

Extend the existing observation validators by explicit union points rather
than weakening them: observation loading continues to reject every variable
node, while rule loading permits only the sort valid at the current grammar
position. In particular, an expression variable cannot occur as a type or
identity, and a field variable cannot stand in for its owner.

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

An ordinary local-binding `path` may participate in complete-expression
generalization against a structurally different expression. For example, a
binding path paired with a literal may become `E` at that path/literal
position, subject to carrier checks.

Only an un-generalizable resolved identity-leaf conflict is not an `E`
boundary at the bare `path`. In particular, two differing rigid external or
foreign identities, or local identities whose attempted alignment is already
incompatible, propagate their disagreement to the nearest *strictly
enclosing* complete expression. That enclosing expression may become `E`; the
bare path may not. This is why differing callees generalize their enclosing
call, and a differing callee in a root call makes the whole source pattern
`E` and is rejected by the degenerate-source check. If the call is nested, the
larger call-level `E` may be accepted. Other complete expressions, including a
nested call whose argument arity differs, remain normal `E` boundaries.

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

Represent each local identity occurrence internally by `(namespace,
observation_local_id)`. A correspondence entry is an ordered pair of those
keys plus its allocated variable. Context construction enforces both forward
and reverse uniqueness within one namespace. The source-expression traversal
may record more than one ordered pair involving the same seed identity so that
the carrier check can reject inconsistent equality partitions with the stated
reason; it must never silently overwrite, merge, or choose one of them.

Compare context in this exact order: each pointer anchor's ID, source type,
and target type in anchor-list order, then `source_type`,
`source_adjusted_type`, `target_type`, and `target_adjusted_type`. Anchor IDs
are paired solely by anchor position and allocate `anchor` variables. If the
same binding ID later occurs in an expression, lookup the anchor mapping rather
than allocating a `binding` variable. A binding cannot be both an anchor and
an ordinary binding in one rule.

The existing observation loader already rejects target-only binding and local
function IDs. Do not weaken that invariant. Lookup-only target rejection is
still necessary for other validated local namespaces, including a local field
or ADT first appearing in the target, and as defense in depth for all sorts.

## Coupled anti-unification

Synthesize the source first and the target second. Maintain an ordered
disagreement table whose conceptual key is:

```text
(variable_sort, value_from_first_observation, value_from_second_observation)
```

The ordered values matter: a reversed pair is a different disagreement.
Reusing a key reuses its variable and thereby preserves repeated-subterm
relationships.

Use immutable semantic subtrees, not serialized variable nodes, in keys. Keep
one allocation counter per sort. Identity correspondence and the disagreement
table may be separate maps, but a successful target lookup must return the
same already allocated variable object. No target traversal mutates either
map or any counter.

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

Dispatch local identity positions before the generic “equal structural value”
case. Two equal-looking observation-local IDs from a repeated observation are
still converted to the appropriate identity variable. Recurse through an
otherwise equal composite when it can contain local identities; do not retain
the complete dictionary merely because Python equality says it is equal.
Equal rigid scalars and external identities remain concrete, and equal integer
magnitudes remain concrete after their surrounding literal is traversed.

Do not introduce variables for operators, mutability, types, external
identities, or list fragments. A difference in such a child may be hidden only
by generalizing a complete enclosing expression. Variable-length sequence
patterns are not part of this phase.

Within matching integer literal nodes, allocate `N` only when the two literal
types are equal, the magnitudes differ, and both value strings match the exact
ASCII regular expression `0|[1-9][0-9]*`. Do not change the existing
observation loader: it deliberately remains more permissive because Python
`str.isdigit()` accepts leading zeroes and non-ASCII digit characters. Two
equal loader-valid noncanonical strings remain the same concrete literal.
Two unequal loader-valid strings where either is noncanonical follow ordinary
complete-expression disagreement and may become `E`, never `N`.

The `N` disagreement key is the ordered pair of canonical magnitude strings,
deliberately excluding the literal type; this permits the same `N` to relate
an `isize` source magnitude to the corresponding `usize` target magnitude.
Equal canonical magnitudes also remain concrete. A literal-type mismatch
bubbles to `E` at the literal expression.

Implement structural descent with an internal result that distinguishes
`success`, `generalize this expression`, and `reject pair`. Constructor,
operator, mutability, fixed scalar, type, or list-arity mismatches request
generalization at the nearest permitted enclosing expression. An unbound
target variable, context contradiction, or a target disagreement absent from
the source table rejects the pair. The un-generalizable identity-conflict rule
above may request generalization only from its strict parent.

### Target pattern

Immediately after source synthesis, reject if the complete source pattern is
one unconstrained `E` variable. Do this before beginning target traversal, so
a degenerate source cannot allocate work, observe a target lookup failure, or
be classified by a later rejection reason.

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

Target traversal first attempts the same structural descent as source
traversal. At an integer-magnitude difference it performs the `N` lookup. At a
request to generalize a permitted complete expression, it performs one `E`
lookup on the complete ordered target subtree pair. It never falls back from a
missing narrow key to a different or larger source disagreement.

## Post-synthesis acceptance

After both patterns exist, apply these remaining syntactic acceptance checks:

1. For each seed, inspect which rule variable carries every non-anchor local
   identity in the context and source pattern. An explicit identity variable
   established by the context or source is one carrier; each expression
   variable whose concrete seed substitution contains that identity is another
   carrier.
2. Reject when one identity has more than one distinct carrier. Repeated
   occurrences of the same carrier are valid. Thus one identity may not be
   split between an identity variable and an expression variable, between two
   identity variables, or between two expression variables.
3. In particular, reject any expression-variable substitution containing an
   anchor ID. The anchor set comes directly from `pointer_anchors`; no separate
   context-carrier comparison is needed for this special case.

An identity wholly contained in one expression-variable substitution is
allowed when it has no other carrier. The complete expression transports that
identity without losing a cross-boundary relationship.

Compute carriers separately for each seed. Traverse that seed's complete
context and synthesized source pattern together with the seed substitution
record:

- add the identity variable itself as a carrier for every concrete local
  identity substituted at an explicit identity-variable occurrence; and
- for each distinct `E` variable, recursively collect every local identity in
  its concrete seed subtree and add that `E` variable as a carrier.

Use a set, so repeated occurrences of one variable are one carrier. Member
owner and member ID are separate concrete identities and are checked
independently. Ignore rigid external identities and foreign symbols. Reject if
either seed gives any non-anchor identity more than one carrier. For anchors,
reject immediately if any `E` substitution contains a concrete identity in
that seed's `pointer_anchors` set; explicit repeated `A` occurrences are
valid. This per-seed calculation catches unequal identity partitions even
when the other seed is consistent.

Do not perform a reconstruction pass in production. Exact reconstruction of
both seed transformations is a correctness property of the coupled
anti-unification implementation and belongs in focused tests.

The test-only reconstruction helper substitutes each variable with its saved
concrete value for seed one or seed two. It must reconstruct all six context
locations and both expression trees exactly. It also checks identity-sort
injectivity and permits equal substitutions for distinct `E` or `N`
variables. The helper is not imported by `extract_rules.py`.

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

“Wire order” for `C` means each pointer-anchor entry in list order, visiting
`id`, `source_type`, and `target_type`, followed by the four root trees in the
order `source_type`, `source_adjusted_type`, `target_type`, and
`target_adjusted_type`. Then visit `source_pattern` and `target_pattern`.
Within a grammar node, visit fields in the declared order used by the existing
observation validator. Maintain one first-occurrence counter per sort. Thus
`A0`, `B0`, and `E0` are independent indices.

For deduplication and sorting, encode the canonical rule alone with
`json.dumps(rule, sort_keys=True, separators=(",", ":"), ensure_ascii=False)`.
Use that exact string both as the set key and final sort key. Do not use the
pretty output bytes or a hash as semantic identity.

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

The exact document and rule key order is:

```json
{
  "schema_version": 1,
  "rules": [
    {
      "source_pattern": {},
      "target_pattern": {},
      "pointer_anchors": [],
      "source_type": {},
      "source_adjusted_type": {},
      "target_type": {},
      "target_adjusted_type": {}
    }
  ]
}
```

The `{}` placeholders above are grammar nodes, not permitted empty objects.
Every pointer-anchor entry has exactly `id`, `source_type`, and `target_type`
in that order. A rule must have at least one pointer anchor, and anchor
variables must be distinct and appear exactly once in that anchor list. Every
variable used outside the anchor list must have a canonical contiguous index
within its sort. Every target variable must also occur in context or source;
`load_rules` enforces these closed-document invariants in addition to exact
node shapes and position-specific sorts.

`rules_to_json` uses `json.dumps(value, indent=2, ensure_ascii=False)` and adds
exactly one terminal newline. It relies on the declared insertion order above,
not `sort_keys`, for human-readable field order. Empty success is exactly:

```json
{
  "schema_version": 1,
  "rules": []
}
```

including one final newline on disk.

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

After synthesis, validate the constructed `RuleDocument` through the same
rule grammar used by `load_rules`, serialize it completely in memory, and only
then touch the destination. Reject an existing output directory or other
nonregular node. A pre-existing regular file or symlink at the exact output
path may be replaced. Create a uniquely named sibling temporary with exclusive
creation, write UTF-8 bytes, flush and close it, then publish with
`os.replace`. On any failure, remove only the owned temporary and leave a
pre-existing output unchanged. Report cleanup failure together with the
primary error. Filesystem durability (`fsync`) is not part of this prototype.

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

Implement in this dependency order:

1. Correct field-region production and update its focused Crat observation
   tests without changing schema version 1.
2. Add the closed rule values, position-sensitive validator, loader, and
   deterministic serializer to `model.py`.
3. Implement semantic freezing, duplicate compression, context alignment,
   coupled source/target anti-unification, carrier checks, reconstruction test
   support, canonicalization, deduplication, and sorting in
   `rule_synthesis.py`.
4. Add `extract_rules.py` and its path, error, and atomic-publication tests.
5. Run the complete focused test plan, then update `prototype-desc.md` only
   after the implementation is complete.

The Phase 6 test plan covers strict input and output parsing, every variable
sort, context identity alignment, ordered disagreement reuse, lookup-only
target synthesis, carrier rejection, field promotion, exact canonical
deduplication, duplicate compression, deterministic output, and
fatal-versus-pair-local failure behavior. Do not replace its exact normalized
fixtures with Rust-like strings; synthesis consumes the normalized JSON tree.

After implementation, run the focused Python local-transformation tests and
the focused Crat observation tests, then the format, lint, type, Clippy, and
broader unit checks required by the affected repositories. Update
`prototype-desc.md` to describe the implemented boundary accurately.

Run from `proctor/stages/crat`:

```bash
cargo test -p tools observation::tests
cargo test -p tools
cargo test --workspace
cargo fmt
cargo clippy --workspace --all-targets
```

Run from `proctor`:

```bash
uv run pytest tests/test_local_transformation.py
uv run pytest
uv run ruff check stages/local-transformation tests/test_local_transformation.py
uv run ruff format --check stages/local-transformation tests/test_local_transformation.py
uv run mypy proctor
uv run mypy stages/local-transformation/model.py \
  stages/local-transformation/rule_synthesis.py \
  stages/local-transformation/extract_rules.py
```

The new standalone command is tested by importing and calling `main`; default
tests do not spawn it, invoke Crat, run Cargo, or access the network. Crat tests
stay in `observation.rs`, use source-string compiler inputs, do not exercise
`crat-tool`, and do not change filesystem state.
