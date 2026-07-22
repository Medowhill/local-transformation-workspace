# Phase 1 Test Plan: Skeleton JSON Generation

## 1. Purpose

This document specifies the complete automated test suite planned for Phase 1
of the local-transformation prototype. Phase 1 implements the
`crat-tool make-skeleton` operation described in `prototype-plan.md`.

The tests cover:

- included-item discovery and deterministic IDs;
- dependency collection and canonicalization;
- statement annotations;
- source-function and signature rendering;
- target skeleton construction;
- explicit local-variable types;
- reuse of Crat's initial pointer decisions without transformation-time
  demotion;
- omission of offset-sign analysis and `SliceCursor`; and
- JSON serialization and whole-output invariants.

The cases below are the complete set of Phase 1 tests planned for the initial
implementation. Later bug fixes may add regression tests, but Phase 1 is not
complete until every case in this document is implemented and passing.

The initial suite contains 80 named cases. Case identifiers are intentionally
stable; gaps correspond to cases removed after the input contract was
clarified and are not tests to implement.

| Area | Cases |
| --- | ---: |
| JSON contract | 6 |
| Item discovery/rendering | 11 |
| Dependencies | 22 |
| Statement labels | 7 |
| Skeleton shape | 14 |
| Target types/pointer analysis | 16 |
| Whole-output/errors | 4 |

## 2. Test execution and filesystem policy

All tests are ordinary Rust tests executed by:

```bash
cd crat
cargo test --workspace
```

Tests that need rustc analysis pass source text directly to
`utils::compilation::run_compiler_on_str`. Parser-only tests use Crat's
in-memory AST parsing helpers. Serialization tests write JSON into a `String`
or `Vec<u8>`. Every test crate is exactly one source string; inline modules are
allowed, but no external source file or checked-in project fixture is opened.

The tests must not:

- create temporary directories or files;
- invoke `crat-tool` as a subprocess;
- test Clap argument parsing or other CLI wiring;
- call the production routine that writes `skeletons.json`;
- construct a Cargo project fixture at test time;
- invoke `cargo` recursively;
- modify environment variables;
- use snapshot files; or
- modify any source file, manifest, lockfile, or generated artifact.

The normal build artifacts created by the outer `cargo test` command are not
test-case filesystem effects. Each test body is filesystem-write-free. Compiler
setup may read the sysroot and Crat's dependency directory.

The production CLI's filesystem wiring is intentionally kept thin and is not
tested in Phase 1. Writing the final `skeletons.json` path is also not tested.
All functional coverage targets the in-memory skeleton-generation API.

## 3. Test organization and comparison policy

Tests will live beside the implementation:

- all 80 named Phase 1 tests, including P1-TYPE-17, under
  `crates/tools/src/` in `#[cfg(test)]` modules.

The tools crate may call public pointer-replacer analysis APIs in its tests.
Do not make `pointer_replacer` dev-depend on `tools`: `tools` already depends on
`pointer_replacer`, so doing so would create a Cargo dependency cycle. Existing
pointer-replacer tests continue to cover the normal transformation behavior.

Shared in-memory helpers may:

1. accept one Rust source string;
2. run the Phase 1 generator through `run_compiler_on_str`;
3. return the owned skeleton records or serialized JSON; and
4. locate records by path for assertions.

Tests should compare structured records and parsed AST whenever whitespace is
irrelevant. Exact string comparison is reserved for one deliberately small JSON
golden case and formatting properties that form part of the interface. Expected
dependency lists are numeric. Where a whole-fixture table also prints symbolic
names for readability, the IDs in that same table define their exact numeric
serialization.

Every case that invokes rustc now provides a complete Rust source block, or
names another case whose complete block is its exact input. Such a reference
means byte-for-byte reuse, not “a similar shape.” Expected Rust snippets are
normalized AST/pretty-print forms: formatting whitespace may be ignored unless
the case explicitly says byte-exact, but item kind, attributes, identifiers,
types, holes, labels, and ordering must match exactly. Numeric dependency arrays
are the concrete serialized output; symbolic names alongside them are only
explanatory.

Some skeleton-shape cases print a normalized snippet without `#[proctor]`
attributes so the control tree is readable. Those cases compare the AST after
temporarily stripping only `proctor` attributes; whenever a label map follows,
the unstripped output must also match that map. The LABEL cases provide the
complete annotation oracle.

Unless a case says otherwise, input functions are source-defined free
`unsafe fn` items and the input source compiles successfully before Phase 1 is
run. Every input models the post-`expand`/post-`unexpand`, pre-`split` state:
one physical source file, possibly with inline modules, and no external
modules, traits, impls, macro definitions, item-producing macro invocations,
`cfg` attributes, statement attributes, or explicit `unsafe` blocks. Every
supported `match` arm body is a block expression. Empty statements and
non-block match arms occur only in their explicit rejection tests.

Generation parses the exact source string into a surface AST for JSON
rendering and must not render from `utils::ast::expanded_ast(tcx)`. HIR/MIR is
still used for name resolution, dependencies, inferred types, and pointer
analysis. Pretty-printing may normalize whitespace and comments, but it must
preserve surface constructs such as `#[derive(...)]`.

The generator maps that separately parsed surface AST directly to HIR with
`utils::ir::AstToHirMapper::map_crate_to_mod(..., false)`. The mapper assigns
fresh AST `NodeId`s and records item `NodeId` to `LocalDefId` and local-node
`NodeId` to `HirId` relationships; tests must not rely on path/source-order
matching or on `NodeId`/`Span` equality across parse sessions. Unexpanded mode
must be propagated recursively into inline modules. HIR dependency traversal
uses the mapped owner, while a mapped binding `HirId` supplies its inferred
type and pointer decision and may be chained through
`map_thir_to_mir(...).binding_to_local` when MIR-local information is needed.

Phase 1 assumes that every HIR item generated by a preserved derive attribute
is recognized by `TyCtxt::is_automatically_derived`. Such generated items are
skipped during unexpanded mapping: they produce no skeleton records and do not
shift the mapping of later surface items. Derives that emit unmarked helper
items are outside the test contract. P1-ITEM-02 covers this behavior inside an
inline module, and P1-ITEM-10 covers root-level preservation and filtering.

All emitted Rust text is presentation-sanitized: remove `#[no_mangle]` and use
the default Rust ABI instead of `extern "C"`. This applies to annotated source,
annotated skeleton, both signatures, and declarations/definitions. The
original compiler input is not sanitized or changed, so analysis still sees
the real attributes and ABI. Preserve `unsafe`, visibility, names, parameter
mutability, and all other supported attributes.

The supported prototype input model assumes that transformable free functions
are `unsafe fn` items. Safe and `const fn` items are therefore not used to
define Phase 1 inclusion/filtering behavior. “Simple local binding” means a
`let IDENT` or `let mut IDENT` binding for which Rust syntax permits an explicit
type; destructuring and `for` patterns are preserved but are not rewritten into
typed `let` bindings.

The tests fix the complete ordered key set, including the previously unnamed
JSON text fields, as follows:

| Kind | Keys in serialization order |
| --- | --- |
| `Fn` | `id`, `path`, `kind`, `name`, `annotated_source`, `annotated_skeleton`, `source_signature`, `target_signature`, `signature_dependencies`, `dependencies` |
| `Static`, `Const` | `id`, `path`, `kind`, `declaration`, `signature_dependencies`, `dependencies` |
| `TyAlias`, `Enum`, `Struct`, `Union` | `id`, `path`, `kind`, `definition`, `dependencies` |

Serialization uses the equivalent of `serde_json::to_string_pretty`: two-space
indentation and no trailing newline. This formatting and the key order above
are part of the Phase 1 CLI contract and make the inline golden in P1-JSON-05
exact rather than implementation-dependent.

For skeleton holes, Phase 1 replaces a simple assignment expression as a whole:
`place = value` and `place += value` each become `todo!()`. `return value` and
`break value` retain their keyword while their value becomes `todo!()`.

Preserving a control/block expression at any nesting depth takes precedence
over that ordinary payload rule. If an initializer, assignment RHS, return
value, or other payload contains an `if`, `match`, loop, or block, the enclosing
syntactic role needed to retain that control expression is preserved; its
condition/scrutinee/iterator and non-control leaves become holes. This rule
reconciles the ordinary initializer cases in P1-SKEL-03 with the
nested-control case in P1-SKEL-06.

## 4. JSON contract tests

### P1-JSON-01 `skeleton_json_is_a_top_level_array_with_pascal_case_kinds`

Exact Rust input:

```rust
pub unsafe fn f() {}
pub static S: i32 = 0;
pub const C: i32 = 0;
pub type A = i32;
pub enum E { V }
pub struct T { pub x: i32 }
pub union U { pub x: i32, pub y: u32 }
```

Expected:

```text
root JSON type = array
(id, path, kind) =
  (0, "f", "Fn")
  (1, "S", "Static")
  (2, "C", "Const")
  (3, "A", "TyAlias")
  (4, "E", "Enum")
  (5, "T", "Struct")
  (6, "U", "Union")
```

Every element has `id`, `path`, and `kind`; there is no `{ "items": ... }`
wrapper.

### P1-JSON-02 `record_variants_serialize_only_their_defined_fields`

Exact Rust input:

```rust
pub unsafe fn f() {}
pub static S: i32 = 0;
pub struct T { pub x: i32 }
```

Expected:

```text
record 0 keys = [id, path, kind, name, annotated_source,
                 annotated_skeleton, source_signature, target_signature,
                 signature_dependencies, dependencies]
record 1 keys = [id, path, kind, declaration,
                 signature_dependencies, dependencies]
record 2 keys = [id, path, kind, definition, dependencies]
```

No other keys occur and no value is `null`.

### P1-JSON-03 `json_round_trip_preserves_embedded_rust_text`

Exact Rust input:

```rust
pub unsafe fn text() -> usize {
    let s = "quote:\" slash:\\ tab:\t line:\n";
    s.len()
}
```

Expected pre-round-trip values:

```text
source_signature = target_signature =
  "pub unsafe fn text() -> usize"
annotated_source =
  "pub unsafe fn text() -> usize {\n    #[proctor(0)]\n    let s = \"quote:\\\" slash:\\\\ tab:\\t line:\\n\";\n    #[proctor(1)]\n    s.len()\n}"
annotated_skeleton =
  "pub unsafe fn text() -> usize {\n    #[proctor(0)]\n    let s: &str = todo!();\n    #[proctor(1)]\n    todo!()\n}"
```

After record -> JSON -> record, all four values are byte-identical.

### P1-JSON-04 `json_serialization_is_byte_deterministic`

Exact Rust input: the complete P1-INTEG-01 input block.

Expected: two calls produce byte-identical JSON. Both have IDs `0..=7` and the
dependency arrays printed in P1-INTEG-01's expected-record table.

### P1-JSON-05 `small_skeleton_json_matches_inline_golden`

Input:

```rust
pub unsafe fn f() -> i32 {
    1
}
```

Exact expected JSON (the Rust snippets use the normalized pretty-printing
contract tested by this case):

```json
[
  {
    "id": 0,
    "path": "f",
    "kind": "Fn",
    "name": "f",
    "annotated_source": "pub unsafe fn f() -> i32 {\n    #[proctor(0)]\n    1\n}",
    "annotated_skeleton": "pub unsafe fn f() -> i32 {\n    #[proctor(0)]\n    todo!()\n}",
    "source_signature": "pub unsafe fn f() -> i32",
    "target_signature": "pub unsafe fn f() -> i32",
    "signature_dependencies": [],
    "dependencies": []
  }
]
```

There is no trailing newline.

### P1-JSON-06 `empty_crate_serializes_as_empty_array`

Exact Rust input is the zero-byte string `""`.

Expected: generation succeeds with no records and serializes exactly as `[]`.

## 5. Included-item discovery, paths, IDs, and rendering

### P1-ITEM-01 `includes_exactly_the_supported_item_kinds`

Exact Rust input:

```rust
pub unsafe fn f() {}
pub static S: i32 = 0;
pub static mut M: i32 = 0;
pub const C: i32 = 0;
pub type A = i32;
pub enum E { V }
pub struct Braced { pub x: i32 }
pub struct Tuple(pub i32);
pub struct Unit;
pub union U { pub x: i32, pub y: u32 }
```

Expected:

```text
0 f       Fn
1 S       Static    declaration = "pub static S: i32;"
2 M       Static    declaration = "pub static mut M: i32;"
3 C       Const
4 A       TyAlias
5 E       Enum
6 Braced  Struct
7 Tuple   Struct
8 Unit    Struct
9 U       Union
```

All dependency lists are empty.

### P1-ITEM-02 `flattens_inline_modules_in_recursive_source_order`

Exact Rust input:

```rust
const A: usize = 1;
mod outer {
    #[derive(Clone, Copy)]
    pub struct S { pub x: i32 }
    mod inner {
        pub unsafe fn f() {}
    }
    pub const B: usize = 2;
}
static Z: i32 = 0;
```

Expected paths and IDs, in order:

```text
0 A
1 outer::S
2 outer::inner::f
3 outer::B
4 Z
```

No `Mod` record is emitted.

All five records have empty dependency lists. The normalized definition of
`outer::S` preserves this exact item shape:

```rust
#[derive(Clone, Copy)]
pub struct S { pub x: i32 }
```

The generated `Clone` and `Copy` HIR impls produce no records and do not shift
the IDs of `outer::inner::f`, `outer::B`, or `Z`. This specifically verifies
that unexpanded AST-to-HIR mapping remains in unexpanded mode while recursing
through an inline module.

### P1-ITEM-03 `distinguishes_same_final_name_in_different_modules`

Exact Rust input:

```rust
mod a {
    pub struct T;
    pub unsafe fn f() {}
}
mod b {
    pub struct T;
    pub unsafe fn f() {}
}
```

Expected:

```text
0 a::T  Struct
1 a::f  Fn, name = "f"
2 b::T  Struct
3 b::f  Fn, name = "f"
```

All four records have empty dependency lists.

### P1-ITEM-04 `preserves_raw_identifiers_in_paths_and_names`

Exact Rust input:

```rust
mod r#type {
    pub unsafe fn r#match() {}
}
```

Expected: the path is exactly `r#type::r#match`, the final function name is
exactly `r#match`, and dependencies resolve by identity rather than by
comparing raw text.

The sole record is `id=0`, `kind=Fn`, with both dependency lists empty.

### P1-ITEM-05 `omits_modules_uses_and_extern_crates_as_records`

Exact Rust input:

```rust
extern crate core as rust_core;

mod m {
    pub struct T;
    pub const C: i32 = 3;
}

use m::T as Alias;
use m::*;

pub unsafe fn f(_alias: Alias) -> i32 {
    let _ = rust_core::mem::size_of::<Alias>();
    C
}
```

Expected:

```text
0 m::T  Struct dependencies=[]
1 m::C  Const  signature_dependencies=[] dependencies=[]
2 f     Fn     signature_dependencies=[0] dependencies=[0,1]
```

There are no records for the extern crate, module, or either `use`.

### P1-ITEM-06 `omits_foreign_modules_and_foreign_items`

Input:

```rust
#![feature(extern_types)]

unsafe extern "C" {
    static FOREIGN: i32;
    fn foreign_fn(x: i32) -> i32;
    type ForeignTy;
}

pub unsafe fn caller() -> i32 {
    foreign_fn(FOREIGN)
}
```

Expected: only `id=0 path="caller" kind=Fn` is emitted, with both dependency
lists `[]`. The foreign module and all three foreign declarations are omitted.

### P1-ITEM-08 `filters_nonrecord_item_kinds_in_supported_input`

This is a direct predicate test and takes no compiler Rust input. It constructs
one minimal `rustc_ast::Item` for each nonrecord kind that can occur in the
supported input model:

```text
ExternCrate, Use, ConstBlock, Mod, ForeignMod, GlobalAsm
```

Expected output from the inclusion predicate is this exact table:

```text
ExternCrate=false  Use=false       ConstBlock=false
Mod=false          ForeignMod=false GlobalAsm=false
```

Traits, impls, macro definitions, item-producing macro calls, delegations, and
external modules are upstream-excluded inputs and do not need predicate tests.

### P1-ITEM-09 `does_not_emit_variant_field_or_constructor_records`

Exact Rust input:

```rust
pub struct S { pub x: i32, pub y: i32 }
pub enum E {
    Unit,
    Tuple(i32),
    Struct { value: i32 },
}
```

Expected: `0 S Struct dependencies=[]` and `1 E Enum dependencies=[]` are the
only records. Fields, variants, and constructors have no IDs.

### P1-ITEM-10 `type_records_preserve_complete_definitions`

Exact Rust input:

```rust
#[repr(C)]
#[derive(Clone, Copy)]
pub struct S { pub x: i32, y: u8 }

#[repr(transparent)]
pub struct W(pub i32);

#[repr(i32)]
pub enum E { A = -1, B = 4 }

pub type Alias = (S, [W; 2]);
```

Expected normalized `definition` values contain exactly these item shapes:

```rust
#[repr(C)]
#[derive(Clone, Copy)]
pub struct S { pub x: i32, y: u8 }
#[repr(transparent)]
pub struct W(pub i32);
#[repr(i32)]
pub enum E { A = -1, B = 4 }
pub type Alias = (S, [W; 2]);
```

The surface AST preserves `#[derive(Clone, Copy)]`; the generator must not use
the expanded AST or emit generated `Clone`/`Copy` impl records. IDs are `S=0`,
`W=1`, `E=2`, `Alias=3`, and `Alias.dependencies=[0,1]`. No field or
discriminant is a hole.

### P1-ITEM-11 `value_records_render_declarations_without_initializers`

Input:

```rust
#[no_mangle]
pub static X: i32 = 1;
pub static mut BUFFER: *mut u8 = core::ptr::null_mut();
pub const SIZE: usize = 4;
```

Expected declarations are structurally equivalent to:

```rust
pub static X: i32;
pub static mut BUFFER: *mut u8;
pub const SIZE: usize;
```

The initializer text is absent from the declaration field.

`X`'s rendered declaration also omits `#[no_mangle]`; the sanitizer operates
on a render clone and the compiler input above remains unchanged.

Concrete record metadata:

```text
0 X      Static declaration="pub static X: i32;" sig=[] deps=[]
1 BUFFER Static declaration="pub static mut BUFFER: *mut u8;" sig=[] deps=[]
2 SIZE   Const  declaration="pub const SIZE: usize;" sig=[] deps=[]
```

### P1-ITEM-12 `function_records_sanitize_prompt_only_header_tokens_and_split_signatures`

Exact Rust input:

```rust
#[no_mangle]
pub unsafe extern "C" fn add(x: i32, y: i32) -> i32 {
    x + y
}
```

Expected:

```text
source_signature = target_signature =
  pub unsafe fn add(x: i32, y: i32) -> i32
annotated_source =
  pub unsafe fn add(x: i32, y: i32) -> i32 {\n    #[proctor(0)]\n    x + y\n}
annotated_skeleton =
  pub unsafe fn add(x: i32, y: i32) -> i32 {\n    #[proctor(0)]\n    todo!()\n}
```

Both complete functions retain visibility, `unsafe`, name, parameter names,
and types. The render-only sanitizer removes `#[no_mangle]` and `extern "C"`
from every emitted text field, but the compiler analysis runs on the unchanged
input shown above. Neither signature contains `proctor` or a body. Both
dependency lists are `[]`.

## 6. Dependency tests

### P1-DEP-01 `collects_direct_function_signature_type_dependencies`

Exact Rust input:

```rust
struct In;
struct Out;
pub unsafe fn f(_input: In, _out: *const Out) -> Out { loop {} }
```

Expected for `f`:

```text
ids: In=0, Out=1, f=2
signature_dependencies = [0,1]
dependencies = [0,1]
```

Repeated uses of `Out` are deduplicated.

### P1-DEP-02 `finds_types_nested_inside_signature_types`

Exact Rust input:

```rust
struct A;
struct B;
struct C;
struct D;
struct E;
struct F;

pub unsafe fn nested(
    _a: &A,
    _b: *mut B,
    _c: [C; 2],
    _d: &[D],
    _e: (E, Option<F>),
) {}
```

Expected: IDs are `A=0` through `F=5`, `nested=6`, and both dependency lists for
`nested` are exactly `[0,1,2,3,4,5]`. `Option` and primitives contribute no ID.

### P1-DEP-03 `collects_const_dependencies_from_array_lengths`

Input:

```rust
const N: usize = 4;
struct T;
pub unsafe fn f(x: [T; N]) -> [u8; N] { let _x = x; loop {} }
```

Expected: IDs are `N=0`, `T=1`, `f=2`; both lists for `f` are exactly `[0,1]`.

### P1-DEP-04 `collects_static_and_const_signature_dependencies`

Exact Rust input:

```rust
#[derive(Clone, Copy)]
struct T;
const N: usize = 2;
static mut X: *const T = core::ptr::null();
const VALUES: [T; N] = [T; N];
```

Expected:

```text
ids: T=0, N=1, X=2, VALUES=3
X.signature_dependencies = X.dependencies = [0]
VALUES.signature_dependencies = VALUES.dependencies = [0,1]
```

### P1-DEP-05 `collects_direct_function_calls_from_bodies`

Exact Rust input:

```rust
pub unsafe fn callee(_x: i32) {}
pub unsafe fn caller() {
    callee(1);
    callee(2);
}
```

Expected: `callee=0`, `caller=1`;
`caller.signature_dependencies=[]` and `caller.dependencies=[0]`.

### P1-DEP-06 `includes_direct_self_recursion`

Exact Rust input:

```rust
pub unsafe fn recur(n: u32) -> u32 {
    if n == 0 { 0 } else { recur(n - 1) }
}
```

Expected: `recur=0`, `signature_dependencies=[]`, `dependencies=[0]`.

### P1-DEP-07 `collects_body_local_type_annotations_and_cast_types`

Exact Rust input:

```rust
struct A;
struct B;
pub unsafe fn f() {
    let x: A = A;
    let p: *const A = &x;
    let _ = p as *const B;
}
```

Expected: `A=0`, `B=1`, `f=2`;
`f.signature_dependencies=[]` and `f.dependencies=[0,1]`.

### P1-DEP-08 `collects_static_and_const_uses_from_function_bodies`

Exact Rust input:

```rust
static mut S: i32 = 1;
const C: i32 = 2;
pub unsafe fn f() -> i32 { S + C }
```

Expected: `S=0`, `C=1`, `f=2`;
`f.signature_dependencies=[]` and `f.dependencies=[0,1]`.

### P1-DEP-09 `resolves_use_aliases_and_globs_to_item_ids`

Exact Rust input:

```rust
mod m {
    pub struct T;
    pub const C: i32 = 3;
}
use m::T as Alias;
use m::*;
pub unsafe fn f(_alias: Alias) -> i32 { C }
```

Expected: `m::T=0`, `m::C=1`, `f=2`;
`f.signature_dependencies=[0]`, `f.dependencies=[0,1]`, and there are no `Use`
records.

### P1-DEP-10 `does_not_confuse_shadowing_locals_with_items`

Exact Rust input:

```rust
unsafe fn value() -> i32 { 5 }
pub unsafe fn f() -> i32 {
    let value = 1;
    value + crate::value()
}
```

Expected: `value=0`, `f=1`;
`f.signature_dependencies=[]` and `f.dependencies=[0]` exactly. Repeated local
uses would not add another ID.

### P1-DEP-11 `ignores_foreign_and_external_dependencies`

Exact Rust input:

```rust
unsafe extern "C" {
    fn foreign(x: i32) -> i32;
    static FOREIGN: i32;
}

pub unsafe fn f() -> usize {
    let x: Option<i32> = Some(foreign(FOREIGN));
    core::mem::size_of_val(&x)
}
```

Expected: the only record is `f` with ID `0`; both dependency lists are `[]`.

### P1-DEP-12 `dependency_lists_are_direct_not_transitive`

Exact Rust input:

```rust
struct A;
struct B(A);
pub unsafe fn callee(_b: B) {}
pub unsafe fn caller(b: B) { callee(b); }
```

Expected:

```text
ids: A=0, B=1, callee=2, caller=3
A.dependencies=[]
B.dependencies=[0]
callee.signature_dependencies=callee.dependencies=[1]
caller.signature_dependencies=[1]
caller.dependencies=[1,2]
```

`A` is not transitively added to either function.

### P1-DEP-13 `canonicalizes_struct_constructors_to_struct_records`

Exact Rust input:

```rust
struct A { x: i32 }
struct B(i32);
struct C;
pub unsafe fn f() {
    let _ = A { x: 1 };
    let _ = B(2);
    let _ = C;
    let _ = A { x: 3 };
}
```

Expected: `A=0`, `B=1`, `C=2`, `f=3`;
`f.signature_dependencies=[]`, `f.dependencies=[0,1,2]`. No constructor has an
ID.

### P1-DEP-14 `canonicalizes_enum_variants_in_expressions_and_patterns`

Exact Rust input:

```rust
enum E { Unit, Tuple(i32), Struct { x: i32 } }
pub unsafe fn f(tag: i32) -> i32 {
    let e = if tag == 0 { E::Unit } else { E::Tuple(tag) };
    let _ = E::Struct { x: tag };
    match e {
        E::Unit => { 0 }
        E::Tuple(x) => { x }
        E::Struct { x } => { x }
    }
}
```

Expected: `E=0`, `f=1`; `f.signature_dependencies=[]`,
`f.dependencies=[0]`. Variants/constructors have no IDs.

### P1-DEP-15 `canonicalizes_field_accesses_to_containing_types`

Exact Rust input:

```rust
struct S { x: i32, y: i32 }
struct T(i32);
unsafe fn make_s() -> S { S { x: 1, y: 2 } }
unsafe fn make_t() -> T { T(3) }
pub unsafe fn f() -> i32 {
    let mut s = make_s();
    let t = make_t();
    s.x = s.y;
    s.x + t.0
}
```

Expected:

```text
ids: S=0, T=1, make_s=2, make_t=3, f=4
f.signature_dependencies=[]
f.dependencies=[0,1,2,3]
```

The field references add `S` and `T`; repeated `S` fields deduplicate.

### P1-DEP-16 `collects_initializer_dependencies_for_statics_and_consts`

Input:

```rust
struct Marker;
const BASE: i32 = 1;
static X: i32 = BASE;
const Y: Marker = Marker;
```

Expected:

- IDs are `Marker=0`, `BASE=1`, `X=2`, `Y=3`;
- `X.declaration` omits `BASE`,
  `X.signature_dependencies = []`, and `X.dependencies = [1]`;
- `Y.declaration` omits its initializer,
  `Y.signature_dependencies = [0]`, and `Y.dependencies = [0]`; and
- constructor canonicalization does not create a second dependency for
  `Marker`.

### P1-DEP-17 `collects_type_alias_dependencies`

Exact Rust input:

```rust
const N: usize = 4;
struct S;
type A = S;
type B = Option<(*mut A, [S; N])>;
```

Expected: `N=0`, `S=1`, `A=2`, `B=3`; `A.dependencies=[1]` and
`B.dependencies=[0,1,2]`. `Option` has no ID.

### P1-DEP-18 `collects_struct_and_union_field_dependencies`

Exact Rust input:

```rust
struct A;
type I = i32;
struct Braced { a: *const A, i: I }
struct Tuple(*mut A, I);
union U { a: *const A, i: I }
```

Expected: `A=0`, `I=1`, `Braced=2`, `Tuple=3`, `U=4`;
`Braced.dependencies=Tuple.dependencies=U.dependencies=[0,1]` and the first two
records have empty dependency lists.

### P1-DEP-19 `collects_enum_payload_and_discriminant_dependencies`

Exact Rust input:

```rust
const D: isize = 1;
struct A;
struct B;
#[repr(isize)]
enum E {
    One(A) = D,
    Two { b: B } = 2,
}
```

Expected: `D=0`, `A=1`, `B=2`, `E=3`; `E.dependencies=[0,1,2]`.

### P1-DEP-20 `handles_self_recursive_and_mutually_recursive_types`

Exact Rust input:

```rust
struct Node { next: *mut Node }
struct A { b: *mut B }
struct B { a: *mut A }
```

Expected:

- IDs are `Node=0`, `A=1`, `B=2`;
- `Node.dependencies=[0]`;
- `A.dependencies=[2]`;
- `B.dependencies=[1]`; and
- dependency collection terminates without computing transitive closure.

### P1-DEP-21 `resolves_same_spelling_in_value_and_type_namespaces`

Input:

```rust
type X = i32;
const X: i32 = 7;

pub unsafe fn f(x: X) -> i32 {
    X + x
}
```

Expected:

- IDs follow source order: type-namespace `X` is `0`, value-namespace `X` is
  `1`, and `f` is `2`;
- `f.signature_dependencies = [0]`;
- `f.dependencies = [0, 1]`; and
- both records have the display path `X`; namespace plus item ID, not
  the textual last path segment, selects the dependency.

### P1-DEP-22 `dependency_lists_are_deduplicated_sorted_and_subset_consistent`

Exact Rust input:

```rust
struct A;
struct B;
struct C;
pub unsafe fn f(_c: C, _a: *const A) -> A {
    let _: B = B;
    let _: B = B;
    loop {}
}
```

Expected:

```text
ids: A=0, B=1, C=2, f=3
f.signature_dependencies=[0,2]
f.dependencies=[0,1,2]
```

Two runs produce those same arrays.

## 7. Statement-annotation tests

Labels are expected to use deterministic depth-first preorder within each
function: label a statement before visiting statements nested inside it.

### P1-LABEL-01 `labels_let_semi_and_tail_statements`

Input:

```rust
unsafe extern "C" { fn consume(x: i32); }
pub unsafe fn f() -> i32 {
    let x = 1;
    consume(x);
    x
}
```

Expected normalized annotated source body:

```rust
{
    #[proctor(0)]
    let x = 1;
    #[proctor(1)]
    consume(x);
    #[proctor(2)]
    x
}
```

The skeleton has labels `0,1,2` at the identical three positions.

### P1-LABEL-02 `labels_reset_for_each_function`

Exact Rust input:

```rust
pub unsafe fn a() -> i32 {
    let x = 1;
    x
}
pub unsafe fn b() -> i32 {
    let y = 2;
    y + 1
}
```

Expected label sequences are `a=[0,1]` and `b=[0,1]` in both source and
skeleton. No function contains label `2`.

### P1-LABEL-03 `nested_labels_follow_depth_first_preorder`

Exact Rust input:

```rust
unsafe fn hit(_x: i32) {}
pub unsafe fn f(flag: bool) {
    if flag {
        hit(1);
        loop {
            hit(2);
            break;
        }
    } else {
        hit(3);
    }
    hit(4);
}
```

Expected preorder mapping:

```text
0 if flag { ... } else { ... }
1 hit(1);
2 loop { ... }
3 hit(2);
4 break;
5 hit(3);
6 hit(4);
```

### P1-LABEL-04 `source_and_skeleton_have_identical_label_trees`

Exact Rust input is the P1-SKEL-12 input block.

Expected source and skeleton position mappings are both exactly:

```text
[(0,root/0), (1,root/0/if-then/0), (2,root/1),
 (3,root/1/while/0), (4,root/2), (5,root/2/for/0),
 (6,root/3), (7,root/3/loop/0), (8,root/4),
 (9,root/4/match-arm-0/0), (10,root/4/match-arm-1/0),
 (11,root/5)]
```

### P1-LABEL-05 `labels_item_statements_inside_function_bodies`

Exact Rust input:

```rust
pub unsafe fn f() -> i32 {
    const LOCAL: i32 = 3;
    let x = LOCAL;
    x
}
```

Expected label mapping is `0=const LOCAL`, `1=let x`, `2=tail x`; the only JSON
record is `f` with ID `0`. `LOCAL` has no record or dependency ID.

### P1-LABEL-07 `rejects_top_level_empty_statement_in_function`

Input:

```rust
pub unsafe fn f() {
    ;
}
```

Expected structured error fields are
`kind=EmptyStatement`, `function_path="f"`, and message containing
`"empty statement cannot be annotated"`. No records are returned.

### P1-LABEL-08 `rejects_nested_empty_statement`

Exact Rust input:

```rust
pub unsafe fn f(flag: bool) {
    if flag {
        loop {
            ;
        }
    }
}
```

Expected structured error fields:

```text
kind = EmptyStatement
function_path = "f"
message contains = "empty statement cannot be annotated"
```

No records are returned.

## 8. Skeleton-shape and placeholder tests

### P1-SKEL-01 `replaces_leaf_expression_payloads_with_todo`

Exact Rust input:

```rust
unsafe fn callee(_x: i32) {}
pub unsafe fn f() -> i32 {
    let x = 1;
    callee(x);
    -x + 2
}
```

Expected normalized skeleton body:

```rust
{
    #[proctor(0)] let x: i32 = todo!();
    #[proctor(1)] todo!();
    #[proctor(2)] todo!()
}
```

The source body still contains `1`, `callee(x)`, and `-x + 2` at labels
`0,1,2`.

### P1-SKEL-02 `materializes_inferred_types_for_simple_bindings`

Exact Rust input:

```rust
struct Local;
pub unsafe fn f() {
    let b = true;
    let i = -1i32;
    let u = 1u64;
    let n = 1.5f32;
    let c = 'x';
    let t = (1i32, 2u8);
    let a = [1u16; 3];
    let r = &i;
    let l = Local;
}
```

Expected skeleton declarations, in order:

```rust
let b: bool = todo!();
let i: i32 = todo!();
let u: u64 = todo!();
let n: f32 = todo!();
let c: char = todo!();
let t: (i32, u8) = todo!();
let a: [u16; 3] = todo!();
let r: &i32 = todo!();
let l: Local = todo!();
```

They carry labels `0..=8`. The annotated source keeps all nine inferred forms.

### P1-SKEL-03 `preserves_mutability_declarations_and_existing_types`

Exact Rust input:

```rust
struct T;
pub unsafe fn f() {
    let mut a = 1;
    let x: T;
    let y: i32 = 2;
    x = T;
    let _ = (a, x, y);
}
```

Expected normalized skeleton statements:

```rust
#[proctor(0)] let mut a: i32 = todo!();
#[proctor(1)] let x: T;
#[proctor(2)] let y: i32 = todo!();
#[proctor(3)] todo!();
#[proctor(4)] let _ = todo!();
```

### P1-SKEL-04 `holes_assignments_and_preserves_return_and_break_roles`

Exact Rust input:

```rust
pub unsafe fn f(mut x: i32) -> i32 {
    x = 1;
    x += 2;
    let y = loop {
        break x;
    };
    return y;
}
```

Expected normalized skeleton:

```rust
pub unsafe fn f(mut x: i32) -> i32 {
    #[proctor(0)] todo!();
    #[proctor(1)] todo!();
    #[proctor(2)] let y: i32 = loop {
        #[proctor(3)] break todo!();
    };
    #[proctor(4)] return todo!();
}
```

### P1-SKEL-05 `preserves_if_and_else_structure`

Exact Rust input:

```rust
unsafe fn sink(_x: i32) {}
pub unsafe fn f(flag: bool) {
    if flag {
        sink(1);
    } else {
        sink(2);
    }
}
```

Expected normalized skeleton:

```rust
#[proctor(0)] if todo!() {
    #[proctor(1)] todo!();
} else {
    #[proctor(2)] todo!();
}
```

### P1-SKEL-06 `preserves_nested_if_and_else_if_structure`

Exact Rust input:

```rust
pub unsafe fn f(a: bool, b: bool, c: bool) -> i32 {
    let x = if a { 1 } else { 2 };
    if b {
        if c { x } else { 3 }
    } else if a {
        4
    } else {
        5
    }
}
```

Expected skeleton control shape:

```rust
let x: i32 = if todo!() { todo!() } else { todo!() };
if todo!() {
    if todo!() { todo!() } else { todo!() }
} else if todo!() {
    todo!()
} else {
    todo!()
}
```

Expected preorder labels are:

```text
0 let x
1 first-if then value
2 first-if else value
3 outer tail if
4 nested if in outer-then
5 nested-if then value
6 nested-if else value
7 else-if then value
8 final else value
```

Source and skeleton use this identical mapping.

### P1-SKEL-07 `preserves_if_let_and_while_let_patterns`

Exact Rust input:

```rust
unsafe fn sink(_x: i32) {}
pub unsafe fn f(mut value: Option<i32>) {
    if let Some(x) = value {
        sink(x);
    }
    while let Some(x) = value {
        sink(x);
        value = None;
    }
}
```

Expected normalized skeleton:

```rust
pub unsafe fn f(mut value: Option<i32>) {
    if let Some(x) = todo!() {
        todo!();
    }
    while let Some(x) = todo!() {
        todo!();
        todo!();
    }
}
```

The outer statements have labels `0` and `2`; their children have labels `1`
and `3,4`, respectively.

### P1-SKEL-08 `preserves_while_for_and_loop_constructs`

Exact Rust input:

```rust
unsafe fn sink(_x: i32) {}
pub unsafe fn f(mut n: i32, pairs: [(i32, i32); 2]) {
    'w: while n > 0 {
        n -= 1;
    }
    for (x, y) in pairs {
        sink(x + y);
    }
    'l: loop {
        break 'l;
    }
}
```

Expected normalized skeleton:

```rust
pub unsafe fn f(mut n: i32, pairs: [(i32, i32); 2]) {
    'w: while todo!() { todo!(); }
    for (x, y) in todo!() { todo!(); }
    'l: loop { break 'l; }
}
```

Outer labels are `0,2,4`; child labels are `1,3,5`.

### P1-SKEL-09 `preserves_match_arms_patterns_guards_and_order`

Exact Rust input:

```rust
enum E { Unit, Tuple(i32), Struct { x: i32 } }
unsafe fn sink(_x: i32) {}
pub unsafe fn f(e: E, n: i32, pair: (i32, i32)) -> i32 {
    let a = match e {
        E::Unit => { 0 }
        E::Tuple(x) if x > 0 => { x }
        E::Tuple(_) => { -1 }
        E::Struct { x } => { sink(x); x },
    };
    let b = match n {
        0 => { 0 }
        1..=3 => { 1 }
        _ => { 2 }
    };
    match pair { (x, y) => { a + b + x + y } }
}
```

Expected normalized skeleton:

```rust
pub unsafe fn f(e: E, n: i32, pair: (i32, i32)) -> i32 {
    let a: i32 = match todo!() {
        E::Unit => { todo!() }
        E::Tuple(x) if todo!() => { todo!() }
        E::Tuple(_) => { todo!() }
        E::Struct { x } => { todo!(); todo!() },
    };
    let b: i32 = match todo!() {
        0 => { todo!() }
        1..=3 => { todo!() }
        _ => { todo!() }
    };
    match todo!() { (x, y) => { todo!() } }
}
```

Expected depth-first preorder label map is:

```text
0  let a
1  unit-arm tail 0
2  guarded tuple-arm tail x
3  fallback tuple-arm tail -1
4  struct-arm sink(x)
5  struct-arm tail x
6  let b
7  b zero-arm tail 0
8  b range-arm tail 1
9  b fallback-arm tail 2
10 function-tail match pair
11 pair-arm tail a + b + x + y
```

Every arm body is a block expression in both source and skeleton, and every
statement within each arm has the corresponding label.

### P1-SKEL-10 `preserves_let_else_and_plain_nested_blocks`

Exact Rust input:

```rust
unsafe fn sink(_x: i32) {}
pub unsafe fn f(value: Option<i32>) -> i32 {
    let Some(x): Option<i32> = value else {
        return 0;
    };
    let y = {
        sink(x);
        x + 1
    };
    y
}
```

Expected normalized skeleton:

```rust
pub unsafe fn f(value: Option<i32>) -> i32 {
    let Some(x): Option<i32> = todo!() else {
        return todo!();
    };
    let y: i32 = {
        todo!();
        todo!()
    };
    todo!()
}
```

The source and skeleton label tree is `0 let-else`, `1 return`, `2 let y`,
`3 sink`, `4 block tail`, `5 function tail`.

### P1-SKEL-11 `preserves_existing_identifiers_paths_and_patterns`

Exact Rust input:

```rust
mod m { pub struct Pair { pub left: i32, pub right: i32 } }
pub unsafe fn keep_names(mut input_value: m::Pair) -> i32 {
    let mut local_total = input_value.left;
    'outer: loop {
        let m::Pair { left: bound_left, right: bound_right } = input_value;
        local_total += bound_left + bound_right;
        break 'outer;
    }
    local_total
}
```

Expected normalized skeleton:

```rust
pub unsafe fn keep_names(mut input_value: m::Pair) -> i32 {
    let mut local_total: i32 = todo!();
    'outer: loop {
        let m::Pair { left: bound_left, right: bound_right } = todo!();
        todo!();
        break 'outer;
    }
    todo!()
}
```

No identifier beginning with `__crat`, `tmp`, or `proctor_tmp` appears.

### P1-SKEL-12 `annotated_source_and_skeleton_snippets_parse_independently`

Exact Rust input:

```rust
unsafe fn sink(_x: i32) {}
pub unsafe fn comprehensive(mut n: i32) -> i32 {
    if n > 0 { sink(n); }
    while n > 1 { n -= 1; }
    for i in 0..n { sink(i); }
    loop { break; }
    match n {
        0 => { sink(0); }
        _ => { sink(n); }
    }
    n
}
```

Expected: `annotated_source` and `annotated_skeleton` each parse as exactly one
item of kind `Fn` named `comprehensive`; their label sequence is exactly
`0..=11`, with the position mapping specified in P1-LABEL-04.

### P1-SKEL-14 `preserves_payloadless_control_expressions_without_holes`

Exact Rust input:

```rust
pub unsafe fn f(flag: bool) {
    if flag { return; }
    loop {
        if flag { continue; }
        break;
    }
}
```

Expected skeleton contains `return;`, `continue;`, and `break;` exactly once
each, never `return todo!()`, `continue todo!()`, or `break todo!()`. Expected
preorder labels are `0 if`, `1 return`, `2 loop`, `3 nested if`, `4 continue`,
`5 break`.

### P1-SKEL-15 `rejects_non_block_match_arm`

Exact Rust input:

```rust
pub unsafe fn f(n: i32) -> i32 {
    match n {
        0 => 1,
        _ => { 2 }
    }
}
```

Expected structured error fields are:

```text
kind = NonBlockMatchArm
function_path = "f"
message contains = "match arm body must be a block expression"
```

No records or JSON are returned. This is the one explicit rejection test for
the supported-input match-arm invariant; other upstream-excluded constructs do
not require validators.

## 9. Target-type and pointer-analysis tests

These tests exercise the skeleton-specific pointer API. Several related
functions may share one compiler input so the expensive whole-program analysis
runs once per test rather than once per individual assertion.

The pointer-replacer API used here exposes the initial `SigDecisions` plus
initial `collect_diffs` decisions before transformation-time `downgrade_*`
logic or AST rewriting. It accepts the tools-only
`PointerDecisionOptions::assume_nonnegative_offsets` option. The option
defaults to `false` for the existing
`crat` pointer pass. Phase 1 always sets it to `true`, skips
`offset_sign_analysis`, supplies `OffsetSignResult::default()`, and must never
produce `PtrKind::SliceCursor`.

All TYPE cases use `pointer_replacer::Config::default()` with empty
`c_exposed_fns`; Phase 1 does not read `proctor.toml`. The separate decision
option changes only the offset/cursor policy.

The expected TYPE decisions are based on current pointer-replacer behavior,
not an independent type system invented for the tools crate. Relevant existing
regressions include:

| Planned behavior | Existing pointer-replacer regression |
| --- | --- |
| non-null scalar reference | `test_non_null_param_to_ref`, `test_local_ptr_to_ref` |
| nullable reference | `test_param_null_check_before_deref_stays_optional`, `test_rewriter_reborrows_repeated_optional_mut_arg_for_optional_callee` |
| slices | `test_raw_eq_slice`, `test_slice_eq_slice`, `test_rewriter_generalizes_wrapper_with_internal_free_after_foreign_use` |
| scalar box | `test_rewriter_rewrites_malloc_scalar_to_opt_box` |
| boxed slice | `test_rewriter_rewrites_calloc_array_to_opt_boxed_slice` |
| returned-borrow lifetime | `test_rewriter_promotes_interprocedural_return_lifetime` |
| nullable returned borrow | `test_rewriter_preserves_nullable_returned_borrow_lifetime`, `test_rewriter_preserves_nullable_returned_borrow_through_local` |
| raw global escape | `test_rewriter_keeps_wrapper_escape_through_global_raw_in_m9` |
| optional box call boundary | `test_rewriter_rewrites_local_call_boundary_for_opt_box` |
| later local-struct demotion | `test_rewriter_downgrades_local_struct_call_conflict_with_scalar_read` |

Those existing tests generally assert final rewritten Rust and therefore do
not substitute for these initial-decision tests. During implementation, run
each exact source block below through the new initial-decision API and assert
the concrete oracle stated here. If an exact fixture contradicts the current
analysis, stop and report the conflict instead of silently changing the test
or implementing special-case logic in `tools`.

### P1-TYPE-01 `keeps_non_pointer_signature_types_unchanged`

Exact Rust input:

```rust
struct S { x: i32 }
pub unsafe fn f(a: i32, b: (u8, bool), c: [i16; 2], s: S) -> (S, usize) {
    (s, a as usize + b.0 as usize + c[0] as usize)
}
```

Expected source and target signatures are both exactly:

```rust
pub unsafe fn f(a: i32, b: (u8, bool), c: [i16; 2], s: S) -> (S, usize)
```

The skeleton tail is `#[proctor(0)] todo!()`.

### P1-TYPE-02 `selects_shared_and_mutable_scalar_references`

Exact Rust input:

```rust
pub unsafe fn read_param(p: *const i32) -> i32 {
    *p
}

pub unsafe fn write_local() -> i32 {
    let mut x = 0i32;
    let p: *mut i32 = &mut x;
    *p = 7;
    x
}

pub unsafe fn read_local() -> i32 {
    let x = 7i32;
    let p: *const i32 = &x;
    *p
}
```

Expected target decisions:

```text
read_param: p = &i32
write_local: p = &mut i32
read_local: p = &i32
```

The `x` locals remain `i32`; none of these three `p` locations remains raw.

### P1-TYPE-03 `selects_optional_references_when_null_is_observable`

Exact Rust input:

```rust
pub unsafe fn read(p: *const i32) -> i32 {
    if p.is_null() { 0 } else { *p }
}
pub unsafe fn write(p: *mut i32) {
    if !p.is_null() { *p = 1; }
}
```

Expected target signatures are:

```rust
pub unsafe fn read(p: Option<&i32>) -> i32
pub unsafe fn write(p: Option<&mut i32>)
```

The source signatures retain `*const i32` and `*mut i32`.

### P1-TYPE-04 `selects_shared_and_mutable_slices_for_array_borrows`

Input:

```rust
pub unsafe fn read_array(a: [i32; 4]) -> i32 {
    let p: *const i32 = a.as_ptr();
    *p.offset(1)
}

pub unsafe fn write_array(mut a: [i32; 4]) -> i32 {
    let p: *mut i32 = a.as_mut_ptr();
    *p.offset(1) = 9;
    a[1]
}
```

Expected target local types are exactly `p: &[i32]` in `read_array` and
`p: &mut [i32]` in `write_array`. No target type contains `SliceCursor`.

### P1-TYPE-05 `promotes_array_derived_locals_to_explicit_slices`

Exact Rust input is the P1-TYPE-04 input block.

Expected skeleton declarations are exactly
`let p: &[i32] = todo!();` in `read_array` and
`let p: &mut [i32] = todo!();` in `write_array`, even though both source
declarations explicitly use raw pointers.

### P1-TYPE-07 `selects_scalar_box_types_from_ownership_analysis`

Exact Rust input:

```rust
unsafe extern "C" { fn malloc(size: usize) -> *mut i32; }
pub unsafe fn alloc() -> *mut i32 {
    let p: *mut i32 = malloc(core::mem::size_of::<i32>());
    *p = 7;
    p
}
```

Expected target signature is `pub unsafe fn alloc() -> Box<i32>` and the
skeleton declaration is `let p: Box<i32> = todo!();`. There is no allocator
rewrite text in the skeleton.

### P1-TYPE-08 `selects_boxed_slice_types_from_ownership_and_fatness`

Exact Rust input:

```rust
unsafe extern "C" { fn calloc(count: usize, size: usize) -> *mut i32; }
pub unsafe fn alloc_array() -> *mut i32 {
    let p: *mut i32 = calloc(4, core::mem::size_of::<i32>());
    *p.offset(1) = 7;
    p
}
```

Expected target signature is `pub unsafe fn alloc_array() -> Box<[i32]>` and
the skeleton declaration is `let p: Box<[i32]> = todo!();`. There is no
allocator rewrite text in the skeleton.

### P1-TYPE-09 `adds_named_lifetimes_for_returned_borrows`

Exact Rust input:

```rust
pub unsafe fn id(x: *mut i32) -> *mut i32 { x }
pub unsafe fn wrap(y: *mut i32) -> *mut i32 { id(y) }
```

Expected:

- target input and output references share the generated named lifetime;
- `id` has target signature
  `pub unsafe fn id<'a>(x: &'a mut i32) -> &'a mut i32`;
- `wrap` has target signature
  `pub unsafe fn wrap<'a>(y: &'a mut i32) -> &'a mut i32`; and
- source signatures remain unchanged.

### P1-TYPE-10 `preserves_nullable_returned_borrow_relationships`

Exact Rust input:

```rust
pub unsafe fn maybe(flag: bool, x: *mut i32) -> *mut i32 {
    if flag { x } else { core::ptr::null_mut() }
}
pub unsafe fn maybe_local(flag: bool, x: *mut i32) -> *mut i32 {
    let r = if flag { x } else { core::ptr::null_mut() };
    r
}
```

Expected target signatures:

```rust
pub unsafe fn maybe<'a>(flag: bool, x: Option<&'a mut i32>)
    -> Option<&'a mut i32>
pub unsafe fn maybe_local<'a>(flag: bool, x: Option<&'a mut i32>)
    -> Option<&'a mut i32>
```

`maybe_local`'s skeleton has `let r: Option<&mut i32> = if ...`.

### P1-TYPE-11 `keeps_required_raw_pointers_raw`

Exact Rust input:

```rust
unsafe extern "C" {
    fn malloc(size: usize) -> *mut core::ffi::c_void;
}

static mut SLOT: *mut *mut i32 = core::ptr::null_mut();

pub unsafe fn void_address(out: *mut core::ffi::c_void) -> usize {
    out as usize
}

pub unsafe fn keep_alias_raw(a: *mut i32, b: *mut i32) -> *mut i32 {
    *a = 1;
    *b = 2;
    a
}

pub unsafe fn alias_caller() -> *mut i32 {
    let mut x = 7i32;
    let p: *mut i32 = &mut x;
    keep_alias_raw(p, p)
}

pub unsafe fn global_escape() {
    let p: *mut *mut i32 =
        malloc(core::mem::size_of::<*mut i32>()) as *mut *mut i32;
    SLOT = p;
}
```

Expected:

- `out` remains exactly `*mut core::ffi::c_void`;
- both `keep_alias_raw` parameters remain `*mut i32`; and
- `global_escape`'s `p` local remains `*mut *mut i32` because it escapes through
  `SLOT`.

Raw retention is not confused with transformation-time demotion.

### P1-TYPE-12 `does_not_change_named_type_or_global_declarations`

Exact Rust input:

```rust
pub struct S { pub p: *mut i32 }
pub union U { pub p: *const i32, pub bits: usize }
pub enum E { Ptr(*mut i32), Empty }
pub type Alias = *const i32;
pub static mut GLOBAL: *mut i32 = core::ptr::null_mut();
pub const NIL: *const i32 = core::ptr::null();
pub unsafe fn use_all(s: S, u: U, e: E, a: Alias) -> usize {
    let _ = (s, u, e, a, GLOBAL, NIL);
    0
}
```

Expected:

- `S::p` remains `*mut i32`, `U::p` remains `*const i32`, `E::Ptr` remains
  `*mut i32`, `Alias` remains `*const i32`, and both global declarations remain
  raw;
- only function signatures and local-variable types may change; and
- dependency collection still reports the named/global items correctly.

Concrete records/dependencies:

```text
0 S       Struct deps=[]
1 U       Union  deps=[]
2 E       Enum   deps=[]
3 Alias   TyAlias deps=[]
4 GLOBAL  Static sig=[] deps=[]
5 NIL     Const  sig=[] deps=[]
6 use_all Fn     sig=[0,1,2,3] deps=[0,1,2,3,4,5]
```

### P1-TYPE-13 `uses_initial_decisions_before_rewriter_fallback_demotion`

Exact Rust input:

```rust
#[repr(C)]
pub struct Tree {
    root_id: i32,
}

pub unsafe fn tree_print_helper(tree: *mut Tree, root_id: i32) {
    (*tree).root_id = root_id;
}

pub unsafe fn caller(tree: *mut Tree) {
    tree_print_helper(tree, (*tree).root_id);
}
```

Expected:

- the skeleton target matches `SigDecisions` plus initial `collect_diffs`;
- the skeleton target for `tree_print_helper` retains the initial
  signature
  `pub unsafe fn tree_print_helper(tree: &mut crate::Tree, root_id: i32)`;
- the ordinary rewriter's final signature for the same fixture has
  `pub unsafe fn tree_print_helper(mut tree: *mut crate::Tree, root_id: i32)`
  because of its call-expression conflict fallback; and
- no ordinary `downgrade_*` or AST-visit fallback affects the skeleton result.

The comparison is entirely in memory.

### P1-TYPE-14 `all_simple_locals_receive_final_explicit_target_types`

Exact inputs are the complete source blocks from P1-TYPE-02, P1-TYPE-03,
P1-TYPE-04, P1-TYPE-07, P1-TYPE-08, P1-TYPE-11, and P1-TYPE-16, each run as its
own compiler input.

Expected local type map:

```text
TYPE-02: write_local.x=i32, write_local.p=&mut i32,
         read_local.x=i32, read_local.p=&i32
TYPE-04: read_array.p=&[i32], write_array.p=&mut [i32]
TYPE-07: alloc.p=Box<i32>
TYPE-08: alloc_array.p=Box<[i32]>
TYPE-11: alias_caller.x=i32, alias_caller.p=&mut i32,
         global_escape.p=*mut *mut i32
TYPE-16: foo.p=Box<i32>, foo.q=Option<Box<i32>>
```

Each named binding occurs with exactly one explicit type in its skeleton.

### P1-TYPE-15 `skeleton_pointer_result_contains_no_cursor_variant`

Exact inputs are P1-TYPE-02, P1-TYPE-04, P1-TYPE-11, P1-TYPE-12, and
P1-TYPE-17's complete source blocks.

Expected:

- the structured pointer result contains no `PtrKind::SliceCursor`;
- rendered target signatures and local types contain no `SliceCursor` or
  `slice_cursor` path; and
- no generated cursor helper item appears in the skeleton JSON.

### P1-TYPE-16 `selects_optional_boxes_at_local_call_boundaries`

Exact Rust input:

```rust
unsafe extern "C" { fn malloc(size: usize) -> *mut i32; }
pub unsafe fn owned_id(mut p: *mut i32) -> *mut i32 { p }
pub unsafe fn foo() -> *mut i32 {
    let p: *mut i32 = malloc(core::mem::size_of::<i32>());
    *p = 7;
    let q: *mut i32 = owned_id(p);
    q
}
```

Expected:

- `owned_id` has target signature
  `pub unsafe fn owned_id(mut p: Option<Box<i32>>) -> Option<Box<i32>>`;
- `foo` has target return type `Option<Box<i32>>`;
- `foo`'s `p` local has type `Box<i32>` and its `q` local has type
  `Option<Box<i32>>`; and
- allocator and call expressions remain holes rather than being rewritten.

### P1-TYPE-17 `tools_mode_disables_conservative_slice_cursors_without_changing_crat_default`

This tools-crate policy test uses an offset that is semantically nonnegative
but that the current sign analysis conservatively classifies as unknown. It
therefore exercises the false-positive case motivating the tools option while
remaining within the production nonnegative-offset contract.

Exact Rust input:

```rust
pub unsafe fn read_at(p: *const i32, offset: isize) -> i32 {
    let nonnegative = offset.max(0);
    *p.offset(nonnegative)
}

pub unsafe fn caller(offset: isize) -> i32 {
    let values = [10i32, 20, 30, 40];
    read_at(values.as_ptr(), offset)
}
```

In two independent `run_compiler_on_str` callbacks, call the same exported
initial-decision API first with its default cursor-aware options and then with
`assume_nonnegative_offsets=true`. Call the tools skeleton generator in the
second callback.

Concrete expected output:

```text
default mode:
  read_at.p structured kind = PtrKind::SliceCursor(false)

tools mode:
  read_at.p structured kind = PtrKind::Slice(false)
  read_at target signature =
    pub unsafe fn read_at(p: &[i32], offset: isize) -> i32
  read_at.nonnegative target local type = isize
  caller.values target local type = [i32; 4]
  every structured decision is not PtrKind::SliceCursor(_)
  every rendered field lacks "SliceCursor" and "slice_cursor"
```

The normal `replace_local_borrows` entry point continues using default mode;
no existing Crat configuration or caller opts into the tools assumption. The
implementation branch for tools mode must construct an empty
`OffsetSignResult` without calling `offset_sign_analysis`; the runtime oracle
checks the observable policy, while code inspection enforces that the analysis
was actually skipped rather than run and postprocessed.


## 10. Whole-output and error tests

### P1-INTEG-01 `comprehensive_fixture_emits_consistent_records`

Input:

```rust
const N: usize = 4;

mod model {
    pub struct Point {
        pub x: i32,
    }

    pub union Bits {
        pub i: i32,
        pub u: u32,
    }

    pub enum Mode {
        Off = 0,
        On = crate::N as isize,
    }

    pub type PointPtr = *mut Point;
    pub static ORIGIN: Point = Point { x: 0 };

    pub unsafe fn read(p: *const Point) -> i32 {
        let x = (*p).x;
        if x > 0 {
            x
        } else {
            crate::helper(x)
        }
    }
}

pub unsafe fn helper(x: i32) -> i32 {
    let mut total = 0;
    for i in 0..x {
        total += i;
    }
    if x <= 0 {
        total
    } else {
        helper(x - 1)
    }
}
```

Expected record model:

| ID | Path | Kind | Signature dependencies | Dependencies |
| ---: | --- | --- | --- | --- |
| 0 | `N` | `Const` | `[]` | `[]` |
| 1 | `model::Point` | `Struct` | n/a | `[]` |
| 2 | `model::Bits` | `Union` | n/a | `[]` |
| 3 | `model::Mode` | `Enum` | n/a | `[0]` |
| 4 | `model::PointPtr` | `TyAlias` | n/a | `[1]` |
| 5 | `model::ORIGIN` | `Static` | `[1]` | `[1]` |
| 6 | `model::read` | `Fn` | `[1]` | `[1,7]` |
| 7 | `helper` | `Fn` | `[]` | `[7]` |

`model::read` has source parameter `p: *const Point` and target parameter
`p: &crate::model::Point`. `helper`'s inferred `total` binding is an explicit
`i32` local in the skeleton; its `for` pattern remains `i`. This one input
therefore combines nested modules, every included item kind, direct and
recursive calls, value/type dependencies, inferred locals, multiple control
structures, a transformed pointer signature, and unchanged named types.

Concrete label maps:

```text
model::read: 0=let x, 1=tail if, 2=then x, 3=else helper(x)
helper:      0=let total, 1=for, 2=total += i,
             3=tail if, 4=then total, 5=else helper(x - 1)
```

Expected:

- the eight records match the table above;
- every function has parseable annotated source and skeleton;
- every function's source/skeleton label trees match;
- signature dependencies are subsets of dependencies;
- all lists are sorted and deduplicated; and
- no cursor type appears.

### P1-INTEG-02 `generation_is_deterministic_across_compiler_runs`

Exact Rust input is the complete P1-INTEG-01 block. Run those same bytes through
two independent `run_compiler_on_str` invocations.

Expected: deserialized records and serialized JSON bytes are identical despite
fresh rustc contexts and different internal `DefId`/`HirId` allocations.

### P1-INTEG-03 `record_paths_and_dependency_ids_are_self_consistent`

Exact Rust input:

```rust
mod a {
    pub struct T;
    pub const C: i32 = 1;
    pub unsafe fn make(_value: T) -> i32 { C }
}
mod b {
    pub struct T;
    pub unsafe fn call(x: crate::a::T) -> i32 {
        crate::a::make(x)
    }
}
pub unsafe fn root(x: a::T, _other: b::T) -> i32 { b::call(x) }
```

Expected:

```text
0 a::T     Struct deps=[]
1 a::C     Const  sig=[] deps=[]
2 a::make  Fn     sig=[0] deps=[0,1]
3 b::T     Struct deps=[]
4 b::call  Fn     sig=[0] deps=[0,2]
5 root     Fn     sig=[0,3] deps=[0,3,4]
```

Thus IDs are exactly `0..=5`, every dependency is in range, paths are unique in
their namespaces, and function names are `make`, `call`, and `root`.

### P1-INTEG-04 `empty_statement_error_prevents_partial_output`

Exact Rust input:

```rust
pub unsafe fn valid() -> i32 { 1 }
pub unsafe fn invalid(flag: bool) {
    if flag {
        ;
    }
}
```

Expected result:

```text
Err { kind=EmptyStatement,
      function_path="invalid",
      message contains "empty statement cannot be annotated" }
records returned = none
JSON returned = none
```

## 11. Completion criteria

Phase 1 testing is complete when:

- all 80 test cases in Sections 4 through 10 exist and pass under
  `cargo test --workspace`;
- tests perform no filesystem writes or project creation;
- no test depends on execution order;
- no test mutates process-global configuration;
- repeated-run determinism tests pass;
- existing default-mode pointer cursor regressions remain unchanged;
- implementation inspection confirms that the tools-mode initial-decision
  branch never calls offset-sign analysis and supplies an empty
  `OffsetSignResult` (the runtime tests enforce its observable no-cursor
  policy, not internal call tracing);
- no tool-mode structured or rendered result contains a cursor variant;
- the JSON renderer uses the separately parsed surface AST structurally mapped
  to HIR in recursively propagated unexpanded mode, skips all automatically
  derived HIR items, preserves derive attributes, and sanitizes `#[no_mangle]`/
  `extern "C"` only on render clones;
- the comprehensive fixture has been manually checked against the JSON schema;
  and
- `cargo fmt`, `cargo clippy --workspace --all-targets`, and
  `cargo test --workspace` pass from `crat/`.
