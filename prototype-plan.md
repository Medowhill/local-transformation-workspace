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
Phase 1. `phase-2-test-plan.md`, together with the Phase 2 lifetime-generic
amendment in Section 14.3 and the corresponding update cases in
`phase-3-test-plan.md`, is the exhaustive executable contract for Phase 2.
`phase-3-test-plan.md` is the exhaustive executable contract for Phase 3.

Phase 1 was implemented and validated before the Phase 2 design was finalized.
Phase 2 includes four intentional adjustments to the completed Phase 1
generator, all specified in Section 5.2: mark every target binding mutable;
make every target function header unsafe while preserving source safety;
exclude every free function whose final identifier is `main`; and force the
supported two-argument `main_0` target `argv` type to
`&mut [&mut [i8]]`. The implementation agent for Phase 2 must make these
generator changes and update the existing Phase 1 Rust tests as specified in
`phase-2-test-plan.md`. During Phase 2, do not edit or reinterpret the
historical `phase-1-test-plan.md`, and do not otherwise reopen Phase 1
behavior. Moving the existing code and tests into the `skeleton` module is an
organizational refactor, not another Phase 1 semantic change. The explicit
Phase 3 amendments below supersede that completed behavior without rewriting
the historical plans.

Phase 2 was implemented and validated before the Phase 3 design was finalized.
Phase 3 adds three intentional amendments to the completed Phase 1/2
implementation:

- validate the complete function lifetime-generic declaration rather than
  validating lifetime names only where they occur inside parameter and return
  types;
- reject every function-local parsed item statement, including the local
  `const` and `static` items that Phase 1 and Phase 2 previously handled; and
- recognize the special zero- and two-argument `main_0` forms using only the
  final identifier symbol and arity, without inspecting types or bodies.

Sections 5.2, 13.1, 13.4, 14.1, 14.5, and 14.8 are normative for the local-item
amendment; Section 14.3 is normative for the lifetime amendment; and Sections
5.2 and 16.4 are normative for arity-only executable recognition. Do not edit
the historical `phase-1-test-plan.md` or `phase-2-test-plan.md`; the required
updates to existing Phase 1 and Phase 2 Rust tests and the additional
regressions are specified in Section 4 of `phase-3-test-plan.md`.

`unsupported.md` is the consolidated conceptual input contract for this
prototype. Some low-level skeleton and validator behavior deliberately handles
constructs listed there (notably `ref` bindings), but that robustness does not
make those constructs supported local-transformation inputs. Phase 2 adds no
supportedness checker or normalization pass.

Phase 4 is implemented after the PROCTOR orchestration framework became
available. The Phase 4 implementation therefore uses PROCTOR's stage
envelopes, shared LLM client, prompt library, usage tracker, and reporting
types directly. It does not implement the historical LiteLLM wrapper proposed
below. `phase-4-test-plan.md` is the exhaustive executable contract for Phase
4 and is normative where it supplies Python-level integration detail.

Phase 4 also changes the assumed pipeline boundary. The Rust project received
by local transformation may have passed through Crat's `split` and `bin`
passes and may subsequently have been changed by a non-local transformation.
Local transformation must therefore always run `expand` followed immediately
by `unexpand` as its first source-preparation operation. `split` and `bin` are
not rerun. The generated Cargo binary target is a separate crate and remains
outside local transformation.

Phase 4 makes two intentional amendments inside the existing Crat
implementation: the `expand` cleanup boundary preserves explicit bin targets,
and skeleton presentation normalizes every non-`ref` binding to `mut` in both
source and target renderings while preserving `ref` versus `ref mut` exactly.
These amendments and the required updates to existing in-memory Crat tests are
part of Phase 4 and are specified in this document and
`phase-4-test-plan.md`. Do not edit any earlier phase test plan.

The prototype will validate the following loop:

1. Copy the read-only input Rust project to a stage-private current project.
2. Run Crat `expand` and `unexpand` on the current project's library crate.
3. Run Crat's existing whole-program pointer analysis and generate target
   function skeletons without rewriting function bodies.
4. Normalize target safety once and build the normalized current project.
5. Translate functions SCC-by-SCC with an LLM.
6. Validate each LLM result structurally with Crat.
7. Generate wrappers and redirect untransformed callers through Crat's
   replacement operation.
8. Temporarily install the candidate library source and run `cargo build`.
9. Keep the source on success or restore the previous source on failure.
10. Repair failures with fresh LLM calls.
11. Continue until all function SCCs have been translated.
12. Copy the final current project to the declared output without `target/`.

The prototype does **not** yet run tests or extract reusable rules. It neither
reads nor writes a rule-set artifact. It preserves `proctor.toml` byte-for-byte
when present but does not interpret its wrapper metadata. A nonempty
`wrappers` list is ignored rather than used to include, exclude, or specially
treat any source function.

## 2. Scope and assumptions

### 2.1 Supported program model

The input is one compilable Cargo project produced by the Translation
component and possibly changed by a non-local transformation. The library
crate may be split across external module files when it enters the stage.
Phase 4 copies the complete input project to its private work directory and
always runs Crat `expand` followed immediately by `unexpand` there. After this
preparation, the transformed library-crate source is exactly one physical Rust
file. That file may contain inline modules, which are traversed recursively,
but it does not contain external `mod foo;` declarations. A supported
executable project additionally contains the generated forwarding bin source
described below; that separate crate is not transformation input.

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
- a source function carrying both `#[no_mangle]` and
  `#[export_name = "..."]`;
- by-reference binding modes (`ref` and `ref mut`) in source patterns, even
  though skeleton generation and structural validation handle them
  mechanically;
- function pointers;
- dynamic dispatch;
- callback patterns through `void *`;
- macro definitions and item-producing macro invocations;
- a call expression written inside the input token tree of an expression- or
  statement-position macro invocation;
- function-body item declarations of every kind, including `const` and
  `static`;
- `cfg` attributes;
- attributes on statements;
- explicit `unsafe` blocks; and
- empty statements.

The function-body item restriction applies recursively to every parsed item
statement (`StmtKind::Item`), including local `const`, `static`, function,
type, struct, enum, union, module, trait, impl, foreign, `use`, `extern crate`,
macro-definition, and item-macro declarations. Phase 1 rejects the enclosing
transformation target as soon as recursive statement labeling encounters one;
it does not label the item or traverse its initializer or body. Expression-
or statement-position macro invocations are not item declarations and remain
governed by the existing macro and skeleton rules.

C2Rust expands C macros before this stage. `unexpand` may restore surface
syntax such as derive attributes and expression-position macro invocations.
Skeleton generation preserves that surface syntax in `annotated_source`; it
must not expand the source AST again.

Expression- and statement-position macro invocations remain supported only
when their source token trees contain no call expression. For example,
`println!("hello")` is supported, while `println!("{}", local_fn())` is not.
Calls introduced internally by expansion of a macro such as `println!` do not
violate this source-syntax restriction. Skeleton dependency collection still
walks expanded HIR and may therefore observe a source call nested inside a
macro invocation. Phase 3 rewrites the unexpanded surface AST and cannot
reliably redirect such a call to a temporary wrapper, which is why that source
form is excluded.

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
library source. Only a root-level library source is supported: after lexical
normalization of optional `.` components, `[lib].path` must identify one file
directly beneath the crate root, such as `lib.rs` or `./lib.rs`. Reject every
`..` component even if it would lexically return to the crate root, because
Crat's Expand cleanup may remove the intermediate directory before writing the
library file. Nested, absolute, and crate-escaping library paths are therefore
unsupported. The additional Cargo bin source is a generated forwarding shim
that only calls the library's safe `main`; it is outside skeleton generation
and is never rewritten by this prototype. Its source path is explicitly
declared by a Cargo `[[bin]]` table. Crat `expand` must preserve every such
explicit bin-target source while removing the library crate's obsolete split
`.rs` files. The supported library source has
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
validation, safety normalization, and item replacement. It may depend on
`pointer_replacer`, `utils`, and the rustc-private crates it needs.
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
`validator/tests.rs`. Phase 3 adds an `item_replacer` module with tests in
`item_replacer/tests.rs`; the module owns both target-safety normalization and
validated item replacement.

Add the executable `src/bin/crat-tool.rs` to the root `crat` package. Follow the
small command-dispatch structure used by `src/bin/crat-finder.rs`: parse
arguments, call the `tools` library, and perform only the requested outer
filesystem I/O. `make-skeleton` locates the crate's library source with
`utils::find_lib_path` and runs the compiler on that source. `validate` only
reads request JSON, calls the parser-only validator, and writes response JSON;
it does not locate or compile a project. Rust parsing and analysis belong in
the library, not in the executable.

The prototype requires four logical Crat operations, exposed through
`crat-tool` subcommands. Phase 1 implements `make-skeleton`, Phase 2 adds
`validate`, and Phase 3 adds `normalize-safety` and `replace`.

Phase 4 also uses the ordinary `crat` binary for its initial
`expand,unexpand` preparation. This is not a fifth `crat-tool` operation.
Before Phase 4, make the ordinary `Expand` pass crate-aware at its filesystem
cleanup boundary:

- parse every explicit `[[bin]].path` in the input `Cargo.toml` once before
  recursively removing Rust files;
- resolve each path relative to the manifest directory and lexically normalize
  `.` and `..`;
- reject an absolute path or a normalized path that escapes the crate root;
- do not recursively follow symlinked directories during cleanup; preserve an
  explicitly named in-crate bin path itself, including when that path names a
  symlink;
- treat multiple manifest spellings that normalize to the same in-crate path
  as one preserved source;
- preserve every resolved bin-target source byte-for-byte;
- retain the existing preservation of the root `build.rs` and `target/`;
- remove every other obsolete `.rs` file before writing the expanded library
  source; and
- do not modify `Cargo.toml`.

This deliberately covers Crat's supported, self-contained forwarding-bin
layout. Phase 4 does not add support for Cargo auto-discovered binaries,
examples, benches, tests, custom build-script paths, or binary crates with
their own module trees.

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

### 4.3 Normalize target safety

Expose the production operation as:

```text
crat-tool normalize-safety \
    --output <output.rs> \
    <input.rs>
```

The command reads one Rust source file and writes one Rust source file. It
does not read skeleton JSON, copy a Cargo project, or modify any other file.
Creation of the stage-private project and installation of the returned source
belong to Phase 4.

The underlying library operation is parser-only and in-memory:

```rust,ignore
pub fn normalize_target_safety(
    source: &str,
) -> Result<String, ReplacementError>;
```

It accepts and returns the exact single-file library source text and performs
no filesystem access. Section 16.1 specifies its complete behavior.

### 4.4 Replace validated items

Expose the production operation as:

```text
crat-tool replace \
    --request <request.json> \
    --output <output.rs> \
    <current-project>
```

The versioned request contains an ordered list of replacement item identities
and the one complete, already validated Rust transformation snippet returned
for the current SCC. It deliberately does not require Python to split or parse
that Rust snippet:

```json
{
  "schema_version": 1,
  "items": [
    {
      "id": 12,
      "path": "foo::bar",
      "name": "bar"
    }
  ],
  "transformation": "unsafe fn bar(...) { ... }"
}
```

Phase 3 supports only function entries, but the request and module use item
terminology because later phases may replace other item kinds. The exact
schema and setup rules are in Section 16.2.

The production command locates and compiles the current project's one library
source, calls the in-memory Rust-aware replacement operation, and writes only
the returned library source to the requested `.rs` output path. It does not
copy or mutate the current project. Phase 4 temporarily swaps this candidate
source into the stage-private current project, builds it, and either keeps the
candidate or restores the prior source. The generated Cargo bin shim and every
other project file therefore remain stage-owned and unchanged by `replace`.

A suitable library API is:

```rust,ignore
pub fn replace_items(
    source: &str,
    request: &ReplacementRequest,
    tcx: TyCtxt<'_>,
) -> Result<String, ReplacementError>;
```

The library receives no project path, destination path, reader, or writer.
The Python orchestrator must not parse or rewrite Rust. All Rust-specific
matching, label removal, header composition, wrapper generation, call
resolution, and source rewriting remain in Crat.

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
- uses the shared presentation binding normalization described below;
- preserves the source statement and control structure;
- uses parseable placeholders such as `todo!()` for unimplemented expressions;
- does not apply transformation-time pointer demotion.

Before cloning the annotated source into the target skeleton, apply one shared
presentation-only binding normalization recursively to every function
parameter and binding pattern: force every by-value identifier binding to
`mut`, preserve `ref` and `ref mut` exactly as written, and leave wildcards
unchanged. This covers simple and destructuring `let`, `let-else`, `if let`,
`while let`, `for`, and match-arm patterns. Consequently,
`annotated_source`, `source_signature`, `annotated_skeleton`, and
`target_signature` all use the same normalized non-`ref` mutability. Applying
the normalization before the source-to-skeleton clone is sufficient because
signature targeting and skeletonization do not introduce binding identifiers.
The input project and all analyses continue to use the unchanged source AST.

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
identifier's symbol is `main_0` and whose arity is two. Recognition checks
only this name and arity. It does not inspect source safety, parameter names or
types, return type, visibility, ABI, attributes, or body. The supported-input
contract guarantees the following source form:

```rust,ignore
unsafe fn main_0(
    argc: core::ffi::c_int,
    argv: *mut *mut core::ffi::c_char,
) -> core::ffi::c_int
```

Preserve the source parameter types and all other source-signature structure,
apart from the shared presentation mutability normalization. Force the target
parameter types to:

```rust,ignore
unsafe fn main_0(
    mut argc: core::ffi::c_int,
    mut argv: &mut [&mut [i8]],
) -> core::ffi::c_int
```

The `argv` override takes precedence over the ordinary pointer-analysis
decision, including a raw-pointer decision. It does not change the first
parameter's type, the return type, or pointer-analysis behavior for any other
function or binding. A function named `main_0` with arity zero uses ordinary
target-type decisions. Any other `main_0` arity is outside the supported
executable model and receives no special override.

This presentation normalization is intentional: neither displayed function
should prevent an LLM from assigning to an existing by-value binding while
translating it. Ordinary binding `mut` is not a semantic target decision.
Preserving `ref` versus `ref mut` avoids changing a binding's borrow type. The
Phase 2 validator continues to ignore `mut` everywhere and accepts either its
presence or absence in the returned transformation. It still enforces
by-value-versus-`ref` mode, binding identity, declaration placement, and
target types.

Before attaching a label to a statement, recursively reject every
`StmtKind::Item` with `GenerationErrorKind::FunctionLocalItem`; the error
identifies the enclosing function path and observed item kind. This includes
local `const` and `static` declarations and items nested at any supported
statement-list depth. Do not label the rejected item or visit its initializer
or body. The excluded `main` remains uninspected because it is omitted before
function-body labeling.

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
Apply the shared non-`ref` binding-mutability normalization above to both
source and target function renderings. Sanitization and binding normalization
are presentation-only: pointer analysis and the input project always use the
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

Implement Phase 4 as the standalone PROCTOR stage
`stages/local-transformation/`, with stage ID `local_transformation`. Follow
the current typed example LLM stage rather than introducing another
orchestration framework. Phase 4 adds this standalone stage and its manifest,
but does not add, remove, reorder, enable, or disable any entry in
`configs/full_pipeline.toml`. Its `stage.toml` contract is:

```toml
id = "local_transformation"
version = "0.1.0"
description = "Transform pointer-local Rust code SCC-by-SCC with validated LLM output."

exec = ["python3", "main.py"]
warmup = ["python3", "main.py", "--build-only"]

[requires]
rust_project = "required"

[produces]
rust_project = true

[config]
crat_dir = { type = "string", default = "../crat", doc = "crat checkout, relative to this stage" }
```

As with `example-llm-stage`, give the stage its own `pyproject.toml` with a
`proctor` dependency sourced from the repository at `../..`, and check in its
`uv.lock`. This lets the orchestrator invoke the standalone stage through its
isolated `uv` environment while the stage directly uses the shared framework.

The dependency-context limit of 100,000 characters and the maximum of ten
repair calls are fixed prototype semantics, not configuration options. Model,
provider, retry, rate-limit, pricing, and provider-specific settings come only
from `stage_input.framework.llm`.

Validate the stage-specific `config` table before other side effects. It may
contain only `crat_dir`. When omitted, use the manifest default `../crat`,
resolved relative to the stage directory in the same manner as the existing
Crat adapter. When supplied, `crat_dir` must be a nonempty string; reject
unknown keys and wrong types with a failure envelope. Report the one effective
resolved setting in `config_used`.

Use `StageInput` and `StageOutput` from `proctor.contracts`. Require a
read-only input `rust_project`, a declared `outputs.rust_project` path that
does not yet exist, and `framework.workdir`. Produce only `rust_project`;
report `rule_set = null`.
Use `framework.workdir` for the current project and all request, response,
candidate, and rollback files. Put a command/diagnostic log under
`outputs.artifacts_dir` when supplied and report its path relative to that
directory in `StageOutput.logs`. When `outputs.artifacts_dir` is absent, keep
the diagnostic log under the work directory for debugging but leave
`StageOutput.logs` empty: the stage contract permits reported logs only
relative to the declared artifacts directory.

The stage is responsible for:

- building both `crat` and `crat-tool` once per Crat commit, using the pinned
  Crat toolchain and the same sysroot/library environment discipline as the
  current `crat-adapter`;
- copying the input project once to a new stage-private current-project
  directory, including an existing root `target/`;
- preparing that copy with ordinary Crat `expand,unexpand` in one in-place
  invocation, with `--unexpand-use-print`;
- invoking `crat-tool` skeleton, safety-normalization, validation, and
  replacement operations;
- loading and checking skeleton JSON without parsing Rust;
- building and scheduling the function SCC graph;
- rendering dependency context and prompts;
- using PROCTOR's LLM client, prompt library, usage tracker, and pricing;
- extracting Rust code blocks without parsing their Rust contents;
- installing one candidate library source transactionally and invoking
  `cargo build`;
- restoring the previous source after failed candidate builds;
- managing the bounded repair loop; and
- copying the final current project to the declared output while excluding
  the root `target/`.

The stage must not parse or rewrite Rust source. Reading `Cargo.toml` with
`tomllib` to obtain the explicit `[lib].path` is project plumbing, not Rust
parsing. Require that value to be a string whose lexically normalized path is
one root-level file inside the crate. Permit optional `.` components but
reject every `..` component; nested, absolute, and crate-escaping library
paths are unsupported and fail before Crat is invoked. The stage never mutates
the input artifact or writes the output destination before every SCC has
succeeded.

### 7.1 Preparation order

Use this exact order:

1. Validate the stage config and envelope paths, refuse an existing output
   destination, parse the input `Cargo.toml`, and validate its `[lib].path` as
   a root-level library source before any build, copy, or tool call.
2. Build or locate `crat` and `crat-tool`.
3. Copy the complete input project to `<workdir>/current`.
4. Run ordinary Crat in-place with passes `expand,unexpand`, the
   `--unexpand-use-print` flag. If the copied project contains a regular
   root-level `config.toml`, pass it with `--config`; otherwise omit
   `--config` and use Crat's defaults.
5. Require the validated library source path to identify a regular file in
   the prepared current project.
6. Run `crat-tool make-skeleton` against the prepared current project.
7. Load the immutable skeleton records.
8. Run `crat-tool normalize-safety` to a scratch `.rs` file and atomically
   install it as the current library source.
9. Run `cargo build` in the current project. Failure here aborts Phase 4; it
   is not an LLM repair opportunity.
10. Build the graph and process all SCCs.
11. After all SCCs succeed, copy current to the output while ignoring only
    the root `target/`.

The Python runner must assemble the four `crat-tool` operations in exactly
these shapes:

```text
<crat-tool> make-skeleton --output <skeletons.json> <current-project>
<crat-tool> normalize-safety --output <normalized.rs> <library-source>
<crat-tool> validate --input <validation-request.json> --output <validation-response.json>
<crat-tool> replace --request <replacement-request.json> --output <candidate.rs> <current-project>
```

Each request/output pathname is stage-private. Before launching an operation
that is expected to create an output, remove any previous scratch output at
that exact path. Success requires both a zero exit status and a newly created
regular output file. A nonzero exit, missing output, non-regular output, or
stale-output reuse is a fatal tool/protocol failure. This applies equally to
skeleton, normalization, validation, and replacement. The ordinary Crat
preparation command is likewise fatal on nonzero exit.

Preparation or integration subprocess failures, malformed skeleton data,
validator `setup_error`, replacement failure, and inability to restore a
source transaction abort the stage immediately. They are tool or invariant
failures, not LLM repair failures.

`proctor.toml`, `Cargo.toml`, `Cargo.lock`, the explicit bin-target sources,
and all non-library files are copied normally and are never rewritten by
Python. Do not require or parse `proctor.toml`; if present, its exact bytes
must reach the final output unchanged. In particular, ignore a nonempty
`wrappers` list. Existing wrapper functions visible in skeleton JSON are
ordinary functions for this prototype because no metadata-aware inclusion or
exclusion is performed.

### 7.2 Shared LLM infrastructure

Use `proctor.llm.client.LlmClient` directly. Construct `UsageTracker` from the
run/stage/item identity and `PricingTable.from_config`, writing to
`framework.usage_log` when supplied and otherwise to
`<framework.workdir>/usage.jsonl`. The fallback keeps direct standalone stage
invocations fully tracked; ordinary PROCTOR runs always supply the official
stage-local usage path. Load the versioned stage-local prompt through
`PromptLibrary`; do not embed an unversioned prompt string in Python.

Make a shallow private copy of `framework.llm` and set its effective
`context_overflow` to `"error"` before constructing `LlmClient`. Do not mutate
the envelope's settings. The current shared client already supports `"error"`
and uses it by default, so Phase 4 requires no framework change for this
policy. A provider `ContextLimitExceeded` therefore records the failed attempt
through the shared tracker and then aborts the entire transformation. It is
never truncated, retried as an SCC repair, or counted against the ten repair
calls. Other LLM errors follow the shared client's provider-retry policy; if
the client ultimately raises, abort the stage.

Each SCC generation is one independent `Request` containing one user message.
No assistant/user history is retained. A repair is another independent
request containing the complete original material plus only the latest failed
text and latest diagnostics. Attach `RequestMetadata` with the run ID, stage
ID, an SCC item string formed by joining the ascending decimal member item IDs
with commas (for example, `0` or `0,3`), and the rendered prompt
ID/version/hash.

Aggregate every usage record, including provider retries and failed calls,
into the final `StageOutput.usage`. Report each distinct provider/model pair
used, in first-observed order, and the one prompt ID/version. A failure output
should still report usage accumulated before failure when it can do so
reliably. Derive the aggregate from the `UsageTracker` records rather than
only from successful `Response` objects: a retrying logical generation may
therefore contribute multiple usage calls. The generation-call metric counts
logical SCC requests, while `StageOutput.usage.calls` counts provider attempts
recorded by the tracker.

Use the framework's reporting convention when aggregating tracker records:

- with no LLM provider attempt, report `usage = null`, `models = []`, and
  `prompts = []`;
- otherwise, sum the integer input, cached-input, and output token fields;
- report `reasoning_tokens = null` when every record has null reasoning usage,
  otherwise sum the nonnull reasoning-token values;
- sum nonnull costs, but report `cost_usd = null` if any token-bearing record
  has unknown cost; a failed zero-token attempt with null cost does not make an
  otherwise known total unknown, and a nonempty collection containing only
  zero-token failed attempts reports `0.0`; and
- report the prompt once whenever at least one logical LLM request was issued,
  including when every provider attempt failed.

Report these exact flat metric keys on success and, where known, on failure:
`function_count`, `scc_count`, `llm_generation_calls`, `repair_calls`,
`structural_failures`, `compilation_failures`, and `cargo_builds`. The Cargo
count includes the normalized initial build. Metrics do not replace the
per-attempt usage log.

### 7.3 Skeleton loading

Represent loaded records with typed Python dataclasses or an equivalent typed
model. Validate only the integration contract needed by Python; do not
duplicate Crat's semantic tests. Require:

- a top-level JSON array;
- integer IDs in the inclusive Rust `u64` range `0..=18446744073709551615`
  (booleans are not integers here);
- globally unique IDs;
- paths unique within the record's dependency namespace: `Fn`, `Static`, and
  `Const` are value-namespace records, while `TyAlias`, `Enum`, `Struct`, and
  `Union` are type-namespace records. Permit one value record and one type
  record to have the same display path, as in the Phase 1 `type X`/`const X`
  regression;
- one of the seven Section 6 kinds;
- the Section 6 required string and dependency-list fields for each kind;
- integer dependency IDs that resolve to an included record; and
- `signature_dependencies` to be a subset of `dependencies` for `Fn`,
  `Static`, and `Const`.

Preserve every Rust text field exactly as decoded from JSON. Sort or deduplicate
nothing while loading: reject duplicate dependency IDs and require Crat's
lists to already be in strictly increasing item-ID order. These checks expose
corrupt or incompatible tool output without retesting how Crat generated the
contents.

## 8. Function graph and SCC scheduling

### 8.1 Function graph

Build a graph containing only transformable `Fn` items.

For each function `f`, inspect function-valued entries in its `dependencies`.

Add an edge:

```text
f -> g
```

when `f` directly calls `g`.

An ID naming a non-function record is not an edge. Foreign functions are
absent from the records and graph. Keep direct self-edges. Traverse function
nodes and adjacency lists in increasing item-ID order so the result does not
depend on JSON object identity or Python set order.

### 8.2 SCCs

Compute strongly connected components and the SCC condensation DAG.

A leaf SCC has no outgoing edge to an unprocessed SCC.

Because edges point from callers to callees, leaf-first processing translates callees before external callers.

A singleton SCC is recursive only when its function has a self-edge. Store
members of each SCC in increasing item-ID order.

### 8.3 Deterministic scheduling

At every scheduling step, recompute or update the set of unprocessed leaf
SCCs. Choose the leaf whose smallest member item ID is smallest. Mark an SCC
processed only after its candidate source builds successfully. This fixes one
exact schedule for disconnected components as well as call chains and cycles.

### 8.4 Function-name uniqueness

Before processing an SCC, the orchestrator checks that the final function names of all SCC members are distinct.

Crat identifies returned functions by name inside the single LLM response. Therefore, uniqueness is required only within the current SCC.

If duplicate function names occur within one SCC, orchestration aborts.

Functions in different SCCs may have the same final name.

Perform this check immediately before the SCC's first LLM request. Use member
item-ID order for the diagnostic. Do not globally reject duplicate final names
in different SCCs.

## 9. Prompt-context construction

Each prompt has two conceptually separate parts:

1. **Transformation Targets**
2. **Dependency Context**

The transformation targets do not count toward the dependency-character limit.

The dependency context has a limit of 100,000 characters.

All dependency records are deduplicated by item ID and ordered deterministically by ID.

Context construction operates on records and strings only. It does not inspect
Rust syntax.

### 9.1 Transformation targets

For every function in the current SCC, include:

- its annotated source function;
- its annotated target skeleton.

These snippets already contain the source and target signatures, so those signatures are not repeated separately.

### 9.2 SCC signatures

If the SCC contains multiple functions, include the source and target signatures of every SCC member in the dependency context.

For a directly recursive singleton SCC, include its own source and target signatures.

For a nonrecursive singleton SCC, omit the redundant self-signature dependency.

Represent an SCC signature dependency with the same function-context rendering
used for an ordinary function dependency. Do not emit an SCC member twice
when it is also present in another member's direct dependency list.

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

Build the union of every target's direct dependencies. Remove IDs belonging to
the current SCC because their required signatures are handled by Section 9.2.
The remaining direct IDs form mandatory depth zero.

Traverse subsequent dependencies breadth-first over the union graph:

- from a `Fn`, `Static`, or `Const`, follow only
  `signature_dependencies`; and
- from a `TyAlias`, `Enum`, `Struct`, or `Union`, follow `dependencies`.

Do not follow the body-only portion of a function, static, or const
dependency. Deduplicate an ID at its shortest depth, and do not traverse back
through an SCC target. Within a depth and in the final rendering, order records
by item ID.

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
3. Render those mandatory entries together in item-ID order.
4. Abort before the LLM call if that rendering exceeds 100,000 Python
   characters.
5. Tentatively add the complete next breadth-first depth.
6. Keep the depth only if the complete re-rendered context is at most 100,000
   characters.
7. Continue until the next complete depth does not fit.
8. Once a depth is rejected, do not consider deeper depths.

If mandatory direct dependencies already exceed 100,000 characters, abort the SCC instead of silently omitting them.

Instructions, transformation targets, prior failed code, and diagnostics are outside this limit.

Count characters with Python `len()` on the fully rendered Unicode string,
including headings, fences, separators, and newlines. An empty dependency
context is the empty string. Join nonempty entries with exactly two newline
characters and add no leading or trailing separator.

Render entries exactly in these shapes, substituting the record's exact text:

````text
### Function <id>: <path>
Source signature:
```rust
<source_signature>
```
Target signature:
```rust
<target_signature>
```
````

````text
### <Static-or-Const> <id>: <path>
```rust
<declaration>
```
````

````text
### <TyAlias-or-Enum-or-Struct-or-Union> <id>: <path>
```rust
<definition>
```
````

Render transformation targets in SCC member-ID order and join them with two
newlines:

````text
### Function <id>: <path>
Source:
```rust
<annotated_source>
```
Target skeleton:
```rust
<annotated_skeleton>
```
````

## 10. Initial LLM prompt template

Store the following exact text as stage-local `PromptLibrary` prompt
`local_transformation`, version `1`, with variables `dependency_context`,
`transformation_targets`, and the single pre-rendered `repair_context`.
The prompt file's frontmatter is exactly:

```toml
+++
id = "local_transformation"
version = 1
description = "Transform one Rust function SCC against Crat skeletons."
variables = ["dependency_context", "transformation_targets", "repair_context"]
+++
```

The exact prompt body begins after that frontmatter:

````text
You are transforming unsafe Rust functions generated from C.

The source code is the original implementation. The target skeleton defines
the transformation goal. A dependency's source signature is its signature
before transformation; its target signature is how transformed code must call
it.

Implement every function in Transformation Targets exactly once. Emit no
other top-level item. Use Dependency Context only as reference; do not emit or
redefine its functions, types, statics, or constants.

Requirements:

1. Exactly preserve source behavior wherever it is defined, including apparent
   bugs. Do not add validation, fallback behavior, or error handling absent
   from the source; preserve its preconditions. For example, if the source
   immediately dereferences a raw pointer, directly unwrap the corresponding
   `Option` instead of adding a conditional check.
2. Use the target skeleton's exact lifetime-generic declaration, parameter
   types, return type, and local-variable types.
3. Call transformed function dependencies using their target signatures.
4. Keep every existing function, parameter, and local-binding name. Preserve
   each existing declaration exactly once in the same label, pattern, and
   control-flow role.
5. Name every new local binding `proctor_temp_var_n`, where `n` is a
   nonnegative integer. Use it only within the consecutive statements carrying
   the same `#[proctor(N)]` label that encloses its declaration, including
   unlabeled code nested within those statements.
6. Do not define a function, type, static, constant, module, or other item
   inside a transformation target.
7. At each existing statement-list level, preserve every source
   `#[proctor(N)]` label in order. A labeled statement may expand only into one
   or more consecutive sibling statements with the same label. Do not insert
   unlabeled siblings at that level, repeat a label in nested code, or label
   newly introduced nested statements.
8. Preserve each existing control form, its direct role, its
   branch/arm/guard/body structure, and all existing nested labels. Plain
   blocks, `if`, `if let`, `while`, `while let`, `for`, `loop`, and `match`
   are distinct. A `let ... else` must remain a `let ... else`. A control form
   used directly as a `let` initializer, `return` value, `break` value, or
   match-arm result must remain in that role. Conditions, scrutinees, patterns,
   and statement contents may be rewritten.
9. If a labeled statement containing a control form expands into multiple
   same-label siblings, exactly one sibling must preserve that form, role, and
   all its existing labeled nested statements. Other siblings must not have a
   control form in that same direct role and must contain no labels below their
   own group label.
10. Do not introduce an explicit `unsafe` block or a statement or expression
    attribute other than the required `#[proctor(N)]` labels.
11. Return exactly one Rust code block delimited by triple-backtick fences.
    Include all requested functions and no prose. Do not use tilde or
    longer-backtick fences.

Example:

Source:

```rust
unsafe fn read_value(mut p: *const i32, mut q: *const i32) -> i32 {
    #[proctor(0)]
    let mut x: i32 = *p.add(1);
    #[proctor(1)]
    return if q.is_null() {
        #[proctor(2)]
        x
    } else {
        #[proctor(3)]
        x + *q
    };
}
```

Target skeleton:

```rust
unsafe fn read_value(mut p: &[i32], mut q: Option<&i32>) -> i32 {
    #[proctor(0)]
    let mut x: i32 = todo!();
    #[proctor(1)]
    return if todo!() {
        #[proctor(2)]
        todo!()
    } else {
        #[proctor(3)]
        todo!()
    };
}
```

Valid output:

```rust
unsafe fn read_value(mut p: &[i32], mut q: Option<&i32>) -> i32 {
    #[proctor(0)]
    let mut x: i32 = p[1];
    #[proctor(1)]
    return if q.is_none() {
        #[proctor(2)]
        x
    } else {
        #[proctor(3)]
        x + *q.unwrap()
    };
}
```

Dependency Context:

{{ dependency_context }}

Transformation Targets:

{{ transformation_targets }}

{{ repair_context }}
````

For a repair request, use:

````text
The previous transformation failed.

Previous transformation:

```rust
<latest failed text>
```

Diagnostics:

```text
<latest diagnostics>
```

Regenerate every function in Transformation Targets.
````

The initial request renders `repair_context` as the empty string. A repair
renders exactly the block above, with only the latest failed text and latest
diagnostics substituted into the complete original prompt. Every render goes
through `PromptLibrary`, and the request metadata records prompt ID
`local_transformation`, version `1`, and the rendered content hash.

## 11. LLM response extraction

The orchestrator instructs the LLM to return exactly one Rust code block
delimited by triple backticks and no prose. Only triple-backtick fences are
recognized; tilde fences and fences of four or more backticks are not blocks.

A recognized opening fence starts in column zero and is exactly three
backticks followed either by no tag or immediately by one nonempty ASCII
language tag containing only letters, digits, `_`, `+`, or `-`. Nothing else
may occur on that line. A recognized closing fence starts in column zero,
contains exactly three backticks and no other character, and ends at the next
line ending or end of response. Accept LF and CRLF as fence-line endings.
Indented fences, inline fences, opening tags containing whitespace or
punctuation outside that set, closing fences with trailing whitespace, and
unclosed fences are not recognized. Pair recognized opening and closing
fences from left to right without overlap.

To tolerate formatting errors:

1. Find all triple-backtick fenced code blocks.
2. Ignore prose outside code blocks.
3. If one block exists, use it.
4. If multiple blocks exist, choose the longest.
5. If multiple longest blocks have equal length, choose the first.
6. If no fenced code block exists, report a structural failure.

Pass the selected block unchanged to Crat.

The orchestrator does not parse Rust.

Measure block length by the number of content characters, excluding the
opening/closing fence, optional language tag, and exactly one LF or CRLF that
separates the content from each fence. Preserve every other character and line
ending exactly; do not otherwise strip or normalize the content. If no block
exists, the raw LLM response becomes the latest failed text and the
deterministic diagnostic is:

```text
The LLM response contained no triple-backtick fenced code block; return exactly one triple-backtick fenced Rust code block.
```

This is an ordinary repairable response-format failure.

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
- existing local-binding names.

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
Function-local parsed items are not existing declarations in supported target
skeletons: Phase 1 rejects them before producing a record, and Phase 2 rejects
them in both expected skeletons and returned transformations.

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

The target skeleton contains only statement-root `#[proctor(N)]` attributes
in function bodies. A result may not introduce any other statement or
expression attribute, and a `proctor` attribute may not appear anywhere
except the root attribute storage of a statement in an expansion group.
Function-header attributes are ignored because item replacement preserves the
current project's attributes instead of accepting attributes from the LLM
header.

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
- no function-local item declaration of any kind; and
- no conflict between request metadata and parsed skeletons.

Report the first setup error in deterministic check order. Setup errors abort
result validation because any result diagnostics would be untrustworthy. A
prohibited function-local item in an expected skeleton is
`invalid_expected_skeleton`; the message identifies the function, label when
available, and observed item kind. Scan item statements recursively at every
supported statement-list depth. This setup check defends the Phase 1/2
boundary even though the amended Phase 1 generator no longer emits such a
skeleton.

A supported expected-skeleton signature may contain only named lifetime
parameters without attributes or bounds and may not contain a syntactically
present `where` clause, even an empty one. Phase 1 never generates those
constructs.

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
- complete lifetime-generic declaration;
- parameter count;
- parameter names;
- parameter types;
- return type.

The generated function must declare exactly the same lifetime parameters as
the target skeleton, in the same order and with the same names. Every generic
parameter must be a lifetime parameter. Added lifetime bounds, type
parameters, and const parameters are rejected. A lifetime parameter may not
carry an attribute, and any syntactically present `where` clause is rejected,
including an empty one. Any difference in this declaration produces one
`generic_parameter_mismatch` for the function. Its message shows the expected
and observed declarations and instructs the LLM to copy the target skeleton's
complete lifetime-generic declaration.

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

Crat does not validate the generated function's:

- visibility;
- ABI;
- `unsafe` qualifier;
- `const` qualifier; or
- binding mutability in parameters.

Explicit lifetime names appearing inside parameter and return types remain
part of structural type comparison independently of the complete
lifetime-generic declaration check.

During replacement, Crat uses the validated lifetime-generic declaration,
parameter patterns and types, return type, and body. It ignores the LLM
function's visibility, ABI, safety, `const` qualifier, and attributes, composing
those properties from the current project as specified in Section 16.3.

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
- prohibition of every function-local item;
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
fixed by `phase-2-test-plan.md`, as explicitly amended by Section 4 of
`phase-3-test-plan.md`: add `generic_parameter_mismatch` and remove the four
obsolete existing-item codes listed in Section 14.8. No remaining code is
repurposed.

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
| Signature | Lifetime-generic declaration; parameter count; parameter names in parameter order; parameter types in parameter order; return type. A generic-declaration mismatch is one `generic_parameter_mismatch` and does not suppress parameter or return checks. A count mismatch suppresses name/type checks for parameter positions that cannot be paired, but not checks for safely paired positions or the return type. |
| Existing declarations | For each target binding in structural preorder, first associate by declaration identity and structural position. One occurrence in a wrong position produces only the applicable location-mismatch error. No occurrence produces `missing_existing_binding`; multiple occurrences produce `duplicate_existing_binding`. For a uniquely associated binding, compare by-value/`ref` mode before explicit-type presence and type. After expected bindings, report result-only bindings and any prohibited function-local items in result source order. |
| Label syntax and placement | Report malformed or misplaced `proctor` attributes in result source order before sequence diagnostics. A malformed attribute occupying an expected group position suppresses a derivative `missing_label` there. A misplaced attribute does not create a statement label. |
| Label sequence | After well-formed root labels are collected, report nested repetition, then nonconsecutive reappearance, then order mismatch, then missing expected labels in target preorder, unexpected labels in result source order, and unlabeled sibling statements in result source order. Do not report `label_order_mismatch` for a sequence already explained by `nonconsecutive_label`. |
| Controls and descendants | Validate the parent control kind, statement role, branch/arm shape, and required control-root count before descendant placement. A parent failure suppresses descendant label, declaration, type, and temporary-locality errors only beneath the unreliable role. With a valid parent, a label found in the wrong branch, arm, or structural statement list produces `descendant_location_mismatch`, not a missing-plus-unexpected-label pair. |
| Generated temporaries | Report invalid generated binding names, duplicate declarations, unresolved generated-temporary references, cross-group references, and macro-token occurrences in that order, each in result source order. Suppress a locality error when the declaration or enclosing expansion group cannot be associated reliably. |
| Body safety and attributes | Report explicit unsafe blocks in result source order, followed by unexpected statement/expression attributes in result source order. These AST-local checks do not depend on successful label/control association. |

For a required control group, zero control roots is
`missing_control_root` and more than one is `multiple_control_roots`. A value
path whose complete local identifier matches `proctor_temp_var_n` but resolves
to neither an expected existing local nor a declared generated temporary is
`unresolved_generated_temporary`. Any function-local `StmtKind::Item` in a
returned transformation uses `unexpected_nested_item`; there is no expected
local-item association or structure comparison. Report the item once and do
not descend into its initializer or body for derivative label, binding,
temporary, attribute, or unsafe-block diagnostics. The obsolete
`missing_existing_item`, `duplicate_existing_item`,
`existing_item_location_mismatch`, and `existing_item_structure_mismatch`
codes are removed.

A full path is not required in validation output. The orchestrator can obtain
it from the skeleton JSON.

## 15. SCC transaction policy

Each SCC is all-or-nothing.

No function in the SCC is committed until:

1. every function passes Crat structural validation; and
2. the current project with the complete SCC candidate installed passes
   `cargo build`.

If either step fails:

- restore the exact pre-attempt library source when a candidate had been
  installed;
- leave the current project source at the last successfully promoted SCC;
- discard partial success within the SCC; and
- regenerate every function in the SCC.

The current project's `target/` is an incremental Cargo cache and is not part
of the source transaction. A failed build may update it. Cargo fingerprints
the next build against the actually installed source, so Phase 4 retains the
cache rather than copying or rolling it back.

## 16. Item replacement and integration

### 16.1 One-time target-safety normalization

Phase 3 provides a pure source-to-source operation that Phase 4 invokes once,
after skeleton generation and before processing the first SCC. It receives the
original single-file library source and mechanically changes every
source-defined free-function header except `main` to `unsafe fn` in one global
operation. It does not consume skeleton JSON or a target-path list.

The in-memory operation:

```rust,ignore
pub fn normalize_target_safety(
    source: &str,
) -> Result<String, ReplacementError>;
```

recursively traverses the unexpanded surface AST through inline modules and
changes every body-bearing free `ItemKind::Fn` whose final identifier symbol
is not `main`. The name comparison therefore also excludes a surface spelling
of `r#main`. It does not inspect function paths, parameter types, return type,
body, visibility, ABI, attributes, `const`, async, or variadic qualifiers.
Those unsupported forms remain preconditions of the supported-input contract,
not normalization errors. Foreign declarations are not body-bearing free
items and remain unchanged.

Normalization:

- occurs after skeleton generation so `annotated_source` and
  `source_signature` still record original source safety;
- inserts `unsafe` only when it is absent and is idempotent for an already
  unsafe target;
- changes only the safety qualifier of every source-defined free function
  except `main`;
- leaves every excluded `main`, foreign declaration, context item, function
  body, type, ABI, visibility, attribute, and generated Cargo bin source
  unchanged; and
- creates no wrapper merely for a safe-to-unsafe normalization.

The production command writes only this returned source to its requested
`.rs` output path. Phase 4 atomically installs it into the stage-private
current project and builds the resulting normalized initial current project.

Normalizing every target before the first callee-first SCC replacement is
required for incremental compilation. Otherwise, replacing a safe callee with
an unsafe target could make an untransformed safe caller ill-formed. Once all
targets are unsafe, calls among still-untransformed and transformed targets may
occur directly inside unsafe functions. Phase 4 runs `cargo build` on this
normalized initial current project before beginning SCC processing.

### 16.2 Versioned replacement request

After structural validation succeeds, the orchestrator passes Crat the current
single-file library source and this typed request:

```rust,ignore
#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
#[serde(deny_unknown_fields)]
pub struct ReplacementRequest {
    pub schema_version: u64,
    pub items: Vec<ReplacementItem>,
    pub transformation: String,
}

#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
#[serde(deny_unknown_fields)]
pub struct ReplacementItem {
    pub id: u64,
    pub path: String,
    pub name: String,
}

pub fn replacement_request_from_json(
    input: &str,
) -> Result<ReplacementRequest, ReplacementError>;
```

Its exact JSON form is:

```json
{
  "schema_version": 1,
  "items": [
    {
      "id": 12,
      "path": "foo::bar",
      "name": "bar"
    }
  ],
  "transformation": "unsafe fn bar(...) { ... }"
}
```

`schema_version` and every `id` are JSON integers in the Rust `u64` range.
Version `1` is the only supported version. Reject unknown fields, an empty
`items` list, duplicate IDs, duplicate paths, duplicate names, empty paths,
and disagreement between `path`'s final identifier and `name`. Both fields use
the immutable skeleton's exact identifier spelling, including `r#` prefixes.
The transformation must parse as a crate containing exactly one top-level free
function for every requested name, with no duplicate, missing, or additional
function and no other top-level item. The request order is the stable
diagnostic and replacement order; transformation function order is irrelevant.
These checks are defensive integration checks even though the transformation
has already passed the Phase 2 validator.

The pure JSON parser performs no I/O and reports malformed JSON, unknown
fields, invalid integer representations, and request-metadata violations as
`InvalidRequest`. Phase 3 tests call it directly; the CLI uses the same parser.

Phase 3 replaces only functions, while `ReplacementRequest`,
`ReplacementItem`, the `item_replacer` module, and `replace_items` use item
terminology so that later phases can extend the operation without renaming its
public surface. The in-memory API is:

```rust,ignore
pub fn replace_items(
    source: &str,
    request: &ReplacementRequest,
    tcx: TyCtxt<'_>,
) -> Result<String, ReplacementError>;
```

`tcx` must have been created by compiling exactly `source`; it is used for
full-path identity and call-target resolution. The operation performs no
filesystem I/O. Use this deliberately small debugging-oriented taxonomy:

```rust,ignore
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum ReplacementErrorKind {
    InvalidRequest,
    InvalidTransformation,
    TargetResolution,
    UnsupportedConversion,
    UnsupportedCallRewrite,
    RewriteFailure,
}

#[derive(Debug, Clone, PartialEq, Eq)]
pub struct ReplacementError {
    pub kind: ReplacementErrorKind,
    pub item: Option<ReplacementItem>,
    pub message: String,
}
```

The item field supplies the requested ID/path/name when one item is
responsible, and the message describes the concrete failure. Do not add
fine-grained stable subcodes or a validator-style total error precedence:
replacement errors are unexpected integration/debugging failures and are not
sent to the LLM for repair. Return the first failure found by the deterministic
request checks and atomic algorithm documented here, iterating requested items
in request order. The production `replace` command exits nonzero and does not
write a usable output source file. These errors are not Phase 2 LLM-validation
failures and do not use the validator response schema. An incompilable but
successfully emitted replacement is instead handled by the ordinary
compiler-diagnostic LLM repair loop.

### 16.3 Atomic replacement algorithm

Resolve and validate the complete transaction against the unchanged current
source before mutating its AST:

1. Parse the unexpanded surface source and recursively map its items and
   expressions to the HIR compiled in `tcx`, using the same unexpanded
   `AstToHirMapper` discipline as skeleton generation and skipping
   automatically derived HIR items.
2. Resolve every requested full path to exactly one current source-defined
   free function. Require its final name to match the request, require it to be
   already `unsafe`, and reject `const`, async, and variadic functions or a
   function carrying both `#[no_mangle]` and `#[export_name = "..."]`.
3. Match every parsed transformation function by name. Recheck the supported
   lifetime-only generic form, plain identifier parameters, parameter count
   and names, and nonvariadic, nonasync form expected from validation.
4. Compare each current parameter and return type to its transformation
   counterpart to decide whether a compatibility wrapper is required. Use the
   same structural type normalization as the Phase 2 validator: ignore spans,
   node IDs, token caches, formatting, and redundant parentheses, but not real
   type structure. Parameter binding mutability does not cause a wrapper.
   Before conversion validation, identify the executable exemption solely by
   the requested function's final identifier symbol `main_0` and arity: arity
   zero leaves the sibling `main` unchanged, while arity two receives the
   explicit no-wrapper migration. Do not inspect parameter types, return type,
   safety, visibility, ABI, attributes, or body to classify these two forms.
5. Allocate all required same-module wrapper names and validate every required
   source/target conversion before emitting or changing any AST node.
6. Resolve every current direct call expression against HIR and snapshot which
   calls target a requested function. Calls inside requested SCC functions are
   not redirected. If an external call that requires redirection originates
   inside a source macro invocation's token input, return an error because the
   unexpanded surface AST cannot rewrite it reliably.
7. Rewrite the snapshotted external calls that require a wrapper, replace all
   requested functions, insert all wrappers, and perform the executable
   `main_0` migration when applicable.
8. Pretty-print the complete crate and return it only after the whole
   transaction succeeds.

This ordering prevents the generated wrapper's own call to the transformed
implementation from being mistaken for an external source call. Any error
leaves the caller's source string untouched and returns no partial output.
Previously inserted same-module wrappers are ordinary preserved items; a
later SCC transaction does not reconstruct or delete them.

For each requested function, compose the transformed implementation as
follows:

- preserve the current function's attributes except for ABI/export movement
  required by a wrapper;
- preserve its visibility and already-normalized `unsafe` qualifier;
- ignore the transformation function's attributes, visibility, ABI, safety,
  and `const` qualifier;
- take the validated lifetime-generic declaration, parameter patterns and
  types, return type, and body from the transformation;
- remove every validated `#[proctor(N)]` statement label recursively from the
  replacement body; and
- preserve no label in the emitted implementation or wrapper.

When no wrapper is required, preserve the current ABI, `#[no_mangle]`,
`#[export_name = ...]`, and all other current function attributes on the
implementation. When a wrapper is required, Section 17.3 defines the only ABI
and export-attribute movement. This header composition deliberately preserves
the original Rust API visibility and the Phase 3 safety normalization while
using the exact signature already checked by Phase 2. It does not copy a
complete LLM-written function item into the project.

### 16.4 Executable `main_0` migration

The excluded safe library `main` is not an LLM transformation target. When an
SCC target's final identifier symbol is `main_0`, classify it using only its
arity. At arity zero, leave the co-located sibling item whose final identifier
symbol is `main` unchanged: it already calls an unsafe `main_0` inside its
existing unsafe block.

At arity two, replace that co-located sibling `main` mechanically in the same
replacement transaction.
Do not generate a compatibility wrapper for the forced `main_0` `argv`
conversion: the supported `main` is migrated directly, and another
untransformed caller of this `main_0` form is unsupported.
Do not inspect either function's parameter types, return type, safety,
visibility, ABI, attributes, or body when recognizing the pair. The
supported-input model guarantees the two exact source forms in Section 2.1.
Any other `main_0` arity is unsupported and receives no executable migration.
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
specified in Section 16.4.

The transformed implementation:

- remains at the original Rust path;
- preserves the original visibility and already-normalized `unsafe` qualifier;
- uses the validated target lifetime-generic declaration, parameters, and
  return type; and
- uses Rust's default ABI when a wrapper is required.

The wrapper:

- is inserted as a sibling in exactly the same module as the transformed
  implementation;
- preserves the implementation's original visibility;
- is always unsafe;
- uses the source parameter and return types and the source ABI;
- delegates to the transformed implementation through its absolute path, such
  as `crate::foo::bar(...)`, rather than an unqualified call that a wrapper
  parameter could shadow;
- is an intentionally unsafe compatibility boundary.

The implementation and wrapper therefore do not widen a private,
`pub(super)`, or `pub(crate)` Rust API. Any caller that could access the
original function can access its sibling wrapper through the same module
boundary. The wrapper is unsafe even if the original function was safe,
because normalization has already made all transformation targets and their
untransformed Rust callers unsafe. The excluded executable `main` is the
deliberate safe exception.

### 17.2 Same-module wrapper name

The base wrapper identifier is:

```text
__proctor_wrapper_<original-final-identifier>
```

For example, a transformed `foo::bar` receives a sibling whose absolute path
begins as:

```text
crate::foo::__proctor_wrapper_bar
```

Treat the name of every sibling item as occupied, regardless of Rust namespace.
If the base identifier is occupied, try `<base>_0`, then `<base>_1`, and so on,
selecting the first identifier not occupied or reserved in that module.
Allocate names in replacement-request order against the complete unchanged
module and names already reserved by the same transaction, so multiple
wrappers added together cannot collide. Raw source identifiers use their
identifier symbol when forming the base. When adding wrappers for a new SCC,
preserve wrappers created for earlier SCCs.

### 17.3 Export handling

When no wrapper is required, the implementation preserves its current ABI and
all current export attributes.

When a wrapper is required:

- remove the explicit source ABI from the transformed implementation so it
  uses Rust's default ABI;
- apply that exact source ABI to the wrapper;
- remove `#[no_mangle]` and `#[export_name = ...]` from the implementation;
- if the source had `#[no_mangle]`, add
  `#[export_name = "<original-final-identifier>"]` to the differently named
  wrapper so the external symbol remains exact; and
- if the source had `#[export_name = "..."]`, move that exact attribute to the
  wrapper.

A source function carrying both `#[no_mangle]` and `#[export_name = "..."]` is
unsupported. Reject it before replacement rather than choosing one attribute
or emitting two wrapper export names.

All other current attributes remain on the transformed implementation and are
not copied to the wrapper. The transformation function's ABI and attributes
are ignored. These rules preserve an explicit ABI even when there is no export
attribute, because the wrapper is the compatibility entry point whenever the
Rust parameter or return types change.

The prototype considers only executable and `cdylib` targets.

### 17.4 Conversion logic

For each wrapper parameter, convert the source-signature value into the target
type independently. Let `p` be the wrapper parameter and `T` the pointee type:

| Target parameter type | Source raw-pointer conversion |
| --- | --- |
| `&T` | `&*(p as *const T)` |
| `&mut T` | `&mut *(p as *mut T)` |
| `Option<&T>` | `(p as *const T).as_ref()` |
| `Option<&mut T>` | `(p as *mut T).as_mut()` |
| `&[T]` | if `p.is_null()`, `&[]`; otherwise `std::slice::from_raw_parts(p as *const T, 1_000_000)` |
| `&mut [T]` | if `p.is_null()`, `&mut []`; otherwise `std::slice::from_raw_parts_mut(p as *mut T, 1_000_000)` |
| `Box<T>` | `Box::from_raw(p as *mut T)` |
| `Option<Box<T>>` | if `p.is_null()`, `None`; otherwise `Some(Box::from_raw(p as *mut T))` |
| raw pointer | preserve it when structurally equal, or cast to the exact target raw-pointer type |

An input conversion to `Box<[T]>` or `Option<Box<[T]>>` is unsupported in
Phase 3 and fails the complete replacement transaction before any output is
emitted. A nonpointer parameter whose source and target types are structurally
equal is passed through unchanged. Any other source/target pair is an
unsupported conversion.

The wrapper calls the implementation once. If the target function returns a
value, bind that call result once before converting it to the source return
type. Let `value` be that single target result:

| Target return type | Source raw-pointer conversion |
| --- | --- |
| `&T` | first cast `value` to `*const T`, then cast that raw pointer to the exact source raw-pointer type |
| `&mut T` | first cast `value` to `*mut T`, then cast that raw pointer to the exact source raw-pointer type |
| `Option<&T>` | `None` becomes a typed null matching the exact source pointer mutability; `Some(value)` first casts the reference to `*const T`, then to the exact source type |
| `Option<&mut T>` | `None` becomes a typed null matching the exact source pointer mutability; `Some(value)` first casts the reference to `*mut T`, then to the exact source type |
| `&[T]` | if empty, return a typed null matching the exact source pointer mutability; otherwise cast `value.as_ptr()` to the exact source type |
| `&mut [T]` | if empty, return a typed null matching the exact source pointer mutability; otherwise cast `value.as_mut_ptr()` to the exact source type |
| `Box<T>` | cast the `*mut T` from `Box::into_raw(value)` to the exact source type |
| `Option<Box<T>>` | `None` becomes a typed null matching the exact source pointer mutability; `Some(value)` casts the `*mut T` from `Box::into_raw(value)` to the exact source type |
| `Box<[T]>` | if empty, drop the box and return a typed null matching the exact source pointer mutability; otherwise cast the `*mut T` from `Box::leak(value).as_mut_ptr()` to the exact source type |
| `Option<Box<[T]>>` | `None` and `Some(empty)` return a typed null matching the exact source pointer mutability, dropping an empty box; `Some(nonempty)` casts the `*mut T` from `Box::leak(value).as_mut_ptr()` to the exact source type |
| raw pointer | preserve it when structurally equal, or cast to the exact source raw-pointer type |

Every null branch uses `std::ptr::null()` for an exact `*const T` source return
and `std::ptr::null_mut()` for an exact `*mut T` source return, followed by a
cast when needed to reproduce the exact source spelling/type. Do not cast a
shared reference directly to `*mut T`: first obtain `*const T` and then cast
the raw pointer. Likewise, first obtain `*mut T` from a mutable reference,
mutable slice, or box before any cast to an exact `*const T` source return. A
nonpointer return whose source and target types are structurally equal passes
through unchanged. Any other pair is unsupported. The unit return requires no
temporary.

These conversions are deliberately unchecked. In particular, nonoptional
references and `Box<T>` do not test nullity; their callers must satisfy Rust's
nonnull, validity, ownership, and aliasing contracts. A null slice input maps
to an empty slice, and an empty slice-like return maps to null. The fixed
provisional slice bound is exactly `1_000_000`. Phase 3 does not introduce
`Option<&[T]>` or `Option<&mut [T]>`; distinguishing nullable slices is
deferred.

Global-variable wrappers are not needed because global types are unchanged.

## 18. Call-site rewriting

When replacing an SCC, Crat rewrites direct calls to each wrapper-requiring SCC
function.

Rewrite only callers outside the current SCC.

Use an absolute wrapper path:

```rust
crate::foo::__proctor_wrapper_bar
```

Crat must:

- leave calls between functions in the same SCC unchanged;
- leave calls to signature-unchanged functions unchanged;
- use HIR resolution rather than path spelling to identify the callee, so
  unqualified, imported, aliased, `self`, `super`, `crate`, and fully
  qualified direct calls are handled consistently;
- replace the complete callee expression with the allocated absolute wrapper
  path while preserving the call's arguments and surrounding expression; and
- rewrite all statically resolved direct calls from external untransformed
  callers, including multiple calls and calls nested in ordinary expressions
  or control flow.

A required external call written inside an expression- or statement-macro
token input is an error, not a silently stale call. Such source is already
excluded by Section 2.1. Calls introduced only by macro expansion do not have a
source call expression to rewrite and do not violate this rule.

Because SCCs are processed callee-first:

- all callees outside the current SCC have already been transformed;
- external callers of the current SCC are still untransformed;
- those external callers temporarily call wrappers;
- when an external caller is later transformed, its whole function is replaced;
- its LLM-generated body directly uses target signatures.

No separate transformed-function registry is required.

The immutable skeleton JSON may contain stale source snippets after earlier call-site rewriting. This is acceptable because each function is later replaced as a whole using its original annotated source and target skeleton.

## 19. Compilation and promotion

After Crat emits the replacement `.rs` file, Phase 4 uses a source-file
transaction in the one stage-private current project:

1. Keep the candidate outside the project tree.
2. Copy the exact current library source to a rollback path outside the
   project tree.
3. Atomically replace the current library source with the candidate.
4. Run:

```bash
cargo build
```

in the current-project directory.

The rollback copy must not have an `.rs` path inside the Cargo project. Wrap
installation and building in `try/finally`: every unsuccessful or exceptional
attempt after installation restores the previous source atomically. Failure
to restore is a fatal stage error. The stage's input artifact remains
read-only throughout, and a killed stage can damage only disposable work
state; a fresh invocation recreates `<workdir>/current` from the input rather
than resuming an uncommitted swap.

### Success

- Keep the installed candidate as the current library source.
- Delete the rollback copy.
- Mark the SCC as processed.
- Select the next leaf SCC.

### Failure

- Capture compiler standard output and standard error.
- Restore the rollback source.
- Keep all other current-project files unchanged. Retain `target/`.
- Start a repair attempt for the entire SCC.

The prototype assumes Crat's integration routines are correct. A nonzero or
malformed `replace` operation aborts rather than asking the LLM to repair an
integration failure. Compiler diagnostics are given to the LLM even when they
refer outside the SCC. The LLM may change only SCC functions. If that is
insufficient, the retry limit eventually aborts orchestration.

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
3. If valid, ask Crat to emit a replacement `.rs` file.
4. Transactionally install it in the current project and run `cargo build`.
5. Keep it on success or restore and retry on failure.

If the SCC has not succeeded after ten repair calls, abort the complete orchestration immediately.

The initial call plus ten repair calls means at most eleven LLM generation
calls per SCC, excluding provider-level retries internal to `LlmClient`.
Increment the repair count before each repair LLM call. Structural
`invalid`, missing-fence extraction, and failed candidate compilation consume
the same counter. Validator `setup_error`, validator process/protocol failure,
replacement failure, preparation failure, safety-normalization failure,
initial normalized-project build failure, malformed skeleton data, exhausted
provider retries, authentication failure, and context overflow abort
immediately and do not consume an SCC repair.

For structural invalidity, use the complete raw text of the validator response
file, unchanged after it has been parsed successfully, as the latest
diagnostics and use the selected code block as the latest failed
transformation. Do not reserialize the parsed JSON: whitespace, key order, and
the presence or absence of a final newline in the tool's response are part of
the diagnostic text sent on that repair. For a missing fence, use the complete
raw response as failed text and the deterministic extraction diagnostic. For
compilation failure, use the selected transformation and both captured
streams, labeled `cargo build stdout:` and `cargo build stderr:`. Do not
truncate diagnostics; if the resulting provider request exceeds its context
window, the required `context_overflow = "error"` behavior aborts honestly.

## 21. Completion

The orchestrator succeeds when every SCC has been translated and the final promoted project builds.

For this prototype:

- all wrappers remain in the final project;
- no test suite is executed;
- no reusable rules are extracted;
- no rule-set file is read or written;
- `proctor.toml` is neither required nor parsed and is preserved unchanged
  when present;
- wrapper metadata from non-local transformations is ignored even when
  nonempty;
- all supported free functions except the mechanically managed executable
  `main` are transformed;
- the final library source remains unsplit;
- explicit Cargo bin-target sources and Cargo metadata remain unchanged; and
- the final output excludes the root `target/`.

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
- the amended Phase 2 expected-skeleton setup check that rejects every
  function-local item;
- parsing result snippets;
- function matching by name;
- exact expected-function set checks;
- exact complete lifetime-generic declaration checks, with only Phase
  1-generated named lifetime parameters supported, no lifetime-parameter
  attributes or syntactically present `where` clause accepted, and
  `generic_parameter_mismatch` added to the stable code set;
- structural target-signature and existing-local-type checks, ignoring all
  binding mutability;
- label expansion groups;
- label syntax, placement, order, grouping, and nesting checks;
- recursive control-kind, branch/arm, plain-block, and `let-else`
  preservation consistent with Phase 1;
- control-statement expansion rules;
- strict declaration-identity, by-value-versus-`ref` binding-mode, and
  placement preservation;
- rejection of every function-local item in a returned transformation;
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

Implement every still-applicable case in `phase-2-test-plan.md` plus the Phase
1/2 amendment cases in Section 4 of `phase-3-test-plan.md`. Update or remove
affected existing Phase 1 and Phase 2 Rust tests, but do not edit either
historical planning document. All functional tests use in-memory APIs and
perform no filesystem writes or subprocess invocation. The CLI wiring is not
tested in Phase 2. The new source-safety and executable decisions add
generator work and generator regressions during Phase 2, but no additional
validator rule: safety was already an ignored LLM-header property, `main`
never enters a validation request, and the forced `main_0` type is validated
through the ordinary structural parameter-type rule. The lifetime-generic
declaration check, complete function-local-item rejection, and arity-only
executable recognition are the three intentional amendments added during
Phase 3 planning.

### Phase 3: Item replacement and integration

Implement and test:

- an `item_replacer` module with a typed versioned `ReplacementRequest`,
  structured `ReplacementError`, the in-memory `replace_items` operation, and
  focused tests in `item_replacer/tests.rs`;
- the completed-Phase-1 generator amendment that rejects every
  function-local `StmtKind::Item`, plus removal of obsolete local-const/static
  generator tests;
- the completed-Phase-2 validator amendment that rejects every
  function-local item in expected skeletons and returned transformations, plus
  removal of obsolete local-item preservation logic, diagnostics, and tests;
- the one-time in-memory safety normalization of every source-defined free
  function except `main` after skeleton generation, without skeleton JSON or
  a path list;
- thin `normalize-safety` and `replace` CLI wiring that each writes only one
  `.rs` output file, without filesystem or CLI tests;
- function replacement by full path;
- preservation of the current function's visibility and normalized safety,
  validated target lifetime generics/parameters/return/body, and the exact
  header-composition rules in Section 16.3;
- recursive label removal;
- fixed mechanical replacement of the excluded executable `main` when
  committing the supported two-argument `main_0`, while leaving the
  zero-argument case and generated bin shim unchanged;
- same-module, collision-free wrapper generation;
- base/`_0`/`_1` wrapper-name allocation in request order across every sibling
  item namespace;
- absolute wrapper-to-implementation delegation;
- no wrapper for safety-only source/target differences;
- no wrapper for the forced two-argument `main_0` conversion;
- ABI, `no_mangle`, and `export_name` movement;
- rejection of simultaneous source `no_mangle` and `export_name`;
- every supported input and output conversion in Section 17.4, including the
  fixed slice bound and empty-slice/null policy;
- rejection of boxed-slice input conversion and all other unsupported pairs
  before output;
- HIR-resolved external call-site rewriting to absolute same-module wrapper
  paths;
- preservation of direct and mutual-recursive calls inside the SCC;
- rejection of a required rewrite inside macro-invocation input; and
- atomic failure with no partial rewritten source and the coarse
  debugging-oriented `ReplacementErrorKind` taxonomy from Section 16.2.

Implement every case in `phase-3-test-plan.md`. All functional tests use source
strings and in-memory compiler/parser APIs. They perform no filesystem writes,
do not construct projects, and do not invoke the CLI or subprocesses.

### Phase 4: Python orchestration

Implement:

- `stages/local-transformation/` as a typed standalone PROCTOR stage requiring
  and producing only `rust_project`;
- Crat/crat-tool warmup and process invocation with deterministic logs;
- one initial immutable input-project copy, retaining `target/` when present;
- mandatory in-place Crat `expand,unexpand` preparation without `split` or
  `bin`;
- the ordinary Crat `expand` cleanup correction that preserves every explicit
  `[[bin]].path`;
- the Crat skeleton-presentation amendment that forces every non-`ref`
  binding to `mut` in both source and target snippets/signatures while
  preserving `ref` and `ref mut` exactly;
- corresponding updates to the existing in-memory Crat skeleton tests, with no
  filesystem or CLI test added;
- strict skeleton JSON integration loading;
- one-time invocation of Phase 3 target-safety normalization and compilation of
  the normalized initial current project;
- SCC-local function-name uniqueness checks;
- function graph construction;
- SCC computation;
- deterministic leaf scheduling;
- exact dependency-entry and transformation-target rendering;
- dependency-context rendering;
- breadth-first type closure;
- 100,000-character dependency budget;
- a versioned stage-local PROCTOR prompt;
- PROCTOR `LlmClient`, `UsageTracker`, pricing, request metadata, and
  StageOutput aggregation;
- mandatory effective `context_overflow = "error"`;
- code-block extraction;
- validation invocation;
- item replacement to a scratch source;
- atomic candidate-source installation, `cargo build`, and rollback;
- repair accounting;
- source promotion without per-attempt project copies;
- byte-preserving `proctor.toml` and non-library project handling; and
- final output copying with root `target/` excluded.

Implement every case in `phase-4-test-plan.md`. Keep default Python tests
offline and independent of the real Crat toolchain, Cargo, and provider APIs.
Phase 4 adds no filesystem-changing Crat test. It updates existing in-memory
Crat skeleton tests only for the presentation-mutability amendment. Python
tests assert command construction, protocol handling, graph/context logic,
transactions, and stage reporting without revalidating skeleton, validator,
replacement, or Expand cleanup semantics.

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
- scalar boxes and supported boxed-slice returns;
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
- methods, traits, source-written generics, and closures;
- nullable-slice target types such as `Option<&[T]>` and
  `Option<&mut [T]>`, instead of Phase 3's provisional null/empty
  correspondence;
- boxed-slice and optional-boxed-slice wrapper input conversion;
- metadata-aware treatment of wrappers introduced by non-local
  transformation;
- reading or updating `proctor.toml` rather than preserving and ignoring it;
- Cargo target layouts beyond the explicit self-contained forwarding bin; and
- structured, token-aware prompt reduction after a provider context overflow.

## Amendment Plan 1: Restricted conditional expressions beneath non-control wrappers

This amendment intentionally relaxes the completed Phase 1 skeleton-generation
rule for control expressions beneath non-control wrappers. It applies after
Phases 1--4 and supersedes only the conflicting nested-control rejection and
statement-labeling requirements in this document. All historical phase test
plans remain unchanged.

### A1.1 Supported conditional-expression shape

An ordinary `if` expression may occur beneath a non-control expression wrapper
when it has the restricted, recursively conditional shape defined below. This
exception models C's conditional (`?:`) expression. It does not generally
permit Rust control flow beneath non-control wrappers.

A *restricted conditional* is an `if` expression satisfying all of these
requirements:

- it is an ordinary `if`, not `if let`;
- it has an `else`;
- its condition contains no control or plain-block expression;
- its `then` block contains exactly one statement, and that statement is a
  tail expression represented by `StmtKind::Expr`;
- its `else` is either:
  - an `else` block containing exactly one `StmtKind::Expr` tail expression; or
  - another restricted conditional, as in a syntactic `else if` chain; and
- each branch tail is either:
  - an expression containing no control or plain-block expression; or
  - a restricted conditional directly, recursively.

Consequently, both of these shapes are supported beneath a non-control
wrapper:

```rust,ignore
if c1 { a } else if c2 { b } else { c }
```

```rust,ignore
if c1 { a } else { if c2 { b } else { c } }
```

The second form is recursive because the sole tail expression of the outer
`else` block is itself a restricted conditional. A missing final `else`, an
empty branch, a semicolon-terminated branch expression, or any branch with a
preceding statement fails the restriction. For example, this remains
unsupported:

```rust,ignore
if c1 {
    foo();
    a
} else {
    b
}
```

An `if`, `if let`, `while`, `while let`, `for`, `loop`, `match`, or plain block
that occurs beneath a non-control wrapper and does not satisfy this exception
continues to produce `GenerationErrorKind::NestedControlPayload`. In
particular, a restricted conditional may occur directly as a branch tail, but
placing it beneath another wrapper inside that tail does not make the branch
restricted:

```rust,ignore
if c1 { 1 + if c2 { a } else { b } } else { c }
```

Direct-root control expressions retain the existing behavior. For example, a
direct `let` initializer whose root is `if` is still structurally preserved
and its branch statements are still labeled. This amendment applies only to a
restricted conditional encountered beneath a non-control wrapper.

### A1.2 Skeleton and labeling behavior

A non-control payload containing only permitted restricted conditionals is
still opaque to skeleton generation. Crat replaces that payload using the
existing hole rules rather than preserving any of its conditional structure.
For example:

```rust,ignore
x = 1 + if y { 2 } else { 3 };
```

is rendered in the skeleton as:

```rust,ignore
todo!();
```

The existing surrounding statement roles remain unchanged. A restricted
conditional inside a `let` initializer produces the existing typed
`let ... = todo!()` skeleton, one inside a direct `return` value produces
`return todo!()`, and one inside a direct `break` value produces
`break todo!()`.

The enclosing statement receives its ordinary numeric `#[proctor(...)]`
label. No statement inside any permitted restricted conditional beneath that
non-control wrapper receives a label, including statements in recursively
nested branch-tail conditionals or syntactic `else if` chains. The same label
tree is rendered in `annotated_source` and `annotated_skeleton`. This makes the
entire enclosing statement group one opaque transformation region.

Skipping labels must not skip input validation. Skeleton generation still
checks the complete source body for function-local items, empty statements,
and non-block match arms before applying the restricted-conditional label
exception.

Dependency collection remains unchanged. It continues to traverse the
original mapped HIR, so calls and referenced items inside an opaque restricted
conditional remain dependencies even though its branch statements are not
represented in the skeleton.

### A1.3 Implementation and validation

Implement the restriction once as a shared structural predicate used by both
nested-control validation and statement labeling. Do not duplicate slightly
different definitions in those paths. The skeletonizer must accept a
non-control payload only when every control or plain-block expression found
beneath it is a permitted restricted conditional; it then applies the existing
payload hole operation.

Update the in-memory Crat skeleton tests without editing any historical phase
test plan. The tests must cover:

- the assignment example above, including a single enclosing label and no
  branch labels in either annotated rendering;
- a syntactic `else if` chain;
- a restricted conditional used directly as a branch tail;
- the existing payload-role behavior for at least `let` and `return`;
- rejection of a branch with a preceding statement;
- rejection when a recursively nested conditional fails the restriction;
- rejection of a missing final `else`, `if let`, and non-`if` control beneath a
  non-control wrapper; and
- preservation of the existing labels and structure for a direct-root `if`.

No validator semantic change is required. The generated expected skeleton
contains only the enclosing labeled hole, and the Phase 2 validator already
permits unlabeled code introduced inside a leaf statement group. Retain or add
focused validator coverage demonstrating that behavior if needed, but do not
relax validation of hand-written expected skeletons containing control beneath
a non-control wrapper.

## Amendment Plan 2: Conservative preservation of transformation-independent statements

This amendment reduces unnecessary LLM rewriting without weakening the local
transformation boundary. It applies after Phases 1--4 and Amendment Plan 1. It
supersedes only the conflicting skeleton-hole, validation, replacement,
orchestration, prompt, JSON, and deferred-preservation requirements in this
document. In particular, “preservation labels” are no longer deferred. All
historical phase test plans remain unchanged. `amendment-2-test-plan.md` is the
exhaustive executable contract for this amendment.

The optimization is intentionally asymmetric. A false negative merely leaves a
statement as an LLM transformation target. A false positive permanently
discards a necessary LLM rewrite and can silently preserve incorrect code.
Consequently, Crat preserves a statement only after proving all conditions
below. Any missing mapping, unresolved callable, unclassifiable type, or other
uncertainty makes the statement require transformation.

### A2.1 Statement dispositions and recursive scope

Every existing numeric statement label has exactly one *statement
disposition*:

- `preserve`: restore the canonical statement from the target skeleton
  mechanically; or
- `transform`: retain the existing skeleton-hole behavior and accept the
  validated LLM expansion group.

The disposition belongs to the complete `rustc_ast::Stmt` subtree rooted at
that label, including its expression, patterns, types, control expressions,
and recursively nested statements. A statement is preservable only when its
entire subtree is preservable. Therefore, a preserved parent has no transformed
descendant. A transformed parent may contain independently preserved or
transformed descendants.

This recursive rule is deliberately conservative. For example, an `if`
statement with a pointer-free condition still receives disposition `transform`
when one nested branch statement requires transformation. Its ordinary
skeleton control form remains, its condition follows the existing hole rule,
and each nested statement follows its own disposition.

The classification is computed from the mapped, compiling source program and
the complete initial target decisions before skeleton payloads are mutated.
It is not inferred later by searching rendered text for `todo!()`.

### A2.2 Transformation-sensitive types

A type is *transformation-sensitive* when any of these rules applies:

1. The type is a raw pointer.
2. The type itself has different source and planned target representations.
3. The type recursively contains a transformation-sensitive component through
   a reference, array, slice, tuple, callable signature, or generic argument.
4. The type is a project-local struct, enum, or union and any fully
   instantiated field of any variant is transformation-sensitive.
5. A future analysis reports that any project-local field's source and target
   types differ, even if neither type contains a raw pointer.
6. Crat cannot prove that the normalized type is free of
   transformation-sensitive components.

Open project-local nominal ADTs recursively, substitute their actual generic
arguments into field types, inspect every enum variant and union field, and
use a visited set keyed by the instantiated nominal type so recursive types
terminate. Type aliases are transparent. This means that moving or copying a
project-local value whose deeply nested field contains a raw pointer is not
preserved. It also keeps the rule correct when a future pass changes a field
from a `Copy` type to a non-`Copy` type.

Do not recursively expose the private representation of a non-local nominal
type such as `Vec<T>`. Treat the non-local nominal definition as opaque, but
still inspect all of its explicit generic arguments. Thus `Vec<i32>` is not
rejected merely because the standard library implements it with a pointer,
while `Vec<*mut i32>` is transformation-sensitive.

The current prototype changes no named type or field definitions, so the
future field-difference input is initially empty. The classifier must
nevertheless centralize this query rather than equating
“transformation-sensitive” with `Ty::is_raw_ptr()`.

The existing supported-input exclusion of source-written generics remains
unchanged. Generic project-local ADT instantiation is nevertheless required
inside the low-level type-sensitivity helper and receives a focused compiler
unit test. That defensive helper coverage does not make a crate containing a
source-written generic item supported end-to-end by skeleton generation or
local-transformation orchestration.

### A2.3 Conservative preservation proof

Crat assigns `preserve` to a statement only when every condition in this
section succeeds for the complete statement subtree.

#### Mapped nodes, declarations, and patterns

- Every relevant surface AST node maps unambiguously to the expected HIR owner
  or local node.
- Every binding declared or referenced by the subtree has structurally equal
  source and target types.
- Every explicit statement-local type, inferred binding type, and complete
  pattern type is not transformation-sensitive.
- A declaration without an initializer is not vacuously safe: its binding and
  explicit or inferred type are still checked.
- Destructuring `let`, `let-else`, `if let`, `while let`, `for`, and match-arm
  patterns are checked recursively.

#### Expressions, places, and adjustments

- Visit every HIR expression corresponding to the subtree, including both
  lvalue/place and rvalue positions.
- Check the expression's unadjusted type, adjusted type, and every
  intermediate compiler adjustment target type.
- Check callable signatures separately; a `FnDef` or method expression does
  not prove that its parameter and return types are pointer-free merely
  because those types are not ordinary generic arguments of the expression
  type.
- Any inference, projection, opaque, error, or other type that cannot be
  normalized and proven insensitive makes the statement require
  transformation.

#### Calls and callable references

Every explicit or compiler-resolved function, method, overloaded operator, or
other callable operation in the subtree must be statically resolved.
Indirect calls and callable values require transformation under the
prototype's no-function-pointer contract.

A resolved call or callable reference requires transformation when any of
these holds:

- the target is a foreign function declaration;
- the target is non-local and its instantiated signature is unsafe;
- the target is a transformable local function whose source and target
  parameter or return types differ; or
- the callable's instantiated signature is transformation-sensitive.

“Non-local” means that the resolved `DefId` is not defined in the current
crate. This deliberately covers unsafe functions and methods from `std`,
`core`, `alloc`, compiler support crates, and third-party crates. A
pointer-free call to a local unsafe function may be preserved when its
signature is unchanged. Target-safety normalization alone is not a signature
change. A pointer-free call to a safe non-local function may likewise be
preserved.

The changed-local-signature check is required independently of expression
types. It prevents a canonical preserved statement from later overwriting a
temporary wrapper redirection or restoring an obsolete call convention.

#### Conservative syntax boundaries

Every expression- or statement-position macro invocation requires
transformation, even when its visible token tree appears pointer-free.
Expanded HIR calls do not provide a sufficiently reliable statement-local
presentation mapping for this optimization. Closures, inline assembly,
indirect calls, and any other unsupported or ambiguously desugared form also
require transformation.

Pointer-free access to a `static mut` or a union field still requires
transformation. These locally unsafe operations are not preservation
candidates. Raw-pointer operations are independently rejected by the type
rules, and unsafe non-local calls are independently rejected by the callable
rules. A pointer-free call to an unchanged-signature local unsafe function
remains governed by the explicit callable rule above and may be preserved.

### A2.4 Skeleton construction and JSON

For a `preserve` statement, put the canonical target-skeleton statement in
`annotated_skeleton` instead of replacing its payload with `todo!()`. The
canonical statement consists of:

- the original statement's expression, pattern, control, and nested-statement
  payload;
- the target local types already required by skeleton generation;
- the shared presentation binding-mutability normalization; and
- the original `#[proctor(N)]` label tree.

It is not necessarily the literal source AST. For example, an inferred
pointer-free local remains explicitly typed and presentation-mutable in the
target skeleton:

```rust,ignore
#[proctor(0)]
let mut x: i32 = y + z;
```

For a `transform` statement, retain all existing Phase 1 and Amendment Plan 1
skeleton behavior. A transformed parent is skeletonized using the existing
control structure and payload-hole rules, while recursively visited child
statements use their own dispositions.

A restricted conditional accepted by Amendment Plan 1 is preserved with its
complete enclosing statement when that statement subtree passes the Amendment
2 proof. If it does not pass, Amendment Plan 1's opaque payload hole and
single-label boundary remain unchanged.

Add these required fields to every `Fn` skeleton record, immediately after
`target_signature`:

```json
{
  "needs_transformation": true,
  "statements_requiring_transformation": [1, 3]
}
```

`statements_requiring_transformation` contains every `transform` label in
strictly increasing numeric order. Every other label in the annotated
skeleton is a `preserve` label. `needs_transformation` is exactly whether that
array is nonempty. The redundancy is intentional: Python schedules without
parsing Rust, while Crat retains the complete label-level proof.

The amended `Fn` serialization key order is:

```text
id, path, kind, name, annotated_source, annotated_skeleton,
source_signature, target_signature, needs_transformation,
statements_requiring_transformation, signature_dependencies, dependencies
```

Other record kinds are unchanged. Crat generation tests must verify that every
listed label exists, every skeleton label has one implied disposition, no
preserved label has a transformed descendant, and the Boolean equals array
nonemptiness.

### A2.5 Preservation-aware validation

The validation request remains schema version 1 because the prototype
protocol has not been released. Amendment 2 defines the final version-1
request shape before its first release; it does not preserve compatibility
with the earlier unmerged development shape. Each expected function carries
its item ID, name, complete annotated target skeleton,
`needs_transformation`, and `statements_requiring_transformation`.
Validation responses likewise remain schema version 1 with their existing
`valid`, `invalid`, and `setup_error` shapes.

Before applying the existing body validation, Crat constructs a canonicalized
copy of the returned function:

1. Align result expansion groups with expected groups at the current
   statement-list level.
2. For a preserved expected label, require its outer expansion group in the
   expected sibling position, discard the complete returned same-label group,
   and insert the one canonical target-skeleton statement subtree.
3. Treat the discarded group as opaque. Do not inspect its declarations,
   attributes, unsafe blocks, controls, temporaries, or descendant labels.
4. For a transformed expected label, retain its returned expansion group and
   apply the existing Phase 2 expansion-group rules. When the expected
   statement has a control root, identify exactly one returned statement in
   the group as the existing Phase 2 structural anchor: it has the expected
   direct statement role and control kind and owns all matched descendant
   statement lists. Zero or multiple anchors is an error. Same-label siblings
   before or after that anchor remain ordinary expansion statements. Recurse
   only through the unique anchor's reliably matched branch, arm, loop, block,
   or let-else roles so nested preserved groups can be restored.
5. Reject a missing, malformed, reordered, nonconsecutive, or structurally
   misplaced outer label when that label is required to locate a canonical
   replacement.
6. Run the existing signature, declaration, label, control, temporary,
   unsafe-block, and attribute validation against the resulting canonicalized
   function.

Because a preserved parent has no transformed descendant, replacing a
preserved parent restores its complete original label subtree. The LLM may
omit or alter descendant labels inside its discarded parent group. Conversely,
a preserved child beneath a transformed parent must remain locatable in the
same validated branch, arm, loop, block, or let-else role.

Canonicalization must happen before whole-body scans. A temporary declaration
invented inside discarded text therefore cannot satisfy a use in a transformed
group, and an unsafe block that exists only inside discarded text does not
cause a spurious validation error.

Put this canonicalization, unique-anchor selection, structural diagnostics,
and metadata validation in one shared Crat module used by both validator and
replacer. Do not implement two similar label-merging algorithms. The shared
operation returns either the canonicalized function plus its matched
structure or a deterministic structural failure; the validator converts that
failure to its ordinary item diagnostics, while the replacer fails atomically.

### A2.6 Preservation-aware replacement

The replacement request likewise remains schema version 1 and adopts its
final unreleased shape. Each requested item includes its existing ID, path,
and name plus the immutable annotated target skeleton,
`needs_transformation`, and `statements_requiring_transformation`.

The replacer independently canonicalizes every returned function with the
shared operation before composing an implementation. It must not trust that a
caller previously invoked the validator. It then performs the existing target
header composition, recursive label removal, wrapper generation, call-site
rewriting, executable migration, and atomic multi-item replacement using the
canonicalized body.

Thus every LLM expansion group for a preserved label is mechanically
discarded. Only a transformed label can contribute LLM-written statements to
the emitted project.

The immutable initial skeleton remains safe to restore after earlier SCCs have
been promoted because the preservation proof rejects every call or callable
reference whose local target signature changes. A preserved call to an
unchanged-signature function never needs wrapper redirection.

### A2.7 SCC orchestration and no-LLM transactions

The scheduling unit remains one SCC:

```text
SCC needs an LLM request
    iff any member record has needs_transformation == true.
```

For an all-preserved SCC:

1. Do not construct dependency context or render a prompt.
2. Do not make an LLM request.
3. Do not invoke the structural validator.
4. Form the complete transformation from the member
   `annotated_skeleton` functions in ascending item-ID order.
5. Pass that transformation and the version-1 item metadata directly to the
   item replacer.
6. Install, build, and promote or fail using the ordinary SCC source
   transaction.

Replacement is still necessary when a function signature changed but its body
did not require an LLM. The replacer may introduce a compatibility wrapper or
perform the fixed `main_0` migration.

For a mixed SCC, make the existing one LLM request for the complete SCC,
including fully preserved member functions. Validation and replacement
mechanically restore their preserved statements. Do not split an SCC or
introduce wrappers between its members merely to avoid presenting a preserved
member to the LLM.

Keep the no-LLM branch direct and avoid trivially unnecessary per-request
work. This amendment does not require broader refactoring solely to make every
provider-adjacent resource lazy. Regardless of process-level initialization
details, an entirely mechanical run makes zero LLM requests and reports:

```text
llm_generation_calls = 0
repair_calls = 0
```

`function_count` and `scc_count` still count every function and SCC.
`cargo_builds` still includes the normalized initial build and each attempted
mechanical or LLM SCC candidate build. A mechanical replacement or build
failure is a fatal integration failure; it is not an LLM repair opportunity.

Keep the existing within-SCC final-name uniqueness check for mechanical and
LLM SCCs because the current complete-snippet replacement protocol identifies
returned functions by final name.

### A2.8 Prompt amendment

The unreleased prompt remains `local_transformation`, version 1. Amendment 2
defines its final version-1 text before the first release. Preserve the
complete body specified in Section 10 byte-for-byte except for inserting this
exact paragraph after the paragraph ending “Use Dependency Context only as
reference; do not emit or redefine its functions, types, statics, or
constants.”:

```text
Complete every generated `todo!()` hole. Preserve every complete labeled statement already present in the Target Skeleton exactly as provided.
```

The validator deliberately accepts different returned contents for a
preserved group because those contents are discarded. The prompt asks the LLM
to reproduce complete statements to reduce noise and improve readability; it
is not the enforcement mechanism.

All prompt loads and request metadata continue to use version 1. The
historical Section 10 text remains unchanged in this document; this amendment
is the normative final text for the implementation's version-1 prompt.

### A2.9 Required regression scope

Implement every case in `amendment-2-test-plan.md`. Update existing Rust and
Python tests whose expected skeletons, JSON keys, request schemas, prompt
version, or SCC event traces are superseded, but do not edit
`phase-1-test-plan.md`, `phase-2-test-plan.md`, `phase-3-test-plan.md`, or
`phase-4-test-plan.md`.

The regression suite must cover at least:

- pointer-free scalar and aggregate statements;
- lvalue, rvalue, pattern, declaration, adjustment, and callable types;
- declarations without initializers;
- nested local ADTs, enums, unions, aliases, generic fields, and recursive
  type cycles;
- the future field-type-change hook;
- foreign calls, unsafe non-local calls, safe non-local calls, local unsafe
  calls, changed local signatures, indirect calls, callable references,
  macros, closures, inline assembly, and ambiguous desugarings;
- fully preserved and mixed nested control trees;
- Amendment Plan 1 interaction;
- preservation-aware validation and replacement, including deliberately
  changed discarded statements;
- all-preserved, mixed, and ordinary transforming SCCs;
- a signature-changing function with an entirely preserved body and required
  wrapper; and
- zero-LLM-call reporting, metrics, build failure, and deterministic ordering.

## Amendment Plan 3: Foreign-function guidance and prompt presentation

This amendment gives the LLM explicit, per-function notice of resolved foreign
function references and removes skeleton-internal presentation details from
ordinary prompt context. It applies after completed Phases 1--4 and Amendment
Plans 1 and 2. It supersedes only the conflicting foreign-function metadata,
function-record key order, skeleton-loading, context-rendering, prompt-
presentation, and prompt-text requirements in this document. All statement-
preservation, SCC scheduling, validation, replacement, repair, transaction,
and compilation behavior remains unchanged.

`amendment-3-test-plan.md` is the exhaustive executable contract for this
amendment. Do not edit any historical phase or amendment test-plan file.

### A3.1 Resolved foreign-function references

For every source function emitted as a `Fn` skeleton record, collect the Rust
declaration identifiers of all foreign functions referenced directly by that
function's compiled body.

Collection is semantic, not textual:

- resolve every function path through rustc;
- include a resolved function when `TyCtxt::is_foreign_item` identifies its
  `DefId` as a foreign item;
- collect the resolved foreign declaration's Rust identifier, even when the
  source reference uses a `use` alias;
- use the Rust declaration identifier rather than an external symbol named by
  `#[link_name]`;
- include a resolved callable reference even when it is not the callee of a
  direct call;
- include references to foreign functions defined through dependency-crate
  metadata as well as foreign declarations in the current crate; and
- traverse the function body only, not the bodies of local callees.

Deduplicate identifiers by their exact string and serialize them in ascending
lexicographic order. Two distinct foreign declarations with the same Rust
identifier therefore produce one name. Repeated calls or references likewise
produce one name.

Do not include:

- foreign statics or foreign types;
- non-foreign functions from `core`, `std`, `alloc`, compiler support crates,
  or third-party crates;
- a source-defined function merely because its header has an explicit ABI such
  as `extern "C"`; or
- a foreign function that is declared but never referenced by the function.

This collection uses the same resolved compiling program that already owns
dependency analysis. Do not infer foreign functions from identifier spellings,
maintain a libc allowlist, parse rendered Rust in Python, or derive the list
from Amendment 2 statement dispositions.

The existing ordinary dependency behavior is unchanged. A foreign reference
does not create an item record, dependency ID, function-graph edge, SCC edge,
or dependency-context declaration. The new name list is guidance attached
only to the referring function.

### A3.2 Function-record JSON

Add this required field to every `Fn` skeleton record:

```json
{
  "foreign_function_names": ["free", "strlen"]
}
```

Emit `foreign_function_names: []` when the function has no resolved foreign
function reference. Do not emit this field for `Static`, `Const`, `TyAlias`,
`Enum`, `Struct`, or `Union` records.

The final function-record serialization key order after Amendment 2 is:

```text
id, path, kind, name, annotated_source, annotated_skeleton,
source_signature, target_signature, needs_transformation,
statements_requiring_transformation, foreign_function_names,
signature_dependencies, dependencies
```

The top-level skeleton JSON remains an unversioned internal artifact produced
and consumed within one local-transformation stage invocation. This additive
pre-release change does not alter a validator or replacement protocol version.

The foreign-name list is independent of Amendment 2 metadata. In particular,
do not enforce an additional Boolean/list invariant between
`foreign_function_names`, `needs_transformation`, and
`statements_requiring_transformation`. Amendment 2 continues to classify a
foreign call or callable reference using its own conservative rules.

### A3.3 Python loading

The Python skeleton model stores `foreign_function_names` as an immutable
sequence of strings. Require the field on every `Fn` record and require:

- a JSON array;
- only nonempty strings;
- no duplicates; and
- strict ascending lexicographic order.

Reject malformed producer output rather than sorting or deduplicating it in
Python. Preserve every name exactly as decoded. Existing item-ID, path,
dependency, statement-disposition, and Rust-text validation remains unchanged.

The field is presentation metadata only. Do not copy it into structural-
validation or item-replacement requests, and do not use it in function-graph,
SCC, or dependency-closure construction.

### A3.4 Prompt item headings and target rendering

Skeleton item IDs remain internal to JSON loading, graph construction,
deterministic ordering, tool requests, usage metadata, and diagnostics. Do not
put an item ID in an ordinary Dependency Context or Transformation Targets
heading.

This does not remove numeric `#[proctor(N)]` statement labels. Those labels are
part of the structural transformation protocol and remain in source,
skeleton, LLM output, validation, and replacement exactly as required by the
completed implementation.

Render every transformation target in ascending member-item-ID order, but
identify it to the LLM only by its final function name:

````text
### Function `<name>`

Foreign function references: `<first>`, `<second>`

Source:
```rust
<annotated_source>
```
Target skeleton:
```rust
<annotated_skeleton>
```
````

The foreign-function line uses the record's existing sorted order and wraps
each identifier in one Markdown code span. Omit the complete line, including
its following blank line, when `foreign_function_names` is empty.

Within-SCC final function names are already required to be unique before an
LLM request, so a target heading never needs a path fallback. Keep the full
record path internally for diagnostics and replacement, but do not render it
in a target heading.

### A3.5 Dependency entry names

Ordinary Dependency Context entries also omit item IDs. Their headings use
the record kind and the final component of the record's crate-relative path:

````text
### Function `<name>`

Source signature:
```rust
<source_signature>
```
Target signature:
```rust
<target_signature>
```
````

````text
### <Static-or-Const> `<name>`

```rust
<declaration>
```
````

````text
### <TyAlias-or-Enum-or-Struct-or-Union> `<name>`

```rust
<definition>
```
````

Always use only that final name. Never put an item ID or any preceding path
component in an ordinary dependency heading. If two dependency records have
the same kind and final name, deliberately render duplicate headings. For
example, records with paths `outer::left::parse` and
`outer::right::parse` both render as:

```text
### Function `parse`
```

Do not add a path-disambiguation algorithm in this prototype. The original
source and the rendered signatures or definitions remain the available
context. If duplicate name-only headings cause a concrete ambiguity in the
future, revise both the prompt context and the corresponding source
presentation together rather than adding paths only to headings.

Continue to order entries internally by item ID and join them with exactly two
newlines. Apply the existing character budget directly to these final
name-only entry strings. The visible omission of IDs and paths does not alter
dependency closure or deterministic ordering.

### A3.6 Prompt sections and exact advisory instruction

The unreleased prompt remains `local_transformation`, version 1, with the same
frontmatter variables and description. Amendment 2's exact complete-statement
paragraph remains in place.

Add this exact requirement after the completed requirement about preserving a
control root and before the prohibition on explicit unsafe blocks:

```text
10. For each listed foreign-function reference, prefer a behavior-equivalent
    safe Rust function or method when one is available; otherwise preserve the
    foreign call.
```

Renumber the existing unsafe-block requirement to 11 and the fenced-output
requirement to 12. Do not add a libc allowlist, mandate replacement, or claim
that every foreign function has a safe equivalent.

Replace the plain `Dependency Context:` and `Transformation Targets:` labels at
the end of the template with Markdown section headings. The exact final
template fragment is:

```jinja
{% if dependency_context %}## Dependency Context

{{ dependency_context }}

{% endif %}## Transformation Targets

{{ transformation_targets }}

{{ repair_context -}}
```

When `dependency_context` is the exact empty string, omit the complete
Dependency Context section. Transformation Targets is always present for an
LLM-backed SCC. The `##` section headings own the `###` item headings described
above. The section headings are outside the 100,000-character dependency-entry
budget.

The prompt still receives the complete original target material on every
repair. Keep Amendment 2's prompt instruction and the existing raw structural
validator and compiler diagnostic behavior unchanged. In particular, this
amendment removes item IDs from authored context headings; it does not sanitize
or reserialize raw validator response text used by a repair.

### A3.7 Advisory behavior and non-enforcement

The foreign-name list is guidance, not a new correctness condition. The LLM
may keep any foreign reference when no behavior-equivalent safe Rust operation
is available. Structural validation, canonical statement restoration, item
replacement, and compilation do not require a listed foreign function to
disappear and do not reject a new or retained foreign call solely because it
is foreign.

This is intentional. Foreign APIs can differ from superficially similar Rust
APIs in allocation ownership, errno behavior, locale behavior, overlap rules,
null-pointer preconditions, integer conversions, or other semantics. The
existing exact-behavior requirement remains stronger than the advisory
preference for safe Rust.

### A3.8 Required regression scope

Implement every case in `amendment-3-test-plan.md`. Update existing Rust and
Python implementation tests whose expected function-record fields, exact JSON,
record helpers, context entries, budgets, target headings, prompt text, or
content hashes are superseded. Do not edit `phase-1-test-plan.md`,
`phase-2-test-plan.md`, `phase-3-test-plan.md`, `phase-4-test-plan.md`, or
`amendment-2-test-plan.md`.

The focused regression suite must cover:

- per-function foreign-name collection, deduplication, and sorting;
- resolved aliases, `#[link_name]`, callable references, and foreign
  declarations loaded from dependency metadata;
- exclusion of foreign statics, foreign types, non-foreign external functions,
  source-defined ABI functions, and unused declarations;
- exact final function-record shape and field ownership;
- strict Python loading of the new string list;
- target rendering with names, no item IDs or paths, and an omitted empty
  foreign-reference line;
- dependency rendering with final names only, including deliberate duplicate
  headings when kinds and names coincide;
- character budgeting over final name-only entry rendering;
- exact `##`/`###` prompt hierarchy and omission of an empty Dependency
  Context section;
- preservation of Amendment 2's prompt paragraph, version 1 metadata, repair
  diagnostics, and statement labels; and
- advisory-only behavior with no validator or replacer protocol change.

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
