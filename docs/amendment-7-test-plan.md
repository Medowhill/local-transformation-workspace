# Amendment 7 Test Plan: `printf` Argument Rules

## 1. Purpose and test discipline

This plan verifies the special whole-statement `printf` transformation defined
in [amendment-7-plan.md](amendment-7-plan.md). Every case gives concrete input
and expected output or rejection. Tests must use current in-memory Crat compiler
harnesses and PROCTOR's existing fake stage harness; they must not invoke the
`crat-tool` CLI, mutate a project-root test tree, require a network service, or
change historical fixtures merely to hide a new required field.

Use exact equality for canonical JSON, skeleton/disposition forests, converted
format strings, macro semantic literals, argument counts/order, sidecars,
statistics, and tool event order. Use structural AST/HIR equality where source
formatting is intentionally normalized. Every negative must assert the precise
local outcome: converter `Unsupported`, ordinary transform fallback, validator
`invalid`, protocol/setup error, no observation, rule candidate miss, or fatal
stage failure.

## 2. Shared compiler fixtures and notation

Unless a case changes it explicitly, use a local supported declaration:

```rust
unsafe extern "C" {
    fn printf(format: *const ::std::os::raw::c_char, ...)
        -> ::std::os::raw::c_int;
}
```

and this C2Rust-style literal chain:

```rust
b"value=%d\n\0" as *const u8 as *const ::std::os::raw::c_char
```

`C("...")` below means run the pure converter on the decoded bytes before the
single terminal NUL. `Ok("rust", ["spec", ...])` means the exact semantic Rust
format string and exact consuming source specifiers. `No` means `Unsupported`,
not panic or partial output.

All positive observation/rule/application fixtures are inside the settled
semantic domain: source C has no undefined behavior and every vararg expression
has the C-defined conversion-compatible promoted type. A negative eligibility
fixture may deliberately fall outside that domain only to prove conservative
rejection and must say so. Tests inspect existing Rust compiler types but do
not add or expect a C default-promotion validator.

For skeleton examples, the label is `#[proctor(0)]`. Pretty-printer whitespace
may follow the existing canonical printer, but the macro path, semantic format,
number/order of arguments, semicolon, disposition, and metadata are exact.

## 3. Pure converter acceptance table

Implement these as table-driven unit tests in the tools-owned converter module.

| Case | Input | Expected |
| --- | --- | --- |
| A7-FMT-01 | `C("plain text\n")` | `Ok("plain text\n", [])` |
| A7-FMT-02 | `C("100%% done")` | `Ok("100% done", [])` |
| A7-FMT-03 | `C("{x} %%")` | `Ok("{{x}} %", [])` |
| A7-FMT-04 | `C("%d")` | `Ok("{}", ["%d"])` |
| A7-FMT-05 | `C("%i %u")` | `Ok("{} {}", ["%i", "%u"])` |
| A7-FMT-06 | `C("%o %x %X")` | `Ok("{:o} {:x} {:X}", ["%o", "%x", "%X"])` |
| A7-FMT-07 | `C("%hhd %hd %ld %lld %jd %zd %td")` | `Ok("{} {} {} {} {} {} {}", ["%hhd","%hd","%ld","%lld","%jd","%zd","%td"])` |
| A7-FMT-08 | `C("%hhu %hu %lu %llu %ju %zu %tu")` | `Ok("{} {} {} {} {} {} {}", ["%hhu","%hu","%lu","%llu","%ju","%zu","%tu"])` |
| A7-FMT-09 | `C("%08x")` | `Ok("{:08x}", ["%08x"])` |
| A7-FMT-10 | `C("%+8d")` | `Ok("{:+8}", ["%+8d"])` |
| A7-FMT-11 | `C("%-8d")` | `Ok("{:<8}", ["%-8d"])` |
| A7-FMT-12 | `C("%-08d")` | `Ok("{:<8}", ["%-08d"])`; C `-` overrides `0` |
| A7-FMT-13 | `C("%s")` | `Ok("{}", ["%s"])` |
| A7-FMT-14 | `C("%f")` | `Ok("{:.6}", ["%f"])` |
| A7-FMT-15 | `C("%F")` | `Ok("{:.6}", ["%F"])` in the finite domain |
| A7-FMT-16 | `C("%lf %lF")` | `Ok("{:.6} {:.6}", ["%lf", "%lF"])`; `l` is a C no-op |
| A7-FMT-17 | `C("%.0f %.3f")` | `Ok("{:.0} {:.3}", ["%.0f", "%.3f"])` |
| A7-FMT-18 | `C("%10.2f")` | `Ok("{:10.2}", ["%10.2f"])` |
| A7-FMT-19 | `C("%-10.2f")` | `Ok("{:<10.2}", ["%-10.2f"])` |
| A7-FMT-20 | `C("%+010.2f")` | `Ok("{:+010.2}", ["%+010.2f"])` |
| A7-FMT-21 | `C("%-+010.2f")` | `Ok("{:<+10.2}", ["%-+010.2f"])` |
| A7-FMT-22 | `C("%#f %#.2f")` | `Ok("{:.6} {:.2}", ["%#f", "%#.2f"])`; decimal point already exists |
| A7-FMT-23 | `C("n=%d s=%s f=%.2f %%")` | `Ok("n={} s={} f={:.2} %", ["%d", "%s", "%.2f"])` |
| A7-FMT-24 | `C("é=%d")` | `Ok("é={}", ["%d"])`; UTF-8 literal text is preserved |
| A7-FMT-25 | `C("%--++0008d %0d %-d")` | `Ok("{:<+8} {} {}", ["%--++0008d", "%0d", "%-d"])`; repeat/no-op flags normalize |

### A7-FMT-26 `accepted_cartesian_matrix_is_complete`

Generate deterministic pure cases, with no host C compiler, for this complete
cross product:

- signed integer conversions `d/i` × lengths `"",hh,h,l,ll,j,z,t` × every
  subset/permutation of accepted flags `-,+,0` × widths absent, `1`, `17`, and
  `2147483647`;
- unsigned conversions `u/o/x/X` × the same lengths × every
  subset/permutation of `-,0` × the same widths;
- `f/F` × lengths `"",l` × every subset/permutation of `-,+,0,#` × widths
  absent, `1`, `17`, `2147483647` × precision absent, `0`, `1`, `17`, and
  `2147483647`, excluding every combination that contains `#` with explicit
  precision `0`; and
- the one bare `%s` case.

For every flag subset, also repeat each present flag once to prove redundant
duplicates normalize. Construct the expected Rust field independently from the
table in the implementation plan: `-` supplies `<` only with width, `-`
removes `0`, `+` follows alignment, width is copied exactly, float precision is
`.6` when absent and `.N` otherwise, `#` is omitted, and the type suffix is
empty/`o`/`x`/`X`. Assert the exact semantic Rust string and exact original
specifier for every generated input. This matrix, not a few representative
rows, defines accepted flag/width/precision coverage.

## 4. Pure converter rejection table

Each input returns `No` for the whole format.

| Case | Input | Reason |
| --- | --- | --- |
| A7-FMT-N01 | `"%"`, `"abc%"` | incomplete conversion |
| A7-FMT-N02 | `"%q"` | unknown conversion |
| A7-FMT-N03 | `"%2$d"` | positional value |
| A7-FMT-N04 | `"%*d"`, `"%2$*3$d"` | dynamic/positional width |
| A7-FMT-N05 | `"%.*f"`, `"%2$.*3$f"` | dynamic/positional precision |
| A7-FMT-N06 | `"% d"`, `"% f"` | space-sign flag |
| A7-FMT-N07 | `"%'d"`, `"%'f"` | locale grouping |
| A7-FMT-N08 | `"%#o"`, `"%#x"`, `"%#X"` | non-equivalent prefixes |
| A7-FMT-N08A | `"%+u"`, `"%+o"`, `"%+x"`, `"%+X"` | plus is accepted only for signed conversions |
| A7-FMT-N09 | `"%.0d"`, `"%.3u"`, `"%08.3x"` | C integer precision |
| A7-FMT-N10 | `"%-s"`, `"%10s"`, `"%.3s"` | byte-count string formatting |
| A7-FMT-N11 | `"%ls"` | wide string |
| A7-FMT-N12 | `"%c"`, `"%lc"` | excluded char conversions |
| A7-FMT-N13 | `"%p"`, `"%n"` | excluded conversions/effects |
| A7-FMT-N14 | `"%e"`, `"%E"` | C exponent spelling is not Rust's |
| A7-FMT-N15 | `"%g"`, `"%G"` | no exact C selection/significant-digit mapping |
| A7-FMT-N16 | `"%a"`, `"%A"` | no standard Rust hexadecimal-float format |
| A7-FMT-N17 | `"%#.0f"`, `"%#.0F"` | Rust cannot force C's decimal point here |
| A7-FMT-N18 | `"%hf"`, `"%llf"`, `"%jf"`, `"%zf"`, `"%tf"` | invalid/unaccepted floating lengths |
| A7-FMT-N19 | `"%Lf"`, `"%LF"` | no exact no-helper `f128::f128` Display mapping |
| A7-FMT-N20 | width/precision `2147483648` | exceeds exact `i32::MAX` cap |
| A7-FMT-N21 | a non-UTF-8 byte in literal text | cannot form equivalent Rust string literal |
| A7-FMT-N22 | `"%5%"`, `"%-%%"` | flags/width on percent are not accepted |

A7-FMT-N19 is required unless the plan is explicitly revised. Record the
evidence in its test comment: pinned `f128` Display uses quadmath `%Qg`, treats
precision as significant digits, and does not honor the needed outer width,
alignment, sign, or zero-padding semantics; the crate's GCC `__float128`
representation is also not universally C `long double`.

### A7-FMT-N23 `rejected_cartesian_matrix_is_complete`

For every consuming conversion and every otherwise accepted combination from
A7-FMT-26, generate a separate `No` case after each applicable mutation:

- insert space or apostrophe in every flag position;
- add `+` to each `u/o/x/X` case or `#` to each integer case;
- add `.0`, `.1`, and `.2147483647` to each integer case;
- add any flag, width, precision, or length to `%s`;
- replace the accepted length with each incompatible length;
- replace fixed width/precision with `*`, positional `N$`, or positional `*N$`;
- replace the conversion with each known-but-unsupported conversion
  `c,p,n,e,E,g,G,a,A` and one unknown byte; and
- for float `#`, set explicit precision to zero.

Assert the complete format is `No`, including a format with an accepted
specifier before and after the mutated one. This proves no partial conversion
leaks from a mixed format.

### A7-FMT-N24 `numeric_limits_are_exact`

Concrete inputs `%2147483647d` and `%.2147483647f` succeed with those exact
digits in their Rust fields. `%2147483648d`, `%.2147483648f`, a 10,000-digit
width, and a 10,000-digit precision return `No` without panic. Leading zeros
are parsed with checked arithmetic and normalize to their numeric value after
flag parsing.

### A7-FMT-N25 `arbitrary_bytes_never_panic`

Call the pure parser under `catch_unwind` for the empty slice, every one-byte
sequence, every two-byte sequence, each byte `0..=255` inserted before/after
`%`, and deterministic long corpora of `%`, digits, braces, NULs, and invalid
UTF-8. Every call returns `Ok` or `Unsupported`; none panics, loops, overflows,
or returns an unconsumed suffix. Eligibility separately rejects interior NUL
and non-UTF-8 literal text.

## 5. Literal and call eligibility

### A7-ELIG-01 `canonical_whole_statement_is_eligible`

Input:

```rust
#[proctor(0)]
printf(b"%d\0" as *const u8 as *const c_char, x);
```

Expected: compiler identity resolves to the shared declaration, literal bytes
are `%d\0`, conversion count is one, and eligibility succeeds.

### A7-ELIG-02 `parentheses_and_validated_const_casts_are_transparent`

Use the required typical input:

```rust
#[proctor(0)]
printf(
    b"%d!\n\0" as *const u8 as *const ::core::ffi::c_char,
    1 as ::core::ffi::c_int,
);
```

Expected: eligible, converted format `{}!\n`, one `%d` consumer, and source
argument intrinsic/adjusted type `i32` on the supported host. Also accept
parentheses around the literal/casts/call and redundant sequences among
`*const u8` and resolved `*const c_char`, provided the terminal cast is exactly
the fixed parameter type.

### A7-ELIG-03 `return_value_uses_are_not_special`

Use separately:

```rust
let n = printf(FMT, x);
return printf(FMT, x);
sink(printf(FMT, x));
printf(FMT, x) + 1;
```

Expected: none is printf-special; each retains the current ordinary LLM hole.
The resolved printf call is also excluded from ordinary foreign-call
observation/rule seeding, so no reusable rule can bypass this boundary.

### A7-ELIG-04 `tail_expression_is_not_ignored_return`

Input a labeled block tail `printf(FMT, x)` without semicolon. Expected:
ineligible and ordinary transform fallback.

### A7-ELIG-05 `literal_terminator_is_exact`

Inputs and expected results:

- `b"%d\0"`: eligible;
- `b"%d"`: no terminal NUL, ineligible;
- `b"%d\0\0"`: interior NUL, ineligible;
- `b"a\0%d\0"`: interior NUL, ineligible;
- `b"\0"`: eligible zero-consuming mechanical format.

### A7-ELIG-06 `only_literal_paren_cast_chain_is_allowed`

Starting from A7-ELIG-02, separately replace the format expression with:

```rust
b"%d\0" as *const u8 as *mut u8 as *const c_char
b"%d\0" as *const u8 as *const u16 as *const c_char
b"%d\0" as *const u8 as usize as *const c_char
&*(b"%d\0" as *const u8) as *const u8 as *const c_char
b"%d\0".as_ptr() as *const c_char
core::mem::transmute::<_, *const c_char>(b"%d\0")
FMT
concat!("%d", "\0").as_ptr() as *const c_char
```

Also use a block, conditional, arithmetic expression, function result, static,
and `include_bytes!`. Expected for every case: cast-chain rejection and the
ordinary whole-statement LLM hole, even if constant propagation could discover
the bytes. No source uses an invalid Rust cast merely to test parser text; each
compiler case must type-check up to the intended eligibility rejection.

### A7-ELIG-07 `linked_identity_not_rust_spelling`

Declare `#[link_name = "printf"] fn c_print(...) -> c_int;` and call `c_print`.
Expected: eligible. Declare a different symbol on a function named `printf`;
expected: ineligible. Import/qualify the supported declaration differently;
expected: compiler identity still succeeds.

### A7-ELIG-08 `prototype_and_abi_are_exact`

Use separate compiling declarations/calls for:

```rust
fn printf(format: *const c_char) -> c_int;                 // nonvariadic
fn printf(format: *const c_char, tag: c_int, ...) -> c_int; // two fixed
fn printf(format: *mut c_char, ...) -> c_int;              // mutable
fn printf(format: *const u16, ...) -> c_int;               // wrong pointee
fn printf(format: *const c_char, ...) -> i64;              // wrong return
```

Repeat the otherwise exact declaration under `extern "C-unwind"` and
`extern "system"`, as a source-defined C function, and as a dependency-owned
foreign item. Expected: compiler fixture succeeds where the ABI is supported,
but printf classifier returns ineligible for every case. Test zero fixed
parameters through the pure resolved-signature classifier descriptor because a
source-language C-variadic declaration without a named parameter is not a
portable compiling Rust fixture. The exact accepted descriptor is local
non-unwinding C ABI, linked symbol `printf`, variadic, one fixed
`*const c_char`, and `c_int` return.

### A7-ELIG-09 `vararg_count_must_equal_consumers`

For format `"%d %% %s"`, exactly two varargs succeeds. Zero, one, or three
varargs rejects the complete special case and emits the current whole-statement
hole. `%%` alone requires zero varargs; one extra rejects.

### A7-ELIG-10 `other_print_functions_are_unchanged`

Direct calls to `fprintf`, `sprintf`, `snprintf`, `vprintf`, `wprintf`, and an
ordinary Rust function named `printf` do not enter this feature. Expected:
current classification, observation, and rule behavior byte-for-byte.

### A7-OPAQUE-01 `return_used_printf_is_ordinary_region_opaque`

Input:

```rust
#[proctor(0)]
let n: c_int = printf(FMT, *p);
```

where the call resolves to the exact prototype but the result use makes it
printf-special ineligible. Expected: ordinary selection returns no region for
the call, `FMT`, `*p`, or the initializer/statement ancestor; ordinary
observation list and ordinary applied-rule set gain nothing from label 0. The
skeleton remains one ordinary transform hole for the LLM.

### A7-OPAQUE-02 `unsupported_format_is_ordinary_region_opaque`

Input whole statement `printf(b"%*d\0" as *const u8 as *const c_char, 5,
*p);`. Expected: unsupported special conversion and ordinary whole-statement
hole. Neither the printf call nor nested `*p` is an ordinary selected region in
observation extraction or application.

### A7-OPAQUE-03 `ancestors_stop_but_disjoint_siblings_survive`

Input one labeled expression statement equivalent to:

```rust
consume(*left + printf(FMT, *hidden), *right);
```

Expected shared ordinary selector marks the exact printf subtree opaque,
selects neither `*hidden`, the containing addition, nor the enclosing
`consume(...)`, but retains the two disjoint regions rooted at `*left` and
`*right` in source order. Ordinary extraction and application return identical
roots/anchors.

### A7-OPAQUE-04 `identity_gate_limits_opacity`

Repeat A7-OPAQUE-03 with a Rust function spelled `printf` and with a foreign
item whose linked symbol is `printf` but prototype is wrong. Expected: neither
is the exact-prototype opaque kind; current ordinary selector behavior applies.

## 6. Skeleton views and disposition forests

### A7-SKEL-01 `one_value_baseline_template`

Input A7-ELIG-01. Expected baseline and no-rule applied skeleton statement:

```rust
#[proctor(0)]
::std::print!("{}", todo!());
```

Both dispositions are `transform`, `needs_transformation` is true, and report
metadata contains label 0 with the original printf Before statement.

### A7-SKEL-02 `multiple_values_have_exact_todos`

Input `printf(b"%d/%08x/%s\0" ..., a, b, c);`. Expected template
`::std::print!("{}/{:08x}/{}", todo!(), todo!(), todo!());` with exactly three
slots in source conversion order.

### A7-SKEL-03 `unsupported_format_keeps_whole_payload_hole`

Input format `%*d` with its width/value args. Expected current ordinary
whole-statement `todo!()` skeleton, not `print!`, no format-argument metadata,
and `transform` disposition.

### A7-SKEL-04 `zero_consumers_are_mechanical_in_both_views`

Input `printf(b"hello %% {world}\n\0" ...);`. Expected both views contain
`::std::print!("hello % {{world}}\n");`, disposition `mechanical`, identical
topology, `needs_transformation == false`, no rule-application marker, and
report metadata for label 0.

### A7-SKEL-05 `mechanical_parent_invariants_are_closed`

Mutate a serialized view so a mechanical node differs between views, contains
a noncanonical payload, has a transform/rule-applied transition, contributes
to `transform_labels`, lacks report metadata, or has a non-preserved child.
Expected: strict Rust and Python loaders/canonicalizers reject the relevant
invariant. A valid mechanical leaf is accepted.

### A7-SKEL-06 `format_argument_rules_are_all_or_nothing`

For three arguments, provide applicable rules for all three. Expected applied
view has three concrete expressions and one `rule_applied` statement. Remove
the middle rule or make its target unmaterializable. Expected entire applied
statement equals the baseline three-todo template and is `transform`; neither
outer argument replacement remains.

### A7-SKEL-07 `ordinary_skeletons_are_unchanged`

Run existing non-printf preservation, preserve-shell, transform, and
rule-applied fixtures after adding `mechanical`. Expected skeleton bytes,
metadata, and dispositions unchanged except complete enum/key lists in tests.

## 7. Validator and replacer defense

Use A7-SKEL-02 as expected view and independently run validator then direct
replacement-library tests. Every attack below must be rejected by both paths
unless it concerns malformed request metadata, which is a setup/protocol error.

| Case | Returned statement | Expected |
| --- | --- | --- |
| A7-VAL-01 | exact `::std::print!("{}/{:08x}/{}", a, b, c);` | `valid` |
| A7-VAL-02 | `print!(...)`, `std::print!(...)`, alias macro | `invalid`, code `printf_macro_path` |
| A7-VAL-03 | `::std::println!(...)`, `::core::print!(...)` | `invalid`, code `printf_macro_path` |
| A7-VAL-04 | function call `print(...)` | `invalid`, code `printf_macro_kind` |
| A7-VAL-05 | braces/brackets delimiter or block wrapper | `invalid`, code `printf_macro_delimiter` or existing group-shape code respectively |
| A7-VAL-06 | changed semantic format; separately a different token spelling with identical decoded value | changed is `invalid`/`printf_format_literal`; semantic-equal spelling is `valid` |
| A7-VAL-07 | reordered/numbered/named fields (`{1}`, `{name}`) | `invalid`, code `printf_format_references` |
| A7-VAL-08 | two or four value arguments | `invalid`, code `printf_argument_count` |
| A7-VAL-09 | `name = expr` macro argument | `invalid`, code `printf_named_argument` |
| A7-VAL-10 | tuple/block/call containing commas as one argument | valid count; never textual comma counting |
| A7-VAL-11 | optional trailing comma after final value | valid and counted identically by validator/replacer |
| A7-VAL-12 | two same-label statements or extra unlabeled sibling | `invalid` with existing deterministic expansion-group code |

### A7-VAL-13 `nested_argument_macro_is_structurally_allowed`

Return `::std::print!("{}", helper!(x));`. Expected structural validation may
succeed and Cargo decides compilation. After accepted build, printf observation
extraction emits nothing for the whole statement because a target argument
contains a nested macro.

### A7-VAL-14 `mechanical_and_rule_payloads_are_restored`

Return altered text for a `mechanical` or `rule_applied` print statement in a
mixed SCC. Expected validator and replacer restore the immutable canonical
statement before further checks; altered macro tokens never reach candidate
source or sidecar.

### A7-VAL-15 `replacer_does_not_trust_validator`

Call `replace_items_with_observations` directly with each invalid transform
template from A7-VAL-02--12. Expected atomic `InvalidTransformation`, no
candidate/sidecar/observation source value, and an item diagnostic naming the
same failed macro invariant. This proves independent defense.

## 8. Expanded target recovery and observations

### A7-OBS-01 `one_argument_print_observation`

Source/accepted target:

```rust
#[proctor(0)] printf(b"%d\0" as *const u8 as *const c_char, x);
#[proctor(0)] ::std::print!("{}", x);
```

Expected `observations == []` and one `printf_observations` member with
`format_specifier == "%d"`, normalized source/target path expressions for the
paired `x`, no anchors when `x: i32`, and source intrinsic/adjusted types both
primitive `i32`. There are no target root type fields.

### A7-OBS-02 `expanded_format_scaffolding_is_not_observed`

For A7-OBS-01 inspect all three sources. Accepted candidate and statement-pair
sidecar still contain plain `x`; only the separate observation source contains
`::std::print!("{}", { x });`. In expanded AST/HIR, expected target expression
is inner user `x`, not the artificial block, `&x`, `ArgumentV1`, a trait shim,
`format_args`, `_print`, an array/tuple, or a compiler-generated match. Assert
the `{ arg }` block prevents literal/simple-argument folding and the inner
expression retains a compiler mapping and its own type.

### A7-OBS-03 `three_pairs_keep_conversion_order`

Source format `%ld/%08x/%.2f` with args `a,b,c`; target print args
`ta,tb,tc`. Expected three printf observations in order with contexts `%ld`,
`%08x`, `%.2f` and pairs `(a,ta)`, `(b,tb)`, `(c,tc)`. No whole-call ordinary
observation is emitted.

### A7-OBS-04 `pointer_anchors_are_complete_and_separate`

Use corresponding functions with source parameter `p: *const i32`, target
parameter `p: &[i32]`, source argument `*p.add(0)`, and target argument `p[0]`
under `%d`. Expected one printf observation rooted at the complete argument,
source root types both primitive `i32`, and this exact anchor type structure:

```json
{
  "id": "<id0>",
  "source_type": {
    "kind": "raw_pointer",
    "mutability": "const",
    "pointee": { "kind": "primitive", "name": "i32" }
  },
  "target_type": {
    "kind": "reference",
    "mutability": "shared",
    "pointee": {
      "kind": "slice",
      "element": { "kind": "primitive", "name": "i32" }
    }
  }
}
```

The source/target expressions use the same binding identity `<id0>`. The exact
grouping key is still specifier plus source root intrinsic/adjusted types;
anchor metadata is not concatenated into that key.

### A7-OBS-05 `two_anchors_keep_source_occurrence_order`

Source argument `compare(*right, *left)` with parameter declaration order
`left,right`. Expected anchors `right,left` by expression occurrence. Repeated
uses of `right` deduplicate by binding identity.

### A7-OBS-06 `source_vararg_types_are_explicit_c_int`

Use this valid source/target pair:

```rust
#[proctor(0)]
printf(b"%hhd\0" as *const u8 as *const c_char, value as c_int);
#[proctor(0)]
::std::print!("{}", value as i8);
```

with `value: i8`. Expected source expression is the explicit cast to `c_int`,
whose normalized intrinsic and adjusted source types are both primitive `i32`
on the fixture target. Target expression normalizes as the explicit `i8` cast,
but no target root type fields are serialized. Do not claim Rust applied a C
default promotion and do not add a promotion validator.

### A7-OBS-07 `different_target_root_types_need_no_context_field`

Use two concrete pairs:

1. `%s`: corresponding source/target function parameters are
   `p: *const c_char` and `p: &str`; source and accepted target arguments are
   the respective `p` bindings, so the root changes from pointer to `&str` and
   the usual anchor retains that target anchor type.
2. `%hhd`: use A7-OBS-06, source root `c_int`, target root `i8`.

Expected: both observations serialize exact source intrinsic/adjusted
`TypeTree`s, source/target expression trees, and any usual pointer anchors;
neither serializes or attempts to infer target root/contextual types. Expanded
formatting wrapper types are absent.

### A7-OBS-07A `unsupported_nominal_source_context_is_ineligible`

As an explicitly negative/out-of-domain fixture, use `%s` with source argument
`p: *const LocalChar` so the Rust C-vararg call type-checks but the source root
`TypeTree` contains an observation-local nominal pointee identity. Expected: no
printf observation for the complete statement. A separate pure exact-context
comparison fixture with a stable external identity compares structurally by
exact crate/path. Never accept two unrelated local identities merely because
each was canonicalized as `<id0>`.

### A7-OBS-08 `nested_macro_is_statement_atomic`

For a three-argument accepted print, put `helper!(b)` only in the middle target
argument. Expected zero printf observations for the label, not observations for
arguments one and three. Other eligible labels in the function still emit.

### A7-OBS-09 `any_mapping_failure_is_statement_atomic`

Inject separately: missing expanded `FormatArgs`, two candidate expansions,
wrong outer callsite, named/captured/reordered format arg, source/target count
mismatch, absent HIR mapping, unsupported normalized expression, incomplete
anchor mapping. Expected zero printf observations for the label. A metadata or
compiler-correspondence contradiction that current policy treats as fatal
remains fatal rather than silently ignored.

### A7-OBS-10 `unconvertible_and_return_used_printf_emit_nothing`

Accepted LLM transformations for `%*d` and `let n = printf(...)` may compile,
but expected no printf observation and no ordinary observation rooted in the
printf call. This enforces exclusive special ownership and scope.

### A7-OBS-11 `mechanical_and_rule_complete_emit_nothing`

Mechanical zero-consuming and rule-applied printf labels are absent from
observation `transform_labels`; extraction is not invoked for an otherwise
complete SCC and emits no members if mixed with another extracted label.

## 9. Closed version-1 wire formats

### A7-WIRE-01 `empty_documents_have_new_exact_bytes`

Expected observation JSON:

```json
{
  "schema_version": 1,
  "observations": [],
  "printf_observations": []
}
```

Expected rule JSON:

```json
{
  "schema_version": 1,
  "rules": [],
  "printf_rules": []
}
```

Both end in one newline and use this member order.

### A7-WIRE-02 `old_version_one_documents_are_rejected`

Remove `printf_observations` or `printf_rules` from otherwise valid version-1
JSON. Expected closed deserialization failure. Adding `serde(default)` or
accepting the old bytes is a test failure. Schema version remains exactly 1.

### A7-WIRE-03 `one_printf_observation_round_trips`

First serialize this anchorless normalized `x -> x` example exactly inside the
document from A7-WIRE-01:

```json
{
  "format_specifier": "%d",
  "source_expression": {
    "kind": "path",
    "value": { "kind": "binding", "id": "<id0>" }
  },
  "target_expression": {
    "kind": "path",
    "value": { "kind": "binding", "id": "<id0>" }
  },
  "pointer_anchors": [],
  "source_type": { "kind": "primitive", "name": "i32" },
  "source_adjusted_type": { "kind": "primitive", "name": "i32" }
}
```

Whitespace follows canonical pretty JSON; object member order above is exact.
Then serialize anchored A7-OBS-04 and assert the usual anchor object and types.
Reparse and reserialize byte-identically. Unknown/missing/duplicate keys,
invalid types, invalid expressions, or bad anchors reject. In particular,
adding `target_type` or `target_adjusted_type` as a printf root field is an
unknown-field error; those names remain valid only inside the usual pointer
anchor object where applicable.

### A7-WIRE-04 `specifier_is_one_supported_consumer`

In an otherwise valid printf observation use `%d` (accept), then `text%d`,
`%d%d`, `%%`, `%*d`, `%q`, or a JSON byte buffer with raw `0xff` inside the
specifier string (reject). The specifier stores the exact one-conversion
substring, not a whole format.

### A7-WIRE-05 `one_printf_rule_round_trips_and_validates_carriers`

Two compatible copies of A7-WIRE-03 produce this anchorless rule body:

```json
{
  "format_specifier": "%d",
  "source_pattern": {
    "kind": "path",
    "value": { "kind": "variable", "sort": "binding", "index": 0 }
  },
  "target_pattern": {
    "kind": "path",
    "value": { "kind": "variable", "sort": "binding", "index": 0 }
  },
  "pointer_anchors": [],
  "source_type": { "kind": "primitive", "name": "i32" },
  "source_adjusted_type": { "kind": "primitive", "name": "i32" }
}
```

Embed it under `printf_rules` with `rules: []`, reparse, and reserialize
byte-identically. Build a second rule synthesized from anchored A7-OBS-04;
expected ordinary anchor variable/type relations. Existing variable sort,
canonical index, carrier, injectivity, target-identity, and type closure errors
reject exactly as for ordinary rules.

### A7-WIRE-06 `merge_keeps_families_and_order`

Merge documents `[ordinary A, printf P1]`, `[ordinary B, printf P2,P3]`, and an
empty document. Expected ordinary array `[A,B]`, printf array `[P1,P2,P3]`,
duplicates preserved, exact input/member order, and canonical bytes. Zero
inputs produces A7-WIRE-01.

### A7-WIRE-07 `markdown_distinguishes_rule_family`

Pretty-print one ordinary and one printf rule. Expected deterministic separate
entries; printf entry includes exact specifier and source intrinsic/adjusted
types and does not pretend the `print!` macro is a normalized expression.

## 10. Family-separated synthesis

### A7-SYN-01 `same_exact_context_anti_unifies`

Two printf observations for `%d`, source root types `i32/i32`, expressions
`*p` -> `p[0]` and `*q` -> `q[0]`, with corresponding anchors. Expected one
canonical printf rule generalized through the existing coupled logic.

### A7-SYN-02 `specifier_mismatch_does_not_pair`

Pair otherwise identical `%d` and `%i`, then `%x` and `%08x`. Expected no
candidate for each pair; exact source specifier includes conversion, flags,
width, precision, and length.

### A7-SYN-03 `intrinsic_or_adjusted_source_type_mismatch_does_not_pair`

Pair observations differing only in intrinsic source type, then only adjusted
source type. Expected no candidate before expression anti-unification. Assert
local type identities are not structurally abstracted to make the exact key
agree; a source context containing an unsupported observation-local nominal
identity is ineligible before pairing rather than compared by anonymized ID.

### A7-SYN-04 `anchors_use_usual_relational_generalization`

With equal printf context, pair observations whose anchor binding IDs differ
but structures/types correspond. Expected normal anchor variable
generalization. Change anchor cardinality, carrier relation, or incompatible
anchor type and expect the same reject behavior as ordinary synthesis. Anchors
do not have to be byte-equal merely to enter the exact context bucket.

### A7-SYN-05 `families_never_cross`

Provide an ordinary observation and printf observation with identical
expression/type bodies. Expected no cross-family candidate. Each family may
synthesize with a second member of its own family.

### A7-SYN-06 `target_reject_propagation_is_unchanged`

Within equal printf context, introduce incompatible target identities or a
rigid mismatch that ordinary coupled anti-unification rejects. Expected no
printf rule; a larger expression variable must not hide `Reject`.

### A7-SYN-06A `target_variables_must_be_source_bound`

Pair observations that would anti-unify a target-only binding, expression,
type-identity, or anchor-type variable absent from the source pattern, exact
source TypeTrees, resolved source identities, and source anchor structure.
Expected: synthesis rejects each rule. A fixed target literal/external identity
is allowed because it introduces no variable. A target variable already bound
by the source expression or source anchor is accepted and reuses that exact
canonical variable; synthesis never invents a value from target context.

### A7-SYN-07 `dedup_and_sort_cover_both_arrays_independently`

Generate duplicates and reverse document/observation order. Expected each
array uses its specified canonical dedup/sort behavior, ordinary rule bytes are
unchanged, and no global ordering interleaves families.

## 11. Matching, ranking, and atomic application

### A7-APPLY-01 `exact_context_selects_bucket`

Apply a `%d`, `i32/i32` rule to matching source arg. Expected match. Change
specifier to `%i`, intrinsic type to `u32`, or adjusted type only; expected
candidate miss without attempting materialization.

### A7-APPLY-02 `ordinary_rules_cannot_fill_printf_slots`

Supply only an ordinary rule whose source pattern would match the arg.
Expected baseline todo remains. Conversely, a printf rule does not apply to an
ordinary selected expression outside eligible printf.

### A7-APPLY-03 `anchor_matching_and_injectivity_are_unchanged`

Use one/two-anchor printf rules and candidate arguments with matching,
repeated, swapped, split-carrier, and aliased bindings. Expected the same
binding/anchor injectivity outcomes as existing ordinary matcher tests.

### A7-APPLY-04 `existing_ranking_selects_within_one_argument`

Provide two applicable printf rules in the same exact bucket. Expected winner
by existing specificity, then distinct substitution cost, target size, and
canonical JSON tie-break. Reverse rule order; result is unchanged.

### A7-APPLY-05 `unmaterializable_winner_allows_ranked_fallback`

Make the first-ranked target unspellable/unparseable and second valid.
Expected second fills that arg. Remove second; expected entire statement rolls
back to baseline todos.

### A7-APPLY-06 `three_argument_commit_is_simultaneous`

Give all three rules targets of different rendered lengths. Expected
installation by original argument identity/order, never offset-sensitive text
replacement. Make any one miss after other two materialize; expected no target
change anywhere in the statement.

### A7-APPLY-07 `implicit_order_cannot_be_changed`

Rules for args one and two may transform expressions, but resulting macro
arguments remain in slots one and two and format contains no `{1}`/`{0}`.
Attempt metadata/rule input that requests reordering; expected closed validation
failure or ordinary miss, never reordered output.

### A7-APPLY-08 `applied_and_mechanical_dispositions_differ`

Zero-consuming conversion is `mechanical` in both views and does not set
`contains_rule_application`. A consuming all-rule result is `rule_applied` only
in applied view and does set it. A partial miss is `transform` in both.

### A7-APPLY-09 `percent_s_pointer_to_str_needs_no_target_root_type`

Synthesize the `%s` rule from two A7-OBS-07 pointer-to-`&str` observations.
Apply it to a third function whose corresponding source/target bindings are
`q: *const c_char` and `q: &str`. Expected source exact context matches the raw
pointer TypeTrees, anchor binding maps `q` and supplies its target anchor type,
target pattern materializes as target `q`, and applied skeleton is
`::std::print!("{}", q);`. No target root/context type lookup occurs. Cargo
build checks that `q: &str` implements the required formatting trait.

### A7-APPLY-10 `percent_hh_promoted_to_narrow_target`

Synthesize the `%hhd` rule from two A7-OBS-06 observations. Apply it to
`small: i8` passed as `small as c_int`. Expected exact source context is
primitive `i32/i32`, source pattern matches the explicit promotion cast,
target pattern materializes `small as i8`, and applied skeleton is
`::std::print!("{}", small as i8);`. The differing target root type is neither
looked up nor stored; Cargo build is the type check.

### A7-APPLY-11 `unbound_target_variable_document_rejects`

Construct a printf rule JSON whose target pattern contains binding variable
index 1 while source pattern/context/anchors bind only index 0. Expected closed
rule validation failure before indexing/application. Repeat for an unbound
expression variable and target-anchor type variable with the same result.

### A7-APPLY-12 `cargo_checks_actual_format_argument_type`

Use an otherwise valid `%d`, `i32/i32` rule whose target pattern is the array
`[x]`, with `x` bound by the source pattern. Expected: rule validation,
matching, and target materialization succeed without inventing a target root
type, producing `::std::print!("{}", [x]);`. The transactional Cargo build then
fails because the array does not implement `Display`; the candidate is rolled
back and the existing rule-involved whole-SCC baseline fallback runs. This is a
build/type failure, not a reason to restore target contextual type inference.

## 12. Python orchestration, fallback, and publication

Implement with `FakeTools`, `FakeClient`, `run_fake`, and exact fake documents
in `proctor/tests/test_local_transformation.py`.

### A7-PY-01 `strict_loader_and_projection_accept_mechanical`

Load a record with one mechanical node and valid report metadata. Expected
`transform_labels == ()`, `contains_rule_application == false`, exact
validation/replacement request projection includes `mechanical`, and prompt
rendering does not list it as a hole. Unknown dispositions still reject.

### A7-PY-02 `mechanical_singleton_uses_no_llm_or_observation`

Fake a singleton containing only converted zero-consuming printf. Expected:
no LLM, validator, or extraction call; replacer and one Cargo build run through
the existing mechanical path; accepted report contains the printf/print pair;
empty merged observation document has both required arrays.

### A7-PY-03 `mechanical_build_failure_is_fatal_without_fallback`

Fail the candidate build for A7-PY-02. Expected rollback and fatal stage
failure, no repair call, no baseline-switch event, and no output artifacts.

### A7-PY-04 `mixed_mechanical_and_transform_scc`

One member is mechanical printf, one needs LLM. Expected one LLM request for
the SCC; canonical mechanical statement remains immutable across validation,
replacement, and repairs; report publishes both relevant accepted pairs; only
the transform member can produce observations.

### A7-PY-05 `printf_rule_complete_scc_is_mechanical_processing`

Applied view contains `rule_applied` printf with all arguments. Expected no
LLM/validator and no extraction, replacement/build occurs, and
`contains_rule_application` is true for fallback classification.

### A7-PY-06 `rule_build_failure_uses_existing_whole_scc_fallback`

In a mixed SCC, fail the first applied build where one printf is rule-applied.
Expected rollback, all SCC members switch once to baseline (including other
ordinary rule applications), one shared repair budget, and no return to applied
views. Mechanical statements are identical across views.

### A7-PY-07 `accepted_transform_print_extracts_after_build_only`

Fake one LLM-generated valid print, successful replacement/build, then a
printf observation document. Expected extraction event after build, retained
document passed opaquely to merge, and publication exactly once. Failed and
superseded attempts contribute nothing.

### A7-PY-08 `python_is_opaque_to_required_arrays`

Fake extraction bytes containing ordinary and printf arrays plus nested
sentinels. Merge fake asserts byte identity and returns different canonical
bytes. Expected Python does not decode/rewrite either family and publishes the
merge output byte-for-byte.

### A7-PY-09 `statistics_include_mechanical`

Accepted final views contain recursively: two preserve, one preserve_shell,
three transform, two rule_applied, and four mechanical nodes. Expected:

```json
"statements": {
  "total": 12,
  "preserve": 2,
  "preserve_shell": 1,
  "rule_applied": 2,
  "transform": 3,
  "mechanical": 4
}
```

Update and assert exact zero/default statistics JSON with `mechanical: 0`.

### A7-PY-10 `statement_pairs_include_mechanical`

Expected Markdown contains a mechanical entry whose Before is the original
`printf(b"hello\0" ...)` and After is canonical
`::std::print!("hello");`. It has normal pointer-variable reporting metadata
when applicable, sorts by item then label with transform pairs, and is not
described as LLM-transformed or rule-applied.

### A7-PY-11 `publication_transaction_stays_atomic`

Exercise stale/symlink destinations and a failure while publishing project,
statement pairs, new-shape observations, and statistics. Expected the existing
one cleanup transaction removes only owned outputs and never publishes an old
two-field observation default.

## 13. Regression and completion matrix

### A7-REG-01 `ordinary_observation_and_rule_corpus_is_stable`

Regenerate current fixtures after adding required empty printf arrays. Expected
ordinary observation/rule member bytes and semantics are unchanged; only the
required enclosing empty array and canonical document bytes change.

### A7-REG-02 `scanf_rigidity_and_foreign_seeds_are_unchanged`

Run existing scan-family, anchorless foreign-call, maximal-region, rigid
literal, synthesis, spelling, and application tests. Expected unchanged
behavior. Supported printf is excluded from the generic foreign seed only at
the new explicit scope boundary; other foreign functions remain eligible.

### A7-REG-03 `non_print_macros_remain_unsupported`

Source/target labels containing `println!`, `write!`, user macros, or a print
macro not paired with eligible printf still follow the existing macro skip and
transform behavior. No ordinary macro expression constructor or observation is
created.

### A7-REG-04 `validator_replacer_preservation_regressions_pass`

Run all current preserve, preserve-shell, rule-applied, nested-control,
temporary, wrapper, call-rewrite, and macro-safety tests. Expected only enum or
complete-key test updates for `mechanical`; no weaker restoration.

### A7-REG-05 `no_public_contract_or_prompt_change`

Expected unchanged `stage.toml`, stage input/output schemas, artifact names,
prompt ID/version/text variables, config keys, CLI subcommands, and output
metrics. The optional input rule file is still read-only and Python-opaque.

Implementation is complete only when:

| Requirement | Cases |
| --- | --- |
| exact conservative conversion | A7-FMT-01--26, A7-FMT-N01--25 |
| statement/identity/literal boundary | A7-ELIG-01--10 |
| ordinary-selector opacity | A7-OPAQUE-01--04 |
| baseline/applied/mechanical views | A7-SKEL-01--07 |
| validator and replacer defense | A7-VAL-01--15 |
| expanded typed extraction | A7-OBS-01--11, including A7-OBS-07A |
| required closed v1 documents | A7-WIRE-01--07 |
| exact-context synthesis | A7-SYN-01--07, including A7-SYN-06A |
| matching and atomic application | A7-APPLY-01--12 |
| orchestration/reporting/fallback | A7-PY-01--11 |
| no semantic drift elsewhere | A7-REG-01--05 |

Run the exact verification commands in Section 15 of the implementation plan.
