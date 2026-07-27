# Phase 1 Detailed Plan: Skeleton JSON Generation

This is a historical implementation plan. Its substantive text was moved
verbatim from the former consolidated `prototype-plan.md`; imperative and
future-tense wording describes the work assigned at the time. New navigation
notes identify where later work changed an earlier component.

See the [historical overview](prototype-plan.md#phase-1-skeleton-json-generation)
and [Phase 1 test plan](phase-1-test-plan.md).

## Crat operation

### 4.1 Generate skeletons

Expose this operation as:

```text
crat-tool make-skeleton --output <skeletons.json> <input-project>
```

Input:

- Rust project directory path.

The project manifest must identify one library source path, and that path must
be the single transformed physical source file described in
[Section 2.1](prototype-plan-misc.md#21-supported-program-model). Both
library and executable targets satisfy this contract. In the supported
executable layout, Cargo also has one generated bin source that only calls the
library's excluded `main`; `make-skeleton` neither reads nor transforms that
file.

Output:

- `skeletons.json`, containing item metadata, dependencies, annotated source functions, and annotated target skeletons.

The production command may read the project and write the requested JSON file.
The underlying library entry point must instead accept the exact source text
plus the rustc context and return owned records or JSON without receiving a
path, writer, or project directory. Phase 1 tests call only this in-memory entry
point with `run_compiler_on_str`; CLI process and filesystem wiring are not
tested in Phase 1.

A suitable API shape is:

```rust,ignore
pub fn make_skeletons(
    source: &str,
    tcx: TyCtxt<'_>,
) -> Result<Vec<ItemRecord>, GenerationError>;
```

Serialization can be a second pure function. The executable reads the same
library source file it passes to `run_compiler_on_path`, captures that exact
text for the callback, serializes the returned records, and writes only the
requested output path.

## 5. Skeleton generation

### 5.1 Analysis reuse

Crat runs its existing whole-program pointer analysis.

The pointer-replacer crate must expose the initial pointer decisions required
by the tools crate. Add and re-export this programmatic option:

```rust,ignore
#[derive(Debug, Clone, Copy, Default)]
pub struct PointerDecisionOptions {
    pub assume_nonnegative_offsets: bool,
}
```

The option defaults to `false`. Existing `replace_local_borrows` uses that
default and remains behaviorally unchanged. The tools-side initial-decision
entry point sets it to `true`; in that branch it does not call
`offset_sign_analysis` and instead installs `OffsetSignResult::default()`.
Keep this as a programmatic decision option rather than a deserialized field in
the normal Crat pointer configuration; `crat` must not opt into it accidentally.
The empty `access_signs` and `field_access_signs` sets are consumed by the
existing decision code as “needs no cursor.” Production function-pointer
grouping and rewrite-decision code consume this shared analysis result and do
not require a separate sign-analysis call.

The exposed result must correspond to the rewriter's initial decision boundary:

1. construct `LifetimePlans`;
2. construct the initial `SigDecisions`;
3. compute the initial `collect_diffs` binding decisions; and
4. snapshot/return those decisions before any `downgrade_*` loop, forced-raw
   normalization, unsupported-rewrite fallback, or AST expression rewrite.

The tools crate must use this API rather than reimplementing pointer analysis.
It is acceptable to change visibility or add owned public result types in
`pointer_replacer`. Prefer returning the smallest representation needed by
the tools crate rather than exposing the entire transform visitor.

The public initial-decision entry point must accept
`&pointer_replacer::Config`, `PointerDecisionOptions`, and `TyCtxt`, and return
the signature decisions (including planned lifetimes) plus the binding/local
decisions needed by skeleton generation. Refactor shared analysis setup as
needed so both this entry point and `replace_local_borrows` use one decision
implementation; the existing rewriter passes `PointerDecisionOptions::default()`.

Phase 1 does not read `proctor.toml` or a Crat pointer configuration. The
`make-skeleton` implementation uses `pointer_replacer::Config::default()`;
in particular, `c_exposed_fns` is empty. The separate
`PointerDecisionOptions` selects only the tools-side nonnegative-offset policy
and must not silently change any other pointer-analysis setting.

The analysis result determines:

- target parameter types;
- target return types;
- target local-variable types.

No expression rewriting is performed during skeleton generation.

### 5.2 Function skeletons

For every transformable function, Crat generates:

- the complete source function;
- the complete target skeleton.

The target skeleton:

- materializes an explicit type for every simple local binding, using the
  analyzed target type for pointer locals;
- preserves an existing explicit local type's surface syntax unless a pointer
  decision changes that local's representation;
- preserves the source statement and control structure;
- uses parseable placeholders such as `todo!()` for unimplemented expressions;
- does not apply transformation-time pointer demotion.
Control structures are preserved at any nesting depth in the statement/control
tree, including their branch, body, pattern, and nested-statement layout. A
control or block expression may be preserved only when it is the root of a
supported statement payload, initializer, return value, break value, or match
arm block tail. A control/block expression nested beneath a non-control wrapper
such as a call, constructor, operator, tuple, cast, or assignment is rejected
with a structured generation error. A later normalization pass may hoist such
expressions before Phase 1. Conditions, scrutinees, iterators, initializers,
assignments, calls, return values, and other non-control expression payloads
are replaced with `todo!()` as appropriate.

Annotated source functions, item definitions, declarations, and target
skeletons are produced from a separately parsed copy of the exact single-file
source text. Use the unexpanded surface AST; do **not** use
`utils::ast::expanded_ast(tcx)` as the JSON-rendering AST. This preserves
surface attributes such as `#[derive(Clone, Copy)]` and any syntax restored by
`unexpand`. Pretty-printing may normalize whitespace and discard comments, so
byte-for-byte formatting is not preserved.

rustc HIR/MIR remains authoritative for resolution, dependencies, inferred
local types, and pointer decisions. Map the separately parsed surface AST
directly to HIR with `utils::ir::AstToHirMapper`: construct the mapper with the
current `TyCtxt` and call
`map_crate_to_mod(&mut surface_crate, tcx.hir_root_module(), false)`. Do not use
`utils::ast::make_ast_to_hir`, which selects expanded-AST behavior. The mapper
structurally walks the surface AST and HIR, assigns fresh AST `NodeId`s, and
records item-to-owner mappings in `AstToHir::global_map` and local-node
mappings in `AstToHir::local_map`; it does not require `NodeId` or `Span`
identity across parse sessions.

The unexpanded mode must be propagated recursively through every inline
module. In particular, change the mapper's current inline-module recursion so
it forwards the caller's `expanded` flag instead of hard-coding `true`. With
`expanded == false`, skip HIR items recognized by
`TyCtxt::is_automatically_derived`; the corresponding surface type item and
its `#[derive(...)]` attribute remain in the presentation AST. Under the Phase
1 assumption above, generated derive items therefore neither receive records
nor shift the mapping of following source items. Any other structural mismatch
is a generation error; do not fall back to path, source-order, or cross-session
span heuristics.

Use `global_map` to associate each included surface item with its
`LocalDefId`, and `local_map` to associate simple surface bindings with their
`HirId`. Obtain inferred binding types from HIR type checking. Pointer
decisions exposed to the tools crate should remain keyed by the same HIR owner
or binding identity. If a MIR local is needed, chain the binding `HirId`
through `utils::ir::map_thir_to_mir(...).binding_to_local`. Collect item
dependencies by traversing the mapped owner's HIR, not by resolving the
presentation AST; this also accounts for dependencies inside expanded
expression-position macros whose internal nodes are absent from the surface
AST.

Before rendering any JSON Rust snippet, clone/sanitize the presentation AST:

- remove `#[no_mangle]` attributes; and
- change every source-defined explicit function ABI to the default Rust ABI.

Apply this to `annotated_source`, `annotated_skeleton`, `source_signature`,
`target_signature`, and any emitted declaration/definition on which the
attribute can occur. Preserve visibility, `unsafe`, parameter names, all other
function attributes, and type-item attributes including `derive` and `repr`.

### 5.3 Statement labels

Every `rustc_ast::Stmt` in a transformable function receives a numeric annotation:

```rust
#[proctor(0)]
```

This includes tail expressions represented by `StmtKind::Expr`.

Label properties:

- labels are unique within one function;
- labels do not need to be globally unique;
- labels appear in source functions and skeletons;
- labels are retained during validation;
- labels are removed before insertion into a compilable project.

An input containing an empty statement is rejected because an empty
`rustc_ast::Stmt` has no node on which to attach the annotation.

An input containing a `match` arm whose body is not a block expression is also
rejected. Supported arms have the form `pattern [if guard] => { ... }`, so every
statement, including a block tail expression, can receive its own label.

## 6. Skeleton JSON

The skeleton output, conventionally named `skeletons.json`, is a top-level JSON
array containing a flat list of included items. Item-kind strings use the
PascalCase names listed below.

Every included item has:

- `id`: a unique natural-number ID;
- `path`: its crate-relative full Rust path;
- `kind`: its item kind.

The item ID is the primary identity for dependency analysis and validation. Full paths are used for diagnostics and source replacement.

IDs are assigned deterministically in recursive source order. Dependency lists
are deduplicated and sorted by item ID.

### 6.1 Included item kinds

Include only:

- `Fn`; its final inclusion rule is preserved in
  [Phase 2's changes to skeleton generation](phase-2-plan.md#changes-to-skeleton-generation);
- `Static`;
- `Const`;
- `TyAlias`;
- `Enum`;
- `Struct`;
- `Union`.

All other item kinds receive no records. In supported Phase 1 input this
principally covers `ExternCrate`, `Use`, inline `Mod`, `ForeignMod`, and
`GlobalAsm`. `ConstBlock` is an expression kind, not an item kind, in the
pinned Rust compiler and is therefore not part of this item-filtering list.
Traits, impls, macro definitions, item-producing
macro invocations, external modules, and `cfg`-selected alternatives are
excluded by the input contract rather than being features that Phase 1 must
validate exhaustively.

Although `Mod` does not receive a record, Crat must recursively traverse modules to discover included items and construct full paths.

References to excluded items are ignored for dependency-context generation.
References to struct constructors, enum variants, and fields are canonicalized
to the containing included type item.

The executable dependency exception added during Phase 3 is preserved in
[Phase 3's changes to skeleton JSON](phase-3-plan.md#changes-to-skeleton-json).

### 6.2 Dependency namespaces

Rust distinguishes value and type namespaces. The JSON should reflect this distinction.

For `Fn`, `Static`, and `Const`, include two dependency lists:

- `signature_dependencies`;
- `dependencies`;

For `TyAlias`, `Enum`, `Struct`, and `Union`, include:

- `dependencies`;

All dependency lists contain item IDs and record only direct dependencies.

#### Signature dependencies

For `Fn`, `Static`, and `Const`, signature dependencies are items referenced by the source declaration signature.

Examples include:

- a type used in a parameter or return type;
- a constant used in an array type such as `[T; N]`.

#### Dependencies

Dependencies are items referenced anywhere in the material used when the item is a transformation target.

For a `Static` or `Const`, this includes its initializer even though the emitted
declaration signature omits the initializer.

For a function, this includes:

- the source signature;
- the source body.

Therefore:

```rust
fn f(x: T) {
    let y: S = todo!();
}
```

has `T` in its (signature) dependencies, while `S` appears only in its dependencies.

The following subset relation should hold:

```text
signature_dependencies ⊆ dependencies
```

### 6.3 Function records

Each `Fn` record contains:

- `id`;
- `path`;
- `name`: the final Rust function identifier;
- `kind`;
- `annotated_source`;
- `annotated_skeleton`;
- `source_signature`;
- `target_signature`;
- the two dependency lists defined above.

A directly recursive function includes its own ID in `dependencies`.

Serialize keys in this order:

```text
id, path, kind, name, annotated_source, annotated_skeleton,
source_signature, target_signature, signature_dependencies, dependencies
```

### 6.4 `Static` and `Const` records

Each `Static` or `Const` record contains:

- `id`;
- `path`;
- `kind`;
- `declaration`: declaration signature without the initializer;
- the two dependency lists.

Serialize keys in this order:

```text
id, path, kind, declaration, signature_dependencies, dependencies
```

Examples:

```rust
static X: i32;
static mut BUFFER: *mut u8;
const SIZE: usize;
```

### 6.5 Type-item records

Each `TyAlias`, `Enum`, `Struct`, or `Union` record contains:

- `id`;
- `path`;
- `kind`;
- `definition`: complete original surface definition;
- `dependencies`;

Serialize keys in this order:

```text
id, path, kind, definition, dependencies
```

Type items are always supplied as complete definitions and are never transformed.
Attributes such as `#[derive(Clone, Copy)]` are part of that definition and are
preserved.

Serialize the top-level record vector with pretty JSON equivalent to
`serde_json::to_string_pretty`, with no trailing newline. No variant emits
undefined fields or `null` placeholders.

## Implementation sequence

### Phase 1: Skeleton JSON generation (completed; historical)

Phase 1 has already been implemented and validated. Its historical scope was:

- `crates/tools` as a workspace library and `src/bin/crat-tool.rs` as the thin
  executable;
- reuse of current pointer analysis through a tools-facing initial-decision
  API;
- a default-off nonnegative-offset option that skips offset-sign analysis only
  in tool mode and cannot produce `SliceCursor` decisions;
- disabling transformation-time demotion;
- parsing and rendering the exact unexpanded single-file surface AST, mapping
  it structurally to HIR with `AstToHirMapper` in recursively propagated
  unexpanded mode, and using HIR/MIR for resolution and types;
- skipping all derive-generated HIR items recognized as automatically derived
  without losing their surface `#[derive(...)]` attributes or shifting later
  item mappings;
- free-function skeleton generation plus context records for the included
  value and type items;
- `todo!()` placeholders;
- statement annotations;
- explicit errors for empty statements, non-block `match` arms, and control or
  block expressions nested beneath non-control expression wrappers;
- source and target signatures;
- render-only removal of `#[no_mangle]` and every explicit function ABI, with
  all project and analysis input left unchanged;
- preservation of `derive`, `repr`, and other supported surface attributes;
- included-item filtering;
- dependency lists in value and type namespaces;
- JSON serialization.

The completed implementation passed `phase-1-test-plan.md`. That historical
plan is not an input to the Phase 2 implementation and must not be edited.
Phase 2 updates the existing Rust test oracles affected by the explicit
generator adjustments below and adds their new regressions only to
`phase-2-test-plan.md`.

The completed Phase 1 verification commands were:

```text
cargo fmt
cargo clippy --workspace --all-targets
cargo test --workspace
```
