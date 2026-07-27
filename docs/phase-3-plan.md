# Phase 3 Detailed Plan: Item Replacement and Integration

This is a historical implementation plan. Its substantive text was moved
verbatim from the former consolidated `prototype-plan.md`; imperative and
future-tense wording describes the work assigned at the time. New navigation
notes identify where later work changed an earlier component.

See the [historical overview](prototype-plan.md#phase-3-item-replacement-and-integration)
and [Phase 3 test plan](phase-3-test-plan.md).

## Historical context

Phase 2 was implemented and validated before the Phase 3 design was finalized.
Phase 3 adds three intentional amendments to the completed Phase 1/2
implementation:

- validate the complete function lifetime-generic declaration rather than
  validating lifetime names only where they occur inside parameter and return
  types;
- reject every function-local parsed item statement, including the local
  `const` and `static` items that Phase 1 and Phase 2 previously handled; and
- recognize the special zero- and two-argument `main_0` forms using only the
  final identifier symbol and arity, without inspecting types or bodies.

Sections 5.2, 13.1, 13.4, 14.1, 14.5, and 14.8 are normative for the local-item
amendment; Section 14.3 is normative for the lifetime amendment; and Sections
5.2 and 16.4 are normative for arity-only executable recognition. Do not edit
the historical `phase-1-test-plan.md` or `phase-2-test-plan.md`; the required
updates to existing Phase 1 and Phase 2 Rust tests and the additional
regressions are specified in Section 4 of `phase-3-test-plan.md`.

## Crat operations

### 4.3 Normalize target safety

Expose the production operation as:

```text
crat-tool normalize-safety \
    --output <output.rs> \
    <input.rs>
```

The command reads one Rust source file and writes one Rust source file. It
does not read skeleton JSON, copy a Cargo project, or modify any other file.
Creation of the stage-private project and installation of the returned source
belong to Phase 4.

The underlying library operation is parser-only and in-memory:

```rust,ignore
pub fn normalize_target_safety(
    source: &str,
) -> Result<String, ReplacementError>;
```

It accepts and returns the exact single-file library source text and performs
no filesystem access. Section 16.1 specifies its complete behavior.

### 4.4 Replace validated items

Expose the production operation as:

```text
crat-tool replace \
    --request <request.json> \
    --output <output.rs> \
    <current-project>
```

The versioned request contains an ordered list of replacement item identities
and the one complete, already validated Rust transformation snippet returned
for the current SCC. It deliberately does not require Python to split or parse
that Rust snippet:

```json
{
  "schema_version": 1,
  "items": [
    {
      "id": 12,
      "path": "foo::bar",
      "name": "bar"
    }
  ],
  "transformation": "unsafe fn bar(...) { ... }"
}
```

Phase 3 supports only function entries, but the request and module use item
terminology because later phases may replace other item kinds. The exact
schema and setup rules are in Section 16.2.

The production command locates and compiles the current project's one library
source, calls the in-memory Rust-aware replacement operation, and writes only
the returned library source to the requested `.rs` output path. It does not
copy or mutate the current project. Phase 4 temporarily swaps this candidate
source into the stage-private current project, builds it, and either keeps the
candidate or restores the prior source. The generated Cargo bin shim and every
other project file therefore remain stage-owned and unchanged by `replace`.

A suitable library API is:

```rust,ignore
pub fn replace_items(
    source: &str,
    request: &ReplacementRequest,
    tcx: TyCtxt<'_>,
) -> Result<String, ReplacementError>;
```

The library receives no project path, destination path, reader, or writer.
The Python orchestrator must not parse or rewrite Rust. All Rust-specific
matching, label removal, header composition, wrapper generation, call
resolution, and source rewriting remain in Crat.

## Amendments to earlier behavior

### Executable recognition and target signature

Phase 2 introduced the forced target type. The consolidated final passage is
kept here because Phase 3 changed how the executable form is recognized; its
displayed binding mutability reflects the later Phase 4 presentation rule.

Apply one special target-signature override to a function whose final
identifier's symbol is `main_0` and whose arity is two. Recognition checks
only this name and arity. It does not inspect source safety, parameter names or
types, return type, visibility, ABI, attributes, or body. The supported-input
contract guarantees the following source form:

```rust,ignore
unsafe fn main_0(
    argc: core::ffi::c_int,
    argv: *mut *mut core::ffi::c_char,
) -> core::ffi::c_int
```

Preserve the source parameter types and all other source-signature structure,
apart from the shared presentation mutability normalization. Force the target
parameter types to:

```rust,ignore
unsafe fn main_0(
    mut argc: core::ffi::c_int,
    mut argv: &mut [&mut [i8]],
) -> core::ffi::c_int
```

The `argv` override takes precedence over the ordinary pointer-analysis
decision, including a raw-pointer decision. It does not change the first
parameter's type, the return type, or pointer-analysis behavior for any other
function or binding. A function named `main_0` with arity zero uses ordinary
target-type decisions. Any other `main_0` arity is outside the supported
executable model and receives no special override.

### Function-local item rejection

Before attaching a label to a statement, recursively reject every
`StmtKind::Item` with `GenerationErrorKind::FunctionLocalItem`; the error
identifies the enclosing function path and observed item kind. This includes
local `const` and `static` declarations and items nested at any supported
statement-list depth. Do not label the rejected item or visit its initializer
or body. The excluded `main` remains uninspected because it is omitted before
function-body labeling.

### Changes to skeleton JSON

References from the excluded executable `main` are likewise omitted; its
special `main_0` migration is handled mechanically during Phase 3 rather than
through a dependency edge.

### Validator and replacement integration

Function-local parsed items are not existing declarations in supported target
skeletons: Phase 1 rejects them before producing a record, and Phase 2 rejects
them in both expected skeletons and returned transformations.

Function-header attributes are ignored because item replacement preserves the
current project's attributes instead of accepting attributes from the LLM
header.

The expected-skeleton setup checks add:

- no function-local item declaration of any kind; and

Report the first setup error in deterministic check order. Setup errors abort
result validation because any result diagnostics would be untrustworthy. A
prohibited function-local item in an expected skeleton is
`invalid_expected_skeleton`; the message identifies the function, label when
available, and observed item kind. Scan item statements recursively at every
supported statement-list depth. This setup check defends the Phase 1/2
boundary even though the amended Phase 1 generator no longer emits such a
skeleton.

A supported expected-skeleton signature may contain only named lifetime
parameters without attributes or bounds and may not contain a syntactically
present `where` clause, even an empty one. Phase 1 never generates those
constructs.

The signature checks add:

- complete lifetime-generic declaration;

The generated function must declare exactly the same lifetime parameters as
the target skeleton, in the same order and with the same names. Every generic
parameter must be a lifetime parameter. Added lifetime bounds, type
parameters, and const parameters are rejected. A lifetime parameter may not
carry an attribute, and any syntactically present `where` clause is rejected,
including an empty one. Any difference in this declaration produces one
`generic_parameter_mismatch` for the function. Its message shows the expected
and observed declarations and instructs the LLM to copy the target skeleton's
complete lifetime-generic declaration.

Explicit lifetime names appearing inside parameter and return types remain
part of structural type comparison independently of the complete
lifetime-generic declaration check.

During replacement, Crat uses the validated lifetime-generic declaration,
parameter patterns and types, return type, and body. It ignores the LLM
function's visibility, ABI, safety, `const` qualifier, and attributes, composing
those properties from the current project as specified in Section 16.3.

The body-validation rules add:

- prohibition of every function-local item;

The validator uses stable machine-readable codes and detailed human-readable
messages. The exhaustive initial code set and precedence expectations are
fixed by `phase-2-test-plan.md`, as explicitly amended by Section 4 of
`phase-3-test-plan.md`: add `generic_parameter_mismatch` and remove the four
obsolete existing-item codes listed in Section 14.8. No remaining code is
repurposed.

Use the following updated rows in the Section 14.8 precedence table:

| Area | Precedence and suppression |
| --- | --- |
| Signature | Lifetime-generic declaration; parameter count; parameter names in parameter order; parameter types in parameter order; return type. A generic-declaration mismatch is one `generic_parameter_mismatch` and does not suppress parameter or return checks. A count mismatch suppresses name/type checks for parameter positions that cannot be paired, but not checks for safely paired positions or the return type. |
| Existing declarations | For each target binding in structural preorder, first associate by declaration identity and structural position. One occurrence in a wrong position produces only the applicable location-mismatch error. No occurrence produces `missing_existing_binding`; multiple occurrences produce `duplicate_existing_binding`. For a uniquely associated binding, compare by-value/`ref` mode before explicit-type presence and type. After expected bindings, report result-only bindings and any prohibited function-local items in result source order. |

For a required control group, zero control roots is
`missing_control_root` and more than one is `multiple_control_roots`. A value
path whose complete local identifier matches `proctor_temp_var_n` but resolves
to neither an expected existing local nor a declared generated temporary is
`unresolved_generated_temporary`. Any function-local `StmtKind::Item` in a
returned transformation uses `unexpected_nested_item`; there is no expected
local-item association or structure comparison. Report the item once and do
not descend into its initializer or body for derivative label, binding,
temporary, attribute, or unsafe-block diagnostics. The obsolete
`missing_existing_item`, `duplicate_existing_item`,
`existing_item_location_mismatch`, and `existing_item_structure_mismatch`
codes are removed.

## 16. Item replacement and integration

### 16.1 One-time target-safety normalization

Phase 3 provides a pure source-to-source operation that Phase 4 invokes once,
after skeleton generation and before processing the first SCC. It receives the
original single-file library source and mechanically changes every
source-defined free-function header except `main` to `unsafe fn` in one global
operation. It does not consume skeleton JSON or a target-path list.

The in-memory operation:

```rust,ignore
pub fn normalize_target_safety(
    source: &str,
) -> Result<String, ReplacementError>;
```

recursively traverses the unexpanded surface AST through inline modules and
changes every body-bearing free `ItemKind::Fn` whose final identifier symbol
is not `main`. The name comparison therefore also excludes a surface spelling
of `r#main`. It does not inspect function paths, parameter types, return type,
body, visibility, ABI, attributes, `const`, async, or variadic qualifiers.
Those unsupported forms remain preconditions of the supported-input contract,
not normalization errors. Foreign declarations are not body-bearing free
items and remain unchanged.

Normalization:

- occurs after skeleton generation so `annotated_source` and
  `source_signature` still record original source safety;
- inserts `unsafe` only when it is absent and is idempotent for an already
  unsafe target;
- changes only the safety qualifier of every source-defined free function
  except `main`;
- leaves every excluded `main`, foreign declaration, context item, function
  body, type, ABI, visibility, attribute, and generated Cargo bin source
  unchanged; and
- creates no wrapper merely for a safe-to-unsafe normalization.

The production command writes only this returned source to its requested
`.rs` output path. Phase 4 atomically installs it into the stage-private
current project and builds the resulting normalized initial current project.

Normalizing every target before the first callee-first SCC replacement is
required for incremental compilation. Otherwise, replacing a safe callee with
an unsafe target could make an untransformed safe caller ill-formed. Once all
targets are unsafe, calls among still-untransformed and transformed targets may
occur directly inside unsafe functions. Phase 4 runs `cargo build` on this
normalized initial current project before beginning SCC processing.

### 16.2 Versioned replacement request

After structural validation succeeds, the orchestrator passes Crat the current
single-file library source and this typed request:

```rust,ignore
#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
#[serde(deny_unknown_fields)]
pub struct ReplacementRequest {
    pub schema_version: u64,
    pub items: Vec<ReplacementItem>,
    pub transformation: String,
}

#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
#[serde(deny_unknown_fields)]
pub struct ReplacementItem {
    pub id: u64,
    pub path: String,
    pub name: String,
}

pub fn replacement_request_from_json(
    input: &str,
) -> Result<ReplacementRequest, ReplacementError>;
```

Its exact JSON form is:

```json
{
  "schema_version": 1,
  "items": [
    {
      "id": 12,
      "path": "foo::bar",
      "name": "bar"
    }
  ],
  "transformation": "unsafe fn bar(...) { ... }"
}
```

`schema_version` and every `id` are JSON integers in the Rust `u64` range.
Version `1` is the only supported version. Reject unknown fields, an empty
`items` list, duplicate IDs, duplicate paths, duplicate names, empty paths,
and disagreement between `path`'s final identifier and `name`. Both fields use
the immutable skeleton's exact identifier spelling, including `r#` prefixes.
The transformation must parse as a crate containing exactly one top-level free
function for every requested name, with no duplicate, missing, or additional
function and no other top-level item. The request order is the stable
diagnostic and replacement order; transformation function order is irrelevant.
These checks are defensive integration checks even though the transformation
has already passed the Phase 2 validator.

The pure JSON parser performs no I/O and reports malformed JSON, unknown
fields, invalid integer representations, and request-metadata violations as
`InvalidRequest`. Phase 3 tests call it directly; the CLI uses the same parser.

Phase 3 replaces only functions, while `ReplacementRequest`,
`ReplacementItem`, the `item_replacer` module, and `replace_items` use item
terminology so that later phases can extend the operation without renaming its
public surface. The in-memory API is:

```rust,ignore
pub fn replace_items(
    source: &str,
    request: &ReplacementRequest,
    tcx: TyCtxt<'_>,
) -> Result<String, ReplacementError>;
```

`tcx` must have been created by compiling exactly `source`; it is used for
full-path identity and call-target resolution. The operation performs no
filesystem I/O. Use this deliberately small debugging-oriented taxonomy:

```rust,ignore
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum ReplacementErrorKind {
    InvalidRequest,
    InvalidTransformation,
    TargetResolution,
    UnsupportedConversion,
    UnsupportedCallRewrite,
    RewriteFailure,
}

#[derive(Debug, Clone, PartialEq, Eq)]
pub struct ReplacementError {
    pub kind: ReplacementErrorKind,
    pub item: Option<ReplacementItem>,
    pub message: String,
}
```

The item field supplies the requested ID/path/name when one item is
responsible, and the message describes the concrete failure. Do not add
fine-grained stable subcodes or a validator-style total error precedence:
replacement errors are unexpected integration/debugging failures and are not
sent to the LLM for repair. Return the first failure found by the deterministic
request checks and atomic algorithm documented here, iterating requested items
in request order. The production `replace` command exits nonzero and does not
write a usable output source file. These errors are not Phase 2 LLM-validation
failures and do not use the validator response schema. An incompilable but
successfully emitted replacement is instead handled by the ordinary
compiler-diagnostic LLM repair loop.

### 16.3 Atomic replacement algorithm

Resolve and validate the complete transaction against the unchanged current
source before mutating its AST:

1. Parse the unexpanded surface source and recursively map its items and
   expressions to the HIR compiled in `tcx`, using the same unexpanded
   `AstToHirMapper` discipline as skeleton generation and skipping
   automatically derived HIR items.
2. Resolve every requested full path to exactly one current source-defined
   free function. Require its final name to match the request, require it to be
   already `unsafe`, and reject `const`, async, and variadic functions or a
   function carrying both `#[no_mangle]` and `#[export_name = "..."]`.
3. Match every parsed transformation function by name. Recheck the supported
   lifetime-only generic form, plain identifier parameters, parameter count
   and names, and nonvariadic, nonasync form expected from validation.
4. Compare each current parameter and return type to its transformation
   counterpart to decide whether a compatibility wrapper is required. Use the
   same structural type normalization as the Phase 2 validator: ignore spans,
   node IDs, token caches, formatting, and redundant parentheses, but not real
   type structure. Parameter binding mutability does not cause a wrapper.
   Before conversion validation, identify the executable exemption solely by
   the requested function's final identifier symbol `main_0` and arity: arity
   zero leaves the sibling `main` unchanged, while arity two receives the
   explicit no-wrapper migration. Do not inspect parameter types, return type,
   safety, visibility, ABI, attributes, or body to classify these two forms.
5. Allocate all required same-module wrapper names and validate every required
   source/target conversion before emitting or changing any AST node.
6. Resolve every current direct call expression against HIR and snapshot which
   calls target a requested function. Calls inside requested SCC functions are
   not redirected. If an external call that requires redirection originates
   inside a source macro invocation's token input, return an error because the
   unexpanded surface AST cannot rewrite it reliably.
7. Rewrite the snapshotted external calls that require a wrapper, replace all
   requested functions, insert all wrappers, and perform the executable
   `main_0` migration when applicable.
8. Pretty-print the complete crate and return it only after the whole
   transaction succeeds.

This ordering prevents the generated wrapper's own call to the transformed
implementation from being mistaken for an external source call. Any error
leaves the caller's source string untouched and returns no partial output.
Previously inserted same-module wrappers are ordinary preserved items; a
later SCC transaction does not reconstruct or delete them.

For each requested function, compose the transformed implementation as
follows:

- preserve the current function's attributes except for ABI/export movement
  required by a wrapper;
- preserve its visibility and already-normalized `unsafe` qualifier;
- ignore the transformation function's attributes, visibility, ABI, safety,
  and `const` qualifier;
- take the validated lifetime-generic declaration, parameter patterns and
  types, return type, and body from the transformation;
- remove every validated `#[proctor(N)]` statement label recursively from the
  replacement body; and
- preserve no label in the emitted implementation or wrapper.

When no wrapper is required, preserve the current ABI, `#[no_mangle]`,
`#[export_name = ...]`, and all other current function attributes on the
implementation. When a wrapper is required, Section 17.3 defines the only ABI
and export-attribute movement. This header composition deliberately preserves
the original Rust API visibility and the Phase 3 safety normalization while
using the exact signature already checked by Phase 2. It does not copy a
complete LLM-written function item into the project.

### 16.4 Executable `main_0` migration

The excluded safe library `main` is not an LLM transformation target. When an
SCC target's final identifier symbol is `main_0`, classify it using only its
arity. At arity zero, leave the co-located sibling item whose final identifier
symbol is `main` unchanged: it already calls an unsafe `main_0` inside its
existing unsafe block.

At arity two, replace that co-located sibling `main` mechanically in the same
replacement transaction.
Do not generate a compatibility wrapper for the forced `main_0` `argv`
conversion: the supported `main` is migrated directly, and another
untransformed caller of this `main_0` form is unsupported.
Do not inspect either function's parameter types, return type, safety,
visibility, ABI, attributes, or body when recognizing the pair. The
supported-input model guarantees the two exact source forms in
[Section 2.1](prototype-plan-misc.md#21-supported-program-model).
Any other `main_0` arity is unsupported and receives no executable migration.
Use this fixed function:

```rust
pub fn main() {
    let mut command_line_arg_storage: Vec<Vec<i8>> = ::std::env::args()
        .map(|arg| {
            ::std::ffi::CString::new(arg)
                .expect("Failed to convert argument into CString.")
                .into_bytes_with_nul()
                .into_iter()
                .map(|byte| byte as i8)
                .collect()
        })
        .collect();

    let argc = command_line_arg_storage.len() as core::ffi::c_int;
    let mut command_line_arg_slices: Vec<&mut [i8]> = command_line_arg_storage
        .iter_mut()
        .map(|arg| arg.as_mut_slice())
        .collect();

    let mut argv_terminator: [i8; 0] = [];
    command_line_arg_slices.push(&mut argv_terminator);

    unsafe {
        ::std::process::exit(
            main_0(argc, command_line_arg_slices.as_mut_slice()) as i32,
        )
    }
}
```

The owned inner vectors retain each argument's trailing NUL. The final empty
slice is the prototype representation of the C `argv[argc] == NULL` sentinel;
`argc` excludes that sentinel. This convention is intentionally fixed and may
not preserve programs that require other `argv` behavior; those programs are
outside the supported input model.

The generated Cargo bin shim remains unchanged because it continues to call
the safe library `main`. The hard-coded `main` body is trusted
project-integration code: its explicit unsafe block, local names, and structure
are not subject to Phase 2 validation or LLM temporary-name rules. The
supported executable model assumes the excluded `main` is the only
untransformed caller that must be migrated for this forced `main_0` signature.

## 17. Wrapper generation

### 17.1 Wrapper creation

Create a wrapper only when a function's source and target parameter or return
types differ. Source-safe versus target-unsafe normalization alone never
creates a wrapper. The forced two-argument `main_0` conversion is also exempt:
its only supported caller, the excluded `main`, is replaced directly as
specified in Section 16.4.

The transformed implementation:

- remains at the original Rust path;
- preserves the original visibility and already-normalized `unsafe` qualifier;
- uses the validated target lifetime-generic declaration, parameters, and
  return type; and
- uses Rust's default ABI when a wrapper is required.

The wrapper:

- is inserted as a sibling in exactly the same module as the transformed
  implementation;
- preserves the implementation's original visibility;
- is always unsafe;
- uses the source parameter and return types and the source ABI;
- delegates to the transformed implementation through its absolute path, such
  as `crate::foo::bar(...)`, rather than an unqualified call that a wrapper
  parameter could shadow;
- is an intentionally unsafe compatibility boundary.

The implementation and wrapper therefore do not widen a private,
`pub(super)`, or `pub(crate)` Rust API. Any caller that could access the
original function can access its sibling wrapper through the same module
boundary. The wrapper is unsafe even if the original function was safe,
because normalization has already made all transformation targets and their
untransformed Rust callers unsafe. The excluded executable `main` is the
deliberate safe exception.

### 17.2 Same-module wrapper name

The base wrapper identifier is:

```text
__proctor_wrapper_<original-final-identifier>
```

For example, a transformed `foo::bar` receives a sibling whose absolute path
begins as:

```text
crate::foo::__proctor_wrapper_bar
```

Treat the name of every sibling item as occupied, regardless of Rust namespace.
If the base identifier is occupied, try `<base>_0`, then `<base>_1`, and so on,
selecting the first identifier not occupied or reserved in that module.
Allocate names in replacement-request order against the complete unchanged
module and names already reserved by the same transaction, so multiple
wrappers added together cannot collide. Raw source identifiers use their
identifier symbol when forming the base. When adding wrappers for a new SCC,
preserve wrappers created for earlier SCCs.

### 17.3 Export handling

When no wrapper is required, the implementation preserves its current ABI and
all current export attributes.

When a wrapper is required:

- remove the explicit source ABI from the transformed implementation so it
  uses Rust's default ABI;
- apply that exact source ABI to the wrapper;
- remove `#[no_mangle]` and `#[export_name = ...]` from the implementation;
- if the source had `#[no_mangle]`, add
  `#[export_name = "<original-final-identifier>"]` to the differently named
  wrapper so the external symbol remains exact; and
- if the source had `#[export_name = "..."]`, move that exact attribute to the
  wrapper.

A source function carrying both `#[no_mangle]` and `#[export_name = "..."]` is
unsupported. Reject it before replacement rather than choosing one attribute
or emitting two wrapper export names.

All other current attributes remain on the transformed implementation and are
not copied to the wrapper. The transformation function's ABI and attributes
are ignored. These rules preserve an explicit ABI even when there is no export
attribute, because the wrapper is the compatibility entry point whenever the
Rust parameter or return types change.

The prototype considers only executable and `cdylib` targets.

### 17.4 Conversion logic

For each wrapper parameter, convert the source-signature value into the target
type independently. Let `p` be the wrapper parameter and `T` the pointee type:

| Target parameter type | Source raw-pointer conversion |
| --- | --- |
| `&T` | `&*(p as *const T)` |
| `&mut T` | `&mut *(p as *mut T)` |
| `Option<&T>` | `(p as *const T).as_ref()` |
| `Option<&mut T>` | `(p as *mut T).as_mut()` |
| `&[T]` | if `p.is_null()`, `&[]`; otherwise `std::slice::from_raw_parts(p as *const T, 1_000_000)` |
| `&mut [T]` | if `p.is_null()`, `&mut []`; otherwise `std::slice::from_raw_parts_mut(p as *mut T, 1_000_000)` |
| `Box<T>` | `Box::from_raw(p as *mut T)` |
| `Option<Box<T>>` | if `p.is_null()`, `None`; otherwise `Some(Box::from_raw(p as *mut T))` |
| raw pointer | preserve it when structurally equal, or cast to the exact target raw-pointer type |

An input conversion to `Box<[T]>` or `Option<Box<[T]>>` is unsupported in
Phase 3 and fails the complete replacement transaction before any output is
emitted. A nonpointer parameter whose source and target types are structurally
equal is passed through unchanged. Any other source/target pair is an
unsupported conversion.

The wrapper calls the implementation once. If the target function returns a
value, bind that call result once before converting it to the source return
type. Let `value` be that single target result:

| Target return type | Source raw-pointer conversion |
| --- | --- |
| `&T` | first cast `value` to `*const T`, then cast that raw pointer to the exact source raw-pointer type |
| `&mut T` | first cast `value` to `*mut T`, then cast that raw pointer to the exact source raw-pointer type |
| `Option<&T>` | `None` becomes a typed null matching the exact source pointer mutability; `Some(value)` first casts the reference to `*const T`, then to the exact source type |
| `Option<&mut T>` | `None` becomes a typed null matching the exact source pointer mutability; `Some(value)` first casts the reference to `*mut T`, then to the exact source type |
| `&[T]` | if empty, return a typed null matching the exact source pointer mutability; otherwise cast `value.as_ptr()` to the exact source type |
| `&mut [T]` | if empty, return a typed null matching the exact source pointer mutability; otherwise cast `value.as_mut_ptr()` to the exact source type |
| `Box<T>` | cast the `*mut T` from `Box::into_raw(value)` to the exact source type |
| `Option<Box<T>>` | `None` becomes a typed null matching the exact source pointer mutability; `Some(value)` casts the `*mut T` from `Box::into_raw(value)` to the exact source type |
| `Box<[T]>` | if empty, drop the box and return a typed null matching the exact source pointer mutability; otherwise cast the `*mut T` from `Box::leak(value).as_mut_ptr()` to the exact source type |
| `Option<Box<[T]>>` | `None` and `Some(empty)` return a typed null matching the exact source pointer mutability, dropping an empty box; `Some(nonempty)` casts the `*mut T` from `Box::leak(value).as_mut_ptr()` to the exact source type |
| raw pointer | preserve it when structurally equal, or cast to the exact source raw-pointer type |

Every null branch uses `std::ptr::null()` for an exact `*const T` source return
and `std::ptr::null_mut()` for an exact `*mut T` source return, followed by a
cast when needed to reproduce the exact source spelling/type. Do not cast a
shared reference directly to `*mut T`: first obtain `*const T` and then cast
the raw pointer. Likewise, first obtain `*mut T` from a mutable reference,
mutable slice, or box before any cast to an exact `*const T` source return. A
nonpointer return whose source and target types are structurally equal passes
through unchanged. Any other pair is unsupported. The unit return requires no
temporary.

These conversions are deliberately unchecked. In particular, nonoptional
references and `Box<T>` do not test nullity; their callers must satisfy Rust's
nonnull, validity, ownership, and aliasing contracts. A null slice input maps
to an empty slice, and an empty slice-like return maps to null. The fixed
provisional slice bound is exactly `1_000_000`. Phase 3 does not introduce
`Option<&[T]>` or `Option<&mut [T]>`; distinguishing nullable slices is
deferred.

Global-variable wrappers are not needed because global types are unchanged.

## 18. Call-site rewriting

When replacing an SCC, Crat rewrites direct calls to each wrapper-requiring SCC
function.

Rewrite only callers outside the current SCC.

Use an absolute wrapper path:

```rust
crate::foo::__proctor_wrapper_bar
```

Crat must:

- leave calls between functions in the same SCC unchanged;
- leave calls to signature-unchanged functions unchanged;
- use HIR resolution rather than path spelling to identify the callee, so
  unqualified, imported, aliased, `self`, `super`, `crate`, and fully
  qualified direct calls are handled consistently;
- replace the complete callee expression with the allocated absolute wrapper
  path while preserving the call's arguments and surrounding expression; and
- rewrite all statically resolved direct calls from external untransformed
  callers, including multiple calls and calls nested in ordinary expressions
  or control flow.

A required external call written inside an expression- or statement-macro
token input is an error, not a silently stale call. Such source is already
excluded by
[Section 2.1](prototype-plan-misc.md#21-supported-program-model). Calls
introduced only by macro expansion do not have a
source call expression to rewrite and do not violate this rule.

Because SCCs are processed callee-first:

- all callees outside the current SCC have already been transformed;
- external callers of the current SCC are still untransformed;
- those external callers temporarily call wrappers;
- when an external caller is later transformed, its whole function is replaced;
- its LLM-generated body directly uses target signatures.

No separate transformed-function registry is required.

The immutable skeleton JSON may contain stale source snippets after earlier call-site rewriting. This is acceptable because each function is later replaced as a whole using its original annotated source and target skeleton.

## Implementation sequence

### Phase 3: Item replacement and integration

Implement and test:

- an `item_replacer` module with a typed versioned `ReplacementRequest`,
  structured `ReplacementError`, the in-memory `replace_items` operation, and
  focused tests in `item_replacer/tests.rs`;
- the completed-Phase-1 generator amendment that rejects every
  function-local `StmtKind::Item`, plus removal of obsolete local-const/static
  generator tests;
- the completed-Phase-2 validator amendment that rejects every
  function-local item in expected skeletons and returned transformations, plus
  removal of obsolete local-item preservation logic, diagnostics, and tests;
- the one-time in-memory safety normalization of every source-defined free
  function except `main` after skeleton generation, without skeleton JSON or
  a path list;
- thin `normalize-safety` and `replace` CLI wiring that each writes only one
  `.rs` output file, without filesystem or CLI tests;
- function replacement by full path;
- preservation of the current function's visibility and normalized safety,
  validated target lifetime generics/parameters/return/body, and the exact
  header-composition rules in Section 16.3;
- recursive label removal;
- fixed mechanical replacement of the excluded executable `main` when
  committing the supported two-argument `main_0`, while leaving the
  zero-argument case and generated bin shim unchanged;
- same-module, collision-free wrapper generation;
- base/`_0`/`_1` wrapper-name allocation in request order across every sibling
  item namespace;
- absolute wrapper-to-implementation delegation;
- no wrapper for safety-only source/target differences;
- no wrapper for the forced two-argument `main_0` conversion;
- ABI, `no_mangle`, and `export_name` movement;
- rejection of simultaneous source `no_mangle` and `export_name`;
- every supported input and output conversion in Section 17.4, including the
  fixed slice bound and empty-slice/null policy;
- rejection of boxed-slice input conversion and all other unsupported pairs
  before output;
- HIR-resolved external call-site rewriting to absolute same-module wrapper
  paths;
- preservation of direct and mutual-recursive calls inside the SCC;
- rejection of a required rewrite inside macro-invocation input; and
- atomic failure with no partial rewritten source and the coarse
  debugging-oriented `ReplacementErrorKind` taxonomy from Section 16.2.

Implement every case in `phase-3-test-plan.md`. All functional tests use source
strings and in-memory compiler/parser APIs. They perform no filesystem writes,
do not construct projects, and do not invoke the CLI or subprocesses.
