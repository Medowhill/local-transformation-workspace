# Amendment 9 Test Plan: proctor-libc 0.3.0 Integration

## 1. Purpose and authority

This plan is the exhaustive acceptance specification for
[amendment-9-plan.md](amendment-9-plan.md). It covers four coordinated changes:

1. every dependency introduced by local transformation or by Crat `prepare`
   uses `proctor-libc` 0.3.0 as its minimum crates.io requirement;
2. exact compiler-resolved, literal `printf` statements support the static
   `d/i/u/o/x/X/f/F/e/E/g/G/a/A/s` formatting domain exposed by
   `proctor_libc::printf`;
3. each function skeleton reports its exact consuming source specifiers so the
   version-1 local-transformation prompt can explain only the relevant
   wrappers, and the five `strto*` entries explain both immutable and mutable
   proctor-libc functions; and
4. ordinary Crat `prepare` adds syntax-directed ctype replacement modeled on
   the libc pass's existing direct `tolower`/`toupper` and
   `__ctype_b_loc`-mask logic, and extends that style to the stated direct and
   table forms using fully qualified proctor-libc calls, without an LLM.

The following choices are fixed for this amendment:

- only `printf` is in the formatting scope, not `fprintf`, `sprintf`,
  `snprintf`, `vprintf`, or any other family member;
- wrapper use is instructed by the prompt, not structurally enforced by the
  validator;
- exact specifiers are function-level skeleton metadata and are unioned over
  the current SCC in Python;
- skeleton JSON remains an internal lockstep format, while prompt, validation,
  replacement, observation, and rule documents remain version 1; and
- all direct ctype functions and the supported table idioms are in scope, and
  classification results may normalize any nonzero C result to `1`.

The existing `proctor-libc` 0.3.0 tests are the authority for wrapper and ctype
runtime semantics. These tests verify that PROCTOR and Crat select those APIs,
preserve the trusted Rust format, and carry the new metadata correctly. They
must not copy the implementation of the wrappers into Crat or Python tests.

Current source and executable tests remain authoritative if they reveal a
factual discrepancy. A discrepancy that changes the behavior below requires a
plan decision; it must not be resolved silently. Planning identifiers such as
`A9-*` and “Amendment 9” are documentation-only and must not occur in code,
test names, fixtures, diagnostics, configuration, or generated files.

## 2. Test ownership and execution

Rust tests belong beside their implementation:

- format parsing, exact `printf` resolution, and template validation:
  `proctor/stages/crat/crates/tools/src/printf.rs`;
- function records, format-metadata collection, dual views, and templates:
  `crates/tools/src/skeleton.rs` and its existing test modules;
- trusted structure and replacement:
  `crates/tools/src/validator.rs`, `preservation.rs`, and
  `item_replacer.rs`;
- observations and rules: `crates/tools/src/observation.rs` and `rule.rs`;
- ctype preparation: `crates/passes/src/preparer/tests.rs`;
- pure dependency normalization: `crates/utils/src/dependency.rs`;
- pure preparation publication ordering: `crates/passes/src/preparer/tests.rs`.

Crat tests call library APIs and in-memory compiler harnesses. They must not
invoke `crat-tool` or `crat`, mutate a project tree, or use a Crat-root
`tests/` directory. Python skeleton loading, prompt selection, SCC behavior,
dependency normalization, and fake-tool integration tests belong in
`proctor/tests/test_local_transformation.py`.

Run from `proctor/stages/crat`:

```bash
cargo test -p tools printf::tests
cargo test -p tools skeleton::tests
cargo test -p tools validator::tests
cargo test -p tools item_replacer::tests
cargo test -p tools observation::tests
cargo test -p tools rule::tests
cargo test -p tools
cargo test -p passes preparer::tests
cargo test -p utils dependency::tests
cargo test -p passes
cargo test --workspace
cargo fmt
cargo clippy --workspace --all-targets
```

Run from `proctor`:

```bash
uv run pytest tests/test_local_transformation.py
uv run pytest
uv run ruff check .
uv run ruff format --check .
uv run mypy proctor
uv run proctor validate -c configs/c2rust_crat_local.toml
```

Also run these offline metadata checks from `proctor/stages/crat` after the
checked-in dependency manifest and lock file are updated:

```bash
cargo metadata --locked --offline --format-version 1 --manifest-path deps_crate/Cargo.toml
cargo check --locked --offline --manifest-path deps_crate/Cargo.toml
```

The default suites remain deterministic, offline, API-key-free, and free of a
real C2Rust/CRAT subprocess. An optional end-to-end translated fixture may be
run after the unit matrix, but it does not replace any case below.

## 3. Shared notation and exact-comparison policy

`C("...")` means run the pure C-format converter on the decoded bytes before
the single terminal NUL. `Ok("rust", ["spec", ...])` means the exact semantic
Rust format and ordered exact consuming source specifiers. `No` means atomic
`Unsupported`: no prefix, partial template, or partial metadata is returned.

Unless a case says otherwise, compiler tests declare exactly this local symbol:

```rust
unsafe extern "C" {
    fn printf(format: *const ::std::os::raw::c_char, ...)
        -> ::std::os::raw::c_int;
}
```

and use this recoverable C2Rust literal shape:

```rust
b"value=%d\n\0" as *const u8 as *const ::std::os::raw::c_char
```

For a successful consuming statement, the canonical skeleton statement is:

```rust
#[proctor(0)]
::std::print!("<semantic Rust format>", todo!());
```

with one `todo!()` per consuming specifier in source order. Within each fixed
fixture, the semantic literal, macro path, argument slot count, semicolon,
dispositions, metadata, and JSON order are exact. Source-to-slot value
correspondence is a prompt obligation, not a structural validator guarantee.
Pretty-printer whitespace is not contractual.

Canonical Rust format fields use this order: alignment, sign, alternate form,
zero padding, width, precision, and conversion trait. C `-` becomes `<` only
when width exists and suppresses `0`; `+` suppresses the wrapper's requested
space sign; a space flag is not encoded in the Rust field; integer precision
is retained and makes the wrapper ignore zero padding; and `#` is retained only
where it affects output: for `o/x/X` and `g/G`; for `f/F/e/E` only at precision
zero; and for `a/A` only at absent or zero precision. A no-op `#` at positive
fixed/scientific/hex precision is omitted canonically. Missing precision
becomes `.6` for `f/F/e/E/g/G`, stays absent for `a/A/s`, and a bare dot means
precision zero. Trait suffixes are `o/x/X` for integers, `e/E` for scientific,
`x/X` for hexadecimal float, and absent otherwise.

`P(input)` means compile `input`, run `preparer::prepare`, require the inner
result to be `Ok`, parse the returned source, compare the complete relevant AST
structurally, and compile it again. The returned preparation value contains the
source and `requires_proctor_libc: bool`. `Same(input, false)` means the
relevant AST is unchanged and the dependency flag is false. Every preparation
error is crate-atomic and requests no dependency.

Ctype recognition is intentionally syntactic and narrow. It mirrors the
existing libc-pass patterns: one-segment direct names and exact C2Rust
deref/offset/cast table shapes. It does not add DefId provenance, linked-symbol,
ABI/signature, or constant-evaluation machinery. A same-shaped occurrence is
rewritten; a shape outside this boundary remains ordinary local-transformation
work.

### 3.1 Fixed ctype and dependency-error oracles

Direct ctype replacement changes only the callee path and preserves the
original argument expression unchanged. Table idioms match only when the
`.offset` argument has the exact outer shape `e as isize`; replacement strips
only that terminal cast and passes `e` unchanged. It adds no cast and performs
no type analysis. A table form whose `.offset` argument has no terminal
`as isize` cast is a nonmatch.

The generic utils dependency seam and preparation publication API return
ordinary `Result` values. A dependency failure uses no `panic!`, `unwrap`, or
`expect`; publication stops before source write with this stable diagnostic
prefix after substituting the displayed manifest path and underlying cause:

```text
failed to ensure proctor-libc dependency in <manifest path>: <cause>
```

## 4. Updated existing regression expectations

### A9-UPD-01 `printf_rejection_rows_move_to_acceptance`

Update the existing Amendment 7 tests whose inputs are now accepted. The old
rejection expectations for space sign, alternate integer form, integer
precision, static string width/precision, `e/E/g/G/a/A`, alternate float at
precision zero, and `L` floating length are removed from executable tests.
Their replacement acceptance rows are A9-FMT-08--24. Historical plan files are
not edited.

All old rejections for incomplete/unknown formats, dynamic or positional
arguments, apostrophe grouping, `c/p/n`, wide strings, invalid lengths,
numeric overflow, non-UTF-8 text, malformed percent conversions, nonliteral
formats, and ineligible function identity remain executable.

### A9-UPD-02 `dependency_expectations_are_0_3_0`

Update every local-transformation fixture that expects `proctor-libc =
"0.1.0"` to expect `"0.3.0"`. Existing `bytemuck = "1.25.2"` and
`xj_scanf = "0.2.6"` expectations and unrelated dependency fields are
byte-for-byte unchanged.

### A9-UPD-03 `function_record_has_one_new_required_field`

Update every function-record JSON fixture and helper to contain:

```json
"printf_format_specifiers": []
```

unless the fixture explicitly tests a consuming lowered `printf`. Static,
constant, type-alias, enum, struct, and union record shapes do not gain the
field. No public stage schema, artifact shape, command, config key, or metric
changes.

### A9-UPD-04 `prompt_and_document_versions_stay_one`

Existing exact assertions continue to require prompt id
`local_transformation`, prompt version `1`, validation/replacement
`schema_version: 1`, and observation/rule `schema_version: 1`. The prompt text
and its exact variable list change in place because the earlier prompt version
has not been merged to the main branch.

### A9-UPD-05 `strto_reference_count_increases_only`

The libc guidance catalog still has 40 activation entries and the same exact
foreign-name set. Its reference-signature total changes from 44 to 49 because
five existing entries gain one mutable reference each. No `_mut` name becomes
a foreign activator.

### A9-UPD-06 `prepare_now_owns_ctype`

Replace the executable Amendment 8 regression that asserted libc was wholly
out of scope. `prepare` now owns only the ctype rewrites and conditional
proctor-libc dependency described here. Match-arm normalization, local-static
lifting, error atomicity, the checked-in local pass list, and every unrelated
pass/config expectation remain unchanged.

## 5. proctor-libc dependency normalization

Implement A9-DEP-01--08 with the existing fake stage and `tmp_path`. In every
success, the read-only input manifest is byte-for-byte unchanged and the
published copy has the expected dependency.

| Case | Input `dependencies.proctor-libc` | Expected published value |
| --- | --- | --- |
| A9-DEP-01 | absent, including absent `[dependencies]` | `"0.3.0"` |
| A9-DEP-02 | `"0.0.1"`, `"0.1.0"`, `"0.2.9"`, or `"0.3.0-alpha.1"` | `"0.3.0"` |
| A9-DEP-03 | `{ version = "0.2", features = ["x"], default-features = false }` | same table and fields, version `"0.3.0"` |
| A9-DEP-04 | `{ default-features = false }` | same table plus `version = "0.3.0"` |
| A9-DEP-05 | `"0.3"`, `"^0.3.0"`, `">=0.3.0"`, `"0.4"`, or `"1"` | preserved exactly |
| A9-DEP-06 | canonical `path`, `git`, `workspace`, or alternate-registry table | preserved with no injected version |
| A9-DEP-07 | alias `{ package = "proctor-libc", version = "0.1" }` under `pl` | fatal canonical-name error before preparation |
| A9-DEP-08 | canonical key aliases `package = "another-crate"` | fatal package-collision error before preparation |

A9-DEP-09 loads a manifest containing serde features plus all three required
dependencies. Expected only a below-minimum proctor-libc requirement changes;
ordering normalization may follow the existing TOML writer, but parsed values
for serde, bytemuck, and xj_scanf are identical.

A9-DEP-10 checks `deps_crate/Cargo.toml` requires exactly crates.io
`proctor-libc = "0.3.0"`; `Cargo.lock` resolves a compatible 0.3.x package and
its correct dependency closure; `cargo metadata --locked --offline` and
`cargo check --locked --offline` succeed. There is no remaining 0.1.x
proctor-libc package in the locked graph.

A9-DEP-11 asserts this exact event order: `build_tools` runs first; the input
project is then copied and its dependency normalized; only afterward may
project preparation, skeleton generation, the initial Cargo build, or the LLM
run. The fake tool inspects the copied manifest at its first project-preparation
call and sees `proctor-libc = "0.3.0"`. A normalization failure has exact fake
events `["build_tools"]` and performs no project preparation, skeleton,
project build, or LLM call.

A9-DEP-12 supplies each malformed manifest independently:

| Malformed input | Exact manifest fragment | Exact local-stage error assertion |
| --- | --- | --- |
| invalid TOML | `[dependencies` | `output.error` starts with `working Cargo.toml is invalid:` |
| non-table dependencies | `dependencies = 7` | `output.error == "Cargo [dependencies] must be a table"` |
| non-string canonical dependency | `[dependencies]` followed by `proctor-libc = 7` | `output.error == "Cargo proctor-libc dependency must be a string or table"` |
| non-string table version | `[dependencies]` followed by `proctor-libc = { version = 7 }` | `output.error == "Cargo proctor-libc dependency version must be a string"` |

Each local-stage case fails with exact fake events `["build_tools"]`, before
project preparation, skeleton generation, project build, or LLM use, and
preserves the input manifest. A9-PREP-02A reuses these four input fragments for
the pure utils normalization seam. When preparation publication supplies
manifest path `/work/project/Cargo.toml`, every failure uses this exact template,
substituting the corresponding underlying cause, and occurs before source write:

```text
failed to ensure proctor-libc dependency in /work/project/Cargo.toml: <cause>
```

## 6. Pure `printf` conversion acceptance

Add table-driven cases in `printf.rs`. Each expected specifier is the exact
source substring, including repeated flags, width, precision, length, and
case.

| Case | Input | Exact expected result |
| --- | --- | --- |
| A9-FMT-01 | `C("plain {text} %%")` | `Ok("plain {{text}} %", [])` |
| A9-FMT-02 | `C("%d %i %u")` | `Ok("{} {} {}", ["%d","%i","%u"])` |
| A9-FMT-03 | `C("%o %x %X")` | `Ok("{:o} {:x} {:X}", ["%o","%x","%X"])` |
| A9-FMT-04 | `C("%hhd %hd %ld %lld %jd %zd %td")` | `Ok("{} {} {} {} {} {} {}", ["%hhd","%hd","%ld","%lld","%jd","%zd","%td"])` |
| A9-FMT-05 | `C("%hhu %hu %lu %llu %ju %zu %tu")` | `Ok("{} {} {} {} {} {} {}", ["%hhu","%hu","%lu","%llu","%ju","%zu","%tu"])` |
| A9-FMT-06 | `C("%08d %-08d %+8d")` | `Ok("{:08} {:<8} {:+8}", ["%08d","%-08d","%+8d"])` |
| A9-FMT-07 | `C("%.5d %08.5d %-08.5u %.0u")` | `Ok("{:.5} {:08.5} {:<8.5} {:.0}", ["%.5d","%08.5d","%-08.5u","%.0u"])` |
| A9-FMT-08 | `C("% d %+ d")` | `Ok("{} {:+}", ["% d","%+ d"])`; space is wrapper metadata, `+` wins |
| A9-FMT-09 | `C("%#o %#x %#X")` | `Ok("{:#o} {:#x} {:#X}", ["%#o","%#x","%#X"])` |
| A9-FMT-10 | `C("%#.0o %#08.4x")` | `Ok("{:#.0o} {:#08.4x}", ["%#.0o","%#08.4x"])` |
| A9-FMT-11 | `C("%f %F %lf %lF")` | `Ok("{:.6} {:.6} {:.6} {:.6}", ["%f","%F","%lf","%lF"])` |
| A9-FMT-12 | `C("%Lf %LF")` | `Ok("{:.6} {:.6}", ["%Lf","%LF"])` |
| A9-FMT-13 | `C("%#.0f %#08.0F %-#08.0f")` | `Ok("{:#.0} {:#08.0} {:<#8.0}", ["%#.0f","%#08.0F","%-#08.0f"])` |
| A9-FMT-14 | `C("%e %E %.2e %12.2E")` | `Ok("{:.6e} {:.6E} {:.2e} {:12.2E}", ["%e","%E","%.2e","%12.2E"])` |
| A9-FMT-15 | `C("%Le %LE %+012.2e %-012.2E")` | `Ok("{:.6e} {:.6E} {:+012.2e} {:<12.2E}", ["%Le","%LE","%+012.2e","%-012.2E"])` |
| A9-FMT-16 | `C("%g %G %.0g %#.6G")` | `Ok("{:.6} {:.6} {:.0} {:#.6}", ["%g","%G","%.0g","%#.6G"])` |
| A9-FMT-17 | `C("%Lg %LG % 010.4g")` | `Ok("{:.6} {:.6} {:010.4}", ["%Lg","%LG","% 010.4g"])` |
| A9-FMT-18 | `C("%a %A %.3a %#.0A")` | `Ok("{:x} {:X} {:.3x} {:#.0X}", ["%a","%A","%.3a","%#.0A"])` |
| A9-FMT-19 | `C("%La %LA %+012.2a %-012.2A")` | `Ok("{:x} {:X} {:+012.2x} {:<12.2X}", ["%La","%LA","%+012.2a","%-012.2A"])` |
| A9-FMT-20 | `C("%s %10s %-10s %.3s %10.3s")` | `Ok("{} {:10} {:<10} {:.3} {:10.3}", ["%s","%10s","%-10s","%.3s","%10.3s"])` |
| A9-FMT-21 | `C("%.d %.u %.s %.f %.e %.g %.a")` | `Ok("{:.0} {:.0} {:.0} {:.0} {:.0e} {:.0} {:.0x}", ["%.d","%.u","%.s","%.f","%.e","%.g","%.a"])` |
| A9-FMT-22 | `C("%--++ 0008.4f")` | `Ok("{:<+8.4}", ["%--++ 0008.4f"])`; repeated flags retained only in metadata |
| A9-FMT-23 | `C("n=% d x=%#08.4x s=%.3s e=%E %%")` | `Ok("n={} x={:#08.4x} s={:.3} e={:.6E} %", ["% d","%#08.4x","%.3s","%E"])` |
| A9-FMT-24 | `C("é=%La")` | `Ok("é={:x}", ["%La"])` |

If the approved canonical field builder orders or normalizes flags differently
from a row above, that is a substantive discrepancy: update this document and
the implementation plan together before coding. Tests must not accept several
equivalent Rust format spellings.

### A9-FMT-25 `finite_boundary_matrix_is_exact`

Add these literal cases. They cover each remaining flag interaction and the
largest accepted numeric field without generating an unbounded or
implementation-derived matrix:

| Input | Exact expected result |
| --- | --- |
| `C("%-+017.4lld %0-+17.4td")` | `Ok("{:<+17.4} {:<+17.4}", ["%-+017.4lld","%0-+17.4td"])` |
| `C("%017.4ju %#017.4zo %0-#17.4X")` | `Ok("{:017.4} {:#017.4o} {:<#17.4X}", ["%017.4ju","%#017.4zo","%0-#17.4X"])` |
| `C("%+#017.2Le %#Le")` | `Ok("{:+017.2e} {:.6e}", ["%+#017.2Le","%#Le"])`; no-op `#` is omitted |
| `C("%#La %#17.2La")` | `Ok("{:#x} {:17.2x}", ["%#La","%#17.2La"])`; positive-precision `#` is omitted |
| `C("%-017.3LG %+#017.3LG")` | `Ok("{:<17.3} {:+#017.3}", ["%-017.3LG","%+#017.3LG"])` |
| `C("%-17.3s")` | `Ok("{:<17.3}", ["%-17.3s"])` |
| `C("%2147483647d %.2147483647s")` | `Ok("{:2147483647} {:.2147483647}", ["%2147483647d","%.2147483647s"])` |

The expected strings are literal test data; tests must not call production
helpers to construct them.

## 7. Pure `printf` rejection and totality

Every row returns `No` for the complete input and never panics.

| Case | Inputs | Reason |
| --- | --- | --- |
| A9-FMT-N01 | `"%"`, `"abc%"` | incomplete conversion |
| A9-FMT-N02 | `"%q"`, `"%v"`, `"%D"` | unknown conversion |
| A9-FMT-N03 | `"%2$d"`, `"%1$s"` | positional value |
| A9-FMT-N04 | `"%*d"`, `"%2$*3$d"`, `"%*.*f"` | dynamic/positional width or precision |
| A9-FMT-N05 | `"%'d"`, `"%'f"`, `"%'s"` | locale grouping |
| A9-FMT-N06 | `"%c"`, `"%lc"`, `"%C"` | excluded character conversion |
| A9-FMT-N07 | `"%p"`, `"%n"`, `"%hn"` | excluded pointer/effect conversion |
| A9-FMT-N08 | `"%ls"`, `"%hs"`, `"%Ls"` | non-byte-string length |
| A9-FMT-N09 | `"%hf"`, `"%llf"`, `"%je"`, `"%zg"`, `"%ta"` | invalid floating length |
| A9-FMT-N10 | `"%+u"`, `"% u"`, `"%+x"` | irrelevant unsigned sign flag excluded |
| A9-FMT-N11 | `"%#d"`, `"%#i"`, `"%#u"` | irrelevant alternate flag excluded |
| A9-FMT-N12 | `"%+s"`, `"% s"`, `"%#s"`, `"%0s"` | unsupported string flags |
| A9-FMT-N13 | `"%5%"`, `"%-%%"`, `"%.0%"` | decorated percent conversion |
| A9-FMT-N14 | `"%2147483648d"`, `"%.2147483648f"`, `"%999999999999999999999999s"` | exceeds `i32::MAX` or checked decimal arithmetic |
| A9-FMT-N15 | byte arrays `[0xff]`, `[b'%', 0xff, b'd']`, `[b'a', 0xc3, 0x28]` | invalid UTF-8 |
| A9-FMT-N16 | `"ok=%d bad=%p"`, `"%10.3s %n"`, `"%e %*d"` | statement-atomic mixed rejection |

A9-FMT-N17 runs each literal byte input under `catch_unwind` and asserts the
exact result:

| Bytes | Exact result |
| --- | --- |
| `b"%"`, `b"%-"`, `b"%."`, `b"%L"`, `b"%hh"`, `b"%999999999999d"` | `No` |
| `[0xff]`, `[0xc3, 0x28]` | `No` |
| `[0x00]` | `Ok("\0", [])` |
| `b"a"` | `Ok("a", [])` |
| `b"{}}{"` | `Ok("{{}}}}{{", [])` |
| `b"%%%d"` | `Ok("%{}", ["%d"])` |
| 10,000 copies of `b'9'` | `Ok(10,000 copies of "9", [])` |

All calls return normally. The final expected string is constructed in the
test as `"9".repeat(10_000)`; it is not a format-parser oracle. Retain the
separate overflow assertions from A9-FMT-N14.

## 8. `printf` eligibility, templates, and function metadata

### A9-ELIG-01 `all_new_formats_keep_the_exact_printf_boundary`

Use one semicolon statement whose literal is:

```text
%d %i %u %o %x %X %f %F %e %E %g %G %a %A %s\n\0
```

and pass, in order, two `i32`, four `u32`, eight `f64`, and one `*const i8`
expressions. Expected one `transform` node and this exact template format:

```text
{} {} {} {:o} {:x} {:X} {:.6} {:.6} {:.6e} {:.6E} {:.6} {:.6} {:x} {:X} {}\n
```

with `argument_count: 15`, fifteen `todo!()` holes, and the exact fifteen
source specifiers in source order. Separately,
`printf(b"hello %%\n\0" as *const u8 as *const i8);` is `mechanical` and
becomes exactly `::std::print!("hello %\n");` with argument count zero and an
empty function-level specifier array.

### A9-ELIG-02 `identity_and_prototype_near_misses_fall_back`

Run the following finite mutations separately with the exact call payload
`("%e", f)` where `f: f64`:

| Mutation | Exact callee/declaration shape | Expected eligibility result |
| --- | --- | --- |
| local item | `fn printf(format: *const i8, value: f64) -> i32`; `printf(format, value)` | `None` |
| inherent method | `printer.printf(format, value)` | `None` |
| closure binding | `let printf = \|format, value\| 0; printf(format, value)` | `None` |
| macro | `printf!(format, value)` | no call candidate |
| wrong linked symbol | `#[link_name = "not_printf"] fn printf(format: *const i8, ...) -> i32` | `None` |
| different foreign symbol | `fn fprintf(stream: *mut FILE, format: *const i8, ...) -> i32` | `None` |
| dependency item | `dependency::printf(format, value)` | `None` |
| Rust ABI | `fn printf(format: *const i8, value: f64) -> i32` | `None` |
| system ABI | `extern "system" { fn printf(format: *const i8, ...) -> i32; }` | `None` |
| unwind ABI | `extern "C-unwind" { fn printf(format: *const i8, ...) -> i32; }` | `None` |
| fixed arguments | `extern "C" { fn printf(format: *const i8, value: f64) -> i32; }` | `None` |
| wrong first parameter | `extern "C" { fn printf(format: usize, ...) -> i32; }` | `None` |
| wrong return | `extern "C" { fn printf(format: *const i8, ...); }` | `None` |
| added fixed parameter | `extern "C" { fn printf(fd: i32, format: *const i8, ...) -> i32; }` | `None` |

For the test-only missing-map case, delete the callee expression's AST-to-HIR
entry before calling `eligible_printf_statement`; the exact result is `None`,
with no template or metadata. A separate positive fixture declares
`#[link_name = "printf"] fn c_printf(format: *const i8, ...) -> i32` and calls
`c_printf`; it remains eligible, while `c_printf` stays in ordinary
foreign-name context.

### A9-ELIG-03 `statement_and_literal_near_misses_fall_back`

Use these independent, compiling inputs with the otherwise eligible foreign
declaration. Every row produces no trusted printf template and contributes no
`printf_format_specifiers`; the statement remains an ordinary transform.

| Boundary | Exact expression |
| --- | --- |
| tail expression | `unsafe fn f(x: f64) -> i32 { printf(b"%e\0".as_ptr() as *const i8, x) }` |
| initializer | `let result = printf(b"%e\0".as_ptr() as *const i8, x);` |
| return value | `return printf(b"%e\0".as_ptr() as *const i8, x);` |
| subexpression | `let result = 1 + printf(b"%e\0".as_ptr() as *const i8, x);` |
| condition | `if printf(b"%e\0".as_ptr() as *const i8, x) != 0 { consume(); }` |
| macro input | `consume!(printf(b"%e\0".as_ptr() as *const i8, x));` |
| runtime pointer | `let format = b"%e\0".as_ptr() as *const i8; printf(format, x);` |
| static value | `printf(FORMAT.as_ptr() as *const i8, x);` where `static FORMAT: &[u8] = b"%e\0"` |
| unsupported cast chain | `printf(b"%e\0".as_ptr() as usize as *const i8, x);` |
| non-UTF-8 literal | `printf(b"%\xffe\0".as_ptr() as *const i8, x);` |
| interior NUL | `printf(b"%e\0ignored\0".as_ptr() as *const i8, x);` |
| missing value | `printf(b"%e\0".as_ptr() as *const i8);` |
| extra value | `printf(b"%e\0".as_ptr() as *const i8, x, x);` |

### A9-SKEL-01 `expanded_format_builds_one_trusted_template`

Input:

```rust
pub unsafe fn show(d: i32, x: u32, f: f64, s: *const i8) {
    printf(
        b"d=% d x=%#08.4x f=%E s=%10.3s\n\0" as *const u8 as *const i8,
        d, x, f, s,
    );
}
```

Expected baseline skeleton contains exactly:

```rust
#[proctor(0)]
::std::print!("d={} x={:#08.4x} f={:.6E} s={:10.3}\n",
              todo!(), todo!(), todo!(), todo!());
```

The template metadata contains that semantic string and argument count `4`.
The function-level field is sorted and deduplicated:

```json
"printf_format_specifiers": ["% d", "%#08.4x", "%10.3s", "%E"]
```

Source-order conversions remain in template and observation metadata; only
the function summary is sorted/deduplicated.

### A9-SKEL-02 `function_metadata_is_complete_and_deterministic`

Use one fixed function with these statements, in this order: `printf("%s",
s)`, `printf("%d", d)`, another `printf("%s", s)`,
`printf("literal")`, `printf("%%")`, unsupported `printf("%p", p)`, and
eligible `%d` inside a block-bodied `if`. Expected exact field
`["%d", "%s"]`; literal text, percent escapes, and the unsupported call add
nothing. Two runs on this unchanged function serialize byte-identically.
Reordering source statements is a different skeleton and need not preserve
whole-record JSON bytes; only the sorted function-level field remains
`["%d", "%s"]`.

### A9-SKEL-03 `metadata_is_function_level_across_views`

Supply a rule that applies to one of two consuming calls. Baseline has two
`transform` nodes; applied has one `rule_applied` and one `transform`.
Expected one shared function-level union covering both exact specifiers. The
field is not duplicated inside `SkeletonView` or `PrintfTemplateMetadata` and
does not vary by view.

### A9-SKEL-04 `empty_metadata_cases_are_explicit`

Functions with no `printf`, unsupported `printf`, or only zero-consuming
mechanical calls emit `"printf_format_specifiers": []`. Nonfunction records
retain their exact old keys. Missing the new function key is not treated as an
old-compatible default.

### A9-SKEL-05 `view_and_report_invariants_do_not_change`

Use a fixed function whose first top-level statement is `%#x`, whose second
top-level statement is an `if` containing `%10.3s`, and whose third top-level
statement is `printf("done")`; provide a rule that covers only `%#x`. Annotation
labels are 0 for `%#x`, 1 for the preserved `if` shell, 2 for the nested
`%10.3s`, and 3 for `printf("done")`. Expected baseline top-level dispositions
are `transform, preserve_shell, mechanical`; applied top-level dispositions are
`rule_applied, preserve_shell, mechanical`; the shell's child remains
`transform` in both views. Report metadata labels are `[0,2,3]` in the baseline
view and `[2,3]` in the applied view. Metadata for labels 2 and 3 is
byte-identical across views, including label 2's existing pointer-variable
fields. `preserve_shell` label 1 and `rule_applied` label 0 are intentionally not
report pairs. The function summary is `["%#x", "%10.3s"]`.

## 9. Validation and replacement defenses

### A9-VAL-01 `new_format_fields_are_trusted`

Use each literal skeleton/transformation pair below with trusted printf
metadata containing its shown format and `argument_count: 1`:

```rust
unsafe fn precision_zero(value: i32) {
    #[proctor(0)] ::std::print!("{:.0}", todo!());
}
// transformation
unsafe fn precision_zero(value: i32) {
    #[proctor(0)] ::std::print!("{:.0}", ::proctor_libc::printf::signed(value));
}

unsafe fn lower_hex(value: u32) {
    #[proctor(0)] ::std::print!("{:#08.4x}", todo!());
}
// transformation
unsafe fn lower_hex(value: u32) {
    #[proctor(0)] ::std::print!("{:#08.4x}", ::proctor_libc::printf::unsigned(value));
}

unsafe fn lower_exp(value: f64) {
    #[proctor(0)] ::std::print!("{:12.2e}", todo!());
}
// transformation
unsafe fn lower_exp(value: f64) {
    #[proctor(0)] ::std::print!("{:12.2e}", ::proctor_libc::printf::scientific(value));
}

unsafe fn upper_exp(value: f64) {
    #[proctor(0)] ::std::print!("{:.6E}", todo!());
}
// transformation
unsafe fn upper_exp(value: f64) {
    #[proctor(0)] ::std::print!("{:.6E}", ::proctor_libc::printf::scientific(value));
}

unsafe fn alternate_general(value: f64) {
    #[proctor(0)] ::std::print!("{:#.6}", todo!());
}
// transformation
unsafe fn alternate_general(value: f64) {
    #[proctor(0)] ::std::print!("{:#.6}", ::proctor_libc::printf::general(value));
}

unsafe fn lower_hex_float(value: f64) {
    #[proctor(0)] ::std::print!("{:.3x}", todo!());
}
// transformation
unsafe fn lower_hex_float(value: f64) {
    #[proctor(0)] ::std::print!("{:.3x}", ::proctor_libc::printf::hex_float(value));
}

unsafe fn upper_hex_float(value: f64) {
    #[proctor(0)] ::std::print!("{:#.0X}", todo!());
}
// transformation
unsafe fn upper_hex_float(value: f64) {
    #[proctor(0)] ::std::print!("{:#.0X}", ::proctor_libc::printf::hex_float(value));
}

unsafe fn bounded_bytes(value: &[i8]) {
    #[proctor(0)] ::std::print!("{:10.3}", todo!());
}
// transformation
unsafe fn bounded_bytes(value: &[i8]) {
    #[proctor(0)] ::std::print!("{:10.3}", ::proctor_libc::printf::byte_string(value));
}

unsafe fn literal_mix(value: i32) {
    #[proctor(0)] ::std::print!("{{}} % {:.0}", todo!());
}
// transformation
unsafe fn literal_mix(value: i32) {
    #[proctor(0)] ::std::print!("{{}} % {:.0}", ::proctor_libc::printf::signed(value));
}
```

Each validation response is exactly `Valid`; replacement preserves the exact
semantic literal and argument slot count. Source-to-slot value correspondence
remains a prompt obligation, not trusted validator metadata.

### A9-VAL-02 `format_and_count_mutations_still_fail`

Against the named A9-VAL-01 base skeleton, validate each complete literal
transformation independently. The response is `Invalid` with exactly the
listed printf validator code:

| Base | Mutated complete transformation | Exact code |
| --- | --- | --- |
| `lower_hex` | `unsafe fn lower_hex(value: u32) { #[proctor(0)] ::std::print!("{:#08.4X}", ::proctor_libc::printf::unsigned(value)); }` | `printf_format_literal` |
| `lower_exp` | `unsafe fn lower_exp(value: f64) { #[proctor(0)] ::std::print!("{:12.3e}", ::proctor_libc::printf::scientific(value)); }` | `printf_format_literal` |
| `lower_exp` | `unsafe fn lower_exp(value: f64) { #[proctor(0)] ::std::print!("{:13.2e}", ::proctor_libc::printf::scientific(value)); }` | `printf_format_literal` |
| `alternate_general` | `unsafe fn alternate_general(value: f64) { #[proctor(0)] ::std::print!("{:.6}", ::proctor_libc::printf::general(value)); }` | `printf_format_literal` |
| `literal_mix` | `unsafe fn literal_mix(value: i32) { #[proctor(0)] ::std::print!("changed {:.0}", ::proctor_libc::printf::signed(value)); }` | `printf_format_literal` |
| `literal_mix` | `unsafe fn literal_mix(value: i32) { #[proctor(0)] ::std::print!("{{x}} % {:.0}", ::proctor_libc::printf::signed(value)); }` | `printf_format_literal` |
| `literal_mix` | `unsafe fn literal_mix(value: i32) { #[proctor(0)] ::std::print!("{{}} %% {:.0}", ::proctor_libc::printf::signed(value)); }` | `printf_format_literal` |
| `lower_hex` | `unsafe fn lower_hex(value: u32) { #[proctor(0)] std::print!("{:#08.4x}", ::proctor_libc::printf::unsigned(value)); }` | `printf_macro_path` |
| `lower_hex` | `unsafe fn lower_hex(value: u32) { #[proctor(0)] ::std::print!["{:#08.4x}", ::proctor_libc::printf::unsigned(value)]; }` | `printf_macro_delimiter` |
| `lower_hex` | `unsafe fn lower_hex(value: u32) { #[proctor(0)] ::std::print!("{0:#08.4x}", ::proctor_libc::printf::unsigned(value)); }` | `printf_format_references` |
| `lower_hex` | `unsafe fn lower_hex(value: u32) { #[proctor(0)] ::std::print!(::std::string::String::from("{:#08.4x}"), ::proctor_libc::printf::unsigned(value)); }` | `printf_format_literal` |
| `lower_hex` | `unsafe fn lower_hex(value: u32) { #[proctor(0)] ::std::print!(concat!("{:#08", ".4x}"), ::proctor_libc::printf::unsigned(value)); }` | `printf_format_literal` |
| `lower_hex` | `unsafe fn lower_hex(value: u32) { #[proctor(0)] ::std::print!("{:#08.4x}"); }` | `printf_argument_count` |
| `lower_hex` | `unsafe fn lower_hex(value: u32) { #[proctor(0)] ::std::print!("{:#08.4x}", ::proctor_libc::printf::unsigned(value), value); }` | `printf_argument_count` |

Swapping two argument expressions while retaining the trusted format and slot
count is deliberately absent from this rejection table.

### A9-VAL-03 `argument_expressions_remain_prompt_enforced`

Submit each pair below against the same trusted format and slot count:

```rust
::std::print!("{}", ::proctor_libc::printf::signed(x));
::std::print!("{}", x);

::std::print!("{}", ::proctor_libc::printf::signed(x).space_sign()); // `% d`
::std::print!("{}", x);                                             // wrong semantics

::std::print!("{:#x}", ::proctor_libc::printf::unsigned(x)); // `%#x`
::std::print!("{:#x}", x);                                  // wrong for C alternate form

::std::print!("{:.6e}", ::proctor_libc::printf::scientific(x)); // `%e`
::std::print!("{:.6e}", x);                                     // wrong C exponent spelling

::std::print!("{:.3}", ::proctor_libc::printf::byte_string(bytes)); // `%.3s`
::std::print!("{:.3}", ::std::str::from_utf8(
    ::bytemuck::cast_slice(bytes)).unwrap());                         // counts chars, not bytes
```

Every line is structurally valid when the rest of its skeleton matches. The
signed and space-sign pairs use `x: i32`, the unsigned pair uses `x: u32`, the
scientific pair uses `x: f64`, and the string pair uses `bytes: &[i8]`, so both
the wrapper and wrong-semantic forms compile. Also
submit a two-slot `%d %d` target with
`signed(second), signed(first)` instead of `signed(first), signed(second)`;
validation accepts the swap. The prompt must explicitly forbid swapping slots
and require the listed wrapper forms. Assert it contains this exact advisory
sentence:

```text
preserve the source order of consuming values: fill the existing argument slots in order and do not swap slots
```

This is intentional: the transactional Cargo build checks Rust types, not C
formatting semantics.

### A9-REP-01 `restoration_cannot_be_weakened_by_broader_formats`

Use this fixed current function:

```rust
pub unsafe fn render(flag: bool, x: f64) -> i32 {
    let keep: i32 = 7;
    if flag { printf(b"%e\0" as *const u8 as *const i8, x); }
    keep
}
```

The skeleton preserves `let keep: i32 = 7;` and the `if` shell, while the
printf child is a transform template `::std::print!("{:.6e}", todo!());`.
An LLM candidate that changes `keep` to `8` is `invalid`. For the accepted
candidate using `scientific(x)`, replacement emits the original preserved
declaration and control shell plus exactly
`::std::print!("{:.6e}", ::proctor_libc::printf::scientific(x));`.
It emits no `printf_template` or `printf_format_specifiers` data into Rust.
Existing signature-wrapper, export, call-redirection, temporary-scope, and
mechanical regression tests run unchanged under A9-REG-01 rather than being
duplicated with unspecified fixtures here.

### A9-REP-02 `replacer_rechecks_the_trusted_format_without_validator`

Bypass `validate` and send `replace_items` the A9-VAL-01 `lower_hex` skeleton,
its trusted `rust_format: "{:#08.4x}"` and `argument_count: 1`, and this exact
transformation:

```rust
unsafe fn lower_hex(value: u32) {
    #[proctor(0)]
    ::std::print!("{:#08.4X}", ::proctor_libc::printf::unsigned(value));
}
```

The replacer returns `ReplacementErrorKind::InvalidTransformation` with exact
message `printf_format_literal: print format literal differs from the expected
converted format`; it writes no replacement and emits no statement pair.

## 10. Observations, documents, synthesis, and application

### A9-OBS-01 `every_wrapper_family_extracts`

Create these literal one-argument functions and accepted targets. Each yields
exactly one printf observation whose `format_specifier`, source expression,
and target expression equal the row:

| Specifier | Source expression | Exact target expression |
| --- | --- | --- |
| `%hhd` | `small as i32` | `::proctor_libc::printf::signed(small as i8)` |
| `%ld` | `long` | `::proctor_libc::printf::signed(long)` |
| `%u` | `u` | `::proctor_libc::printf::unsigned(u)` |
| `%#08.4x` | `x` | `::proctor_libc::printf::unsigned(x)` |
| `%f` | `f` | `::proctor_libc::printf::fixed(f)` |
| `%LF` | `wide` | `::proctor_libc::printf::fixed_upper(wide)` |
| `%E` | `e` | `::proctor_libc::printf::scientific(e)` |
| `%Lg` | `wide` | `::proctor_libc::printf::general(wide)` |
| `%G` | `g` | `::proctor_libc::printf::general_upper(g)` |
| `%La` | `wide` | `::proctor_libc::printf::hex_float(wide)` |
| `%10.3s` | `s` | `::proctor_libc::printf::byte_string(s)` |

Add exact space-flag rows `% d`, `% f`, `% E`, `% g`, `% G`, `% a`, and
`% A`; their source expressions are respectively `d`, `f`, `e`, `g`, `g`,
`a`, and `a`, and each target is the corresponding mapping above followed by
`.space_sign()`. The `%hhd` row proves that the target observation contains the
post-length-conversion `i8`, not only the promoted source `i32`. The `L` rows
use a target binding of exact type `f128::f128`.

### A9-OBS-02 `mixed_statement_is_ordered_and_atomic`

Use A9-SKEL-01. Expected four observations in trusted macro-slot/source order with
specifiers `% d`, `%#08.4x`, `%E`, `%10.3s`. If one argument cannot be mapped,
typed, anchored, or matched to the trusted template, the complete statement
contributes no printf observation. Other accepted statements remain.

### A9-OBS-03 `semantic_literal_is_not_an_observation`

Formats with literal braces, `%%`, widths, and precision generate only value
observations. The trusted literal and wrapper function paths are rigid target
structure, not expression variables or ordinary foreign-call seeds.

### A9-WIRE-01 `version_one_documents_accept_the_expanded_domain`

Round-trip version-1 observation and rule documents containing this exact
specifier list:

```text
%d, %hhi, %u, %#08.4o, %x, %X, %f, %LF, %e, %E,
%g, %G, %a, %A, %10.3s, % d
```

The parsed value and canonical reserialization preserve every string and both
required ordinary/printf arrays. Mutate one document at a time: `%q` returns a
`DocumentError` containing `.format_specifier is unsupported`; `%d%d` returns
one containing `.format_specifier must be exactly one consuming conversion`;
`schema_version: 2` contains `unsupported observation schema_version 2` or
`unsupported rule schema_version 2`; missing required arrays, an extra closed
field, or target binding index 1 with only index 0 bound each returns
`DocumentError` and no parsed document.

### A9-SYN-01 `exact_specifier_remains_the_grouping_key`

Two compatible observations with exact `%#08.4x` synthesize one deterministic
printf rule. `%#08.4x` never pairs with `%#8.4x`, `%#08.5x`, `%#08.4X`, `%x`,
or `%08.4x`. Repeat for `% d` versus `%d`, `%E` versus `%e`, `%Lg` versus
`%g`, and `%10.3s` versus `%.3s`.

### A9-SYN-02 `wrapper_bearing_targets_preserve_identity_carriers`

For each scalar pair below, create anchorless observations differing only in
the local binding names `left` and `right`, then synthesize. For `%10.3s`, use
pointer-linked observations whose local bindings are the carriers of matching
raw-pointer-to-shared-slice anchors.

| Specifier | Target roots in the two observations | Exact rigid rule target |
| --- | --- | --- |
| `%d` | `signed(left)`, `signed(right)` | `signed(<binding0>)` |
| `%#08.4x` | `unsigned(left)`, `unsigned(right)` | `unsigned(<binding0>)` |
| `%f` | `fixed(left)`, `fixed(right)` | `fixed(<binding0>)` |
| `%F` | `fixed_upper(left)`, `fixed_upper(right)` | `fixed_upper(<binding0>)` |
| `%E` | `scientific(left)`, `scientific(right)` | `scientific(<binding0>)` |
| `%g` | `general(left)`, `general(right)` | `general(<binding0>)` |
| `%G` | `general_upper(left)`, `general_upper(right)` | `general_upper(<binding0>)` |
| `%A` | `hex_float(left)`, `hex_float(right)` | `hex_float(<binding0>)` |
| `%10.3s` | `byte_string(left)`, `byte_string(right)` | `byte_string(<anchor0>)` |
| `% d` | `signed(left).space_sign()`, `signed(right).space_sign()` | `signed(<binding0>).space_sign()` |

All paths are fully qualified in actual expressions. Expected one rule per
row, with the listed exact specifier and rigid call/method structure. A single
observation yields zero rules. Pairing `signed(left)` with `unsigned(right)` at
the same fabricated specifier yields zero rules. `<binding0>` above denotes the
exact JSON expression
`{"kind":"path","value":{"kind":"variable","sort":"binding","index":0}}`.
`<anchor0>` denotes the exact JSON expression
`{"kind":"path","value":{"kind":"variable","sort":"anchor","index":0}}`;
its corresponding rule pointer-anchor entry uses the same anchor variable with
source type `*const i8` and target type `&[i8]`. These identity-aware path
carriers do not authorize arbitrary-expression generalization.

### A9-APPLY-01 `rules_materialize_and_compile`

Apply each A9-SYN-02 rule to a third compatible function. Expected applied
view uses the exact trusted field and fully qualified wrapper expression,
marks the statement `rule_applied`, and compiles. A partial argument miss keeps
the whole statement `transform`. A rule for one exact specifier never applies
to any near-miss listed in A9-SYN-01.

### A9-APPLY-02 `cargo_remains_the_target_type_check`

Use these two exact rigid rule targets independently:

```rust
::proctor_libc::printf::signed([1_i32, 2_i32])
::proctor_libc::printf::byte_string(17_i32)
```

The first is attached to an otherwise valid `%d` rule and the second to an
otherwise valid `%s` rule. In each case rule-document validation succeeds,
selection/materialization produces a `rule_applied` print statement, and
structural candidate validation succeeds because the trusted format and slot
count are unchanged. Cargo compilation then fails: `[i32; 2]` is not an
accepted `signed` input, while `byte_string` requires `&[i8]` rather than `i32`.
The candidate is rolled back and the existing whole-SCC baseline
fallback runs once. Do not add target root-type inference or wrapper
enforcement to make either case fail earlier.

## 11. Python skeleton loading and format guidance

### A9-PY-WIRE-01 `function_field_is_required_and_closed`

Load a minimal valid function with `printf_format_specifiers: []`, then test
each mutation independently: missing field, extra field, nonarray, Boolean,
nonstring entry, empty string, duplicate, and descending order. Expected
`SkeletonError` naming the record and field. A valid sorted array becomes an
immutable tuple. Nonfunction record keys are unchanged.

### A9-PY-WIRE-02 `classifier_is_minimal_without_reimplementing_lowering`

Feed the focused classifier the exact arrays below. Each returns only the
listed wrapper family:

| Metadata input | Exact selected family |
| --- | --- |
| `["%d", "%lli"]` | `signed` |
| `["%#o", "%08x", "%X", "%u"]` | `unsigned` |
| `["%f"]` | `fixed` |
| `["%LF"]` | `fixed_upper` |
| `["%LE", "%e"]` | `scientific` |
| `["%g"]` | `general` |
| `["%LG"]` | `general_upper` |
| `["%LA", "%a"]` | `hex_float` |
| `["%10.3s"]` | `byte_string` |

The exact invalid inputs `""`, `"é%d"`, `"text%d"`, `"%q"`, and `"%%"`
raise `SkeletonError` naming `printf_format_specifiers`. To prove Python is not
a second C parser, synthetic `%bogusf`, `%*d`, and `%d%d` conservatively select
`fixed`, `signed`, and `signed`, respectively, by recognized final byte. Such
values cannot be emitted by Crat. Rust remains authoritative for full grammar,
numeric bounds, and canonical fields.

### A9-PY-GUIDE-01 `one_family_does_not_explain_another`

Render guidance separately for each metadata array in A9-PY-WIRE-02. The
signed classifier input `['%d','%lli']` produces nonempty generic invariants
but no signed call, signed accepted-input list, or integer-only `i128`/`u128`
warning because every signed member is proven native-safe. Each other output
contains the exact fully qualified selected call expression once and contains
none of this fixed forbidden set after removing the selected member:

```text
proctor_libc::printf::signed
proctor_libc::printf::unsigned
proctor_libc::printf::fixed
proctor_libc::printf::fixed_upper
proctor_libc::printf::scientific
proctor_libc::printf::general
proctor_libc::printf::general_upper
proctor_libc::printf::hex_float
proctor_libc::printf::byte_string
```

No rendered line begins with `use ` or `fn `. Inputs
`["%.0d","% d"]`, `["%X","%o","%u","%x"]`, and
`["%E","%Le","%e"]` each produce one family entry, not one entry per
specifier. Each adapter rendering includes exactly its relevant accepted-input
group: signed `i8`/`i16`/`i32`/`i64`/`isize`, unsigned
`u8`/`u16`/`u32`/`u64`/`usize`, floating `f32`/`f64`/`f128::f128`, or byte
string `&[i8]`; unrelated groups are absent.

### A9-PY-GUIDE-01A `native_signed_proof_is_universal_and_fail_closed`

Each of `%d`, `%i`, `%hhd`, `%hd`, `%ld`, `%lld`, `%jd`, `%zd`, `%td`,
`%8d`, `%-8d`, `%+8d`, `%08d`, and valid repetitions/permutations of `-`,
`+`, and `0` renders nonempty trusted-format, slot-order, and C-length generic
guidance without the signed call, signed type set, adapter introduction, or
integer-only `i128`/`u128` warning. Each of `%.d`, `%.0d`, `%.5d`, `%08.5d`,
`%-08.5i`, `% d`, `%+ d`, `%*d`, `%d%d`, and `%bogusd` retains the signed
call and type set. The retained text says to use
`proctor_libc::printf::signed(value)` for integer precision or space-sign
behavior, to pass ordinary signed conversions as their correctly converted
values, and to retain the adapter conservatively when native safety is not
proven.

For `['%d','%E']`, scientific guidance remains but signed-specific guidance is
absent. For `['%d','%.0d']`, signed guidance appears once with the narrow
wording above. Two safe signed SCC members omit signed guidance regardless of
member order; adding one exceptional signed member forces retention regardless
of order. An exceptional signed dependency outside the current SCC does not
force retention.

### A9-PY-GUIDE-02 `space_sign_is_selected_from_flags`

Compare `%d`, `% d`, `%+d`, `%+ d`, `%08.3d`, `% f`, `% E`, `% G`, `% A`,
and synthetic `% s`. Expected `.space_sign()` guidance only for
signed/floating specifiers containing a space in their flag prefix. `%+ d`
says that the Rust `+` field wins while `.space_sign()` remains safe/required
guidance. `% s` cannot be emitted by Crat, but the minimal classifier
conservatively selects `byte_string` and does not mention `.space_sign()`;
unsigned and string guidance never mentions the method.

### A9-PY-GUIDE-03 `guidance_states_the_type_and_qualification_contract`

With all nine families selected, assert the guidance contains these exact
behavioral fragments once: `fully qualified`, `after the C length conversion`,
`do not define or import`, `let Rust infer their return types`,
`Rust's e/E formatting trait`, `Rust's x/X formatting trait`, the exact signed,
unsigned, floating, and byte-string input sets, `first NUL`, `counts bytes`,
and `valid UTF-8`. It contains each corresponding directly usable call
expression:

```rust
proctor_libc::printf::signed(value)
proctor_libc::printf::unsigned(value)
proctor_libc::printf::fixed(value)
proctor_libc::printf::fixed_upper(value)
proctor_libc::printf::scientific(value)
proctor_libc::printf::general(value)
proctor_libc::printf::general_upper(value)
proctor_libc::printf::hex_float(value)
proctor_libc::printf::byte_string(value)
```

It contains none of the internal trait or return-type names `SignedValue`,
`UnsignedValue`, `FixedValue`, `Signed<T>`, `Unsigned<T>`, `Fixed<T>`,
`FixedUpper<T>`, `Scientific<T>`, `General<T>`, `GeneralUpper<T>`,
`HexFloat<T>`, or `ByteString<'_>`, and explicitly rejects casts to unsupported
`i128`/`u128`. For metadata `["% d"]`, it shows
`proctor_libc::printf::signed(value)`, explicitly says to chain `.space_sign()`
directly on that call result, and says `+ takes precedence`. For metadata
`["%d"]`, both `.space_sign()` fragments and all signed-specific text are
absent, but the generic invariants remain. Assert the prompt does not contain
the module-level prose beginning `Formatting adapters for C printf semantics`.
Every nonempty format-guidance rendering contains the exact A9-VAL-03 advisory
sentence once.

### A9-PY-SCC-01 `only_current_members_contribute`

Function 0 depends on nonmember function 1. Function 0 has `['%d']`; function
1 has `['%E']`. Rendering SCC `(0,)` contains only generic printf invariants
and no adapter call, while rendering `(1,)` mentions only `scientific`.
Dependency context does not activate format guidance.

### A9-PY-SCC-02 `member_union_is_stable`

An SCC has these valid sorted member arrays: `["% d","%s"]`,
`["%#x","%E"]`, and `["%d","%s"]`. Forward and reversed member traversal
produce byte-identical guidance in catalog order with one entry for signed,
unsigned, scientific, and byte string, and one space-sign explanation.

### A9-PY-SCC-03 `no_llm_means_no_prompt`

An SCC containing only zero-argument mechanical calls or complete
`rule_applied` calls performs replacement/build without an LLM, validator, or
prompt rendering. Empty format arrays produce no guidance section. If another
member still needs the LLM, function-level metadata from a rule-applied call
may cause bounded over-inclusion; this is the approved tradeoff and is tested
as stable behavior.

### A9-PY-SCC-04 `repair_and_fallback_keep_complete_guidance`

Start with an applied view containing rule-applied `%#x` and native-safe `%d`
calls and an LLM `%E` call. Render the initial request, a validation repair, a
Cargo-diagnostic repair, then force rule-involved build failure and switch the
whole SCC to baseline. Every emitted request has the same complete
member-level unsigned and scientific guidance, omits signed-specific guidance,
uses prompt id/version `local_transformation`/`1`, and has current failure
context only. No request returns to the applied view after fallback.

### A9-PY-PROMPT-01 `version_one_template_has_the_new_slot`

The exact version-1 frontmatter variable array is:

```toml
["dependency_context", "transformation_targets", "repair_context",
 "use_xj_scanf_guidance", "libc_guidance", "printf_guidance",
 "foreign_static_guidance"]
```

Rendering with `printf_guidance=""` contains none of the nine wrapper paths in
A9-PY-GUIDE-01. Rendering with exceptional signed guidance contains
`proctor_libc::printf::signed` once. Existing text `Return exactly one Rust
code block delimited by triple-backtick fences.` remains present in both.
Request metadata records the new content hash but exact id/version
`local_transformation`/`1`; stage output reports exactly that one prompt use.

## 12. Mutable `strto*` prompt references

### A9-STRTO-01 `all_five_entries_show_const_and_mutable_forms`

Parameterize by exact foreign name and require exactly these two signatures in
the one selected guidance entry:

```rust
pub fn strtod(buf: &[i8]) -> ((f64, &[i8]), Result<(), StrtoFloatError>);
pub fn strtod_mut(buf: &mut [i8]) -> ((f64, &mut [i8]), Result<(), StrtoFloatError>);
pub fn strtof(buf: &[i8]) -> ((f32, &[i8]), Result<(), StrtoFloatError>);
pub fn strtof_mut(buf: &mut [i8]) -> ((f32, &mut [i8]), Result<(), StrtoFloatError>);
pub fn strtold(buf: &[i8]) -> ((f128::f128, &[i8]), Result<(), StrtoFloatError>);
pub fn strtold_mut(buf: &mut [i8]) -> ((f128::f128, &mut [i8]), Result<(), StrtoFloatError>);
pub fn strtol(buf: &[i8], base: i32) -> ((i64, &[i8]), Result<(), StrtoIntError>);
pub fn strtol_mut(buf: &mut [i8], base: i32) -> ((i64, &mut [i8]), Result<(), StrtoIntError>);
pub fn strtoul(buf: &[i8], base: i32) -> ((u64, &[i8]), Result<(), StrtoIntError>);
pub fn strtoul_mut(buf: &mut [i8], base: i32) -> ((u64, &mut [i8]), Result<(), StrtoIntError>);
```

The exact activation-to-reference pairs are `strtod`/`strtod_mut`,
`strtof`/`strtof_mut`, `strtold`/`strtold_mut`, `strtol`/`strtol_mut`, and
`strtoul`/`strtoul_mut`. Each selected entry uses both fully qualified paths,
states that the immutable form accepts and returns a suffix of `&[i8]`, states
that the mutable form accepts and returns a suffix of `&mut [i8]`, and retains
the current `Result` error semantics shown in the signatures.

### A9-STRTO-02 `activation_remains_exact_and_concise`

`strtod` activates only its two references; it does not activate the other four
families or printf guidance. Names `strtod_mut`, `rust_strtod`, `vstrtod`, and
case variants activate nothing because they are not compiler-reported C
foreign names. Combined SCC foreign names deduplicate and remain in catalog
order. Initial, repair, and baseline-fallback prompts contain identical
references and remain version 1.

## 13. Direct ctype preparation

Use this exact mapping in table-driven compiler tests:

| C symbol | Replacement |
| --- | --- |
| `isalnum` | `::proctor_libc::isalnum` |
| `isalpha` | `::proctor_libc::isalpha` |
| `isblank` | `::proctor_libc::isblank` |
| `iscntrl` | `::proctor_libc::iscntrl` |
| `isdigit` | `::proctor_libc::isdigit` |
| `isgraph` | `::proctor_libc::isgraph` |
| `islower` | `::proctor_libc::islower` |
| `isprint` | `::proctor_libc::isprint` |
| `ispunct` | `::proctor_libc::ispunct` |
| `isspace` | `::proctor_libc::isspace` |
| `isupper` | `::proctor_libc::isupper` |
| `isxdigit` | `::proctor_libc::isxdigit` |
| `tolower` | `::proctor_libc::tolower` |
| `toupper` | `::proctor_libc::toupper` |

### A9-CTYPE-D01 `all_direct_functions_rewrite`

For each row, declare `fn name(c: c_int) -> c_int` inside a local
non-unwinding `extern "C"` block and call it with an effectful expression:

```rust
pub unsafe fn classify(mut n: i32) -> i32 {
    isalpha({ n += 1; n })
}
```

Expected call is exactly
`::proctor_libc::isalpha({ n += 1; n })`, the argument occurs once, the
foreign declaration may remain until later unused-item cleanup, the result
compiles, and `requires_proctor_libc` is true. No cast or type-driven rewrite is
applied to the direct-call argument. Add these four finite fixtures:
`isalpha(isdigit(n))`, `if isalpha(n) != 0 { 1 } else { 0 }`,
`return isalpha(n);`, and `match n { 0 => isalpha(n), _ => 0 }`. Their exact
rewritten callees are respectively nested `::proctor_libc::isalpha` and
`::proctor_libc::isdigit`, then one fully qualified `isalpha` in each remaining
fixture; each sets `requires_proctor_libc` once.

### A9-CTYPE-D02 `direct_recognition_matches_the_established_syntax`

For all fourteen names, test a one-segment `ExprKind::Path` callee with exactly
one argument. Expected rewrite follows the textual mapping table. The pass does
not inspect a declaration's ABI, signature, linked symbol, or provenance; the
ordinary compiler harness merely ensures each positive fixture is valid input.
For every `ctype_name` in the fourteen-row mapping, instantiate this exact
source-defined-local template after substituting that name:

```rust
fn ctype_name(c: i32) -> i32 { c + 100 }
fn use_it(c: i32) -> i32 { ctype_name(c) }
```

Despite each local definition, `ctype_name(c)` rewrites exactly to the mapped
`::proctor_libc::ctype_name(c)` and sets `requires_proctor_libc`; these fourteen
positive rows fix the approved syntax-only/no-provenance behavior.

### A9-CTYPE-D03 `direct_near_misses_are_untouched`

Compile each finite row independently. Every call remains unchanged and the
preparation result has `requires_proctor_libc == false`:

| Boundary | Declaration/binding and exact call |
| --- | --- |
| zero arguments | `fn isalpha() -> i32 { 7 }`; `isalpha()` |
| two arguments | `fn isalpha(a: i32, b: i32) -> i32 { a + b }`; `isalpha(c, c)` |
| qualified path | `mod helpers { pub fn isalpha(c: i32) -> i32 { c } }`; `helpers::isalpha(c)` |
| method call | `impl Helper { fn isalpha(&self, c: i32) -> i32 { c } }`; `helper.isalpha(c)` |
| indirect value | `fn isalpha_fn(c: i32) -> i32 { c }`; `let classify: fn(i32) -> i32 = isalpha_fn; classify(c)` |
| macro token | `macro_rules! isalpha { ($c:expr) => { $c } }`; `isalpha!(c)` |
| prefixed spelling | `fn c_isalpha(c: i32) -> i32 { c }`; `c_isalpha(c)` |
| suffixed spelling | `fn isalpha_extra(c: i32) -> i32 { c }`; `isalpha_extra(c)` |

ABI/signature/link-name/DefId hardening is deliberately not tested or
implemented in this amendment; unmatched forms remain ordinary
local-transformation work.

### A9-CTYPE-D04 `defined_domain_rewrites_and_compiles`

Prepare and compile each finite row without executing it:

| Exact source expression | Exact prepared expression |
| --- | --- |
| `isalpha(-1)` | `::proctor_libc::isalpha(-1)` |
| `isalpha(65)` | `::proctor_libc::isalpha(65)` |
| `iscntrl(31)` | `::proctor_libc::iscntrl(31)` |
| `isspace(9)` | `::proctor_libc::isspace(9)` |
| `isdigit(48)` | `::proctor_libc::isdigit(48)` |
| `tolower(-1)` | `::proctor_libc::tolower(-1)` |
| `tolower(65)` | `::proctor_libc::tolower(65)` |
| `toupper(-1)` | `::proctor_libc::toupper(-1)` |
| `toupper(97)` | `::proctor_libc::toupper(97)` |

Every row sets `requires_proctor_libc == true`. Runtime values—including
classification normalization to `0/1`, EOF handling, and ASCII case
conversion—are cited expectations of the published proctor-libc v0.3.0 tests,
not re-tested by Crat. Specifically, those upstream tests are expected to cover
these input/output pairs: `isalpha(-1)=0`, `isalpha(65)=1`,
`isalpha(127)=0`; `iscntrl(31)=1`, `iscntrl(32)=0`, `iscntrl(127)=1`;
`isspace(9)=1`, `isspace(13)=1`, `isspace(32)=1`, `isspace(33)=0`;
`isdigit(47)=0`, `isdigit(48)=1`, `isdigit(57)=1`, `isdigit(58)=0`;
`tolower(-1)=-1`, `tolower(64)=64`, `tolower(65)=97`, `tolower(90)=122`,
`tolower(91)=91`; and `toupper(-1)=-1`, `toupper(96)=96`,
`toupper(97)=65`, `toupper(122)=90`, `toupper(123)=123`. Locale-changing
behavior and negative values other than EOF remain outside the guaranteed
domain.

## 14. glibc ctype table idioms

Positive compiler fixtures declare these common C2Rust accessor signatures so
the input type-checks:

```rust
fn __ctype_b_loc() -> *mut *const u16;
fn __ctype_tolower_loc() -> *mut *const i32;
fn __ctype_toupper_loc() -> *mut *const i32;
```

Recognition itself is syntactic: a zero-argument one-segment call with the
shown name inside the exact deref/offset shape. It does not validate the
declaration, linked symbol, or provenance.

### A9-CTYPE-M01 `all_single_mask_names_map`

For each mapping below, compile the C2Rust shape:

```rust
*(*__ctype_b_loc()).offset(c as i32 as isize) as i32
    & _ISalpha as i32 as u16 as i32
```

The left operand must have exactly the current libc-pass form: cast of the
loaded value, final dereference, `.offset(e as isize)` with one argument,
parenthesized first dereference, then a zero-argument one-segment
`__ctype_b_loc` call. The right operand is a cast-peeled one-segment `_IS*`
path. Expected replacement is the matching fully qualified call with exact
argument `c as i32`: only the terminal `as isize` is stripped, the retained
expression is evaluated once, and `requires_proctor_libc` is true:

| Constant | Replacement |
| --- | --- |
| `_ISupper` | `::proctor_libc::isupper` |
| `_ISlower` | `::proctor_libc::islower` |
| `_ISalpha` | `::proctor_libc::isalpha` |
| `_ISdigit` | `::proctor_libc::isdigit` |
| `_ISxdigit` | `::proctor_libc::isxdigit` |
| `_ISspace` | `::proctor_libc::isspace` |
| `_ISprint` | `::proctor_libc::isprint` |
| `_ISgraph` | `::proctor_libc::isgraph` |
| `_ISblank` | `::proctor_libc::isblank` |
| `_IScntrl` | `::proctor_libc::iscntrl` |
| `_ISpunct` | `::proctor_libc::ispunct` |
| `_ISalnum` | `::proctor_libc::isalnum` |

Use these exact `u32` fixture values: `_ISupper=256`, `_ISlower=512`,
`_ISalpha=1024`, `_ISdigit=2048`, `_ISxdigit=4096`, `_ISspace=8192`,
`_ISprint=16384`, `_ISgraph=32768`, `_ISblank=1`, `_IScntrl=2`, `_ISpunct=4`,
and `_ISalnum=8`. The pass selects by textual name and does not evaluate these
values; changing `_ISalpha` to `7` in a separate positive fixture still
rewrites to `::proctor_libc::isalpha(...)`.

Also compile this compatible source-defined accessor fixture:

```rust
fn __ctype_b_loc() -> *mut *const u16 { ::std::ptr::null_mut() }
const _ISalpha: u32 = 1024;
unsafe fn classify(c: i32) -> i32 {
    *(*__ctype_b_loc()).offset(c as isize) as i32 & _ISalpha as i32
}
```

Despite the local definition, the table expression rewrites exactly to
`::proctor_libc::isalpha(c)`, compiles, and sets
`requires_proctor_libc == true`. This positively fixes the approved
syntax-only/no-provenance behavior for table accessors.

### A9-CTYPE-M02 `numeric_and_zero_comparison_contexts_are_exact`

Let the exact selected node be:

```rust
*(*__ctype_b_loc()).offset(c as i32 as isize) as i32
    & _ISalpha as i32 as u16 as i32
```

Test it bare and inside `node != 0`, `0 != node`, `node == 0`, and `0 == node`.
The outer operator and operand order remain structurally unchanged after
`node` is replaced. A set result is `1`, not raw `_ISalpha`.

The exact replacement for `node` is
`::proctor_libc::isalpha(c as i32)`: the terminal `as isize` is removed and no
cast is added. Repeat with index `c as u8 as i32 as isize`; its exact
replacement is `::proctor_libc::isalpha(c as u8 as i32)`. Every inner cast
survives, no expression type is queried, and `c` occurs once.

### A9-CTYPE-M03 `lower_and_upper_tables_rewrite`

The exact lookup forms:

```rust
*(*__ctype_tolower_loc()).offset(c as i32 as isize)
*(*__ctype_toupper_loc()).offset(c as i32 as isize)
```

have exact outputs `::proctor_libc::tolower(c as i32)` and
`::proctor_libc::toupper(c as i32)`.

Use the two shown loads directly and inside source-defined one-segment inline
helpers named `tolower` and `toupper`; both loads produce those same exact
calls. Recognition is by accessor spelling and exact shape. For an effectful
index `({ c += 1; c }) as isize`, the exact outputs are
`::proctor_libc::tolower({ c += 1; c })` and
`::proctor_libc::toupper({ c += 1; c })`. Only the terminal `as isize` is
removed, no cast is added, and the block appears once.

### A9-CTYPE-M04 `table_near_misses_are_untouched`

Each literal expression below is `Same(input, false)` when it is the only
candidate:

```rust
*__ctype_b_loc()
_ISalpha as i32 & (*(*__ctype_b_loc()).offset(c as isize) as i32)
(*(*__ctype_b_loc()).offset(c as isize) as i32) & (_ISalpha | _ISdigit) as i32
(*(*__ctype_b_loc()).offset(c as isize) as i32) & mask
(*(*crate::__ctype_b_loc()).offset(c as isize) as i32) & _ISalpha as i32
(*(*__ctype_b_loc()).offset(c as isize) as i32) & crate::_ISalpha as i32
(*(*__ctype_b_loc(0)).offset(c as isize) as i32) & _ISalpha as i32
(*(*__ctype_b_loc()).offset(c) as i32) & _ISalpha as i32
(*__ctype_b_loc()).add(c as usize)
*(*__ctype_tolower_loc()).offset(c)
*(*__ctype_toupper_loc()).offset(c)
*(*__ctype_tolower_loc()).add(c as usize)
*(*crate::__ctype_toupper_loc()).offset(c as isize)
```

The three no-terminal-cast rows declare `c: isize`; all are nonmatches even
though they type-check. Give every other fixture compatible local declarations
so it compiles. Also remove, one at a time, the required loaded-value cast,
receiver parentheses, first deref, or final deref from the positive b-loc
form; each is a nonmatch. No inner accessor is partially rewritten, no error is
raised, and the expressions remain ordinary local-transformation work.

## 15. Prepare result, dependency, and pass integration

### A9-PREP-01 `result_flag_is_exact`

Pure `preparer` tests assert:

- match-arm wrapping/local-static lifting only: transformed source and
  `requires_proctor_libc == false`;
- one successful direct or table ctype rewrite: `true`;
- multiple ctype rewrites: still one Boolean `true`;
- ctype near misses only: `false`; and
- any `PrepareError`: no successful result or dependency signal.

### A9-PREP-02 `publication_requests_dependency_only_after_success`

Exercise `PreparationResult` publication through its module-private pure
callback seam, without invoking a binary or filesystem, to capture writes and
dependency requests. A successful result with flag false writes once and
requests none; flag true writes once and requests canonical crates.io
`proctor-libc` minimum 0.3.0 exactly once. A failed dependency operation writes
no transformed source. A later source-write failure is returned after the
dependency request. Established `PrepareError` cases remain covered at the
preparer library seam and produce no `PreparationResult` or dependency signal.
With manifest path `/work/project/Cargo.toml` and injected cause
`Cargo [dependencies] must be a table`, the exact publication diagnostic is:

```text
failed to ensure proctor-libc dependency in /work/project/Cargo.toml: Cargo [dependencies] must be a table
```

Publication returns unsuccessfully before invoking the source-write operation.
The dependency-ensure path contains no `panic!`, `unwrap`, or `expect`.

`utils::dependency::ensure_crates_io_dependency` and its pure normalization
function implement the generic preservation-aware manifest policy used here.
`PreparationResult::publish` calls that utils API only when successful
preparation requests proctor-libc, then writes the source. The binary contains
only thin pass dispatch and has no tests or test-only helpers. The generic utils
API does not call the existing unconditional `utils::add_dependency`, and the
pass library does not hand-edit Cargo text.

### A9-PREP-02A `target_manifest_policy_matches_the_stage`

Exercise the module-private pure normalization seam behind
`utils::dependency::ensure_crates_io_dependency` with the same manifest values
as A9-DEP-01--08 and A9-DEP-12. Expected
absent/lower crates.io requirements
become 0.3.0; features and default-feature fields survive table upgrades;
sufficient requirements and canonical path/git/workspace/alternate-registry
entries are preserved. Aliases, canonical-name collisions, and all four exact
A9-DEP-12 malformed inputs return `Err` without source output. After
`PreparationResult::publish` adds context, each diagnostic starts with
`failed to ensure proctor-libc dependency in /work/project/Cargo.toml: ` and
ends with its specific cause; no `unwrap`, `expect`, or panic occurs. Other
dependencies and package/lib tables remain identical. A no-ctype-rewrite
result makes no dependency-planner call and performs no manifest rewrite.

### A9-PREP-02B `dependency_success_precedes_source_write`

Use the module-private pure callback seam behind `PreparationResult::publish`,
not a filesystem or binary test. For a ctype result, dependency planning/commit
occurs before the transformed source write. A dependency failure produces no
source write. A later source-write failure is fatal but may leave the now-valid,
harmless dependency; this amendment does not require a custom two-file rollback
protocol. A no-rewrite result writes the prepared source and performs no
dependency action.

### A9-PREP-03 `preparation_result_is_atomic`

Combine a non-block arm, a movable local static, and a qualifying ctype call,
then trigger each established prepare error: scoped-static dependency,
unsupported direct async function/prelude, or missing mapping. Expected no
source is returned, no ctype call or old construct is partially changed, and
no proctor-libc dependency is requested.

### A9-PREP-04 `prepare_is_idempotent_with_ctype`

Run prepare twice on this finite fixture (with the declarations/constants from
Section 14):

```rust
pub unsafe fn combined(c: i32) -> i32 {
    static CALLS: i32 = 1;
    match c {
        0 => isalpha(c) + CALLS,
        _ => (*(*__ctype_b_loc()).offset(c as isize) as i32
              & _ISspace as i32)
             + *(*__ctype_tolower_loc()).offset(c as isize),
    }
}
```

The first result compiles, lifts `CALLS`, block-wraps both arms, introduces
exactly one each of `::proctor_libc::isalpha`, `isspace`, and `tolower`, and
sets the dependency flag. Their exact arguments are `c`, `c`, and `c`: direct
replacement preserves its argument, while both table replacements strip only
the terminal `as isize`. The second canonical AST equals the first and its
dependency flag is false. No path becomes doubly qualified and no dependency
request is duplicated.

### A9-PREP-05 `neighboring_passes_accept_the_output`

Use this exact prepared input pair:

```rust
pub unsafe fn direct(c: i32) -> i32 {
    if ::proctor_libc::isalpha(c) != 0 { 1 } else { 0 }
}
pub unsafe fn table(c: i32) -> i32 {
    ::proctor_libc::isspace(c)
}
```

Run it separately through `simpl`, configured `unsafe` cleanup with `direct`
and `table` in `c_exposed_fns`, and configured `unexpand`, compiling each
output. Every output contains exactly one `::proctor_libc::isalpha(c)` and one
`::proctor_libc::isspace(c)` call and contains no `__ctype_b_loc`. In a second
input, retain unused foreign declarations for `isalpha` and `__ctype_b_loc`;
prepare may leave them, while configured unsafe unused-item cleanup removes
them without removing the two public functions. Skeleton foreign-body metadata
for `direct`/`table` is empty. No Python ctype guidance entry exists.

### A9-PREP-06 `checked_in_pipeline_needs_no_config_change`

`proctor/configs/c2rust_crat_local.toml` still contains the existing `prepare`
position and gains no ctype flag, duplicate pass, or versioned planning name.
The local-transformation stage's internal preparation remains exactly
`("prepare", current_project, ("expand", "unexpand"), True)` and does not
invoke ordinary `Pass::Prepare` again.

## 16. Cross-component end-to-end scenarios

### A9-E2E-01 `prepare_then_local_printf_transformation`

Build an in-memory/fake-project source with a direct `isalpha` call and:

```rust
printf(b"% d %#08.4x %E %10.3s\n\0" as *const u8 as *const i8,
       d, x, f, s);
```

After ordinary prepare, expected ctype call is fully qualified and the target
manifest requires proctor-libc 0.3.0. Skeleton generation lowers the printf,
emits four exact function-level specifiers, and does not report `isalpha` as a
remaining foreign body call. Python renders only signed/space-sign, unsigned,
scientific, and byte-string wrapper guidance. The accepted wrapper-bearing LLM
candidate validates, builds, replaces, and yields four observations. Published
artifacts retain their current names and version-1 document shapes.

### A9-E2E-02 `rule_failure_falls_back_without_losing_metadata`

Apply a learned unsigned rule to the `%#08.4x` argument, leave `%E` for the
LLM, and force the applied candidate build to fail. Expected one rollback and
whole-SCC baseline fallback; the second prompt still includes both unsigned
and scientific guidance from function-level metadata; only the accepted
baseline attempt contributes observations and report pairs.

### A9-E2E-03 `unsupported_cases_do_not_leak_partial_state`

Combine an unsupported `%p`, a qualified fake `helpers::isalpha`, and no other
supported operation. Expected ordinary transform behavior, empty printf
metadata, no format guidance, no ctype rewrite, and no prepare dependency
request. The local stage still normalizes its independent required
proctor-libc dependency
to 0.3.0 before building, so absence of a prepare request is not inferred from
the final manifest alone; the test observes both seams separately.

## 17. Regression and completion matrix

### A9-REG-01 `non_printf_tool_behavior_is_stable`

Run current scanf, generic foreign call/static, pointer decision, maximal
region, rigid literal, ordinary observation/rule, preservation, wrapper, and
call-redirection suites. Expected no semantic difference except the new empty
function metadata key in skeleton fixtures.

### A9-REG-02 `non_ctype_prepare_behavior_is_stable`

Run all Amendment 8 match-arm, local-static, name-allocation, DefId rewrite,
dependency validation, export, error-layer, idempotence, and interaction tests.
Expected unchanged source/error behavior and `requires_proctor_libc == false`
unless the fixture deliberately adds a qualifying ctype use.

### A9-REG-03 `determinism_and_closed_boundaries_hold`

For one fixed source, fixed rule set, fixed fake LLM response, and fixed member
order, run the transformation twice. Expected byte-identical skeleton JSON,
rendered prompts, accepted reports, observation/rule JSON, and prepared Rust.
Independently reorder SCC member traversal and expect only the canonically
ordered guidance text to remain identical; independently reorder source
statements and expect only the sorted function-level specifier array to remain
identical. Reordered wire arrays themselves are rejected, not normalized.
Dependency planning on the same manifest yields the same parsed TOML result
and one request. Unknown JSON fields and malformed metadata fail at their
owning boundary.

### A9-REG-04 `no_unapproved_surface_change`

Expected unchanged stage input/output schemas, `stage.toml`, artifact names,
statistics shape, rule-set input mutability, CLI subcommands, local config,
`proctor.toml`, and all document schema versions. The only prompt change is the
focused format-guidance slot and the five mutable strto references. The only
new prepare side effect is a conditional canonical proctor-libc 0.3.0 request.

Implementation is complete only when:

| Requirement | Cases |
| --- | --- |
| all dependency sites use and preserve a 0.3.0 minimum correctly | A9-UPD-02, A9-DEP-01--12, A9-PREP-02--02B |
| the complete approved static printf domain lowers canonically | A9-UPD-01, A9-FMT-01--25 |
| unsupported printf remains atomic and total | A9-FMT-N01--17, A9-ELIG-02--03 |
| trusted templates and function-level metadata are exact | A9-SKEL-01--05, A9-VAL-01--03, A9-REP-01--02 |
| observations and version-1 rules cover every wrapper family | A9-OBS-01--03, A9-WIRE-01, A9-SYN-01--02, A9-APPLY-01--02 |
| Python selects complete but concise SCC guidance | A9-PY-WIRE-01--02, A9-PY-GUIDE-01--03 (including 01A), A9-PY-SCC-01--04, A9-PY-PROMPT-01 |
| all five strto families explain mutable borrowing | A9-UPD-05, A9-STRTO-01--02 |
| direct and table ctype forms follow the approved common syntax | A9-CTYPE-D01--04, A9-CTYPE-M01--04 |
| prepare-result atomicity, dependency ordering, idempotence, and pass interactions hold | A9-UPD-06, A9-PREP-01--06 |
| cross-component behavior and unrelated regressions hold | A9-E2E-01--03, A9-REG-01--04 |

The exact commands in Section 2 must all pass. Any optional real-project run
is additional evidence, not a substitute for this matrix.
