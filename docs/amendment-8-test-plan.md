# Amendment 8 Test Plan: Crat `prepare` Pass

## 1. Purpose and authority

This plan is the exhaustive acceptance specification for the ordinary Crat
`prepare` pass defined in
[amendment-8-plan.md](amendment-8-plan.md). The pass prepares expanded,
compiler-valid Rust for PROCTOR's local-transformation stage by:

1. wrapping every match-arm body that is not already a block in an explicit
   block; and
2. lifting every function-local static into the nearest lexical module that
   contains its owning function, renaming it and its compiler-resolved uses
   when the original name has any other crate-wide value-namespace binder.

The feature is an ordinary `crat` pass in `crates/passes`; it is not a
`crat-tool` operation. The only pipeline configuration change is insertion of
`"prepare"` between `"enum"` and `"simpl"` in
`proctor/configs/c2rust_crat_local.toml`. There is no libc work in this change.

Current implementation and tests remain authoritative if they reveal a
factual discrepancy. Any such discrepancy that changes behavior in this plan
requires an explicit plan decision rather than a silent implementation change.
Planning names such as `A8-*` and “Amendment 8” are documentation-only; do not
put them in code, test names, fixtures, diagnostics, configuration values, or
file names.

## 2. Test ownership and execution

Pass tests belong beside the implementation, preferably in
`proctor/stages/crat/crates/passes/src/preparer/tests.rs`, and call the pass
library directly. Compiler-resolved cases use the existing
`utils::compilation::run_compiler_on_str` harness. They must not invoke
`crat-tool` or `crat`, create files, mutate a project tree, or use a Crat-root
`tests/` directory.

Pure `Pass` CLI/TOML parsing tests may live in `src/bin/crat.rs` and must not
run the CLI or touch the filesystem. Adapter and checked-in-configuration
tests belong in `proctor/tests/test_crat_adapter.py`; they may use `tmp_path`
and the existing fake command runner but remain offline.

Run from `proctor/stages/crat`:

```bash
cargo test -p passes preparer::tests
cargo test --bin crat
cargo test -p passes
cargo test --workspace
cargo fmt
cargo clippy --workspace --all-targets
```

Run from `proctor`:

```bash
uv run pytest tests/test_crat_adapter.py
uv run proctor validate -c configs/c2rust_crat_local.toml
uv run pytest
uv run ruff check .
uv run ruff format --check .
uv run mypy proctor
```

The default suites remain deterministic, offline, API-key-free, and free of a
real C2Rust/CRAT build. The existing CRAT smoke E2E uses the adapter default
chain, which intentionally excludes `prepare`, so it is not evidence for this
feature.

## 3. Exact-comparison policy and shared notation

`T(input)` means compile `input`, require the outer compiler result to be
`Ok`, run `preparer::prepare`, require its inner result to be `Ok`, parse the
returned source, and compare its complete relevant AST structurally.
Pretty-printer whitespace and optional commas after block match arms are not
contractual. Item order, identifiers, attributes, mutability, types,
initializers, paths, arm patterns/guards/attributes, block safety, statements,
and tail-expression placement are exact. Every positive transformed result is
compiled again with `run_compiler_on_str(..., utils::type_check)`.

`Same(input)` means `T(input)` has the same complete relevant AST as `input`.
`Err(static, dependency)` means the outer compiler result is `Ok` and the pass
returns the inner structured `PrepareError::ScopedDependency` identifying both
compiler definitions; its display text names both items. `Ok(Err(error))` at
the CLI boundary prints `prepare failed: <display text>`, exits nonzero, and
writes no transformed source. A mapping failure similarly uses
`PrepareError::MissingMapping` and names the involved construct. Tests match
structured variants in the library and only stable diagnostic fragments at
the CLI boundary. An outer compiler `Err` remains the existing compiler
failure path; it is never relabeled as a `PrepareError` or prefixed with
`prepare failed:`.

All analysis, dependency validation, name allocation, and required AST/HIR
mapping validation finish before mutation or output. Thus every error is
crate-atomic: no arm is wrapped, static removed, identifier rewritten, or
partial source returned/written.

Unless stated otherwise, inputs include:

```rust
#![allow(dead_code, non_snake_case, non_upper_case_globals)]
```

The expected snippets below omit only that harness attribute. “Immediately
before” means immediately before the direct child of the destination module
that contains the static's owning function. For a free function this is the
function; for a method it is the containing `impl` or `trait`; for a static in
a nested local function, closure, or async block it is the outer function. A
local `mod` is a real lexical module boundary: a static owned by a function in
that module moves immediately before that function inside the module. Multiple
statics at one insertion point retain depth-first lexical discovery order.

## 4. Updated existing regression cases

### A8-UPD-01 `default_pass_plan_excludes_optional_prepare`

Update the existing adapter default-plan test. Input is
`resolve_pass_plan({})`. Expected exact pass names and existing default flags:

```python
[
    "expand", "extern", "preprocess", "outparam", "punning", "enum",
    "pointer", "io", "libc", "static", "simpl", "interface", "unsafe",
    "unexpand", "split", "bin",
]
```

`prepare` is absent. The existing flags for `extern`, `outparam`, `io`,
`unsafe`, and `unexpand` are byte-for-byte unchanged. The oracle must no longer
assume that every key in `PLUGINS` belongs to the canonical default chain,
because `prepare` is a known opt-in pass outside that chain.

### A8-UPD-02 `local_tool_rejections_remain_defensive`

Keep the existing direct `crat-tool` library tests unchanged. Concrete inputs
to skeleton generation remain:

```rust
fn arm(x: i32) -> i32 { match x { 0 => 1, _ => 2 } }
fn item() -> i32 { static X: i32 = 1; X }
```

Expected when these unprepared sources are passed directly to tools:
generation still fails atomically for, respectively, a non-block match arm and
a function-local item. The new ordinary pass does not weaken tools-side
validation and is not automatically invoked by `crat-tool`.

### A8-UPD-03 `local_transformation_internal_preparation_is_unchanged`

In the existing Python fake-tools regression, the local-transformation stage's
own preparation call remains exactly:

```python
("prepare", current_project, ("expand", "unexpand"), True)
```

It must not gain the ordinary `prepare` pass. The checked-in pipeline has
already run it in `crat-adapter`; changing the local stage would duplicate work
and affect standalone-stage behavior.

## 5. Match-arm block normalization

### A8-MATCH-01 `wraps_every_non_block_arm_body`

Input:

```rust
fn choose(x: i32) -> i32 {
    match x {
        0 => 7,
        1 => x + 1,
        _ => return 9,
    }
}
```

Expected body:

```rust
match x {
    0 => { 7 }
    1 => { x + 1 }
    _ => { return 9 }
}
```

Each inserted block has default safety and contains the original expression as
its tail expression. The `return` remains a tail expression; no semicolon or
temporary is invented.

### A8-MATCH-02 `preserves_existing_blocks_exactly`

Input:

```rust
fn choose(x: i32) -> i32 {
    match x {
        0 => { let y = x + 1; y },
        _ => unsafe { core::ptr::read(&x) },
    }
}
```

Expected: `Same(input)`. Both ordinary and `unsafe` block expressions are
already blocks and are not double-wrapped or made safe.

### A8-MATCH-03 `preserves_patterns_guards_and_arm_attributes`

Input:

```rust
#[allow(unused_attributes)]
fn choose(x: Option<i32>) -> i32 {
    match x {
        #[allow(unreachable_patterns)]
        Some(value) if value > 0 => value,
        None | Some(_) => 0,
    }
}
```

Expected:

```rust
match x {
    #[allow(unreachable_patterns)]
    Some(value) if value > 0 => { value }
    None | Some(_) => { 0 }
}
```

Patterns, binding identity, guard expression, attribute, and arm order are
unchanged.

### A8-MATCH-04 `normalizes_nested_matches_recursively`

Input:

```rust
fn nested(x: i32, y: i32) -> i32 {
    match x {
        0 => match y { 0 => 1, _ => 2 },
        _ => 3,
    }
}
```

Expected:

```rust
match x {
    0 => { match y { 0 => { 1 }, _ => { 2 } } }
    _ => { 3 }
}
```

The inner arms and then the non-block outer body all satisfy the postcondition.

### A8-MATCH-05 `normalizes_matches_in_every_expression_owner`

Input:

```rust
static TOP: i32 = match 0 { 0 => 1, _ => 2 };
const C: i32 = match 1 { 1 => 3, _ => 4 };
fn owners() {
    let closure = || match 2 { 2 => 5, _ => 6 };
    let _future = async { match 3 { 3 => 7, _ => 8 } };
}
```

Expected: each of the eight numeric arm bodies is an explicit `{ number }`
block. `TOP`, `C`, the closure, and the async block otherwise remain unchanged.

### A8-MATCH-06 `wraps_non_block_control_and_special_expressions`

Input:

```rust
fn forms(x: i32) -> i32 {
    match x {
        0 => if x == 0 { 1 } else { 2 },
        1 => loop { break 3 },
        2 => (4),
        _ => 5,
    };
    0
}

fn unit(x: i32) {
    match x { 0 => (), _ => () }
}
```

Expected arm bodies are exactly `{ if ... }`, `{ loop ... }`, `{ (4) }`,
`{ 5 }`, and the two `{ () }` bodies. Parentheses and control structure are
retained inside the new block.

### A8-MATCH-07 `match_only_input_has_no_unrelated_changes`

Input has imports, attributes, type aliases, a struct, a top-level static, and
one function containing no match and no local static:

```rust
use core::ffi::c_int as Int;
#[repr(C)] struct Pair { left: Int, right: Int }
static VALUE: Int = 1;
fn sum(pair: Pair) -> Int { pair.left + pair.right + VALUE }
```

Expected: `Same(input)`. This guards the surgical scope of the visitor.

## 6. Static discovery, lifting, and placement

### A8-LIFT-01 `lifts_direct_function_static_before_function`

Input:

```rust
fn next() -> i32 {
    static COUNT: i32 = 4;
    COUNT + 1
}
```

Expected module items:

```rust
static COUNT: i32 = 4;
fn next() -> i32 { COUNT + 1 }
```

The local item statement is removed without leaving an empty statement.

### A8-LIFT-02 `lifts_nested_control_flow_statics_in_lexical_order`

Input:

```rust
fn nested(flag: bool) -> i32 {
    if flag {
        static FIRST: i32 = 1;
        loop {
            static SECOND: i32 = 2;
            break FIRST + SECOND;
        }
    } else {
        static THIRD: i32 = 3;
        THIRD
    }
}
```

Expected item order and function:

```rust
static FIRST: i32 = 1;
static SECOND: i32 = 2;
static THIRD: i32 = 3;
fn nested(flag: bool) -> i32 {
    if flag { loop { break FIRST + SECOND; } } else { THIRD }
}
```

### A8-LIFT-03 `lifts_from_closure_async_and_local_function`

Input:

```rust
fn outer() {
    let closure = || { static CLOSURE: i32 = 1; CLOSURE };
    let future = async { static FUTURE: i32 = 2; FUTURE };
    fn inner() -> i32 { static INNER: i32 = 3; INNER }
    let _ = (closure, future, inner());
}
```

Expected module item order:

```rust
static CLOSURE: i32 = 1;
static FUTURE: i32 = 2;
static INNER: i32 = 3;
fn outer() {
    let closure = || { CLOSURE };
    let future = async { FUTURE };
    fn inner() -> i32 { INNER }
    let _ = (closure, future, inner());
}
```

All three move before `outer`, the module-level ancestor. The local function
itself remains local and outside this feature's scope.

### A8-LIFT-04 `lifts_method_statics_before_containing_item`

Input:

```rust
struct S;
impl S {
    fn one() -> i32 { static ONE: i32 = 1; ONE }
    fn two(&self) -> i32 { static TWO: i32 = 2; TWO }
}
trait T {
    fn defaulted() -> i32 { static THREE: i32 = 3; THREE }
}
```

Expected module item order:

```rust
struct S;
static ONE: i32 = 1;
static TWO: i32 = 2;
impl S { fn one() -> i32 { ONE } fn two(&self) -> i32 { TWO } }
static THREE: i32 = 3;
trait T { fn defaulted() -> i32 { THREE } }
```

No invalid associated static is inserted into the `impl` or `trait`.

### A8-LIFT-05 `uses_each_nearest_lexical_module`

Input:

```rust
static ROOT: i32 = 0;
mod left {
    pub fn f() -> i32 { static CELL: i32 = 1; CELL }
}
mod right {
    pub mod inner {
        pub fn g() -> i32 { static CELL: i32 = 2; CELL }
    }
}
```

Expected structural output:

```rust
static ROOT: i32 = 0;
mod left {
    static CELL_0: i32 = 1;
    pub fn f() -> i32 { CELL_0 }
}
mod right {
    pub mod inner {
        static CELL_1: i32 = 2;
        pub fn g() -> i32 { CELL_1 }
    }
}
```

The same spelling on the two moved statics is a crate-wide collision despite
different modules; lexical discovery assigns `_0` then `_1`.

### A8-LIFT-06 `distinguishes_module_owned_and_function_owned_statics`

Input:

```rust
static ROOT: i32 = 0;
mod ordinary { static MODULE: i32 = 1; }
fn f() {
    mod local_module {
        pub static OWNED: i32 = 2;
        pub fn inner() -> i32 {
            static NESTED: i32 = 3;
            OWNED + NESTED
        }
    }
    let _ = local_module::inner();
}
```

Expected relevant output:

```rust
static ROOT: i32 = 0;
mod ordinary { static MODULE: i32 = 1; }
fn f() {
    mod local_module {
        pub static OWNED: i32 = 2;
        static NESTED: i32 = 3;
        pub fn inner() -> i32 { OWNED + NESTED }
    }
    let _ = local_module::inner();
}
```

`ROOT`, `MODULE`, and `OWNED` are directly owned by modules and remain in
place, even though `local_module` itself is a function-local item. `NESTED` is
owned by `inner`, so it moves into `local_module` immediately before `inner`.
The local module itself remains local and outside `prepare`.

### A8-LIFT-07 `preserves_static_payload_and_non_export_attributes`

Input:

```rust
fn f() -> u32 {
    #[used]
    #[link_section = ".data.demo"]
    static mut CELL: u32 = 7;
    unsafe { CELL += 1; CELL }
}
```

Expected:

```rust
#[used]
#[link_section = ".data.demo"]
static mut CELL: u32 = 7;
fn f() -> u32 { unsafe { CELL += 1; CELL } }
```

The pass preserves every non-export attribute, mutability, explicit type, and
initializer; it does not add visibility or safety wrappers.

## 7. Crate-wide value-name collection and suffix allocation

For these cases, a crate binder reserves its compiler symbol anywhere in the
local crate, regardless of Rust lexical scope. Distinct compiler definitions
are counted, not textual occurrences. A moved static keeps `name` only when no
other crate binder has that symbol and `name` is not an active implicit-prelude
value name at its destination module. Otherwise candidates are exactly
`name_0`, `name_1`, ...; choose the smallest candidate absent from the complete
original crate-binder set, the active implicit-prelude value names for the
destination module, and earlier allocations.

Active implicit-prelude names are allocation-only reservations. Derive them
from compiler resolution for each destination module, honoring edition,
`no_std`, `no_implicit_prelude`, and a compiler-supported custom prelude; never
hardcode a `std`/`core` name list. They do not contribute crate-binder
multiplicity, but matching an original static name is independently a
collision and requires renaming. Prelude names also reserve generated
candidates.

### A8-NAME-01 `value_item_binders_force_rename`

Use independent compiling subcases, putting the colliding declaration in
`mod other` and the local static in `mod target`:

```rust
// function:       mod other { pub fn NAME() {} }
// constant:       mod other { pub const NAME: i32 = 0; }
// static:         mod other { pub static NAME: i32 = 0; }
// foreign item:   mod other { unsafe extern "C" { pub static NAME: i32; } }
mod target { pub fn f() -> i32 { static NAME: i32 = 1; NAME } }
```

Expected in every subcase: target contains
`static NAME_0: i32 = 1;` and `f` returns `NAME_0`. A module-level function,
const, static, or foreign value declaration is a reserved value binder.

### A8-NAME-02 `imports_aliases_and_globs_force_rename`

Use independent inputs:

```rust
mod source { pub static ORIGINAL: i32 = 0; pub static GLOB_NAME: i32 = 0; }
mod imported { use crate::source::ORIGINAL as NAME; }
mod globbed { use crate::source::*; }
mod target_a { pub fn f() -> i32 { static NAME: i32 = 1; NAME } }
mod target_b { pub fn g() -> i32 { static GLOB_NAME: i32 = 2; GLOB_NAME } }
```

Expected: the two moved definitions and resolved uses become `NAME_0` and
`GLOB_NAME_0`. Explicit aliases and names introduced by an explicit glob are
reserved even when the import is in another module and unused. An implicit
prelude is handled separately as an allocation-only reservation, not as a
crate-owned binder.

### A8-NAME-03 `only_value_namespace_constructors_force_rename`

Input:

```rust
mod shapes {
    pub struct TUPLE(pub i32);
    pub struct UNIT;
    pub enum E { VARIANT, PAYLOAD(i32), RECORD { x: i32 } }
}
mod target {
    pub fn f() {
        static TUPLE: i32 = 1;
        static UNIT: i32 = 2;
        static VARIANT: i32 = 3;
        static PAYLOAD: i32 = 4;
        static RECORD: i32 = 5;
        let _ = (TUPLE, UNIT, VARIANT, PAYLOAD, RECORD);
    }
}
```

Expected moved names and tuple uses are `TUPLE_0`, `UNIT_0`, `VARIANT_0`,
`PAYLOAD_0`, and unchanged `RECORD`. Tuple/unit struct constructors and
tuple/unit enum-variant constructors reserve value names. A struct-like enum
variant has no ValueNS constructor binder and does not force a rename solely
from its member name.

### A8-NAME-04 `all_pattern_binding_forms_force_rename`

Use one crate with a function `binders` containing these compiler bindings:

```rust
struct S { field: i32, rest: i32 }
fn binders(
    PARAM: i32,
    (TUPLE, _): (i32, i32),
    S { field: STRUCT, .. }: S,
) {
    let LET = 0;
    let [ARRAY, _] = [0, 1];
    if let Some(IF_LET) = Some(1) { let _ = IF_LET; }
    while let Some(WHILE_LET) = None::<i32> { let _ = WHILE_LET; }
    for FOR in 0..1 { let _ = FOR; }
    match Some(1) { Some(MATCH) => { let _ = MATCH; }, None => {} }
    let _closure = |CLOSURE: i32| CLOSURE;
    let _ = (PARAM, TUPLE, STRUCT, LET, ARRAY);
}
mod target {
    pub fn f() {
        static PARAM: i32 = 0;
        static TUPLE: i32 = 1;
        static STRUCT: i32 = 2;
        static LET: i32 = 3;
        static ARRAY: i32 = 4;
        static IF_LET: i32 = 5;
        static WHILE_LET: i32 = 6;
        static FOR: i32 = 7;
        static MATCH: i32 = 8;
        static CLOSURE: i32 = 9;
        let _ = (
            PARAM, TUPLE, STRUCT, LET, ARRAY, IF_LET, WHILE_LET, FOR,
            MATCH, CLOSURE,
        );
    }
}
```

Expected: every moved static in `target` is renamed to its corresponding
`<name>_0`, and each use bound to it is rewritten. Destructuring, match,
if-let, while-let, for, and closure parameters are all HIR pattern bindings.

### A8-NAME-05 `local_value_items_and_const_generics_force_rename`

Input:

```rust
fn binders<const N: usize>() {
    const LOCAL_CONST: i32 = 0;
    fn LOCAL_FN() {}
    static LOCAL_STATIC: i32 = 0;
    let _ = (N, LOCAL_CONST, LOCAL_FN, LOCAL_STATIC);
}
mod target {
    pub fn f() {
        static N: i32 = 1;
        static LOCAL_CONST: i32 = 2;
        static LOCAL_FN: i32 = 3;
        static LOCAL_STATIC: i32 = 4;
        let _ = (N, LOCAL_CONST, LOCAL_FN, LOCAL_STATIC);
    }
}
```

Expected root output contains
`static LOCAL_STATIC_0: i32 = 0;` immediately before `binders`, whose tuple use
becomes `LOCAL_STATIC_0`. Expected target names are `N_0`, `LOCAL_CONST_0`,
`LOCAL_FN_0`, and `LOCAL_STATIC_1`, with exact target uses rewritten. The two
moved `LOCAL_STATIC` definitions receive suffixes in lexical order.

### A8-NAME-06 `non_value_names_do_not_force_rename`

Use this complete input, with the statics in a disjoint module:

```rust
struct FIELD_ONLY { NAME: i32 }
struct BRACED_TYPE { value: i32 }
type ALIAS_ONLY = i32;
trait TRAIT_ONLY { fn NAME(&self); const NAME_CONST: i32; }
impl FIELD_ONLY { fn METHOD_ONLY(&self) {} const ASSOCIATED_ONLY: i32 = 0; }
macro_rules! MACRO_ONLY { () => { 0 } }
fn lifetimes<'LIFETIME_ONLY>(x: &'LIFETIME_ONLY i32) { let _ = x; }
fn labels() { 'LABEL_ONLY: loop { break 'LABEL_ONLY; } }
mod target {
    pub fn f() {
        static NAME: i32 = 1;
        static BRACED_TYPE: i32 = 2;
        static ALIAS_ONLY: i32 = 3;
        static TRAIT_ONLY: i32 = 4;
        static METHOD_ONLY: i32 = 5;
        static ASSOCIATED_ONLY: i32 = 6;
        static MACRO_ONLY: i32 = 7;
        static LIFETIME_ONLY: i32 = 8;
        static LABEL_ONLY: i32 = 9;
        let _ = (
            NAME, BRACED_TYPE, ALIAS_ONLY, TRAIT_ONLY, METHOD_ONLY,
            ASSOCIATED_ONLY, MACRO_ONLY, LIFETIME_ONLY, LABEL_ONLY,
        );
    }
}
```

Expected: every target static keeps its original name. The braced named struct
introduces no value constructor. Fields, associated member names, type-only
items, lifetime parameters, labels, and macro-only names are excluded.

### A8-NAME-07 `suffix_allocation_uses_complete_original_set`

Input:

```rust
fn reserves(NAME: i32, NAME_0: i32, NAME_2: i32) {
    let _ = (NAME, NAME_0, NAME_2);
}
fn first() -> i32 { static NAME: i32 = 10; NAME }
fn second() -> i32 { static NAME: i32 = 20; NAME }
```

Expected:

```rust
fn reserves(NAME: i32, NAME_0: i32, NAME_2: i32) { /* unchanged */ }
static NAME_1: i32 = 10;
fn first() -> i32 { NAME_1 }
static NAME_3: i32 = 20;
fn second() -> i32 { NAME_3 }
```

The later original `NAME_2` is reserved before allocation begins, and the
first allocation is reserved before processing the second moved static.

### A8-NAME-08 `suffix_is_appended_to_complete_base_name`

Use these independent inputs:

```rust
fn reserve(NAME_0: i32) { let _ = NAME_0; }
fn f() -> i32 { static NAME_0: i32 = 1; NAME_0 }
```

```rust
fn reserve(r#type: i32) { let _ = r#type; }
fn f() -> i32 { static r#type: i32 = 1; r#type }
```

```rust
fn reserve(π: i32) { let _ = π; }
fn f() -> i32 { static π: i32 = 1; π }
```

Expected moved definition/use in the three subcases: `NAME_0_0` (not
`NAME_1`), `type_0`, and `π_0`, respectively.

### A8-NAME-09 `allocation_is_deterministic_across_runs`

Run `T` at least twice on A8-NAME-07 and A8-LIFT-05. Expected complete output
bytes from the canonical printer are equal across runs. Hash-map iteration,
DefId numeric values, module traversal implementation, or compiler query order
must not influence discovery or suffix assignment.

### A8-NAME-10 `standard_prelude_names_force_destination_rename`

Input under the harness's ordinary edition and implicit prelude:

```rust
mod target {
    pub fn resolved_from_prelude() -> Option<i32> {
        let value: Option<i32> = None;
        drop(0_i32);
        value
    }
    pub fn f() -> i32 {
        static None: i32 = 1;
        static drop: i32 = 2;
        None + drop
    }
}
```

Expected analysis: resolve the active prelude bindings used by
`target::resolved_from_prelude` through rustc, and find their semantic `None`
and `drop` names in the same destination module's prelude
allocation reservations. Do not assert whether their defining crate is spelled
`std` or `core`. Crate-binder multiplicity is exactly one for each name—the
corresponding moved static—and excludes the implicit-prelude binding. Prelude
collision nevertheless requires renaming. Expected relevant output:

```rust
mod target {
    pub fn resolved_from_prelude() -> Option<i32> {
        let value: Option<i32> = None;
        drop(0_i32);
        value
    }
    static None_0: i32 = 1;
    static drop_0: i32 = 2;
    pub fn f() -> i32 { None_0 + drop_0 }
}
```

The existing `None` and `drop` paths in `resolved_from_prelude` are unchanged
and still resolve to the prelude definitions. Only paths whose compiler
identity is one of the moved static `DefId`s are rewritten.

### A8-NAME-11 `no_std_uses_the_compiler_selected_core_prelude`

Input:

```rust
#![no_std]
fn probe() {
    let value: Option<i32> = None;
    drop(0_i32);
    let _ = value;
}
fn f() -> i32 {
    static None: i32 = 1;
    static drop: i32 = 2;
    None + drop
}
```

Expected: compiler resolution identifies the active `no_std` prelude ValueNS
bindings for `None` and `drop`; both are allocation reservations but add no
crate-binder multiplicity. They independently collide with the two original
static names, so output contains `static None_0`, `static drop_0`, and
`fn f() -> i32 { None_0 + drop_0 }`. The paths in `probe` remain spelled
`None`/`drop`, retain their compiler-resolved core-prelude identities, and the
output compiles as `no_std`. The test contains no hardcoded list of
standard-prelude paths or DefIds.

### A8-NAME-12 `disabled_and_custom_preludes_follow_resolution`

First input:

```rust
#![no_implicit_prelude]
fn f() -> i32 {
    static None: i32 = 1;
    static drop: i32 = 2;
    None + drop
}
```

Expected analysis: the destination module has no implicit-prelude reservation
for `None` or `drop`; each crate-binder multiplicity is one and both statics
lift unchanged.

On the pinned compiler supporting its internal custom-prelude attribute, also
use this compiler fixture:

```rust
#![feature(prelude_import)]
#![no_implicit_prelude]
mod custom_prelude { pub static NAME_0: i32 = 0; }
#[prelude_import]
use crate::custom_prelude::*;
fn reserve(NAME: i32) { let _ = NAME; }
fn f() -> i32 { static NAME: i32 = 1; NAME }
```

Expected analysis obtains `NAME_0` from the compiler's active custom-prelude
resolution and includes it in the destination's allocation reservations. The
compiler-recognized `#[prelude_import]` is prelude machinery rather than an
ordinary crate-owned import binder; an ordinary source `use ...::*` remains
covered by A8-NAME-02. No prelude entry contributes binder multiplicity.
Because `NAME` collides with `reserve`'s parameter and candidate `NAME_0` is
occupied by the active custom prelude, output is
`static NAME_1: i32 = 1; fn f() -> i32 { NAME_1 }`. If the pinned compiler's
test harness cannot author a custom prelude directly, exercise the same
resolved prelude-entry input through the internal analysis seam; do not replace
it with a hardcoded production name list.

Independently prove suffix reservation at the allocation helper's internal
seam with this compiling source shape:

```rust
fn reserve(NAME: i32) { let _ = NAME; }
fn f() -> i32 { static NAME: i32 = 1; NAME }
```

Provide the analyzed allocation inputs exactly as follows:

```text
crate-owned binders named NAME = {reserve::NAME, f::NAME}
crate-owned binders named NAME_0 = {}
earlier allocated names = {}
active prelude ValueNS names at f's destination module = {NAME_0}
```

Expected allocation is `f::NAME -> NAME_1`, and the transformed definition and
bound use are `static NAME_1: i32 = 1;` and `fn f() -> i32 { NAME_1 }`. The
control input with the same crate-owned sets but an empty active-prelude set
allocates `NAME_0`. Thus no crate-owned `NAME_0` binder can explain the skip;
the active destination-prelude reservation independently does so. This is a
test-only call to the internal allocator with an already compiler-classified
prelude-name set, not permission to hardcode `NAME_0` in production.

## 8. Compiler-identity use rewriting

### A8-REF-01 `rewrites_all_resolved_uses_of_renamed_static`

Input:

```rust
fn reserve(CELL: i32) { let _ = CELL; }
fn f(flag: bool) -> *const i32 {
    static mut CELL: i32 = 1;
    unsafe {
        let before = CELL;
        CELL = before + 1;
        if flag { &raw const CELL } else { core::ptr::addr_of!(CELL) }
    }
}
```

Expected lifted definition is `static mut CELL_0: i32 = 1`. Every read,
assignment target, raw-address operand, and expanded `addr_of!` occurrence that
resolves to that `DefId` refers to `CELL_0`. The unrelated parameter binder and
its use in `reserve` remain `CELL`. In a separate `x86_64`-gated compiling
subcase, use `core::arch::asm!("/* {0} */", sym CELL)` on a colliding local
static; the mapped `sym` operand must likewise become `CELL_0`.

### A8-REF-02 `same_spelling_other_definitions_are_not_rewritten`

Input:

```rust
mod a { pub fn f() -> i32 { static X: i32 = 1; X } }
mod b { pub static X: i32 = 2; pub fn g() -> i32 { X } }
fn h(X: i32) -> i32 { X }
```

Expected: only `a`'s moved static and its bound use become `X_0`. `b::X`, the
use in `b::g`, parameter `X`, and the use in `h` are unchanged. This must be
decided by compiler identity, never text replacement.

### A8-REF-03 `rewrites_references_between_lifted_statics`

Input:

```rust
fn reserve(A: i32, B: i32) { let _ = (A, B); }
fn f() -> i32 {
    static A: i32 = B + 1;
    static B: i32 = 2;
    A + B
}
```

Expected:

```rust
static A_0: i32 = B_0 + 1;
static B_0: i32 = 2;
fn f() -> i32 { A_0 + B_0 }
```

Forward references remain valid after lifting, and definitions, initializer
paths, and body paths use the same allocation map.

### A8-REF-04 `unrenamed_static_uses_remain_spelled_the_same`

Input:

```rust
fn f() -> i32 { static UNIQUE: i32 = 1; UNIQUE + UNIQUE }
```

Expected: `static UNIQUE` is lifted and both uses remain `UNIQUE`; no synthetic
suffix is added merely because the static itself contributes one binder.

## 9. Initializer and type dependency validation

Module-owned dependencies and other statics lifted by the same pass are
allowed. Any function/block-scoped import, const, type, function, module, or
other local item required to resolve a moved static's explicit type or
initializer is rejected. Compiler-invalid attempts to capture a parameter,
ordinary local, type parameter, or outer `Self` fail in rustc before the pass
callback and are not redefined as `PrepareError`s.

### A8-DEP-01 `allows_module_and_external_dependencies`

Input:

```rust
use core::sync::atomic::{AtomicUsize, Ordering};
type Word = usize;
const START: Word = 3;
const fn make() -> Word { START }
fn f() -> Word {
    static COUNT: AtomicUsize = AtomicUsize::new(make());
    COUNT.load(Ordering::Relaxed)
}
```

Expected: `COUNT` moves before `f` unchanged, and module imports, `Word`,
`START`, `make`, `AtomicUsize`, and `Ordering` remain resolvable. No
module-owned dependency forces rejection.

### A8-DEP-02 `allows_dependencies_between_lifted_statics`

Input:

```rust
fn f() -> &'static i32 {
    static VALUE: i32 = 9;
    static REFERENCE: &i32 = &VALUE;
    REFERENCE
}
```

Expected:

```rust
static VALUE: i32 = 9;
static REFERENCE: &i32 = &VALUE;
fn f() -> &'static i32 { REFERENCE }
```

The dependency is allowed even when the source order is reversed, subject to
ordinary Rust static-initializer validity.

Repeat with
`static VALUE: i32 = { const INNER: i32 = 9; INNER };`. Expected: the complete
initializer block and its own `INNER` definition move as part of `VALUE` and
do not trigger scoped-dependency rejection.

### A8-DEP-03 `rejects_function_scoped_const_dependency`

Input:

```rust
fn f() -> usize {
    const N: usize = 4;
    static DATA: [u8; N] = [0; N];
    DATA.len()
}
```

Expected: `Err(DATA, N)`. The display diagnostic names `DATA` and `N`; there is
no returned partially wrapped or lifted source.

### A8-DEP-04 `rejects_function_scoped_type_and_constructor_dependencies`

Use independent inputs:

```rust
fn f() {
    struct Local { value: i32 }
    static DATA: Local = Local { value: 1 };
    let _ = DATA.value;
}
```

```rust
fn g() {
    type Local = i32;
    static DATA: Local = 1;
    let _ = DATA;
}
```

Expected: `Err(DATA, Local)` for each. Both a local type reference and a local
constructor/value dependency are scoped dependencies; one error is enough to
reject the complete crate deterministically.

### A8-DEP-05 `rejects_function_scoped_function_and_module_dependencies`

Use independent inputs:

```rust
fn f() -> i32 {
    const fn local() -> i32 { 1 }
    static DATA: i32 = local();
    DATA
}
```

```rust
fn g() -> i32 {
    mod local { pub const N: i32 = 1; }
    static DATA: i32 = local::N;
    DATA
}
```

Expected: `Err(DATA, local)` in each case, identifying the local function or
local module definition on which the initializer path depends.

### A8-DEP-06 `rejects_block_scoped_explicit_import`

Input:

```rust
mod source { pub const N: usize = 2; }
fn f() -> usize {
    use crate::source::N as LOCAL_N;
    static DATA: [u8; LOCAL_N] = [0; LOCAL_N];
    DATA.len()
}
```

Expected: `Err(DATA, LOCAL_N)`. The ultimate target `source::N` is module-owned,
but the spelling required after lifting is introduced by a function-scoped
import and therefore is not usable at module scope.

### A8-DEP-07 `rejects_block_scoped_glob_import`

Input:

```rust
mod source { pub const N: usize = 2; }
fn f() -> usize {
    use crate::source::*;
    static DATA: [u8; N] = [0; N];
    DATA.len()
}
```

Expected: `Err(DATA, N)` whose dependency identifies the scoped glob/import
binding rather than silently treating the resolved target as sufficient.

### A8-DEP-08 `fully_qualified_path_ignores_unrelated_local_import`

Input:

```rust
mod source { pub const N: usize = 2; }
fn f() -> usize {
    use crate::source::N as UNUSED;
    static DATA: [u8; crate::source::N] = [0; crate::source::N];
    DATA.len()
}
```

Expected: `DATA` lifts successfully and keeps its fully qualified paths. An
unrelated local import does not cause blanket rejection.

### A8-DEP-09 `one_invalid_static_rejects_all_changes`

Input:

```rust
fn good(x: i32) -> i32 {
    static GOOD: i32 = 1;
    match x { 0 => GOOD, _ => x }
}
fn bad() -> usize {
    const N: usize = 2;
    static BAD: [u8; N] = [0; N];
    BAD.len()
}
```

Expected: `Err(BAD, N)`. The pass returns no source. In a CLI/adapter failure
simulation, the input project source remains byte-for-byte unchanged: `GOOD`
is not lifted, neither match arm is committed as a block, and `BAD` is not
removed. This proves whole-pass preflight and atomicity.

### A8-DEP-10 `missing_compiler_mapping_is_structured_and_atomic`

At the narrow internal analysis seam, start from the compiling A8-REF-01 AST
and its complete AST-to-HIR map, then remove separately the mapping for the
local static definition and for one path that resolves to it. Expected in each
subcase: `PrepareError::MissingMapping` identifies the static or path context,
the display diagnostic names that construct, and no transformed source is
returned. This uses an internal test helper or accepts the map as an analysis
argument; it must not require malformed Rust, filesystem state, or a new public
test-only API.

### A8-DEP-11 `compiler_rejection_is_not_a_prepare_error`

Use these independent compiler-invalid inputs:

```rust
fn f(value: i32) -> i32 {
    static DATA: i32 = value;
    DATA
}
```

```rust
fn f<const N: usize>() -> usize {
    static DATA: [u8; N] = [0; N];
    DATA.len()
}
```

Expected: `run_compiler_on_str` returns its outer compiler `Err` for the
illegal runtime-local capture and outer generic use before `prepare` can
return an inner result. Neither failure is
`PrepareError::ScopedDependency`/`MissingMapping`, neither diagnostic gets a
`prepare failed:` prefix, and no transformed source exists. This keeps rustc
input rejection distinct from valid-input preparation rejection.

## 10. Export-name preservation

The five cases in this section assert the immediate output of `prepare` only.
They preserve external symbols across lifting/renaming at that pass boundary;
they do not override the later `unsafe` pass's existing configured policy.
A8-INT-05 separately asserts the actual local-pipeline interaction.

### A8-EXPORT-01 `unrenamed_no_mangle_static_keeps_no_mangle`

Input:

```rust
fn f() -> i32 {
    #[no_mangle]
    static EXPORTED: i32 = 1;
    EXPORTED
}
```

Expected moved item remains exactly `#[no_mangle] static EXPORTED: i32 = 1;`.
No `export_name` is added because the Rust identifier did not change.

### A8-EXPORT-02 `renamed_no_mangle_static_preserves_original_symbol`

Input:

```rust
fn reserve(EXPORTED: i32) { let _ = EXPORTED; }
fn f() -> i32 {
    #[no_mangle]
    static EXPORTED: i32 = 1;
    EXPORTED
}
```

Expected moved item/use:

```rust
#[export_name = "EXPORTED"]
static EXPORTED_0: i32 = 1;
fn f() -> i32 { EXPORTED_0 }
```

The `no_mangle` attribute is removed from the renamed Rust item and replaced by
exactly one `export_name` containing the original symbol.

### A8-EXPORT-03 `explicit_export_name_is_preserved_across_rename`

Input:

```rust
fn reserve(INTERNAL: i32) { let _ = INTERNAL; }
fn f() -> i32 {
    #[export_name = "wire_symbol"]
    static INTERNAL: i32 = 1;
    INTERNAL
}
```

Expected moved item is
`#[export_name = "wire_symbol"] static INTERNAL_0: i32 = 1;`, with the body
using `INTERNAL_0`. The explicit linked symbol is neither replaced with
`"INTERNAL"` nor derived from the new Rust name.

### A8-EXPORT-04 `rename_without_export_attribute_adds_none`

Input:

```rust
fn reserve(PRIVATE: i32) { let _ = PRIVATE; }
fn f() -> i32 { static PRIVATE: i32 = 1; PRIVATE }
```

Expected: `static PRIVATE_0: i32 = 1;` with no `no_mangle`, `export_name`, or
other invented linkage attribute. Renaming a private Rust item does not create
an exported symbol.

### A8-EXPORT-05 `non_export_attributes_survive_export_conversion`

Input is A8-EXPORT-02 with `#[used]` and `#[link_section = ".demo"]` adjacent
to `#[no_mangle]`. Expected: both unrelated attributes remain exactly once and
in their original relative order; only `no_mangle` is replaced by
`#[export_name = "EXPORTED"]`.

## 11. Pass, CLI, adapter, and checked-in configuration

### A8-WIRE-01 `pass_parses_from_cli_and_toml`

In pure `crat` binary unit tests:

- `Args::try_parse_from(["crat", "--pass", "prepare", "input"])` succeeds and
  yields exactly one `Pass::Prepare`;
- TOML `passes = ["prepare"]` deserializes to exactly one `Pass::Prepare`; and
- an unknown spelling still fails under existing clap/serde behavior.

There is no `[prepare]` config table and no prepare-specific CLI option.

### A8-WIRE-02 `pass_dispatch_uses_preparer_without_side_effects`

The `Pass::Prepare` match arm invokes `run_compiler_on_path` once with
`preparer::prepare`, writes its successful returned source to the current
library file once, requests no Cargo dependency, and performs no other
filesystem mutation. This is established by code-level dispatch inspection
plus the pass-library tests; repository rules prohibit a Crat CLI filesystem
test. Assert all three nested-result cases: outer `Ok(Ok(source))` writes once;
outer `Ok(Err(PrepareError))` prints `prepare failed: ...`, exits nonzero, and
writes nothing; outer compiler `Err` follows the pre-existing compiler-failure
path, never acquires the prepare prefix, and writes nothing.

### A8-WIRE-03 `adapter_accepts_explicit_prepare`

Input:

```python
resolve_pass_plan({"passes": ["enum", "prepare", "simpl"]})
```

Expected exact result:

```python
[("enum", []), ("prepare", []), ("simpl", [])]
```

`prepare` has no built-in flags. A `pass_args.prepare` list follows existing
generic replacement semantics; an unselected prepare entry still produces
`pass_args contains passes not selected for this run`.

### A8-WIRE-04 `final_pass_prepare_has_an_opt_in_prefix`

Input `resolve_pass_plan({"final_pass": "prepare"})` produces exactly:

```python
[
    ("expand", []),
    ("extern", ["--extern-ignore-return-type", "--extern-ignore-param-type"]),
    ("preprocess", []),
    ("outparam", ["--outparam-simplify"]),
    ("punning", []),
    ("enum", []),
    ("prepare", []),
]
```

Input `{"final_pass": "pointer"}` remains the old prefix ending
`enum, pointer` with no `prepare`. This proves that adding an opt-in chain edge
does not insert the pass into later canonical defaults.

### A8-WIRE-05 `adapter_command_contains_exact_prepare_pass`

Use the existing fake `run_logged` adapter harness with plugin `prepare` and
flags `[]`. Expected command has the existing environment/output arguments and
exactly `--pass prepare` for selection, with no prepare-specific argument. The
fake output directory is threaded normally to the next selected pass.

### A8-WIRE-06 `local_pipeline_order_is_exact`

Parse `proctor/configs/c2rust_crat_local.toml`. Expected
`stages.crat.config.passes` exactly:

```python
[
    "expand", "extern", "preprocess", "enum", "prepare", "simpl",
    "unsafe", "unexpand", "split", "bin",
]
```

`prepare` occurs once, immediately after `enum` and before `simpl`. `libc` is
absent. `pass_args` has no prepare or libc entry. Pipeline order, LLM settings,
timeouts, and all other configuration remain unchanged.

### A8-WIRE-07 `adapter_defaults_and_manifest_are_unchanged`

Compare adapter `stage.toml` and `resolve_pass_plan({})` with their prior
contract. Expected: no new stage config key; default `final_pass = "bin"` and
all existing pass flags remain unchanged; `prepare` is merely an accepted pass
name. No stage envelope/schema/version, artifact kind, `proctor.toml`, prompt,
metric, or local-transformation protocol changes.

### A8-WIRE-08 `libc_is_completely_out_of_scope`

Regression oracle: `libc_replacer` source/tests, `Pass::Libc` dispatch, adapter
libc defaults, and every checked-in config other than the one explicit pass
insertion are byte-for-byte unchanged. No ctype-only option, CLI flag, config
field, or new `libc` occurrence is introduced.

## 12. Idempotence and pass interactions

### A8-INT-01 `prepare_is_idempotent`

Input combines A8-MATCH-04, A8-LIFT-02, a renamed static, and an exported
static. Run `prepare` once, compile, then run it again. Expected complete
second canonical AST equals the first: no additional blocks, moves, suffixes,
or linkage changes. In particular `NAME_0` does not become `NAME_0_0` and an
introduced `export_name` is not duplicated.

### A8-INT-02 `enum_prepare_simpl_sequence_compiles`

Input:

```rust
type Kind = core::ffi::c_uint;
const ZERO: Kind = 0;
const ONE: Kind = 1;
fn classify(x: Kind) -> i32 {
    static CALLS: i32 = 1;
    match x { ZERO => CALLS, ONE => CALLS + 1, _ => 0 }
}
```

Run the actual library entry points sequentially:
`enum_replacer::replace_enums`, `preparer::prepare`, then
`simplifier::simplify`, compiling between passes if the harness supports it and
always after the final pass. Expected final invariants:

- no function-local static remains;
- every match arm body is a block;
- the lifted static is in `classify`'s module and all uses resolve to it; and
- enum and simplifier transformations retain their existing semantics.

Do not assert incidental enum formatting beyond those invariants and the
existing enum tests.

### A8-INT-03 `prepare_postcondition_matches_local_tool_preconditions`

For each source in A8-UPD-02, run `prepare` directly. Expected output compiles,
contains no function-local static, and contains no non-block match arm. This
is the exact syntactic boundary consumed later by local transformation. The
test stays in the passes crate; it does not add a `passes -> tools` dependency
or invoke `crat-tool`.

### A8-INT-04 `inline_modules_survive_later_split_boundary`

Input is A8-LIFT-05. Expected prepared AST places each lifted static inside its
own nearest inline module, never at crate root. Existing splitter tests remain
unchanged and later splitting therefore writes each static with that module's
contents. No new splitter filesystem test is required.

### A8-INT-05 `configured_unsafe_policy_follows_prepare_exports`

Use this combined input, with `f`, `g`, and `h` included in
`c_exposed_fns` so existing unused-function policy does not obscure the
attribute assertions:

```rust
fn reserve_renamed(RENAMED: i32) { let _ = RENAMED; }
fn reserve_internal(INTERNAL: i32) { let _ = INTERNAL; }
pub fn f() -> i32 {
    #[no_mangle]
    static UNCHANGED: i32 = 1;
    UNCHANGED
}
pub fn g() -> i32 {
    #[no_mangle]
    static RENAMED: i32 = 2;
    RENAMED
}
pub fn h() -> i32 {
    #[export_name = "wire_symbol"]
    static INTERNAL: i32 = 3;
    INTERNAL
}
```

First run `prepare`. Its exact relevant output follows Section 10:
`UNCHANGED` retains `#[no_mangle]`, renamed `RENAMED_0` has
`#[export_name = "RENAMED"]`, and renamed `INTERNAL_0` retains
`#[export_name = "wire_symbol"]`.

Then run `unsafe_resolver::resolve_unsafe` with the actual adapter defaults:
`remove_unused = true`, `remove_no_mangle = true`,
`remove_extern_c = true`, and `replace_pub = true`, plus the stated exposed
function set. Expected output compiles; all three statics remain module items;
`UNCHANGED` has no `no_mangle` or `export_name`; `RENAMED_0` still has exactly
`#[export_name = "RENAMED"]`; and `INTERNAL_0` still has exactly
`#[export_name = "wire_symbol"]`. This is existing `unsafe` behavior, not a
request to change that pass or its adapter flags.

Independently run the configured `unexpand` behavior (`use_print = true`) on
A8-MATCH-01 and A8-LIFT-07 prepared output. Expected: it compiles, inserted
blocks remain blocks, lifted statics remain module items, and no
function-local static reappears.

## 13. Regression and completion matrix

### A8-REG-01 `no_static_input_is_unchanged`

Run current representative `passes` fixtures without local statics or
non-block match arms through `prepare`. Expected: same AST and no dependency or
name-allocation side effects.

### A8-REG-02 `top_level_statics_are_never_renamed_by_prepare`

Input:

```rust
mod a { pub static DUP: i32 = 1; }
mod b { pub static DUP: i32 = 2; }
```

Expected: `Same(input)`. Crate-wide collision rules choose names only for
moved statics; existing module statics are not renamed or relocated.

### A8-REG-03 `output_is_source_ordered_and_buildable`

Combine multiple root functions, inline modules, methods, nested statics,
matches, allowed dependencies, and collisions from Sections 5--10. Expected:
two repeated runs produce the same module-item order and exact canonical
source, and the complete result type-checks. No generated name occurs in the
original crate-binder occupied set, its destination module's active prelude
reservation set, or the earlier allocation set.

### A8-REG-04 `no_contract_or_dependency_change`

Expected unchanged Cargo dependencies/features, stage manifests, schemas,
artifact paths, `config.toml` and `proctor.toml` models, local-transformation
prompt/records, output metrics, and tool subcommands. The pass returns only
transformed Rust source or a structured preparation error.

Implementation is complete only when:

| Requirement | Cases |
| --- | --- |
| every non-block match arm gets one block | A8-MATCH-01--07 |
| all agreed function-like scopes lift to the correct module | A8-LIFT-01--07 |
| every binder class and resolved prelude reservation is enforced | A8-NAME-01--12 |
| renaming follows DefId and rewrites only bound occurrences | A8-REF-01--04 |
| allowed dependencies work and both error layers stay distinct | A8-DEP-01--11 |
| prepare-boundary exports and later unsafe policy are exact | A8-EXPORT-01--05, A8-INT-05 |
| CLI/serde/adapter/config wiring is exact and opt-in | A8-WIRE-01--08 |
| idempotence and neighboring pass contracts hold | A8-INT-01--05 |
| existing tools defenses and unrelated behavior do not regress | A8-UPD-01--03, A8-REG-01--04 |
