# Shared Historical Prototype Plan Material

This file preserves historical plan material that applies across multiple
implementation task sets. For current implemented behavior, start with
[prototype-desc.md](prototype-desc.md). For the historical reading order, see
[prototype-plan.md](prototype-plan.md).

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
amendment in
[Phase 3's validator corrections](phase-3-plan.md#validator-and-replacement-integration)
and the corresponding update cases in
`phase-3-test-plan.md`, is the exhaustive executable contract for Phase 2.
`phase-3-test-plan.md` is the exhaustive executable contract for Phase 3.
`unsupported.md` is the consolidated conceptual input contract for this
prototype. Some low-level skeleton and validator behavior deliberately handles
constructs listed there (notably `ref` bindings), but that robustness does not
make those constructs supported local-transformation inputs. Phase 2 adds no
supportedness checker or normalization pass.
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
[Section 5.2](phase-1-plan.md#52-function-skeletons);
the input project's real header is unchanged.

### Target skeleton

The function skeleton produced from Crat's pointer-analysis result.

### Target signature

The signature selected by Crat's analysis and present in the target skeleton,
using the same presentation-only ABI/export sanitization as the source
signature. It is always unsafe, and the supported two-argument `main_0`
receives the
[target-signature `argv` override](phase-3-plan.md#executable-recognition-and-target-signature)
after ordinary pointer analysis.

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
