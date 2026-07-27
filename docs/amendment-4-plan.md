# Amendment 4 Detailed Plan

This is a historical implementation plan. Its substantive text was moved
verbatim from the former consolidated `prototype-plan.md`; imperative and
future-tense wording describes the work assigned at the time. New navigation
notes identify where later work changed an earlier component.

See the [historical overview](prototype-plan.md#amendment-4).
See the [Amendment 4 test plan](amendment-4-test-plan.md).

## Amendment Plan 4: Scope-aware synthesized type spelling

This amendment makes every type synthesized by the skeleton generator valid
and natural in the module that contains the target function. It applies after
completed Phases 1--4 and Amendment Plans 1--3. It supersedes only the
conflicting type-rendering expectations for materialized inferred locals and
for parameter, return, and local types whose representation is introduced or
changed by the tools-side initial pointer decisions.

The motivating failure is a function in module `crate::src::lib` whose source
uses the same-module type `cb_rgb`. The skeleton materialized an inferred
local as:

```rust,ignore
let mut init: src::lib::cb_rgb = todo!();
```

`Ty::to_string()` produced a diagnostic crate-relative path, not a path that
resolves from inside `crate::src::lib`. The structural validator then required
that invalid spelling exactly: changing it to `cb_rgb` was a local-type
mismatch, while retaining it failed the candidate build. The correct preferred
spelling is `cb_rgb`; if no suitable name were in scope, the correct local-crate
fallback would be `crate::src::lib::cb_rgb`.

`amendment-4-test-plan.md` is the exhaustive executable contract for this
amendment. Do not edit any historical phase or amendment test-plan file.

### A4.1 Normative scope and non-goals

Apply this amendment whenever the tools skeleton generator creates type syntax
that was not already an unchanged complete source type:

- a type materialized for an inferred simple local binding;
- a target parameter type introduced or changed by a pointer decision;
- a target return type introduced or changed by a pointer decision; and
- a target local type introduced or changed by a pointer decision, whether the
  source local had an explicit raw-pointer type or an inferred raw-pointer
  type.

Apply the policy recursively to named components inside references, raw
pointers, slices, arrays, tuples, function pointers, and generic arguments.
This includes the pointee inside introduced `&T`, `&mut T`,
`Option<&T>`, `Box<T>`, optional boxes, boxed slices, and their returned-borrow
forms.

Continue to preserve a complete explicit local type byte-for-byte at the AST
level, subject to ordinary pretty-print normalization, when no pointer decision
changes that local's representation. This amendment does not rewrite source
signatures, item definitions, global declarations, or arbitrary type syntax in
preserved source expressions.

Amendment 2's preserved-parent boundary remains unchanged. Skeletonization does
not descend into a wholly preserved statement merely to materialize types for
inferred locals nested inside it. Such nested statements are mechanically
restored and never need to be rewritten by the LLM. A simple local statement
that is itself visited continues to receive its required explicit target type.

This is a presentation and source-validity change only. Do not change:

- pointer analysis, pointer-kind selection, lifetime planning, or the
  tools-only no-cursor option;
- the ordinary Crat pointer rewriter's existing canonical type spelling;
- statement disposition, preservation, labels, or skeleton holes;
- the structural validator's syntax-structural type comparison;
- validator or replacement request/response schemas;
- skeleton JSON fields or ordering;
- Python loading, prompts, SCC scheduling, repair, or compilation policy;
- existing item-replacer constructor recognition or wrapper helper spelling;
- wrapper conversion semantics and schemas.

The validator remains structural. The producer must therefore place one valid,
stable spelling in the target skeleton; the validator must not be changed to
resolve or equate alternative paths.

Every emitted `annotated_source` and `annotated_skeleton` remains a complete
function item and must parse as such. `source_signature` and
`target_signature` are bodyless function headers; tests may parse them by
appending a dummy body, but must not describe the header alone as a complete
Rust item. When reporting a local failure, use the local binding name (for
example, local `iterator` in function `consume`), never a synthetic
`consume::iterator` item path.

### A4.2 Containing-module scope and resolved candidates

Construct one scope-aware type speller per source function. Its lexical type
scope is the module returned by
`TyCtxt::parent_module_from_def_id(function_def_id)`. Under the supported
program model, source-written function generics and function-local item
declarations are absent, so module type-namespace resolution is the only
additional lexical name layer needed, apart from the implicit-prelude handling
for introduced standard constructors in A4.5. Value bindings with the same
spelling do not affect the type namespace.

Use rustc's completed resolution, not source-text inspection. Query
`TyCtxt::module_children_local` for the containing module and retain bindings
whose `Res` is in the type namespace. These children include definitions in
the module and private, public, renamed, and effective glob imports. Match a
candidate to the exact resolved `DefId`; do not match only a symbol, rendered
path, normalized `Ty`, or equal layout.

Candidate lookup is namespace-specific. A same-spelled value, macro, or
lifetime binding does not disqualify a valid type-namespace candidate, while a
same-spelled type-namespace binding to the wrong `DefId` does. Validate every
source-shortening and fallback candidate in the type namespace; do not rely on
an expression-path lookup or on symbol uniqueness across namespaces.

Preserve the resolver-provided `Ident`, including raw-identifier spelling and
hygiene, when constructing a one-segment AST path. Do not reconstruct a
candidate by concatenating text. Candidate enumeration and selection must be
deterministic and independent of `module_children_local` iteration order.

For one named definition, choose a one-segment spelling by this exact order:

1. If a reused source type component is already a one-segment path and
   `AstToHir::path_span_to_res` proves that it resolves to the intended
   definition, retain that exact source identifier.
2. If the intended definition is itself defined in the containing module,
   prefer its own module binding over every alias.
3. Otherwise collect all containing-module bindings that resolve to the exact
   definition and choose the lexicographically smallest rendered identifier.
   This includes direct imports, renamed imports, and effective glob imports.
4. If there is no such binding, do not shorten the path.

The first rule preserves a source-written pointee alias even when another alias
would sort earlier. The second makes a same-module definition such as
`cb_rgb` remain `cb_rgb` even if that module also imports an alias for it. The
third gives multiple-import cases a stable oracle. A type alias is a distinct
resolved definition: preserve or select it only when the source path resolves
to that alias. Do not invent an arbitrary transparent alias merely because its
normalized semantic type equals the inferred type.

These rules deliberately distinguish four precedence situations: a
source-written alias wins for that reused component; an own definition wins
for semantic rendering in its defining module; an imported alias participates
only when it resolves to the exact requested definition; and an unrelated
transparent alias never becomes a semantic candidate. The implementation
tests must pin all four, including a type/value same-name namespace case.

### A4.3 Source-AST-first pointer target construction

When a pointer decision changes a parameter, explicit return, or explicitly
typed local, use the mapped source AST type as a spelling hint before rendering
its semantic MIR type.

If the source AST type is syntactically the raw pointer or reference whose
semantic representation is being replaced:

1. clone its inner AST type;
2. recursively visit every mapped path in that clone;
3. use `AstToHir::path_span_to_res` to identify each path's own resolved
   definition;
4. replace a path with the preferred one-segment candidate from A4.2 when one
   exists; and
5. otherwise retain the already compiling source path exactly.

Then wrap the resulting inner AST type with the pointer kind and lifetime
selected by the existing analysis. For example:

```rust,ignore
use crate::model::Point as P;

// source
unsafe fn read(pointer: *const P) -> i32

// target
unsafe fn read(pointer: &P) -> i32
```

This retains aliases, relative paths, raw identifiers, nested generic
arguments, array lengths, tuples, and other surface syntax that rustc has
already proven valid in the containing module. If the source is
`*const crate::model::Point` and `Point` is also a proven local binding, the
inner path is shortened to `Point`; if no short binding exists, the valid
`crate::model::Point` path remains.

For a reused source component, select candidates against that component's
mapped `Res`, not against the normalized semantic pointee's nominal `DefId`.
In particular, a source path to a type alias resolves to the alias definition
even though MIR normalizes the pointee to the aliased type. The complete mapped
source pointer already proves that the cloned inner syntax denotes the
semantic pointee; keep the alias and shorten it only through another binding
to that same alias definition.

If a source type alias hides the pointer itself, the AST path cannot be peeled
as a pointee hint: an alias denoting `*const T` does not denote `T`. Fall back
to semantic pointee rendering in that case. Likewise, an inferred raw-pointer
local has no explicit source type hint and uses the semantic path below.

Do not scan initializer or expression text to guess a type name. An
unqualified source use is useful only through its resolved module binding or
through a mapped source type path.

### A4.4 Semantic rendering and visible absolute fallback

For an inferred local or any target component without a reusable source AST
component, recursively render the semantic rustc `Ty` with the same candidate
selector.

For every nominal `DefId`:

1. use the A4.2 preferred one-segment binding when present;
2. otherwise obtain a visible/source-printable path candidate from rustc
   resolution metadata and visible-path machinery;
3. force the resulting local-crate path to begin with `crate::`, or the
   external-crate path to begin with an absolute source-visible extern binding
   such as `::std::` or `::rust_std::`; and
4. accept the candidate only after proving that it parses and accessibly
   resolves from the containing module to the intended definition.

The pinned rustc printer has relevant but deliberately narrower guarantees:
its visible-path logic consults `visible_parent_map` and prefers public
re-export paths when a definition's real parent path is hidden;
`with_no_trimmed_paths` disables unrelated one-name diagnostic trimming; and
`with_crate_prefix` requests `crate::` for a local-crate path. There is no
single rustc API that promises “the best path from this containing module,”
and the pinned printer's visible-path shortcut is primarily for external
items. Use the containing-module binding table from A4.2 for local one-segment
names. For longer local paths, search rustc's resolved module-child graph from
the crate root, following only bindings accessible from the containing module,
including `pub`, `pub(crate)`, and restricted re-export bindings when the
containing module is within their visibility. For external paths, use the
external visible-parent/module-child data that backs the printer, but root the
candidate only at an external-crate binding actually available to source: an
extern-prelude binding or an explicit `extern crate ... as ...` alias. Do not
invent the dependency crate's canonical name when the source can name it only
through an alias. Choose the root alias and the remainder of the path together
by the fewest total path segments and then lexicographically by the complete
rendered absolute path. This same shortest-then-lexicographic rule applies to
multiple local re-export routes. The visible printer may seed or cross-check
this search, but its display output alone is not proof that a candidate is
valid from the containing module.

The verification step is semantic, not a string prefix check. Parse the
candidate as an AST type path; traverse the corresponding local or external
module children in the type namespace, including re-export bindings; check
visibility/accessibility from the containing module; and require the final
`Res`/`DefId` to be the intended definition. Preserve or escape raw
identifiers in every printed segment. For an external visible path such as
`std::collections::HashMap`, add the leading `::` before validation so a
same-named local module cannot capture it.

For external validation, first prove that the leading segment is the selected
source-visible extern binding and resolve that binding to the intended external
crate. Then walk that crate's public module/re-export children. An explicit
`extern crate std as rust_std;` therefore yields
`::rust_std::hash::DefaultHasher`, not `::std::hash::DefaultHasher`, when
`std` is not separately source-visible. The test uses the sysroot `std`
already loaded by the pinned toolchain and requires no Cargo dependency or
download.

Never construct source syntax from `tcx.def_path_str(def_id)`. A definition
path is diagnostic identity, not necessarily an accessible Rust path: it can
pass through private implementation modules hidden by a public re-export.
The method may appear only in diagnostics attached to a
`TypeSpelling` error. Likewise, do not accept a type merely because a rustc
display string parses. It must pass the identity and accessibility check
above. If rustc has no visible candidate, qualification cannot be made
unambiguous, or validation fails, return the structured generation error.

Render supported compound types recursively so every nominal component follows
the same rule. Reuse/refactor the existing `utils::ir` MIR-type formatter
rather than maintaining two divergent type grammars: add one fallible internal
formatter with an explicit formatting policy. The policy supplies nominal
paths, region handling, unsupported-shape behavior, and the few legacy
presentation choices needed for byte compatibility. Preserve the existing
`mir_ty_to_string(ty, tcx)` API and output for all ordinary Crat callers. The
tools-specific caller supplies the source-valid scope-aware policy.

The ordinary pointer rewriter continues using the current canonical selector.

The shared core must be fallible even if the legacy public wrapper retains its
existing infallible contract for already-supported ordinary callers. In tools
mode it must render recursively, including generic arguments, nested
references/raw pointers/slices/arrays/tuples, and singleton tuples with their
required trailing comma. Preserve supported function-pointer safety, ABI,
inputs, output, and regions/binders. If the current formatter cannot faithfully
spell a binder, region, projection, opaque, or another semantic shape, the
tools-specific path returns `TypeSpelling`; it must not erase the information.
Source-AST-first cloning remains the way to retain surface-only details such
as a named array length. Amendment 4 supports ordinary function-pointer types
but deliberately treats higher-ranked callable binders as unsupported and
returns `TypeSpelling`; faithful higher-ranked rendering is deferred.

The legacy `mir_ty_to_string` wrapper selects a compatibility policy that
supplies the current canonical nominal renderer and retains every existing
presentation choice, including its treatment of regions and singleton tuples,
plus any pre-existing terminal fallback needed for byte-compatible output.
The tools caller selects the source-valid policy: scope-aware nominal paths,
faithful expressible regions, a singleton-tuple comma, and errors for
unsupported shapes. Only the legacy wrapper may convert a formatter result
through an invariant justified by its old contract; the tools path must
propagate every error.

The tools-side formatting path must report, rather than panic or emit a
placeholder, when it encounters an unsupported or unnameable semantic type.
It must not use `Ty::to_string()` as a source-syntax fallback. No unchecked
`parse_ty`, `todo!()`, `unreachable!()`, or `panic!()` may sit on a
source-dependent type-spelling path. Parsing a formatter result must return a
structured parse failure and propagate `TypeSpelling`; assertions remain
permitted only for invariants already proved independently of the input.

### A4.5 Introduced `Option` and `Box` constructors

`Option` and `Box` are themselves named type constructors introduced by
pointer target rendering. This amendment deliberately does not add
scope-dependent aliases or qualified spellings for them.

For these newly introduced wrapper-constructor nodes, A4.5 is a specific
exception to A4.2--A4.4: do not run the general nominal candidate or visible
fallback selector on the constructor itself. Continue to apply A4.2--A4.4 to
its pointee and other nested user-type components.

Before emitting a target type whose selected pointer kind introduces
`Option`, `Box`, or both:

1. require the ordinary implicit prelude to be enabled;
2. resolve the corresponding bare type-namespace identifier from the
   containing module, with ordinary lexical shadowing; and
3. require its exact `Res`/`DefId` to be the standard `Option` or owned `Box`
   lang-item definition.

Implement that bare lookup with resolved scope data. First inspect the
containing module's effective type-namespace children for an own/imported/glob
binding of the identifier; a compiled source has already resolved any
ambiguity there. If no such binding exists, inspect the source-visible extern
prelude before the standard implicit prelude. An implicit `--extern` binding,
or an explicit root `extern crate ... as Option`/`Box` alias inserted into the
extern prelude, shadows the standard-prelude constructor even though it is not
a child of the nested containing module. Treat that resolved crate binding as
the wrong `Res` and reject it; do not skip directly to the standard prelude.
Only if neither the containing-module scope nor the extern prelude binds the
name, consult the active standard implicit prelude selected by rustc for that
crate/module and edition, including the `core` rather than `std` prelude in a
`#![no_std]` crate. Do not assume that prelude enabled means both constructors
exist: `Option` is available from the core prelude, while owned `Box` is
unresolved in the tested `#![no_std]` scope. Compare the resulting definition
to the exact lang item.

Determine prelude enablement for the containing module, not once for the crate.
The effective status is disabled when crate-level
`#![no_implicit_prelude]` or `#[no_implicit_prelude]` on any module in the
containing module's ancestor chain applies to it. The chain includes the
containing module itself. Do not implement a shortcut that inspects only crate
attributes or only the crate plus the immediate containing-module attributes.

Centralize which checks are required in one pure helper over `PtrKind`; do not
repeat constructor-detection conditionals across parameter, return, and local
rendering. Its exhaustive mapping is:

| `PtrKind` | Require `Option` | Require `Box` |
| --- | ---: | ---: |
| `Ref(_)` | no | no |
| `Raw(_)` | no | no |
| `Slice(_)` | no | no |
| `SliceCursor(_)` | no | no |
| `OptRef(_)` | yes | no |
| `Box` | no | yes |
| `BoxedSlice` | no | yes |
| `OptBox` | yes | yes |
| `OptBoxedSlice` | yes | yes |

The helper must match every current enum variant explicitly so adding a future
variant causes a compile-time non-exhaustive-match failure until its
constructor requirements are decided. Table-test this helper independently of
rustc name resolution.

Emit only the existing bare `Option` and `Box` spellings after those checks
pass. An alias such as `Maybe` or `Owned` does not satisfy the invariant when
the conventional bare identifier is shadowed or unavailable. Do not emit an
alias or an absolute `::core::option::Option` or `::std::boxed::Box`
fallback.

Apply the precondition only to constructors actually introduced by the
selected target kind at the current parameter, return, or local. An unrelated
user type named `Option` does not block a `Box<T>` target, and an unrelated
user type named `Box` does not block an `Option<&T>` target. A kind that
introduces both constructors must satisfy both checks. On failure, return the
structured type-spelling error from A4.6 before emitting a partial record.

Source-AST-first handling remains authoritative for reused pointee syntax, but
it does not override this precondition for a newly introduced standard
constructor.

Normalizing or renaming colliding user items before local transformation could
make more inputs satisfy this invariant. Record that as a deferred future
pre-local-transformation step; do not implement it in Amendment 4.

### A4.6 Fallible generation and atomic errors

Add one general skeleton-generation error kind for any required type spelling
that cannot be emitted, named `GenerationErrorKind::TypeSpelling`. Use it both
for a genuinely unrenderable semantic type and for an introduced
`Option`/`Box` prelude-invariant failure. The error identifies:

- the enclosing function path;
- whether the location is a parameter, return, or local binding;
- the parameter or binding name when one exists;
- the semantic type or definition path useful for diagnosis; and
- the reason no valid spelling could be produced.

Keep the existing `GenerationError` field shape. `function_path` remains its
dedicated field; encode the precise parameter/return/local location and reason
in `message` rather than adding another schema field.

Make target-signature application, target-type construction, and
skeletonization propagate this error through `make_function_record` and
`make_skeletons`. Do not panic, parse a knowingly invalid placeholder, omit a
required simple-local type, or return partial records/JSON.

For a prelude-invariant failure, the reason names the required constructor,
the selected pointer kind, and whether the ordinary prelude is disabled, the
bare identifier is unresolved, or the bare identifier resolves to a different
definition. For a collision, include the conflicting resolved definition in
the diagnostic. The existing function-local-item, empty-statement,
control-shape, and AST/HIR mapping errors retain their kinds and behavior.
The general type-spelling path is also the defensive boundary for opaque,
projection, inaccessible, or otherwise unsupported semantic types; it does
not expand the supported source-written-generic input model.

### A4.7 Implementation boundary

Implement the scope policy in the Crat tools skeleton path under
`proctor/stages/crat/crates/tools/src/skeleton.rs`, and put the narrowly shared
fallible semantic formatter under `crates/utils/src/ir/` so the tools path and
legacy formatter use one MIR type grammar.

Use one function-scoped, immutable context (the exact type name is
implementation-local) containing `TyCtxt`, the function and containing-module
`DefId`s, the `AstToHir` mapping, the deterministic type-namespace candidate
table, effective-prelude state, and source-visible external-crate roots. Give
it three fallible operations:

1. shorten a cloned mapped source path only after exact `Res` validation;
2. render a semantic nominal `DefId` through a short candidate or validated
   absolute fallback; and
3. construct the final pointer target AST type after checking its introduced
   constructor requirements.

Keep source-AST cloning and semantic formatting as distinct entry paths that
share operations 2 and 3. Do not force source syntax through a semantic
round-trip merely to reuse the formatter.

Pass the containing function `LocalDefId`, `AstToHir`, and optional source AST
type hint explicitly to target-type construction. Do not put resolver state in
process globals and do not rerun rustc or parse `use` items textually.
Compute or query the effective ordinary-prelude status for that containing
module by accounting for crate-level disablement and every ancestor module
through the containing module itself. Do not cache one crate-wide boolean or
check only the crate and immediate containing-module attributes.

Change the tools target-type boundary from `Ty` to `Result<Ty,
GenerationError>` (or an equivalent local error converted at that boundary).
Signature parameters are processed in source order, then the return, while
locals retain statement order; the first failure aborts the current function
and `make_skeletons` returns no record vector. Build a record only after its
entire signature and body types succeed, so constructor checks cannot leak a
partial signature or JSON value.

When target-signature application changes an explicit raw-pointer return,
pass the mapped source return AST type as its own A4.3 hint, independently of
the input hint. Peel and reuse the return pointee's resolved alias identity
before applying the existing output lifetime plan. A shared returned-borrow
lifetime must not cause either the input or output pointee spelling to be
re-rendered from the normalized semantic type.

In `Skeletonizer::flat_map_stmt`, when a pointer decision changes an explicitly
typed raw-pointer local, pass that local's mapped source AST type as the A4.3
hint. Peel and reuse its pointee syntax exactly as for a parameter; do not
normalize a source alias to the semantic underlying nominal type. Inferred
pointer locals continue to use A4.4 because they have no explicit source type
hint.

Apply the same source-hint path before wrapping a pointee in introduced
`Box<T>` or `Option<Box<T>>`. Explicit `OptBox` parameters and returns and
explicit `Box`/`OptBox` locals retain a mapped pointee alias exactly as
references do; the wrapper kind does not authorize semantic normalization of
the pointee.

The source-AST shortener and semantic renderer must share one candidate
selection implementation. The candidate table is immutable for a function and
may be constructed once, then reused for its signature and locals. Earlier SCC
replacement cannot invalidate it: skeleton generation occurs once against the
prepared source, module items/imports are preserved, and LLM output cannot add
module/type items.

Keep the existing bare standard-constructor AST construction and add only the
A4.5 precondition in the tools skeleton path. Do not add constructor aliases or
qualified constructor paths, and do not modify item-replacer constructor
recognition, wrapper helpers, conversion logic, or schemas.

The fixed two-argument `main_0` `argv` override contains only primitive types
and remains exactly `&mut [&mut [i8]]`.

### A4.8 Superseded expectations and required regression scope

Implement every case in `amendment-4-test-plan.md`. Update implementation tests
whose exact target strings are superseded, without editing any historical test
plan.

In particular:

- the tools skeleton target in the completed initial-decision regression
  changes from `&mut crate::Tree` to `&mut Tree`, while the ordinary pointer
  rewriter's existing `*mut crate::Tree` output remains unchanged;
- the comprehensive nested-module skeleton target changes from
  `&crate::model::Point` to `&Point`; and
- inferred locals in the motivating nested module use `cb_rgb`, never
  `src::lib::cb_rgb`.

The focused regression suite must cover:

- the motivating nested-module inferred local;
- same-module definitions;
- direct, renamed, and glob imports;
- own-definition/source-alias/transparent-alias precedence and a same-spelled
  value/type namespace case;
- source pointee aliases and valid relative paths;
- shortening of a qualified source alias through a proven one-segment binding,
  plus semantic fallback when an alias hides the pointer itself;
- a transformed statement with an unchanged explicit nominal local type;
- source-AST alias identity for both sides of an explicit pointer
  input/return pair using distinct source-visible aliases, preserving the
  existing shared returned-borrow lifetime plan and proving that the return
  does not reuse the input hint;
- source-AST alias identity for an explicitly typed pointer local transformed
  through `Skeletonizer::flat_map_stmt`, using a distinct local alias
  separately from parameter hints and inferred-local semantic rendering;
- source-AST alias identity beneath introduced `Box` and `OptBox` wrappers for
  explicit parameters, returns, and locals, with distinct aliases pinning each
  source-hint location;
- multiple aliases and deterministic selection;
- same-spelling collisions that require a `crate::` fallback;
- a local public re-export whose definition path traverses a private module;
- a locally accessible `pub(crate)` re-export, shortest-route selection, and a
  lexicographic tie between equally short local re-export routes;
- an external public re-export whose definition path traverses hidden
  implementation modules;
- an external crate available only through an explicit `extern crate ... as
  ...` source alias, with an absolute alias-rooted fallback and deterministic
  extern-root selection;
- compound pointees and inferred raw-pointer locals;
- recursive semantic-formatting coverage for generics, nested pointers,
  references, slices, arrays, singleton tuples, Rust- and C-ABI ordinary
  function pointers, and an unsupported higher-ranked binder error, with an
  exact table preserving the legacy `mir_ty_to_string` API/output;
- parameter, return, and local pointer targets;
- returned-borrow lifetimes;
- raw identifiers in both one-segment and qualified fallback paths;
- direct source-hint target construction for every current `PtrKind`, including
  both boolean payload values;
- one centralized exhaustive `PtrKind`-to-standard-constructor requirement
  mapping, table-tested for every current variant;
- ordinary unshadowed prelude `Option` and `Box` targets retaining their
  existing bare spelling;
- successful bare `Option` resolution through the core prelude in a
  `#![no_std]` crate, while bare `Box` remains unsupported there unless
  otherwise bound;
- successful exact-lang-item resolution through explicit bare standard
  imports;
- atomic type-spelling failure when a selected target kind needs a bare
  `Option` or `Box` that is shadowed, even when a standard alias is present;
- renamed-import and glob-import constructor collisions, and an `OptBox` case
  where `Option` is standard but `Box` is shadowed;
- separate parameter, return, and local-binding error locations, including an
  owned local in a function whose signature contains no pointer;
- unresolved bare `Box` under `#![no_std]` and `Box` rejection under
  crate-level `#![no_implicit_prelude]`;
- rejection when the extern prelude binds a required bare constructor name to
  a crate before standard-prelude lookup;
- atomic type-spelling failure when a selected target kind needs either
  constructor in a containing module whose ordinary prelude is disabled by
  crate-level `#![no_implicit_prelude]` or `#[no_implicit_prelude]` on any
  ancestor module, including the containing module itself;
- deterministic first-error selection in parameter source order and local
  statement order;
- collisions that are irrelevant to the selected target kind remaining
  accepted, with the pointer decisions asserted before each invariant result;
- preserved parents whose nested inferred locals remain unmaterialized;
- unnameable semantic types producing an atomic structured error;
- unchanged ordinary pointer-rewriter output and unchanged protocols; and
- validation, replacement, and a second compiler invocation proving that a
  generated spelling remains valid when inserted in its original module.
