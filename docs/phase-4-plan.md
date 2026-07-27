# Phase 4 Detailed Plan: Python Orchestration

This is a historical implementation plan. Its substantive text was moved
verbatim from the former consolidated `prototype-plan.md`; imperative and
future-tense wording describes the work assigned at the time. New navigation
notes identify where later work changed an earlier component.

See the [historical overview](prototype-plan.md#phase-4-python-orchestration)
and [Phase 4 test plan](phase-4-test-plan.md).

## Historical context

Phase 4 is implemented after the PROCTOR orchestration framework became
available. The Phase 4 implementation therefore uses PROCTOR's stage
envelopes, shared LLM client, prompt library, usage tracker, and reporting
types directly. It does not implement the historical LiteLLM wrapper proposed
below. `phase-4-test-plan.md` is the exhaustive executable contract for Phase
4 and is normative where it supplies Python-level integration detail.

Phase 4 also changes the assumed pipeline boundary. The Rust project received
by local transformation may have passed through Crat's `split` and `bin`
passes and may subsequently have been changed by a non-local transformation.
Local transformation must therefore always run `expand` followed immediately
by `unexpand` as its first source-preparation operation. `split` and `bin` are
not rerun. The generated Cargo binary target is a separate crate and remains
outside local transformation.

Phase 4 makes two intentional amendments inside the existing Crat
implementation: the `expand` cleanup boundary preserves explicit bin targets,
and skeleton presentation normalizes every non-`ref` binding to `mut` in both
source and target renderings while preserving `ref` versus `ref mut` exactly.
These amendments and the required updates to existing in-memory Crat tests are
part of Phase 4 and are specified in this document and
`phase-4-test-plan.md`. Do not edit any earlier phase test plan.
Phase 4 also uses the ordinary `crat` binary for its initial
`expand,unexpand` preparation. This is not a fifth `crat-tool` operation.
Before Phase 4, make the ordinary `Expand` pass crate-aware at its filesystem
cleanup boundary:

- parse every explicit `[[bin]].path` in the input `Cargo.toml` once before
  recursively removing Rust files;
- resolve each path relative to the manifest directory and lexically normalize
  `.` and `..`;
- reject an absolute path or a normalized path that escapes the crate root;
- do not recursively follow symlinked directories during cleanup; preserve an
  explicitly named in-crate bin path itself, including when that path names a
  symlink;
- treat multiple manifest spellings that normalize to the same in-crate path
  as one preserved source;
- preserve every resolved bin-target source byte-for-byte;
- retain the existing preservation of the root `build.rs` and `target/`;
- remove every other obsolete `.rs` file before writing the expanded library
  source; and
- do not modify `Cargo.toml`.

This deliberately covers Crat's supported, self-contained forwarding-bin
layout. Phase 4 does not add support for Cargo auto-discovered binaries,
examples, benches, tests, custom build-script paths, or binary crates with
their own module trees.

## Changes to skeleton presentation

The target skeleton:

- uses the shared presentation binding normalization described below;

Before cloning the annotated source into the target skeleton, apply one shared
presentation-only binding normalization recursively to every function
parameter and binding pattern: force every by-value identifier binding to
`mut`, preserve `ref` and `ref mut` exactly as written, and leave wildcards
unchanged. This covers simple and destructuring `let`, `let-else`, `if let`,
`while let`, `for`, and match-arm patterns. Consequently,
`annotated_source`, `source_signature`, `annotated_skeleton`, and
`target_signature` all use the same normalized non-`ref` mutability. Applying
the normalization before the source-to-skeleton clone is sufficient because
signature targeting and skeletonization do not introduce binding identifiers.
The input project and all analyses continue to use the unchanged source AST.
This presentation normalization is intentional: neither displayed function
should prevent an LLM from assigning to an existing by-value binding while
translating it. Ordinary binding `mut` is not a semantic target decision.
Preserving `ref` versus `ref mut` avoids changing a binding's borrow type. The
Phase 2 validator continues to ignore `mut` everywhere and accepts either its
presence or absence in the returned transformation. It still enforces
by-value-versus-`ref` mode, binding identity, declaration placement, and
target types.
Apply the shared non-`ref` binding-mutability normalization above to both
source and target function renderings. Sanitization and binding normalization
are presentation-only: pointer analysis and the input project always use the
unchanged source, and later project-rewriting operations must recover
ABI/export information from the project AST rather than from the sanitized
JSON.

## 7. Python orchestrator

Implement Phase 4 as the standalone PROCTOR stage
`stages/local-transformation/`, with stage ID `local_transformation`. Follow
the current typed example LLM stage rather than introducing another
orchestration framework. Phase 4 adds this standalone stage and its manifest,
but does not add, remove, reorder, enable, or disable any entry in
`configs/full_pipeline.toml`. Its `stage.toml` contract is:

```toml
id = "local_transformation"
version = "0.1.0"
description = "Transform pointer-local Rust code SCC-by-SCC with validated LLM output."

exec = ["python3", "main.py"]
warmup = ["python3", "main.py", "--build-only"]

[requires]
rust_project = "required"

[produces]
rust_project = true

[config]
crat_dir = { type = "string", default = "../crat", doc = "crat checkout, relative to this stage" }
```

As with `example-llm-stage`, give the stage its own `pyproject.toml` with a
`proctor` dependency sourced from the repository at `../..`, and check in its
`uv.lock`. This lets the orchestrator invoke the standalone stage through its
isolated `uv` environment while the stage directly uses the shared framework.

The dependency-context limit of 100,000 characters and the maximum of ten
repair calls are fixed prototype semantics, not configuration options. Model,
provider, retry, rate-limit, pricing, and provider-specific settings come only
from `stage_input.framework.llm`.

Validate the stage-specific `config` table before other side effects. It may
contain only `crat_dir`. When omitted, use the manifest default `../crat`,
resolved relative to the stage directory in the same manner as the existing
Crat adapter. When supplied, `crat_dir` must be a nonempty string; reject
unknown keys and wrong types with a failure envelope. Report the one effective
resolved setting in `config_used`.

Use `StageInput` and `StageOutput` from `proctor.contracts`. Require a
read-only input `rust_project`, a declared `outputs.rust_project` path that
does not yet exist, and `framework.workdir`. Produce only `rust_project`;
report `rule_set = null`.
Use `framework.workdir` for the current project and all request, response,
candidate, and rollback files. Put a command/diagnostic log under
`outputs.artifacts_dir` when supplied and report its path relative to that
directory in `StageOutput.logs`. When `outputs.artifacts_dir` is absent, keep
the diagnostic log under the work directory for debugging but leave
`StageOutput.logs` empty: the stage contract permits reported logs only
relative to the declared artifacts directory.

The stage is responsible for:

- building both `crat` and `crat-tool` once per Crat commit, using the pinned
  Crat toolchain and the same sysroot/library environment discipline as the
  current `crat-adapter`;
- copying the input project once to a new stage-private current-project
  directory, including an existing root `target/`;
- preparing that copy with ordinary Crat `expand,unexpand` in one in-place
  invocation, with `--unexpand-use-print`;
- invoking `crat-tool` skeleton, safety-normalization, validation, and
  replacement operations;
- loading and checking skeleton JSON without parsing Rust;
- building and scheduling the function SCC graph;
- rendering dependency context and prompts;
- using PROCTOR's LLM client, prompt library, usage tracker, and pricing;
- extracting Rust code blocks without parsing their Rust contents;
- installing one candidate library source transactionally and invoking
  `cargo build`;
- restoring the previous source after failed candidate builds;
- managing the bounded repair loop; and
- copying the final current project to the declared output while excluding
  the root `target/`.

The stage must not parse or rewrite Rust source. Reading `Cargo.toml` with
`tomllib` to obtain the explicit `[lib].path` is project plumbing, not Rust
parsing. Require that value to be a string whose lexically normalized path is
one root-level file inside the crate. Permit optional `.` components but
reject every `..` component; nested, absolute, and crate-escaping library
paths are unsupported and fail before Crat is invoked. The stage never mutates
the input artifact or writes the output destination before every SCC has
succeeded.

### 7.1 Preparation order

Use this exact order:

1. Validate the stage config and envelope paths, refuse an existing output
   destination, parse the input `Cargo.toml`, and validate its `[lib].path` as
   a root-level library source before any build, copy, or tool call.
2. Build or locate `crat` and `crat-tool`.
3. Copy the complete input project to `<workdir>/current`.
4. Run ordinary Crat in-place with passes `expand,unexpand`, the
   `--unexpand-use-print` flag. If the copied project contains a regular
   root-level `config.toml`, pass it with `--config`; otherwise omit
   `--config` and use Crat's defaults.
5. Require the validated library source path to identify a regular file in
   the prepared current project.
6. Run `crat-tool make-skeleton` against the prepared current project.
7. Load the immutable skeleton records.
8. Run `crat-tool normalize-safety` to a scratch `.rs` file and atomically
   install it as the current library source.
9. Run `cargo build` in the current project. Failure here aborts Phase 4; it
   is not an LLM repair opportunity.
10. Build the graph and process all SCCs.
11. After all SCCs succeed, copy current to the output while ignoring only
    the root `target/`.

The Python runner must assemble the four `crat-tool` operations in exactly
these shapes:

```text
<crat-tool> make-skeleton --output <skeletons.json> <current-project>
<crat-tool> normalize-safety --output <normalized.rs> <library-source>
<crat-tool> validate --input <validation-request.json> --output <validation-response.json>
<crat-tool> replace --request <replacement-request.json> --output <candidate.rs> <current-project>
```

Each request/output pathname is stage-private. Before launching an operation
that is expected to create an output, remove any previous scratch output at
that exact path. Success requires both a zero exit status and a newly created
regular output file. A nonzero exit, missing output, non-regular output, or
stale-output reuse is a fatal tool/protocol failure. This applies equally to
skeleton, normalization, validation, and replacement. The ordinary Crat
preparation command is likewise fatal on nonzero exit.

Preparation or integration subprocess failures, malformed skeleton data,
validator `setup_error`, replacement failure, and inability to restore a
source transaction abort the stage immediately. They are tool or invariant
failures, not LLM repair failures.

`proctor.toml`, `Cargo.toml`, `Cargo.lock`, the explicit bin-target sources,
and all non-library files are copied normally and are never rewritten by
Python. Do not require or parse `proctor.toml`; if present, its exact bytes
must reach the final output unchanged. In particular, ignore a nonempty
`wrappers` list. Existing wrapper functions visible in skeleton JSON are
ordinary functions for this prototype because no metadata-aware inclusion or
exclusion is performed.

### 7.2 Shared LLM infrastructure

Use `proctor.llm.client.LlmClient` directly. Construct `UsageTracker` from the
run/stage/item identity and `PricingTable.from_config`, writing to
`framework.usage_log` when supplied and otherwise to
`<framework.workdir>/usage.jsonl`. The fallback keeps direct standalone stage
invocations fully tracked; ordinary PROCTOR runs always supply the official
stage-local usage path. Load the versioned stage-local prompt through
`PromptLibrary`; do not embed an unversioned prompt string in Python.

Make a shallow private copy of `framework.llm` and set its effective
`context_overflow` to `"error"` before constructing `LlmClient`. Do not mutate
the envelope's settings. The current shared client already supports `"error"`
and uses it by default, so Phase 4 requires no framework change for this
policy. A provider `ContextLimitExceeded` therefore records the failed attempt
through the shared tracker and then aborts the entire transformation. It is
never truncated, retried as an SCC repair, or counted against the ten repair
calls. Other LLM errors follow the shared client's provider-retry policy; if
the client ultimately raises, abort the stage.

Each SCC generation is one independent `Request` containing one user message.
No assistant/user history is retained. A repair is another independent
request containing the complete original material plus only the latest failed
text and latest diagnostics. Attach `RequestMetadata` with the run ID, stage
ID, an SCC item string formed by joining the ascending decimal member item IDs
with commas (for example, `0` or `0,3`), and the rendered prompt
ID/version/hash.

Aggregate every usage record, including provider retries and failed calls,
into the final `StageOutput.usage`. Report each distinct provider/model pair
used, in first-observed order, and the one prompt ID/version. A failure output
should still report usage accumulated before failure when it can do so
reliably. Derive the aggregate from the `UsageTracker` records rather than
only from successful `Response` objects: a retrying logical generation may
therefore contribute multiple usage calls. The generation-call metric counts
logical SCC requests, while `StageOutput.usage.calls` counts provider attempts
recorded by the tracker.

Use the framework's reporting convention when aggregating tracker records:

- with no LLM provider attempt, report `usage = null`, `models = []`, and
  `prompts = []`;
- otherwise, sum the integer input, cached-input, and output token fields;
- report `reasoning_tokens = null` when every record has null reasoning usage,
  otherwise sum the nonnull reasoning-token values;
- sum nonnull costs, but report `cost_usd = null` if any token-bearing record
  has unknown cost; a failed zero-token attempt with null cost does not make an
  otherwise known total unknown, and a nonempty collection containing only
  zero-token failed attempts reports `0.0`; and
- report the prompt once whenever at least one logical LLM request was issued,
  including when every provider attempt failed.

Report these exact flat metric keys on success and, where known, on failure:
`function_count`, `scc_count`, `llm_generation_calls`, `repair_calls`,
`structural_failures`, `compilation_failures`, and `cargo_builds`. The Cargo
count includes the normalized initial build. Metrics do not replace the
per-attempt usage log.

### 7.3 Skeleton loading

Represent loaded records with typed Python dataclasses or an equivalent typed
model. Validate only the integration contract needed by Python; do not
duplicate Crat's semantic tests. Require:

- a top-level JSON array;
- integer IDs in the inclusive Rust `u64` range `0..=18446744073709551615`
  (booleans are not integers here);
- globally unique IDs;
- paths unique within the record's dependency namespace: `Fn`, `Static`, and
  `Const` are value-namespace records, while `TyAlias`, `Enum`, `Struct`, and
  `Union` are type-namespace records. Permit one value record and one type
  record to have the same display path, as in the Phase 1 `type X`/`const X`
  regression;
- one of the seven [Section 6](phase-1-plan.md#6-skeleton-json) kinds;
- the [Section 6](phase-1-plan.md#6-skeleton-json) required string and
  dependency-list fields for each kind;
- integer dependency IDs that resolve to an included record; and
- `signature_dependencies` to be a subset of `dependencies` for `Fn`,
  `Static`, and `Const`.

Preserve every Rust text field exactly as decoded from JSON. Sort or deduplicate
nothing while loading: reject duplicate dependency IDs and require Crat's
lists to already be in strictly increasing item-ID order. These checks expose
corrupt or incompatible tool output without retesting how Crat generated the
contents.

## 8. Function graph and SCC scheduling

### 8.1 Function graph

Build a graph containing only transformable `Fn` items.

For each function `f`, inspect function-valued entries in its `dependencies`.

Add an edge:

```text
f -> g
```

when `f` directly calls `g`.

An ID naming a non-function record is not an edge. Foreign functions are
absent from the records and graph. Keep direct self-edges. Traverse function
nodes and adjacency lists in increasing item-ID order so the result does not
depend on JSON object identity or Python set order.

### 8.2 SCCs

Compute strongly connected components and the SCC condensation DAG.

A leaf SCC has no outgoing edge to an unprocessed SCC.

Because edges point from callers to callees, leaf-first processing translates callees before external callers.

A singleton SCC is recursive only when its function has a self-edge. Store
members of each SCC in increasing item-ID order.

### 8.3 Deterministic scheduling

At every scheduling step, recompute or update the set of unprocessed leaf
SCCs. Choose the leaf whose smallest member item ID is smallest. Mark an SCC
processed only after its candidate source builds successfully. This fixes one
exact schedule for disconnected components as well as call chains and cycles.

### 8.4 Function-name uniqueness

Before processing an SCC, the orchestrator checks that the final function names of all SCC members are distinct.

Crat identifies returned functions by name inside the single LLM response. Therefore, uniqueness is required only within the current SCC.

If duplicate function names occur within one SCC, orchestration aborts.

Functions in different SCCs may have the same final name.

Perform this check immediately before the SCC's first LLM request. Use member
item-ID order for the diagnostic. Do not globally reject duplicate final names
in different SCCs.

## 9. Prompt-context construction

Each prompt has two conceptually separate parts:

1. **Transformation Targets**
2. **Dependency Context**

The transformation targets do not count toward the dependency-character limit.

The dependency context has a limit of 100,000 characters.

All dependency records are deduplicated by item ID and ordered deterministically by ID.

Context construction operates on records and strings only. It does not inspect
Rust syntax.

### 9.1 Transformation targets

For every function in the current SCC, include:

- its annotated source function;
- its annotated target skeleton.

These snippets already contain the source and target signatures, so those signatures are not repeated separately.

### 9.2 SCC signatures

If the SCC contains multiple functions, include the source and target signatures of every SCC member in the dependency context.

For a directly recursive singleton SCC, include its own source and target signatures.

For a nonrecursive singleton SCC, omit the redundant self-signature dependency.

Represent an SCC signature dependency with the same function-context rendering
used for an ordinary function dependency. Do not emit an SCC member twice
when it is also present in another member's direct dependency list.

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

Build the union of every target's direct dependencies. Remove IDs belonging to
the current SCC because their required signatures are handled by Section 9.2.
The remaining direct IDs form mandatory depth zero.

Traverse subsequent dependencies breadth-first over the union graph:

- from a `Fn`, `Static`, or `Const`, follow only
  `signature_dependencies`; and
- from a `TyAlias`, `Enum`, `Struct`, or `Union`, follow `dependencies`.

Do not follow the body-only portion of a function, static, or const
dependency. Deduplicate an ID at its shortest depth, and do not traverse back
through an SCC target. Within a depth and in the final rendering, order records
by item ID.

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
3. Render those mandatory entries together in item-ID order.
4. Abort before the LLM call if that rendering exceeds 100,000 Python
   characters.
5. Tentatively add the complete next breadth-first depth.
6. Keep the depth only if the complete re-rendered context is at most 100,000
   characters.
7. Continue until the next complete depth does not fit.
8. Once a depth is rejected, do not consider deeper depths.

If mandatory direct dependencies already exceed 100,000 characters, abort the SCC instead of silently omitting them.

Instructions, transformation targets, prior failed code, and diagnostics are outside this limit.

Count characters with Python `len()` on the fully rendered Unicode string,
including headings, fences, separators, and newlines. An empty dependency
context is the empty string. Join nonempty entries with exactly two newline
characters and add no leading or trailing separator.

Render entries exactly in these shapes, substituting the record's exact text:

````text
### Function <id>: <path>
Source signature:
```rust
<source_signature>
```
Target signature:
```rust
<target_signature>
```
````

````text
### <Static-or-Const> <id>: <path>
```rust
<declaration>
```
````

````text
### <TyAlias-or-Enum-or-Struct-or-Union> <id>: <path>
```rust
<definition>
```
````

Render transformation targets in SCC member-ID order and join them with two
newlines:

````text
### Function <id>: <path>
Source:
```rust
<annotated_source>
```
Target skeleton:
```rust
<annotated_skeleton>
```
````

## 10. Initial LLM prompt template

Store the following exact text as stage-local `PromptLibrary` prompt
`local_transformation`, version `1`, with variables `dependency_context`,
`transformation_targets`, and the single pre-rendered `repair_context`.
The prompt file's frontmatter is exactly:

```toml
+++
id = "local_transformation"
version = 1
description = "Transform one Rust function SCC against Crat skeletons."
variables = ["dependency_context", "transformation_targets", "repair_context"]
+++
```

The exact prompt body begins after that frontmatter:

````text
You are transforming unsafe Rust functions generated from C.

The source code is the original implementation. The target skeleton defines
the transformation goal. A dependency's source signature is its signature
before transformation; its target signature is how transformed code must call
it.

Implement every function in Transformation Targets exactly once. Emit no
other top-level item. Use Dependency Context only as reference; do not emit or
redefine its functions, types, statics, or constants.

Requirements:

1. Exactly preserve source behavior wherever it is defined, including apparent
   bugs. Do not add validation, fallback behavior, or error handling absent
   from the source; preserve its preconditions. For example, if the source
   immediately dereferences a raw pointer, directly unwrap the corresponding
   `Option` instead of adding a conditional check.
2. Use the target skeleton's exact lifetime-generic declaration, parameter
   types, return type, and local-variable types.
3. Call transformed function dependencies using their target signatures.
4. Keep every existing function, parameter, and local-binding name. Preserve
   each existing declaration exactly once in the same label, pattern, and
   control-flow role.
5. Name every new local binding `proctor_temp_var_n`, where `n` is a
   nonnegative integer. Use it only within the consecutive statements carrying
   the same `#[proctor(N)]` label that encloses its declaration, including
   unlabeled code nested within those statements.
6. Do not define a function, type, static, constant, module, or other item
   inside a transformation target.
7. At each existing statement-list level, preserve every source
   `#[proctor(N)]` label in order. A labeled statement may expand only into one
   or more consecutive sibling statements with the same label. Do not insert
   unlabeled siblings at that level, repeat a label in nested code, or label
   newly introduced nested statements.
8. Preserve each existing control form, its direct role, its
   branch/arm/guard/body structure, and all existing nested labels. Plain
   blocks, `if`, `if let`, `while`, `while let`, `for`, `loop`, and `match`
   are distinct. A `let ... else` must remain a `let ... else`. A control form
   used directly as a `let` initializer, `return` value, `break` value, or
   match-arm result must remain in that role. Conditions, scrutinees, patterns,
   and statement contents may be rewritten.
9. If a labeled statement containing a control form expands into multiple
   same-label siblings, exactly one sibling must preserve that form, role, and
   all its existing labeled nested statements. Other siblings must not have a
   control form in that same direct role and must contain no labels below their
   own group label.
10. Do not introduce an explicit `unsafe` block or a statement or expression
    attribute other than the required `#[proctor(N)]` labels.
11. Return exactly one Rust code block delimited by triple-backtick fences.
    Include all requested functions and no prose. Do not use tilde or
    longer-backtick fences.

Example:

Source:

```rust
unsafe fn read_value(mut p: *const i32, mut q: *const i32) -> i32 {
    #[proctor(0)]
    let mut x: i32 = *p.add(1);
    #[proctor(1)]
    return if q.is_null() {
        #[proctor(2)]
        x
    } else {
        #[proctor(3)]
        x + *q
    };
}
```

Target skeleton:

```rust
unsafe fn read_value(mut p: &[i32], mut q: Option<&i32>) -> i32 {
    #[proctor(0)]
    let mut x: i32 = todo!();
    #[proctor(1)]
    return if todo!() {
        #[proctor(2)]
        todo!()
    } else {
        #[proctor(3)]
        todo!()
    };
}
```

Valid output:

```rust
unsafe fn read_value(mut p: &[i32], mut q: Option<&i32>) -> i32 {
    #[proctor(0)]
    let mut x: i32 = p[1];
    #[proctor(1)]
    return if q.is_none() {
        #[proctor(2)]
        x
    } else {
        #[proctor(3)]
        x + *q.unwrap()
    };
}
```

Dependency Context:

{{ dependency_context }}

Transformation Targets:

{{ transformation_targets }}

{{ repair_context }}
````

For a repair request, use:

````text
The previous transformation failed.

Previous transformation:

```rust
<latest failed text>
```

Diagnostics:

```text
<latest diagnostics>
```

Regenerate every function in Transformation Targets.
````

The initial request renders `repair_context` as the empty string. A repair
renders exactly the block above, with only the latest failed text and latest
diagnostics substituted into the complete original prompt. Every render goes
through `PromptLibrary`, and the request metadata records prompt ID
`local_transformation`, version `1`, and the rendered content hash.

## 11. LLM response extraction

The orchestrator instructs the LLM to return exactly one Rust code block
delimited by triple backticks and no prose. Only triple-backtick fences are
recognized; tilde fences and fences of four or more backticks are not blocks.

A recognized opening fence starts in column zero and is exactly three
backticks followed either by no tag or immediately by one nonempty ASCII
language tag containing only letters, digits, `_`, `+`, or `-`. Nothing else
may occur on that line. A recognized closing fence starts in column zero,
contains exactly three backticks and no other character, and ends at the next
line ending or end of response. Accept LF and CRLF as fence-line endings.
Indented fences, inline fences, opening tags containing whitespace or
punctuation outside that set, closing fences with trailing whitespace, and
unclosed fences are not recognized. Pair recognized opening and closing
fences from left to right without overlap.

To tolerate formatting errors:

1. Find all triple-backtick fenced code blocks.
2. Ignore prose outside code blocks.
3. If one block exists, use it.
4. If multiple blocks exist, choose the longest.
5. If multiple longest blocks have equal length, choose the first.
6. If no fenced code block exists, report a structural failure.

Pass the selected block unchanged to Crat.

The orchestrator does not parse Rust.

Measure block length by the number of content characters, excluding the
opening/closing fence, optional language tag, and exactly one LF or CRLF that
separates the content from each fence. Preserve every other character and line
ending exactly; do not otherwise strip or normalize the content. If no block
exists, the raw LLM response becomes the latest failed text and the
deterministic diagnostic is:

```text
The LLM response contained no triple-backtick fenced code block; return exactly one triple-backtick fenced Rust code block.
```

This is an ordinary repairable response-format failure.

## 15. SCC transaction policy

Each SCC is all-or-nothing.

No function in the SCC is committed until:

1. every function passes Crat structural validation; and
2. the current project with the complete SCC candidate installed passes
   `cargo build`.

If either step fails:

- restore the exact pre-attempt library source when a candidate had been
  installed;
- leave the current project source at the last successfully promoted SCC;
- discard partial success within the SCC; and
- regenerate every function in the SCC.

The current project's `target/` is an incremental Cargo cache and is not part
of the source transaction. A failed build may update it. Cargo fingerprints
the next build against the actually installed source, so Phase 4 retains the
cache rather than copying or rolling it back.

## 19. Compilation and promotion

After Crat emits the replacement `.rs` file, Phase 4 uses a source-file
transaction in the one stage-private current project:

1. Keep the candidate outside the project tree.
2. Copy the exact current library source to a rollback path outside the
   project tree.
3. Atomically replace the current library source with the candidate.
4. Run:

```bash
cargo build
```

in the current-project directory.

The rollback copy must not have an `.rs` path inside the Cargo project. Wrap
installation and building in `try/finally`: every unsuccessful or exceptional
attempt after installation restores the previous source atomically. Failure
to restore is a fatal stage error. The stage's input artifact remains
read-only throughout, and a killed stage can damage only disposable work
state; a fresh invocation recreates `<workdir>/current` from the input rather
than resuming an uncommitted swap.

### Success

- Keep the installed candidate as the current library source.
- Delete the rollback copy.
- Mark the SCC as processed.
- Select the next leaf SCC.

### Failure

- Capture compiler standard output and standard error.
- Restore the rollback source.
- Keep all other current-project files unchanged. Retain `target/`.
- Start a repair attempt for the entire SCC.

The prototype assumes Crat's integration routines are correct. A nonzero or
malformed `replace` operation aborts rather than asking the LLM to repair an
integration failure. Compiler diagnostics are given to the LLM even when they
refer outside the SCC. The LLM may change only SCC functions. If that is
insufficient, the retry limit eventually aborts orchestration.

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
3. If valid, ask Crat to emit a replacement `.rs` file.
4. Transactionally install it in the current project and run `cargo build`.
5. Keep it on success or restore and retry on failure.

If the SCC has not succeeded after ten repair calls, abort the complete orchestration immediately.

The initial call plus ten repair calls means at most eleven LLM generation
calls per SCC, excluding provider-level retries internal to `LlmClient`.
Increment the repair count before each repair LLM call. Structural
`invalid`, missing-fence extraction, and failed candidate compilation consume
the same counter. Validator `setup_error`, validator process/protocol failure,
replacement failure, preparation failure, safety-normalization failure,
initial normalized-project build failure, malformed skeleton data, exhausted
provider retries, authentication failure, and context overflow abort
immediately and do not consume an SCC repair.

For structural invalidity, use the complete raw text of the validator response
file, unchanged after it has been parsed successfully, as the latest
diagnostics and use the selected code block as the latest failed
transformation. Do not reserialize the parsed JSON: whitespace, key order, and
the presence or absence of a final newline in the tool's response are part of
the diagnostic text sent on that repair. For a missing fence, use the complete
raw response as failed text and the deterministic extraction diagnostic. For
compilation failure, use the selected transformation and both captured
streams, labeled `cargo build stdout:` and `cargo build stderr:`. Do not
truncate diagnostics; if the resulting provider request exceeds its context
window, the required `context_overflow = "error"` behavior aborts honestly.

## Implementation sequence

### Phase 4: Python orchestration

Implement:

- `stages/local-transformation/` as a typed standalone PROCTOR stage requiring
  and producing only `rust_project`;
- Crat/crat-tool warmup and process invocation with deterministic logs;
- one initial immutable input-project copy, retaining `target/` when present;
- mandatory in-place Crat `expand,unexpand` preparation without `split` or
  `bin`;
- the ordinary Crat `expand` cleanup correction that preserves every explicit
  `[[bin]].path`;
- the Crat skeleton-presentation amendment that forces every non-`ref`
  binding to `mut` in both source and target snippets/signatures while
  preserving `ref` and `ref mut` exactly;
- corresponding updates to the existing in-memory Crat skeleton tests, with no
  filesystem or CLI test added;
- strict skeleton JSON integration loading;
- one-time invocation of Phase 3 target-safety normalization and compilation of
  the normalized initial current project;
- SCC-local function-name uniqueness checks;
- function graph construction;
- SCC computation;
- deterministic leaf scheduling;
- exact dependency-entry and transformation-target rendering;
- dependency-context rendering;
- breadth-first type closure;
- 100,000-character dependency budget;
- a versioned stage-local PROCTOR prompt;
- PROCTOR `LlmClient`, `UsageTracker`, pricing, request metadata, and
  StageOutput aggregation;
- mandatory effective `context_overflow = "error"`;
- code-block extraction;
- validation invocation;
- item replacement to a scratch source;
- atomic candidate-source installation, `cargo build`, and rollback;
- repair accounting;
- source promotion without per-attempt project copies;
- byte-preserving `proctor.toml` and non-library project handling; and
- final output copying with root `target/` excluded.

Implement every case in `phase-4-test-plan.md`. Keep default Python tests
offline and independent of the real Crat toolchain, Cargo, and provider APIs.
Phase 4 adds no filesystem-changing Crat test. It updates existing in-memory
Crat skeleton tests only for the presentation-mutability amendment. Python
tests assert command construction, protocol handling, graph/context logic,
transactions, and stage reporting without revalidating skeleton, validator,
replacement, or Expand cleanup semantics.
