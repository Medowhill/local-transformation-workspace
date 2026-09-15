# Amendment 8 Detailed Plan: Local-Transformation Preparation

## 1. Purpose and authority

This amendment adds an ordinary Crat pass named `prepare` for two syntactic and
scope normalizations required before PROCTOR's local-transformation stage:

1. give every non-block match-arm body an explicit block; and
2. lift every function-local static into the module containing its function,
   renaming it and its compiler-resolved uses when crate-wide value-name or
   destination-prelude collisions require that.

The pass runs in the CRAT adapter portion of the local pipeline. It belongs to
the `crat` pass CLI and the `passes` crate, not to `crat-tool` or the `tools`
crate.

This file is self-contained implementation guidance for a fresh session.
Current tests and implementation remain authoritative if they reveal a factual
discrepancy. Amendment 8 is planning-document terminology only; do not put
planning labels in PROCTOR or Crat code, tests, fixtures, diagnostics,
configuration values, commands, or file names.

The exhaustive concrete cases are in
[amendment-8-test-plan.md](amendment-8-test-plan.md). After implementation,
update [prototype-desc.md](prototype-desc.md) through its required documentation
workflow. Do not revise historical plan files or historical sections.

## 2. Research context, ownership, and non-goals

The research prototype fixes whole-program types before translating local
function bodies. Crat's skeleton generator consequently requires regular
block-shaped match arms and rejects function-local items. C2Rust output can
contain expression-bodied match arms and function-local statics, even though
neither construct needs LLM judgment. This pass removes those two avoidable
sources of local-transformation failure symbolically, before `simpl` and before
the later `unsafe`, `unexpand`, `split`, and `bin` passes.

Ownership is intentionally narrow:

- `proctor/stages/crat/crates/passes` owns compiler-resolved analysis and AST
  transformation;
- `proctor/stages/crat/src/bin/crat.rs` exposes the ordinary pass;
- `proctor/stages/crat-adapter/main.py` makes the pass selectable without
  changing the adapter's default chain; and
- `proctor/configs/c2rust_crat_local.toml` selects it for the local pipeline.

Out of scope:

- any `crat-tool`, local-transformation protocol, prompt, rule, observation,
  skeleton, or stage-envelope change;
- any `libc` pass option, `libc` rewrite, or addition of `libc` to the local
  configuration;
- moving function-local functions, constants, type aliases, structs, enums,
  unions, traits, impls, modules, foreign modules, or macro definitions;
- hoisting a module declared inside a function;
- rewriting names by spelling without compiler identity;
- finding a minimally sufficient name-conflict set instead of the deliberately
  conservative crate-wide set below; or
- changing match patterns, guards, attributes, ordering, or control semantics.

Direct `async fn` declarations are unsupported, including free, local, impl,
and trait functions. Async blocks remain supported inside non-async function
bodies. Crates using `#[no_implicit_prelude]` or a source-authored/custom
`#[prelude_import]` are also unsupported. These inputs must fail through the
structured preparation-error layer before AST/HIR mapping or rewriting.

Other function-local items remain unsupported by local transformation. A local
module also remains a local item and may still make that stage reject the
function even though `prepare` can normalize match arms and statics in function
bodies declared inside that module.

## 3. Current implementation facts to preserve

### 3.1 Crat pass shape

`proctor/stages/crat/src/bin/crat.rs` defines the Clap/Serde `Pass` enum and
dispatches each selected pass through
`utils::compilation::run_compiler_on_path`. A source-producing pass compiles the
current library, returns printed Rust, and writes it back only after the
compiler callback succeeds.

The simple AST/HIR passes obtain the expanded AST with
`utils::ast::expanded_ast`, call `utils::ast::make_ast_to_hir` before changing
the tree, remove compiler-injected unnecessary items, mutate the AST, and print
it with `rustc_ast_pretty::pprust::crate_to_string_for_macros`. The AST/HIR map
contains global item IDs, local node IDs, and compiler resolutions for mapped
paths, including inline-assembly `sym` operands. The new pass must follow that
pattern and must not reimplement compilation or source discovery.

`proctor/stages/crat/crates/passes/src/lib.rs` exports the current pass modules.
Focused pass tests live in module-local test modules, for example
`enum_replacer/tests.rs`; they compile strings in memory and do not exercise a
CLI or filesystem.

### 3.2 Adapter pass planning

`proctor/stages/crat-adapter/main.py` has one `PLUGINS` table that serves two
roles: it admits explicit `passes` entries, and predecessor links let
`plugin_chain(final_pass)` construct a canonical chain. The current default is
`final_pass = "bin"`. `resolve_pass_plan` passes every selected name to Crat as
one `--pass` invocation. The focused behavior is tested in
`proctor/tests/test_crat_adapter.py`.

Adding an entry to `PLUGINS` does not require putting it on the predecessor path
to `bin`. The existing test that equates the default plan with
`list(PLUGINS)` is true only because the current table happens to be one linear
chain; it must be made explicit when `prepare` becomes a selectable side branch.

### 3.3 Existing local-transformation restriction

`proctor/stages/crat/crates/tools/src/skeleton.rs` currently emits
`GenerationErrorKind::NonBlockMatchArm` when a match-arm body is not
`ExprKind::Block`, and separately rejects every function-local item. This
amendment does not relax either check. The configured upstream `prepare` pass
makes the intended local-pipeline input satisfy the match-arm check and removes
only local `static` items; the checks remain defensive for direct tool use and
for all other local items.

## 4. Match-arm normalization

Traverse the complete expanded AST recursively. For every match arm whose body
exists and is not already `rustc_ast::ExprKind::Block`, replace only its body
with a newly constructed ordinary block expression whose:

- statement list is empty;
- tail expression is the complete original arm body;
- block check mode is the ordinary/default mode; and
- span is derived from the original body so diagnostics remain useful.

In source terms:

```rust
pattern if guard => expression,
```

becomes structurally equivalent to:

```rust
pattern if guard => { expression },
```

The pretty-printer controls optional comma and whitespace spelling. Tests must
compare structure or normalized output rather than require an optional comma
after a block arm.

An existing block expression is unchanged as a tree, including ordinary,
`unsafe`, `const`, or other compiler-supported block modes represented by
`ExprKind::Block`. Preserve the arm's pattern, guard, attributes, ID, span, and
order. Preserve attributes and parentheses belonging to the old body on the
new tail expression; do not migrate them to the wrapper. Recurse so nested
matches in scrutinees, guards, arm bodies, closures, async bodies, and all other
expressions are normalized as well.

The transformation is syntax-only. It must not add semicolons, turn the old
expression into a statement, discard its value, or change divergence. Running
`prepare` again finds only block-bodied arms and makes no further structural
change.

## 5. Which statics are lifted and where

### 5.1 Lexical ownership

A static is function-local when its item statement is lexically owned by a
supported non-async function or method body rather than directly by a module.
Handle such statics recursively in:

- free-function bodies;
- provided trait-method and impl-method bodies;
- closures and async blocks inside those bodies; and
- nested local-function bodies.

Reject a direct async free function, local function, impl method, or trait
function before invoking the shared AST/HIR mapper. This restriction does not
apply to an async block nested in a supported non-async body.

Lift each static to the nearest lexical module containing the innermost
function that owns it. This is the same module as the function, never its impl,
trait, block, or function body. For example, a method's local static is emitted
beside the containing `impl`, not as an associated item.

A module declared inside a function is nevertheless a module ownership
boundary. A static directly contained by that module is already module-owned
and is not lifted. A static local to a function declared inside that module is
lifted into that module. The pass never hoists or otherwise removes the local
module itself.

### 5.2 Deterministic placement

Discover liftable statics in recursive expanded-source order: module item
order, then function/body statement and expression order, match-arm order, and
nested-item order. For each nearest containing module, insert a lifted static
immediately before the direct module child that lexically contains its owning
function. That direct child may be a function, impl, trait, or a local module.
When several statics share an insertion point, retain discovery order. Remove
the corresponding `StmtKind::Item` from its original block.

Rust item order does not determine static initialization. This placement keeps
the transformed definition close to its former owner while giving one stable
output independent of hash-map iteration. Preserve the complete item—type,
initializer, mutability, visibility, safety, attributes other than the specific
export adjustment in Section 8, and source span.

## 6. Crate-wide value-name accounting

### 6.1 Reserved binder domain

Before mutation, collect the textual spelling of every distinct
compiler-resolved value-namespace binder introduced anywhere in the crate.
Comparison deliberately ignores module, lexical-scope, and syntax-context
separation: a same-spelled binder anywhere is a collision for this pass.

The set includes:

- value-namespace module children, including named imports, glob-introduced
  bindings, reexports, unit/tuple struct constructors, and unit/tuple enum
  variant constructors;
- value bindings introduced by function- or block-scoped named, aliased, and
  glob imports;
- source-defined and foreign value items, including functions, constants, and
  statics, whether module- or block-owned;
- every HIR pattern binding, including parameters, `let`, `if let`, `while
  let`, `for`, and match arms; and
- const generic parameters.

Count distinct binding sites/identities, not reference occurrences. Count one
binder only once when HIR represents the same logical binding more than once,
such as corresponding alternatives of an or-pattern. An import binding counts
as a binding distinct from its target definition even though both resolve to
the same value.

Exclude field names, associated member names, type-only names, lifetime names,
labels, and macro-only names. In particular, a named-field selector or method
name does not force an otherwise unnecessary static rename. A type definition
counts only when it also introduces a value constructor. A struct-like enum
variant does not introduce a ValueNS constructor and therefore does not reserve
its spelling; only its fields and type-side variant identity exist in the
excluded namespaces.

For each textual name, retain its crate-owned binder multiplicity as well as
the global occupied-name set. Each liftable static contributes its own original
binder. Section 6.2 adds a separate, destination-specific collision source for
implicit-prelude ValueNS names. Considering only the crate-owned set here:

- one uniquely named local static has no crate-owned collision;
- two same-named local statics both require new names; and
- a local static matching any other included binder requires a new name even
  when the two could not conflict in the destination module.

### 6.2 Fresh-enough allocation

Separately compute the implicit-prelude ValueNS names from rustc's normal
compiler-injected standard prelude, not from a hard-coded list. The result must
therefore follow the crate's edition and `no_std` selection of `std` or `core`.
All supported destination modules use this compiler-injected prelude.

Reject any AST attribute named `no_implicit_prelude`. Also reject any expanded
AST item with a non-dummy source span and an attribute named `prelude_import`;
this distinguishes a source-authored/custom prelude import from rustc's
dummy-spanned injected standard-prelude item. Perform both checks before the
shared AST/HIR mapper. The ordinary dummy-spanned compiler injection is not an
unsupported source construct and remains the source of reservation names.

Prelude names remain separate from crate-owned binder multiplicity, but they
participate in both collision detection and allocation at the destination. A
liftable static is therefore colliding when either (a) another crate-owned
binder has its original spelling, or (b) that original spelling is an active
implicit-prelude ValueNS name at its destination module. Condition (b) renames
the static even when its own definition is the only crate-owned binder with
that name.

The same standard-prelude set reserves every generated candidate: a candidate
`name_N` is unavailable when that spelling is active through the implicit
prelude. The compiler-injected `#[prelude_import]` belongs only to this
separate set even though it is visible in the expanded source. An ordinary
source-written glob `use` is instead a crate-owned import binder under Section
6.1; a source-authored item marked `#[prelude_import]` is rejected as described
above.

Allocate replacements only for colliding liftable statics, in the discovery
order from Section 5. For original semantic name `name`, try:

```text
name_0, name_1, name_2, ...
```

and choose the first candidate absent from the complete original crate-owned
occupied set, all names allocated earlier by this pass, and the active
implicit-prelude ValueNS reservation set of this static's destination module.
Suffix search restarts at zero for each static; it increases by exactly one and
must never wrap. An already suffixed name follows the same literal rule, so
`name_0` tries `name_0_0` first. Use the identifier's semantic spelling,
without a raw identifier prefix, and construct a valid AST identifier from the
selected symbol.

Do not rename a static absent both kinds of collision, and do not rename any
existing conflicting binder or prelude item. Do not use a general random/fresh-
name generator or Crat's unrelated wrapper/temp prefixes. Allocation must not
depend on hash iteration.

## 7. Compiler-resolved dependency and use handling

### 7.1 Preflight dependency rule

Moving a static changes the lexical scope of its type and initializer. Before
mutating anything, inspect compiler-resolved paths throughout both for every
liftable static, including paths in nested constant expressions, array lengths,
qualified paths, and inline-assembly operands.

Accept dependencies on:

- definitions already available from the destination module under the used
  path; and
- any other static included in the same complete lifting plan, because its
  bound uses are rewritten and it will also become module-owned.

Reject the complete pass if a type or initializer relies on any definition or
binding outside the static's own moved subtree that is introduced by a
function or block scope and is not another lifted static. This includes local
imports (named, aliased, or glob), constants, functions, type aliases, nominal
types, modules, and other local items. Definitions inside the static's own
type/initializer subtree move with that static and do not cause rejection.

Local-import dependence must be recognized as a binding/scope fact, not lost
by looking only at the imported target's final `DefId`: use HIR path-segment
resolution plus lexical item ancestry so a path reached through a block-scoped
named or glob import is rejected even when its ultimate definition is
module-owned. When identical aliases to the identical target are nested, report
the nearest lexically active binding's definition rather than an outer binding
encountered earlier in source order. Do not silently qualify such paths or lift
a dependency closure.

The compiler has already rejected illegal captures such as a static using an
ordinary runtime local or an outer generic parameter. The pass must still use
the same general resolved preflight and must not infer safety merely from a
name spelling.

### 7.2 Identity-based rewriting

Build the complete `LocalDefId -> new Symbol` rename plan before mutation.
Rewrite the declaration identifier of each renamed lifted static and every
mapped AST path whose compiler resolution is exactly
`Res::Def(DefKind::Static { .. }, that_static_def_id)`. Cover every path-bearing
context represented by the AST/HIR mapper, including ordinary expressions and
inline-assembly `sym` static operands. Change only the path segment naming that
resolved definition; preserve qualification and all other segments.

Never rewrite by textual equality. Same-spelled locals, items, imports,
constructors, fields, methods, labels, and unrelated statics remain unchanged.
This rule also rewrites a moved static's reference to another moved static
when the latter was renamed.

Every declaration, reference, scoped dependency, destination module, and
insertion anchor needed by the transform must have an unambiguous compiler/AST
mapping. A missing mapping is a fatal preparation error, not permission to make
a textual guess.

## 8. Export attributes

At the output boundary of `prepare`, renaming must preserve an existing linked
symbol but must never create a new export for a non-exported static.

- A static with `#[export_name = "symbol"]` keeps that attribute and exact
  symbol whether or not its Rust identifier changes.
- An unrenamed static with `#[no_mangle]` keeps `no_mangle` unchanged.
- A renamed static with `#[no_mangle]` loses `no_mangle` and receives exactly
  one semantically equivalent `#[export_name = "original_name"]`, where
  `original_name` is the original identifier's semantic spelling.
- A static with neither attribute gains neither attribute, even when renamed.

Preserve all unrelated attributes and their order. Replace `no_mangle` at its
attribute position rather than moving unrelated metadata. Detect attributes
semantically with rustc's attribute APIs and construct the string-valued
`export_name` AST attribute safely rather than concatenating source text.
Inputs carrying an invalid or contradictory export-attribute combination are
already rejected by compilation before the transformation callback; do not
invent recovery semantics for an uncompilable crate.

This guarantee is deliberately scoped to the `prepare` pass output. In the
checked-in local pipeline, the later `unsafe` pass receives its existing
`--unsafe-remove-no-mangle` adapter default: it removes `no_mangle` from
statics, while it does not remove `export_name`. Consequently, an unrenamed
`no_mangle` static can lose that export later in the configured pipeline,
whereas an explicit `export_name` and the `export_name` created for a renamed
`no_mangle` static survive that particular cleanup. This amendment does not
change `unsafe`, its adapter flags, or the local configuration to strengthen an
end-to-end export guarantee.

## 9. Analysis, errors, and atomicity

Structure the pass as analysis/plan construction followed by one AST rewrite.
The analysis must finish all of the following before removing, inserting,
renaming, or wrapping anything:

1. expanded-AST rejection of unsupported direct async functions and prelude
   configurations, before the shared mapper;
2. AST/HIR mapping;
3. local-static discovery and destination/insertion planning;
4. crate-wide binder accounting, implicit-prelude collision
   accounting, and deterministic name allocation;
5. type/initializer dependency validation; and
6. declaration and bound-use mapping validation.

Expose the pass as a result-returning operation, such as
`preparer::prepare(tcx) -> Result<String, PrepareError>`, with stable error
categories for unsupported direct async functions, unsupported prelude
configuration, rejected scoped dependency, and missing/ambiguous compiler
mapping. A scoped-dependency diagnostic must identify both the local static
and the actual scoped binding or qualified path segment on which it depends. A
mapping diagnostic must identify the involved static or source construct.
Select the first error in deterministic source/reference order; do not let a
hash map choose it.

Add prepare-specific CLI error handling; no general existing fatal-pass
convention provides it. Keep the two result layers distinct. First,
`run_compiler_on_path` returns its outer compiler result, whose existing
failure behavior remains unchanged. Only an outer success exposes the inner
`Result<String, PrepareError>` returned by `preparer::prepare`. For
`Ok(Err(error))`, print the concise diagnostic `prepare failed: {error}`, exit
nonzero, and do not write transformed source. For `Ok(Ok(source))`, write the
source exactly once. A preparation error may not yield partially wrapped match
arms or partially lifted statics on disk.

Successful output must compile. Since later selected Crat passes compile their
input again, invalid output would normally be caught, but focused tests must
also compile the returned string directly so `prepare` is independently
verified.

## 10. Concrete implementation surfaces

Implement the smallest pass-specific module that keeps analysis and mutation
separate. At minimum inspect and update:

- `proctor/stages/crat/crates/passes/src/preparer.rs` and, following the current
  module-test convention, `preparer/tests.rs`: implement analysis, match
  wrapping, static extraction/insertion, identity rewrites, export handling,
  diagnostics, and focused in-memory tests;
- `proctor/stages/crat/crates/passes/src/lib.rs`: export `preparer`;
- `proctor/stages/crat/src/bin/crat.rs`: add `Prepare` to `Pass` and dispatch it
  to `preparer::prepare`, with no pass-specific CLI option or TOML config;
- `proctor/stages/crat-adapter/main.py`: add `prepare` as a known plugin with
  predecessor `enum` and no default flags, but do not connect `pointer`,
  `simpl`, or any existing successor to it;
- `proctor/tests/test_crat_adapter.py`: make the legacy default chain assertion
  explicit, prove `prepare` is absent from it, and prove explicit selection and
  argument planning accept `prepare`;
- `proctor/configs/c2rust_crat_local.toml`: insert `"prepare"` immediately
  after `"enum"` and immediately before `"simpl"`; and
- `docs/prototype-desc.md`, after implementation: proportionately describe the
  configured preparation and remove the claim that non-block arms and local
  statics necessarily reach skeleton-generation rejection, while retaining
  the defensive direct-tool restrictions and other unsupported local items.

Do not add a `[prepare]` config table, `--prepare-*` flags, a dependency side
effect, a stage manifest field, or a new Python adapter abstraction. Explicit
`pass_args.prepare = []` remains valid through the adapter's generic mechanism;
there are no nonempty prepare arguments.

## 11. Adapter and configuration semantics

Add the adapter entry conceptually as:

```text
"prepare": ("enum", [])
```

This makes all of the following true:

- an explicit `passes = ["prepare"]` is accepted and invokes `--pass prepare`;
- `final_pass = "prepare"` selects the predecessor chain through `enum` and
  then `prepare`; and
- the existing default `final_pass = "bin"` follows its unchanged predecessor
  path and therefore does not include `prepare`.

Do not rewire `pointer` to depend on `prepare`, and do not infer ordering
constraints for explicit pass arrays. Only the checked-in local configuration
opts in. Its exact relevant sequence becomes:

```text
expand, extern, preprocess, enum, prepare, simpl, unsafe, unexpand, split, bin
```

There is no `libc` entry in that configuration and no change to `libc` behavior.

## 12. Determinism, compatibility, and preserved behavior

Determinism is defined by expanded-source traversal, original binder spelling,
compiler-resolved standard-prelude membership, ascending numeric
suffixes, and stable insertion order. Sort only data whose compiler API does
not promise source order; retain source spans/ordinals as the tie-break.
Hash-based sets/maps may answer membership questions but must not choose error
order, names, or emitted item order.

The operation preserves:

- at the `prepare` output boundary, each static's `DefId`-bound uses, type,
  initializer, mutability, and linked symbol, subject to the downstream
  `unsafe` qualification in Section 8;
- all unrelated binding names and path spellings;
- every unrelated existing path that resolves to an implicit-prelude value;
  insertion and renaming must leave its syntax unchanged and it must resolve to
  the same prelude definition when the prepared output is recompiled;
- match semantics and value production;
- existing Crat adapter defaults and pass flags; and
- all PROCTOR schemas, stage contracts, local-transformation records, prompts,
  rules, observations, statistics, and artifacts.

No schema or stage version bump is required. The local configuration's resolved
pass list and run fingerprint intentionally change. Older Crat binaries that do
not recognize `prepare` cannot execute the updated local configuration and must
be rebuilt through the adapter's existing cache/fingerprint mechanism.

## 13. Implementation sequence

1. Add the in-memory pass shell, result/error surface, unsupported-input
   preflight, AST/HIR mapping, and module-local test harness.
2. Implement and test recursive non-block match-arm wrapping independently.
3. Implement local-static discovery, lexical module/insertion anchors, and
   deterministic extraction/insertion without renaming.
4. Implement the exact crate-wide value-binder collector and multiplicities.
5. Add deterministic suffix allocation, compiler-resolved standard-prelude
   reservations, and compiler-identity declaration/use
   rewriting.
6. Add scoped-dependency and complete mapping preflight so all rejection occurs
   before mutation.
7. Add the four export-attribute cases and compile every successful output.
8. Wire `Pass::Prepare`, then add the adapter's selectable side branch and its
   focused planning tests without altering the default chain.
9. Update only the local configuration and validate its exact resolved order.
10. Run all companion-plan cases and broader checks, then update current
    prototype documentation using the required skill workflow.

## 14. Required verification

Crat tests must remain inside crate modules, must not invoke `crat-tool`, must
not change filesystem state, and must not use a project-root `tests/`
directory. Run from `proctor/stages/crat`:

```bash
cargo test -p passes preparer::tests
cargo test -p passes
cargo test --workspace
cargo fmt
cargo clippy --workspace --all-targets
```

Resolve all Clippy warnings. Use a targeted `#[allow(clippy::...)]` only for
`len_without_is_empty`, `too_many_arguments`, or `type_complexity` when
necessary.

Run from `proctor`:

```bash
uv run pytest tests/test_crat_adapter.py
uv run proctor validate -c configs/c2rust_crat_local.toml
uv run pytest
uv run ruff check .
uv run ruff format --check .
uv run mypy proctor
```

No live LLM, network service, `crat-tool` invocation, or project-root test
fixture is needed. A real local-pipeline smoke run is optional when its C2Rust
and Crat toolchain prerequisites are already available; it does not replace
the in-memory pass tests or adapter/config checks.

## 15. Completion criteria

The change is complete only when:

- every non-block match arm recursively receives one ordinary tail-expression
  block and every existing block arm remains structurally unchanged;
- every function-local static in the settled lexical scope is emitted once as
  a direct child of the correct module and removed once from its old block;
- statics absent both crate-owned and destination-prelude collisions retain
  their names, while every colliding moved static gets the first `name_N`
  absent globally and from its destination's active implicit-prelude ValueNS
  names, in deterministic order;
- an original spelling active in the destination's implicit-prelude ValueNS
  triggers renaming even without another crate-owned binder, and unrelated
  prelude-resolved paths retain both syntax and compiler resolution;
- every renamed use is selected by the static's compiler identity and no
  same-spelled unrelated occurrence changes;
- unsupported scoped dependencies and missing mappings reject atomically with
  actionable deterministic diagnostics;
- direct async functions, `no_implicit_prelude`, and source-authored/custom
  prelude imports reject atomically before AST/HIR mapping;
- at the `prepare` output boundary, linked symbols follow the exact
  export-attribute matrix and non-exported statics remain non-exported, without
  changing the later `unsafe` pass's existing treatment of `no_mangle`;
- successful output recompiles and a second preparation is structurally
  idempotent;
- the adapter accepts explicit `prepare` while its default plan is unchanged;
- the local configuration resolves to
  `enum, prepare, simpl` at the relevant boundary and contains no `libc`; and
- every case in the companion test plan, focused/full verification, formatting,
  linting, config validation, and current-documentation update is complete.
