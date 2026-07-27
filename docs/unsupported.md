# Unsupported Input for the Local Transformation Prototype

## 1. Purpose and status

This document consolidates the current conceptual unsupported-input contract
for PROCTOR's local type-directed transformation prototype. It is derived from
[prototype-desc.md](prototype-desc.md), the focused implementation tests, and
the detailed historical plans linked from
[prototype-plan.md](prototype-plan.md).

“Unsupported” means that the local transformation is not intended to receive a
project containing or requiring the listed feature. It does **not** mean that
the current skeleton generator, validator, compiler driver, or orchestrator
necessarily detects and rejects it. A component may parse, preserve, ignore,
or even successfully process an unsupported construct. Some component tests
deliberately exercise such constructs to keep easy low-level behavior robust.

No general supportedness checker or unsupported-feature normalization pass is
implemented. The stage's one-time safety-qualifier normalization is a separate
mechanical integration step and does not enforce this contract. A future tool
may check this contract or rewrite selected unsupported constructs into
supported forms.

This list is intended to be exhaustive for the supportedness decisions
currently recorded by the cited plans and discussions. A future feature is not
implicitly supported merely because it is absent here; it must be classified
deliberately when its design is added.

This is an input contract. It is not a duplicate list of invalid LLM responses
or validation-protocol errors. For example, malformed `#[proctor(...)]` labels,
missing returned functions, wrongly named generated temporaries, and reordered
expansion groups are invalid transformation outputs, not source-project
features listed here.

Unless an exception is stated below, a project is conceptually unsupported if
any source-defined part of the local-transformation input contains one of the
listed features. Item restrictions apply recursively through every supported
inline module.

## 2. Required pipeline and project form

The following project forms are unsupported:

- A project that does not already pass `cargo build`.
- A project that is not Rust 2021.
- A project whose manifest does not declare `[lib].path` as one relative,
  root-level regular source file. Absolute paths, paths containing `..`, and
  nested library paths are rejected.
- A project with multiple logical Proctor targets, multiple transformed
  library crates, multiple build configurations, or a target kind other than
  an executable or `cdylib` library. The library-plus-generated-bin layout
  below is one logical executable target, not a violation of this rule.
- A project that cannot be built with the ordinary `cargo build` contract.
- A non-Linux target for the initial pipeline contract.
- A library exporting global variables. The initial library contract supports
  exported functions only.
- A project requiring source selection through `cfg` or another build
  configuration that changes the item tree seen by the transformation.

The stage copies the complete input project and itself runs ordinary Crat
`expand` followed by `unexpand`. External module declarations such as
`mod foo;` and their files may therefore be present in the stage input.
Preparation must consolidate the transformed library into the declared
root-level library file; skeleton generation then recursively traverses the
resulting inline modules. The stage does not rerun Crat's `split` or `bin`
passes.

An executable is supported only in Crat's C2Rust layout: the Cargo project has
the same one transformed library source plus one generated bin source that
only calls the library's safe `main`. The bin source is not transformation
input. An executable-only Cargo target without that library source, an
additional bin body requiring transformation, or another executable layout is
unsupported.

The prototype copies but does not interpret or update `proctor.toml`. Merely
having that file does not trigger rejection. An input is unsupported when it
contains pre-existing compatibility wrappers whose identity must affect
target selection or rewriting, because the stage cannot consume their
metadata. Conceptually, functions known to be wrappers from non-local
transformation are not local-transformation targets.

## 3. Unsupported item and function forms

### 3.1 Unsupported function forms

Source-defined safe and unsafe free functions are both transformation targets.
Skeleton generation preserves source safety in source renderings and makes
every target skeleton unsafe. Before incremental replacement, safety
normalization mechanically makes every source-defined free function except
`main` unsafe. It traverses the prepared library source directly and does not
consume skeleton JSON or inspect function signatures or bodies. The following
function forms remain unsupported:

- `const fn`.
- `async fn`.
- Source-defined variadic functions.
- Methods of any kind, including inherent methods, trait methods, provided
  trait methods, and associated functions defined in an `impl`.
- Functions with a non-identifier parameter pattern, including wildcard,
  tuple, struct, tuple-struct, slice, `@`, or other destructuring parameters.
- A function carrying both `#[no_mangle]` and `#[export_name = "..."]`.

An explicit source ABI such as `extern "C"`, visibility, an export attribute,
or `#[no_mangle]` is not unsupported by itself. The simultaneous
`no_mangle`/`export_name` combination above is the exception. Skeleton
generation sanitizes explicit ABI and `no_mangle` syntax only in presentation
text and preserves the real project metadata for later wrapper handling.

Foreign function declarations and calls, including libc calls, are context
rather than transformation targets. Their presence is not unsupported merely
because they are foreign. Variadic foreign declarations such as `printf` are
allowed; only body-bearing source-defined variadic free functions are
unsupported. Foreign items do not become skeleton records or ordinary
dependency entries, so a project is unsupported if successful local
translation would require transforming a foreign declaration or treating it
as an ordinary local function target. Resolved foreign-function declaration
names are recorded as advisory prompt metadata for referring functions, but
they do not become graph dependencies.

### 3.2 Executable `main` and `main_0`

Skeleton generation unconditionally omits every free function whose final
identifier's symbol is `main`, including the surface spelling `r#main`,
without inspecting its body. Omission is not a supportedness decision. A
supported executable library contains exactly one co-located library-module
pair of:

- `unsafe fn main_0() -> core::ffi::c_int` and C2Rust's safe `pub fn main()`
  wrapper that calls it inside an unsafe block and exits; or
- `unsafe fn main_0(core::ffi::c_int,
  *mut *mut core::ffi::c_char) -> core::ffi::c_int` and C2Rust's safe
  `pub fn main()` wrapper that constructs `argc`/`argv`, calls it inside an
  unsafe block, and exits.

The following are unsupported:

- Any other source-defined free function named `main`.
- Any other `main_0` signature paired with executable `main`.
- Multiple executable `main`/`main_0` pairs.
- A two-argument `main_0` whose behavior is not compatible with representing
  `argv` as `&mut [&mut [i8]]`, NUL-terminated argument buffers, and one final
  empty-slice sentinel corresponding to `argv[argc] == NULL`.
- Another untransformed caller of the two-argument `main_0` that would also
  require migration when its forced signature is inserted.

The two supported safe `main` wrappers are mechanically managed compatibility
boundaries, not transformation targets. Their explicit unsafe blocks are
allowed as the sole exception to the general explicit-unsafe-block
restriction. For the two-argument form, item replacement uses the fixed
implementation summarized in the
[prototype description](prototype-desc.md#replacement-and-compatibility); the
generated Cargo bin shim remains unchanged.

Skeleton generation and item replacement recognize these cases using only
final identifier symbols and `main_0` arity. A zero-argument `main_0` leaves
its co-located sibling `main` unchanged; a two-argument `main_0` receives the
forced target `argv` type and causes that sibling `main` to be replaced
mechanically. Recognition does not inspect safety, parameter names or types,
return type, visibility, ABI, attributes, or either body. The exact forms
above remain a supported-input assumption rather than a recognition check.

### 3.3 Traits, impls, and dispatch

The following are unsupported:

- Explicit trait definitions.
- Trait methods and associated trait items.
- Every explicit `impl`, whether inherent or trait-based.
- Associated types and associated constants that require trait/impl support.
- Trait objects and dynamic dispatch, including `dyn Trait`.
- Rust delegation items or other delegation syntax.

Compiler-generated impls from an ordinary preserved `#[derive(...)]` are a
special case described in Section 3.7.

### 3.4 Source-written generic declarations

Source-written generics are unsupported, including:

- Type parameters.
- Const parameters.
- Explicit lifetime parameters.
- Generic functions, structs, enums, unions, type aliases, traits, or impls.
- Generic bounds, lifetime bounds, and `where` clauses.
- Other declarations whose correct handling requires comparing or rewriting a
  source generic parameter list.

Concrete uses of already-defined generic types, such as `Option<i32>`,
`Box<MyType>`, slices, and standard collections, are not unsupported merely
because their type constructor is generic.

Skeleton generation may introduce named lifetime parameters in a **target**
signature when pointer analysis selects related borrowed types. Those
generated target lifetimes do not make source-written generics supported.

### 3.5 Closures and callable values

The following are unsupported in the source project:

- Closures.
- Function-pointer types, values, parameters, returns, fields, statics, and
  indirect calls.
- Callbacks represented directly as function pointers.
- Callback encodings passed through `void *`.

Structural validation understands closure-parameter binding names in a
returned snippet so it can enforce generated-temporary naming consistently.
That low-level validator behavior does not make source closures supported.

### 3.6 Function-local items

Every parsed function-local item statement (`StmtKind::Item`) is unsupported,
recursively at every statement-list depth. This includes local `const`,
`static` (including `static mut`), functions, type aliases, structs, enums,
unions, modules, traits, impls, foreign blocks, `use`, `extern crate`, macro
definitions, and item-position macro invocations.

Skeleton generation rejects a transformation target as soon as recursive
statement labeling encounters such an item and does not label or enter the
item's initializer or body. Structural validation rejects every function-local
item in both an expected skeleton and a returned transformation. Expression-
and statement-position macro invocations are expressions/statements rather
than item declarations and remain governed by Section 3.7.

### 3.7 Macros, conditional compilation, and generated items

The following are unsupported:

- Source macro definitions.
- Item-producing macro invocations.
- Item-position procedural or declarative macro use that changes the item tree
  expected by surface-AST-to-HIR mapping.
- `cfg` and `cfg_attr` alternatives.
- Derive or attribute expansion that emits additional helper items not marked
  by rustc as automatically derived.

Ordinary expression- or statement-position macro invocations are supported
only when their source token input contains no call expression. C2Rust macros
are expected to have been expanded before this stage, while `unexpand` may
restore selected expression macros. Skeleton generation treats a standalone
macro statement as a labeled expression hole. For example, `println!("hello")`
is supported, while `println!("{}", local_fn())` is unsupported. Calls
introduced internally by macro expansion do not violate this source-token
restriction. Expanded-HIR dependency collection may still observe a source
call nested in a macro input, but item replacement cannot reliably redirect
that call in the unexpanded surface AST.

Ordinary `derive`, `repr`, and other supported item attributes are preserved.
A derive is supported only under the assumption that every generated HIR item
is recognized by `TyCtxt::is_automatically_derived`, allowing skeleton
generation to skip it without disturbing surface/HIR alignment.

## 4. Unsupported bindings and patterns

The following source binding forms are unsupported:

- By-reference binding modes written with `ref` or `ref mut`.
- Any parameter pattern other than one identifier, as stated in Section 3.1.

This does not mean that the low-level components are unable to represent
`ref`. Skeleton generation preserves `ref` and `ref mut` exactly while making
non-`ref` bindings mutable in prompt-facing presentations. Validation
preserves by-value-versus-`ref` mode and the referenced binding identities
while ignoring ordinary `mut`. This component behavior does not change the
prototype-level restriction because pointer analysis does not support source
`ref` bindings.

Outside parameters and `ref` binding mode, existing local-pattern structure is
supported and preserved. This includes simple and destructuring `let`,
`let-else`, `if let`, `while let`, `for`, match-arm, tuple, tuple-struct,
struct-field, slice, `@`, `or`, reference-pattern, and parenthesized-pattern
positions. Wildcards introduce no binding.

## 5. Unsupported statements, attributes, and control shapes

The following are unsupported anywhere in a transformable source function,
including nested blocks:

- Empty statements (`;` with no statement node to label).
- A `match` arm whose body is not a block expression.
- Source statement attributes.
- Source expression attributes.
- Pre-existing `#[proctor(...)]` statement labels. These are generated by
  skeleton generation, not supplied by the source project.
- Explicit `unsafe { ... }` blocks.

Unsafe operations themselves are supported directly inside an `unsafe fn`.
The unsupported feature is an explicit nested unsafe block in a transformation
target. The two supported, excluded C2Rust `main` wrappers are the sole
source-input exception, and the fixed replacement `main` also contains a
trusted unsafe block.

Attributes on a function-local item do not create an exception: the enclosing
item statement is unsupported independently of its attributes.

### 5.1 Control expressions beneath non-control wrappers

A control or plain-block expression is normally supported only when it is the
root of one of these structural roles:

- an expression or semicolon statement;
- a direct `let` initializer;
- the direct value of `return`;
- the direct value of `break`; or
- a supported match-arm block tail.

The one exception is an ordinary `if` used like a C conditional expression. It
may occur beneath a non-control wrapper when all of the following hold:

- it is an ordinary `if`, not `if let`;
- it has an `else`;
- its condition contains neither control expressions nor a `let` expression;
- each branch contains exactly one tail expression and no other statements;
  and
- any control expression in a branch tail is recursively another conditional
  of this same restricted form.

This restricted conditional is opaque: its interior receives no statement
labels and remains part of the enclosing statement region.

Except for that form, an input is unsupported when an `if`, `if let`, `while`,
`while let`, `for`, `loop`, `match`, or plain block that needs structural
preservation occurs beneath a non-control expression wrapper. Unsupported
intervening wrappers include:

- calls or method calls;
- constructors;
- unary or binary operators;
- tuples;
- casts;
- assignments and compound assignments;
- indexing, field access, or other non-control expression structure that
  prevents the control expression from being the direct structural root.

Control structures may otherwise nest arbitrarily through their own branch,
arm, loop, plain-block, and `let-else` bodies. Recursive `else if` shape,
match-arm count/order, and guard presence are supported structural features.

## 6. Unsupported pointer and non-local transformation cases

The local transformation assumes that abstraction recovery and discipline
repair have already removed cases that need non-local redesign. Remaining
instances of the following are unsupported:

- Actual negative pointer offsets. Tools mode deliberately skips offset-sign
  analysis and assumes the preceding stage has eliminated them.
- Pointer behavior that requires preserving a separate base pointer and signed
  cursor/offset state.
- Aliased pointers without a unique owner when safe local types require
  ownership or borrow restructuring.
- Simultaneously active mutable aliases.
- Pointer conversions whose correct representation requires non-local state,
  statement reordering, parameter-count changes, or coordinated signature
  changes beyond the fixed pointer-analysis skeleton.
- A C data-structure implementation that still requires non-local abstraction
  recovery, such as a dynamic array, linked list, hash table, intrusive or
  self-referential structure, memory pool, or arena that must be replaced as a
  unit.
- A required `SliceCursor` target representation. Prototype tools mode forces
  nonnegative array-like decisions to ordinary slices and never produces
  `SliceCursor`.

### 6.1 Named types

Pointer-containing named-type transformation is unsupported. The initial
prototype should not receive pointer-containing structs, unions, enums, or
type aliases whose representation must change. Named type definitions are
dependency context and remain unchanged.

Structs, enums, unions, and type aliases that can remain unchanged are
supported as dependency records. Their fields, variants, and constructors do
not receive separate transformation IDs.

### 6.2 Globals

Global-variable transformation is unsupported. Global `static` and `const`
types remain unchanged, and the prototype does not generate global-variable
wrappers. A project is unsupported if successful translation requires
changing a global's type, representation, initialization strategy, exposure,
or access discipline.

Ordinary globals that can remain unchanged are supported as dependency
context. This includes scalar constants/statics and other globals whose
existing representation remains valid.

### 6.3 API wrapper conversion

For any function whose parameter or return types change, the prototype
supports compatibility wrappers only for the following conversions.

From a raw-pointer source parameter to a target parameter:

- references and optional references;
- slices, using the provisional null-to-empty behavior and a fixed
  1,000,000-element bound;
- scalar boxes and optional scalar boxes; and
- raw pointers.

Target parameters of type `Option<&[T]>`, `Option<&mut [T]>`, `Box<[T]>`, or
`Option<Box<[T]>>` are unsupported. Nullable target slices and boxed-slice
input conversion are deferred.

From a target return to a raw-pointer source return:

- references and optional references;
- slices, using the provisional empty-to-null behavior;
- scalar boxes and optional scalar boxes;
- boxed slices and optional boxed slices; and
- raw pointers.

Structurally identical nonpointer parameters and returns pass through. Any
other signature change, including a custom-type conversion, is unsupported.
Exported-global wrappers are unsupported.

### 6.4 Synthesized type spelling

An input is unsupported when a pointer-analysis decision requires generated
type syntax that cannot be named from the target function's containing
module. Skeleton generation fails atomically rather than emitting invalid
Rust.

In particular, introduced bare `Option` and `Box` constructors require the
ordinary prelude and must resolve to the standard language items. Shadowing,
`#![no_implicit_prelude]`, an inaccessible type, or another unavailable path
is unsupported when it prevents a required target type from being spelled.
Existing source type syntax is reused and shortened by resolved identity where
possible, so these features are not independently unsupported when no
unspellable type must be synthesized.

## 7. Unsupported graph, naming, and size cases

The following orchestration cases are unsupported:

- An SCC containing two transformation targets with the same final Rust
  identifier, even when their full module paths differ. Validation requests
  match functions by final name and require names to be unique within the
  request.
- An SCC whose mandatory context—recursive SCC-member signatures plus all
  direct dependencies—exceeds 100,000 rendered characters. The orchestrator
  aborts rather than omitting a mandatory entry.
- A case that requires modifying functions outside the current SCC to make one
  replacement attempt structurally valid before the normal wrapper/call-site
  migration can occur, other than the specified mechanical replacement of
  executable `main`. The one-time safety normalization happens before
  SCC processing and is not an SCC repair.

Direct recursion and mutually recursive SCCs are not unsupported. They are
explicitly part of the implemented SCC and item-replacement behavior.

The 100,000-character limit applies to rendered dependency context, not to
transformation targets, instructions, diagnostics, or prior failed code.
Deeper transitive type-dependency levels may be omitted only by the documented
breadth-first whole-depth budget policy; mandatory context entries may not be
silently omitted.

## 8. Context items that are not transformation targets

The following constructs may occur in an otherwise supported input but are not
local transformation targets or independent skeleton records:

- Inline modules, which are traversed recursively.
- Top-level `extern crate` and `use` items.
- Foreign modules, foreign functions, foreign statics, and foreign types.
- Every free function named `main`, which skeleton generation omits
  unconditionally; only the two forms in Section 3.2 are supported input.
- Top-level `GlobalAsm`.
- Struct fields, enum variants, and constructors; their containing named type
  is the dependency identity.
- Wrappers generated by the local transformation itself during incremental
  SCC migration.

References to excluded context items may be omitted from dependency-context
generation. Their presence does not make the input unsupported unless another
section above applies.

## 9. Component behavior that does not change this input contract

The following component-level behavior is intentional. It either implements a
supported case or keeps a low-level component robust, but does not by itself
widen the supported input language:

- Skeleton generation explicitly diagnoses empty statements, non-block match
  arms, function-local items, and unsupported control expressions beneath
  non-control wrappers.
- Skeleton generation handles `ref` bindings mechanically.
- Skeleton generation emits unsafe target headers for safe source functions;
  safe source functions are supported, but `const fn` remains unsupported.
- Skeleton generation omits every free function named `main` without
  validating that its body is one of the two supported executable forms.
- Skeleton generation forces the supported two-argument `main_0` `argv` target
  type instead of following its ordinary pointer-analysis decision,
  recognizing the case only by the `main_0` identifier symbol and arity two.
- Structural validation checks binding identity, structural position, and
  by-value-versus-`ref` mode for `ref` patterns.
- Structural validation checks generated bindings in closure parameters even
  though source closures are unsupported.
- The parser-only validator may accept Rust that would later fail name
  resolution or type checking; successful parsing is not a supportedness
  decision.
- Item filtering may silently omit unsupported or context-only item kinds
  rather than rejecting the project.

These behaviors are useful for robust components and actionable diagnostics.
They must not be interpreted as approval to include the corresponding feature
in a local-transformation input project.

## 10. Deferred enforcement

The current work does not add a general supportedness checker or normalization
for the unsupported constructs below. Future work may:

- implement a dedicated supportedness checker;
- report every unsupported construct with source locations;
- hoist unsupported control expressions out of non-control wrappers;
- eliminate or normalize `ref` bindings;
- normalize non-block match arms and empty statements;
- split or rewrite function-local items;
- run abstraction recovery or discipline repair before local transformation;
  or
- expand the supported input contract deliberately, with corresponding plans
  and tests.

Until then, upstream preparation and evaluation-fixture selection are
responsible for satisfying this document.
