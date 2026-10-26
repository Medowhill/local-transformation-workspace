# Supportedness and Research Intent

This reference preserves the local stage's useful input assumptions and goals
without turning historical plans into current behavior. Paths are relative to
the standalone PROCTOR root. Verify mechanically enforced restrictions in
current code/tests before changing them.

## Distinguish contract from detection

Unsupported input means that the stage is not intended to receive the feature;
it does not imply that every component rejects it. A parser or in-memory test
may handle an excluded construct without widening the end-to-end input
contract. There is no general supportedness checker. Skeleton, validation,
source assembly, preparation, and Cargo each enforce narrower boundaries.
Output-protocol errors such as malformed labels or missing functions are
distinct from source-input restrictions.

Upstream preparation and fixture selection remain responsible for satisfying
the conceptual contract. Crat's ordinary `prepare` pass now normalizes selected
non-block match arms and lifts local statics. Defensive skeleton rejection
still applies if those forms remain; the local stage only runs `expand`/`unexpand`
itself. Do not equate 'supported by one helper' with 'supported for translation'.

## Project and target form

The intended input is one Linux, Rust-2021, already compilable Cargo project
with one logical PROCTOR target and one transformed library crate. Its
`Cargo.toml` must explicitly declare `[lib].path` as one relative regular file
directly under the crate root, optionally with `./` components. Absolute paths,
any `..` component, and nested library paths are rejected. External module
files may exist on entry; `expand`/`unexpand` consolidates the library before
skeleton generation. No final `split`/`bin` pass runs in this stage.

An executable uses Crat's library-plus-explicit-forwarding-bin layout. The `bin`
shim calls the library's safe `main` and is not a transformation target. An
executable-only project, extra executable bodies needing translation,
multiple logical targets/configurations, or another Cargo layout is outside
the current model. Libraries use the pipeline's `cdylib` contract and export
functions rather than global variables.

`proctor.toml` must contain `target_kind`, `target_name`, `api_functions`, and
`wrappers`, and `wrappers` must be empty. The stage checks this before work and
preserves the manifest. It does not interpret nonempty prior wrapper
relationships or promise source ABI preservation.

The initial target template retains complete non-function crate context and
typed target-signature stubs for non-`main` free functions. That context must
type-check against the target interfaces. Imports/reexports of retained
functions are not inherently unsupported; references requiring omitted `main`
or old signatures can still make the initial library build fail. There is no
deferred repair for initial-template compilation.

## Functions, declarations, and dispatch

Source-defined safe and `unsafe` free functions, including nested inline-module
functions, are targets except for every final identifier named `main` (including
raw `r#main`). Target functions become `unsafe`; explicit source ABI, visibility,
and an individual export attribute remain real project metadata.

The intended input excludes:

- `const fn`, `async fn`, body-bearing variadic functions, and non-identifier
  parameter patterns;
- methods, explicit `impl`/`trait` declarations and their associated items, trait objects,
  dynamic dispatch, and delegation syntax;
- source-written type, `const`, or lifetime generic declarations, generic bounds,
  and `where` clauses, even though concrete uses of generic types and
  analysis-generated target lifetimes are permitted;
- function-pointer values/types/indirect calls and callbacks, including `void*`
  callback encodings;
- a function carrying both `no_mangle` and `export_name`; and
- remaining function-local items of any kind at any nested depth, including
  constants, statics, functions, types, use/foreign declarations, modules,
  macros, traits, and impls.

Foreign declarations, including variadic libc functions, are allowed context
and advisory metadata rather than local transformation targets. Ordinary
preserved `derive`-generated `impl` blocks are allowed only when `rustc` identifies all
their generated items as automatically derived. Named types, constants,
statics, and inline modules that remain unchanged are ordinary context.

Opaque closures are supported within their enclosing expression region. Their
parameters, local bindings, nested statements, and control are not separately
labeled or assigned target-local types; outer captures still obey surrounding
binding and temporary rules. Restrictions on local items and other excluded
features continue inside closures. The ordinary preparation boundary permits
`async` blocks inside otherwise admitted functions, while excluding `async`
functions, disabled implicit preludes, and source-authored custom preludes.

## Statements, patterns, and source syntax

Source `ref`/`ref mut` binding modes remain conceptually excluded because pointer
analysis does not support them, even though skeleton/validator code preserves
their distinction. Outside parameters and those modes, existing local pattern
structure is supported and preserved, including destructuring, `let ... else`,
`if let`/`while let`, `loop`/`match` bindings, and reference/or/`@` patterns.

Outside opaque closure bodies, input excludes empty statements, non-block
match-arm bodies that survive preparation, source statement/expression
attributes, pre-existing `#[proctor(...)]` labels, and explicit nested `unsafe` blocks.
Unsafe operations may occur directly inside `unsafe` functions; managed
executable `main` wrappers are the special explicit-`unsafe` boundary.

Control expressions normally need a direct structural role as a statement,
`let` initializer, `return`/`break` value, or admitted match-arm result. Controls
buried under calls, operators, casts, assignments, indexing, constructors,
or another non-control wrapper are excluded unless they have the narrow
C-conditional-like form: ordinary `if` with `else`, no control/`let` in the condition,
exactly one tail expression per branch, and any nested branch control another
conditional of this same form. This admitted expression remains opaque within
its surrounding labeled region. Ordinary control nesting through branches,
blocks, arms, loops, and `let ... else` otherwise remains supported.

Source macro definitions, item-producing macros, source-selection `cfg`/`cfg_attr`
alternatives, and expansions that invalidate surface-AST/HIR correspondence
are excluded. Expression/statement macro invocations can remain only when
their source token input contains no call expression; internally expanded
calls do not by themselves violate this source restriction. Observation-only
rewriting cannot reliably redirect an old-signature call hidden in macro
tokens. Supported `derive`/`repr` and other ordinary item attributes are retained.

Required synthesized type names must be visible from the target module.
Existing type syntax is reused/shortened by resolved identity when possible.
Introduced bare `Option`/`Box` require the ordinary prelude and standard language
items; shadowing, unavailable paths, or inaccessible types can therefore make
a required target type unspellable. Skeleton generation fails atomically.

## Pointer, compatibility, and graph limits

Non-local recovery/repair is assumed to have removed true negative offsets,
simultaneous mutable aliases, missing unique ownership, and cases requiring
new base/cursor state, statement reordering, parameter-count changes, or
coordinated redesign beyond the fixed skeleton. A still-unrecovered data
structure needing unit replacement belongs upstream. Local tools skip offset-
sign analysis and never select `SliceCursor`; this is an assumption about the
input, not evidence that negative offsets are absent.

Changing pointer-containing named types or global types/representations is
outside current scope. Unchanged structs/enums/unions/aliases and globals
remain valid dependency context. There are no global wrappers or automatic
library API adapters.

Finalization restores excluded `main` after all SCCs. Supported C2Rust
executable entrypoints use one co-located safe `main` and `unsafe` `main_0` pair:
`main_0` either has no arguments or C `argc`/raw `argv` arguments. The two-argument
target forces `argv` to `&mut [&mut [i8]]`; finalization replaces its sibling `main`
with the trusted argument-slice forwarding boundary, including NUL-terminated
buffers and the final empty-slice sentinel. Zero-argument `main_0` leaves `main`
unchanged. Recognition uses names/arity rather than proving the source wrapper
shape. Other `main` bodies are restored unchanged and must compile; another
caller needing migration or incompatible `argv` behavior is outside the model.

API entries are checked against original Rust names/export names. Accepted
functions retain original paths and export attributes with transformed
signatures; an exported slice/box can therefore have a different C ABI. A
successful final build proves neither C-ABI compatibility nor behavior.

Direct and mutual recursion are admitted. Two members of one SCC sharing a
final Rust name are not, because one validation request identifies functions
by final name. Mandatory dependency context above 100,000 rendered characters
aborts; no direct dependency or recursive member signature may be silently
dropped. Deeper type-context depths may be omitted only as whole breadth-first
levels. Repair cannot modify functions outside the current SCC.

## Research goals and deferred work

The broader PROCTOR research goal is to combine symbolic correctness and cost
with neural idiomaticity, using non-local recovery/repair before constrained
local translation. Local analysis fixes global type decisions; successful
aligned typed regions yield reusable context-sensitive rewrites that should
reduce LLM work across functions and programs. This motivates preserving
identity, type context, labels, and acceptance provenance rather than learning
from arbitrary compilable text.

The implemented loop includes rule application, preservation, constrained
generation/repair, compilation, typed observations, and offline synthesis.
It does not run a test package inside this stage or establish semantic
equivalence, fully safe output, or a declining-cost guarantee. Wider pipeline
test stages and the framework's optional global test gate are separate.

Still-intended extensions include semantic/test validation, richer non-local
repair and supportedness checking/normalization, named-type/global changes,
broader Rust input, deliberate API-preserving compatibility boundaries, and
more complete rule context/application. The general design's retained library
wrappers and stage-produced rule sets are goals rather than current local-stage
contracts. Do not revive historical serial wrapper migration, claim closures
or rules are wholly absent, or assume the manifest is ignored.
