# Phase 7 Rule-Application Examples

## Metavariables

| Form | Sort |
|---|---|
| `[A0]` | anchor binding identity |
| `[B0]` | ordinary binding identity |
| `[E0]` | expression |
| `[N0]` | integer magnitude |
| `[F0]` | function identity |
| `[M0]` | method identity |

## Pattern specificity

| ID | Left pattern | Relation | Right pattern |
|---|---|---|---|
| S1 | `*[A0].offset(1)` | left is more specific | `*[A0].offset([N0])` |
| S2 | `*[A0].offset([B0] as isize)` | left is more specific | `*[A0].offset([E0])` |
| S3 | `*[A0].offset([E0] + [E0])` | left is more specific | `*[A0].offset([E0] + [E1])` |
| S4 | `*[A0].offset([E0] + 1)` | left is more specific | `*[A0].offset([E0])` |
| S5 | `[A0].[M0]([B0] as isize)` | left is more specific | `[A0].[M0]([E0])` |
| S6 | `[F0]([A0], [E0] + 1)` | left is more specific | `[F0]([A0], [E0])` |
| S7 | `*[A0].offset([E0])` | equally specific | `*[A0].offset([E0])` |
| S8 | `*[A0].offset([E0] - [E1])` | equally specific by alpha-renaming | `*[A0].offset([E1] - [E0])` |
| S9 | `*[A0].offset(1)` | incomparable | `*[A0].offset(2)` |
| S10 | `*[A0].offset([E0] + 1)` | incomparable | `*[A0].offset(1 + [E0])` |
| S11 | `*[A0].offset([E0])` | incomparable | `*[A0].add([E0])` |
| S12 | `*[A0].offset([E0] + [E0])` | incomparable | `*[A0].offset([E0] + 1)` |
| S13 | `(*[A0]).field` | incomparable | `*[A0][[E0]]` |
| S14 | `[A0].is_null()` | incomparable | `[A0].[M0]()` |
| S15 | `libc::free([A0])` | incomparable | `[F0]([A0])` |

## Source-pattern applicability

| ID | Source pattern | Rust expression region | Result |
|---|---|---|---|
| A1 | `*[A0].offset([E0] as isize)` | `*p.offset(i as isize)` | Applicable: `[A0] = p`, `[E0] = i`. |
| A2 | `*[A0].offset([E0] as isize)` | `*p.offset((i + 1) as isize)` | Applicable: `[A0] = p`, `[E0] = i + 1`. |
| A3 | `*[A0].offset(([E0] + [E0]) as isize)` | `*p.offset((i + i) as isize)` | Applicable: `[A0] = p`, `[E0] = i`. |
| A4 | `*[A0].offset(([E0] + [E0]) as isize)` | `*p.offset((i + j) as isize)` | Not applicable because the repeated `[E0]` occurrences would require different substitutions. |
| A5 | `*[A0].offset([E0])` | `*p.offset(p as isize)` | Not applicable because `[E0]` would contain the anchor binding `p`. |
| A6 | `*[A0].offset([B0] as isize)` | `*p.offset(p as isize)` | Not applicable because distinct anchor and binding variables would bind the same local identity. |
| A7 | `*[A0].offset(if [B0] { [E0] } else { 0 })` | `*p.offset(if flag { flag as isize } else { 0 })` | Not applicable because `flag` would be carried by both `[B0]` and `[E0]`. |
| A8 | `*[A0].offset([E0] + [E1])` | `*p.offset((i as isize) + (i as isize))` | Not applicable because `i` would be split across the distinct carriers `[E0]` and `[E1]`. |
| A9 | `*[A0].offset([E0] + [E1])` | `*p.offset(1 + 1)` | Applicable: `[A0] = p`, `[E0] = 1`, `[E1] = 1`. |
| A10 | `*[A0].offset([N0])` | `*p.offset(4)` | Applicable: `[A0] = p`, `[N0] = 4`. |
| A11 | `*[A0].offset(1)` | `*p.offset(2)` | Not applicable because the fixed integer magnitudes differ. |
| A12 | `*[A0].offset([E0])` | `*p.add(i)` | Not applicable because the fixed method identities differ. |
| A13 | `*[A0] == *[A0]` | `*p == *p` | Applicable: both anchor occurrences bind `p`. |
| A14 | `*[A0] == *[A0]` | `*p == *q` | Not applicable because the repeated `[A0]` occurrences would bind different locals. |
| A15 | `*[A0].offset(([B0] - [B1]) as isize)` | `*p.offset((i - i) as isize)` | Not applicable because distinct binding variables must bind distinct local identities. |
| A16 | `*[A0].offset([E0])` | `*p.offset((i + 1) as isize)` | Applicable: the non-anchor local `i` has only the carrier `[E0]`. |

## Target-type inference

| ID | Rust context containing the region `□` | Context-side target types and declarations | Inferred context-required type |
|---|---|---|---|
| T1 | `let q: *mut i32 = □;` | `q: Option<&mut i32>` | `Option<&mut i32>` |
| T2 | `q = □;` | `q: &mut [i32]` | `&mut [i32]` |
| T3 | `*out = □;` | `out: &mut Option<&i32>` | `Option<&i32>` |
| T4 | `(*holder).ptr = □;` | `holder: &mut Holder`, `Holder::ptr: *mut i32` | `*mut i32` |
| T5 | `consume(□);` | `fn consume(_: &mut [i32])` | `&mut [i32]` |
| T6 | `ffi_consume(□);` | `unsafe extern "C" fn ffi_consume(_: *mut i32)` | `*mut i32` |
| T7 | `return □;` | current return type `Option<&i32>` | `Option<&i32>` |
| T8 | `{ update(); □ }` | current return type `Box<[i32]>` | `Box<[i32]>` |
| T9 | `Holder { ptr: □ }` | `Holder::ptr: *mut i32` | `*mut i32` |
| T10 | `consume::<i32>(□);` | `fn consume<T>(_: &mut [T])` | `&mut [i32]` |
| T11 | `fp(□);` | `fp: fn(*mut i32)` | unavailable |
| T12 | `*get_slot() = □;` | `fn get_slot() -> &'static mut Option<&'static i32>` | `Option<&'static i32>` |
| T13 | `sink.consume(□);` | `sink: Sink` | unavailable |
| T14 | `□;` | — | unavailable |
| T15 | `let r = if flag { □ } else { q };` | `q: Option<&i32>` | unavailable |
| T16 | `slots[i] = □;` | `slots: &mut [Option<&i32>]` | `Option<&i32>` |
| T17 | `(q) = □;` | `q: Box<i32>` | `Box<i32>` |
| T18 | `*slot = □;` | `slot: Box<Option<&i32>>` | `Option<&i32>` |
| T19 | `*out = □;` | `out: Option<&mut Option<&i32>>` | `Option<&i32>` |

## Rule application results

| ID | Source pattern | Target pattern | Rust expression region | Expression after application |
|---|---|---|---|---|
| R1 | `*[A0].offset([E0] as isize)` | `[A0][[E0] as usize]` | `*p.offset(i as isize)` | `p[i as usize]` |
| R2 | `*[A0].add([E0])` | `[A0][[E0]]` | `*p.add(i)` | `p[i]` |
| R3 | `[A0].is_null()` | `[A0].is_none()` | `p.is_null()` | `p.is_none()` |
| R4 | `![A0].is_null()` | `[A0].is_some()` | `!p.is_null()` | `p.is_some()` |
| R5 | `&mut *[A0]` | `[A0]` | `&mut *p` | `p` |
| R6 | `&*[A0]` | `[A0]` | `&*p` | `p` |
| R7 | `*[A0]` | `*[A0].as_deref().unwrap()` | `*p` | `*p.as_deref().unwrap()` |
| R8 | `*[A0]` | `*[A0].as_deref_mut().unwrap()` | `*p` | `*p.as_deref_mut().unwrap()` |
| R9 | `(*[A0]).value` | `[A0].value` | `(*p).value` | `p.value` |
| R10 | `[F0]([A0])` | `[F0]([A0].as_deref().unwrap())` | `read(p)` | `read(p.as_deref().unwrap())` |
| R11 | `[A0].offset([E0] as isize)` | `&[A0][[E0] as usize]` | `p.offset(i as isize)` | `&p[i as usize]` |
| R12 | `*[A0].offset([E0] as isize)` | `[A0].as_deref().unwrap()[[E0] as usize]` | `*p.offset(i as isize)` | `p.as_deref().unwrap()[i as usize]` |
| R13 | `*[A0].offset(([E0] + [N0]) as isize)` | `[A0][[E0] + [N0]]` | `*p.offset((i + 1) as isize)` | `p[i + 1]` |
