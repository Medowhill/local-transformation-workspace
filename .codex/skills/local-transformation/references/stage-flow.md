# Local-Transformation Stage Flow

All implementation paths are relative to the standalone PROCTOR root
(`./proctor` in this workspace). Current code and focused tests take precedence
over this reference. Short Python filenames below refer to
`stages/local-transformation/`; `work/` denotes `framework.workdir`.

## Ownership map

| Surface | Responsibility |
| --- | --- |
| `stages/local-transformation/main.py` and `stages/local-transformation/stage.toml` | Standalone `--input`/`--output` entrypoint, `--build-only` warmup, stage contract and defaults |
| `stages/local-transformation/stage.py` | Boundary checks, dependency normalization, SCC workers, acceptance/repair, metrics, final publication |
| `stages/local-transformation/model.py` | Strict skeleton and replacement-metadata loading, graph/SCC construction, dependency-context rendering |
| `stages/local-transformation/protocol.py` and `stages/local-transformation/prompts/local_transformation.md` | Requests, tool argument vectors, prompt rendering, fenced-response extraction |
| `stages/local-transformation/libc_guidance.py` and `stages/local-transformation/printf_guidance.py` | Advisory guidance selected from compiler-provided member metadata |
| `stages/local-transformation/tooling.py` | Tool builds/cache/environment, thin subprocess wrappers, output checks, Cargo and rollback transactions |
| `stages/crat/crates/tools/src/` and `stages/crat/src/bin/crat-tool.rs` | Compiler-aware generation, validation, freezing/assembly, observations, and reusable rules |
| `tests/test_local_transformation.py` | Python stage behavior with fake tools and clients, concurrency, contracts, failure/publication checks |

Do not implement Rust parsing, type analysis, or observation/rule semantics in
Python. Conversely, keep scheduling, provider repair, build policy, and stage
artifacts out of the compiler library. Shared LLM, usage, contract, and
orchestration infrastructure remains in the `proctor/` package.

## Preparation and immutable analysis

The entrypoint always writes a stage output envelope on a handled failure.
`run_stage` first validates paths/configuration and loads `proctor.toml`. The
manifest must explicitly include `api_functions` and an empty `wrappers` list;
nonempty `wrappers` and a library API named `main` are rejected before tool work.
The destination project must not exist. Artifact and usage destinations must
not overlap protected input or scratch boundaries.

The stage builds or reuses release `crat` and `crat-tool` binaries, then copies the
read-only project into `work/analysis`. `tooling.py` keys its build cache from
`HEAD` plus relevant source/manifest content, including uncommitted source
changes; existence of a binary alone is insufficient. It invokes a plain
`cargo build` in `stages/crat/deps_crate/` before the release binaries and sets
Crat's rustc-private runtime environment. Crat's compiler helper additionally expects
host-target dependencies under
`stages/crat/deps_crate/target/<host>/debug/deps`; use the `$crat` skill's toolchain/setup guidance rather than assuming this plain build
provisions that path. The build-only warmup uses the stage's sibling Crat
checkout; an invocation's `crat_dir` is resolved by normal stage configuration.

Dependency normalization ensures crates.io minimums declared by
`REQUIRED_CRATES_IO_DEPENDENCIES` in `stage.py`: `bytemuck` 1.25.2, `xj_scanf` 0.2.6,
and `proctor-libc` 0.3.0. It adds or raises canonical requirements while retaining
other dependency table fields. Path, git, workspace, and alternate-registry
dependencies remain untouched. Invalid requirement strings normalize to the
minimum; malformed dependency/table shapes and conflicting package aliases
fail before preparation.

Ordinary Crat `expand` and `unexpand` run adjacently on the analysis copy, with
`--unexpand-use-print`, consolidating external modules into the declared
root-level library source. This stage-local rerun is distinct from the
upstream Crat pass chain. The local pipeline configurations select `prepare`
between `enum` and `simpl` upstream; the local stage does not itself run `prepare`.
That ordinary pass normalizes non-block match arms and function-local statics,
and handles its supported fixed-C-locale ctype/`setlocale` patterns. Use `$crat`
for its exact rewrite and dependency policy.

One `make-skeleton` compiler session emits both skeleton views and
`frozen-project.json`, including original IDs, normalized analysis source,
target signatures, complete non-function context, and observation helpers.
Safety normalization makes every source-defined free function except `main`
`unsafe`. The stage then creates `work/current` from the analysis project and
installs `make-initial --frozen` output: full retained crate context plus typed
target-signature `todo!()` stubs at the original function paths, with `main` omitted.
The initial `cargo build --lib -j 1` must pass before SCC work.

No full-program analysis is recomputed from a worker's stub crate. The
normalized analysis source and frozen decisions remain immutable throughout
translation.

## SCC execution

`model.py` constructs edges only between function-valued direct dependencies,
retains self-recursion, and computes Tarjan SCCs. Members are sorted by
original item ID and jobs by their smallest member ID. This order stabilizes
dispatch and artifacts; it is not a callee-first requirement. Direct and
mutual recursion are supported. Duplicate final function names are allowed in
different SCCs but fatal within one request.

Each long-lived worker owns a private project, Cargo target/build directory,
client, usage tracker, tool log, and mutable job state. It resets the library
source to the same frozen template for every new SCC while keeping its private
Cargo cache. Only current SCC members receive real implementations; all other
functions retain their target-signature stubs. No job consumes an earlier
accepted body. Both `CARGO_TARGET_DIR` and `CARGO_BUILD_BUILD_DIR` are forced to
the private target directory, overriding inherited or project-configured
shared directories.

Every job begins with its `applied` views. If no selected view has `transform`
holes, the stage uses the skeletons mechanically and skips LLM/validator calls.
Otherwise one LLM request covers every SCC member, including already solved
members. Prompting and subsequent operations must use exactly the same views.
The context and constrained response contract are in
[contracts-and-prompts.md](contracts-and-prompts.md).

After extraction and, for LLM output, structural validation, `add-functions --frozen`
installs the current SCC into the frozen template and writes the
candidate, canonical statement pairs, observation source, and metadata. The
stage cross-checks all sidecars before installation. Unfilled whole-function
stubs are repairable for LLM work and fatal for mechanical work; they must not
reach compilation or observation extraction.

## Acceptance and repair

A candidate is copied into the worker library through a rollback transaction,
then `cargo build --lib -j 1` runs against the private target directory. Failure
or a builder exception restores the previous source; restoration failure is
fatal. A successful build must leave bytes matching the candidate digest.
Only then are canonical statement pairs, accepted source/sidecars, and views
retained. If `transform` labels remain, Crat extracts observations before the
worker takes another SCC. The saved `accepted.rs` precedes extraction, so its
presence alone does not establish SCC success.

One SCC has an initial generation and at most ten repair generations shared
across both views. Missing fenced code, invalid returned structure, unchanged
whole-function stubs, and ordinary candidate-build failures are repairable.
Only the latest failed transformation and diagnostics are supplied to the next
complete SCC prompt; failure history does not accumulate.

A build failure involving `rule_applied` statements rolls back and permanently
switches every SCC member to `baseline`, consuming the same existing budget.
The stage never returns to `applied` views. A failed mechanical candidate with
no rule application is fatal. Tool/setup/protocol errors, terminal provider
errors, context overflow, extraction failure, and initial/final build failure
are also fatal; they do not become source-repair requests.

An SCC failure stops further dispatch while in-flight jobs finish. Reporting
waits for those jobs and their usage. The stage has no per-SCC checkpoint or
resume protocol and no whole-project repair after final integration fails.

## Assembly, learning, and publication

When every SCC succeeds, the stage rechecks retained digests, asks Crat to
assemble exactly one accepted implementation per original function, and
restores `main` through finalization. Assembly uses original source order, not
thread completion order. No observation helper or placeholder function enters
the assembled final project.

Finalization validates manifest API names, preserves the manifest, and installs
source plus `proctor.toml` as one rollback transaction for a normal full
`cargo build`. Accepted functions remain at their original paths with transformed
signatures and original export attributes. No compatibility wrappers are
generated. Scratch `assembled.rs` and `final.rs` are consumed by rename;
inspect `work/current` and logs for late-failure evidence. The special
executable boundary is described in
[supportedness-and-intent.md](supportedness-and-intent.md).

Accepted `transform` regions alone contribute observations. Failed,
superseded, preserved, `mechanical`, and rule-complete regions do not become
new learning examples. Crat merges accepted documents in deterministic job
order, preserving member order and duplicates; Python neither parses nor
reserializes their contents. The merged document is emitted even when empty.
Rule synthesis and pretty-printing are separate Crat operations rather than
stage outputs.

Successful publication copies the complete final project except its root
`target/` and writes `statement-pairs.md`, `observations.json`, and `statistics.json`
under `outputs.artifacts_dir`, or the work directory when absent. Project and
artifacts form one cleanup transaction; a merge/copy/publication failure
reports no usable output. Scratch observation helpers and per-SCC documents
remain outside the published project. The statement-pair report includes
accepted `transform` and `mechanical` regions with relevant pointer-type metadata;
it is diagnostic, not a semantic proof or a rule-set artifact.

Stage output includes effective configuration, models, prompt provenance,
aggregate usage/cost, function/SCC/generation/repair/failure/build counts, and
tool/worker logs. Provider retries are separate usage attempts within one
logical generation. Worker clients share one rate limiter. Failed calls and
in-flight usage still count; optional exchange dumping retains every rendered
prompt and response under SCC/generation directories.

## Focused verification

In `tests/test_local_transformation.py`, select tests by observable concern:

| Change | Useful coverage/search terms |
| --- | --- |
| Records, graph, context | `dual_view_loader`, `dependency`, `independent_scc_order`, `mandatory_overflow` |
| Prompts and guidance | `prompt`, `libc_guidance`, `printf_guidance`, `foreign_static_guidance`, `repair_prompt` |
| Boundaries and preparation | `invalid_project_manifest`, `required_dependency`, `preparation_and_initialization`, `overlaps`, `symlink` |
| Acceptance and repair | `metadata_digests`, `unfilled_whole_function_stub`, `fallback`, `rollback`, `extraction_runs_only` |
| Workers and usage | `concurrent_workers`, `single_worker`, `failure_stops_dispatch`, `rate_limiter`, `private_cargo` |
| Assembly and publication | `candidate_source`, `final_build_failure`, `merged_observation`, `publish`, `statistics` |

Keep Python tests independent of live LLM providers and the real Crat toolchain.
Compiler behaviors belong in the relevant in-memory Crat module tests, not a
test that shells out to `crat-tool` or changes filesystem state.
