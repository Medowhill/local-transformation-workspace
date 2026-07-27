# Amendment 1 Detailed Plan

This is a historical implementation plan. Its substantive text was moved
verbatim from the former consolidated `prototype-plan.md`; imperative and
future-tense wording describes the work assigned at the time. New navigation
notes identify where later work changed an earlier component.

See the [historical overview](prototype-plan.md#amendment-1).

## Amendment Plan 1: Restricted conditional expressions beneath non-control wrappers

This amendment intentionally relaxes the completed Phase 1 skeleton-generation
rule for control expressions beneath non-control wrappers. It applies after
Phases 1--4 and supersedes only the conflicting nested-control rejection and
statement-labeling requirements in this document. All historical phase test
plans remain unchanged.

### A1.1 Supported conditional-expression shape

An ordinary `if` expression may occur beneath a non-control expression wrapper
when it has the restricted, recursively conditional shape defined below. This
exception models C's conditional (`?:`) expression. It does not generally
permit Rust control flow beneath non-control wrappers.

A *restricted conditional* is an `if` expression satisfying all of these
requirements:

- it is an ordinary `if`, not `if let`;
- it has an `else`;
- its condition contains no control or plain-block expression;
- its `then` block contains exactly one statement, and that statement is a
  tail expression represented by `StmtKind::Expr`;
- its `else` is either:
  - an `else` block containing exactly one `StmtKind::Expr` tail expression; or
  - another restricted conditional, as in a syntactic `else if` chain; and
- each branch tail is either:
  - an expression containing no control or plain-block expression; or
  - a restricted conditional directly, recursively.

Consequently, both of these shapes are supported beneath a non-control
wrapper:

```rust,ignore
if c1 { a } else if c2 { b } else { c }
```

```rust,ignore
if c1 { a } else { if c2 { b } else { c } }
```

The second form is recursive because the sole tail expression of the outer
`else` block is itself a restricted conditional. A missing final `else`, an
empty branch, a semicolon-terminated branch expression, or any branch with a
preceding statement fails the restriction. For example, this remains
unsupported:

```rust,ignore
if c1 {
    foo();
    a
} else {
    b
}
```

An `if`, `if let`, `while`, `while let`, `for`, `loop`, `match`, or plain block
that occurs beneath a non-control wrapper and does not satisfy this exception
continues to produce `GenerationErrorKind::NestedControlPayload`. In
particular, a restricted conditional may occur directly as a branch tail, but
placing it beneath another wrapper inside that tail does not make the branch
restricted:

```rust,ignore
if c1 { 1 + if c2 { a } else { b } } else { c }
```

Direct-root control expressions retain the existing behavior. For example, a
direct `let` initializer whose root is `if` is still structurally preserved
and its branch statements are still labeled. This amendment applies only to a
restricted conditional encountered beneath a non-control wrapper.

### A1.2 Skeleton and labeling behavior

A non-control payload containing only permitted restricted conditionals is
still opaque to skeleton generation. Crat replaces that payload using the
existing hole rules rather than preserving any of its conditional structure.
For example:

```rust,ignore
x = 1 + if y { 2 } else { 3 };
```

is rendered in the skeleton as:

```rust,ignore
todo!();
```

The existing surrounding statement roles remain unchanged. A restricted
conditional inside a `let` initializer produces the existing typed
`let ... = todo!()` skeleton, one inside a direct `return` value produces
`return todo!()`, and one inside a direct `break` value produces
`break todo!()`.

The enclosing statement receives its ordinary numeric `#[proctor(...)]`
label. No statement inside any permitted restricted conditional beneath that
non-control wrapper receives a label, including statements in recursively
nested branch-tail conditionals or syntactic `else if` chains. The same label
tree is rendered in `annotated_source` and `annotated_skeleton`. This makes the
entire enclosing statement group one opaque transformation region.

Skipping labels must not skip input validation. Skeleton generation still
checks the complete source body for function-local items, empty statements,
and non-block match arms before applying the restricted-conditional label
exception.

Dependency collection remains unchanged. It continues to traverse the
original mapped HIR, so calls and referenced items inside an opaque restricted
conditional remain dependencies even though its branch statements are not
represented in the skeleton.

### A1.3 Implementation and validation

Implement the restriction once as a shared structural predicate used by both
nested-control validation and statement labeling. Do not duplicate slightly
different definitions in those paths. The skeletonizer must accept a
non-control payload only when every control or plain-block expression found
beneath it is a permitted restricted conditional; it then applies the existing
payload hole operation.

Update the in-memory Crat skeleton tests without editing any historical phase
test plan. The tests must cover:

- the assignment example above, including a single enclosing label and no
  branch labels in either annotated rendering;
- a syntactic `else if` chain;
- a restricted conditional used directly as a branch tail;
- the existing payload-role behavior for at least `let` and `return`;
- rejection of a branch with a preceding statement;
- rejection when a recursively nested conditional fails the restriction;
- rejection of a missing final `else`, `if let`, and non-`if` control beneath a
  non-control wrapper; and
- preservation of the existing labels and structure for a direct-root `if`.

No validator semantic change is required. The generated expected skeleton
contains only the enclosing labeled hole, and the Phase 2 validator already
permits unlabeled code introduced inside a leaf statement group. Retain or add
focused validator coverage demonstrating that behavior if needed, but do not
relax validation of hand-written expected skeletons containing control beneath
a non-control wrapper.
