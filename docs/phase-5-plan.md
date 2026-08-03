# Phase 5 Detailed Plan: Validated Expression Observation Collection

This is a prospective implementation plan for the local-transformation
prototype. Reconcile it with the current implementation and tests when work
begins, and do not describe it as implemented in
[prototype-desc.md](prototype-desc.md) until the work is complete.

See the
[historical-plan overview](prototype-plan.md#phase-5-validated-expression-observation-collection-planned)
and the exhaustive [Phase 5 test plan](phase-5-test-plan.md).

## Goal and boundary

Phase 5 collects concrete, compiler-checked expression observations from
build-accepted SCC replacements. One observation maps a source expression
region rooted at a raw-pointer binding occurrence to the expression at the
same structural position in the accepted target, and records the source and
target type facts needed by future rule work. An observation is evidence, not
a rule.

Extraction runs only after Python has installed the ordinary candidate and its
Cargo build has succeeded. Failed and superseded attempts contribute nothing.
Identical concrete observations are deliberately retained: repetition is
evidence frequency. This phase does not synthesize, generalize, rank, select,
persist, or apply rules. It does not change pointer decisions, prompts,
validation, SCC scheduling, repair policy, generated project behavior, or the
contents and schema of `statement-pairs.md` and its scratch sidecar.

## Ownership and implementation surfaces

Begin with the current implementation and focused tests. The expected change
surfaces are:

- `proctor/stages/crat/crates/tools/src/item_replacer.rs`, for one canonical
  replacement calculation, source copies, observation source, and correspondence;
- a new `proctor/stages/crat/crates/tools/src/observation.rs`, exported by
  `lib.rs`, for the specialized compiler path, alignment, closed wire types,
  and extraction;
- `proctor/stages/crat/src/bin/crat-tool.rs`, for thin file/JSON handling and
  the new `extract-observations` command;
- `proctor/stages/local-transformation/{model.py,protocol.py,tooling.py,stage.py}`,
  for strict protocol values, post-build invocation, accumulation, and
  publication; and
- focused Crat tools/CLI tests and
  `proctor/tests/test_local_transformation.py`.

Keep all Rust parsing, rustc compiler queries, AST rewriting, correspondence,
alignment, anonymization, and serialization types in `crates/tools`. Python
must not parse or rewrite Rust. Keep the specialized compiler runner private to
the tools crate initially; do not generalize `crates/utils` for this one use.

## Settled version-1 file protocols

All protocols remain schema version 1. The prototype has not merged, so Phase
5 extends the prospective version-1 contracts instead of landing a version 2.
Every JSON object uses the exact keys specified here, rejects unknown keys, and
uses JSON integers only for the stated unsigned range; a Boolean is never an
integer. JSON producers pretty-print with two-space indentation and no
producer-dependent map order. Scratch JSON may omit a terminal newline;
`observations.json` always has one terminal newline.

### Replacement request

The exact replacement request becomes:

```json
{
  "schema_version": 1,
  "items": [
    {
      "id": 7,
      "path": "module::read",
      "name": "read",
      "skeleton": "unsafe fn read() { /* canonical labeled target */ }",
      "needs_transformation": true,
      "statements_requiring_transformation": [0]
    }
  ],
  "transformation": "unsafe fn read() { /* accepted labeled target */ }",
  "accepted_correspondence": []
}
```

The request preserves caller-provided item order; replacement never sorts it.
`statements_requiring_transformation` remains in canonical increasing label
order. Existing item fields retain their current meanings.
`accepted_correspondence` is the exact ordered tuple retained from earlier
accepted SCCs, and each record has this closed shape:

```json
{
  "item_id": 3,
  "logical_path": "module::callee",
  "implementation_path": "module::callee",
  "wrapper_path": "module::__proctor_wrapper_callee"
}
```

Paths are canonical crate-relative paths without a leading `crate::` and use
`::` between valid identifier segments. `wrapper_path` is either such a string
or JSON `null`. Across accepted and current records, item IDs are unique;
logical paths are unique among logical paths; implementation paths are unique
among implementation paths; and non-null wrapper paths are unique among
wrappers. Within one record, `logical_path == implementation_path` is allowed
and is the normal prototype representation. Across two different records, a
logical path may not equal the other's implementation path. A wrapper path must
differ from every logical/implementation path. Python treats records as opaque
typed values: it checks exact shape/order/equality but never infers Rust
identity from a path or wrapper spelling.

### Replacement outputs and observation metadata

The `replace` command becomes:

```text
crat-tool replace \
  --request replacement-request.json \
  --output candidate.rs \
  --statement-pairs-output replacement-statement-pairs.json \
  --observation-source-output replacement-observation.rs \
  --observation-metadata-output replacement-observation-metadata.json \
  current-project
```

All four output paths must be pairwise distinct. `candidate.rs` and the
schema-version-1 statement-pair sidecar remain byte-for-byte identical to the
pre-Phase-5 output for the same request. `replacement-observation.rs` is
separate and is never installed. The metadata has exactly:

```json
{
  "schema_version": 1,
  "candidate_sha256": "<64 lowercase hexadecimal digits>",
  "statement_pairs_sha256": "<64 lowercase hexadecimal digits>",
  "observation_source_sha256": "<64 lowercase hexadecimal digits>",
  "accepted_correspondence": [],
  "new_correspondence": [
    {
      "item_id": 7,
      "logical_path": "module::read",
      "implementation_path": "module::read",
      "wrapper_path": "module::__proctor_wrapper_read"
    }
  ],
  "current_items": [
    {
      "item_id": 7,
      "logical_path": "module::read",
      "source_copy_path": "module::__proctor_source_read",
      "implementation_path": "module::read",
      "wrapper_path": "module::__proctor_wrapper_read",
      "transform_labels": [0]
    }
  ]
}
```

Digests are SHA-256 over exact file bytes and bind all three companion outputs
to one calculation. `accepted_correspondence` must equal the request field
value-for-value and order-for-order. `new_correspondence` and `current_items`
preserve request order and agree on their shared fields. Each `transform_labels`
equals the request item's `statements_requiring_transformation`. The source-copy
path is current-attempt-only and is never promoted; source-copy paths are unique
and differ from every logical, implementation, wrapper, and other source-copy
path in current metadata. Wrapper allocation follows
the current candidate algorithm in request order and is frozen before
source-copy allocation, which also follows request order and cannot perturb
candidate names or bytes. The private base is
`__proctor_source_<unraw-function-name>` (plus the existing numeric collision
suffix); for example, `r#type` uses `__proctor_source_type`.

“Never sorts the request” applies to replacement planning and allocation. The
existing statement-pair sidecar retains its current independent canonical
ordering so its bytes remain unchanged.

The CLI removes stale regular files or symlinks at the four exact destinations,
rejects an unexpected directory or other nonregular node, and writes all
temporary sibling files before renaming finals. Cross-file rename is not
atomic: if a later rename fails, it removes every earlier final and all owned
temporaries and reports cleanup failures with the primary error. Python
independently applies the same stale-output policy, requires four regular
nonsymlink outputs, verifies each digest and all cross-file invariants before
installing the candidate, and removes every attempt output after any process or
protocol failure.

### Extraction command and result

After build acceptance Python invokes:

```text
crat-tool extract-observations \
  --metadata replacement-observation-metadata.json \
  --output extracted-observations.json \
  replacement-observation.rs
```

Observation source, metadata, and output paths must be pairwise distinct. The
command strictly loads the metadata, verifies the observation-source digest,
and consumes no current project. It writes the same closed observation document
used for the final artifact:

```json
{
  "schema_version": 1,
  "observations": []
}
```

It writes the output only on success and exits nonzero with the structured
error kind/message on failure. Unsupported statements and observations are
successful skips and may produce an empty list.

## Observation-source construction

### One calculation, two source trees

Replacement parses and resolves the real current source once and constructs
the ordinary candidate and observation source from separate AST clones of one
replacement plan. It must not scrape the candidate or use prompt
`annotated_source`, `annotated_skeleton`, or `todo!()` payloads as source
bodies.

For each requested current function the plan retains its item ID, logical
path, current AST item and `LocalDefId`, canonical accepted target before label
removal, frozen implementation/wrapper plan, transform labels, and a
reserved source-copy name. The ordinary candidate follows the existing path
exactly, including label removal, call redirection, wrappers, export handling,
and the special two-argument `main_0` boundary.

The observation tree contains, as same-module siblings:

- the canonical target implementation with every PROCTOR statement label;
- the same generated compatibility wrapper and executable boundary as needed
  for resolution, but with outer attributes stripped; and
- a private, collision-free clone of the real pre-replacement function, with
  its old signature/body and newly materialized canonical source labels.

After applying all source-copy call-path rewrites, rerun the same deterministic
depth-first statement labeler used by skeleton generation on each real current
clone. Current-source statement topology is invariant until that function's
SCC is replaced, so this regenerates canonical numeric labels while retaining
earlier-SCC call redirections. This needs no additional source-label wire data,
annotated source body, or second validation protocol. The accepted target is
already canonicalized and validated by replacement.

Within a current SCC, compiler-resolved recursion and calls between source
copies are rewritten to absolute source-copy paths. Their target counterparts
continue to call target implementations. A required source-copy rewrite hidden
inside macro token input is fatal. Calls to accepted earlier SCCs retain the
current wrapper spelling and are related to their implementation only by the
explicit accepted correspondence. Calls outside the prototype remain as
written.

Strip every outer function attribute from every observation-source source copy,
target implementation, and wrapper without inspecting it. Retain PROCTOR
statement attributes until extraction records them. Do not recursively strip
other statement/expression attributes: those inputs remain unsupported. Inputs
whose correctness depends on non-PROCTOR attributes are outside the supported
observation model. None of this changes candidate attribute handling.

The sibling safe `main` rewritten for two-argument `main_0` has no source copy,
label metadata, or logical record. The complete observation source must compile
after the specialized runner removes PROCTOR labels.

## Accepted correspondence lifecycle

One correspondence record says that a logical source-defined free function,
its implementation at the original path, and an optional compatibility
wrapper are the same callable for alignment. Names are never reverse-engineered.

Python supplies all previously accepted records on every replacement request.
The metadata echoes them and proposes current records. After the candidate
build succeeds, extraction uses both sets. Python appends
`new_correspondence` only after extraction succeeds, or after the validated
build when the no-transform-label optimization below omits extraction. A failed
replacement, build, extraction, or stage never promotes it.

Mechanical SCCs still emit and validate all four replacement outputs, build
the candidate, and promote new correspondence. They may skip the extraction
process when every `transform_labels` list is empty, because they cannot emit
an observation. This optimization cannot change outputs, correspondence,
statement-pair acceptance, or Cargo-build counts.

## Specialized unexpanded-AST observation compiler path

Alignment and dumping use the unexpanded surface AST. Expansion is used only
to obtain resolution, AST/HIR correspondence, type-check results, and
adjustments for macro-free surface nodes.

The tools-private runner performs these steps:

1. Parse the observation source with the same compiler configuration and crate
   context as `crat-tool`, retaining a clone of the unexpanded surface AST.
2. In only metadata-declared source copies and implementations, parse canonical
   outer `#[proctor(<one unsuffixed u32 integer>)]` statement attributes. For
   every `transform_labels` entry, require exactly one source statement and one
   target group; the target group may repeat the label only on consecutive
   statements.
3. Reject malformed labels, multiple source occurrences or nonconsecutive
   target groups for a transform label, absent corresponding transform labels,
   and metadata/function/path disagreement. Do not revalidate full label sets
   or statement topology against skeleton data.
4. Record owning function, label, unexpanded surface node identity, and span;
   remove all PROCTOR attributes in memory.
5. If either complete selected source statement or its target group contains
   any unexpanded `MacCall`, mark that label as a conservative skip. Do not
   inspect expansion output or try origin recovery.
6. Expand, resolve, lower, and type-check the label-free crate while retaining
   mappings from the recorded macro-free surface nodes to HIR/type results.
   Mapping zero or multiple nodes is fatal. AST/HIR mappings are never compared
   across functions as identity.
7. Return only owned wire values from the compiler callback; no `TyCtxt`-bound
   value escapes.

The ordinary `utils::compilation::run_compiler_on_path` and all its consumers
remain unchanged. Unsupported rustc AST variants are classified as Reject or
unsupported dump nodes; extraction must never panic because a new variant is
encountered.

## Function and binding correspondence

Pair source-copy and target parameters by zero-based index and require the same
`Symbol`; both must be simple identifier bindings. Pair eligible simple locals
by their direct declaration label and `Symbol`. The label identifies exactly
one simple local per side. Shadowed locals therefore remain distinct.

For a source local that already had an explicit annotation, require the paired
target annotation to be present; normalize each side and require it to equal
that side's recorded source/selected-target binding type. The per-side types may
differ. For an inferred source local, obtain its normalized binding type from
compiler results, require the paired target to have the materialized explicit
annotation emitted by the skeleton, and require that annotation to equal the
selected target binding type. Source `type: null` versus target `type:
TypeTree` is then an expected fixed-skeleton difference in alignment and
dumping. The inferred raw-pointer source local remains an eligible anchor.
Every other annotation-presence or per-side type combination is a fatal
correspondence inconsistency. Valid explicit or inferred pairs are fixed
skeleton facts rather than textual non-region mismatches.
Generated `proctor_temp_var_<n>` bindings, destructuring/control-pattern
bindings, and target-only bindings have no source peer and cannot anchor.

Pair source-defined free-function paths through logical correspondence:
current-SCC source-copy calls map to current implementations, and accepted
earlier wrappers map to their implementations. All statically resolved direct
calls are allowed as surrounding context, including standard-library,
external-Rust, foreign `extern "C"`, and libc calls. Unresolved and
indirect/function-pointer calls are rejected; spelling is never used to invent
correspondence. Malformed, contradictory, or dangling correspondence is fatal.
When no valid record relates two otherwise well-formed source-defined callable
identities, their non-region mismatch is a conservative alignment skip rather
than a metadata error.

## Statement selection and topology

Only labels listed in `transform_labels` are candidates. A preserved label never
emits an observation. Require one source statement and one canonical target
group for every declared label; malformed grouping is fatal. A target group of
more than one consecutive statement is a successful skip because there is no
single target alignment unit.

Nested labels are independent units. For an outer control, retain its shell,
conditions, guards, and other unlabeled expressions, but replace every nested
labeled subtree by an opaque leaf carrying its label and structural role. Never
mine syntax in that leaf through the outer label. If one selected outer region
itself encloses any opaque nested labeled subtree, discard that region; other
disjoint outer regions remain eligible. Process transformed nested labels
independently; preserved nested labels stay opaque and never emit. A macro
anywhere in the complete source statement or target group, including inside an
otherwise opaque nested label, skips that label.

## Anchor and region selection

An anchor is one unexpanded expression-path occurrence whose compiler
resolution is a paired simple parameter/local and whose normalized source
outer semantic type is `RawPtr`. Multiple occurrences of one binding are
distinct anchors. Fields, statics, constants, call results, pointer-containing
ADTs, generated/destructured bindings, and target-only bindings do not seed.

For each occurrence independently, walk expression-parent edges within its
alignment unit. `Grow` replaces the region root with the parent and continues;
`Finish` retains the current root; `Reject` discards only that anchor. An
impossible edge is handled as Reject, never a panic. Parentheses are
transparent. Decisions use compiler semantic types and adjustments, and
overloaded operators are not builtin.

Group selected regions by surface AST node identity, not rendered text. Merge
identical roots, union their anchor occurrences, and deduplicate serialized
anchor bindings by resolved binding identity in first source-occurrence order.
After merging, if any two roots overlap by strict ancestry, skip the complete
statement. Otherwise align every disjoint root simultaneously. Two textual
copies of `*pointer` are distinct nodes and are not merged.

Pointer-like means only these tools-mode shapes:

- raw pointers;
- shared/mutable references and shared/mutable slices;
- `Option` of those references/slices;
- `Box<T>` and `Option<Box<T>>`; and
- `Box<[T]>` and `Option<Box<[T]>>`.

Recognize `Option` and `Box` by resolved standard-library identity. Exclude
slice cursors and pointer-containing ADTs. For a resolved builtin raw-pointer
method receiver, the complete Grow allowlist is exactly `offset`, `add`, `sub`,
`wrapping_offset`, `wrapping_add`, `wrapping_sub`, `offset_from`, and
`is_null`; every other raw-pointer receiver method rejects.

### Closed parent-edge policy

`Finish` is only a region boundary. It no longer tries to infer a target
context-required type; extraction has complete source and target expressions
and records their actual types.

| Parent role | Decision |
| --- | --- |
| `Array`/`Tup` element, `Repeat` value | Finish if the current expression is non-pointer-like; otherwise Reject. |
| `Repeat` count | A local anchor is not legal in a typed Rust array repeat count; defensively Reject. |
| direct `Call` argument | Finish for any statically resolved direct callee; unresolved/indirect calls Reject. |
| `Call` callee | Reject. |
| allowlisted builtin raw-pointer `MethodCall` receiver | Grow. |
| other pointer-like method receiver | Reject. |
| non-pointer method receiver | Finish. |
| resolved method argument | Finish; unresolved method calls Reject. |
| builtin `Binary` operand | Finish; overloaded operands Reject. |
| `Unary(Deref)` operand | Grow. |
| other builtin unary operand | Finish; overloaded operands Reject. |
| pointer-like `Cast` operand | Grow; otherwise Finish. |
| `If`/`While` condition or `Match` guard | Finish. |
| `If` branch tail, `Match` scrutinee/arm result | Finish if non-pointer-like; otherwise Reject, subject to opaque labels. |
| `Assign` left or right | Finish. |
| builtin non-pointer `AssignOp` operand | Finish; otherwise Reject. |
| `Field` base | Finish if non-pointer-like; otherwise Reject. |
| `Index` base | Reject. |
| builtin scalar `Index` index | Finish; otherwise Reject. |
| `AddrOf` operand | Grow. |
| `Return` operand | Finish. |
| declared `Struct` field value | Finish; functional-update base Rejects. |
| unlabeled, statement-free, single-tail `Block` | Finish at its tail; other block crossings Reject. |
| `Paren` | Grow transparently and omit. |
| alignment-unit or containing-statement boundary | Finish. |
| `ConstBlock`, expression `Let`/type ascription, `ForLoop`, `Loop` boundary, `Closure`, `Gen`, `Await`, `Use`, `TryBlock`, `Range`, `Break`, inline assembly, `Try`, `Yield`, `Yeet`, `Become`, `FormatArgs`, `UnsafeBinderCast`, macro, or unsupported/error variant | Reject defensively. |

Literal/path leaves do not have children. Do not broaden this table during
implementation merely because another rustc AST variant is traversable.

## Source-guided alignment

Normalize away parentheses and align the complete source and target units in
one recursive traversal. At a selected source root, map it to the complete
target expression in the same role and descend into neither. Else require the
same construct, ordered child roles and arity, and equivalent non-region
leaves, then recurse. Opaque nested labels compare by label and role.

Non-region binding paths compare through binding correspondence; source-defined
free functions compare through logical correspondence; all other supported
resolved definitions compare by semantic identity. Literal values, operators,
mutability, resolved field/variant/type identity, labels, and semicolon roles
must match, except for the paired fixed local-type rules above. A
different construct is allowed only at a selected root. Never
search a target subtree. Any unmatched root, reused target node, unsupported
non-region node, or non-region mismatch skips the complete statement.

Each successfully mapped region produces one observation after identical-root
merging. If converting either mapped expression or any of that observation's
six-or-more recorded type facts to the
closed schema fails, discard only that observation; other disjoint observations
from the statement remain valid once alignment itself succeeded.

## Closed observation wire schema

The root document has exactly `schema_version` and `observations`:

```json
{
  "schema_version": 1,
  "observations": [
    {
      "source_expression": {"kind": "path", "value": {"kind": "binding", "id": "<id0>"}},
      "target_expression": {"kind": "path", "value": {"kind": "binding", "id": "<id0>"}},
      "pointer_anchors": [
        {
          "id": "<id0>",
          "source_type": {"kind": "raw_pointer", "mutability": "const", "pointee": {"kind": "primitive", "name": "i32"}},
          "target_type": {"kind": "reference", "mutability": "shared", "pointee": {"kind": "primitive", "name": "i32"}}
        }
      ],
      "source_type": {"kind": "raw_pointer", "mutability": "const", "pointee": {"kind": "primitive", "name": "i32"}},
      "source_adjusted_type": {"kind": "raw_pointer", "mutability": "const", "pointee": {"kind": "primitive", "name": "i32"}},
      "target_type": {"kind": "reference", "mutability": "shared", "pointee": {"kind": "primitive", "name": "i32"}},
      "target_adjusted_type": {"kind": "reference", "mutability": "shared", "pointee": {"kind": "primitive", "name": "i32"}}
    }
  ]
}
```

No observation contains a statement label, path provenance, source span,
function/SCC/attempt ID, original local name, validation result, or inferred
future context-required type.

### Simplified normalized types

Use one recursive `TypeTree` for expression types/adjusted types, anchor
binding types, and type syntax inside expression trees. Expression types come
from rustc type-check results; mapped AST/HIR type syntax such as a cast target
is resolved to semantic `Ty` first. Never serialize AST spelling.

Normalize aliases and projections to their resolved semantic type recursively
and erase regions/lifetimes. The closed variants are:

```text
Primitive  { kind: "primitive", name }
Slice      { kind: "slice", element }
Array      { kind: "array", element, length }
RawPtr     { kind: "raw_pointer", mutability, pointee }
Ref        { kind: "reference", mutability, pointee }
Tup        { kind: "tuple", elements }
Adt        { kind: "adt", adt_kind, identity, arguments }
```

Primitive `name` is exactly one of `bool`, `char`, `str`, `never`, `i8`, `i16`,
`i32`, `i64`, `i128`, `isize`, `u8`, `u16`, `u32`, `u64`, `u128`, `usize`,
`f16`, `f32`, `f64`, or `f128` when that primitive is supported by the selected
compiler/target. Unit is an empty tuple.
Array `length` is an evaluated `u64`. Mutability is `"const"`/`"mut"` for raw
pointers and `"shared"`/`"mutable"` for references. `adt_kind` is `"struct"`,
`"enum"`, or `"union"`. ADT `arguments` contains every recursively
representable rustc-normalized type argument, including representable default
arguments such as `Box<T, alloc::alloc::Global>`; lifetimes only are omitted.
Const arguments other than array length are unsupported.

An external/library ADT identity is:

```json
{"kind": "external", "crate": "alloc", "path": ["boxed", "Box"]}
```

It uses the defining crate name and canonical rustc definition-path segments,
independent of imports and reexports and without compiler hashes. A crate-local
identity is:

```json
{"kind": "local", "id": "<struct0>"}
```

with the prefix matching its ADT kind. Allocate local ADT IDs by first
occurrence across source expression, target expression, each ordered anchor's
source/target types, then the four expression-type fields in wire order; the
same resolved definition reuses its ID. Local field and variant identities
similarly use `<fieldN>` and `<variantN>`, include their anonymized owner ADT
identity, and reuse IDs across source/target.

Unsupported types include functions/function items, closures/generators,
dynamic traits, foreign types, parameters, inference/error types, unresolved
projections or opaques after normalization, and ADTs with unsupported generic
arguments. Failure to represent any expression-level or anchor type required
by one observation discards that observation.

### Expressions and containers

Every node is an object tagged by `kind`; unknown kinds or keys are invalid.
These are the exact expression variants and fields:

```text
array       { kind, elements: [Expr] }
call        { kind, callee: Expr, arguments: [Expr] }
method_call { kind, receiver: Expr, method: ValueIdentity, arguments: [Expr] }
tuple       { kind, elements: [Expr] }
binary      { kind, operator: BinaryOperator, left: Expr, right: Expr }
unary       { kind, operator: UnaryOperator, operand: Expr }
literal     { kind, value: Literal }
cast        { kind, expression: Expr, type: TypeTree }
if          { kind, condition: Expr, then: Block, else: Expr|null }
while       { kind, condition: Expr, body: Block }
loop        { kind, body: Block }
assign      { kind, left: Expr, right: Expr }
assign_op   { kind, operator: BinaryOperator, left: Expr, right: Expr }
field       { kind, base: Expr, field: FieldIdentity }
index       { kind, base: Expr, index: Expr }
range       { kind, start: Expr|null, end: Expr|null, limits: "half_open"|"closed" }
path        { kind, value: ValueIdentity }
address_of  { kind, borrow: "reference"|"raw", mutability: "const"|"mut", expression: Expr }
break       { kind, value: Expr|null }
continue    { kind }
return      { kind, value: Expr|null }
struct      { kind, adt: AdtIdentity, variant: VariantIdentity|null, fields: [StructField], rest: Expr|null }
repeat      { kind, value: Expr, count: Expr }
block       { kind, block: Block }
```

`Block` is `{ "statements": [Statement] }`. A statement is exactly
`{"kind":"let","pattern":Pattern,"type":TypeTree|null,"initializer":Expr|null}`
or `{"kind":"expression","expression":Expr,"semicolon":bool}`. Only simple
binding and wildcard patterns are supported:
`{"kind":"binding","id":"<idN>","mutability":"immutable"|"mutable","by_ref":"no"|"shared"|"mutable"}`
and `{"kind":"wildcard"}`. Parentheses are omitted. Any other expression,
pattern, statement, attribute, let-else, or block kind in either mapped tree
discards that observation. Unlabeled loops, break, and continue use the variants
above; an explicit Rust loop label anywhere in a mapped tree discards that
observation.

`AdtIdentity` is exactly the external/local identity object defined above.
`FieldIdentity` and `VariantIdentity` are either
`{"kind":"external","crate":"...","path":["..."]}` or
`{"kind":"local","owner":AdtIdentity,"id":"<fieldN>"}` /
`{"kind":"local","owner":AdtIdentity,"id":"<variantN>"}`. A `StructField`
is `{"field":FieldIdentity,"value":Expr}`; field order is source order and
duplicates are invalid. `struct.variant` is non-null only for an enum variant.

`BinaryOperator` is one of `add`, `subtract`, `multiply`, `divide`, `remainder`,
`and`, `or`, `bit_xor`, `bit_and`, `bit_or`, `shift_left`, `shift_right`,
`equal`, `not_equal`, `less`, `less_equal`, `greater`, `greater_equal`.
`UnaryOperator` is `deref`, `not`, or `negate`.

Literal shapes are closed and semantic: Boolean is
`{"kind":"bool","value":true|false}`, character/string are
`{"kind":"char","value":"..."}` and
`{"kind":"string","value":"..."}`, and byte is
`{"kind":"byte","value":0..255}`. Byte and C strings are
`{"kind":"byte_string","value":[0..255]}` and
`{"kind":"c_string","value":[0..255]}`; the C terminator is omitted.
Integer is
`{"kind":"integer","value":"<unsigned decimal magnitude>","type":"<integer primitive>"}`;
float is
`{"kind":"float","bits":"<lowercase fixed-width hexadecimal>","type":"f16|f32|f64|f128"}`.
Signs remain unary expressions. Literal suffix/import spelling is not retained.

Resolved binding values are `{"kind":"binding","id":"<idN>"}` and
source-defined free functions are `{"kind":"function","id":"<fnN>"}`.
External Rust definitions use
`{"kind":"external","crate":"...","path":["..."]}` with canonical defining
identity. A resolved foreign item is supported only when its ABI is exactly
`extern "C"`: functions use
`{"kind":"foreign_function","symbol":"<resolved-link-symbol>"}` and statics
use `{"kind":"foreign_static","symbol":"<resolved-link-symbol>"}`. The ABI is
implied and omitted; every other foreign ABI discards the observation. An ADT constructor path uses
`{"kind":"constructor","adt":AdtIdentity,"variant":VariantIdentity|null}`.
Fields and variants use the identity shapes above. No generic textual-path
fallback exists; unresolved or unsupported resolution discards.

Crate-local module and associated constants use
`{"kind":"constant","id":"<constN>"}`; source-defined statics use
`{"kind":"static","id":"<staticN>"}`; source-defined inherent methods and
associated functions use `{"kind":"method","id":"<methodN>"}`. Resolved
identity, not spelling, controls reuse. Crate-local trait methods are
unsupported and discard the observation.

### Anonymization and ordering

Every anonymized namespace is local to one observation, starts at zero, is
contiguous, and resets for the next observation. Thus `<struct0>` in two
observations makes no cross-observation identity claim.

Allocate IDs for every resolved source binding in pure source-expression
preorder, including source-only bindings, then reuse those IDs for paired target
references. A target binding whose paired source binding was not encountered
in that selected source expression discards the observation without allocating
or reusing another ID. Allocate free-function, local ADT, field, variant, constant,
static, and method namespaces separately in source-tree then target-tree
first-occurrence order. Repeated identities reuse IDs.

Within an extraction result order observations by current item ID, label
depth-first order, then merged region root in source left-to-right preorder.
Within an observation order pointer anchors by first source occurrence.
Python appends extraction results in leaf-first SCC schedule order and never
sorts or deduplicates them.

## Error and skip taxonomy

Successful conservative skips include preserved/multi-statement/macro labels,
no anchors, rejected parent edges, strict region overlap, pointer-valued joins,
alignment mismatch, indirect/unresolved call, unsupported expression/type
tree, target-only binding/function, and any unrepresentable recorded type.

Fatal structured tools errors include malformed/unsupported protocol,
digest/file inconsistency, missing/duplicate requested functions, ambiguous or
absent transform-label pairs, malformed/contradictory/dangling correspondence,
macro-hidden source-copy call rewrite, parameter/local pairing disagreement,
and inability to parse, resolve, expand, lower, type-check, or recover a
metadata-declared macro-free node. Serialization failure after accepting a
typed result is fatal. `crat-tool` exits nonzero without a usable partial file;
Python treats these as fatal, not repairable LLM failures.

An extraction failure after a build is fatal and does not trigger another LLM
generation. It publishes no stage outputs and promotes no correspondence or
observations. It need not roll the build-accepted current source back merely to
retry a transformation that was already accepted.

## Python accumulation and publication

Strict Python loaders mirror every closed shape and cross-record invariant.
They reject unknown/missing keys, unknown tags, invalid integers/IDs/paths,
invalid order in lists whose order metadata can verify, duplicate
correspondence, digest mismatch, target-only anonymized IDs, and stale outputs.
They preserve observation producer order and do not repair, sort, or
deduplicate it.

After a successful build, load/extract, append observations, promote new
correspondence, and accept statement pairs as one in-memory commit. The final
artifact is always named `observations.json` and is placed beside
`statement-pairs.md`: normally under `outputs.artifacts_dir`, otherwise under
`framework.workdir`. A successful run with no observations publishes exactly:

```json
{
  "schema_version": 1,
  "observations": []
}
```

with one terminal newline. It is a data artifact and is not listed in
`StageOutput.logs`.

Extend invocation-start stale-output handling, destination overlap checks,
temporary sibling files, `os.replace`, rollback, and cleanup across the final
Rust project, Markdown report, and JSON document. Cross-file publication is not
filesystem-atomic: prepare both artifact temporary files first, then publish
the Rust project, Markdown, and JSON in that order while tracking every
successful rename. If temporary creation, copying, or any final rename fails,
remove every temporary and final path created by this invocation, including
finals renamed before the failure, while preserving unrelated paths. Reject
unexpected directories and unsafe symlinks at exact destinations. Cleanup is a
best-effort rollback with deterministic primary-plus-cleanup diagnostics; a
failed stage returns no usable output even though another process could have
observed a transient subset between renames.

The stage envelope, configuration, metrics, prompt/usage reporting, and
statement-pair renderer otherwise remain unchanged.

## Implementation sequence and completion

1. Add exact version-1 Rust/Python protocol types, serializers, loaders, and
   malformed-value tests.
2. Refactor replacement into one immutable plan; freeze candidate behavior,
   then emit observation source, metadata, digests, and current correspondence.
3. Implement source-copy call rewriting, deterministic relabeling, attributes,
   collision handling, and observation-source compilation.
4. Implement the tools-private label-removing unexpanded-AST runner and
   compiler mappings.
5. Implement binding/function correspondence, region selection, identical-root
   merge, strict-overlap rejection, and simultaneous alignment.
6. Implement normalized type conversion, closed dump trees, deterministic
   anonymization, and observation serialization.
7. Add the thin `extract-observations` command and four-output replacement
   cleanup.
8. Extend Python post-build extraction, accepted correspondence, strict
   accumulation, and transactional publication.
9. Run the complete [Phase 5 test plan](phase-5-test-plan.md), focused and
    workspace Rust tests, focused Python lint/tests, `cargo fmt`, and
    `cargo clippy --workspace --all-targets`.
10. After implementation, reorganize `prototype-desc.md` at its existing level
    of detail. Do not edit completed historical phase/amendment documents.

Phase 5 is complete only when candidate and statement-pair bytes are unchanged,
all planned protocols and variants have exact passing tests, conservative skips
never panic, successful empty runs publish `observations.json`, no observation
source or metadata enters the output project, and the current-description document has
been updated.
