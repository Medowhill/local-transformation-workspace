# Amendment 4 Test Plan: Scope-aware synthesized type spelling

## 1. Purpose

This document specifies the complete focused regression suite for Amendment
Plan 4 in [amendment-4-plan.md](amendment-4-plan.md). It covers resolver-backed
type-name selection, source-AST pointee reuse, semantic fallback rendering,
standard `Option`/`Box` constructor safety, structured generation failure, and
compiler-backed insertion of generated types in the original module.

The historical phase and amendment test plans are not edited. Existing
implementation tests whose exact type strings conflict with Amendment 4 are
updated as specified here. This amendment changes no JSON field, schema
version, validator rule, replacement protocol, Python model, prompt, SCC
schedule, pointer decision, or ordinary pointer-rewriter output.

The suite contains 18 named cases:

| Area | Cases |
| --- | ---: |
| Existing regression updates | 2 |
| Inferred-local scope spelling | 6 |
| Pointer-introduced type spelling | 5 |
| Preservation and failure boundaries | 2 |
| Validation/replacement/compiler integration | 1 |
| Unchanged APIs and determinism | 2 |

## 2. Test execution and ownership policy

All new and updated tests are Rust tests beside the existing Crat tools
skeleton, validator, and item-replacer modules. Run them with:

```bash
cd proctor/stages/crat
cargo test -p tools skeleton::tests
cargo test -p tools validator::tests
cargo test -p tools item_replacer::tests
cargo test --workspace
```

Constructor-precondition tests belong only to `skeleton::tests`. The validator
and item-replacer commands above rerun unchanged protocol and wrapper
regressions; they do not authorize new constructor recognition.

After implementation, run the required Crat checks:

```bash
cd proctor/stages/crat
cargo fmt
cargo clippy --workspace --all-targets
cargo test --workspace
```

Skeleton semantic tests pass the exact sources in Section 3 to
`utils::compilation::run_compiler_on_str`. Parser-only structural tests remain
parser-only. The integration case obtains owned generated/replaced Rust from
one compiler callback, lets that `TyCtxt` end, and compiles the owned result in
a separate `run_compiler_on_str` invocation. Never nest compiler callbacks.

Tests create no fixture files, Cargo projects, subprocesses, snapshots, or
environment-variable mutations. They do not invoke `crat-tool`, Cargo, the
Python stage, or an LLM. Normal outer `cargo test` build artifacts are not
test-case filesystem effects.

No test edits:

- `phase-1-test-plan.md`;
- `phase-2-test-plan.md`;
- `phase-3-test-plan.md`;
- `phase-4-test-plan.md`;
- `amendment-2-test-plan.md`;
- `amendment-3-test-plan.md`; or
- `configs/full_pipeline.toml`.

## 3. Comparison policy and exact shared Rust inputs

All input blocks below are complete compiler input. When a case names a shared
input, it uses those exact bytes; ellipses are never implicit Rust.

As a baseline subtest, pass every `A4-SRC-*` block independently to the pinned
compiler before any Amendment-4 assertion and require successful parsing,
resolution, and type checking. A planned generation error is therefore about
the synthesized target type, never a pre-existing source error. Do not combine
blocks into one crate, because crate attributes and module scopes are part of
their exact oracles.

Compare parsed AST types when formatting is irrelevant. Exact expected
spellings below are semantic oracles: path segment count, leading `crate::` or
`::`, raw spelling, aliases, generic nesting, mutability, and lifetime names
must match. Presentation-only binding `mut`, ordinary whitespace, spans,
`NodeId`s, and token caches are ignored unless explicitly shown as part of a
complete signature.

Every generated `annotated_source` and `annotated_skeleton` must parse as a
complete function item. `source_signature` and `target_signature` are
bodyless headers; append an empty dummy body before passing either one to the
item parser. No expected target type contains the invalid diagnostic-relative
path `src::lib::cb_rgb`.

### A4-SRC-MOTIVATING

This is the minimal semantic shape of the observed
`B01_organic/tritanopia_lib` failure. The foreign call makes the enclosing
statement require transformation, so skeletonization visits and types the
nested `init` binding.

```rust
unsafe extern "C" {
    fn transform(value: f64) -> f64;
}

pub mod src {
    pub mod lib {
        #[derive(Clone, Copy)]
        pub struct cb_rgb {
            pub r: f32,
            pub g: f32,
            pub b: f32,
        }

        pub unsafe fn cb_remove_gamma_rgb(rgb: cb_rgb) -> cb_rgb {
            let result = {
                let init = cb_rgb {
                    r: crate::transform(rgb.r as f64) as f32,
                    g: crate::transform(rgb.g as f64) as f32,
                    b: crate::transform(rgb.b as f64) as f32,
                };
                init
            };
            result
        }
    }
}
```

### A4-SRC-IMPORTS

```rust
pub mod model {
    #[repr(C)]
    pub struct Direct(pub i32);

    #[repr(C)]
    pub struct Renamed(pub i32);

    #[repr(C)]
    pub struct Globbed(pub i32);
}

pub mod direct {
    use crate::model::Direct;

    pub unsafe fn make() -> i32 {
        let value = core::mem::zeroed::<Direct>();
        value.0
    }
}

pub mod renamed {
    use crate::model::Renamed as R;

    pub unsafe fn make() -> i32 {
        let value = core::mem::zeroed::<R>();
        value.0
    }
}

pub mod globbed {
    use crate::model::*;

    pub unsafe fn make() -> i32 {
        let value = core::mem::zeroed::<Globbed>();
        value.0
    }
}
```

`core::mem::zeroed` is an unsafe non-local call, so each `let value` statement
is transformed rather than hidden inside an Amendment-2 preserved subtree.

### A4-SRC-CANDIDATES

```rust
pub mod left {
    #[repr(C)]
    pub struct Thing {
        pub value: i32,
    }
}

pub mod right {
    #[repr(C)]
    pub struct Thing {
        pub value: i32,
    }
}

pub mod aliases {
    use crate::left::Thing as Zed;
    use crate::left::Thing as Alpha;

    pub unsafe fn inferred() -> i32 {
        let value = core::mem::zeroed::<crate::left::Thing>();
        value.value
    }

    pub unsafe fn source_hint(pointer: *const Zed) -> i32 {
        (*pointer).value
    }
}

pub mod collision {
    use crate::right::Thing;

    pub unsafe fn inferred() -> i32 {
        let value = core::mem::zeroed::<crate::left::Thing>();
        value.value
    }

    pub unsafe fn use_right(value: Thing) -> i32 {
        value.value
    }
}
```

### A4-SRC-CANDIDATE-PRECEDENCE

```rust
pub mod model {
    #[repr(C)]
    pub struct Item {
        pub value: i32,
    }
}

pub mod own {
    #[repr(C)]
    pub struct Local {
        pub value: i32,
    }

    use self::Local as Alias;

    pub unsafe fn inferred() -> i32 {
        let value = core::mem::zeroed::<Local>();
        value.value
    }

    pub unsafe fn source(pointer: *const Alias) -> i32 {
        (*pointer).value
    }
}

pub mod transparent {
    pub type Transparent = crate::model::Item;

    pub unsafe fn inferred() -> i32 {
        let value = core::mem::zeroed::<crate::model::Item>();
        value.value
    }
}

pub mod namespace {
    use crate::model::Item as Name;

    #[allow(non_upper_case_globals)]
    pub const Name: i32 = 7;

    pub unsafe fn inferred() -> i32 {
        let value = core::mem::zeroed::<Name>();
        value.value
    }
}
```

### A4-SRC-REEXPORTS

The local type is defined under a private module but exposed through a public
re-export. `DefaultHasher` is externally exposed as
`std::hash::DefaultHasher` while its defining `std::hash::random` module is
private.

```rust
pub mod api {
    mod hidden {
        #[repr(C)]
        pub struct Public {
            pub value: i32,
        }
    }

    pub use hidden::Public as Exposed;
}

pub mod consumer {
    pub mod std {}

    pub unsafe fn local() -> i32 {
        let value = core::mem::zeroed::<crate::api::Exposed>();
        value.value
    }

    pub unsafe fn external() -> usize {
        let value = ::std::hash::DefaultHasher::new();
        ::core::mem::size_of_val(&value)
    }
}
```

### A4-SRC-LOCAL-FALLBACK-ROUTES

```rust
pub(crate) mod restricted_api {
    mod hidden {
        #[repr(C)]
        pub(crate) struct Restricted {
            pub(crate) value: i32,
        }
    }

    pub(crate) use hidden::Restricted as Exposed;
}

pub mod short {
    mod hidden {
        #[repr(C)]
        pub struct Short {
            pub value: i32,
        }
    }

    pub use hidden::Short as S;
}

pub mod longer {
    pub mod route {
        pub use crate::short::S;
    }
}

pub mod alpha {
    mod hidden {
        #[repr(C)]
        pub struct Tie {
            pub value: i32,
        }
    }

    pub use hidden::Tie as T;
}

pub mod beta {
    pub use crate::alpha::T;
}

pub mod consumer {
    pub unsafe fn restricted() -> i32 {
        let value =
            core::mem::zeroed::<crate::restricted_api::Exposed>();
        value.value
    }

    pub unsafe fn shortest() -> i32 {
        let value = core::mem::zeroed::<crate::short::S>();
        value.value
    }

    pub unsafe fn tie() -> i32 {
        let value = core::mem::zeroed::<crate::alpha::T>();
        value.value
    }
}
```

### A4-SRC-EXTERNAL-ROOT-ALIAS

```rust
#![no_std]

extern crate std as rust_std;
extern crate std as alt_std;

pub mod consumer {
    pub unsafe fn external_alias() -> usize {
        let value = rust_std::hash::DefaultHasher::new();
        core::mem::size_of_val(&value)
    }
}
```

### A4-SRC-SOURCE-PATHS

```rust
pub mod model {
    #[repr(C)]
    pub struct Point {
        pub value: i32,
    }

    pub type PointAlias = Point;
}

pub mod consumer {
    use crate::model::PointAlias as P;
    use crate::model::PointAlias as LocalP;
    use crate::model::PointAlias as ReturnP;

    pub unsafe fn alias(pointer: *const P) -> i32 {
        (*pointer).value
    }

    pub unsafe fn local_alias(pointer: *const P) -> i32 {
        let local: *const LocalP = pointer;
        (*local).value
    }

    pub unsafe fn alias_id(pointer: *const P) -> *const ReturnP {
        pointer
    }

    pub unsafe fn relative(pointer: *const super::model::Point) -> i32 {
        (*pointer).value
    }
}
```

### A4-SRC-SOURCE-HINT-EDGES

```rust
pub mod model {
    #[repr(C)]
    pub struct Point {
        pub value: i32,
    }

    pub type PointAlias = Point;
    pub type PointPtr = *const Point;
}

pub mod consumer {
    use crate::model::PointAlias as P;

    pub unsafe fn qualified_alias(
        pointer: *const crate::model::PointAlias,
    ) -> i32 {
        (*pointer).value
    }

    pub unsafe fn optional_alias(pointer: *const P) -> i32 {
        if pointer.is_null() {
            0
        } else {
            (*pointer).value
        }
    }

    pub unsafe fn explicit_nominal() -> i32 {
        let value: crate::model::PointAlias =
            core::mem::zeroed::<crate::model::PointAlias>();
        value.value
    }

    pub unsafe fn hidden_pointer_alias(
        pointer: crate::model::PointPtr,
    ) -> i32 {
        (*pointer).value
    }
}
```

### A4-SRC-DIRECT-HINTS

```rust
#[repr(C)]
pub struct P {
    pub value: i32,
}

pub unsafe fn hint(pointer: *const P) -> i32 {
    (*pointer).value
}
```

### A4-SRC-RECURSIVE-TYPES

```rust
pub const WIDTH: usize = 2;

pub struct Wrap<T>(pub T);

pub type Callback =
    unsafe fn(*const i32) -> *const i32;

pub type CCallback =
    unsafe extern "C" fn(*const i32) -> *const i32;

pub unsafe fn grammar(
    singleton: (Wrap<(*const i32, &'static [i32])>,),
    array: [Wrap<i32>; WIDTH],
    callback: Callback,
    c_callback: CCallback,
) -> usize {
    let _ = singleton;
    let _ = array;
    core::mem::size_of_val(&callback)
        + core::mem::size_of_val(&c_callback)
}

pub unsafe fn higher_ranked(
    callback: for<'a> fn(&'a i32) -> &'a i32,
    value: &i32,
) -> i32 {
    *callback(value)
}
```

### A4-SRC-POINTERS

```rust
#[repr(C)]
pub struct Node {
    pub value: i32,
}

pub unsafe fn update_and_return(pointer: *mut Node) -> *mut Node {
    (*pointer).value += 1;
    pointer
}

pub unsafe fn local_pointer() -> i32 {
    let mut node = Node { value: 1 };
    let pointer = &mut node as *mut Node;
    (*pointer).value += 1;
    node.value
}
```

### A4-SRC-COMPOUND

```rust
pub mod types {
    #[repr(C)]
    pub struct A {
        pub value: i32,
    }

    #[repr(C)]
    pub struct B {
        pub value: i32,
    }
}

pub mod consumer {
    use crate::types::A as Alpha;
    use crate::types::B;

    pub unsafe fn mutate(pointer: *mut (Alpha, [B; 2])) {
        (*pointer).0.value += (*pointer).1[0].value;
    }

    pub unsafe fn local() -> i32 {
        let mut value = (
            Alpha { value: 1 },
            [B { value: 2 }, B { value: 3 }],
        );
        let pointer = &mut value as *mut (Alpha, [B; 2]);
        (*pointer).0.value += 1;
        value.0.value
    }
}
```

### A4-SRC-RAW-IDENTIFIERS

```rust
pub mod r#type {
    #[repr(C)]
    pub struct r#match {
        pub value: i32,
    }

    pub unsafe fn read(pointer: *const r#match) -> i32 {
        (*pointer).value
    }

    pub unsafe fn inferred() -> i32 {
        let value = core::mem::zeroed::<r#match>();
        value.value
    }
}
```

### A4-SRC-QUALIFIED-RAW-FALLBACK

```rust
pub mod r#type {
    #[repr(C)]
    pub struct r#match {
        pub value: i32,
    }
}

pub mod consumer {
    pub unsafe fn inferred() -> i32 {
        let value =
            core::mem::zeroed::<crate::r#type::r#match>();
        value.value
    }
}
```

### A4-SRC-STANDARD-CONSTRUCTORS

```rust
pub mod wrapped {
    unsafe extern "C" {
        fn malloc(size: usize) -> *mut i32;
    }

    pub unsafe fn read(p: *const i32) -> i32 {
        if p.is_null() {
            0
        } else {
            *p
        }
    }

    pub unsafe fn owned_id(mut p: *mut i32) -> *mut i32 {
        p
    }

    pub unsafe fn foo() -> *mut i32 {
        let p: *mut i32 =
            malloc(core::mem::size_of::<i32>());
        *p = 7;
        let q: *mut i32 = owned_id(p);
        q
    }

    pub unsafe fn allocate() -> *mut i32 {
        let p: *mut i32 =
            malloc(core::mem::size_of::<i32>());
        *p = 7;
        p
    }
}
```

### A4-SRC-STANDARD-BARE-IMPORTS

```rust
pub mod imported {
    use core::option::Option;
    use std::boxed::Box;

    unsafe extern "C" {
        fn malloc(size: usize) -> *mut i32;
    }

    pub unsafe fn read(p: *const i32) -> i32 {
        if p.is_null() {
            0
        } else {
            *p
        }
    }

    pub unsafe fn allocate() -> *mut i32 {
        let p: *mut i32 =
            malloc(core::mem::size_of::<i32>());
        *p = 7;
        p
    }
}
```

### A4-SRC-NO-STD-OPTION-SUCCESS

The ordinary core prelude remains enabled and supplies the standard
`Option`, even though owned `Box` is not in that prelude.

```rust
#![no_std]

pub unsafe fn read(p: *const i32) -> i32 {
    if p.is_null() {
        0
    } else {
        *p
    }
}
```

### A4-SRC-NAMED-OPTIONAL-BOX

```rust
pub mod model {
    #[repr(C)]
    pub struct Point {
        pub value: i32,
    }

    pub type PointAlias = Point;
}

pub mod consumer {
    use crate::model::PointAlias as P;
    use crate::model::PointAlias as LocalP;
    use crate::model::PointAlias as LocalQ;
    use crate::model::PointAlias as ReturnP;

    unsafe extern "C" {
        fn malloc(size: usize) -> *mut LocalP;
    }

    pub unsafe fn owned_id(mut p: *mut P) -> *mut ReturnP {
        p
    }

    pub unsafe fn foo() -> *mut ReturnP {
        let p: *mut LocalP =
            malloc(core::mem::size_of::<LocalP>());
        (*p).value = 7;
        let q: *mut LocalQ = owned_id(p);
        q
    }
}
```

### A4-SRC-OPTION-COLLISION

The explicit standard alias does not make the source supported: the bare
identifier still resolves to the user definition.

```rust
pub mod wrapped {
    pub struct Option;
    use core::option::Option as Maybe;

    pub unsafe fn read(
        first: *const i32,
        second: *const i32,
    ) -> i32 {
        if first.is_null() {
            if second.is_null() {
                0
            } else {
                *second
            }
        } else {
            *first
        }
    }
}
```

### A4-SRC-BOX-COLLISION

The explicit standard alias does not make the source supported: the bare
identifier still resolves to the user definition.

```rust
pub mod wrapped {
    pub struct Box;
    use std::boxed::Box as Owned;

    unsafe extern "C" {
        fn malloc(size: usize) -> *mut i32;
    }

    pub unsafe fn allocate() -> *mut i32 {
        let p: *mut i32 =
            malloc(core::mem::size_of::<i32>());
        *p = 7;
        p
    }
}
```

### A4-SRC-RENAMED-CONSTRUCTOR-COLLISION

```rust
pub mod fake {
    pub struct WrongOption;
}

pub mod renamed {
    use crate::fake::WrongOption as Option;

    pub unsafe fn read(p: *const i32) -> i32 {
        if p.is_null() {
            0
        } else {
            *p
        }
    }
}
```

### A4-SRC-GLOB-CONSTRUCTOR-COLLISION

```rust
pub mod fake {
    pub mod glob {
        pub struct Box;
    }
}

pub mod globbed {
    use crate::fake::glob::*;

    unsafe extern "C" {
        fn malloc(size: usize) -> *mut i32;
    }

    pub unsafe fn allocate() -> *mut i32 {
        let p: *mut i32 =
            malloc(core::mem::size_of::<i32>());
        *p = 7;
        p
    }
}
```

### A4-SRC-OPTBOX-PARTIAL-CONSTRUCTOR-COLLISION

```rust
pub mod wrapped {
    pub struct Box;
    use core::option::Option;

    unsafe extern "C" {
        fn malloc(size: usize) -> *mut i32;
    }

    pub unsafe fn owned_id(mut p: *mut i32) -> *mut i32 {
        p
    }

    pub unsafe fn foo() -> *mut i32 {
        let p: *mut i32 =
            malloc(core::mem::size_of::<i32>());
        *p = 7;
        let q: *mut i32 = owned_id(p);
        q
    }
}
```

### A4-SRC-LOCAL-BOX-COLLISION

The transformed pointer local is owned, while the function signature contains
no pointer. This pins the local-specific error location independently of
parameter and return failures.

```rust
pub mod consumer {
    pub struct Box;

    unsafe extern "C" {
        fn malloc(size: usize) -> *mut i32;
        fn free(pointer: *mut core::ffi::c_void);
    }

    pub unsafe fn local_only() -> i32 {
        let first: *mut i32 =
            malloc(core::mem::size_of::<i32>());
        *first = 7;
        let second: *mut i32 =
            malloc(core::mem::size_of::<i32>());
        *second = 11;
        let value = *first + *second;
        free(first as *mut core::ffi::c_void);
        free(second as *mut core::ffi::c_void);
        value
    }
}
```

### A4-SRC-EXTERN-PRELUDE-CONSTRUCTOR-COLLISION

The root `extern crate` alias is not a child of the nested `wrapped` module,
but rustc also inserts it into the extern prelude. Bare `Option` in `wrapped`
therefore resolves to the external crate alias before the standard prelude.

```rust
extern crate core as Option;

pub mod wrapped {
    pub unsafe fn read(p: *const i32) -> i32 {
        if p.is_null() {
            0
        } else {
            *p
        }
    }
}
```

### A4-SRC-IRRELEVANT-COLLISIONS

Each module shadows a constructor that its selected target kind does not use.

```rust
pub mod box_only {
    pub struct Option;

    unsafe extern "C" {
        fn malloc(size: usize) -> *mut i32;
    }

    pub unsafe fn allocate() -> *mut i32 {
        let p: *mut i32 =
            malloc(core::mem::size_of::<i32>());
        *p = 7;
        p
    }
}

pub mod option_only {
    pub struct Box;

    pub unsafe fn read(p: *const i32) -> i32 {
        if p.is_null() {
            0
        } else {
            *p
        }
    }
}
```

### A4-SRC-NO-IMPLICIT-PRELUDE-REJECTION

```rust
#![no_implicit_prelude]

extern crate core;

pub mod wrapped {
    use crate::core::option::Option;

    pub unsafe fn read(p: *const i32) -> i32 {
        if p.is_null() {
            0
        } else {
            *p
        }
    }
}
```

### A4-SRC-NO-STD-BOX-REJECTION

`#![no_std]` keeps the ordinary core prelude but does not put owned `Box` in
that prelude. The exact raw-pointer input compiles; skeleton generation must
diagnose the unresolved introduced bare constructor.

```rust
#![no_std]

unsafe extern "C" {
    fn malloc(size: usize) -> *mut i32;
}

pub unsafe fn allocate() -> *mut i32 {
    let p: *mut i32 =
        malloc(core::mem::size_of::<i32>());
    *p = 7;
    p
}
```

### A4-SRC-BOX-NO-IMPLICIT-PRELUDE-REJECTION

```rust
#![no_implicit_prelude]

extern crate core;

unsafe extern "C" {
    fn malloc(size: usize) -> *mut i32;
}

pub unsafe fn allocate() -> *mut i32 {
    let p: *mut i32 =
        malloc(core::mem::size_of::<i32>());
    *p = 7;
    p
}
```

### A4-SRC-MODULE-NO-IMPLICIT-PRELUDE-REJECTION

```rust
#[no_implicit_prelude]
pub mod wrapped {
    pub unsafe fn read(p: *const i32) -> i32 {
        if p.is_null() {
            0
        } else {
            *p
        }
    }
}
```

### A4-SRC-ANCESTOR-NO-IMPLICIT-PRELUDE-REJECTION

```rust
#[no_implicit_prelude]
pub mod outer {
    pub mod middle {
        pub mod inner {
            pub unsafe fn read(p: *const i32) -> i32 {
                if p.is_null() {
                    0
                } else {
                    *p
                }
            }
        }
    }
}
```

### A4-SRC-PRESERVED-PARENT

```rust
pub struct Local {
    pub value: i32,
}

pub unsafe fn preserved(flag: bool) -> i32 {
    if flag {
        let value = Local { value: 1 };
        value.value
    } else {
        0
    }
}
```

### A4-SRC-UNNAMEABLE

This compiler-valid robustness input is outside the supported
source-written-generic model. Its inferred local has an opaque semantic type
that cannot be written as an ordinary local annotation.

```rust
pub fn values() -> impl Iterator<Item = i32> {
    0..3
}

pub unsafe fn consume() {
    let iterator = values();
    core::mem::drop(iterator);
}
```

### A4-SRC-TREE

This is the exact completed Phase-1 initial-decision regression input.

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

### A4-SRC-COMPREHENSIVE

This is the exact completed Phase-1 comprehensive-fixture input.

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

## 4. Existing regression updates

### A4-UPDATE-01 `same_module_pointer_targets_supersede_crate_qualified_skeleton_oracles`

Use exact inputs A4-SRC-TREE and A4-SRC-COMPREHENSIVE.

Update the existing implementation test
`uses_initial_decisions_before_rewriter_fallback_demotion` so the tools
skeleton target signature is exactly:

```rust
pub unsafe fn tree_print_helper(
    mut tree: &mut Tree,
    mut root_id: i32,
)
```

The ordinary pointer rewriter assertion in the same test remains exactly
crate-qualified and raw:

```rust
pub unsafe fn tree_print_helper(
    mut tree: *mut crate::Tree,
    root_id: i32,
)
```

Formatting may be normalized to one line, but the skeleton type has one
segment `Tree`, while the ordinary rewriter retains `crate::Tree`.

Update `comprehensive_fixture_emits_consistent_records` so
`model::read`'s target parameter is exactly:

```rust
p: &Point
```

It is no longer `p: &crate::model::Point`. Keep every record ID, path,
dependency list, label map, and other comprehensive-fixture assertion
unchanged.

### A4-UPDATE-02 `existing_pointer_and_protocol_regressions_change_only_rendered_tools_types`

Use exact inputs A4-SRC-TREE and A4-SRC-COMPREHENSIVE.

Run the existing target-type, JSON, validator, item-replacer, and pointer
rewriter regressions. Update only exact tools-skeleton type strings made
shorter by Amendment 4. In particular:

- `InitialPointerDecisions`, `PtrKind`, and lifetime plans are identical before
  and after the spelling change;
- ordinary `pointer_replacer::replace_local_borrows` output remains unchanged;
- function-record keys and JSON key order remain unchanged;
- validation and replacement request/response JSON remain schema version 1;
- Python records and prompts receive only the corrected Rust string; and
- no validator path-equivalence rule is added.

This is an explicit update matrix, not a request to alter historical test-plan
files.

## 5. Inferred-local scope-spelling tests

### A4-INFER-01 `nested_same_module_inferred_local_uses_local_name`

Use exact input A4-SRC-MOTIVATING.

Expected `src::lib::cb_remove_gamma_rgb` target skeleton fragments are:

```rust
pub unsafe fn cb_remove_gamma_rgb(
    mut rgb: cb_rgb,
) -> cb_rgb
```

```rust
let mut result: cb_rgb = {
    let mut init: cb_rgb = todo!();
    init
};
```

The existing labels remain on `result`, `init`, and `init`'s use. Assert:

```text
contains "let mut init: cb_rgb"
does not contain "src::lib::cb_rgb"
does not contain "crate::src::lib::cb_rgb"
```

The same-module binding is preferred over the absolute fallback. The source
rendering continues to show the inferred `let mut init = cb_rgb { ... };`
without an inserted source annotation.

### A4-INFER-02 `direct_renamed_and_glob_imports_name_inferred_locals`

Use exact input A4-SRC-IMPORTS.

Expected target-local declarations are:

```text
direct::make  -> let mut value: Direct = todo!();
renamed::make -> let mut value: R = todo!();
globbed::make -> let mut value: Globbed = todo!();
```

For each path, assert that the chosen one-segment type resolves through the
containing module's type namespace to the exact nominal `DefId`. Do not accept
`crate::model::Direct`, `crate::model::Renamed`,
`crate::model::Globbed`, or the defining module's bare name when the only
available local spelling is the alias `R`.

### A4-INFER-03 `multiple_aliases_are_deterministic_and_source_hint_wins`

Use exact inputs A4-SRC-CANDIDATES and
A4-SRC-CANDIDATE-PRECEDENCE.

For `aliases::inferred`, there is no source local type hint. Both `Alpha` and
`Zed` resolve to `left::Thing`, so the exact inferred local type is:

```rust
let mut value: Alpha = todo!();
```

Run generation twice in independent compiler callbacks and require identical
records and serialized JSON bytes.

For `aliases::source_hint`, the source pointee is already the one-segment path
`Zed`, so the target signature uses:

```rust
mut pointer: &Zed
```

It must not change to the lexicographically earlier `Alpha`. This pins the
source-hint priority over general deterministic alias selection.

For A4-SRC-CANDIDATE-PRECEDENCE, require:

```text
own::inferred local value       = Local
own::source parameter pointer   = &Alias
transparent::inferred local     = crate::model::Item
namespace::inferred local       = Name
```

`own::inferred` proves that an own definition wins over an imported alias for
semantic rendering. `own::source` independently proves that a source-written
alias wins for a reused component. `transparent::inferred` must not become
`Transparent`: it has no exact nominal one-segment binding in scope, and a
transparent alias with the same normalized semantic type is not a candidate
for the nominal `Item` `DefId`, so the local fallback is
`crate::model::Item`. In `namespace`, both the renamed type import and the
value constant are spelled `Name`; the type-namespace binding remains valid
and the value-namespace collision must not suppress it. Resolve every accepted
path through the type namespace to the expected `DefId`.

### A4-INFER-04 `wrong_same_spelling_binding_requires_absolute_local_fallback`

Use exact input A4-SRC-CANDIDATES.

In `collision`, the one-segment `Thing` resolves to `right::Thing`, not
`left::Thing`. The exact inferred declaration in `collision::inferred` is:

```rust
let mut value: crate::left::Thing = todo!();
```

It must not be `Thing`, `right::Thing`, or unrooted `left::Thing`.
`collision::use_right` continues to use its unchanged source type `Thing`.
This proves candidate matching is by type-namespace `DefId`, not spelling.

### A4-INFER-05 `local_visible_fallback_uses_public_reexport_not_private_definition_path`

Use exact inputs A4-SRC-REEXPORTS and A4-SRC-LOCAL-FALLBACK-ROUTES.

`consumer` has no one-segment binding for the local nominal type. The exact
target-local declaration in `consumer::local` is:

```rust
let mut value: crate::api::Exposed = todo!();
```

It must not expose a definition path through the private module:

```rust
crate::api::hidden::Public
```

The test obtains the nominal `DefId` from the inferred local, confirms that the
public re-export path resolves to that same definition, and confirms that the
private defining path is not accessible from `consumer`. This is a semantic
identity/accessibility assertion, not only a rendered-string assertion.

For A4-SRC-LOCAL-FALLBACK-ROUTES, require:

```text
consumer::restricted local value = crate::restricted_api::Exposed
consumer::shortest local value   = crate::short::S
consumer::tie local value        = crate::alpha::T
```

The first route is `pub(crate)` rather than `pub`; it is usable because the
containing module is within that visibility. For `Short`, both
`crate::short::S` and `crate::longer::route::S` resolve to the same definition,
so the fewer-segment path wins. For `Tie`, `crate::alpha::T` and
`crate::beta::T` have equal segment counts, so the lexicographically smaller
complete path wins. Validate identity and accessibility for every route and
also enumerate the losing valid route, so the test actually pins the ordering
rule.

### A4-INFER-06 `external_visible_fallback_is_absolute_and_uses_public_reexport`

Use exact inputs A4-SRC-REEXPORTS and A4-SRC-EXTERNAL-ROOT-ALIAS.

The exact target-local declaration in `consumer::external` is:

```rust
let mut value: ::std::hash::DefaultHasher =
    ::std::hash::DefaultHasher::new();
```

Amendment 2 may preserve the initializer, so compare the complete canonical
declaration rather than expecting `todo!()`. The type path must begin with
`::std`, resolve to the inferred `DefaultHasher` definition, and use the
public `std::hash::DefaultHasher` re-export. It must not expose the private
defining path `std::hash::random::DefaultHasher` and must not be an unrooted
`std::hash::DefaultHasher` that a local `std` module could capture.

This regression uses the standard-library dependency already present in every
compiler callback; it adds no Cargo fixture or external download.

For A4-SRC-EXTERNAL-ROOT-ALIAS, the exact target local is:

```rust
let mut value: ::alt_std::hash::DefaultHasher =
    rust_std::hash::DefaultHasher::new();
```

The source is `#![no_std]` and explicitly makes the external crate
source-visible as both `rust_std` and `alt_std`. Both complete fallback paths
have the same segment count, so the exact deterministic winner is the
lexicographically smaller `::alt_std::hash::DefaultHasher`. Require its leading
segment to resolve through that exact extern-crate alias before walking the
external re-export path. Do not silently substitute
`::std::hash::DefaultHasher`, select `::rust_std::...` for the synthesized type
merely by copying the initializer root, emit an unrooted alias path, or expose
the private defining path. Keep the source initializer itself unchanged.
Enumerate all source-visible extern roots for the target crate, then choose the
valid complete path by segment count and lexicographic order. This input uses
only the pinned sysroot `std`; it creates no Cargo dependency and performs no
download.

## 6. Pointer-introduced type-spelling tests

### A4-PTR-01 `source_alias_and_relative_pointee_paths_are_reused`

Use exact inputs A4-SRC-SOURCE-PATHS and A4-SRC-SOURCE-HINT-EDGES.

Before spelling assertions, obtain tools-mode `initial_pointer_decisions` and
locate subjects by `DefId`/HIR identity. Require:

```text
consumer::local_alias input pointer = Ref(false)
consumer::local_alias binding local = Ref(false)
consumer::alias_id input pointer = Ref(false)
consumer::alias_id output = Ref(false)
```

Also require the explicitly typed `let local` statement's label to be in
`statements_requiring_transformation`. A decision or disposition mismatch is a
pointer-analysis/preservation regression and must stop the spelling oracle.
For `consumer::alias_id`, require the exact lifetime plan:

```text
input_lifetimes[0] = Some("a")
output_lifetime = Some("a")
input_lifetimes[0] == output_lifetime
```

The equality assertion is semantic: both returned-borrow slots must carry the
same generated `Symbol`, not merely two strings that happen to print alike.

Expected target signatures are:

```rust
pub unsafe fn alias(mut pointer: &P) -> i32
```

```rust
pub unsafe fn local_alias(mut pointer: &P) -> i32
```

```rust
pub unsafe fn alias_id<'a>(
    mut pointer: &'a P,
) -> &'a ReturnP
```

```rust
pub unsafe fn relative(
    mut pointer: &super::model::Point,
) -> i32
```

The exact target local in `consumer::local_alias` is:

```rust
let mut local: &LocalP = todo!();
```

The alias path resolves to the `PointAlias` definition and remains `P`; Crat
does not replace it with the normalized underlying `Point`. Assert this
independently for the parameter source hint `P`, the explicit return source
hint `ReturnP`, and the explicitly typed local source hint `LocalP` handled by
`Skeletonizer::flat_map_stmt`. These three bindings resolve to the same
`PointAlias` `DefId` but have intentionally distinct source spellings. The
explicit return of `alias_id` must not reuse the input's `P`, and
`local_alias`'s local must not reuse its parameter spelling; none may become
`Point`, `crate::model::Point`, or another qualified path to the defining
type. No one-segment binding for `model::Point` exists in `consumer`, so the
already valid relative path remains instead of being changed to `Point` or
`crate::model::Point`.

This separates four paths through the implementation: parameter source-AST
reuse (`alias`), explicit-return source-AST reuse (`alias_id`), explicitly
typed local source-AST reuse (`local_alias`), and semantic rendering of
inferred locals covered by A4-PTR-03.

For A4-SRC-SOURCE-HINT-EDGES, first require:

```text
consumer::qualified_alias input pointer     = Ref(false)
consumer::optional_alias input pointer      = OptRef(false)
consumer::hidden_pointer_alias input pointer = Ref(false)
```

Then require these exact target fragments:

```rust
pub unsafe fn qualified_alias(
    mut pointer: &P,
) -> i32
```

```rust
pub unsafe fn optional_alias(
    mut pointer: Option<&P>,
) -> i32
```

```rust
let mut value: crate::model::PointAlias = todo!();
```

```rust
pub unsafe fn hidden_pointer_alias(
    mut pointer: &crate::model::Point,
) -> i32
```

The qualified source pointee path resolves to the `PointAlias` definition and
is shortened through its in-scope alias `P`; source-AST-first does not mean
“retain qualification unconditionally.” The `OptRef` target proves the same
alias policy applies beneath an introduced wrapper. The explicitly typed
nominal local is in a transformed statement but its complete non-pointer type
is unchanged, so it retains its valid source spelling rather than being
gratuitously shortened. Finally, `PointPtr` hides the pointer itself and
cannot be peeled as a pointee hint. Its normalized pointee is `model::Point`;
`P` resolves to the distinct `PointAlias` definition, so semantic fallback
must use `crate::model::Point`, not `P`, `PointPtr`, or
`crate::model::PointPtr`.

### A4-PTR-02 `same_module_parameter_return_local_and_lifetime_types_are_short`

Use exact input A4-SRC-POINTERS.

Expected target signature:

```rust
pub unsafe fn update_and_return<'a>(
    mut pointer: &'a mut Node,
) -> &'a mut Node
```

Expected target-local type map for `local_pointer`:

```text
node    = Node
pointer = &mut Node
```

The source signature and source local continue to use raw pointers. Assert that
no target field for either function contains `crate::Node`. Existing lifetime
identity and mutability assertions remain unchanged; Amendment 4 changes only
the nominal path component.

### A4-PTR-03 `compound_pointees_and_inferred_pointer_locals_recurse`

Use exact inputs A4-SRC-COMPOUND, A4-SRC-DIRECT-HINTS, and
A4-SRC-RECURSIVE-TYPES.

Expected target parameter type for `consumer::mutate`:

```rust
&mut (Alpha, [B; 2])
```

Expected target-local types for `consumer::local`:

```text
value   = (Alpha, [B; 2])
pointer = &mut (Alpha, [B; 2])
```

The parameter exercises a syntactic source pointee hint. The inferred
`pointer` local exercises semantic recursive rendering without an explicit
source type. Both must select `Alpha` and `B` independently by exact `DefId`
and preserve tuple, array, length, and reference mutability.

Using the mapped `*const P` parameter from A4-SRC-DIRECT-HINTS, call the
low-level source-hint target builder directly for every `PtrKind`. Use no
generated lifetime for this table and require these exact parsed AST types:

| `PtrKind` | Exact target type |
| --- | --- |
| `Ref(false)` | `&P` |
| `Ref(true)` | `&mut P` |
| `OptRef(false)` | `Option<&P>` |
| `OptRef(true)` | `Option<&mut P>` |
| `Box` | `Box<P>` |
| `OptBox` | `Option<Box<P>>` |
| `Raw(false)` | `*const P` |
| `Raw(true)` | `*mut P` |
| `BoxedSlice` | `Box<[P]>` |
| `OptBoxedSlice` | `Option<Box<[P]>>` |
| `Slice(false)` | `&[P]` |
| `Slice(true)` | `&mut [P]` |
| `SliceCursor(false)` | `crate::slice_cursor::SliceCursor<'_, P>` |
| `SliceCursor(true)` | `crate::slice_cursor::SliceCursorMut<'_, P>` |

The `Option`/`Box` rows run with the default ordinary prelude and first satisfy
the same constructor precondition as production. This is a builder test, not
a claim that pointer analysis selects every kind for one source function.
Every nominal `P` node is the cloned source path and resolves to the local
`P` definition.

Use A4-SRC-RECURSIVE-TYPES for compiler-backed low-level formatter tests. It
must render the semantic forms below without flattening nested components:

```text
(Wrap<(*const i32, &'static [i32])>,)
[Wrap<i32>; 2]
unsafe fn(*const i32) -> *const i32
unsafe extern "C" fn(*const i32) -> *const i32
```

The singleton tuple retains its comma. Generic arguments, raw pointers,
reference lifetime, slice, array, callable safety, ABI, inputs, and output are
all recursive assertions, not one opaque string comparison. The two callback
parameters independently pin the default Rust ABI and explicit C ABI.
Separately clone the source AST array component and prove source-AST-first
retains the named length `[Wrap<i32>; WIDTH]`, since semantic rendering is
allowed to evaluate it to `2`.

For the `for<'a> fn(&'a i32) -> &'a i32` parameter, require the planned
`GenerationErrorKind::TypeSpelling` unsupported-binder result. Amendment 4
does not expand the formatter grammar to higher-ranked binders and must not
erase `for<'a>` or its region relation. Preserve the ordinary unsafe function
pointer above; return `TypeSpelling` for the higher-ranked binder until a
separate amendment adds faithful support.

Finally, table-test the shared fallible core independently of skeleton
generation:

- the legacy `mir_ty_to_string(ty, tcx)` wrapper returns these exact
  pre-amendment strings for the corresponding semantic types:

  ```text
  singleton = (crate::Wrap<(*const i32, &[i32])>)
  array      = [crate::Wrap<i32>; 2]
  callback   = unsafe fn(*const i32) -> *const i32
  c_callback = unsafe extern "C" fn(*const i32) -> *const i32
  ```

  The missing singleton comma and omitted `'static` in this legacy table are
  intentional compatibility oracles; only the tools policy emits the
  source-correct forms above;
- the scope-aware hook changes only nominal path leaves;
- a deliberately invalid path result from a test selector returns a parse/
  `TypeSpelling` error instead of reaching unchecked `parse_ty`; and
- under the tools unsupported-shape policy, every unsupported semantic branch
  returns an error rather than calling `todo!()`, `unreachable!()`, or
  `panic!()`.

### A4-PTR-04 `raw_identifiers_remain_parseable_in_inferred_and_pointer_types`

Use exact inputs A4-SRC-RAW-IDENTIFIERS and
A4-SRC-QUALIFIED-RAW-FALLBACK.

Expected target fragments:

```rust
pub unsafe fn read(
    mut pointer: &r#match,
) -> i32
```

```rust
let mut value: r#match = todo!();
```

The function record path remains `r#type::read` or
`r#type::inferred` as applicable. Neither target contains an unescaped
type-position keyword `match`, and both annotated snippets parse.

For A4-SRC-QUALIFIED-RAW-FALLBACK, `consumer` has no one-segment binding.
Require:

```rust
let mut value: crate::r#type::r#match = todo!();
```

The `crate` root is not raw, while each keyword segment is emitted as the
resolver-provided raw identifier. Parse the type and walk the type namespace
to the exact `r#match` `DefId`; a string that merely happens to parse is
insufficient.

### A4-PTR-05 `standard_constructors_require_exact_bare_prelude_resolution`

First unit-test the pure constructor-requirement helper with this exhaustive
table. Include both boolean values for every payload-bearing variant:

```text
Ref(false)            -> Option=no,  Box=no
Ref(true)             -> Option=no,  Box=no
Raw(false)            -> Option=no,  Box=no
Raw(true)             -> Option=no,  Box=no
Slice(false)          -> Option=no,  Box=no
Slice(true)           -> Option=no,  Box=no
SliceCursor(false)    -> Option=no,  Box=no
SliceCursor(true)     -> Option=no,  Box=no
OptRef(false)         -> Option=yes, Box=no
OptRef(true)          -> Option=yes, Box=no
Box                   -> Option=no,  Box=yes
BoxedSlice            -> Option=no,  Box=yes
OptBox                -> Option=yes, Box=yes
OptBoxedSlice         -> Option=yes, Box=yes
```

The test must be a direct helper test with no compiler callback. Its match over
`PtrKind` remains exhaustive rather than using a wildcard arm.

Use exact inputs A4-SRC-STANDARD-CONSTRUCTORS,
A4-SRC-STANDARD-BARE-IMPORTS, A4-SRC-NO-STD-OPTION-SUCCESS,
A4-SRC-NAMED-OPTIONAL-BOX, and A4-SRC-IRRELEVANT-COLLISIONS.

Before checking generation success or rendered spelling, obtain tools-mode
`initial_pointer_decisions` and require:

```text
A4-SRC-STANDARD-CONSTRUCTORS:
wrapped::read input p       = OptRef(false)
wrapped::owned_id input p   = OptBox
wrapped::owned_id output    = OptBox
wrapped::foo output         = OptBox
wrapped::foo local p        = Box
wrapped::foo local q        = OptBox
wrapped::allocate output    = Box
wrapped::allocate local p   = Box

A4-SRC-NAMED-OPTIONAL-BOX:
consumer::owned_id input p  = OptBox
consumer::owned_id output   = OptBox
consumer::foo output        = OptBox
consumer::foo local p       = Box
consumer::foo local q       = OptBox

A4-SRC-STANDARD-BARE-IMPORTS:
imported::read input p       = OptRef(false)
imported::allocate output    = Box
imported::allocate local p   = Box

A4-SRC-NO-STD-OPTION-SUCCESS:
read input p                 = OptRef(false)

A4-SRC-IRRELEVANT-COLLISIONS:
box_only::allocate output   = Box
box_only::allocate local p  = Box
option_only::read input p   = OptRef(false)
```

Locate signatures and bindings by `DefId`/HIR identity. A decision mismatch is
a pointer-analysis regression and must stop the spelling assertions.
The `read` body is the existing tools optional-reference fixture and the
`allocate` body is the existing tools scalar-box fixture.
`owned_id`/`foo` reproduce the repository's real tools
`optional_box_fixture`; the enclosing module and shared `malloc` declaration
do not change those bodies' pointer semantics.
A4-SRC-NAMED-OPTIONAL-BOX repeats that proven semantic shape with every
pointee written through a distinct binding to the same source type alias. In
A4-SRC-IRRELEVANT-COLLISIONS, each collision item is unreferenced by the
corresponding analyzed body. Keep the direct assertions above so each complete
input independently pins that equivalence.

For A4-SRC-STANDARD-CONSTRUCTORS, require these exact bare target types:

```text
wrapped::read.p          = Option<&i32>
wrapped::owned_id.p      = Option<Box<i32>>
wrapped::owned_id return = Option<Box<i32>>
wrapped::foo return      = Option<Box<i32>>
wrapped::foo.p           = Box<i32>
wrapped::foo.q           = Option<Box<i32>>
wrapped::allocate return = Box<i32>
wrapped::allocate.p      = Box<i32>
```

Resolve the bare `Option` and `Box` paths from `wrapped` and require their exact
definitions to be the standard `Option` and owned `Box` lang items.
Also require the exact target signatures:

```rust
pub unsafe fn owned_id(
    mut p: Option<Box<i32>>,
) -> Option<Box<i32>>
```

```rust
pub unsafe fn foo() -> Option<Box<i32>>
```

For A4-SRC-STANDARD-BARE-IMPORTS, generation also succeeds with:

```text
imported::read.p          = Option<&i32>
imported::allocate return = Box<i32>
imported::allocate.p      = Box<i32>
```

Resolve `Option` and `Box` through the explicit bare imports and require the
exact standard lang-item definitions. This proves a valid explicit import is
accepted under the ordinary enabled prelude; the implementation must not
mistake every explicit binding for a shadowing collision.

For A4-SRC-NO-STD-OPTION-SUCCESS, generation succeeds with the exact target:

```rust
pub unsafe fn read(mut p: Option<&i32>) -> i32
```

Resolve bare `Option` through the ordinary core prelude to the exact standard
`Option` lang item. This must not be rejected merely because `#![no_std]` is
present; the separate no-std `Box` case below proves that the same enabled
prelude does not supply owned `Box`.

For A4-SRC-NAMED-OPTIONAL-BOX, require the exact target signatures:

```rust
pub unsafe fn owned_id(
    mut p: Option<Box<P>>,
) -> Option<Box<ReturnP>>
```

```rust
pub unsafe fn foo() -> Option<Box<ReturnP>>
```

Require these exact target locals in `consumer::foo`:

```rust
let mut p: Box<LocalP> = todo!();
```

```rust
let mut q: Option<Box<LocalQ>> = todo!();
```

Resolve `P`, `ReturnP`, `LocalP`, and `LocalQ` in their respective parameter,
explicit return, and explicitly typed local positions to the same
`model::PointAlias` `DefId`. Their distinct spellings prove that no parameter,
return, or local hint is reused for another location. The introduced standard
constructor nodes remain the accepted bare `Option` and `Box`, but their
source-AST pointees must not normalize to `Point`,
`crate::model::Point`, or another qualified path to the defining type. This
assertion is independent for the `OptBox` parameter, the `OptBox` returns, the
`Box` local `p`, and the `OptBox` local `q`.

For A4-SRC-IRRELEVANT-COLLISIONS, generation succeeds with:

```text
box_only::allocate return = Box<i32>
box_only::allocate.p      = Box<i32>
option_only::read.p       = Option<&i32>
```

The user `box_only::Option` is irrelevant to the `Box` decisions, and the user
`option_only::Box` is irrelevant to the `OptRef(false)` decision. The
implementation must not preflight both constructor names globally.

Retain the existing unshadowed optional-reference, box, optional-box, and
boxed-slice skeleton regressions with their existing bare constructor strings.
The exhaustive pure-helper table above covers `OptBoxedSlice`; do not invent a
compiler fixture or claim an existing optional-boxed-slice skeleton regression.
Existing item-replacer constructor recognition, wrapper helper spelling,
conversion semantics, request/response schemas, and tests remain unchanged;
Amendment 4 adds no alias or qualified-constructor support to the item replacer.

## 7. Preservation and failure boundaries

### A4-PRES-01 `wholly_preserved_parent_does_not_materialize_nested_local`

Use exact input A4-SRC-PRESERVED-PARENT.

The complete `if` is Amendment-2 preservable and its containing label is not in
`statements_requiring_transformation`. The exact nested declaration in
`annotated_skeleton` remains:

```rust
let mut value = Local { value: 1 };
```

It does not become:

```rust
let mut value: Local = Local { value: 1 };
```

This test prevents Amendment 4 from adding a traversal beneath a wholly
preserved statement. It does not weaken the existing rule that a visited
simple local receives an explicit target type.

### A4-ERR-01 `type_spelling_failures_are_structured_and_atomic`

Use exact inputs A4-SRC-OPTION-COLLISION, A4-SRC-BOX-COLLISION,
A4-SRC-RENAMED-CONSTRUCTOR-COLLISION,
A4-SRC-GLOB-CONSTRUCTOR-COLLISION,
A4-SRC-OPTBOX-PARTIAL-CONSTRUCTOR-COLLISION,
A4-SRC-LOCAL-BOX-COLLISION,
A4-SRC-EXTERN-PRELUDE-CONSTRUCTOR-COLLISION,
A4-SRC-NO-IMPLICIT-PRELUDE-REJECTION,
A4-SRC-NO-STD-BOX-REJECTION,
A4-SRC-BOX-NO-IMPLICIT-PRELUDE-REJECTION,
A4-SRC-MODULE-NO-IMPLICIT-PRELUDE-REJECTION,
A4-SRC-ANCESTOR-NO-IMPLICIT-PRELUDE-REJECTION, and A4-SRC-UNNAMEABLE
independently.

Before invoking the constructor invariant, obtain tools-mode
`initial_pointer_decisions` for every constructor input and require:

```text
A4-SRC-OPTION-COLLISION:
wrapped::read input first   = OptRef(false)
wrapped::read input second  = OptRef(false)

A4-SRC-BOX-COLLISION:
wrapped::allocate output    = Box
wrapped::allocate local p   = Box

A4-SRC-RENAMED-CONSTRUCTOR-COLLISION:
renamed::read input p       = OptRef(false)

A4-SRC-GLOB-CONSTRUCTOR-COLLISION:
globbed::allocate output    = Box
globbed::allocate local p   = Box

A4-SRC-OPTBOX-PARTIAL-CONSTRUCTOR-COLLISION:
wrapped::owned_id input p   = OptBox
wrapped::owned_id output    = OptBox
wrapped::foo output         = OptBox
wrapped::foo local p        = Box
wrapped::foo local q        = OptBox

A4-SRC-LOCAL-BOX-COLLISION:
consumer::local_only local first  = Box
consumer::local_only local second = Box

A4-SRC-EXTERN-PRELUDE-CONSTRUCTOR-COLLISION:
wrapped::read input p       = OptRef(false)

A4-SRC-NO-IMPLICIT-PRELUDE-REJECTION:
wrapped::read input p       = OptRef(false)

A4-SRC-NO-STD-BOX-REJECTION:
allocate output             = Box
allocate local p            = Box

A4-SRC-BOX-NO-IMPLICIT-PRELUDE-REJECTION:
allocate output             = Box
allocate local p            = Box

A4-SRC-MODULE-NO-IMPLICIT-PRELUDE-REJECTION:
wrapped::read input p       = OptRef(false)

A4-SRC-ANCESTOR-NO-IMPLICIT-PRELUDE-REJECTION:
outer::middle::inner::read input p = OptRef(false)
```

Locate each subject by `DefId`/HIR identity. If a decision differs, report a
pointer-analysis regression rather than accepting the following error oracle.

Each constructor-invariant input returns the single general error kind
`GenerationErrorKind::TypeSpelling`, with no records and therefore no JSON.
Keep the existing `GenerationError` fields; the location and reason assertions
below apply to `message`:

```text
A4-SRC-OPTION-COLLISION:
function_path = "wrapped::read"
message identifies location parameter "first"
message identifies selected kind OptRef(false)
message identifies required bare constructor Option
message says bare Option resolves to wrapped::Option, not standard Option
the second parameter is independently OptRef(false), proving the first
failure is selected in parameter source order

A4-SRC-BOX-COLLISION:
function_path = "wrapped::allocate"
message identifies location return
message identifies selected kind Box
message identifies required bare constructor Box
message says bare Box resolves to wrapped::Box, not standard owned Box

A4-SRC-RENAMED-CONSTRUCTOR-COLLISION:
function_path = "renamed::read"
message identifies location parameter "p"
message identifies selected kind OptRef(false)
message identifies required bare constructor Option
message says bare Option resolves through the renamed import to
fake::WrongOption, not standard Option

A4-SRC-GLOB-CONSTRUCTOR-COLLISION:
function_path = "globbed::allocate"
message identifies location return
message identifies selected kind Box
message identifies required bare constructor Box
message says bare Box resolves through the glob import to fake::glob::Box,
not standard owned Box

A4-SRC-OPTBOX-PARTIAL-CONSTRUCTOR-COLLISION:
function_path = "wrapped::owned_id"
message identifies location parameter "p"
message identifies selected kind OptBox
an independent resolver assertion proves bare Option is standard Option
message says bare Box resolves to wrapped::Box, not standard owned Box

A4-SRC-LOCAL-BOX-COLLISION:
function_path = "consumer::local_only"
message identifies location local "first"
message identifies selected kind Box
message identifies required bare constructor Box
message says bare Box resolves to consumer::Box, not standard owned Box
the second local is independently Box, proving the first failure is selected
in local statement order

A4-SRC-EXTERN-PRELUDE-CONSTRUCTOR-COLLISION:
function_path = "wrapped::read"
message identifies location parameter "p"
message identifies selected kind OptRef(false)
message identifies required bare constructor Option
message says bare Option resolves through the extern prelude to the external
crate alias Option, not standard Option

A4-SRC-NO-IMPLICIT-PRELUDE-REJECTION:
function_path = "wrapped::read"
message identifies location parameter "p"
message identifies selected kind OptRef(false)
message identifies required bare constructor Option
message says the containing module wrapped has its ordinary implicit prelude
disabled
an independent resolver assertion proves its explicit bare Option import is
the exact standard Option lang item, so disabled-prelude rejection takes
precedence over an otherwise valid binding

A4-SRC-NO-STD-BOX-REJECTION:
function_path = "allocate"
message identifies location return
message identifies selected kind Box
message identifies required bare constructor Box
message says the ordinary core prelude is enabled but bare Box is unresolved

A4-SRC-BOX-NO-IMPLICIT-PRELUDE-REJECTION:
function_path = "allocate"
message identifies location return
message identifies selected kind Box
message identifies required bare constructor Box
message says the crate root has its ordinary implicit prelude disabled

A4-SRC-MODULE-NO-IMPLICIT-PRELUDE-REJECTION:
function_path = "wrapped::read"
message identifies location parameter "p"
message identifies selected kind OptRef(false)
message identifies required bare constructor Option
message says the containing module wrapped has its ordinary implicit prelude
disabled

A4-SRC-ANCESTOR-NO-IMPLICIT-PRELUDE-REJECTION:
function_path = "outer::middle::inner::read"
message identifies location parameter "p"
message identifies selected kind OptRef(false)
message identifies required bare constructor Option
message says the containing module outer::middle::inner has its effective ordinary
implicit prelude disabled by ancestor module outer
```

The `Maybe` and `Owned` aliases in the collision inputs do not change either
failure. Do not emit an alias, an absolute standard path, a knowingly
misresolved bare type, or a partial function record.

For A4-SRC-UNNAMEABLE, local `iterator` in function `consume` has an opaque
semantic type. Because
Amendment 2 cannot prove that type transformation-insensitive,
skeletonization visits the local; because ordinary Rust cannot write its
concrete opaque type as a local annotation, generation returns:

```text
kind = GenerationErrorKind::TypeSpelling
function_path = "consume"
message identifies location local "iterator"
message identifies an opaque or unnameable semantic type
records returned = none
JSON returned = none
```

Do not panic, emit `impl Iterator<Item = i32>` as a local type, omit the local
type, or parse a placeholder. This robustness case does not add opaque/source
generics to the supported end-to-end input model.

## 8. Validation, replacement, and compiler integration

### A4-INTEG-01 `generated_local_name_validates_replaces_and_compiles_in_original_module`

Use exact current compiler input A4-SRC-MOTIVATING.

First normalize target safety through the existing in-memory API if required
by the replacement helper, then generate skeletons. Use the exact generated
record for `src::lib::cb_remove_gamma_rgb`. Construct the transformation by
copying its exact target signature and labels and filling the generated holes
as follows:

```rust
pub unsafe fn cb_remove_gamma_rgb(mut rgb: cb_rgb) -> cb_rgb {
    #[proctor(0)]
    let mut result: cb_rgb = {
        #[proctor(1)]
        let mut init: cb_rgb = cb_rgb {
            r: crate::transform(rgb.r as f64) as f32,
            g: crate::transform(rgb.g as f64) as f32,
            b: crate::transform(rgb.b as f64) as f32,
        };
        #[proctor(2)]
        init
    };
    #[proctor(3)]
    result
}
```

Use the generated Amendment-2 disposition metadata rather than hard-coding a
different label set. Expected sequence:

1. structural validation returns `valid`;
2. item replacement targets exact path
   `src::lib::cb_remove_gamma_rgb`;
3. all `proctor` labels are removed;
4. no wrapper is generated because the signature type is unchanged;
5. the replacement remains `pub unsafe fn cb_remove_gamma_rgb`, preserving
   the source item's public visibility;
6. emitted code contains `let mut init: cb_rgb`; and
7. a separate second `run_compiler_on_str` invocation accepts the complete
   emitted crate.

Run an additional parser-only contrast in which only the transformation's
local annotation is changed to `crate::src::lib::cb_rgb`. The validator keeps
its existing structural rule and returns `local_type_mismatch`; Amendment 4
does not make alternative paths semantically interchangeable.

## 9. Unchanged APIs and determinism

### A4-UNCHANGED-01 `ordinary_pointer_rewriter_and_decisions_are_unchanged`

Use exact inputs A4-SRC-TREE, A4-SRC-POINTERS, and A4-SRC-COMPOUND.

For each input, obtain `InitialPointerDecisions` directly and through
skeleton generation in independent compiler callbacks. Assert identical
`PtrKind` and lifetime choices. For A4-SRC-TREE, retain the exact ordinary
rewriter oracle:

```rust
pub unsafe fn tree_print_helper(
    mut tree: *mut crate::Tree,
    root_id: i32,
)
```

The tools skeleton alone uses `&mut Tree`. No normal Crat configuration or
rewriter caller receives a scope-spelling option. No cursor decision or
transformation-time demotion behavior changes.

### A4-UNCHANGED-02 `scope_tables_and_serialized_output_are_deterministic`

Use exact inputs A4-SRC-IMPORTS, A4-SRC-CANDIDATES,
A4-SRC-CANDIDATE-PRECEDENCE, A4-SRC-REEXPORTS,
A4-SRC-LOCAL-FALLBACK-ROUTES, A4-SRC-EXTERNAL-ROOT-ALIAS,
A4-SRC-RAW-IDENTIFIERS, and A4-SRC-QUALIFIED-RAW-FALLBACK.

Run each exact input through two independent `run_compiler_on_str` callbacks.
Require equal structured records and byte-identical pretty JSON. In
particular:

```text
aliases::inferred = Alpha
aliases::source_hint = Zed
collision::inferred = crate::left::Thing
consumer::local = crate::api::Exposed
consumer::external = ::std::hash::DefaultHasher
consumer::restricted = crate::restricted_api::Exposed
consumer::shortest = crate::short::S
consumer::tie = crate::alpha::T
consumer::external_alias = ::alt_std::hash::DefaultHasher
```

Do not depend on `module_children_local` iteration order, `DefId` allocation,
hash-map order, or spans. Re-run the existing function-record key-order golden
to prove this amendment adds no field and changes no protocol.

## 10. Completion criteria

Amendment 4 is complete when:

- all 18 named cases are implemented and passing;
- every superseded implementation oracle is updated without editing a
  historical test-plan file;
- the skeleton tools path never parses `Ty::to_string()` as source syntax;
- containing-module candidates come from rustc resolution and exact
  type-namespace `DefId` matches;
- source pointee syntax is reused before semantic fallback;
- an explicit raw-pointer return preserves its distinct mapped source alias
  separately from the input while retaining their exact shared generated
  lifetime;
- an explicitly typed pointer local preserves its distinct mapped source alias
  through `Skeletonizer::flat_map_stmt`, independently of parameter and
  inferred-local rendering;
- explicit `Box`/`OptBox` parameters, returns, and locals preserve their
  mapped source pointee alias beneath the introduced bare wrappers;
- same-module, direct-import, renamed-import, glob-import, alias, relative,
  collision, compound, lifetime, and raw-identifier cases match the exact
  spellings above;
- one pure exhaustive helper maps every current `PtrKind` to its `Option` and
  `Box` requirements and passes the complete boolean-payload table;
- introduced `Option` and `Box` retain their existing bare spelling only when
  the ordinary prelude is enabled and each relevant bare identifier resolves
  to the exact standard lang-item definition;
- bare `Option` succeeds through the core prelude under `#![no_std]`, while
  bare owned `Box` remains unresolved there;
- an extern-prelude binding named `Option` or `Box` is checked before the
  standard prelude and rejected when it resolves to a crate rather than the
  required lang item;
- relevant user collisions and effective `no_implicit_prelude` from the crate
  or any module ancestor through the containing module itself produce an
  atomic `TypeSpelling` error after the intended pointer decisions are
  asserted;
- collisions irrelevant to the selected target kind do not reject generation;
- no alias or absolute standard-constructor support is added to skeleton
  generation or item replacement;
- any pre-local-transformation collision renaming/normalization remains
  explicitly deferred;
- parameter and local failures select the first location in source order;
- rustc visible paths, rather than raw definition paths, supply fallbacks;
- local fallbacks use `crate::...` and external fallbacks use an absolute
  `::source_visible_extern_binding::...` whenever no safe short or existing
  source path is available;
- every fallback parses and accessibly resolves from the containing module to
  the intended definition;
- local and external public re-exports hide their private defining paths;
- wholly preserved parents retain their current traversal boundary;
- an unnameable type returns the same atomic `TypeSpelling` generation error;
- structural validation remains syntactic and protocol version 1;
- the compiler-backed insertion case succeeds in the original nested module;
- ordinary pointer-rewriter output and pointer decisions remain unchanged;
- repeated output is byte deterministic; and
- from `proctor/stages/crat`, `cargo fmt`,
  `cargo clippy --workspace --all-targets`, and `cargo test --workspace` pass.
