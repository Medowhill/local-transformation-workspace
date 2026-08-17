# Amendment 6 Detailed Plan: Maximal Regions and Foreign-Call Seeds

## 1. Purpose and authority

This amendment widens the local-transformation prototype in two related ways:

1. expression-region selection keeps every inclusion-maximal selected subtree
   instead of abandoning a statement when two selected subtrees overlap; and
2. a supported direct foreign-function call is a region-selection seed even
   when its expression contains no raw-pointer anchor.

It also makes scan-family format literals rigid during coupled rule synthesis,
because a rule that generalizes a format string can compile while changing the
meaning of the conversion. The existing normalized observation and rule ASTs
already represent Rust string, byte-string, and C-string literals; this work
changes selection, validation, synthesis, and materialization policy without
adding a new wire constructor. For prompt guidance, it also retains each
foreign function's Rust declaration name while adding a distinct supported
local C linked symbol to the same existing metadata list.

This file is self-contained implementation guidance for a fresh session. The
current implementation and tests remain authoritative if they expose a factual
discrepancy. Amendment 6 is planning-document terminology only and must not be
used in Crat or PROCTOR code, tests, fixtures, diagnostics, configuration,
commands, or file names.

The exhaustive acceptance cases are in
[amendment-6-test-plan.md](amendment-6-test-plan.md). After implementation,
update [prototype-desc.md](prototype-desc.md) through its required documentation
workflow; do not rewrite historical plan sections.

## 2. Settled decisions

The following policy is final and requires no further design choice.

- Selected expression regions are AST subtrees in one alignment unit. Two
  distinct selected roots therefore cannot cross: they are disjoint or one is
  a strict ancestor of the other.
- Coalesce selection to **all inclusion-maximal roots**, not to one globally
  largest root. Retain disjoint maximal roots.
- Merge identical roots as before. For a strict ancestor/descendant pair,
  discard the descendant root and retain the ancestor.
- Transfer every pointer anchor from discarded descendants to the retained
  ancestor. Deduplicate by resolved source binding and serialize anchors in
  first source-occurrence order.
- Compute `lhs` from the final retained root. Do not inherit a discarded
  descendant's `lhs` value.
- `promoted_field` is root-local. Identical-root merges may combine that flag,
  but a retained ancestor must not inherit field-promotion status from a
  discarded descendant whose root differs.
- The retained roots are pairwise disjoint. Rule application remains atomic per
  statement: every retained root must be covered and materialized before any
  replacement is installed.
- A maximal parent region intentionally supersedes possible child-region rule
  opportunities. There is no nested rewrite composition or child-rule fallback
  inside a selected maximal parent.
- Every supported foreign function is eligible; there is no allowlist.
- A supported foreign seed is a direct call whose callee resolves to a function
  item declared in a locally owned foreign module with exact non-unwinding C
  ABI (`ExternAbi::C { unwind: false }`, conventionally an
  `extern "C" { ... }` block).
- Exclude source-defined `extern "C" fn` definitions, `C-unwind`, `system` and
  other ABIs, calls through function pointers or callbacks, unresolved calls,
  and foreign declarations owned by dependency crates such as `libc::free`.
- A foreign seed starts at the complete call expression and uses the existing
  upward `select_region` parent-edge policy. Thus `*strchr(s, 'a')` selects the
  dereference after growing from the call.
- Observations and rules may have `pointer_anchors: []`. Their document schema
  remains version 1; the JSON shape and canonical member order do not change.
- Existing target-context requirements remain. In particular, applying a rule
  to a pointer-like root still requires the current contextual target adjusted
  type. Foreign-call seeding does not invent a new target-type producer.
- Existing one-source-statement/one-target-expression alignment and statement
  atomicity remain. A macro anywhere in the selected labeled source statement
  or target group still skips observation extraction; macro calls produced by
  an LLM therefore do not produce rules.
- For scan-family calls, only the format argument is literal-rigid:
  `scanf` argument 0, `fscanf` argument 1, and `sscanf` argument 1.
- On the corresponding target side, the rigid format positions are
  `xj_scanf::legacy::scanf` argument 0,
  `xj_scanf::legacy::brscanf` argument 1, and
  `xj_scanf::legacy::bscanf` argument 1.
- The scan-family identity is compiler-resolved. Source calls use the supported
  foreign linked symbol (`scanf`, `fscanf`, or `sscanf`), not textual Rust
  spelling. Target calls use the resolved external crate/path identity, not a
  suffix-only string comparison.
- Prompt-facing `foreign_function_names` retains every compiler-resolved Rust
  declaration name. For a supported locally declared exact-C-ABI foreign
  function with an explicit distinct `#[link_name]`, also include that linked
  symbol in the same sorted, deduplicated list. Thus
  `#[link_name = "scanf"] fn rust_scanf(...)` contributes both `rust_scanf`
  and `scanf`, so the existing PROCTOR exact-name predicate activates the
  xj-scanf guidance without a new record field or prompt variable.
- Every `String`, `ByteString`, or `CString` literal below the designated
  format argument is rigid. Corresponding literals must have the same
  normalized kind and semantic value across the two observations being
  synthesized. A mismatch rejects that observation pair.
- Source and target expressions are compared separately during coupled
  anti-unification. A source `b"%d\0"` and target `"%d"` are valid in one
  observation; each is compared only with the corresponding side of another
  observation.
- Outside those resolved format-argument subtrees, preserve current synthesis
  behavior: unequal string-like literals may be abstracted as expression
  variables. Integer-magnitude generalization is unchanged.
- `Reject` from a rigid literal mismatch must propagate through all enclosing
  expressions and blocks. It may not be weakened into `Generalize` and hidden
  by a larger expression metavariable.
- No schema-version bump is made. New version-1 documents are intentionally
  rejected by older binaries that still require a nonempty anchor list.

## 3. Current implementation facts to preserve

### 3.1 Ownership and files

The implementation belongs to Crat's `tools` library under
`proctor/stages/crat/crates/tools/src/`:

- `observation.rs` owns the closed observation AST, compiler normalization,
  expression-tree construction, shared region selection, source/target
  alignment, and observation extraction.
- `rule.rs` owns closed observation/rule validation, coupled
  anti-unification, canonicalization, matching, ranking, substitution, and the
  selection result used by the materializer.
- `skeleton.rs` owns source-side application inputs, target-context inference,
  scope-aware rendering, expression parsing, simultaneous installation, and
  applied-view construction.
- `rule/markdown.rs` owns human-readable rule rendering.
- `lib.rs` exposes the library operations used by the thin CLI.
- `proctor/tests/test_local_transformation.py` owns the required orchestration
  regressions for SCC scheduling, applied-to-baseline fallback, accepted-
  observation publication, and Python's opaque handling of anchorless
  version-1 documents. Those tests use the existing temporary/fake-project and
  fake-tool behavior; they do not invoke a real Crat CLI or network service.

The thin `crat-tool` command and the Python local-transformation stage do not
need a protocol change. They pass observation and rule documents opaquely.

### 3.2 Existing region selector

`ExpressionTree::add` builds one explicit parent tree, elides parentheses, and
records raw-pointer local path occurrences in source preorder. The shared
`select_expression_regions` function currently:

1. expands each eligible pointer occurrence with `select_region`;
2. calls `merge_regions` for identical roots;
3. computes `lhs`; and
4. returns `None` when `regions_overlap` finds strict ancestry.

Observation extraction calls this selector before source-guided alignment.
Application calls it through `select_rule_regions`. The shared selector must
remain the single source of truth so extraction and application cannot drift.

### 3.3 Existing normalized identities and literals

`DumpContext::definition_identity` currently recognizes a foreign item only
when its locally owned parent foreign module has exact C ABI. It normalizes the
identity to `ValueIdentity::ForeignFunction { symbol }`, where `symbol` is
`#[link_name]` when present and otherwise the Rust declaration name.

`Literal` and `RuleLiteral` already contain:

```text
String { value: String }
ByteString { value: Vec<u8> }
CString { value: Vec<u8> }
```

String raw/escape syntax is normalized to the decoded Unicode value. Byte
strings retain every explicit byte, including an explicit trailing zero.
Rust C-string literals include an implicit terminal zero in the compiler
literal; `DumpContext::literal_with_type` deliberately removes exactly that
compiler-supplied terminator and stores the semantic payload. For example:

```text
"%d"      -> String { value: "%d" }
b"%d\0"  -> ByteString { value: [37, 100, 0] }
c"%d"     -> CString { value: [37, 100] }
```

Do not change these wire meanings.

### 3.4 Existing anti-unification and the block defect

`literal_expression` keeps equal noninteger literals concrete and currently
turns unequal noninteger literal expressions into `E` variables. The internal
`Walk` outcomes distinguish `Ok`, `Generalize`, `IdentityConflict`, and
`Reject`. `expression_child` already propagates `Reject` rather than hiding it.

The expression-statement branch in `block` is the exception: its current
`let Walk::Ok(...) = expression(...) else { return Walk::Generalize; }`
collapses `Reject` into `Generalize`. Correct this branch so `Reject` returns
`Reject`, while the pre-existing generalizable outcomes retain their current
meaning. Audit all other composite/list/optional paths and add regression
tests; do not broadly redefine `Walk` semantics.

### 3.5 Existing materialization defect for foreign symbols

Normalized foreign identity is the linked C symbol, but
`skeleton::value_spelling` currently emits that symbol as a bare Rust path.
That is wrong when a declaration uses `#[link_name]`, is nested in a module, or
is reached through an import alias. Matching must remain semantic by linked
symbol, while rendering must use an accessible Rust spelling.

### 3.6 Existing prompt-metadata alias gap

`skeleton::ForeignFunctionVisitor` currently builds advisory prompt metadata
with compiler foreign-item Rust declaration names. It has a deliberately
broader ownership boundary than the observation predicate and does not use
`#[link_name]`. Consequently,
`#[link_name = "scanf"] fn rust_scanf(...)` normalizes to scan symbol `scanf`
for observations but the current prompt lists `rust_scanf`, so the prompt's
special scan guidance is not activated by that declaration.

Close this gap without changing the `FunctionRecord` shape. The visitor must
always retain the resolved Rust declaration name. When the same shared helper
used by seed discovery identifies a locally owned exact-C-ABI foreign function
with an explicit distinct `#[link_name]`, insert that linked symbol as a second
entry. Keep the existing `BTreeSet` ordering and deduplication, so an absent or
identical link name still yields one entry. Do not add linked-symbol aliases for
excluded ABIs or dependency-owned declarations; their existing Rust-name
metadata remains unchanged.

PROCTOR's parser, `_uses_xj_scanf_guidance` exact-name predicate, prompt
template, and prompt version remain unchanged. For the example above, the
rendered function context lists both `rust_scanf` and `scanf`, and the existing
predicate sees `scanf` and enables the guidance. This is an intentional prompt-
content change for affected local `link_name` references, not a new protocol
field or a replacement of source-facing Rust spelling.

## 4. Region-selection design

### 4.1 One deterministic seed stream

Generalize the internal anchor-only occurrence stream into one source-preorder
seed stream. A seed is one of:

- a paired eligible raw-pointer local path occurrence, carrying its source and
  target binding identities; or
- a supported direct foreign call, carrying the call root and resolved foreign
  definition/linked symbol needed internally.

Record a foreign seed when visiting the `ExprKind::Call` node, before visiting
its callee and arguments. Record a pointer seed when visiting its path node, as
today. Preserve an internal preorder ordinal for stable coalescing and anchor
ordering. The ordinal is not serialized.

Foreign definition eligibility must share one `DefId`-level helper with
foreign identity normalization and prompt metadata. The helper must require
all of:

1. the definition has `DefKind::Fn`;
2. `tcx.is_foreign_item(def_id)` is true;
3. the foreign item's parent is local and is a foreign module; and
4. the foreign module ABI is exactly non-unwinding C.

It returns the linked symbol (`#[link_name]` when present, otherwise the Rust
declaration name). Seed discovery adds the expression-level requirements: the
expression is a direct `ExprKind::Call`, its callee is a path whose type-check
result resolves to that function definition, and the helper accepts the
definition. Prompt metadata may invoke the helper for either call or
function-value references and therefore must not require a call expression.

Do not infer eligibility from the callee's printed name, from prompt metadata,
or merely from an `extern "C" fn` function type.

### 4.2 Expansion and seed-local failure

Run every seed through the existing `select_region` state machine starting at:

- the pointer path node for a pointer seed; or
- the complete call node for a foreign seed.

A seed rejected by expansion is discarded independently; it does not suppress
disjoint valid seeds. A foreign call at a normal call-argument boundary
finishes at the call. A foreign call immediately under dereference or address
formation grows under the existing rules. No parent-edge decision changes in
this amendment.

### 4.3 Inclusion-maximal coalescing

Replace `merge_regions` plus the fatal `regions_overlap` check with one
deterministic coalescing operation:

1. merge candidate regions with identical root AST identity;
2. identify every candidate whose root has another candidate root as a strict
   ancestor;
3. discard those nonmaximal candidates;
4. transfer their anchor occurrences to the nearest retained ancestor (the
   unique maximal selected ancestor in a tree);
5. deduplicate anchors by resolved source binding, choosing the first source
   occurrence and retaining its paired target binding; and
6. order retained regions by their root's source preorder and anchors within a
   region by their occurrence preorder.

An inconsistent target binding for two occurrences of one source binding is a
compiler/correspondence invariant failure handled by the existing higher-level
correspondence checks; do not silently pick conflicting data.

The result must be pairwise disjoint by construction. Keep a debug assertion
or focused internal check for this postcondition rather than retaining
statement-level overlap rejection as ordinary behavior.

### 4.4 Root-local metadata

After coalescing:

- recompute `lhs` exactly as the current selector does, from the retained root
  and its transparent-parent assignment relationship;
- retain `promoted_field` only if the retained root itself was selected by a
  seed whose expansion promoted to that field root; and
- for identical roots, `promoted_field` is true if any identical-root
  candidate selected that same root by field promotion.

Transferred anchors do not transfer either flag.

### 4.5 Extraction and application consequences

Source-guided alignment treats each retained maximal root as opaque and maps
it to the complete target expression in the same structural role. It does not
attempt to align discarded child roots separately. Observation dumping emits
one observation per retained root, including all transferred anchors.

Rule application receives the same retained roots. It selects and materializes
one rule per root and installs the pairwise-disjoint replacements
simultaneously. If any maximal root is uncovered or unmaterializable, discard
all tentative replacements for that statement. Do not try child rules as a
fallback.

## 5. Anchorless observation and rule documents

Remove only the nonempty-list requirements in `validate_observation` and
`validate_rule`. Preserve all conditional validation:

- when anchors exist, IDs/variables are distinct and canonical;
- observation anchors refer to source bindings, occur in source occurrence
  order, have raw-pointer source types, and retain valid target types;
- rule anchors use the `anchor` sort, are canonical, and have valid raw-pointer
  source types; and
- carrier and injectivity checks continue to use the actual anchor set, which
  may now be empty.

Do not weaken the requirement that a target local binding/function identity be
available from the source expression. An anchorless foreign-call observation
may still contain ordinary binding identities such as the `x` in
`scanf(..., &mut x as *mut i32)`; those remain ordinary `binding` variables in
a synthesized rule, not anchor variables.

Canonical serialization remains:

```json
"pointer_anchors": []
```

in the existing member position. Empty-anchor rules participate in matching,
specificity, substitution cost, target-size ranking, canonical tie-breaking,
Markdown rendering, deduplication, and sorting exactly like other rules. Do
not special-case them to lower priority or skip them.

## 6. Foreign-call observations

### 6.1 Complete `scanf` example

The recorded successful transformation at:

```text
proctor/runs/B01_synthetic_034_cast_to_char_ptr_int-20260817T083010-f56d46/
  stages/02-local_transformation/out/artifacts/llm-exchanges/
  scc-2/generation-00/
```

changes:

```rust
scanf(
    b"%d\0" as *const u8 as *const i8,
    &mut x as *mut i32,
)
```

to:

```rust
xj_scanf::legacy::scanf("%d", &mut [&mut x])
```

After this amendment, extraction emits one observation for the complete call,
even though `x: i32` is not a raw-pointer anchor:

- `pointer_anchors` is `[]`;
- `lhs` is `false`;
- source intrinsic/adjusted and target intrinsic/adjusted root types are all
  primitive `i32`;
- the source callee is `ForeignFunction { symbol: "scanf" }`;
- the first source argument contains `ByteString([37, 100, 0])` below two
  const-pointer casts;
- the target callee is the resolved external identity
  `xj_scanf::legacy::scanf`;
- the first target argument is `String("%d")`; and
- source and target occurrences of `x` share one ordinary anonymized binding
  ID, normally `<id0>`.

The test plan gives the exact normalized JSON.

### 6.2 Call plus pointer anchor

If a supported foreign call contains an eligible raw-pointer binding argument,
the call seed selects the call while the pointer seed may select a descendant.
Maximal coalescing retains one call-derived region and transfers the binding's
anchor metadata to it. This is one observation/rule region, not one foreign
region plus one anchor region.

### 6.3 Pointer-valued foreign calls

For:

```rust
*strchr(s, b'a' as i32)
```

start from the complete `strchr(...)` call and apply the existing dereference
edge, selecting the complete dereference. If `s` is an eligible raw-pointer
binding, transfer its nested anchor to the dereference region. If it is not an
eligible binding, the region is anchorless. During rule application, the
pointer-like source root still requires the existing contextual target
adjusted type; a discarded-value statement supplies none and therefore does
not become spuriously applicable.

## 7. Scan-family format-literal rigidity

### 7.1 Recognition

Before coupled expression traversal for an observation pair, derive protected
literal locations independently for source and target normalized expressions.
Recognition is exact:

| Side | Resolved callee identity | Protected argument |
| --- | --- | ---: |
| source | foreign linked symbol `scanf` | 0 |
| source | foreign linked symbol `fscanf` | 1 |
| source | foreign linked symbol `sscanf` | 1 |
| target | external crate `xj_scanf`, path `legacy::scanf` | 0 |
| target | external crate `xj_scanf`, path `legacy::brscanf` | 1 |
| target | external crate `xj_scanf`, path `legacy::bscanf` | 1 |

Require the call to have the listed argument. A malformed/short call is simply
not recognized as a protected position; valid Rust type checking normally
precludes it. Do not recognize local functions or dependency functions merely
named `scanf`, suffix matches, methods, aliases without matching resolved
identity, or other `xj_scanf` functions.

The selected region may contain the scan call below an enclosing dereference,
cast, address operation, block, or other supported normalized expression.
Walk the complete normalized region and mark string-like literal nodes below
the designated argument of every recognized scan-family call. Nested
recognized calls each protect their own format argument. Do not protect
`fscanf`'s stream argument, `sscanf`'s input argument, scan targets, or
unrelated literals elsewhere in the region.

### 7.2 Pair semantics

At a corresponding literal pair, use the rigid rule when that position is a
protected format position in both observations on the side currently being
synthesized. Equal normalized literal kind and value returns the existing
concrete node. Any mismatch involving `String`, `ByteString`, or `CString`
returns `Walk::Reject` immediately.

Do not compare a source format literal with a target format literal. Source
traversal compares the two source observations; target traversal compares the
two target observations. Target traversal remains lookup-only. A protected
target mismatch therefore rejects instead of reusing a source `E` disagreement.

If only one of the paired terms marks a location as the corresponding protected
format position, reject the pair rather than silently falling back to ordinary
generalization. This prevents structurally mismatched scan calls from bypassing
the semantic guard.

Outside jointly corresponding protected positions, call the current
`literal_expression` behavior unchanged. Equal literals remain concrete;
unequal noninteger literals may become an expression variable.

### 7.3 Reject propagation

Preserve `Walk::Reject` through:

- expression children and every expression-list position;
- every optional-expression position, including range endpoints, `if` else,
  struct rest, and return/break values;
- arrays, tuples, calls, method receivers and arguments, binary expressions,
  assignments, assignment operators, unary expressions, casts, fields,
  indices, ranges, address operations, repeats, and struct field values;
- `if` conditions/then blocks/else expressions, `while` conditions/bodies,
  loop bodies, and explicit block expressions;
- expression statements and `let` initializers inside normalized blocks; and
- both source traversal and target lookup traversal.

`path`, `literal`, and `continue` are leaves and have no child-propagation
branch; the protected literal leaf itself returns `Reject`. Before coupled
descent, compare the two protected-location maps for the side being walked. If
only one observation marks a corresponding location as protected, reject that
side immediately, without allocating an expression disagreement. This makes
one-sided scan recognition unable to fall back to a larger `E` because a
callee or wrapper happened to be visited first.

Only `Generalize` and `IdentityConflict` may request an enclosing expression
metavariable where current semantics permit it. A rejected rigid literal must
never allocate an `E` variable at any ancestor. State rollback on a rejected
source walk and lookup-only target behavior remain unchanged.

## 8. Foreign identity and Rust spelling

Keep `ForeignFunction { symbol }` as the version-1 semantic wire identity. The
linked symbol remains the equality/matching key, including `#[link_name]`.
Rendering must not treat that symbol as Rust syntax.

Extend rule-region syntax provenance so each matched concrete foreign **path
occurrence** can retain its compiler-resolved source spelling. Examples:

```rust
ffi::rust_scanf(...)
alias(...)
rust_scanf(...)
```

may all normalize to `ForeignFunction { symbol: "scanf" }`; a target pattern
that reuses a fixed occurrence should render with that occurrence's accessible
matched spelling, not bare `scanf` merely because that is the link symbol.

Do not collapse provenance to one `symbol -> spelling` map. One selected region
can contain two declarations or aliases with the same linked symbol. Retain
source expression ordinal/AST identity and resolved declaration identity beside
each spelling. When a substituted target foreign-path occurrence corresponds
to one unique retained source occurrence or unique equal normalized source
subtree, use that occurrence's spelling. If several source occurrences remain
semantically indistinguishable, use the declaration-resolution fallback below
only when it selects one accessible declaration; otherwise the candidate is a
miss. Provenance affects rendering only, never normalized equality or rule
bytes.

For a fixed target foreign identity without a matched source occurrence, use
the compiler's locally owned exact-C-ABI declarations to find an accessible
Rust value path from the current function's module. Select a unique resolved
declaration with that linked symbol and use the existing scope-aware value-path
machinery. If no accessible declaration is available or the symbol is
ambiguous, the rule candidate is unmaterializable and application tries the
next ranked candidate. Never emit an unverified linked symbol as a bare Rust
identifier.

This provenance is internal application state. Do not add a path field to the
observation/rule schema, and do not change semantic equality or canonical rule
bytes.

## 9. Compatibility and non-goals

### 9.1 Compatibility

- Observation and rule schema versions stay at 1.
- Canonical JSON member names/order and all expression/literal constructors are
  unchanged.
- New readers accept both empty and nonempty anchor lists. Existing nonempty
  documents retain their meaning and canonical bytes.
- Old readers may reject new anchorless version-1 values; this is an accepted
  validation widening rather than a versioned shape change.
- PROCTOR's stage envelope, `rule_set` artifact, saved exchange format,
  statistics, prompt template/version, and Python opaque-document handling do
  not change.
- The `FunctionRecord` JSON shape is unchanged. For a supported local foreign
  reference with an explicit distinct `link_name`, `foreign_function_names`
  gains the linked symbol beside the retained Rust name. Existing Python
  parsing already accepts the sorted extra string. Rendered prompt text and
  saved exchange contents intentionally change for those functions.

### 9.2 Non-goals

- nested rewrite composition or fallback from a maximal region to child rules;
- arbitrary substring/span overlap handling;
- indirect calls, callbacks, methods, or dependency-owned foreign declarations
  as seeds;
- widening supported ABIs;
- a foreign-function allowlist;
- generalized callee-signature/effect constraints;
- general semantic validation of learned rules;
- runtime test execution by the local-transformation stage;
- macro observation/rule synthesis;
- new literal variants or source-spelling preservation for literal tokens;
- globally rigid string-like literals; or
- a schema-version bump.

## 10. Implementation sequence

Implement in this order so each step has a focused invariant.

1. **Consolidate foreign resolution.** Add one tools-private helper that
   classifies a locally declared exact-C-ABI foreign function definition and
   returns its linked symbol. Reuse it in normalized identity dumping, prompt
   metadata, and seed discovery. Seed discovery separately requires the
   expression to be a direct `ExprKind::Call` whose callee resolves to that
   definition.
2. **Augment prompt metadata.** Keep each resolved foreign function's Rust
   declaration name and, when the shared helper reports an explicit distinct
   linked symbol for a supported local exact-C-ABI function, insert that symbol
   into the existing sorted `foreign_function_names` set. Do not change the
   record shape, Python loader, guidance predicate, or prompt template.
3. **Generalize seed collection.** Record pointer occurrences and supported
   foreign calls in deterministic source preorder, retaining internal
   occurrence ordinals.
4. **Implement maximal coalescing.** Replace strict-overlap abandonment with
   identical-root merge, strict-descendant removal, anchor transfer/dedup/order,
   and pairwise-disjoint postcondition. Recompute root-local metadata.
5. **Exercise both consumers.** Update observation extraction and
   `select_rule_regions`/application to consume the same coalesced regions.
6. **Relax anchor validation.** Permit empty lists in observation and rule
   version-1 validators while preserving every nonempty-list invariant.
7. **Audit empty-anchor downstream paths.** Cover synthesis, carrier checks,
   matching, specificity/ranking, canonicalization, Markdown, and application;
   remove accidental indexing assumptions rather than adding dummy anchors.
8. **Add scan-format context.** Compute compiler-normalized source and external
   target protected argument locations, thread that context through coupled
   source/target expression walking, and make string-like mismatches reject.
9. **Fix rejection propagation.** Correct the normalized-block expression
   statement path and verify all other composite paths retain `Reject`.
10. **Fix foreign rendering.** Capture matched foreign Rust syntax and add the
   scope-aware declaration fallback; treat failure/ambiguity as one candidate
   miss.
11. **Add the exhaustive focused tests** in the companion test plan, including
    updated tests whose old expected outcome was overlap abandonment or empty-
    anchor rejection.
12. **Update current documentation** only after code/tests establish behavior.
    Use the `prototype-desc` skill for `docs/prototype-desc.md` and keep this
    historical plan unchanged.

## 11. Required verification

Crat tests must remain in module-local test modules, must not invoke the
`crat-tool` CLI, must not modify filesystem state, and must not use a
project-root `tests/` directory. These restrictions apply to Crat tests, not to
PROCTOR's existing pytest suite: the required orchestration regressions belong
in `proctor/tests/test_local_transformation.py` and may use its established
`tmp_path`, fake project, fake provider, and fake Crat-tool fixtures.

Run from `proctor/stages/crat` after modifying Rust:

```bash
cargo test -p tools observation::tests
cargo test -p tools rule::tests
cargo test -p tools skeleton::tests
cargo test -p tools
cargo test --workspace
cargo fmt
cargo clippy --workspace --all-targets
```

Resolve all Clippy warnings. A targeted `#[allow(clippy::...)]` is permitted
only for `len_without_is_empty`, `too_many_arguments`, or `type_complexity`.

The stage protocol does not change, but the companion test plan requires
orchestration regressions at the Python ownership boundary. Run from `proctor`:

```bash
uv run pytest tests/test_local_transformation.py
```

This focused pytest command is mandatory for Amendment 6. Production Python
should remain unchanged unless a failing regression exposes an existing opaque-
file boundary defect; do not add a Python observation/rule parser.

Then update `docs/prototype-desc.md` to describe maximal region selection,
foreign-call seeds, anchorless documents, and scan-format rigidity as current
behavior. Do not describe the amendment number in implementation-facing prose.

## 12. Completion criteria

The amendment is complete only when all of the following are true:

- selection returns every inclusion-maximal region in stable source order;
- retained regions are pairwise disjoint and carry all deduplicated descendant
  anchors in source order;
- extraction and application use identical selection results;
- supported direct local exact-C-ABI calls seed anchorless regions;
- excluded call categories do not seed;
- the concrete `scanf` transformation emits the specified anchorless
  observation;
- version-1 observation/rule validation accepts empty anchors without weakening
  nonempty validation;
- empty-anchor rules synthesize, round-trip, rank, render, and apply;
- scan-family format literals are rigid only at the specified resolved argument
  positions on both source and corresponding target calls;
- unequal protected string/byte-string/C-string literals reject through every
  enclosing expression/block, while unrelated literals retain current
  generalization;
- foreign calls materialize with valid Rust declaration/alias/module spelling
  rather than a link symbol;
- a supported local foreign `link_name` reference retains its Rust declaration
  name, adds its distinct linked symbol once in canonical order, and activates
  existing xj-scanf guidance for linked `scanf`, `fscanf`, or `sscanf` symbols;
- statement atomicity, pointer-root target context, macro skipping, canonical
  ordering, and all existing pointer-only behavior remain intact; and
- the focused/full Crat tests, mandatory focused PROCTOR pytest, formatting,
  and Clippy checks pass.
