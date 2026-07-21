# Initial Prototype Plan: LLM-Based Local Pointer Transformation

## 1. Purpose

This document specifies the first implementation milestone for the local-transformation component of Proctor.

It is intended to be used together with:

- the hybrid C-to-Rust research plan; and
- the Proctor pipeline component specification.

Those documents describe the overall research motivation and cross-component interfaces. This document focuses only on implementing the initial local-transformation prototype.

The prototype will validate the following loop:

1. Run Crat's existing whole-program pointer analysis.
2. Generate target function skeletons without rewriting function bodies.
3. Translate functions SCC-by-SCC with an LLM.
4. Validate each LLM result structurally with Crat.
5. Insert valid results into a fresh project.
6. Generate wrappers and redirect untransformed callers.
7. Run `cargo build`.
8. Repair failures with fresh LLM calls.
9. Continue until all function SCCs have been translated.

The prototype does **not** yet run tests, extract reusable rules, consume `proctor.toml`, or account for wrappers introduced by non-local transformations.

## 2. Scope and assumptions

### 2.1 Supported program model

The input is one compilable Rust crate produced by C2Rust.

Only source-defined free functions are transformed. Functions may appear in nested modules.

The prototype assumes the absence of:

- methods and `impl` items relevant to transformation;
- traits and trait methods;
- closures;
- generics;
- async functions;
- variadic functions;
- parameter patterns;
- function pointers;
- dynamic dispatch;
- callback patterns through `void *`;
- unexpanded macros.

C2Rust expands C macros before this stage.

Foreign functions, including libc functions, may be called or replaced by the LLM, but they are:

- never transformation targets;
- omitted from dependency context; and
- not represented as transformable function records.

### 2.2 Pointer-analysis scope

Crat's current pointer analysis is reused without modification.

For this prototype:

- named type definitions are never changed;
- global `static` and `const` types are never changed;
- function parameter types may change;
- function return types may change;
- local-variable types may change.

The prototype should initially be evaluated on programs that do not require pointer-containing structs, unions, enums, or type aliases to be transformed.

Crat's current rewriting phase may demote some analyzed pointer types back to raw pointers when rewriting is difficult. Skeleton generation must disable this transformation-time demotion and follow the initial analysis result exactly.

### 2.3 Rust assumptions

- The input project already passes `cargo build`.
- The project uses Rust 2021.
- Transformable functions are `unsafe fn`.
- The LLM must not introduce explicit `unsafe` blocks.
- Unsafe operations may be used directly inside these unsafe functions.
- The prototype supports executable targets and `cdylib` library targets.

## 3. Terminology

### Source function

The function before pointer-type transformation.

### Source signature

The signature in the source function.

### Target skeleton

The function skeleton produced from Crat's pointer-analysis result.

### Target signature

The signature selected by Crat's analysis and present in the target skeleton.

### Transformation target

A function in the SCC currently being rewritten by the LLM.

### Dependency context

Supporting declarations supplied to the LLM but not rewritten in the current request.

### Expansion group

The consecutive output statements corresponding to one labeled source statement.

## 4. Required Crat operations

The prototype requires three logical Crat operations. They may be exposed through the existing Crat CLI or through new subcommands.

### 4.1 Analyze and generate skeletons

Input:

- Rust project directory path.

Output:

- one JSON analysis file containing item metadata, dependencies, annotated source functions, and annotated target skeletons.

### 4.2 Validate an LLM transformation

Input:

- concatenated expected target skeletons;
- one LLM-generated Rust snippet;
- expected function names and item IDs.

Output:

- machine-readable validation JSON.

### 4.3 Create a candidate project

Input:

- current Rust project directory path;
- candidate-project destination path;
- JSON mapping from full Rust function paths to validated transformed snippets.

Output:

- a complete candidate Rust project containing replacements, wrappers, and rewritten call sites.

The Python orchestrator must not parse or rewrite Rust. All Rust-specific processing remains in Crat.

## 5. Skeleton generation

### 5.1 Analysis reuse

Crat runs its existing whole-program pointer analysis.

The analysis result determines:

- target parameter types;
- target return types;
- target local-variable types.

No expression rewriting is performed during skeleton generation.

### 5.2 Function skeletons

For every transformable function, Crat generates:

- the complete source function;
- the complete target skeleton.

The target skeleton:

- uses the target signature;
- uses analyzed target local-variable types;
- preserves the source statement and control structure;
- uses parseable placeholders such as `todo!()` for unimplemented expressions;
- does not apply transformation-time pointer demotion.

### 5.3 Statement labels

Every `rustc_ast::Stmt` in a transformable function receives a numeric annotation:

```rust
#[proctor(0)]
```

This includes tail expressions represented by `StmtKind::Expr`.

Label properties:

- labels are unique within one function;
- labels do not need to be globally unique;
- labels appear in source functions and skeletons;
- labels are retained during validation;
- labels are removed before insertion into a compilable project.

## 6. Analysis JSON

The analysis output is one JSON file containing a flat list of included items.

Every included item has:

- `id`: a unique natural-number ID;
- `path`: its crate-relative full Rust path;
- `kind`: its item kind.

The item ID is the primary identity for dependency analysis and validation. Full paths are used for diagnostics and source replacement.

### 6.1 Included item kinds

Include only:

- `Fn`;
- `Static`;
- `Const`;
- `TyAlias`;
- `Enum`;
- `Struct`;
- `Union`.

Do not emit records for:

- `ExternCrate`;
- `Use`;
- `ConstBlock`;
- `Mod`;
- `ForeignMod`;
- `GlobalAsm`;
- `Trait`;
- `TraitAlias`;
- `Impl`;
- `MacCall`;
- `MacroDef`;
- `Delegation`;
- `DelegationMac`.

Although `Mod` does not receive a record, Crat must recursively traverse modules to discover included items and construct full paths.

References to excluded items are ignored for dependency-context generation.

### 6.2 Dependency namespaces

Rust distinguishes value and type namespaces. The JSON should reflect this distinction.

For `Fn`, `Static`, and `Const`, include four dependency lists:

- `signature_dependencies`;
- `dependencies`;

For `TyAlias`, `Enum`, `Struct`, and `Union`, include:

- `dependencies`;

All dependency lists contain item IDs and record only direct dependencies.

#### Signature dependencies

For `Fn`, `Static`, and `Const`, signature dependencies are items referenced by the source declaration signature.

Examples include:

- a type used in a parameter or return type;
- a constant used in an array type such as `[T; N]`.

#### Dependencies

Dependencies are items referenced anywhere in the material used when the item is a transformation target.

For a function, this includes:

- the source signature;
- the source body.

Therefore:

```rust
fn f(x: T) {
    let y: S = todo!();
}
```

has `T` in its (signature) dependencies, while `S` appears only in its dependencies.

The following subset relation should hold:

```text
signature_dependencies ⊆ dependencies
```

### 6.3 Function records

Each `Fn` record contains:

- `id`;
- `path`;
- `name`: the final Rust function identifier;
- `kind`;
- `annotated_source`;
- `annotated_skeleton`;
- `source_signature`;
- `target_signature`;
- the two dependency lists defined above.

A directly recursive function includes its own ID in `dependencies`.

### 6.4 `Static` and `Const` records

Each `Static` or `Const` record contains:

- `id`;
- `path`;
- `kind`;
- declaration signature without the initializer;
- the two dependency lists.

Examples:

```rust
static X: i32;
static mut BUFFER: *mut u8;
const SIZE: usize;
```

### 6.5 Type-item records

Each `TyAlias`, `Enum`, `Struct`, or `Union` record contains:

- `id`;
- `path`;
- `kind`;
- complete original definition;
- `dependencies`;

Type items are always supplied as complete definitions and are never transformed.

## 7. Python orchestrator

The orchestrator is a separate Python program.

It is responsible for:

- invoking Crat analysis and skeleton generation;
- loading the analysis JSON;
- building the function-call graph;
- computing SCCs;
- scheduling SCCs;
- constructing dependency context;
- constructing LLM prompts;
- calling the LLM through LiteLLM;
- extracting Rust code from responses;
- invoking Crat validation;
- invoking Crat candidate-project generation;
- running `cargo build`;
- managing repair attempts;
- promoting successful candidate projects.

It must not parse Rust source.

### 7.1 LLM abstraction

Use LiteLLM directly for the first prototype.

Keep the LiteLLM-specific code behind a minimal interface, for example:

```python
class LlmClient:
    def generate(self, prompt: str) -> str:
        ...
```

Each call is independent. No conversation history is retained.

This interface should be replaceable by the team's shared LLM framework later.

## 8. Function graph and SCC scheduling

### 8.1 Function graph

Build a graph containing only transformable `Fn` items.

For each function `f`, inspect function-valued entries in its `dependencies`.

Add an edge:

```text
f -> g
```

when `f` directly calls `g`.

Foreign functions are absent from the graph.

### 8.2 SCCs

Compute strongly connected components and the SCC condensation DAG.

A leaf SCC has no outgoing edge to an unprocessed SCC.

Because edges point from callers to callees, leaf-first processing translates callees before external callers.

A singleton SCC is recursive when its function has a self-edge.

### 8.3 Deterministic scheduling

When multiple leaf SCCs are available, choose deterministically, for example by the smallest item ID in each SCC.

### 8.4 Function-name uniqueness

Before processing an SCC, the orchestrator checks that the final function names of all SCC members are distinct.

Crat identifies returned functions by name inside the single LLM response. Therefore, uniqueness is required only within the current SCC.

If duplicate function names occur within one SCC, orchestration aborts.

Functions in different SCCs may have the same final name.

## 9. Prompt-context construction

Each prompt has two conceptually separate parts:

1. **Transformation Targets**
2. **Dependency Context**

The transformation targets do not count toward the dependency-character limit.

The dependency context has a limit of 100,000 characters.

All dependency records are deduplicated by item ID and ordered deterministically by ID.

### 9.1 Transformation targets

For every function in the current SCC, include:

- its annotated source function;
- its annotated target skeleton.

These snippets already contain the source and target signatures, so those signatures are not repeated separately.

### 9.2 SCC signatures

If the SCC contains multiple functions, include the source and target signatures of every SCC member in the dependency context.

For a directly recursive singleton SCC, include its own source and target signatures.

For a nonrecursive singleton SCC, omit the redundant self-signature dependency.

### 9.3 Value dependencies

For each transformation target:

- inspect its `dependencies`.

For a function dependency, include:

- source signature;
- target signature.

For a `Static` or `Const` dependency, include:

- declaration signature.

Then, follow a dependency's signature dependencies only.

Therefore, if:

```text
a -> b -> c
```

the prompt for `a` includes `b`'s source and target signatures but not `c`'s signatures unless `c` appears in `b`'s signature.

### 9.4 Type dependencies

Type dependencies are followed transitively, as they have only `dependencies` but no `signature_dependencies`.

### 9.5 Transitive closure

Traverse dependencies breadth-first.

Examples:

- if target function `a` calls `b` and `b`'s signature refers to `T`, include `T`;
- if `a`'s refers to `S`, include `S`;
- if `T` refers to `U`, include `U`;
- continue transitively subject to the character budget.

### 9.6 Dependency budget

The 100,000-character budget applies only to the rendered dependency-context section.

Use this policy:

1. Add mandatory SCC signature dependencies.
2. Add all direct dependencies.
4. Tentatively add the complete next breadth-first depth.
5. Keep the depth only if the total dependency context remains within 100,000 characters.
6. Continue until the next complete depth does not fit.
7. Once a depth is rejected, do not consider deeper depths.

If mandatory direct dependencies already exceed 100,000 characters, abort the SCC instead of silently omitting them.

Instructions, transformation targets, prior failed code, and diagnostics are outside this limit.

## 10. Initial LLM prompt template

Use a prompt following this structure.

````text
You are transforming functions in unsafe Rust code generated from C.

Definitions:

- Source code is the current function implementation before pointer-type
  transformation.
- The source signature is the signature in the source code.
- The target skeleton was generated from whole-program static pointer analysis.
- The target signature is the signature in the target skeleton.
- A dependency's source signature shows its signature before transformation.
- A dependency's target signature shows how transformed code must call it.

Implement every function in the Transformation Targets section.

Requirements:

1. Preserve the exact behavior of the source code.
2. Strictly follow the parameter types, return type, and local-variable types
   fixed by the target skeleton.
3. Call transformed function dependencies using their target signatures.
4. Do not change any existing function name, parameter name, local-binding name,
   or item name.
5. New local bindings may be introduced only with names of the form
   `proctor_temp_var_n`, where `n` is a nonnegative integer.
6. A new temporary binding may be used only within the labeled expansion group
   in which it is declared, including unlabeled nested code inside that group.
7. Do not define additional functions, types, statics, constants, modules, or
   other items.
8. Every source statement label must appear at least once in the output.
9. A source statement may expand into one or more consecutive sibling statements
   with the same label.
10. Repeated occurrences of one label must be consecutive siblings at the same
    statement-list level. Do not repeat the same label in nested statements.
11. Newly introduced nested statements must be unlabeled.
12. Preserve the order of existing labels.
13. Preserve every existing control-structure kind and its existing labeled
    branch/body structure. Conditions, scrutinees, patterns, and statement
    contents may be rewritten.
14. If a labeled control statement expands into multiple sibling statements,
    exactly one sibling must preserve the original control structure. The other
    siblings must not be control-root statements and must contain no existing
    labels.
15. Do not introduce an explicit `unsafe` block. These functions are already
    unsafe functions, and unsafe operations may be used directly when needed.
16. Return exactly one fenced Rust code block containing all requested functions.
    Do not return prose.

Example:

Source:

```rust
unsafe fn read_value(p: *const i32) -> i32 {
    #[proctor(0)]
    let x: i32 = *p.add(1);
    #[proctor(1)]
    x
}
```

Target skeleton:

```rust
unsafe fn read_value(p: &[i32]) -> i32 {
    #[proctor(0)]
    let x: i32 = todo!();
    #[proctor(1)]
    todo!()
}
```

Valid output:

```rust
unsafe fn read_value(p: &[i32]) -> i32 {
    #[proctor(0)]
    let x: i32 = p[1];
    #[proctor(1)]
    x
}
```

Dependency Context:

{{DEPENDENCY_CONTEXT}}

Transformation Targets:

{{TRANSFORMATION_TARGETS}}

{{REPAIR_CONTEXT}}
````

For an initial request, `REPAIR_CONTEXT` is empty.

For a repair request, use:

````text
The previous transformation failed.

Previous transformation:

```rust
{{FAILED_TRANSFORMATION}}
```

Diagnostics:

```text
{{DIAGNOSTICS}}
```

Regenerate every function in the Transformation Targets section.
````

Each repair request uses the complete original prompt plus the latest failed transformation and latest diagnostics.

## 11. LLM response extraction

The orchestrator instructs the LLM to return exactly one fenced Rust code block and no prose.

To tolerate formatting errors:

1. Find all fenced code blocks.
2. Ignore prose outside code blocks.
3. If one block exists, use it.
4. If multiple blocks exist, choose the longest.
5. If multiple longest blocks have equal length, choose the first.
6. If no fenced code block exists, report a structural failure.

Pass the selected block unchanged to Crat.

The orchestrator does not parse Rust.

## 12. Structural model

### 12.1 Expansion groups

Each labeled source statement maps to a nonempty expansion group in the output.

An expansion group consists of one or more consecutive sibling statements:

- carrying the same source label;
- at the same statement-list level.

A label may not be repeated in nested statements.

### 12.2 Leaf statements

If a labeled source statement has no existing labeled descendants, it may:

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

When a source statement is an existing labeled control statement, the output must preserve its control kind.

The following kinds are distinct:

- `if`;
- `if let`;
- `while`;
- `while let`;
- `for`;
- `loop`;
- `match`.

Examples:

- `if` may not become `if let` or `match`;
- `while` may not become `for` or `loop`;
- `match` must remain `match`.

The output may rewrite:

- conditions;
- scrutinees;
- patterns;
- branch bodies;
- loop bodies;
- match-arm bodies.

It must preserve:

- the original control kind;
- the existence and role of branches;
- existing match-arm order and structure;
- existing labeled descendants and their placement.

### 12.4 Expanding a control statement

A labeled control statement may expand into multiple consecutive same-label sibling statements.

Exactly one sibling must:

- be a control-root statement;
- preserve the original control kind;
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

For each original statement list:

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

## 13. Identifier and temporary-variable rules

### 13.1 Existing identifiers

The transformation must preserve:

- function names;
- parameter names;
- existing local-binding names;
- existing item names.

References to functions, methods, fields, and operations may change when needed for translation. For example, a libc call may be replaced by a slice method.

### 13.2 New bindings

Every newly introduced local binding must have the form:

```text
proctor_temp_var_n
```

where `n` is a nonnegative integer.

For simplicity, Crat does not check whether the original program already uses this prefix. Any collision may be detected later by compilation.

Crat should reject duplicate declarations or shadowing of the same generated temporary name within one transformed function.

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

Crat should validate binding identity rather than relying only on identifier text.

This restriction keeps each labeled expansion syntactically local and suitable for later modular rule extraction.

## 14. Crat validator

The validator receives:

- concatenated expected skeleton functions;
- selected LLM-generated Rust snippet;
- expected function IDs and names.

Crat parses both snippets and matches functions by final name.

### 14.1 Whole-snippet checks

Require that:

- the snippet parses;
- it contains exactly the expected function definitions;
- every expected function appears exactly once;
- no additional items are introduced;
- no explicit `unsafe` block is introduced.

### 14.2 Signature checks

For each generated function, validate:

- function name;
- parameter count;
- parameter names;
- parameter types;
- return type.

Compare parsed types structurally, not textually.

The prototype excludes:

- variadic signatures;
- parameter patterns;
- generics;
- async functions.

Crat does not need to validate the generated function's:

- visibility;
- ABI;
- `unsafe` qualifier;
- `const` qualifier.

During replacement, Crat discards those header properties from the LLM result and emits its own exact target header.

### 14.3 Label and control checks

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
- temporary-variable naming and locality.

### 14.4 Validation output

Return a JSON list of failures.

Item-specific failure:

```json
{
  "id": 12,
  "failed_snippet": "unsafe fn f(...) { ... }",
  "errors": [
    "Label 3 is missing.",
    "Temporary proctor_temp_var_0 defined by label 2 is referenced from label 4."
  ]
}
```

Whole-snippet failure:

```json
{
  "id": null,
  "failed_snippet": "...",
  "errors": [
    "The returned Rust snippet does not parse: ..."
  ]
}
```

A full path is not required in validation output. The orchestrator can obtain it from the analysis JSON.

## 15. SCC transaction policy

Each SCC is all-or-nothing.

No function in the SCC is committed until:

1. every function passes Crat structural validation; and
2. the complete candidate project passes `cargo build`.

If either step fails:

- discard the entire candidate;
- discard partial success within the SCC;
- regenerate every function in the SCC.

## 16. Candidate-project generation

After structural validation succeeds, the orchestrator calls Crat with:

- current Rust project directory path;
- fresh candidate-project destination path;
- JSON mapping from full Rust function paths to transformed snippets.

Example:

```json
{
  "foo::bar": "unsafe fn bar(...) { ... }",
  "foo::baz": "unsafe fn baz(...) { ... }"
}
```

Crat then:

1. Copies the current project into the candidate destination.
2. Parses each transformed function.
3. Removes all `#[proctor(...)]` labels.
4. Replaces each original SCC function.
5. Uses Crat's exact target signature rather than trusting the LLM header.
6. Creates wrappers when source and target signatures differ.
7. Rewrites calls from external untransformed callers.
8. Writes the complete candidate project.

The current project is never modified in place.

## 17. Wrapper generation

### 17.1 Wrapper creation

Create a wrapper only when a function's source and target signatures differ.

The transformed implementation:

- remains at the original Rust path;
- uses the target signature.

The wrapper:

- uses the source signature;
- delegates to the transformed implementation;
- is an intentionally unsafe compatibility boundary.

All generated wrappers and transformed functions may be made `pub unsafe fn` to avoid privacy problems.

### 17.2 Wrapper module

For:

```text
foo::bar
```

generate the wrapper at:

```text
crate::proctor_translation_wrapper::foo::bar
```

The wrapper module mirrors the original module hierarchy.

For this prototype, assume that `proctor_translation_wrapper` does not conflict with the source crate.

Do not traverse the generated wrapper module during later call-site rewriting.

When adding wrappers for a new SCC, preserve wrappers created for earlier SCCs.

### 17.3 Export handling

For a signature-changing externally exported function:

- remove `#[no_mangle]` from the transformed implementation;
- give the wrapper the source signature;
- preserve the source ABI on the wrapper;
- add `#[no_mangle]` to the wrapper.

If a function's signature does not change, preserve any required export attributes on the original function.

The prototype considers only executable and `cdylib` targets.

### 17.4 Conversion logic

Follow Crat's existing wrapper-conversion logic as much as possible.

The prototype may use its current unsafe conversions for:

- references;
- optional references;
- slices;
- optional slices;
- boxes;
- optional boxes;
- boxed slices;
- optional boxed slices.

This includes the existing provisional slice-bound strategy.

These wrappers are intentionally unsafe and assume valid caller behavior.

Global-variable wrappers are not needed because global types are unchanged.

## 18. Call-site rewriting

When committing an SCC, Crat rewrites calls to each signature-changing SCC function.

Rewrite only callers outside the current SCC.

Use an absolute wrapper path:

```rust
crate::proctor_translation_wrapper::foo::bar
```

Crat must:

- leave calls between functions in the same SCC unchanged;
- not traverse the generated wrapper module;
- leave calls to signature-unchanged functions unchanged;
- rewrite all statically resolved calls from external untransformed callers.

Because SCCs are processed callee-first:

- all callees outside the current SCC have already been transformed;
- external callers of the current SCC are still untransformed;
- those external callers temporarily call wrappers;
- when an external caller is later transformed, its whole function is replaced;
- its LLM-generated body directly uses target signatures.

No separate transformed-function registry is required.

The immutable analysis JSON may contain stale source snippets after earlier call-site rewriting. This is acceptable because each function is later replaced as a whole using its original annotated source and target skeleton.

## 19. Compilation and promotion

After Crat creates a candidate project, run:

```bash
cargo build
```

in that candidate directory.

### Success

- Promote the candidate to become the current project.
- Mark the SCC as processed.
- Select the next leaf SCC.

### Failure

- Capture compiler standard output and standard error.
- Discard the candidate project.
- Keep the current project unchanged.
- Start a repair attempt for the entire SCC.

The prototype assumes Crat's integration routines are correct. Compiler diagnostics are given to the LLM even when they refer outside the SCC. The LLM may change only SCC functions. If that is insufficient, the retry limit eventually aborts orchestration.

## 20. Repair policy

The initial LLM generation does not count as a repair attempt.

After the initial failure, allow at most ten additional LLM calls for the SCC.

The ten-attempt limit combines:

- structural-validation repairs;
- compilation repairs.

Every additional LLM call consumes one repair attempt, regardless of its eventual failure category.

Each repair call:

- starts a fresh LLM session;
- receives the complete original prompt;
- receives the latest failed transformation;
- receives the latest structural or compiler diagnostics;
- regenerates every function in the SCC.

Each repair response repeats the full pipeline:

1. Extract the selected Rust code block.
2. Run Crat structural validation.
3. If valid, create a fresh candidate project.
4. Run `cargo build`.
5. Promote on success or retry on failure.

If the SCC has not succeeded after ten repair calls, abort the complete orchestration immediately.

## 21. Completion

The orchestrator succeeds when every SCC has been translated and the final promoted project builds.

For this prototype:

- all wrappers remain in the final project;
- no test suite is executed;
- no reusable rules are extracted;
- no rule-set file is read or written;
- `proctor.toml` is not required;
- wrappers from non-local transformations are ignored;
- all supported free functions are transformed.

This milestone demonstrates:

- whole-program skeleton generation;
- constrained LLM function rewriting;
- dependency-aware SCC scheduling;
- structural validation;
- wrapper-based incremental migration;
- compiler-guided repair.

It does not yet establish semantic correctness.

## 22. Implementation sequence

### Phase 1: Analysis JSON and skeleton generation

Implement and test:

- reuse of current pointer analysis;
- disabling transformation-time demotion;
- function-only skeleton generation;
- `todo!()` placeholders;
- statement annotations;
- source and target signatures;
- included-item filtering;
- dependency lists in value and type namespaces;
- JSON serialization.

Use small fixtures to inspect all emitted records manually.

### Phase 2: Structural validator

Implement and unit-test:

- parsing concatenated skeleton and result snippets;
- function matching by name;
- exact expected-function set checks;
- target-signature checks;
- label expansion groups;
- label order and nesting checks;
- control-kind preservation;
- control-statement expansion rules;
- existing-binding preservation;
- generated-temp naming;
- generated-temp locality;
- explicit unsafe-block rejection;
- item-specific and global JSON errors.

Create both positive and negative tests for every structural rule.

### Phase 3: Candidate-project generation

Implement and test:

- copying into a fresh destination;
- function replacement by full path;
- target-header enforcement;
- label removal;
- wrapper generation;
- wrapper-module maintenance;
- export handling;
- external call-site rewriting;
- avoidance of wrapper-module traversal.

Test manually on:

- a simple caller-callee chain;
- direct recursion;
- a mutually recursive SCC;
- nested modules;
- an exported `cdylib` function.

### Phase 4: Python orchestration

Implement:

- Crat process invocation;
- analysis JSON loading;
- SCC-local function-name uniqueness checks;
- function graph construction;
- SCC computation;
- deterministic leaf scheduling;
- dependency-context rendering;
- breadth-first type closure;
- 100,000-character dependency budget;
- prompt construction;
- LiteLLM client;
- code-block extraction;
- validation invocation;
- candidate generation;
- `cargo build`;
- repair accounting;
- project promotion and cleanup.

Keep LiteLLM behind a replaceable client abstraction.

### Phase 5: End-to-end evaluation

Begin with programs having:

- no pointer-containing named types;
- no function pointers;
- acyclic call graphs;
- unique function names within every SCC;
- simple reference and slice transformations.

Then add:

- direct recursion;
- mutually recursive functions;
- optional references;
- boxes and boxed slices;
- exported library functions;
- cases that require structural repair;
- cases that require compiler-guided repair.

Record at least:

- number of functions and SCCs;
- initial-generation success rate;
- structural-repair count;
- compilation-repair count;
- final build success;
- failure reason when orchestration aborts.

## 23. Deferred work

The following are intentionally outside this prototype:

- test execution and semantic validation;
- reusable-rule extraction;
- rule application;
- preservation labels;
- improved pointer analysis;
- pointer-containing named-type transformation;
- global-variable transformation;
- global-variable wrappers;
- custom-type wrapper generation;
- function pointers and callbacks;
- methods, traits, generics, and closures;
- integration with non-local transformation wrappers;
- `proctor.toml` integration;
- replacement of LiteLLM with the team's shared framework.
