# Phase 2 Detailed Plan: Structural Validator

This is a historical implementation plan. Its substantive text was moved
verbatim from the former consolidated `prototype-plan.md`; imperative and
future-tense wording describes the work assigned at the time. New navigation
notes identify where later work changed an earlier component.

See the [historical overview](prototype-plan.md#phase-2-structural-validator)
and [Phase 2 test plan](phase-2-test-plan.md). Corrections designed and
implemented with the next task set are preserved in
[Phase 3's amendments to earlier behavior](phase-3-plan.md#amendments-to-earlier-behavior).

## Historical context

Phase 1 was implemented and validated before the Phase 2 design was finalized.
Phase 2 includes four intentional adjustments to the completed Phase 1
generator, all specified in Section 5.2: mark every target binding mutable;
make every target function header unsafe while preserving source safety;
exclude every free function whose final identifier is `main`; and force the
supported two-argument `main_0` target `argv` type to
`&mut [&mut [i8]]`. The implementation agent for Phase 2 must make these
generator changes and update the existing Phase 1 Rust tests as specified in
`phase-2-test-plan.md`. During Phase 2, do not edit or reinterpret the
historical `phase-1-test-plan.md`, and do not otherwise reopen Phase 1
behavior. Moving the existing code and tests into the `skeleton` module is an
organizational refactor, not another Phase 1 semantic change. The explicit
Phase 3 amendments below supersede that completed behavior without rewriting
the historical plans.

## Crat operation

### 4.2 Validate an LLM transformation

Expose this operation as:

```text
crat-tool validate --input <request.json> --output <response.json>
```

The request contains:

- an ordered list of expected functions, each containing its item ID, final
  name, and complete annotated target skeleton; and
- one LLM-generated Rust transformation snippet containing all requested
  functions.

Output:

- one machine-readable response whose status is `valid`, `invalid`, or
  `setup_error`, as specified in Section 14.

The production command reads the request JSON, invokes the in-memory validator,
and writes the response JSON. It exits with status zero after successfully
writing any of the three response statuses. It exits nonzero only when it
cannot complete the response protocol, such as invalid CLI invocation,
unreadable input, unwritable output, serialization failure, or a panic. In
particular, structural validation failure is ordinary output, not a process
failure.

Phase 2 tests exercise only in-memory request construction, parsing,
validation, and response serialization. They do not invoke the CLI or perform
filesystem I/O.

## Changes to skeleton generation

The target skeleton:

- uses the target signature and is always `unsafe fn`, even when the source
  function is safe;

Function safety follows the same source/target separation. Preserve whether
the source function is safe or unsafe in `annotated_source` and
`source_signature`. Set the header to `unsafe fn` in
`annotated_skeleton` and `target_signature`. This is a target normalization,
not an inferred property and not a validator decision. Phase 2 ignores the
LLM-returned function's safety qualifier; Phase 3 inserts Crat's unsafe target
header.

Do not emit a `Fn` record for any free function whose final identifier's
symbol is `main`, at any inline-module depth. This includes a surface spelling
of `r#main`. The exclusion is unconditional and does not inspect the function
body. Such a function receives no item ID, has no skeleton, and is absent from
dependency lists and the function graph. `main_0` remains an ordinary
transformation target.

### Included function-record rule

- `Fn`, except a free function whose final identifier is exactly `main`;

## 12. Structural model

### 12.1 Expansion groups

Each labeled target-skeleton statement maps to a nonempty expansion group in
the output.

An expansion group consists of one or more consecutive sibling statements:

- carrying the same target-skeleton label;
- at the same statement-list level.

Every statement in a group has exactly one outer `#[proctor(N)]` attribute.
The token spelling of `N` must match the canonical unsuffixed decimal grammar
`0|[1-9][0-9]*`, and its value must be in the `u32` range. Leading zeroes,
numeric separators, signs, radix prefixes, and integer suffixes are rejected;
for example, `00`, `1_0`, `+1`, `0x1`, and `0u32` are malformed. A label may
not be repeated in nested statements. A `proctor` attribute with a malformed
path, argument, argument count, or integer token is an item-specific validation
error rather than an unknown label. Validate the original literal-token
spelling before or alongside numeric conversion; checking only a normalized
AST integer value is insufficient because it can erase separators, radix, or
suffix distinctions.

At every statement-list level corresponding to a target-skeleton statement
list, the result consists only of the expected expansion groups in expected
order. It may not contain an unlabeled sibling between groups. Unlabeled
statements may occur only inside newly introduced nested code that is itself
inside one expansion-group statement and does not correspond to an existing
target-skeleton statement list.

### 12.2 Leaf statements

If a labeled target-skeleton statement has no existing labeled descendants, it
may:

- remain one labeled statement;
- expand into multiple consecutive same-label sibling statements;
- introduce new nested expressions, blocks, or control flow internally.

Any newly introduced nested statements must be unlabeled.

Valid:

```rust
#[proctor(1)]
let x = if p.is_some() {
    *p.unwrap()
} else {
    0
};
```

Valid one-to-many expansion:

```rust
#[proctor(1)]
let proctor_temp_var_0 = p.unwrap();

#[proctor(1)]
let x = *proctor_temp_var_0;
```

Invalid nested repetition:

```rust
#[proctor(1)]
let x = if p.is_some() {
    #[proctor(1)]
    *p.unwrap()
} else {
    #[proctor(1)]
    0
};
```

### 12.3 Existing control structures

Phase 1 preserves a control or plain block expression only when it is reached
through one of these statement roles:

- the root expression of an expression or semicolon statement;
- the direct initializer of a `let` statement;
- the direct value of `return`;
- the direct value of `break`; or
- the block tail of a supported match arm.

These wrappers are structural. The output must preserve the same role: for
example, a `return if ...` remains a return whose direct value is an `if`, and
a `let x = loop ...` retains the existing `x` declaration with a direct `loop`
initializer. The validator does not look through calls, constructors,
operators, tuples, casts, assignments, or other non-control expression
wrappers to find a control root.

When a target-skeleton statement has such an existing control root, the output
must preserve its control kind. The following kinds are distinct:

- plain block;
- `if`;
- `if let`;
- `while`;
- `while let`;
- `for`;
- `loop`;
- `match`.

`let-else` is a distinct structural statement form rather than a control-root
expression. Its initializer role, else-block role, and recursively labeled
descendants must be preserved.

Examples:

- `if` may not become `if let` or `match`;
- `while` may not become `for` or `loop`;
- `match` must remain `match`;
- a plain block may not become a loop; and
- `let-else` may not become a plain `let` followed by an `if`.

The output may rewrite:

- conditions;
- scrutinees;
- patterns;
- branch bodies;
- loop bodies;
- match-arm bodies.

It must preserve:

- the original control kind;
- whether an `if` has an `else`;
- the complete recursive `else if` structure, even though an `else if`
  expression has no independent statement label;
- the number and order of match arms;
- the presence or absence of each match-arm guard;
- the existence and role of `let-else`, branch, loop, match-arm, and plain
  block bodies;
- existing labeled descendants and their placement.

Patterns may be rewritten, but Section 13 still requires every binding
declaration in an existing pattern to remain exactly once in the corresponding
pattern role and permits only correctly named generated temporaries as new
bindings.

### 12.4 Expanding a control statement

A labeled control statement may expand into multiple consecutive same-label sibling statements.

Exactly one sibling must:

- be a control-root statement;
- preserve the original control kind;
- preserve the original statement role;
- contain all recursively preserved labeled descendants.

Every other sibling must:

- not be a control-root statement;
- contain no existing labels.

Valid transformation of an original `if`:

```rust
#[proctor(1)]
let proctor_temp_var_0 = p.is_some();

#[proctor(1)]
if proctor_temp_var_0 {
    #[proctor(2)]
    x = *p.unwrap();
} else {
    #[proctor(3)]
    x = 0;
}

#[proctor(1)]
let y = x + 1;
```

Invalid because two same-label siblings are control-root statements:

```rust
#[proctor(1)]
if condition_a {
    #[proctor(2)]
    x = *p.unwrap();
} else {
    #[proctor(3)]
    x = 0;
}

#[proctor(1)]
if condition_b {
    x += 1;
}
```

Invalid because the original control kind changed:

```rust
#[proctor(1)]
while condition {
    #[proctor(2)]
    x = *p.unwrap();
}
```

### 12.5 Label-order rules

For each target-skeleton statement list:

- every original label appears at least once;
- no new label appears;
- expansion groups occur in original label order;
- each expansion group is consecutive;
- no label may reappear after another label begins.

If the original order is:

```text
1, 2
```

then these are invalid:

```text
2, 1
```

and:

```text
1, 2, 1
```

### 12.6 Preservation of labeled descendants

Existing labeled descendants must remain recursively associated with their original labeled ancestor.

They may not be:

- moved to another branch;
- moved to another match arm;
- moved outside their original control statement;
- wrapped by a newly introduced labeled ancestor;
- duplicated;
- removed.

New unlabeled nested code may be introduced inside an expansion group.

Association is checked by structural role, not only by preorder. A descendant
must remain in the same `if` branch, loop body, `let-else` body, plain block,
or match-arm index. Moving a label between two branches while preserving
global preorder is invalid.

## 13. Identifier and temporary-variable rules

### 13.1 Existing identifiers

The transformation must preserve each existing declaration identity, not just
a function-wide set of spellings:

- function names;
- parameter names;
- existing local-binding names.

Every parameter remains in its original parameter position. Every local or
pattern binding remains declared exactly once in the same expansion group and
the same recursive structural role as in the target skeleton. This includes
bindings in destructuring and `let-else` patterns, `if let`, `while let`, `for`,
and the individual arms of a `match`. Two source bindings with the same
spelling in different scopes remain two distinct declarations in their
respective locations.

Represent each declaration role with a stable structural position:

- a parameter is anchored by its zero-based parameter index;
- a statement-local pattern is anchored by the expected label, the
  zero-based statement position within that expansion group's preserved
  structural list, and its role (`let` pattern, `let-else` pattern, `if let`
  pattern, `while let` pattern, `for` pattern, closure parameter, or match-arm
  pattern with zero-based arm index);
- nested anchors append the preserved branch, loop body, match-arm, plain
  block, or `let-else` body path; and
- within a pattern, append exact child segments for binding nodes and for
  tuple/tuple-struct indices, struct field names or indices, slice
  prefix/rest/suffix positions, `@` binder versus subpattern, `or`-pattern
  alternative indices, and reference/parenthesized subpatterns.

Constructor and path spellings, literals, ranges, and other nonbinding pattern
content may change only when the projected binding-position paths remain
unchanged. Adding or removing a wrapper or moving a binding between tuple
slots, struct fields, slice positions, `@` roles, `or` alternatives, reference
patterns, branches, or arms changes its declaration role and is rejected.
Bindings with the same spelling across alternatives of one valid `or` pattern
form one Rust declaration identity: validate the complete alternative-position
set as one identity rather than reporting the alternatives as duplicates.
Preserve by-value versus `ref` binding mode; ignore only `mut`, including the
`mut` in `ref mut`. After a binding is uniquely associated with its expected
structural position, a by-value/`ref` difference is
`existing_binding_mode_mismatch`, not a location mismatch. Its message names
the binding, label and pattern role, expected and observed binding modes, and
instructs the LLM to restore or remove `ref` while explaining that `mut` may
differ.

An existing declaration may not be deleted, duplicated, moved to another
label, moved to another branch or arm, or replaced by a generated temporary.

For every existing local declaration, preserve whether an explicit type is
present. If present, its result type must structurally equal the target
skeleton's type. This enforces the pointer-analysis decision materialized by
Phase 1. Binding mutability is the sole exception: the validator ignores
`mut`/non-`mut` differences for parameters and all local or pattern bindings.

References to functions, methods, fields, and operations may change when needed for translation. For example, a libc call may be replaced by a slice method.

### 13.2 New bindings

Every newly introduced local binding must have the form:

```text
proctor_temp_var_n
```

where `n` is a nonnegative integer.

This rule applies to every new binding pattern visible in the parsed result,
including simple and destructuring `let` bindings, `let-else`, `if let`,
`while let`, `for`, `match` arms, and closure parameters. Binding mutability is
unrestricted.

The Phase 1 input contract does not reserve this prefix. If an expected
function already contains a binding with this spelling, it is an existing
binding and retains its existing declaration identity. A newly introduced
binding may not reuse that spelling.

Crat rejects duplicate declarations or shadowing of the same generated
temporary name anywhere within one transformed function. Different generated
temporaries must have different numeric suffixes; suffixes need not be
contiguous or start at zero.

### 13.3 Temporary locality

A generated temporary belongs to the expansion group containing its declaration.

All references to that binding must occur:

- in sibling statements in the same expansion group; or
- in unlabeled nested code inside those sibling statements.

It may not be used by a statement carrying another label.

Valid:

```rust
#[proctor(1)]
let proctor_temp_var_0 = p.unwrap();

#[proctor(1)]
let x = *proctor_temp_var_0;

#[proctor(2)]
use_value(x);
```

Invalid:

```rust
#[proctor(1)]
let proctor_temp_var_0 = p.unwrap();

#[proctor(2)]
let x = *proctor_temp_var_0;
```

Crat validates binding identity and lexical scope rather than treating every
identifier token with the same spelling as a reference. Because duplicate or
shadowing declarations of a generated temporary are prohibited, each accepted
temporary declaration has one unambiguous identity.

Macro token trees are opaque to parser-only Rust AST name resolution. A
generated-temporary identifier may therefore not occur anywhere inside a
macro invocation. The validator reports a specific error instructing the LLM
to move that use into ordinary Rust syntax. Other macro invocations remain
allowed when they satisfy the surrounding structural rules.

This restriction keeps each labeled expansion syntactically local and suitable for later modular rule extraction.

### 13.4 Attributes

The target skeleton contains only statement-root `#[proctor(N)]` attributes
in function bodies. A result may not introduce any other statement or
expression attribute, and a `proctor` attribute may not appear anywhere
except the root attribute storage of a statement in an expansion group.

## 14. Crat validator

The in-memory validator receives a request with:

- `schema_version`, which must be the integer `1`;
- an ordered `expected_functions` array; and
- the selected LLM-generated Rust `transformation`.

Each expected-function entry contains:

- `id`: its Phase 1 item ID;
- `name`: its final Rust identifier; and
- `skeleton`: its complete annotated target skeleton.

The exact request schema is:

```json
{
  "schema_version": 1,
  "expected_functions": [
    {
      "id": 12,
      "name": "f",
      "skeleton": "pub unsafe fn f(mut p: &[i32]) -> i32 { ... }"
    }
  ],
  "transformation": "pub unsafe fn f(p: &[i32]) -> i32 { ... }"
}
```

`schema_version` and every `id` are JSON integers, not strings or floating
point values. `schema_version` must equal `1`; every `id` must be in the Rust
`u64` range. Reject unknown JSON fields as setup errors. IDs and names must
each be unique within the request. Each skeleton string must parse as exactly
one free function whose name matches its entry, and it must contain a valid
Phase 1 label/control tree with unique labels. A violation in request metadata
or an expected skeleton is a setup error, not an LLM validation failure.

Suitable public API shapes are:

```rust,ignore
pub fn validate(request: &ValidationRequest) -> ValidationResponse;
pub fn validate_json(input: &str) -> String;
```

The typed API performs no I/O. The JSON API parses an input string and returns
pretty JSON even when request deserialization fails; malformed request JSON is
represented by `status = "setup_error"`.

Every response uses response `schema_version: 1`, including responses to
malformed JSON and requests whose request `schema_version` is unsupported.

Crat parses the result as a crate and matches its top-level functions by final
name. Expected order is request order; result function order is irrelevant.

### 14.1 Setup checks

Before inspecting an LLM result, require:

- supported `schema_version`;
- no unknown request fields;
- unique expected IDs;
- unique expected names;
- nonempty expected function list;
- exactly one correctly named free function in each skeleton string;
- a supported target signature in each skeleton;
- well-formed, unique numeric statement labels;
- a valid Phase 1 control/statement tree;
- no conflict between request metadata and parsed skeletons.



### 14.2 Whole-snippet checks

Require that:

- the snippet parses;
- it contains exactly the expected function definitions;
- every expected function appears exactly once;
- no additional top-level items are introduced.

The exact-function-set check includes every top-level item kind: `use`,
modules, constants, types, macro definitions or invocations, and foreign items
are all unexpected. Duplicate or missing expected functions and extra
functions are whole-snippet failures. If parsing or exact-function-set checking
fails, return one global failure and suppress per-function checks.

### 14.3 Signature checks

For each generated function, validate:

- function name;
- parameter count;
- parameter names;
- parameter types;
- return type.


Compare parsed types structurally, not textually. Ignore source spans, node
IDs, token caches, formatting, and redundant parentheses. Otherwise require
the same type structure, including paths, path qualification, mutability
inside pointer/reference types, tuples, arrays and lengths, generic arguments,
and explicit lifetime names. Do not perform name resolution or treat aliases
as equivalent. An omitted return and an explicit `-> ()` are distinct
structures.

The prototype excludes:

- variadic signatures;
- parameter patterns;
- source type and const generics, generic bounds, and where clauses; Phase 1
  generated lifetime parameters remain supported;
- async functions.

Crat does not validate the generated function's:

- visibility;
- ABI;
- `unsafe` qualifier;
- `const` qualifier; or
- binding mutability in parameters.


### 14.4 Binding-type checks

After matching expansion groups and structural roles, compare every existing
local declaration against its target-skeleton declaration. The existing
binding identities and their placement must match Section 13.1. An explicit
target type must be present and structurally equal in the result; a target
declaration without an explicit type must remain without one. Ignore binding
mutability during every such comparison.

### 14.5 Label, control, binding, and attribute checks

Validate all rules in Sections 12 and 13, including:

- nonempty expansion groups;
- consecutive same-label siblings;
- no nested repetition of a label;
- no deleted or new labels;
- original label order;
- preserved control kind;
- preserved labeled descendants;
- only one control-root statement when expanding a labeled control statement;
- unlabeled newly introduced nested statements;
- exact preservation and placement of existing declarations;
- target local-variable types;
- temporary-variable naming and locality;
- prohibition of generated temporaries inside macro token trees;
- prohibition of new statement/expression attributes; and
- prohibition of explicit unsafe blocks.

An explicit unsafe block is a user-written `unsafe { ... }` block anywhere
inside a returned function body, including newly introduced nested code.
Report `explicit_unsafe_block` on that matched function's item-specific
failure, with its expected ID and name and the complete pretty-printed matched
function as `failed_snippet`. The message identifies the enclosing expansion
group label when association is available. Unsafe-block detection is
independent of label/control association and continues even when another body
rule fails.

Diagnostics must be repair-oriented. Each message identifies the function and
item ID when available, the relevant label and structural role when available,
what was expected, what was observed, and the concrete corrective constraint.
Do not emit cascades: if a missing or malformed parent group/control makes
descendant association unreliable, report the parent error and suppress
derivative descendant, declaration, and temporary-locality errors.

### 14.6 Validation response

Every successfully processed request produces one of three tagged response
objects. Serialize with pretty JSON equivalent to
`serde_json::to_string_pretty`, in the key orders shown below, with no trailing
newline.

Validation success:

```json
{
  "schema_version": 1,
  "status": "valid"
}
```

Validation failure:

```json
{
  "schema_version": 1,
  "status": "invalid",
  "failures": [
    {
      "id": 12,
      "name": "f",
      "failed_snippet": "unsafe fn f(...) { ... }",
      "errors": [
        {
          "code": "missing_label",
          "message": "Function `f` (item 12): label 3 is missing. Restore label 3 in its original structural position."
        }
      ]
    }
  ]
}
```

For an item-specific failure, `failed_snippet` is the complete pretty-printed
matched result function. For a global failure, `id` and `name` are null and
`failed_snippet` is the complete unchanged transformation input:

```json
{
  "schema_version": 1,
  "status": "invalid",
  "failures": [
    {
      "id": null,
      "name": null,
      "failed_snippet": "...",
      "errors": [
        {
          "code": "result_parse_error",
          "message": "The returned Rust transformation does not parse: ..."
        }
      ]
    }
  ]
}
```

Setup error:

```json
{
  "schema_version": 1,
  "status": "setup_error",
  "error": {
    "code": "duplicate_expected_name",
    "message": "Expected function name `f` appears twice. Function names must be unique within one validation request."
  }
}
```

`errors` is nonempty, and each error contains only `code` and `message` in that
order. `failures` is nonempty and is ordered by the request's expected-function
order after any global failure. Within one function, emit errors in a fixed
validation-pass order and then in target-skeleton structural preorder.
Repeated validation of the same bytes must produce byte-identical JSON.


### 14.7 Exit behavior

The `validate` process exits zero after writing any `valid`, `invalid`, or
`setup_error` response. The orchestrator proceeds only for `valid`, requests an
LLM repair only for `invalid`, and aborts immediately for `setup_error`.

The process exits nonzero only when it cannot produce and write a trustworthy
response, including:

- invalid CLI invocation;
- failure to read the request path;
- failure to write the response path;
- response-serialization failure; or
- panic or other unexpected internal failure.

CLI usage and filesystem failures may be diagnosed on standard error because a
response file cannot be guaranteed in those cases.

### 14.8 Error aggregation and precedence

Use these stages:

1. Deserialize and setup-check the request. Return the first `setup_error` on
   failure.
2. Parse the result. Return one global failure on failure.
3. Check the exact top-level function set. Return one global failure containing
   all deterministically ordered set errors on failure.
4. Validate each matched function independently, including explicit unsafe
   blocks. Accumulate all reliable item-specific errors.

Order exact-function-set errors as follows: missing expected functions in
request order, duplicate returned functions in result source order, then
unexpected returned functions or other items in result source order.

Within a function, signature errors do not prevent body checks when the body
can still be associated safely. A broken label/control parent suppresses only
checks that depend on that association. Independent siblings and independent
rules continue to be checked so one repair response can address multiple
reliable problems.

For item-specific reporting, use this stable category order without changing
the analysis dependencies needed to establish reliable associations:

1. signature;
2. existing declaration identity and target local type;
3. labels and expansion groups;
4. controls and descendant placement;
5. generated temporaries;
6. explicit unsafe blocks and body attributes.

Within one category, use target-skeleton structural preorder, then result
source order for result-only constructs.

Use the following same-category precedence and suppression rules:

| Area | Precedence and suppression |
| --- | --- |
| Label syntax and placement | Report malformed or misplaced `proctor` attributes in result source order before sequence diagnostics. A malformed attribute occupying an expected group position suppresses a derivative `missing_label` there. A misplaced attribute does not create a statement label. |
| Label sequence | After well-formed root labels are collected, report nested repetition, then nonconsecutive reappearance, then order mismatch, then missing expected labels in target preorder, unexpected labels in result source order, and unlabeled sibling statements in result source order. Do not report `label_order_mismatch` for a sequence already explained by `nonconsecutive_label`. |
| Controls and descendants | Validate the parent control kind, statement role, branch/arm shape, and required control-root count before descendant placement. A parent failure suppresses descendant label, declaration, type, and temporary-locality errors only beneath the unreliable role. With a valid parent, a label found in the wrong branch, arm, or structural statement list produces `descendant_location_mismatch`, not a missing-plus-unexpected-label pair. |
| Generated temporaries | Report invalid generated binding names, duplicate declarations, unresolved generated-temporary references, cross-group references, and macro-token occurrences in that order, each in result source order. Suppress a locality error when the declaration or enclosing expansion group cannot be associated reliably. |
| Body safety and attributes | Report explicit unsafe blocks in result source order, followed by unexpected statement/expression attributes in result source order. These AST-local checks do not depend on successful label/control association. |

A full path is not required in validation output. The orchestrator can obtain
it from the skeleton JSON.

## Implementation sequence

### Phase 2: Structural validator

Implement and unit-test:

- the four Phase 1 generator adjustments from Section 5.2: recursively mark
  every target binding mutable; make every target function header unsafe while
  preserving source safety; omit every free function named `main`; and force
  the supported two-argument `main_0` target `argv` type to
  `&mut [&mut [i8]]`;
- all affected existing Phase 1 Rust test oracles and the additional
  regressions listed in `phase-2-test-plan.md`, without editing the historical
  Phase 1 plan;
- moving the existing Phase 1 implementation and tests into a `skeleton`
  module, adding a separate `validator` module, and keeping `lib.rs` limited to
  required crate-level attributes/`extern crate` declarations, module
  declarations, and public re-exports, with no implementation or large tests;
- the typed in-memory validation API and the pure JSON-string API;
- the versioned request schema with per-function IDs, names, and target
  skeletons;
- setup validation and the `setup_error` response;
- the amended Phase 2 expected-skeleton setup check that rejects every
  function-local item;
- parsing result snippets;
- function matching by name;
- exact expected-function set checks;
- exact complete lifetime-generic declaration checks, with only Phase
  1-generated named lifetime parameters supported, no lifetime-parameter
  attributes or syntactically present `where` clause accepted, and
  `generic_parameter_mismatch` added to the stable code set;
- structural target-signature and existing-local-type checks, ignoring all
  binding mutability;
- label expansion groups;
- label syntax, placement, order, grouping, and nesting checks;
- recursive control-kind, branch/arm, plain-block, and `let-else`
  preservation consistent with Phase 1;
- control-statement expansion rules;
- strict declaration-identity, by-value-versus-`ref` binding-mode, and
  placement preservation;
- rejection of every function-local item in a returned transformation;
- generated-temp naming;
- generated-temp locality;
- generated-temp macro-token rejection;
- new statement/expression-attribute rejection;
- item-specific explicit unsafe-block rejection;
- deterministic, cascade-suppressed, repair-oriented diagnostics with stable
  codes;
- the `valid`, `invalid`, and `setup_error` JSON response schemas;
- deterministic response serialization; and
- the thin `crat-tool validate --input ... --output ...` command with the exit
  behavior in Section 14.7.

Implement every still-applicable case in `phase-2-test-plan.md` plus the Phase
1/2 amendment cases in Section 4 of `phase-3-test-plan.md`. Update or remove
affected existing Phase 1 and Phase 2 Rust tests, but do not edit either
historical planning document. All functional tests use in-memory APIs and
perform no filesystem writes or subprocess invocation. The CLI wiring is not
tested in Phase 2. The new source-safety and executable decisions add
generator work and generator regressions during Phase 2, but no additional
validator rule: safety was already an ignored LLM-header property, `main`
never enters a validation request, and the forced `main_0` type is validated
through the ordinary structural parameter-type rule. The lifetime-generic
declaration check, complete function-local-item rejection, and arity-only
executable recognition are the three intentional amendments added during
Phase 3 planning.
