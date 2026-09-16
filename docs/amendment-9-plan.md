# Amendment 9 Detailed Plan: proctor-libc 0.3.0 Integration

## 1. Purpose and authority

This amendment integrates the published `proctor-libc` 0.3.0 release into the
local-transformation prototype. It has five connected pieces:

1. require `proctor-libc` 0.3.0 wherever the local pipeline or ordinary Crat
   output can introduce its APIs;
2. broaden the existing deterministic whole-statement `printf` conversion to
   the static formats supported by the new formatting adapters;
3. carry exact per-function consuming format specifiers through skeleton JSON
   so PROCTOR can render concise SCC-specific adapter guidance;
4. expose the new mutable `strto*` variants in the existing libc prompt
   guidance; and
5. extend the ordinary Crat `prepare` pass to replace supported C `ctype.h`
   calls and canonical glibc ctype-table idioms with safe `proctor_libc`
   calls.

This file is self-contained implementation guidance for a fresh session.
Current tests and implementation remain authoritative if they reveal a factual
discrepancy. Amendment 9 is planning-document terminology only; do not put
planning labels in PROCTOR or Crat code, tests, fixtures, diagnostics,
configuration values, commands, or file names.

The exhaustive concrete cases are in
[amendment-9-test-plan.md](amendment-9-test-plan.md). After implementation,
update [prototype-desc.md](prototype-desc.md) through its required documentation
workflow and append the concise historical entry and links to
[prototype-plan.md](prototype-plan.md). Do not revise earlier detailed plans or
their historical claims.

## 2. Research goal and approved decisions

The local-transformation prototype fixes whole-program target types, uses an
LLM only for remaining statement-local work, and learns typed rules from
build-accepted transformations. Every deterministic conversion that removes a
formatting or ctype pointer idiom therefore has two benefits: it reduces unsafe
work immediately and exposes more useful argument expressions for observation
and rule synthesis.

The following decisions are fixed for this amendment:

- deterministic print lowering remains limited to `printf`; do not add
  `fprintf`, `sprintf`, `snprintf`, `vprintf`, wide printing, or another output
  sink;
- support static `d`, `i`, `u`, `o`, `x`, `X`, `f`, `F`, `e`, `E`, `g`, `G`,
  `a`, `A`, and `s` conversions within the precise domain below, including
  applicable flags, static width and precision, floating `L`, and a bare `.`
  precision;
- formatting adapters remain prompt guidance, at the same advisory level as
  existing `proctor_libc` replacements; do not add wrapper enforcement to the
  validator or replacer;
- exact consuming format specifiers are stored at function-record granularity,
  then unioned for each SCC by Python;
- all existing version numbers remain unchanged. In particular, prompt ID
  `local_transformation` remains version `1`, validation/replacement,
  observation, and rule documents remain schema version `1`, and no stage or
  manifest version is bumped; and
- `prepare` handles all fourteen direct ctype APIs, all twelve supported
  `__ctype_b_loc` classes, and canonical `__ctype_tolower_loc` and
  `__ctype_toupper_loc` table lookups. Classification results are deliberately
  normalized to exactly `0` or `1`, including when a source table-mask
  expression would expose glibc's nonzero mask value;
- direct ctype rewrites preserve their argument unchanged; table rewrites
  require terminal `e as isize`, remove exactly that cast, preserve `e`
  unchanged, and add neither a cast nor type analysis; and
- target-manifest dependency failures use one contextual CLI-local `Result`
  path before source output, without extending `PrepareRunError` or
  `PrepareError`.

The semantic boundary assumes the C locale, inputs on which the C ctype APIs
are defined (`EOF` or an `unsigned char` value), and the accepted
zero/nonzero-to-`0`/`1` normalization. The formatting boundary likewise uses
the C locale and the rounding behavior documented by `proctor-libc` 0.3.0.
`%s` output selected by its precision and first NUL must be valid UTF-8, and
`%L` uses the project's `f128::f128` mapping even when a host C `long double`
has another representation. These are explicit prototype domains, not facts
to infer dynamically.

## 3. Current behavior that must remain intact

### 3.1 Local project preparation

`proctor/stages/local-transformation/stage.py` copies the input project to its
work directory and calls `_ensure_required_dependencies` before invoking Crat.
The helper adds or raises crates.io requirements for `bytemuck`, `xj_scanf`,
and `proctor-libc`, preserves unrelated dependency-table fields, rejects an
alias for a required canonical crate name, and leaves git, path, workspace, or
non-crates.io dependencies unchanged. Only the minimum for `proctor-libc`
changes in this amendment, from `0.1.0` to `0.3.0`.

The stage then performs `expand` and `unexpand`, asks `crat-tool` for immutable
baseline and applied skeleton views, normalizes safety, and requires an initial
Cargo build. SCCs start from applied views and fall back once to baseline views
after a rule-involved candidate build failure. Candidate installation,
replacement, build acceptance, observation extraction, and publication remain
transactional and unchanged.

### 3.2 Existing `printf` special case

`proctor/stages/crat/crates/tools/src/printf.rs` owns one total format converter
and exact macro-template validation. A candidate is eligible only when all of
the following remain true:

1. the complete payload of a semicolon statement is a direct call after
   peeling parentheses;
2. the callee resolves to a locally declared foreign item in a non-unwinding C
   ABI block, its effective linked symbol is exactly `printf`, it is variadic,
   it has one fixed `*const c_char` input, and it returns `c_int`;
3. its format is recovered from the existing accepted byte-string-literal cast
   chain, has one terminal NUL and no interior NUL, and its pre-NUL bytes are
   valid UTF-8;
4. the complete format parses successfully; and
5. the number of consuming conversions exactly equals the number of value
   arguments.

The converter is all-or-nothing. An ineligible call remains ordinary LLM work;
no supported prefix is lowered out of a partly unsupported format. Uses of the
`printf` return value, nonliteral formats, other literal-construction forms,
wrong prototypes or ABIs, dependency-owned declarations, and other print
families remain ineligible.

An eligible zero-value call becomes one canonical `::std::print!` statement
with disposition `mechanical`. An argument-bearing call becomes a trusted
format template whose value expressions are `todo!()` in the baseline view.
The applied view installs the complete statement only when one selected printf
rule covers every consuming argument. Existing exact validation of the macro
path, delimiter, semantic string literal, implicit argument references, and
argument count remains unchanged.

Printf-specific observation extraction continues to pair, in order, each
exact source specifier, source vararg, and recovered expanded Rust format
argument. Recovery is statement-atomic. Rule validation, synthesis, grouping,
selection, and application continue to use the exact source specifier and
source types; the ordinary and printf rule families remain separate.

### 3.3 Skeleton and prompt boundary

`FunctionRecord` currently contains per-function resolved foreign function and
static names, but not format information. Crat does not construct SCCs; Python
loads the records, builds the function graph, and schedules SCCs. The new
metadata must therefore be per function even though its consumer unions it per
SCC.

The skeleton JSON is an internal, unversioned, lockstep protocol. Rust derives
its record serialization, while `model.py` deliberately requires each record
to have exactly its known fields. Adding one required field therefore requires
simultaneous Rust producer, Python loader, fake-record, and protocol tests. It
does not change the public stage envelope or the versioned validation,
replacement, observation, or rule shapes.

The prompt already renders foreign-function-specific `proctor_libc` guidance
only when relevant names occur in an SCC. `libc_guidance.py` represents one
immutable or mutable pair as two references and its generic renderer then names
both alternatives. The five `strto*` entries currently have only their shared
slice reference.

### 3.4 Ordinary `prepare`

`proctor/stages/crat/crates/passes/src/preparer.rs` currently performs two
compiler-guided normalizations: block-wrap non-block match arms and lift
function-local statics. It validates unsupported input and every required
AST/HIR mapping before mutation, then returns either one complete source string
or a `PrepareError`. `src/bin/crat.rs` writes source only after successful
compiler execution and preparation.

The local CRAT configuration already selects `prepare` after `enum` and before
`simpl`. This amendment does not change the adapter, configuration, pass name,
or pass order. The ctype work belongs in this existing pass because it is
mechanical preparation for local transformation and must run before skeleton
generation; it does not belong in the Python stage, `crat-tool`, or Crat's
separate `libc` pass.

## 4. Component ownership and implementation surfaces

Keep responsibilities on these existing boundaries:

- `proctor/stages/crat/crates/tools/src/printf.rs` owns format grammar,
  conversion classes, C-to-Rust format-field construction, eligibility, and
  trusted print parsing and validation;
- `proctor/stages/crat/crates/tools/src/skeleton.rs` owns per-function format
  metadata, including the field's canonical sorted serialization, together
  with template and view construction;
- `observation.rs` and `rule.rs` continue to reuse the common converter for
  printf extraction, validation, synthesis, selection, and application;
- `validator.rs`, `preservation.rs`, and `item_replacer.rs` retain independent
  enforcement of trusted print formats and counts without interpreting prompt
  guidance;
- `proctor/stages/local-transformation/model.py` owns strict skeleton-record
  loading;
- a focused Python formatting-guidance module, preferably
  `proctor/stages/local-transformation/printf_guidance.py`, owns fail-closed
  specifier classification and concise prompt text;
- `stage.py` owns dependency preparation, SCC-level metadata union, and prompt
  orchestration; `protocol.py` owns the prompt-render input and version-1
  rendering boundary; and `prompts/local_transformation.md` owns the behavioral
  instruction shown to the LLM;
- `libc_guidance.py` owns the five new mutable `strto*` signatures;
- `proctor/stages/crat/crates/passes/src/preparer.rs` owns syntax-directed ctype
  recognition and rewriting and reports whether generated code requires
  `proctor-libc`;
- `proctor/stages/crat/src/bin/crat.rs` owns project source/dependency commit
  behavior for ordinary passes;
- `proctor/stages/crat/crates/utils/src/lib.rs` may own a reusable
  dependency-ensuring helper if the CLI cannot keep the logic local without
  duplication; and
- `proctor/stages/crat/deps_crate/{Cargo.toml,Cargo.lock}` owns the pinned build
  dependency that lets rustc-private compiler invocations resolve generated
  `proctor_libc` paths.

Primary tests remain module-local under the owning Rust source modules and in
`proctor/tests/test_local_transformation.py`. Crat tests must not invoke the
`crat-tool` CLI, mutate filesystem state, or use a project-root `tests/`
directory.

## 5. Dependency policy

### 5.1 Python local-transformation stage

Change only the `proctor-libc` entry in
`REQUIRED_CRATES_IO_DEPENDENCIES` to requirement `"0.3.0"` and minimum key
`(0, 3, 0, 1)`. Preserve the current dependency algorithm:

- add `proctor-libc = "0.3.0"` when absent;
- replace a crates.io string requirement whose recognized lower bounds do not
  admit at least 0.3.0;
- for a crates.io dependency table, change or add only `version`, retaining
  features, default-feature settings, registry spelling, and other fields;
- retain an already sufficient crates.io requirement;
- retain git, path, workspace, or other non-crates.io dependencies rather than
  silently converting them to crates.io; and
- reject canonical-name aliases and malformed dependency values exactly as
  today.

Do not add a second stage-side ctype dependency path. Every project reaching
the local-transformation stage already receives this dependency preparation
before that stage invokes `crat-tool`; the ordinary Crat CLI needs the separate
conditional path in Section 5.3 for direct `prepare` runs.

### 5.2 Crat build dependency

Update `deps_crate/Cargo.toml` to `proctor-libc = "0.3.0"` and regenerate only
the corresponding `Cargo.lock` resolution and transitive entries. This is a
build/runtime prerequisite for compiler invocations that encounter paths
introduced by `prepare`; it is not permission to add a Rust dependency from
the `passes` or `tools` crate to the `proctor-libc` implementation. Those
components generate and inspect source syntax only.

### 5.3 Ordinary `prepare` output

Change `preparer::prepare` to return a result such as:

```text
PreparationResult {
    code: String,
    requires_proctor_libc: bool,
}
```

Set `requires_proctor_libc` if and only if at least one committed direct or
table-based ctype rewrite introduces a `proctor_libc::...` path. Match-arm
wrapping and local-static lifting alone must leave it false. An unrecognized
ctype near-match also leaves it false.

When the flag is true, the Crat CLI must ensure a canonical target-project
dependency with minimum crates.io version 0.3.0. Its behavior should match the
stage policy above for absent, string, table, alias, and non-registry cases so
running `prepare` directly does not destroy existing dependency configuration.
Implement this preservation-aware, CLI-owned seam as
`ensure_proctor_libc_dependency`. It must not call the current unconditional
`utils::add_dependency`, which can replace an existing table wholesale.

Complete preparation before changing either file. Preserve the current
no-source-write guarantee for `PrepareError`. When the result requires the
crate, successfully ensure the manifest dependency before writing generated
source, so a dependency error cannot leave source that refers to an unavailable
crate. A no-rewrite result must not touch the manifest. Do not build a custom
two-file rollback protocol: if the later source write fails, the command fails
and may leave only a harmless extra dependency.

Have `ensure_proctor_libc_dependency` return a CLI-local `Result` covering its
manifest read, parse, validation, and write failures. Handle that result in the
`Pass::Prepare` CLI branch before invoking the source-write callback. Do not add
a `PrepareRunError` or `PrepareError` variant, and do not use `panic!`,
`unwrap`, or `expect` anywhere on this dependency-ensure path. Report failure
with this stable diagnostic prefix, substituting the displayed target manifest
path and the specific cause:

```text
failed to ensure proctor-libc dependency in <manifest path>: <cause>
```

Then terminate the CLI unsuccessfully without writing generated source. This
ordering does not require source/manifest two-file atomicity.

Do not edit `Cargo.lock` in a transformed target project directly. Subsequent
Cargo/rustc preparation and build steps resolve it through the existing tool
flow. Only Crat's checked-in `deps_crate/Cargo.lock` is regenerated in source
control.

## 6. Expanded static `printf` grammar

### 6.1 Universal parsing rules

Keep `convert_printf_format` total on arbitrary bytes and retain its single
left-to-right, all-or-nothing result. Literal UTF-8 is copied, Rust braces are
escaped, and `%%` emits one literal percent without a consuming conversion.
Every consuming conversion retains its exact decoded substring from `%`
through the conversion byte as `source_specifier`.

Continue to reject:

- incomplete or unknown conversions;
- a non-UTF-8 format;
- positional values, widths, or precisions;
- `*` width or precision;
- the locale-dependent apostrophe flag;
- numeric widths or precisions greater than `i32::MAX` or any checked-decimal
  overflow;
- `c`, `p`, `n`, wide strings, and every conversion or length combination not
  admitted below; and
- flags whose meaning is not admitted for that conversion family.

Add a `space` bit to the parsed flags. The space flag is accepted for signed
integer and floating conversions but is not representable in the Rust format
field; the LLM uses `.space_sign()` on the selected adapter instead.

After `.`, reject `*`; otherwise parse zero or more decimal digits. No digits
means precision zero, so `%.d`, `%.s`, `%.f`, `%.e`, `%.g`, and `%.a` are valid
precision-zero spellings when their conversion is otherwise supported. Keep
the exact bare-dot source specifier while emitting `.0` in the canonical Rust
field.

### 6.2 Canonical format-field construction

Build each Rust field in this fixed order:

```text
{ : [<] [+] [#] [0] [width] [.precision] [trait] }
```

Omit `:` when every option and trait is absent. The spaces above are notation,
not output. Apply these normalization rules:

- emit `<` only when `-` has a width;
- emit `0` only when a width exists and `-` is absent;
- emit `+` for admitted sign-capable conversions whenever the C `+` flag is
  present;
- never emit the C space flag;
- on integer formats, retain an explicit precision; the adapter implements C's
  rule that precision disables zero padding even if the field still carries
  `0`;
- normalize missing `f`, `F`, `e`, `E`, `g`, and `G` precision to `.6`;
- leave `a`, `A`, and `s` precision absent when it is absent in C; and
- retain `#` where it can affect output: every accepted octal/hex integer and
  general-float use, fixed/scientific precision zero, and hex-float precision
  zero or absent. It may be omitted from fixed/scientific fields whose positive
  precision already guarantees a radix point.

This preserves the existing canonical strings for formats already accepted
before this amendment while making every newly accepted semantic distinction
available to the v0.3.0 adapters.

Use this complete family table:

| C conversion | Lengths | Accepted flags | Rust trait suffix | Adapter |
| --- | --- | --- | --- | --- |
| `d`, `i` | empty, `hh`, `h`, `l`, `ll`, `j`, `z`, `t` | `-`, `+`, space, `0` | none | `proctor_libc::printf::signed` |
| `u` | empty, `hh`, `h`, `l`, `ll`, `j`, `z`, `t` | `-`, `0` | none | `proctor_libc::printf::unsigned` |
| `o` | same integer lengths | `-`, `#`, `0` | `o` | `proctor_libc::printf::unsigned` |
| `x` | same integer lengths | `-`, `#`, `0` | `x` | `proctor_libc::printf::unsigned` |
| `X` | same integer lengths | `-`, `#`, `0` | `X` | `proctor_libc::printf::unsigned` |
| `f` | empty, `l`, `L` | `-`, `+`, space, `#`, `0` | none | `proctor_libc::printf::fixed` |
| `F` | empty, `l`, `L` | `-`, `+`, space, `#`, `0` | none | `proctor_libc::printf::fixed_upper` |
| `e` | empty, `l`, `L` | `-`, `+`, space, `#`, `0` | `e` | `proctor_libc::printf::scientific` |
| `E` | empty, `l`, `L` | `-`, `+`, space, `#`, `0` | `E` | `proctor_libc::printf::scientific` |
| `g` | empty, `l`, `L` | `-`, `+`, space, `#`, `0` | none | `proctor_libc::printf::general` |
| `G` | empty, `l`, `L` | `-`, `+`, space, `#`, `0` | none | `proctor_libc::printf::general_upper` |
| `a` | empty, `l`, `L` | `-`, `+`, space, `#`, `0` | `x` | `proctor_libc::printf::hex_float` |
| `A` | empty, `l`, `L` | `-`, `+`, space, `#`, `0` | `X` | `proctor_libc::printf::hex_float` |
| `s` | empty only | `-` | none | `proctor_libc::printf::byte_string` |

Every integer family accepts static precision. Reject `#` on `d`, `i`, or `u`;
reject `+` and space on all unsigned conversions; reject non-string flags on
`s`. Repeated and permuted admitted flags normalize deterministically as the
existing converter does.

The adapter receives the value after the C length conversion. For example,
`%hhd` requires an `i8` value even though the C variadic argument was promoted
to `int`; unsigned families analogously require the corresponding unsigned
primitive. Empty/`l` floating conversions use the appropriate promoted
`f64`, while `L` uses `f128::f128`.

## 7. Skeleton templates and exact format metadata

Extend the internal `PrintfConversionKind` only as needed to distinguish the
new semantic families during conversion and tests. Do not serialize the kind
as a second source of truth. The exact source specifier remains authoritative
for observations, rules, and Python guidance.

Add this required field to `FunctionRecord`, adjacent to the existing foreign
reference context:

```text
printf_format_specifiers: Vec<String>
```

Populate it from every `PrintfConversion.source_specifier` in every eligible
template in the function. Store a lexicographically sorted, duplicate-free
list, using `BTreeSet` or an equivalently deterministic construction. Include
only consuming conversions:

- literal format text and `%%` add no entry;
- an eligible zero-argument mechanical print has an empty list;
- repeated identical consuming specifiers add one entry;
- exact spellings that normalize to the same Rust field remain distinct; and
- an ineligible or atomically unsupported `printf` contributes no entry.

The field describes the baseline source function, not one selected view. An
argument-bearing eligible print is baseline `transform`; it may become
`rule_applied` in the applied view. Consequently guidance can include a wrapper
whose only current occurrence is immutable under an applied rule when another
part of the SCC still needs the LLM. This bounded over-inclusion is accepted to
keep the metadata parallel to foreign-name context and to guarantee complete
guidance after baseline fallback. A fully mechanical or rule-complete SCC makes
no LLM request, so any computed guidance is not rendered.

Do not add format fields to `SkeletonView` or `PrintfTemplateMetadata`, and do
not forward them in validation or replacement requests. Retain
`PrintfTemplateMetadata { rust_format, argument_count }` and all existing
cross-view, preservation, validation, and restoration invariants.

Expanded converter support automatically expands eligible skeleton templates,
printf observation extraction, version-1 printf observation/rule validation,
and rule application because those paths reuse `convert_printf_format` and
`eligible_printf_statement`. Preserve that single parser as the source of
truth; do not create family-specific acceptance rules in those consumers.

## 8. Strict Python loading and SCC guidance

### 8.1 Internal record loading

Add `printf_format_specifiers: tuple[str, ...] = ()` to Python's `ItemRecord`,
but require the JSON key explicitly for every function record. Load it with a
strict helper parallel to `_foreign_names`:

- the value must be an array;
- every member must be a nonempty string;
- members must be strictly increasing, rejecting duplicates and out-of-order
  values; and
- value/type errors identify the record and field deterministically.

Nonfunction record shapes do not acquire the field. Update every fake/function
record producer rather than weakening the exact-object check or silently
defaulting missing wire data.

### 8.2 Fail-closed advisory classification

Python must not implement a second C format grammar. It receives only exact
specifiers that Crat has already accepted. Its classifier performs the minimum
needed for prompt selection:

1. require an ASCII string beginning with `%` and ending in one of the approved
   consuming conversion bytes;
2. map only the terminal conversion byte to the adapter family in the table in
   Section 6;
3. scan the contiguous flag prefix immediately after `%` and record whether it
   contains the space flag; and
4. raise `SkeletonError` for an impossible or unknown emitted value instead of
   omitting potentially necessary guidance.

Do not revalidate width, precision, length, flag order, or complete C syntax in
Python. Duplicated grammar could drift from Crat; malformed internal output is
a protocol failure, not an unsupported source program.

For each SCC, union `printf_format_specifiers` from all member records, derive
the required adapter families and whether `.space_sign()` is mentioned, and
deduplicate guidance in a fixed catalog order. This calculation is independent
of member order and does not change graph edges, dependency context, target
rendering, view fallback, or repair budgets.

Thread the same SCC guidance through every prompt-rendering path: the initial
request, validation/build repair requests, and the whole-SCC baseline fallback.
Because metadata describes baseline functions rather than views, the union is
unchanged across fallback; do not infer formats from mutable rendered code.

### 8.3 Prompt content

Add the `printf_guidance` prompt variable to `PromptRenderInput`,
`render_prompt`, and the version-1 template. Render no formatting section when
the SCC has no selected adapter family. Otherwise state concisely that:

- the target skeleton's Rust format string, static width, precision, trait
  choice, and number of argument slots are trusted and must not change;
- each consuming value should use the listed fully qualified
  `proctor_libc::printf` adapter for its source conversion;
- preserve the source order of consuming values: fill the existing argument
  slots in order and do not swap slots;
- no item should be imported or defined for these calls;
- the value passed to an integer or floating adapter is the value after its C
  length conversion;
- `.space_sign()` is used when an applicable source specifier contains the
  space flag, with Rust `+` retaining precedence; and
- `byte_string` accepts an `&[i8]`, stops at the first NUL subject to precision,
  counts width and precision in bytes, and requires the selected bytes to be
  valid UTF-8.

Include only the signatures and conversion mapping for adapter families used
by that SCC:

- `signed` for `d`/`i`;
- `unsigned` for `u`/`o`/`x`/`X`;
- `fixed` and `fixed_upper` independently for `f` and `F`;
- `scientific` once for either `e` or `E`, with the target field selecting the
  Rust `e`/`E` trait;
- `general` and `general_upper` independently for `g` and `G`;
- `hex_float` once for either `a` or `A`, with the target field selecting the
  Rust `x`/`X` trait; and
- `byte_string` for `s`.

Mention `.space_sign()` only if at least one selected signed/floating specifier
contains the space flag. The C `+` flag overrides a simultaneous space flag;
calling `.space_sign()` remains correct because the Rust formatter sees `+`.
Do not show unrelated adapters merely because they exist in `proctor-libc`.

The slot-order instruction is advisory. The validator continues to allow any
structurally valid Rust expression as a trusted print argument and enforces the
trusted format and argument count, not semantic correspondence to source
argument order; Cargo build remains the acceptance gate. This is the same
advisory replacement model as the existing libc guidance; do not add
format-wrapper-shaped validation or change observation boundaries.

## 9. Mutable `strto*` prompt references

For each existing `LibcGuidance` entry for `strtod`, `strtof`, `strtold`,
`strtol`, and `strtoul`, add the corresponding second reference:

```rust
pub fn strtod_mut(buf: &mut [i8])
    -> ((f64, &mut [i8]), Result<(), StrtoFloatError>);
pub fn strtof_mut(buf: &mut [i8])
    -> ((f32, &mut [i8]), Result<(), StrtoFloatError>);
pub fn strtold_mut(buf: &mut [i8])
    -> ((f128::f128, &mut [i8]), Result<(), StrtoFloatError>);
pub fn strtol_mut(buf: &mut [i8], base: i32)
    -> ((i64, &mut [i8]), Result<(), StrtoIntError>);
pub fn strtoul_mut(buf: &mut [i8], base: i32)
    -> ((u64, &mut [i8]), Result<(), StrtoIntError>);
```

Describe each as returning the mutable unconsumed suffix with the same value
and status semantics as the immutable variant. The existing generic renderer
will then offer `proctor_libc::<name>` or `proctor_libc::<name>_mut`, as
appropriate. Tell the LLM to choose according to the transformed input and the
required mutability of the returned suffix; do not prefer a mutable variant
when a shared suffix is sufficient.

Do not add ctype APIs to LLM libc guidance. The prepare pass removes supported
ctype work before local skeleton generation, and unsupported near-matches are
intentionally left as ordinary source rather than presented as an encouraged
LLM replacement.

## 10. Lightweight ctype preparation

### 10.1 Direct function calls

Follow the established syntax-directed style in
`libc_replacer::TransformVisitor::visit_expr`; do not add DefId, foreign-item,
link-name, ABI, or signature analysis for these rewrites. Recognize an
`ExprKind::Call` only when its callee is an unqualified
`ExprKind::Path(None, path)` with exactly one segment textually named one of:

```text
isalnum  isalpha  isblank  iscntrl  isdigit  isgraph  islower
isprint  ispunct  isspace  isupper  isxdigit  tolower  toupper
```

Require exactly one call argument; wrong arity is a nonmatch, never a panic.
Rewrite the call node to the corresponding fully qualified
`::proctor_libc::<same_name>(original_argument)` in any expression context.
Preserve the complete argument expression, its evaluation count, and
surrounding casts or operators. Qualified paths, method or indirect calls, and
all other callee shapes remain unchanged. This intentionally has the same
name-based limitations as the existing libc pass; the user approved the
simpler recognizer, and ordinary missed shapes remain available to local LLM
transformation.

The twelve classification functions return exactly `0` or `1`.
`tolower`/`toupper` return the input unchanged outside the corresponding ASCII
case range, including `EOF` and non-ASCII byte values within the defined input
domain. The replacement does not access a ctype table or errno.

### 10.2 `__ctype_b_loc` masks

For `__ctype_b_loc`, recognize the established common C2Rust expression shape
already handled by the libc pass. The broader direct-call set and the two
case-conversion lookup accessors are new prepare coverage that merely reuse the
same lightweight matching style:

1. the selected node is a bitwise `&`;
2. its left operand is an integral cast around one dereference of
   `.offset(e as isize)`;
3. the parenthesized offset receiver is one dereference of a zero-argument call
   to an unqualified one-segment path textually named `__ctype_b_loc`; and
4. its right operand, after recursively peeling cast shells, is an unqualified
   one-segment path with one of these textual names:

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

Replace the exact bit-and node with
`::proctor_libc::<class>(e)`. The replacement is allowed
in any outer context: a zero/nonzero comparison, a cast, a condition, or a bare
numeric use. It deliberately normalizes a set result to `1` rather than
retaining the glibc bit value. The offset index must have the exact terminal
form `e as isize`. Strip exactly that one outer cast and pass `e` unchanged;
this preserves every inner conversion, especially explicit conversions through
`c_uchar` and `c_int`. Do not add another cast or query the type of `e`.

Do not resolve or evaluate the `_IS*` constant, calculate target-endian mask
values, or accept broader equivalent syntax. Bare table loads, dynamic or
combined masks, reversed bit-and operands, `.add`, extra operations or
dereferences, qualified accessor or mask paths, and other structural variants
remain unchanged. In particular, an offset index without terminal `as isize`
is a nonmatch left for local LLM work. Do not partially replace an accessor or
mask in a near-match.

### 10.3 Lowercase and uppercase lookup tables

Recognize the common table-load AST shape: one dereference of
`.offset(e as isize)`, whose parenthesized receiver is one dereference of a
zero-argument call to an unqualified one-segment path textually named
`__ctype_tolower_loc` or `__ctype_toupper_loc`. Replace that exact load node
with `::proctor_libc::tolower(e)` or `::proctor_libc::toupper(e)`. Preserve
surrounding casts, comparisons, and operators. Strip exactly the terminal
`as isize`, preserve `e` unchanged, and add no cast or type analysis.

Qualified accessors, `.add`, extra calls or operations, and all table shapes
outside this common idiom remain unchanged. Do not rewrite the accessor call by
itself and do not add type, signature, or provenance analysis. An otherwise
similar offset index without terminal `as isize` is a nonmatch left for local
LLM work.

### 10.4 Index preservation rule

Direct ctype calls reuse their original argument expression unchanged. For the
two supported table-idiom families, the terminal offset-only `as isize` is part
of the recognition contract: remove exactly that cast and reuse its operand
`e` unchanged as the `proctor_libc` argument. Do not synthesize `as i32`, even
when it would make a broader expression compile, and do not inspect the type of
`e`. Inputs outside this exact common C2Rust shape remain ordinary local-
transformation work.

### 10.5 Analysis, rewrite ordering, and idempotence

Keep ctype recognition inside the prepare AST visitor and use ordered
pattern-matching guards like the existing libc pass. Every guard must be total:
wrong arity or a missing nested node yields a nonmatch rather than `panic!()`.
Replace a complete direct call, bit-and, or table-load node only after its whole
supported shape matches. Walk or replace in an order that cannot partially
rewrite a nested accessor from a rejected larger idiom.

Run the existing match-arm and local-static planning in the same preparation
operation. Existing compiler-map failures still return `PrepareError` before
source output. Generated fully qualified `::proctor_libc` calls do not match
the one-segment source patterns on a second run. Existing block/static
normalization is also idempotent, so a second successful `prepare` produces
structurally equivalent source and does not request a new dependency edit.

## 11. Failure, determinism, compatibility, and version policy

The following behavior is required:

- one unsupported byte, flag, conversion, width, precision, or argument count
  rejects the complete printf special case without changing ordinary
  transformation behavior;
- format parsing uses checked arithmetic and never panics or loops on arbitrary
  bytes;
- for fixed source/function contents, format specifier metadata is exact,
  sorted, duplicate-free, stable across runs, and independent of hash
  iteration. This field-level guarantee does not claim byte-identical complete
  skeleton JSON after source declarations or items are reordered;
- malformed Rust-produced metadata is a fatal Python skeleton protocol error,
  not silently ignored prompt context;
- wrapper guidance order follows one fixed catalog, independent of SCC member
  order, repeated formats, or set iteration;
- ctype candidates are selected by the approved narrow textual names and
  common AST shapes; unrecognized near-matches remain unchanged for local LLM
  work;
- `PrepareError` precedes source and manifest changes, and a CLI-local
  dependency-ensure error uses the required contextual diagnostic and prevents
  the generated source write. The source/manifest pair is not promised to be
  transactionally atomic;
- no dependency is added by `prepare` unless generated source actually
  references `proctor_libc`; and
- all existing source ordering, local-static placement/renaming, match-arm
  semantics, labels, skeleton topology, applied-to-baseline fallback,
  observation atomicity, rule ordering, build transactions, and artifacts are
  preserved.

Keep prompt version `1`. The template has not been released as a stable master
version, and exact rendered content continues to be identified by the SHA-256
recorded on every LLM usage attempt. Keep the prompt frontmatter, explicit
`PromptLibrary.get(..., version=1)`, request metadata, stage-output
`PromptUse`, and related assertions at version `1`.

Keep every existing schema version at `1`. The new required function record
field is an internal lockstep producer/consumer change on an unversioned
skeleton format. Observation and rule objects gain no fields; they accept more
exact specifier values through the expanded shared converter. Older valid
version-1 documents remain valid. No stage-envelope, stage-manifest,
configuration, artifact, statistics, or `proctor.toml` shape changes.

## 12. Explicit non-goals

Do not include any of the following:

- deterministic lowering for print functions other than `printf`;
- support for a used `printf` return value, output errors, flushing
  differences, or a general Rust macro transformation;
- dynamic or positional printf arguments, the apostrophe flag, `c`, `p`, `n`,
  wide strings, nonliteral-format recovery, or data-flow recovery of formats;
- arbitrary-byte `%s` output, active non-C locales, host rounding-mode tracking,
  or host-specific `long double` emulation beyond the published wrapper;
- validator enforcement that an LLM selected a particular adapter;
- serialization of adapter choice or parsed format fields alongside the exact
  source specifier;
- a Python reimplementation of the C format parser;
- observation/rule schema changes or merging of ordinary and printf rule
  families;
- moving ctype logic into Crat's `libc` pass or adding that pass to the local
  configuration;
- rewriting arbitrary ctype-table arithmetic, mask combinations, bare table
  values, noncanonical accessors, or locale-changing calls;
- adding ctype functions to LLM prompt guidance;
- changing existing adapter pass planning or pipeline configuration; or
- modifying `proctor-libc` itself.

## 13. Implementation sequence

1. Update Crat's checked-in dependency crate to `proctor-libc` 0.3.0 and
   regenerate its lockfile; update the Python local-stage minimum and its
   focused manifest-preservation tests.
2. Expand the pure Rust format model and converter, retaining exact old
   canonical outputs where applicable. Complete the finite accepted boundary
   matrix A9-FMT-01--25 and rejected matrix A9-FMT-N01--16 owned by the
   companion test plan before changing skeleton consumers; these include bare
   precision, atomic mixed rejection, overflow, and arbitrary-byte no-panic
   cases.
3. Thread the expanded converter through skeleton templates and add the
   required sorted per-function `printf_format_specifiers` field. Update Rust
   field sort/dedup tests and all Rust `FunctionRecord` constructors; do not
   assert byte-identical whole JSON for reordered source.
4. Extend Python's strict record model/loader and every fake record. Add the
   minimal fail-closed classifier, SCC union, deterministic formatting-guidance
   renderer, prompt-render input, and version-1 template section.
5. Add mutable references to all five `strto*` guidance entries and verify the
   shared renderer names immutable and mutable alternatives once, in catalog
   order, only for relevant foreign functions.
6. Exercise the expanded converter through trusted validation, replacement,
   observation extraction, version-1 document validation, synthesis,
   selection, and atomic printf rule application. Do not duplicate format
   acceptance logic in those modules.
7. Extend `preparer` with a result/dependency signal and the lightweight
   one-segment direct ctype call matcher. Add all direct-function positives,
   wrong-arity cases, qualified/indirect negatives, and idempotence tests.
8. Add exact ctype accessor/mask discovery and table-node replacement for the
   twelve classes, then the lowercase/uppercase lookup families. Test
   the required terminal `e as isize`, unchanged reuse of `e`, rejection when
   that cast is absent, representative outer use contexts, 0/1 normalization,
   exact-name mapping, structural rejection, and idempotence.
9. Add target-manifest minimum-version handling to the Crat CLI. Ensure the
   dependency after successful preparation and before source output; test its
   focused helper, stable contextual diagnostic, and no-source-write failure
   behavior without making Crat library tests mutate a project tree.
10. Run focused and full verification, validate the unchanged local pipeline
   configuration, then update the current prototype description and concise
   historical overview through their required documentation workflows.

## 14. Required verification

Run focused Crat tools tests from `proctor/stages/crat`:

```bash
cargo test -p tools printf::tests
cargo test -p tools skeleton::tests
cargo test -p tools validator::tests
cargo test -p tools item_replacer::tests
cargo test -p tools observation::tests
cargo test -p tools rule::tests
cargo test -p tools
```

Run preparation and CLI-focused tests, then the cross-workspace checks:

```bash
cargo test -p passes preparer::tests
cargo test -p passes
cargo test --workspace
cargo fmt
cargo clippy --workspace --all-targets
```

Resolve every Clippy warning. Use a targeted `#[allow(clippy::...)]` only for
`len_without_is_empty`, `too_many_arguments`, or `type_complexity` when
necessary.

Run from `proctor`:

```bash
uv run pytest tests/test_local_transformation.py
uv run pytest
uv run ruff check .
uv run ruff format --check .
uv run mypy proctor
uv run proctor validate -c configs/c2rust_crat_local.toml
```

The default tests must remain deterministic, offline, and API-key-free. Use
the existing in-memory rustc harness for Crat behavior and fake tools/LLM
clients for Python orchestration. A real local-pipeline run is optional when
the pinned rustc, Crat, C2Rust, and Cargo prerequisites are already available;
it does not replace focused tests.

## 15. Completion criteria

The amendment is complete only when:

- every local-transformation working project has a preserved or upgraded
  canonical `proctor-libc` requirement of at least 0.3.0 under the existing
  dependency policy;
- Crat's dependency crate resolves 0.3.0 and ordinary `prepare` adds the same
  minimum only after a ctype rewrite requires it;
- the expanded static format matrix produces the exact canonical Rust fields,
  exact ordered conversion records, and no partial output for unsupported
  formats;
- a bare precision is zero, numeric overflow is rejected, and arbitrary input
  cannot panic the converter;
- eligible zero-argument and argument-bearing printf statements retain all
  mechanical, transform, rule-applied, validation, replacement, observation,
  fallback, and reporting invariants;
- for fixed function contents, every function skeleton record contains an
  exact, deterministically ordered `printf_format_specifiers` array, and Python
  rejects malformed array shape, ordering, member types, or unsupported
  classifier inputs;
- each prompted SCC receives all and only the adapter families implied by its
  function records, plus space-sign guidance only when needed, without a
  prompt-version bump;
- all five `strto*` entries accurately expose both shared and mutable suffix
  APIs;
- direct and canonical table-based ctype idioms rewrite to the exact fully
  qualified safe function, normalize classification to `0`/`1`, retain index
  conversions, and leave all near-matches unchanged;
- preparation's ctype matching remains deliberately syntax-directed,
  deterministic, and structurally idempotent; preparation or dependency errors
  cannot write generated source that lacks its required crate;
- direct ctype arguments remain unchanged, while table rewrites require and
  remove exactly one terminal `as isize`, reuse its operand unchanged, and add
  no cast or type analysis;
- dependency-ensure failures use the CLI-local contextual `Result` path and
  stable diagnostic without a new preparation/run error variant, panic,
  unwrap, or generated source write;
- no public schema, stage contract, artifact, config, or nonlocal prototype
  behavior changes; and
- every case in the companion test plan, all focused/full checks, formatting,
  linting, config validation, and current-documentation updates pass.
