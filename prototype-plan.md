# Initial Prototype Plan: LLM-Based Local Pointer Transformation

## 1. Purpose

This document specifies the first implementation milestone for the local-transformation component of Proctor.

It is intended to be used together with:

- the hybrid C-to-Rust research plan; and
- the Proctor pipeline component specification.

Those documents describe the overall research motivation and cross-component interfaces. This document focuses only on implementing the initial local-transformation prototype.

Where this document narrows a more general behavior in `research-plan.md` or
`proctor-spec.md`, this prototype document is normative for the initial
prototype. `phase-1-test-plan.md` is the exhaustive executable contract for
Phase 1.

Phase 1 was implemented and validated before the Phase 2 design was finalized.
Phase 2 includes four intentional adjustments to the completed Phase 1
generator, all specified in Section 5.2: mark every target binding mutable;
make every target function header unsafe while preserving source safety;
exclude every free function whose final identifier is `main`; and force the
supported two-argument `main_0` target `argv` type to
`&mut [&mut [i8]]`. The implementation agent for Phase 2 must make these
generator changes and update the existing Phase 1 Rust tests as specified in
`phase-2-test-plan.md`. Do not edit or reinterpret the historical
`phase-1-test-plan.md`, and do not otherwise reopen Phase 1 behavior. Moving
the existing code and tests into the `skeleton` module is an organizational
refactor, not another Phase 1 semantic change.

`unsupported.md` is the consolidated conceptual input contract for this
prototype. Some low-level skeleton and validator behavior deliberately handles
constructs listed there (notably `ref` bindings), but that robustness does not
make those constructs supported local-transformation inputs. Phase 2 adds no
supportedness checker or normalization pass.

The prototype will validate the following loop:

1. Run Crat's existing whole-program pointer analysis.
2. Generate target function skeletons without rewriting function bodies.
3. Translate functions SCC-by-SCC with an LLM.
4. Validate each LLM result structurally with Crat.
5. Insert valid results into a fresh project.
6. Generate wrappers and redirect untransformed callers.
7. Run `cargo build`.
8. Repair failures with fresh LLM calls.
9. Continue until all function SCCs have been translated.

The prototype does **not** yet run tests, extract reusable rules, consume `proctor.toml`, or account for wrappers introduced by non-local transformations.

## 2. Scope and assumptions

### 2.1 Supported program model

The input is one compilable Rust crate produced by C2Rust. At this pipeline
point, Crat's `expand` and `unexpand` passes have already run, while `split` has
not run. Consequently, the transformed library-crate source is exactly one
physical Rust file. That file may contain inline modules, which are traversed
recursively, but it does not contain external `mod foo;` declarations. A
supported executable project may additionally contain the generated forwarding
bin source described below; that file is not transformation input.

Only source-defined free functions are transformed. Functions may appear in
nested modules. Source-defined safe and unsafe free functions are both
transformation targets, except that every free function whose final
identifier's symbol is `main` is excluded unconditionally, including the raw
spelling `r#main`. Phase 1 does not inspect a `main` body to decide whether to
exclude it.

The prototype assumes the absence of:

- all methods and `impl` items;
- all traits and trait methods;
- closures;
- source-written generics (Phase 1 may add named lifetime parameters to target
  signatures for returned borrows);
- `const fn`;
- async functions;
- source-defined variadic free functions;
- parameter patterns;
- by-reference binding modes (`ref` and `ref mut`) in source patterns, even
  though skeleton generation and structural validation handle them
  mechanically;
- function pointers;
- dynamic dispatch;
- callback patterns through `void *`;
- macro definitions and item-producing macro invocations;
- function-body item declarations other than `const` and `static`;
- `cfg` attributes;
- attributes on statements;
- explicit `unsafe` blocks; and
- empty statements.

The function-body item restriction applies to parsed item statements
(`StmtKind::Item`). Function-local `const` and `static` declarations are
supported. Function-local functions, types, structs, enums, unions, modules,
traits, impls, foreign blocks, item macros, and every other item kind are
outside the supported input model. Expression- or statement-position macro
invocations are not item declarations and remain governed by the existing
macro and skeleton rules. When a supported local `const` or `static`
initializer contains a block, its statements and bindings remain part of the
recursive Phase 1 label tree and target skeleton.

C2Rust expands C macros before this stage. `unexpand` may restore surface
syntax such as derive attributes and expression-position macro invocations.
Skeleton generation preserves that surface syntax in `annotated_source`; it
must not expand the source AST again.

Phase 1 assumes that every HIR item generated by expanding a preserved derive
attribute is recognized by `TyCtxt::is_automatically_derived`. Derives that
emit additional unmarked helper items are outside the supported input model.

Every `match` arm in supported input has a block-expression body. A non-block
arm is rejected explicitly because statements inside every arm must be
individually labelable.

Foreign functions, including libc functions, may be called or replaced by the LLM, but they are:

- never transformation targets;
- omitted from dependency context; and
- not represented as transformable function records.

Foreign declarations may be variadic. The source-defined-variadic restriction
does not apply to an `extern` declaration without a Rust body.

For an executable target, the one transformed source file is still the Cargo
library source. The additional Cargo bin source is a generated forwarding shim
that only calls the library's safe `main`; it is outside skeleton generation
and is never rewritten by this prototype. The supported library source has
exactly one of these C2Rust entry-point pairs, with `main` and `main_0` in the
same library module:

1. `unsafe fn main_0() -> core::ffi::c_int` plus a safe `pub fn main()` that
   calls `main_0` inside an unsafe block and passes its result to
   `std::process::exit`; or
2. `unsafe fn main_0(argc: core::ffi::c_int,
   argv: *mut *mut core::ffi::c_char) -> core::ffi::c_int` plus C2Rust's safe
   `pub fn main()` that constructs the raw `argc`/`argv`, calls `main_0`
   inside an unsafe block, and exits with its result.

The excluded library `main` and its explicit unsafe block are a mechanically
managed executable compatibility boundary. They are the sole exception to the
general explicit-unsafe-block input restriction. Other `main` bodies and other
`main_0` signatures are unsupported, but Phase 1 does not enforce that
supportedness condition: it excludes every `main` by name and otherwise emits
ordinary records. A future supportedness checker may validate the pair.

### 2.2 Pointer-analysis scope

Crat's current pointer analysis is reused with a tools-only decision option,
conceptually `assume_nonnegative_offsets: true`. The preceding non-local
transformation is assumed to have eliminated actual negative pointer offsets.
Offset-sign analysis can conservatively report false positives, so tool mode
must skip it entirely rather than running it and trusting its result. Tool mode
uses an empty `OffsetSignResult`; all array-like borrows therefore use plain
slices, and `SliceCursor` is never a target type.

The existing `crat` pointer pass keeps its current behavior. Its default mode
continues to run offset-sign analysis and may introduce `SliceCursor`. The new
option defaults to the current cursor-aware behavior and is enabled only by
the tools-side initial-decision API. Do not obtain tool-mode output by first
choosing cursors and converting them to slices afterward; prevent cursor
decisions at their source.

For this prototype:

- named type definitions are never changed;
- global `static` and `const` types are never changed;
- function parameter types may change;
- function return types may change;
- local-variable types may change.

The prototype should initially be evaluated on programs that do not require pointer-containing structs, unions, enums, or type aliases to be transformed.

Crat's current rewriting phase may demote some analyzed pointer types back to raw pointers when rewriting is difficult. Skeleton generation must disable this transformation-time demotion and follow the initial analysis result exactly.

### 2.3 Rust assumptions

- The input project already passes `cargo build`.
- The project uses Rust 2021.
- Transformable source functions may be safe or unsafe; every target skeleton
  and inserted transformed function is `unsafe fn`.
- The LLM must not introduce explicit `unsafe` blocks.
- Unsafe operations may be used directly inside these unsafe functions.
- The prototype supports executable targets and `cdylib` library targets.

## 3. Terminology

### Source function

The function before pointer-type transformation.

### Source signature

The signature in the source function. In skeleton JSON it is a presentation
copy with `#[no_mangle]` and any explicit function ABI removed as specified in
Section 5.2;
the input project's real header is unchanged.

### Target skeleton

The function skeleton produced from Crat's pointer-analysis result.

### Target signature

The signature selected by Crat's analysis and present in the target skeleton,
using the same presentation-only ABI/export sanitization as the source
signature. It is always unsafe, and the supported two-argument `main_0`
receives the Section 5.2 `argv` override after ordinary pointer analysis.

### Transformation target

A function in the SCC currently being rewritten by the LLM.

### Dependency context

Supporting declarations supplied to the LLM but not rewritten in the current request.

### Expansion group

The consecutive output statements corresponding to one labeled source statement.

## 4. Crat layout and required operations

Add `crates/tools` as a member of the `crat` Cargo workspace. This library crate
contains the Rust-aware, in-memory implementation of skeleton generation,
validation, and, in a later phase, candidate-project generation. It may depend
on `pointer_replacer`, `utils`, and the rustc-private crates it needs.
Add `tools = { path = "crates/tools" }` to the root package dependencies so the
root binary can call the library; do not place the implementation directly in
the binary.

Keep `crates/tools/src/lib.rs` minimal. It retains required crate-level feature
attributes and `extern crate` declarations, declares modules, and re-exports
their public APIs; it contains no skeleton/validator implementation or large
test module. Move the existing skeleton-generation implementation into a
`skeleton` module and put the structural validator in a separate `validator`
module so the Phase 1 API remains source-compatible. Keep each module's tests
beside that module, for example in `skeleton/tests.rs` and
`validator/tests.rs`.

Add the executable `src/bin/crat-tool.rs` to the root `crat` package. Follow the
small command-dispatch structure used by `src/bin/crat-finder.rs`: parse
arguments, call the `tools` library, and perform only the requested outer
filesystem I/O. `make-skeleton` locates the crate's library source with
`utils::find_lib_path` and runs the compiler on that source. `validate` only
reads request JSON, calls the parser-only validator, and writes response JSON;
it does not locate or compile a project. Rust parsing and analysis belong in
the library, not in the executable.

The prototype requires three logical Crat operations, exposed through
`crat-tool` subcommands. Phase 1 implements only `make-skeleton`.

### 4.1 Generate skeletons

Expose this operation as:

```text
crat-tool make-skeleton --output <skeletons.json> <input-project>
```

Input:

- Rust project directory path.

The project manifest must identify one library source path, and that path must
be the single transformed physical source file described in Section 2.1. Both
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

### 4.2 Validate an LLM transformation

Expose this operation as:

```text
crat-tool validate --input <request.json> --output <response.json>
```

The request contains:

- an ordered list of expected functions, each containing its item ID, final
  name, and complete annotated target skeleton; and
- one LLM-generated Rust transformation snippet containing all requested
  functions.

Output:

- one machine-readable response whose status is `valid`, `invalid`, or
  `setup_error`, as specified in Section 14.

The production command reads the request JSON, invokes the in-memory validator,
and writes the response JSON. It exits with status zero after successfully
writing any of the three response statuses. It exits nonzero only when it
cannot complete the response protocol, such as invalid CLI invocation,
unreadable input, unwritable output, serialization failure, or a panic. In
particular, structural validation failure is ordinary output, not a process
failure.

Phase 2 tests exercise only in-memory request construction, parsing,
validation, and response serialization. They do not invoke the CLI or perform
filesystem I/O.

### 4.3 Create a candidate project

Input:

- current Rust project directory path;
- candidate-project destination path;
- JSON mapping from full Rust function paths to validated transformed snippets.

Output:

- a complete candidate Rust project containing replacements, wrappers, and rewritten call sites.

The Python orchestrator must not parse or rewrite Rust. All Rust-specific processing remains in Crat.

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

- uses the target signature and is always `unsafe fn`, even when the source
  function is safe;
- materializes an explicit type for every simple local binding, using the
  analyzed target type for pointer locals;
- preserves an existing explicit local type's surface syntax unless a pointer
  decision changes that local's representation;
- marks every binding identifier as mutable, regardless of its source
  mutability;
- preserves the source statement and control structure;
- uses parseable placeholders such as `todo!()` for unimplemented expressions;
- does not apply transformation-time pointer demotion.

The all-bindings-mutable rule applies recursively to every binding pattern in
the target skeleton, including function parameters, simple and destructuring
`let` patterns, `let-else` patterns, `if let` and `while let` patterns, `for`
patterns, `match`-arm patterns, and bindings nested inside supported
function-local `const`/`static` initializer blocks. Wildcards do not introduce
bindings. A by-reference binding keeps its by-reference mode and becomes
`ref mut`. `annotated_source` and `source_signature` retain the source
mutability exactly; only `annotated_skeleton` and `target_signature` receive
the permissive mutability update.

Function safety follows the same source/target separation. Preserve whether
the source function is safe or unsafe in `annotated_source` and
`source_signature`. Set the header to `unsafe fn` in
`annotated_skeleton` and `target_signature`. This is a target normalization,
not an inferred property and not a validator decision. Phase 2 ignores the
LLM-returned function's safety qualifier; Phase 3 inserts Crat's unsafe target
header.

Do not emit a `Fn` record for any free function whose final identifier's
symbol is `main`, at any inline-module depth. This includes a surface spelling
of `r#main`. The exclusion is unconditional and does not inspect the function
body. Such a function receives no item ID, has no skeleton, and is absent from
dependency lists and the function graph. `main_0` remains an ordinary
transformation target.

Apply one special target-signature override to a function whose final
identifier is `main_0` and whose source parameter and return types are
structurally exactly:

```rust,ignore
unsafe fn main_0(
    argc: core::ffi::c_int,
    argv: *mut *mut core::ffi::c_char,
) -> core::ffi::c_int
```

Preserve the source signature unchanged, but force the target parameter types
to:

```rust,ignore
unsafe fn main_0(
    argc: core::ffi::c_int,
    argv: &mut [&mut [i8]],
) -> core::ffi::c_int
```

Visibility and binding `mut` do not affect recognition; the supported
executable form itself is unsafe. The actual target rendering also applies the
all-bindings-mutable rule to both parameter patterns. The `argv` override takes
precedence over the ordinary pointer-analysis decision, including a
raw-pointer decision. It does not change `argc`, the return type, or
pointer-analysis behavior for any other function or binding. A zero-argument
`main_0` uses ordinary target-type decisions.

This mutation is intentional: the target skeleton must not prevent an LLM from
assigning to an existing binding while translating a pointer operation.
Mutability is not a semantic target decision. The Phase 2 validator therefore
ignores binding mutability everywhere and accepts either the presence or
absence of `mut` in the returned transformation. It still enforces binding
identity, declaration placement, and target types.

Supported function-local `const` and `static` items are an exception to the
ordinary expression-hole and inferred-local-type rules. Preserve each complete
item and initializer expression tree, while recursively attaching the normal
statement labels and applying only the all-bindings-mutable update inside
initializer blocks. Do not materialize new explicit types for bindings inside
an item initializer. This is the item-preservation behavior validated in
Section 13.1, not an additional Phase 1 semantic change.

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
Preserve source binding mutability in source renderings; apply the separate
all-bindings-mutable skeleton rule above to target renderings. Sanitization is
presentation-only: pointer analysis and the input project always use the
unchanged source, and later project-rewriting operations must recover
ABI/export information from the project AST rather than from the sanitized
JSON.

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

- `Fn`, except a free function whose final identifier is exactly `main`;
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
References from the excluded executable `main` are likewise omitted; its
special `main_0` migration is handled mechanically during Phase 3 rather than
through a dependency edge.

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

## 7. Python orchestrator

The orchestrator is a separate Python program.

It is responsible for:

- invoking Crat analysis and skeleton generation;
- loading the skeleton JSON;
- building the function-call graph;
- computing SCCs;
- scheduling SCCs;
- constructing dependency context;
- constructing LLM prompts;
- calling the LLM through LiteLLM;
- extracting Rust code from responses;
- invoking Crat validation;
- invoking Crat candidate-project generation;
- running `cargo build`;
- managing repair attempts;
- promoting successful candidate projects.

It must not parse Rust source.

### 7.1 LLM abstraction

Use LiteLLM directly for the first prototype.

Keep the LiteLLM-specific code behind a minimal interface, for example:

```python
class LlmClient:
    def generate(self, prompt: str) -> str:
        ...
```

Each call is independent. No conversation history is retained.

This interface should be replaceable by the team's shared LLM framework later.

## 8. Function graph and SCC scheduling

### 8.1 Function graph

Build a graph containing only transformable `Fn` items.

For each function `f`, inspect function-valued entries in its `dependencies`.

Add an edge:

```text
f -> g
```

when `f` directly calls `g`.

Foreign functions are absent from the graph.

### 8.2 SCCs

Compute strongly connected components and the SCC condensation DAG.

A leaf SCC has no outgoing edge to an unprocessed SCC.

Because edges point from callers to callees, leaf-first processing translates callees before external callers.

A singleton SCC is recursive when its function has a self-edge.

### 8.3 Deterministic scheduling

When multiple leaf SCCs are available, choose deterministically, for example by the smallest item ID in each SCC.

### 8.4 Function-name uniqueness

Before processing an SCC, the orchestrator checks that the final function names of all SCC members are distinct.

Crat identifies returned functions by name inside the single LLM response. Therefore, uniqueness is required only within the current SCC.

If duplicate function names occur within one SCC, orchestration aborts.

Functions in different SCCs may have the same final name.

## 9. Prompt-context construction

Each prompt has two conceptually separate parts:

1. **Transformation Targets**
2. **Dependency Context**

The transformation targets do not count toward the dependency-character limit.

The dependency context has a limit of 100,000 characters.

All dependency records are deduplicated by item ID and ordered deterministically by ID.

### 9.1 Transformation targets

For every function in the current SCC, include:

- its annotated source function;
- its annotated target skeleton.

These snippets already contain the source and target signatures, so those signatures are not repeated separately.

### 9.2 SCC signatures

If the SCC contains multiple functions, include the source and target signatures of every SCC member in the dependency context.

For a directly recursive singleton SCC, include its own source and target signatures.

For a nonrecursive singleton SCC, omit the redundant self-signature dependency.

### 9.3 Value dependencies

For each transformation target:

- inspect its `dependencies`.

For a function dependency, include:

- source signature;
- target signature.

For a `Static` or `Const` dependency, include:

- declaration signature.

Then, follow a dependency's signature dependencies only.

Therefore, if:

```text
a -> b -> c
```

the prompt for `a` includes `b`'s source and target signatures but not `c`'s signatures unless `c` appears in `b`'s signature.

### 9.4 Type dependencies

Type dependencies are followed transitively, as they have only `dependencies` but no `signature_dependencies`.

### 9.5 Transitive closure

Traverse dependencies breadth-first.

Examples:

- if target function `a` calls `b` and `b`'s signature refers to `T`, include `T`;
- if `a`'s refers to `S`, include `S`;
- if `T` refers to `U`, include `U`;
- continue transitively subject to the character budget.

### 9.6 Dependency budget

The 100,000-character budget applies only to the rendered dependency-context section.

Use this policy:

1. Add mandatory SCC signature dependencies.
2. Add all direct dependencies.
4. Tentatively add the complete next breadth-first depth.
5. Keep the depth only if the total dependency context remains within 100,000 characters.
6. Continue until the next complete depth does not fit.
7. Once a depth is rejected, do not consider deeper depths.

If mandatory direct dependencies already exceed 100,000 characters, abort the SCC instead of silently omitting them.

Instructions, transformation targets, prior failed code, and diagnostics are outside this limit.

## 10. Initial LLM prompt template

Use a prompt following this structure.

````text
You are transforming functions in unsafe Rust code generated from C.

Definitions:

- Source code is the current function implementation before pointer-type
  transformation.
- The source signature is the signature in the source code.
- The target skeleton was generated from whole-program static pointer analysis.
- The target signature is the signature in the target skeleton.
- A dependency's source signature shows its signature before transformation.
- A dependency's target signature shows how transformed code must call it.

Implement every function in the Transformation Targets section.

Requirements:

1. Preserve the exact behavior of the source code.
2. Strictly follow the parameter types, return type, and local-variable types
   fixed by the target skeleton. The skeleton deliberately marks every binding
   `mut`; you may keep or remove `mut` as needed because binding mutability is
   not validated.
3. Call transformed function dependencies using their target signatures.
4. Do not change any existing function name, parameter name, local-binding name,
   or item name. Preserve each existing declaration exactly once in its
   original label and branch, arm, loop, or block role. Preserve every existing
   function-local `const` or `static` declaration exactly, including its
   attributes, type, `static mut` status, and initializer, apart from the
   permitted binding-`mut` differences described above.
5. New local bindings may be introduced only with names of the form
   `proctor_temp_var_n`, where `n` is a nonnegative integer.
6. A new temporary binding may be used only within the labeled expansion group
   in which it is declared, including unlabeled nested code inside that group.
7. Do not define additional functions, types, statics, constants, modules, or
   other items.
8. Every source statement label must appear at least once in the output.
9. A source statement may expand into one or more consecutive sibling statements
   with the same label.
10. Repeated occurrences of one label must be consecutive siblings at the same
    statement-list level. Do not repeat the same label in nested statements.
11. Newly introduced nested statements must be unlabeled.
12. Preserve the order of existing labels.
13. Preserve every existing control-structure kind and its existing labeled
    branch/body structure. Conditions, scrutinees, patterns, and statement
    contents may be rewritten.
14. If a labeled control statement expands into multiple sibling statements,
    exactly one sibling must preserve the original control structure. The other
    siblings must not be control-root statements and must contain no existing
    labels.
15. Do not introduce an explicit `unsafe` block. These functions are already
    unsafe functions, and unsafe operations may be used directly when needed.
16. Do not add statement or expression attributes other than the required
    `#[proctor(N)]` statement labels.
17. Return exactly one fenced Rust code block containing all requested functions.
    Do not return prose.

Example:

Source:

```rust
unsafe fn read_value(p: *const i32) -> i32 {
    #[proctor(0)]
    let x: i32 = *p.add(1);
    #[proctor(1)]
    x
}
```

Target skeleton:

```rust
unsafe fn read_value(mut p: &[i32]) -> i32 {
    #[proctor(0)]
    let mut x: i32 = todo!();
    #[proctor(1)]
    todo!()
}
```

Valid output:

```rust
unsafe fn read_value(p: &[i32]) -> i32 {
    #[proctor(0)]
    let x: i32 = p[1];
    #[proctor(1)]
    x
}
```

Dependency Context:

{{DEPENDENCY_CONTEXT}}

Transformation Targets:

{{TRANSFORMATION_TARGETS}}

{{REPAIR_CONTEXT}}
````

For an initial request, `REPAIR_CONTEXT` is empty.

For a repair request, use:

````text
The previous transformation failed.

Previous transformation:

```rust
{{FAILED_TRANSFORMATION}}
```

Diagnostics:

```text
{{DIAGNOSTICS}}
```

Regenerate every function in the Transformation Targets section.
````

Each repair request uses the complete original prompt plus the latest failed transformation and latest diagnostics.

## 11. LLM response extraction

The orchestrator instructs the LLM to return exactly one fenced Rust code block and no prose.

To tolerate formatting errors:

1. Find all fenced code blocks.
2. Ignore prose outside code blocks.
3. If one block exists, use it.
4. If multiple blocks exist, choose the longest.
5. If multiple longest blocks have equal length, choose the first.
6. If no fenced code block exists, report a structural failure.

Pass the selected block unchanged to Crat.

The orchestrator does not parse Rust.

## 12. Structural model

### 12.1 Expansion groups

Each labeled target-skeleton statement maps to a nonempty expansion group in
the output.

An expansion group consists of one or more consecutive sibling statements:

- carrying the same target-skeleton label;
- at the same statement-list level.

Every statement in a group has exactly one outer `#[proctor(N)]` attribute.
The token spelling of `N` must match the canonical unsuffixed decimal grammar
`0|[1-9][0-9]*`, and its value must be in the `u32` range. Leading zeroes,
numeric separators, signs, radix prefixes, and integer suffixes are rejected;
for example, `00`, `1_0`, `+1`, `0x1`, and `0u32` are malformed. A label may
not be repeated in nested statements. A `proctor` attribute with a malformed
path, argument, argument count, or integer token is an item-specific validation
error rather than an unknown label. Validate the original literal-token
spelling before or alongside numeric conversion; checking only a normalized
AST integer value is insufficient because it can erase separators, radix, or
suffix distinctions.

At every statement-list level corresponding to a target-skeleton statement
list, the result consists only of the expected expansion groups in expected
order. It may not contain an unlabeled sibling between groups. Unlabeled
statements may occur only inside newly introduced nested code that is itself
inside one expansion-group statement and does not correspond to an existing
target-skeleton statement list.

### 12.2 Leaf statements

If a labeled target-skeleton statement has no existing labeled descendants, it
may:

- remain one labeled statement;
- expand into multiple consecutive same-label sibling statements;
- introduce new nested expressions, blocks, or control flow internally.

Any newly introduced nested statements must be unlabeled.

Valid:

```rust
#[proctor(1)]
let x = if p.is_some() {
    *p.unwrap()
} else {
    0
};
```

Valid one-to-many expansion:

```rust
#[proctor(1)]
let proctor_temp_var_0 = p.unwrap();

#[proctor(1)]
let x = *proctor_temp_var_0;
```

Invalid nested repetition:

```rust
#[proctor(1)]
let x = if p.is_some() {
    #[proctor(1)]
    *p.unwrap()
} else {
    #[proctor(1)]
    0
};
```

### 12.3 Existing control structures

Phase 1 preserves a control or plain block expression only when it is reached
through one of these statement roles:

- the root expression of an expression or semicolon statement;
- the direct initializer of a `let` statement;
- the direct value of `return`;
- the direct value of `break`; or
- the block tail of a supported match arm.

These wrappers are structural. The output must preserve the same role: for
example, a `return if ...` remains a return whose direct value is an `if`, and
a `let x = loop ...` retains the existing `x` declaration with a direct `loop`
initializer. The validator does not look through calls, constructors,
operators, tuples, casts, assignments, or other non-control expression
wrappers to find a control root.

When a target-skeleton statement has such an existing control root, the output
must preserve its control kind. The following kinds are distinct:

- plain block;
- `if`;
- `if let`;
- `while`;
- `while let`;
- `for`;
- `loop`;
- `match`.

`let-else` is a distinct structural statement form rather than a control-root
expression. Its initializer role, else-block role, and recursively labeled
descendants must be preserved.

Examples:

- `if` may not become `if let` or `match`;
- `while` may not become `for` or `loop`;
- `match` must remain `match`;
- a plain block may not become a loop; and
- `let-else` may not become a plain `let` followed by an `if`.

The output may rewrite:

- conditions;
- scrutinees;
- patterns;
- branch bodies;
- loop bodies;
- match-arm bodies.

It must preserve:

- the original control kind;
- whether an `if` has an `else`;
- the complete recursive `else if` structure, even though an `else if`
  expression has no independent statement label;
- the number and order of match arms;
- the presence or absence of each match-arm guard;
- the existence and role of `let-else`, branch, loop, match-arm, and plain
  block bodies;
- existing labeled descendants and their placement.

Patterns may be rewritten, but Section 13 still requires every binding
declaration in an existing pattern to remain exactly once in the corresponding
pattern role and permits only correctly named generated temporaries as new
bindings.

### 12.4 Expanding a control statement

A labeled control statement may expand into multiple consecutive same-label sibling statements.

Exactly one sibling must:

- be a control-root statement;
- preserve the original control kind;
- preserve the original statement role;
- contain all recursively preserved labeled descendants.

Every other sibling must:

- not be a control-root statement;
- contain no existing labels.

Valid transformation of an original `if`:

```rust
#[proctor(1)]
let proctor_temp_var_0 = p.is_some();

#[proctor(1)]
if proctor_temp_var_0 {
    #[proctor(2)]
    x = *p.unwrap();
} else {
    #[proctor(3)]
    x = 0;
}

#[proctor(1)]
let y = x + 1;
```

Invalid because two same-label siblings are control-root statements:

```rust
#[proctor(1)]
if condition_a {
    #[proctor(2)]
    x = *p.unwrap();
} else {
    #[proctor(3)]
    x = 0;
}

#[proctor(1)]
if condition_b {
    x += 1;
}
```

Invalid because the original control kind changed:

```rust
#[proctor(1)]
while condition {
    #[proctor(2)]
    x = *p.unwrap();
}
```

### 12.5 Label-order rules

For each target-skeleton statement list:

- every original label appears at least once;
- no new label appears;
- expansion groups occur in original label order;
- each expansion group is consecutive;
- no label may reappear after another label begins.

If the original order is:

```text
1, 2
```

then these are invalid:

```text
2, 1
```

and:

```text
1, 2, 1
```

### 12.6 Preservation of labeled descendants

Existing labeled descendants must remain recursively associated with their original labeled ancestor.

They may not be:

- moved to another branch;
- moved to another match arm;
- moved outside their original control statement;
- wrapped by a newly introduced labeled ancestor;
- duplicated;
- removed.

New unlabeled nested code may be introduced inside an expansion group.

Association is checked by structural role, not only by preorder. A descendant
must remain in the same `if` branch, loop body, `let-else` body, plain block,
or match-arm index. Moving a label between two branches while preserving
global preorder is invalid.

## 13. Identifier and temporary-variable rules

### 13.1 Existing identifiers

The transformation must preserve each existing declaration identity, not just
a function-wide set of spellings:

- function names;
- parameter names;
- existing local-binding names;
- existing item names.

Every parameter remains in its original parameter position. Every local or
pattern binding remains declared exactly once in the same expansion group and
the same recursive structural role as in the target skeleton. This includes
bindings in destructuring and `let-else` patterns, `if let`, `while let`, `for`,
and the individual arms of a `match`. Two source bindings with the same
spelling in different scopes remain two distinct declarations in their
respective locations.

Represent each declaration role with a stable structural position:

- a parameter is anchored by its zero-based parameter index;
- a statement-local pattern is anchored by the expected label, the
  zero-based statement position within that expansion group's preserved
  structural list, and its role (`let` pattern, `let-else` pattern, `if let`
  pattern, `while let` pattern, `for` pattern, closure parameter, or match-arm
  pattern with zero-based arm index);
- nested anchors append the preserved branch, loop body, match-arm, plain
  block, or `let-else` body path; and
- within a pattern, append exact child segments for binding nodes and for
  tuple/tuple-struct indices, struct field names or indices, slice
  prefix/rest/suffix positions, `@` binder versus subpattern, `or`-pattern
  alternative indices, and reference/parenthesized subpatterns.

Constructor and path spellings, literals, ranges, and other nonbinding pattern
content may change only when the projected binding-position paths remain
unchanged. Adding or removing a wrapper or moving a binding between tuple
slots, struct fields, slice positions, `@` roles, `or` alternatives, reference
patterns, branches, or arms changes its declaration role and is rejected.
Bindings with the same spelling across alternatives of one valid `or` pattern
form one Rust declaration identity: validate the complete alternative-position
set as one identity rather than reporting the alternatives as duplicates.
Preserve by-value versus `ref` binding mode; ignore only `mut`, including the
`mut` in `ref mut`. After a binding is uniquely associated with its expected
structural position, a by-value/`ref` difference is
`existing_binding_mode_mismatch`, not a location mismatch. Its message names
the binding, label and pattern role, expected and observed binding modes, and
instructs the LLM to restore or remove `ref` while explaining that `mut` may
differ.

An existing declaration may not be deleted, duplicated, moved to another
label, moved to another branch or arm, or replaced by a generated temporary.
The only supported existing function-local items are `const` and `static`.
Each remains exactly once in its original expansion group and structural
position. Before item-structure comparison, recursively remove only
`#[proctor(N)]` attributes from both compared items because label syntax and
placement are validated separately. Then, ignoring spans, node IDs, token
caches, and formatting, require the complete parsed AST to equal the target
skeleton: item kind, name, visibility, every non-`proctor` attribute, declared
type, `static mut` status, and initializer are all exact. The sole comparison
normalization is the global binding-mutability exception: `mut` may differ on
binding patterns inside a const/static initializer, but `static mut` may not
differ because it is item mutability. No new nested item of any kind may be
introduced. Exact initializer structure overrides the general expansion
permission for labels nested inside that initializer: those nested statements
remain one-for-one and structurally exact apart from binding `mut`. The outer
item statement remains an ordinary expansion-group member, so additional
non-item siblings with its label are governed by the normal expansion and
generated-binding rules.

Associate an expected local item by its complete Rust identifier before
comparing location and structure. A unique same-named item with the wrong
`const`/`static` kind is `existing_item_structure_mismatch`, not a
missing-plus-unexpected pair. A renamed item has no matching identity and is
reported as the expected item missing plus the result-only item unexpected.

For every existing local declaration, preserve whether an explicit type is
present. If present, its result type must structurally equal the target
skeleton's type. This enforces the pointer-analysis decision materialized by
Phase 1. Binding mutability is the sole exception: the validator ignores
`mut`/non-`mut` differences for parameters and all local or pattern bindings.

References to functions, methods, fields, and operations may change when needed for translation. For example, a libc call may be replaced by a slice method.

### 13.2 New bindings

Every newly introduced local binding must have the form:

```text
proctor_temp_var_n
```

where `n` is a nonnegative integer.

This rule applies to every new binding pattern visible in the parsed result,
including simple and destructuring `let` bindings, `let-else`, `if let`,
`while let`, `for`, `match` arms, and closure parameters. Binding mutability is
unrestricted.

The Phase 1 input contract does not reserve this prefix. If an expected
function already contains a binding with this spelling, it is an existing
binding and retains its existing declaration identity. A newly introduced
binding may not reuse that spelling.

Crat rejects duplicate declarations or shadowing of the same generated
temporary name anywhere within one transformed function. Different generated
temporaries must have different numeric suffixes; suffixes need not be
contiguous or start at zero.

### 13.3 Temporary locality

A generated temporary belongs to the expansion group containing its declaration.

All references to that binding must occur:

- in sibling statements in the same expansion group; or
- in unlabeled nested code inside those sibling statements.

It may not be used by a statement carrying another label.

Valid:

```rust
#[proctor(1)]
let proctor_temp_var_0 = p.unwrap();

#[proctor(1)]
let x = *proctor_temp_var_0;

#[proctor(2)]
use_value(x);
```

Invalid:

```rust
#[proctor(1)]
let proctor_temp_var_0 = p.unwrap();

#[proctor(2)]
let x = *proctor_temp_var_0;
```

Crat validates binding identity and lexical scope rather than treating every
identifier token with the same spelling as a reference. Because duplicate or
shadowing declarations of a generated temporary are prohibited, each accepted
temporary declaration has one unambiguous identity.

Macro token trees are opaque to parser-only Rust AST name resolution. A
generated-temporary identifier may therefore not occur anywhere inside a
macro invocation. The validator reports a specific error instructing the LLM
to move that use into ordinary Rust syntax. Other macro invocations remain
allowed when they satisfy the surrounding structural rules.

This restriction keeps each labeled expansion syntactically local and suitable for later modular rule extraction.

### 13.4 Attributes

Apart from attributes already present on a supported function-local `const` or
`static` item, the target skeleton contains only statement-root
`#[proctor(N)]` attributes in function bodies. A result may not introduce any
other statement or expression attribute, and a `proctor` attribute may not
appear anywhere except the root attribute storage of a statement in an
expansion group. Existing non-`proctor` attributes on a local `const` or
`static` are item structure and must remain exact under Section 13.1; adding,
removing, or changing one produces `existing_item_structure_mismatch`, not
`unexpected_body_attribute`. Function-header attributes are ignored because
candidate generation discards the LLM header.

## 14. Crat validator

The in-memory validator receives a request with:

- `schema_version`, which must be the integer `1`;
- an ordered `expected_functions` array; and
- the selected LLM-generated Rust `transformation`.

Each expected-function entry contains:

- `id`: its Phase 1 item ID;
- `name`: its final Rust identifier; and
- `skeleton`: its complete annotated target skeleton.

The exact request schema is:

```json
{
  "schema_version": 1,
  "expected_functions": [
    {
      "id": 12,
      "name": "f",
      "skeleton": "pub unsafe fn f(mut p: &[i32]) -> i32 { ... }"
    }
  ],
  "transformation": "pub unsafe fn f(p: &[i32]) -> i32 { ... }"
}
```

`schema_version` and every `id` are JSON integers, not strings or floating
point values. `schema_version` must equal `1`; every `id` must be in the Rust
`u64` range. Reject unknown JSON fields as setup errors. IDs and names must
each be unique within the request. Each skeleton string must parse as exactly
one free function whose name matches its entry, and it must contain a valid
Phase 1 label/control tree with unique labels. A violation in request metadata
or an expected skeleton is a setup error, not an LLM validation failure.

Suitable public API shapes are:

```rust,ignore
pub fn validate(request: &ValidationRequest) -> ValidationResponse;
pub fn validate_json(input: &str) -> String;
```

The typed API performs no I/O. The JSON API parses an input string and returns
pretty JSON even when request deserialization fails; malformed request JSON is
represented by `status = "setup_error"`.

Every response uses response `schema_version: 1`, including responses to
malformed JSON and requests whose request `schema_version` is unsupported.

Crat parses the result as a crate and matches its top-level functions by final
name. Expected order is request order; result function order is irrelevant.

### 14.1 Setup checks

Before inspecting an LLM result, require:

- supported `schema_version`;
- no unknown request fields;
- unique expected IDs;
- unique expected names;
- nonempty expected function list;
- exactly one correctly named free function in each skeleton string;
- a supported target signature in each skeleton;
- well-formed, unique numeric statement labels;
- a valid Phase 1 control/statement tree;
- no function-local item declaration other than `const` or `static`; and
- no conflict between request metadata and parsed skeletons.

Report the first setup error in deterministic check order. Setup errors abort
result validation because any result diagnostics would be untrustworthy. A
prohibited function-local item in an expected skeleton is
`invalid_expected_skeleton`; the message identifies the function, label when
available, observed item kind, and the `const`/`static`-only restriction. This
is a Phase 2 setup check over Phase 1 output, not another Phase 1 generator
change. Scan item statements recursively at every supported statement-list
depth, including blocks inside supported local-item initializers.

### 14.2 Whole-snippet checks

Require that:

- the snippet parses;
- it contains exactly the expected function definitions;
- every expected function appears exactly once;
- no additional top-level items are introduced.

The exact-function-set check includes every top-level item kind: `use`,
modules, constants, types, macro definitions or invocations, and foreign items
are all unexpected. Duplicate or missing expected functions and extra
functions are whole-snippet failures. If parsing or exact-function-set checking
fails, return one global failure and suppress per-function checks.

### 14.3 Signature checks

For each generated function, validate:

- function name;
- parameter count;
- parameter names;
- parameter types;
- return type.

Compare parsed types structurally, not textually. Ignore source spans, node
IDs, token caches, formatting, and redundant parentheses. Otherwise require
the same type structure, including paths, path qualification, mutability
inside pointer/reference types, tuples, arrays and lengths, generic arguments,
and explicit lifetime names. Do not perform name resolution or treat aliases
as equivalent. An omitted return and an explicit `-> ()` are distinct
structures.

The prototype excludes:

- variadic signatures;
- parameter patterns;
- source type and const generics, generic bounds, and where clauses; Phase 1
  generated lifetime parameters remain supported;
- async functions.

Crat does not need to validate the generated function's:

- visibility;
- ABI;
- `unsafe` qualifier;
- `const` qualifier; or
- binding mutability in parameters.

The validator does not separately compare the function generic-parameter
declaration because candidate generation replaces the complete header.
Explicit lifetime names appearing inside parameter and return types are still
part of structural type comparison.

During replacement, Crat discards those header properties from the LLM result and emits its own exact target header.

### 14.4 Binding-type checks

After matching expansion groups and structural roles, compare every existing
local declaration against its target-skeleton declaration. The existing
binding identities and their placement must match Section 13.1. An explicit
target type must be present and structurally equal in the result; a target
declaration without an explicit type must remain without one. Ignore binding
mutability during every such comparison.

### 14.5 Label, control, binding, and attribute checks

Validate all rules in Sections 12 and 13, including:

- nonempty expansion groups;
- consecutive same-label siblings;
- no nested repetition of a label;
- no deleted or new labels;
- original label order;
- preserved control kind;
- preserved labeled descendants;
- only one control-root statement when expanding a labeled control statement;
- unlabeled newly introduced nested statements;
- exact preservation and placement of existing declarations;
- target local-variable types;
- temporary-variable naming and locality;
- prohibition of generated temporaries inside macro token trees;
- exact preservation of existing function-local `const` and `static` items;
- prohibition of every new function-local item;
- prohibition of new statement/expression attributes; and
- prohibition of explicit unsafe blocks.

An explicit unsafe block is a user-written `unsafe { ... }` block anywhere
inside a returned function body, including newly introduced nested code.
Report `explicit_unsafe_block` on that matched function's item-specific
failure, with its expected ID and name and the complete pretty-printed matched
function as `failed_snippet`. The message identifies the enclosing expansion
group label when association is available. Unsafe-block detection is
independent of label/control association and continues even when another body
rule fails.

Diagnostics must be repair-oriented. Each message identifies the function and
item ID when available, the relevant label and structural role when available,
what was expected, what was observed, and the concrete corrective constraint.
Do not emit cascades: if a missing or malformed parent group/control makes
descendant association unreliable, report the parent error and suppress
derivative descendant, declaration, and temporary-locality errors.

### 14.6 Validation response

Every successfully processed request produces one of three tagged response
objects. Serialize with pretty JSON equivalent to
`serde_json::to_string_pretty`, in the key orders shown below, with no trailing
newline.

Validation success:

```json
{
  "schema_version": 1,
  "status": "valid"
}
```

Validation failure:

```json
{
  "schema_version": 1,
  "status": "invalid",
  "failures": [
    {
      "id": 12,
      "name": "f",
      "failed_snippet": "unsafe fn f(...) { ... }",
      "errors": [
        {
          "code": "missing_label",
          "message": "Function `f` (item 12): label 3 is missing. Restore label 3 in its original structural position."
        }
      ]
    }
  ]
}
```

For an item-specific failure, `failed_snippet` is the complete pretty-printed
matched result function. For a global failure, `id` and `name` are null and
`failed_snippet` is the complete unchanged transformation input:

```json
{
  "schema_version": 1,
  "status": "invalid",
  "failures": [
    {
      "id": null,
      "name": null,
      "failed_snippet": "...",
      "errors": [
        {
          "code": "result_parse_error",
          "message": "The returned Rust transformation does not parse: ..."
        }
      ]
    }
  ]
}
```

Setup error:

```json
{
  "schema_version": 1,
  "status": "setup_error",
  "error": {
    "code": "duplicate_expected_name",
    "message": "Expected function name `f` appears twice. Function names must be unique within one validation request."
  }
}
```

`errors` is nonempty, and each error contains only `code` and `message` in that
order. `failures` is nonempty and is ordered by the request's expected-function
order after any global failure. Within one function, emit errors in a fixed
validation-pass order and then in target-skeleton structural preorder.
Repeated validation of the same bytes must produce byte-identical JSON.

The validator uses stable machine-readable codes and detailed human-readable
messages. The exhaustive initial code set and precedence expectations are
fixed by `phase-2-test-plan.md`; later regressions may add codes but may not
repurpose an existing code.

### 14.7 Exit behavior

The `validate` process exits zero after writing any `valid`, `invalid`, or
`setup_error` response. The orchestrator proceeds only for `valid`, requests an
LLM repair only for `invalid`, and aborts immediately for `setup_error`.

The process exits nonzero only when it cannot produce and write a trustworthy
response, including:

- invalid CLI invocation;
- failure to read the request path;
- failure to write the response path;
- response-serialization failure; or
- panic or other unexpected internal failure.

CLI usage and filesystem failures may be diagnosed on standard error because a
response file cannot be guaranteed in those cases.

### 14.8 Error aggregation and precedence

Use these stages:

1. Deserialize and setup-check the request. Return the first `setup_error` on
   failure.
2. Parse the result. Return one global failure on failure.
3. Check the exact top-level function set. Return one global failure containing
   all deterministically ordered set errors on failure.
4. Validate each matched function independently, including explicit unsafe
   blocks. Accumulate all reliable item-specific errors.

Order exact-function-set errors as follows: missing expected functions in
request order, duplicate returned functions in result source order, then
unexpected returned functions or other items in result source order.

Within a function, signature errors do not prevent body checks when the body
can still be associated safely. A broken label/control parent suppresses only
checks that depend on that association. Independent siblings and independent
rules continue to be checked so one repair response can address multiple
reliable problems.

For item-specific reporting, use this stable category order without changing
the analysis dependencies needed to establish reliable associations:

1. signature;
2. existing declaration identity and target local type;
3. labels and expansion groups;
4. controls and descendant placement;
5. generated temporaries;
6. explicit unsafe blocks and body attributes.

Within one category, use target-skeleton structural preorder, then result
source order for result-only constructs.

Use the following same-category precedence and suppression rules:

| Area | Precedence and suppression |
| --- | --- |
| Signature | Parameter count; parameter names in parameter order; parameter types in parameter order; return type. A count mismatch suppresses name/type checks for parameter positions that cannot be paired, but not checks for safely paired positions or the return type. |
| Existing declarations | For each target declaration in structural preorder, first associate by declaration identity and structural position. One occurrence in a wrong position produces only the applicable location-mismatch error. No occurrence produces `missing_*`; multiple occurrences produce `duplicate_*`. For a uniquely associated binding, compare by-value/`ref` mode before explicit-type presence and type. Compare the complete structure of a uniquely associated local `const` or `static` after location association. After expected declarations, report result-only bindings/items in result source order. |
| Label syntax and placement | Report malformed or misplaced `proctor` attributes in result source order before sequence diagnostics. A malformed attribute occupying an expected group position suppresses a derivative `missing_label` there. A misplaced attribute does not create a statement label. |
| Label sequence | After well-formed root labels are collected, report nested repetition, then nonconsecutive reappearance, then order mismatch, then missing expected labels in target preorder, unexpected labels in result source order, and unlabeled sibling statements in result source order. Do not report `label_order_mismatch` for a sequence already explained by `nonconsecutive_label`. |
| Controls and descendants | Validate the parent control kind, statement role, branch/arm shape, and required control-root count before descendant placement. A parent failure suppresses descendant label, declaration, type, and temporary-locality errors only beneath the unreliable role. With a valid parent, a label found in the wrong branch, arm, or structural statement list produces `descendant_location_mismatch`, not a missing-plus-unexpected-label pair. |
| Generated temporaries | Report invalid generated binding names, duplicate declarations, unresolved generated-temporary references, cross-group references, and macro-token occurrences in that order, each in result source order. Suppress a locality error when the declaration or enclosing expansion group cannot be associated reliably. |
| Body safety and attributes | Report explicit unsafe blocks in result source order, followed by unexpected statement/expression attributes in result source order. These AST-local checks do not depend on successful label/control association. |

For a required control group, zero control roots is
`missing_control_root` and more than one is `multiple_control_roots`. A value
path whose complete local identifier matches `proctor_temp_var_n` but resolves
to neither an expected existing local nor a declared generated temporary is
`unresolved_generated_temporary`. Existing nested items use
`duplicate_existing_item` when repeated and
`existing_item_location_mismatch` when a unique occurrence is in the wrong
expansion group or structural role. These association errors suppress
derivative missing/unexpected errors for the same identity. A uniquely
associated local `const` or `static` whose parsed structure differs uses
`existing_item_structure_mismatch`.

A full path is not required in validation output. The orchestrator can obtain
it from the skeleton JSON.

## 15. SCC transaction policy

Each SCC is all-or-nothing.

No function in the SCC is committed until:

1. every function passes Crat structural validation; and
2. the complete candidate project passes `cargo build`.

If either step fails:

- discard the entire candidate;
- discard partial success within the SCC;
- regenerate every function in the SCC.

## 16. Candidate-project generation

### 16.1 One-time target-safety normalization

Phase 3 provides a Rust project-rewrite operation that Phase 4 invokes once,
after skeleton generation and before processing the first SCC. It receives the
original project, a fresh destination, and the full paths of all `Fn` records
in the immutable skeleton JSON. It copies the project and mechanically changes
every listed function header to `unsafe fn` in one global operation.

This normalization:

- occurs after skeleton generation so `annotated_source` and
  `source_signature` still record original source safety;
- changes only the safety qualifier of transformation targets;
- leaves every excluded `main`, foreign declaration, context item, function
  body, type, ABI, visibility, attribute, and generated Cargo bin source
  unchanged;
- creates no wrapper merely for a safe-to-unsafe normalization; and
- produces the initial current project from which SCC candidates are copied.

Normalizing every target before the first callee-first SCC replacement is
required for incremental compilation. Otherwise, replacing a safe callee with
an unsafe target could make an untransformed safe caller ill-formed. Once all
targets are unsafe, calls among still-untransformed and transformed targets may
occur directly inside unsafe functions. Phase 4 runs `cargo build` on this
normalized initial current project before beginning SCC processing.

### 16.2 Per-SCC candidate insertion

After structural validation succeeds, the orchestrator calls Crat with:

- current Rust project directory path;
- fresh candidate-project destination path;
- JSON mapping from full Rust function paths to transformed snippets.

Example:

```json
{
  "foo::bar": "unsafe fn bar(...) { ... }",
  "foo::baz": "unsafe fn baz(...) { ... }"
}
```

Crat then:

1. Copies the current project into the candidate destination.
2. Parses each transformed function.
3. Removes all `#[proctor(...)]` labels.
4. Replaces each original SCC function.
5. Uses Crat's exact unsafe target signature rather than trusting the LLM
   header.
6. Creates wrappers when source and target parameter or return types differ,
   except for the forced two-argument `main_0` conversion.
7. Rewrites calls from external untransformed callers.
8. Applies the special executable-`main` migration below when the SCC contains
   the supported two-argument `main_0`.
9. Writes the complete candidate project.

The current project is never modified in place.

### 16.3 Executable `main_0` migration

The excluded safe library `main` is not an LLM transformation target. When an
SCC containing the supported zero-argument `main_0` is inserted, leave
`main` unchanged: it already calls an unsafe `main_0` inside its existing
unsafe block.

When an SCC containing the supported two-argument `main_0` is inserted,
replace the excluded `main` mechanically in the same candidate transaction.
Do not generate a compatibility wrapper for the forced `main_0` `argv`
conversion: the supported `main` is migrated directly, and another
untransformed caller of this `main_0` form is unsupported.
Use this fixed function:

```rust
pub fn main() {
    let mut command_line_arg_storage: Vec<Vec<i8>> = ::std::env::args()
        .map(|arg| {
            ::std::ffi::CString::new(arg)
                .expect("Failed to convert argument into CString.")
                .into_bytes_with_nul()
                .into_iter()
                .map(|byte| byte as i8)
                .collect()
        })
        .collect();

    let argc = command_line_arg_storage.len() as core::ffi::c_int;
    let mut command_line_arg_slices: Vec<&mut [i8]> = command_line_arg_storage
        .iter_mut()
        .map(|arg| arg.as_mut_slice())
        .collect();

    let mut argv_terminator: [i8; 0] = [];
    command_line_arg_slices.push(&mut argv_terminator);

    unsafe {
        ::std::process::exit(
            main_0(argc, command_line_arg_slices.as_mut_slice()) as i32,
        )
    }
}
```

The owned inner vectors retain each argument's trailing NUL. The final empty
slice is the prototype representation of the C `argv[argc] == NULL` sentinel;
`argc` excludes that sentinel. This convention is intentionally fixed and may
not preserve programs that require other `argv` behavior; those programs are
outside the supported input model.

The generated Cargo bin shim remains unchanged because it continues to call
the safe library `main`. The hard-coded `main` body is trusted
project-integration code: its explicit unsafe block, local names, and structure
are not subject to Phase 2 validation or LLM temporary-name rules. The
supported executable model assumes the excluded `main` is the only
untransformed caller that must be migrated for this forced `main_0` signature.

## 17. Wrapper generation

### 17.1 Wrapper creation

Create a wrapper only when a function's source and target parameter or return
types differ. Source-safe versus target-unsafe normalization alone never
creates a wrapper. The forced two-argument `main_0` conversion is also exempt:
its only supported caller, the excluded `main`, is replaced directly as
specified in Section 16.3.

The transformed implementation:

- remains at the original Rust path;
- uses the unsafe target signature.

The wrapper:

- uses the source parameter and return types and any required source ABI/export
  surface, but is itself unsafe;
- delegates to the transformed implementation;
- is an intentionally unsafe compatibility boundary.

Every generated wrapper and transformed implementation is `unsafe fn`,
regardless of source safety or the LLM-returned qualifier. They may be made
`pub` as needed to avoid privacy problems. The excluded executable `main` is
the deliberate safe exception.

### 17.2 Wrapper module

For:

```text
foo::bar
```

generate the wrapper at:

```text
crate::proctor_translation_wrapper::foo::bar
```

The wrapper module mirrors the original module hierarchy.

For this prototype, assume that `proctor_translation_wrapper` does not conflict with the source crate.

Do not traverse the generated wrapper module during later call-site rewriting.

When adding wrappers for a new SCC, preserve wrappers created for earlier SCCs.

### 17.3 Export handling

For a signature-changing externally exported function:

- remove `#[no_mangle]` from the transformed implementation;
- give the wrapper the source signature;
- preserve the source ABI on the wrapper;
- add `#[no_mangle]` to the wrapper.

If a function's signature does not change, preserve any required export attributes on the original function.

The prototype considers only executable and `cdylib` targets.

### 17.4 Conversion logic

Follow Crat's existing wrapper-conversion logic as much as possible.

The prototype may use its current unsafe conversions for:

- references;
- optional references;
- slices;
- optional slices;
- boxes;
- optional boxes;
- boxed slices;
- optional boxed slices.

This includes the existing provisional slice-bound strategy.

These wrappers are intentionally unsafe and assume valid caller behavior.

Global-variable wrappers are not needed because global types are unchanged.

## 18. Call-site rewriting

When committing an SCC, Crat rewrites calls to each signature-changing SCC function.

Rewrite only callers outside the current SCC.

Use an absolute wrapper path:

```rust
crate::proctor_translation_wrapper::foo::bar
```

Crat must:

- leave calls between functions in the same SCC unchanged;
- not traverse the generated wrapper module;
- leave calls to signature-unchanged functions unchanged;
- rewrite all statically resolved calls from external untransformed callers.

Because SCCs are processed callee-first:

- all callees outside the current SCC have already been transformed;
- external callers of the current SCC are still untransformed;
- those external callers temporarily call wrappers;
- when an external caller is later transformed, its whole function is replaced;
- its LLM-generated body directly uses target signatures.

No separate transformed-function registry is required.

The immutable skeleton JSON may contain stale source snippets after earlier call-site rewriting. This is acceptable because each function is later replaced as a whole using its original annotated source and target skeleton.

## 19. Compilation and promotion

After Crat creates a candidate project, run:

```bash
cargo build
```

in that candidate directory.

### Success

- Promote the candidate to become the current project.
- Mark the SCC as processed.
- Select the next leaf SCC.

### Failure

- Capture compiler standard output and standard error.
- Discard the candidate project.
- Keep the current project unchanged.
- Start a repair attempt for the entire SCC.

The prototype assumes Crat's integration routines are correct. Compiler diagnostics are given to the LLM even when they refer outside the SCC. The LLM may change only SCC functions. If that is insufficient, the retry limit eventually aborts orchestration.

## 20. Repair policy

The initial LLM generation does not count as a repair attempt.

After the initial failure, allow at most ten additional LLM calls for the SCC.

The ten-attempt limit combines:

- structural-validation repairs;
- compilation repairs.

Every additional LLM call consumes one repair attempt, regardless of its eventual failure category.

Each repair call:

- starts a fresh LLM session;
- receives the complete original prompt;
- receives the latest failed transformation;
- receives the latest structural or compiler diagnostics;
- regenerates every function in the SCC.

Each repair response repeats the full pipeline:

1. Extract the selected Rust code block.
2. Run Crat structural validation.
3. If valid, create a fresh candidate project.
4. Run `cargo build`.
5. Promote on success or retry on failure.

If the SCC has not succeeded after ten repair calls, abort the complete orchestration immediately.

## 21. Completion

The orchestrator succeeds when every SCC has been translated and the final promoted project builds.

For this prototype:

- all wrappers remain in the final project;
- no test suite is executed;
- no reusable rules are extracted;
- no rule-set file is read or written;
- `proctor.toml` is not required;
- wrappers from non-local transformations are ignored;
- all supported free functions except the mechanically managed executable
  `main` are transformed.

This milestone demonstrates:

- whole-program skeleton generation;
- constrained LLM function rewriting;
- dependency-aware SCC scheduling;
- structural validation;
- wrapper-based incremental migration;
- compiler-guided repair.

It does not yet establish semantic correctness.

## 22. Implementation sequence

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

### Phase 2: Structural validator

Implement and unit-test:

- the four Phase 1 generator adjustments from Section 5.2: recursively mark
  every target binding mutable; make every target function header unsafe while
  preserving source safety; omit every free function named `main`; and force
  the supported two-argument `main_0` target `argv` type to
  `&mut [&mut [i8]]`;
- all affected existing Phase 1 Rust test oracles and the additional
  regressions listed in `phase-2-test-plan.md`, without editing the historical
  Phase 1 plan;
- moving the existing Phase 1 implementation and tests into a `skeleton`
  module, adding a separate `validator` module, and keeping `lib.rs` limited to
  required crate-level attributes/`extern crate` declarations, module
  declarations, and public re-exports, with no implementation or large tests;
- the typed in-memory validation API and the pure JSON-string API;
- the versioned request schema with per-function IDs, names, and target
  skeletons;
- setup validation and the `setup_error` response;
- the Phase 2 expected-skeleton setup check that permits only function-local
  `const`/`static` items, without adding another Phase 1 generator change;
- parsing result snippets;
- function matching by name;
- exact expected-function set checks;
- structural target-signature and existing-local-type checks, ignoring all
  binding mutability;
- label expansion groups;
- label syntax, placement, order, grouping, and nesting checks;
- recursive control-kind, branch/arm, plain-block, and `let-else`
  preservation consistent with Phase 1;
- control-statement expansion rules;
- strict declaration-identity, by-value-versus-`ref` binding-mode, and
  placement preservation;
- exact structural preservation of existing function-local `const` and
  `static` items and rejection of every new function-local item;
- generated-temp naming;
- generated-temp locality;
- generated-temp macro-token rejection;
- new statement/expression-attribute rejection;
- item-specific explicit unsafe-block rejection;
- deterministic, cascade-suppressed, repair-oriented diagnostics with stable
  codes;
- the `valid`, `invalid`, and `setup_error` JSON response schemas;
- deterministic response serialization; and
- the thin `crat-tool validate --input ... --output ...` command with the exit
  behavior in Section 14.7.

Implement every case in `phase-2-test-plan.md`. All functional tests use
in-memory APIs and perform no filesystem writes or subprocess invocation. The
CLI wiring is not tested in Phase 2. The new source-safety and executable
decisions add generator work and generator regressions during Phase 2, but no
additional validator rule: safety was already an ignored LLM-header property,
`main` never enters a validation request, and the forced `main_0` type is
validated through the ordinary structural parameter-type rule.

### Phase 3: Candidate-project generation

Implement and test:

- the one-time whole-project safety normalization of every transformation
  target after skeleton generation;
- copying into a fresh destination;
- function replacement by full path;
- unconditional unsafe-target-header enforcement;
- label removal;
- fixed mechanical replacement of the excluded executable `main` when
  committing the supported two-argument `main_0`, while leaving the
  zero-argument case and generated bin shim unchanged;
- wrapper generation;
- no wrapper for safety-only source/target differences;
- no wrapper for the forced two-argument `main_0` conversion;
- wrapper-module maintenance;
- export handling;
- external call-site rewriting;
- avoidance of wrapper-module traversal.

Test manually on:

- safe transformation targets that call one another across SCCs;
- a simple caller-callee chain;
- direct recursion;
- a mutually recursive SCC;
- nested modules;
- both supported executable `main_0`/`main` forms; and
- an exported `cdylib` function.

### Phase 4: Python orchestration

Implement:

- Crat process invocation;
- skeleton JSON loading;
- one-time invocation of Phase 3 target-safety normalization and compilation of
  the normalized initial current project;
- SCC-local function-name uniqueness checks;
- function graph construction;
- SCC computation;
- deterministic leaf scheduling;
- dependency-context rendering;
- breadth-first type closure;
- 100,000-character dependency budget;
- prompt construction;
- LiteLLM client;
- code-block extraction;
- validation invocation;
- candidate generation;
- `cargo build`;
- repair accounting;
- project promotion and cleanup.

Keep LiteLLM behind a replaceable client abstraction.

### Phase 5: End-to-end evaluation

Begin with programs having:

- no pointer-containing named types;
- no function pointers;
- acyclic call graphs;
- unique function names within every SCC;
- simple reference and slice transformations.

Then add:

- direct recursion;
- mutually recursive functions;
- optional references;
- boxes and boxed slices;
- exported library functions;
- cases that require structural repair;
- cases that require compiler-guided repair.

Record at least:

- number of functions and SCCs;
- initial-generation success rate;
- structural-repair count;
- compilation-repair count;
- final build success;
- failure reason when orchestration aborts.

## 23. Deferred work

The following are intentionally outside this prototype:

- test execution and semantic validation;
- reusable-rule extraction;
- rule application;
- preservation labels;
- improved pointer analysis;
- pointer-containing named-type transformation;
- global-variable transformation;
- global-variable wrappers;
- custom-type wrapper generation;
- function pointers and callbacks;
- methods, traits, generics, and closures;
- integration with non-local transformation wrappers;
- `proctor.toml` integration;
- replacement of LiteLLM with the team's shared framework.
