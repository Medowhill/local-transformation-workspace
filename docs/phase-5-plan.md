# Phase 5 Detailed Plan: Validated Expression Observation Collection

This is a prospective implementation plan for the local-transformation
prototype. It must be reconciled with the current implementation and tests
when work begins. Do not treat it as implemented until
[prototype-desc.md](prototype-desc.md) says that it is.

See the [historical-plan overview](prototype-plan.md#phase-5-validated-expression-observation-collection-planned).
No separate Phase 5 test plan has been written yet.

## Goal and boundary

Phase 5 records reusable evidence from accepted transformations by mapping
source expression regions containing raw-pointer variables to corresponding
target expressions.

An *observation* is one concrete source-expression/target-expression mapping
with the type facts needed to interpret it later. Observations are not rules.
Rule synthesis may consume them online during a later transformation or
offline across programs without changing this phase.

The extractor treats SCC validation as a black box. It runs only after the
orchestrator accepts the installed SCC candidate and records nothing about how
acceptance was established. Future tests or stronger validation do not change
observation extraction.

## SCC workflow

One canonical replacement calculation produces two library sources:

1. the ordinary candidate source, exactly as replacement emits it today; and
2. an analysis source containing the transformed functions, generated
   compatibility wrappers, and private collision-free copies of the replaced
   source functions.

Only the ordinary candidate is installed and validated. After the orchestrator
accepts it, the orchestrator invokes Crat on the separate analysis path without
swapping it into the project. Failed or superseded attempts yield no
observations.

The current `statement-pairs.md` flow remains unchanged. The new structured
artifact does not use that Markdown report as input.

## Analysis source and function correspondence

Both versions retain matching canonical `#[proctor(N)]` labels. Source copies
use real pre-replacement functions rather than prompt skeletons, are private,
and have export-related attributes removed.

The replacer also emits explicit mappings between:

- each logical function and its source-copy and target full paths;
- compatibility wrappers, transformed implementations, and their common
  logical callees.

Crat resolves source and target names independently, then correlates preserved
parameters by parameter index and `Symbol`, and simple locals by declaration
statement label and `Symbol`. Compiler-local IDs are not compared across the
two functions. User-defined free functions correspond through their logical
crate-relative paths and the replacer's wrapper/implementation mapping.

Source-copy recursion and within-SCC calls use corresponding source copies.
Earlier-SCC calls may use wrappers while target calls use implementations;
alignment compares reported logical callees rather than path spelling.

Keep the existing bare label syntax. The pinned rustc API does not provide a
mutable expanded AST after unique `NodeId`s have been assigned. A specialized
Crat runner records labels by function and structural locator/span while the
parsed AST is mutable, removes canonical PROCTOR labels in place, and expands.
It then resolves the locations to real node IDs, retains the surface AST before
lowering steals it, and builds existing AST/HIR and type-checking data. This
avoids attribute errors without changing the label format.

## Statements considered

Consider only labels whose source disposition is `transform`; preserved
statements do not produce observations.

Require one source and one target statement per ordinary label. Skip a label
whose accepted target expands to multiple statements.

Use label topology to avoid processing control syntax twice. If a control
contains nested labeled statements, treat those subtrees as opaque at the
outer label and process them independently; the outer unit retains only its
control shell and unlabeled expressions such as conditions and guards. If a
control contains no nested labels, use the complete control statement as its
alignment unit. The region and dump-tree allowlists still determine whether
that unit can produce observations.

## Source expression regions

Region discovery is source-driven. An anchor is a simple parameter or local
binding occurrence with compiler-resolved raw-pointer source type. Fields,
statics, call results, and arbitrary pointer-valued expressions are excluded.

For each anchor, walk through AST parents within its selected labeled statement.
Every parent edge has one of these outcomes:

- **Grow**: include the parent in the source region and continue upward;
- **Finish**: keep the current expression as the region and use the parent to
  derive the target context-required type after alignment;
- **Reject**: extract no region for this anchor; or
- **Impossible**: the construct cannot contain a resolved local-expression
  anchor, but handle an unexpected mapping as `Reject` rather than crashing.

“Pointer-like” below is determined from compiler types rather than syntax. An
operation is builtin only when type checking confirms that it is not overloaded.
Every `Finish` remains conditional on deriving one unique target required type;
otherwise the aligned region is skipped. Boundary kinds guide this internal
derivation but are not serialized in the observation.

| Parent construct and edge | Region decision |
| --- | --- |
| `ConstBlock` | `Impossible` for a local anchor; otherwise `Reject`. |
| `Array` element | `Finish` when the current element is non-pointer-like; reject a pointer-like element in this phase. Decide from the element, not the aggregate type. |
| `Tup` element | Same as `Array`, using the corresponding tuple slot. |
| `Repeat` element or count | Treat the repeated value like an `Array` element; `Finish` a non-pointer scalar count and otherwise reject. |
| `Call` callee | `Reject`; indirect and function-pointer calls are outside this phase. |
| `Call` argument | `Finish` only for a statically resolved, supported callee and use its target formal parameter type; reject foreign/libc and unresolved calls. |
| `MethodCall` receiver with raw-pointer type | `Grow` through the resolved raw-pointer operation, including offset/add/sub, wrapping variants, and null checks. |
| `MethodCall` receiver with another adjusted pointer-like type | `Reject`. |
| `MethodCall` receiver with non-pointer type | `Finish`. |
| `MethodCall` argument | `Finish` only when the target formal parameter type is uniquely derivable; otherwise reject. |
| `Binary` operand | `Finish` before a builtin binary operator so `*x + *y` yields two regions; reject overloaded operators. Whole-operator rewrites are outside this phase. |
| `Unary(Deref)` operand | `Grow`. |
| Other `Unary` operand | `Finish` for a builtin operator; reject an overloaded operator. |
| `Lit` | `Impossible`. |
| `Cast` operand | `Grow` when the current operand type is pointer-like; otherwise `Finish`. |
| `Type` or expression `Let` | `Reject`. |
| `If` condition | `Finish` with required type `bool`. |
| `If` branch tail | `Finish` a non-pointer-like result; reject a pointer-like result. Apply the labeled-control policy above before reaching this edge. |
| `While` condition | `Finish` with required type `bool`; the body is handled through its own labels. |
| `ForLoop` | `Reject`. |
| `Loop` | Do not cross the loop/body boundary; `Reject` if reached. |
| `Match` scrutinee | `Finish` a non-pointer-like scrutinee; reject a pointer-like scrutinee. |
| `Match` guard | `Finish` with required type `bool`. |
| `Match` arm result | Process separately labeled statements; otherwise finish only a non-pointer-like result and reject a pointer-like result. |
| `Closure` | `Reject`. |
| `Block` | Do not cross a labeled boundary. For an unlabeled, statement-free, single-tail block, `Finish` at the tail; otherwise reject if reached. |
| `Gen`, `Await`, `Use`, or `TryBlock` | `Reject`. |
| `Assign` RHS | `Finish`; derive the target required type from the target assignment place. |
| `Assign` LHS | `Finish` the current place expression and use the corresponding target place type. |
| `AssignOp` operand | `Finish` only for a builtin, non-pointer operation; otherwise reject. |
| `Field` base | `Finish` when the current base is non-pointer-like and the target ADT base type is derivable; reject a pointer-like base. This permits contextual `*p -> p` in `(*p).f -> p.f`. |
| `Index` base | `Reject`. |
| `Index` index | `Finish` a builtin scalar index; otherwise reject. |
| `Range` | `Reject`. |
| `Underscore` | `Impossible`. |
| `Path` | It is a seed only when it resolves to an eligible raw-pointer parameter or simple local; other paths do not start regions. |
| `AddrOf` operand | `Grow`. |
| `Break` value | `Reject`. |
| `Continue` | `Impossible`. |
| `Ret` operand | `Finish`; use the target function return type. |
| `InlineAsm` | `Reject`. |
| `OffsetOf` | `Impossible`. |
| `MacCall` | `Reject`. |
| `Struct` field value | `Finish` using the target declared field type, which remains unchanged in this prototype; reject the functional-update base. |
| `Paren` | `Grow`, but erase the parentheses from the simplified tree and treat them as transparent during alignment. |
| `Try`, `Yield`, `Yeet`, or `Become` | `Reject`. |
| `IncludedBytes` | `Impossible`. |
| `FormatArgs` or `UnsafeBinderCast` | `Reject`. |
| `Err` or `Dummy` | `Impossible`; defensively reject if encountered. |

At the containing-statement boundary, a `StmtKind::Let` initializer finishes
with the selected target type of its declared local. A function-body tail
finishes with the target return type. A root expression or semicolon statement
with no enclosing type requirement is skipped because it has no unique target
context-required type. Region growth never crosses the selected labeled
statement or opaque nested-label boundary.

These decisions deliberately exclude pointer-valued aggregates and control
joins, indirect or foreign calls, ordinary pointer-like method receivers,
overloaded operations, macros, and other shapes that need more context. A
future phase may extend the table explicitly; implementation must not silently
broaden it while collecting observations.

Parentheses are transparent. Overlapping regions skip the complete statement;
otherwise align all regions simultaneously, including independent `*x` and
`*y` changes in one expression.

## Source/target alignment

Alignment uses normalized surface ASTs. HIR and type checking identify anchors
and semantic identities but do not replace the correspondence structure.

Conceptually, recursively align the two statement trees as follows:

```text
align(source, target):
    ignore transparent parentheses
    if source is a selected region:
        map it to the complete target expression and do not descend
    otherwise:
        require the same AST construct, ordered child roles, and arity
        require equivalent non-region leaves
        align corresponding children
```

A different construct is permitted only at a selected source-region root. Any
other mismatch rejects the complete statement, preventing unrelated target
edits from creating partial mappings. Wrapper and implementation paths compare
equal only through the explicit logical-callee mapping.

This is source-guided wildcard alignment, not lockstep traversal and not a
search over arbitrary target subtrees. The unchanged surrounding AST fixes the
target position corresponding to each source region.

## Observation artifact

After alignment, convert each expression to a small, versioned tree owned by
this phase. Serialize only supported constructs, never rustc trees or IDs.

The dump tree supports `Array`, `Call`, `MethodCall`, `Tup`, `Binary`, `Unary`,
`Lit`, `Cast`, `If`, `While`, `Loop`, `Assign`, `AssignOp`, `Field`, `Index`,
`Range`, `Path`, `AddrOf`, `Break`, `Continue`, `Ret`, `Struct`, and `Repeat`,
plus the block and statement-list containers needed by supported controls.
Parentheses are transparent. Both mapped expressions must be represented
losslessly; discard the observation if either side contains another construct.

Rename resolved local bindings deterministically as `<id0>`, `<id1>`, and so
on, using the same ID on both sides and discarding original spellings. Rename
user-defined free functions similarly as `<fn0>`, `<fn1>`, and so on. The
internal correspondence uses the parameter/declaration-label and logical-item
rules above; wrapper and implementation references therefore receive the same
`<fnN>`. Standard-library and libc functions and method names remain literal,
as do fields, variants, and type names.

If the target tree refers to a local binding or user-defined free function with
no source-tree correspondence, discard the observation. Source-only references
remain representable because later synthesis can bind them from a rule's source
side. This rule also covers calls included as sibling subtrees while a region
grows, such as a helper call inside a raw-pointer offset argument.

Each observation contains only:

- the simplified source expression tree;
- the simplified target expression tree;
- each pointer-anchor ID with its source and target binding types;
- the source expression type;
- the source adjusted expression type, always present even when unchanged; and
- the target context-required type.

Derive the target context-required type from the target assignment place,
callee parameter, function return, field declaration, or other surrounding
construct. Future application must use the same resolver before transformed
code exists. Skip a region without one unique required type. Do not store a
source required type or target actual/adjusted types.

Use one deterministic, pretty-printed JSON form for both machines and humans;
do not maintain a second observation rendering.

Keep unsupported and ambiguous statements separate from fatal tool failures.
They contribute no observation; missing mapped functions, inconsistent
replacer metadata, or an unusable analysis source are internal errors.

Accumulate accepted observations across SCCs and publish them only when the
complete stage succeeds. The artifact must be deterministic and suitable for
later merging across runs, but Phase 5 does not define clustering,
anti-unification, rule confidence, or activation policy.

## Ownership and non-goals

Crat owns analysis-source construction, compiler-backed region discovery,
source/target alignment, and observation serialization. `crat-tool` remains a
thin file/protocol boundary. Python invokes extraction after SCC acceptance,
validates the output protocol, accumulates observations, and publishes the
artifact; it does not parse Rust.

Phase 5 does not:

- synthesize, generalize, rank, store, or apply transformation rules;
- serialize exact local or user-function names, rustc trees or IDs, boundary
  categories, use modes, validation metadata, or application history;
- change pointer decisions, prompts, structural validation, SCC scheduling,
  repair policy, or candidate validation;
- accept observations from failed candidates;
- support multi-statement target expansions or overlapping source regions;
- change the existing statement-pair report or add another Markdown view; or
- place source copies, labels, or analysis-only code in the validated project.

## Implementation sequence

1. Extend replacement with the separate analysis source and correspondence
   metadata while leaving the ordinary candidate byte-for-byte unchanged.
2. Add the specialized Crat analysis path for retaining label locations while
   compiling an unlabeled AST.
3. Implement conservative statement selection, region discovery, and direct
   AST alignment.
4. Define and serialize the deterministic observation protocol.
5. Invoke extraction only after SCC acceptance and accumulate its output.
6. Publish the run artifact without changing `statement-pairs.md`.
7. After implementation, reorganize `prototype-desc.md` to describe the
   current behavior rather than appending planning history.
