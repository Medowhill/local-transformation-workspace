# Amendment 2 Detailed Plan

This is a historical implementation plan. Its substantive text was moved
verbatim from the former consolidated `prototype-plan.md`; imperative and
future-tense wording describes the work assigned at the time. New navigation
notes identify where later work changed an earlier component.

See the [historical overview](prototype-plan.md#amendment-2).
See the [Amendment 2 test plan](amendment-2-test-plan.md).

## Amendment Plan 2: Conservative preservation of transformation-independent statements

This amendment reduces unnecessary LLM rewriting without weakening the local
transformation boundary. It applies after Phases 1--4 and Amendment Plan 1. It
supersedes only the conflicting skeleton-hole, validation, replacement,
orchestration, prompt, JSON, and deferred-preservation requirements in this
document. In particular, “preservation labels” are no longer deferred. All
historical phase test plans remain unchanged. `amendment-2-test-plan.md` is the
exhaustive executable contract for this amendment.

The optimization is intentionally asymmetric. A false negative merely leaves a
statement as an LLM transformation target. A false positive permanently
discards a necessary LLM rewrite and can silently preserve incorrect code.
Consequently, Crat preserves a statement only after proving all conditions
below. Any missing mapping, unresolved callable, unclassifiable type, or other
uncertainty makes the statement require transformation.

### A2.1 Statement dispositions and recursive scope

Every existing numeric statement label has exactly one *statement
disposition*:

- `preserve`: restore the canonical statement from the target skeleton
  mechanically; or
- `transform`: retain the existing skeleton-hole behavior and accept the
  validated LLM expansion group.

The disposition belongs to the complete `rustc_ast::Stmt` subtree rooted at
that label, including its expression, patterns, types, control expressions,
and recursively nested statements. A statement is preservable only when its
entire subtree is preservable. Therefore, a preserved parent has no transformed
descendant. A transformed parent may contain independently preserved or
transformed descendants.

This recursive rule is deliberately conservative. For example, an `if`
statement with a pointer-free condition still receives disposition `transform`
when one nested branch statement requires transformation. Its ordinary
skeleton control form remains, its condition follows the existing hole rule,
and each nested statement follows its own disposition.

The classification is computed from the mapped, compiling source program and
the complete initial target decisions before skeleton payloads are mutated.
It is not inferred later by searching rendered text for `todo!()`.

### A2.2 Transformation-sensitive types

A type is *transformation-sensitive* when any of these rules applies:

1. The type is a raw pointer.
2. The type itself has different source and planned target representations.
3. The type recursively contains a transformation-sensitive component through
   a reference, array, slice, tuple, callable signature, or generic argument.
4. The type is a project-local struct, enum, or union and any fully
   instantiated field of any variant is transformation-sensitive.
5. A future analysis reports that any project-local field's source and target
   types differ, even if neither type contains a raw pointer.
6. Crat cannot prove that the normalized type is free of
   transformation-sensitive components.

Open project-local nominal ADTs recursively, substitute their actual generic
arguments into field types, inspect every enum variant and union field, and
use a visited set keyed by the instantiated nominal type so recursive types
terminate. Type aliases are transparent. This means that moving or copying a
project-local value whose deeply nested field contains a raw pointer is not
preserved. It also keeps the rule correct when a future pass changes a field
from a `Copy` type to a non-`Copy` type.

Do not recursively expose the private representation of a non-local nominal
type such as `Vec<T>`. Treat the non-local nominal definition as opaque, but
still inspect all of its explicit generic arguments. Thus `Vec<i32>` is not
rejected merely because the standard library implements it with a pointer,
while `Vec<*mut i32>` is transformation-sensitive.

The current prototype changes no named type or field definitions, so the
future field-difference input is initially empty. The classifier must
nevertheless centralize this query rather than equating
“transformation-sensitive” with `Ty::is_raw_ptr()`.

The existing supported-input exclusion of source-written generics remains
unchanged. Generic project-local ADT instantiation is nevertheless required
inside the low-level type-sensitivity helper and receives a focused compiler
unit test. That defensive helper coverage does not make a crate containing a
source-written generic item supported end-to-end by skeleton generation or
local-transformation orchestration.

### A2.3 Conservative preservation proof

Crat assigns `preserve` to a statement only when every condition in this
section succeeds for the complete statement subtree.

#### Mapped nodes, declarations, and patterns

- Every relevant surface AST node maps unambiguously to the expected HIR owner
  or local node.
- Every binding declared or referenced by the subtree has structurally equal
  source and target types.
- Every explicit statement-local type, inferred binding type, and complete
  pattern type is not transformation-sensitive.
- A declaration without an initializer is not vacuously safe: its binding and
  explicit or inferred type are still checked.
- Destructuring `let`, `let-else`, `if let`, `while let`, `for`, and match-arm
  patterns are checked recursively.

#### Expressions, places, and adjustments

- Visit every HIR expression corresponding to the subtree, including both
  lvalue/place and rvalue positions.
- Check the expression's unadjusted type, adjusted type, and every
  intermediate compiler adjustment target type.
- Check callable signatures separately; a `FnDef` or method expression does
  not prove that its parameter and return types are pointer-free merely
  because those types are not ordinary generic arguments of the expression
  type.
- Any inference, projection, opaque, error, or other type that cannot be
  normalized and proven insensitive makes the statement require
  transformation.

#### Calls and callable references

Every explicit or compiler-resolved function, method, overloaded operator, or
other callable operation in the subtree must be statically resolved.
Indirect calls and callable values require transformation under the
prototype's no-function-pointer contract.

A resolved call or callable reference requires transformation when any of
these holds:

- the target is a foreign function declaration;
- the target is non-local and its instantiated signature is unsafe;
- the target is a transformable local function whose source and target
  parameter or return types differ; or
- the callable's instantiated signature is transformation-sensitive.

“Non-local” means that the resolved `DefId` is not defined in the current
crate. This deliberately covers unsafe functions and methods from `std`,
`core`, `alloc`, compiler support crates, and third-party crates. A
pointer-free call to a local unsafe function may be preserved when its
signature is unchanged. Target-safety normalization alone is not a signature
change. A pointer-free call to a safe non-local function may likewise be
preserved.

The changed-local-signature check is required independently of expression
types. It prevents a canonical preserved statement from later overwriting a
temporary wrapper redirection or restoring an obsolete call convention.

#### Conservative syntax boundaries

Every expression- or statement-position macro invocation requires
transformation, even when its visible token tree appears pointer-free.
Expanded HIR calls do not provide a sufficiently reliable statement-local
presentation mapping for this optimization. Closures, inline assembly,
indirect calls, and any other unsupported or ambiguously desugared form also
require transformation.

Pointer-free access to a `static mut` or a union field still requires
transformation. These locally unsafe operations are not preservation
candidates. Raw-pointer operations are independently rejected by the type
rules, and unsafe non-local calls are independently rejected by the callable
rules. A pointer-free call to an unchanged-signature local unsafe function
remains governed by the explicit callable rule above and may be preserved.

### A2.4 Skeleton construction and JSON

For a `preserve` statement, put the canonical target-skeleton statement in
`annotated_skeleton` instead of replacing its payload with `todo!()`. The
canonical statement consists of:

- the original statement's expression, pattern, control, and nested-statement
  payload;
- the target local types already required by skeleton generation;
- the shared presentation binding-mutability normalization; and
- the original `#[proctor(N)]` label tree.

It is not necessarily the literal source AST. For example, an inferred
pointer-free local remains explicitly typed and presentation-mutable in the
target skeleton:

```rust,ignore
#[proctor(0)]
let mut x: i32 = y + z;
```

For a `transform` statement, retain all existing Phase 1 and Amendment Plan 1
skeleton behavior. A transformed parent is skeletonized using the existing
control structure and payload-hole rules, while recursively visited child
statements use their own dispositions.

A restricted conditional accepted by Amendment Plan 1 is preserved with its
complete enclosing statement when that statement subtree passes the Amendment
2 proof. If it does not pass, Amendment Plan 1's opaque payload hole and
single-label boundary remain unchanged.

Add these required fields to every `Fn` skeleton record, immediately after
`target_signature`:

```json
{
  "needs_transformation": true,
  "statements_requiring_transformation": [1, 3]
}
```

`statements_requiring_transformation` contains every `transform` label in
strictly increasing numeric order. Every other label in the annotated
skeleton is a `preserve` label. `needs_transformation` is exactly whether that
array is nonempty. The redundancy is intentional: Python schedules without
parsing Rust, while Crat retains the complete label-level proof.

The amended `Fn` serialization key order is:

```text
id, path, kind, name, annotated_source, annotated_skeleton,
source_signature, target_signature, needs_transformation,
statements_requiring_transformation, signature_dependencies, dependencies
```

Other record kinds are unchanged. Crat generation tests must verify that every
listed label exists, every skeleton label has one implied disposition, no
preserved label has a transformed descendant, and the Boolean equals array
nonemptiness.

### A2.5 Preservation-aware validation

The validation request remains schema version 1 because the prototype
protocol has not been released. Amendment 2 defines the final version-1
request shape before its first release; it does not preserve compatibility
with the earlier unmerged development shape. Each expected function carries
its item ID, name, complete annotated target skeleton,
`needs_transformation`, and `statements_requiring_transformation`.
Validation responses likewise remain schema version 1 with their existing
`valid`, `invalid`, and `setup_error` shapes.

Before applying the existing body validation, Crat constructs a canonicalized
copy of the returned function:

1. Align result expansion groups with expected groups at the current
   statement-list level.
2. For a preserved expected label, require its outer expansion group in the
   expected sibling position, discard the complete returned same-label group,
   and insert the one canonical target-skeleton statement subtree.
3. Treat the discarded group as opaque. Do not inspect its declarations,
   attributes, unsafe blocks, controls, temporaries, or descendant labels.
4. For a transformed expected label, retain its returned expansion group and
   apply the existing Phase 2 expansion-group rules. When the expected
   statement has a control root, identify exactly one returned statement in
   the group as the existing Phase 2 structural anchor: it has the expected
   direct statement role and control kind and owns all matched descendant
   statement lists. Zero or multiple anchors is an error. Same-label siblings
   before or after that anchor remain ordinary expansion statements. Recurse
   only through the unique anchor's reliably matched branch, arm, loop, block,
   or let-else roles so nested preserved groups can be restored.
5. Reject a missing, malformed, reordered, nonconsecutive, or structurally
   misplaced outer label when that label is required to locate a canonical
   replacement.
6. Run the existing signature, declaration, label, control, temporary,
   unsafe-block, and attribute validation against the resulting canonicalized
   function.

Because a preserved parent has no transformed descendant, replacing a
preserved parent restores its complete original label subtree. The LLM may
omit or alter descendant labels inside its discarded parent group. Conversely,
a preserved child beneath a transformed parent must remain locatable in the
same validated branch, arm, loop, block, or let-else role.

Canonicalization must happen before whole-body scans. A temporary declaration
invented inside discarded text therefore cannot satisfy a use in a transformed
group, and an unsafe block that exists only inside discarded text does not
cause a spurious validation error.

Put this canonicalization, unique-anchor selection, structural diagnostics,
and metadata validation in one shared Crat module used by both validator and
replacer. Do not implement two similar label-merging algorithms. The shared
operation returns either the canonicalized function plus its matched
structure or a deterministic structural failure; the validator converts that
failure to its ordinary item diagnostics, while the replacer fails atomically.

### A2.6 Preservation-aware replacement

The replacement request likewise remains schema version 1 and adopts its
final unreleased shape. Each requested item includes its existing ID, path,
and name plus the immutable annotated target skeleton,
`needs_transformation`, and `statements_requiring_transformation`.

The replacer independently canonicalizes every returned function with the
shared operation before composing an implementation. It must not trust that a
caller previously invoked the validator. It then performs the existing target
header composition, recursive label removal, wrapper generation, call-site
rewriting, executable migration, and atomic multi-item replacement using the
canonicalized body.

Thus every LLM expansion group for a preserved label is mechanically
discarded. Only a transformed label can contribute LLM-written statements to
the emitted project.

The immutable initial skeleton remains safe to restore after earlier SCCs have
been promoted because the preservation proof rejects every call or callable
reference whose local target signature changes. A preserved call to an
unchanged-signature function never needs wrapper redirection.

### A2.7 SCC orchestration and no-LLM transactions

The scheduling unit remains one SCC:

```text
SCC needs an LLM request
    iff any member record has needs_transformation == true.
```

For an all-preserved SCC:

1. Do not construct dependency context or render a prompt.
2. Do not make an LLM request.
3. Do not invoke the structural validator.
4. Form the complete transformation from the member
   `annotated_skeleton` functions in ascending item-ID order.
5. Pass that transformation and the version-1 item metadata directly to the
   item replacer.
6. Install, build, and promote or fail using the ordinary SCC source
   transaction.

Replacement is still necessary when a function signature changed but its body
did not require an LLM. The replacer may introduce a compatibility wrapper or
perform the fixed `main_0` migration.

For a mixed SCC, make the existing one LLM request for the complete SCC,
including fully preserved member functions. Validation and replacement
mechanically restore their preserved statements. Do not split an SCC or
introduce wrappers between its members merely to avoid presenting a preserved
member to the LLM.

Keep the no-LLM branch direct and avoid trivially unnecessary per-request
work. This amendment does not require broader refactoring solely to make every
provider-adjacent resource lazy. Regardless of process-level initialization
details, an entirely mechanical run makes zero LLM requests and reports:

```text
llm_generation_calls = 0
repair_calls = 0
```

`function_count` and `scc_count` still count every function and SCC.
`cargo_builds` still includes the normalized initial build and each attempted
mechanical or LLM SCC candidate build. A mechanical replacement or build
failure is a fatal integration failure; it is not an LLM repair opportunity.

Keep the existing within-SCC final-name uniqueness check for mechanical and
LLM SCCs because the current complete-snippet replacement protocol identifies
returned functions by final name.

### A2.8 Prompt amendment

The unreleased prompt remains `local_transformation`, version 1. Amendment 2
defines its final version-1 text before the first release. Preserve the
complete body specified in
[Section 10](phase-4-plan.md#10-initial-llm-prompt-template) byte-for-byte
except for inserting this
exact paragraph after the paragraph ending “Use Dependency Context only as
reference; do not emit or redefine its functions, types, statics, or
constants.”:

```text
Complete every generated `todo!()` hole. Preserve every complete labeled statement already present in the Target Skeleton exactly as provided.
```

The validator deliberately accepts different returned contents for a
preserved group because those contents are discarded. The prompt asks the LLM
to reproduce complete statements to reduce noise and improve readability; it
is not the enforcement mechanism.

All prompt loads and request metadata continue to use version 1. The
historical [Section 10](phase-4-plan.md#10-initial-llm-prompt-template) text
remains unchanged in that document; this amendment
is the normative final text for the implementation's version-1 prompt.

### A2.9 Required regression scope

Implement every case in `amendment-2-test-plan.md`. Update existing Rust and
Python tests whose expected skeletons, JSON keys, request schemas, prompt
version, or SCC event traces are superseded, but do not edit
`phase-1-test-plan.md`, `phase-2-test-plan.md`, `phase-3-test-plan.md`, or
`phase-4-test-plan.md`.

The regression suite must cover at least:

- pointer-free scalar and aggregate statements;
- lvalue, rvalue, pattern, declaration, adjustment, and callable types;
- declarations without initializers;
- nested local ADTs, enums, unions, aliases, generic fields, and recursive
  type cycles;
- the future field-type-change hook;
- foreign calls, unsafe non-local calls, safe non-local calls, local unsafe
  calls, changed local signatures, indirect calls, callable references,
  macros, closures, inline assembly, and ambiguous desugarings;
- fully preserved and mixed nested control trees;
- Amendment Plan 1 interaction;
- preservation-aware validation and replacement, including deliberately
  changed discarded statements;
- all-preserved, mixed, and ordinary transforming SCCs;
- a signature-changing function with an entirely preserved body and required
  wrapper; and
- zero-LLM-call reporting, metrics, build failure, and deterministic ordering.
