---
name: proctor
description: Repository onboarding and implementation guidance for the PROCTOR C-to-Rust orchestration framework. Use when working in this repository on pipeline configuration, stage contracts or adapters, orchestration and resume behavior, LLM providers and usage tracking, prompt/context infrastructure, test-package or TRACTOR-vector execution, abstraction recovery, local transformation and rule integration, benchmarks, end-to-end translation, documentation, debugging, reviews, or planning changes against the current implementation.
---

# Proctor

Use this skill as the repository map for the PROCTOR orchestration framework. Start from the current code and tests; treat `plan_docs/` as design intent and deferred work unless the implementation confirms otherwise.

## Work at the PROCTOR root

- Locate the standalone PROCTOR root by `pyproject.toml`, the `proctor/` package, and `stages/`; in the enclosing research workspace it is `proctor/` rather than the workspace root.
- Resolve all paths in this skill relative to that root and inspect its own git/submodule status before editing.
- Keep the repository self-contained. Do not make its code, tests, configs, fixtures, or tooling depend on the enclosing workspace's plans or helper files.

## Route the task

Read only the references needed for the task:

- Read [architecture.md](references/architecture.md) first for unfamiliar or cross-cutting work, data flow, ownership, and source-of-truth rules.
- Read [contracts-and-stages.md](references/contracts-and-stages.md) before changing envelopes, artifact kinds, `proctor.toml`, `stage.toml`, stage adapters, native stages, or the local-transformation protocol.
- Read [configuration-and-runs.md](references/configuration-and-runs.md) before changing config, CLI behavior, run directories, checkpoint/resume, bench, LLM settings, usage, prompts, or context retrieval.
- Read [testing-and-status.md](references/testing-and-status.md) before implementation or review to select tests and distinguish implemented behavior from planned features.

For a narrow task, inspect the named implementation and its matching test after reading the relevant reference. Do not re-read the whole repository.

## Establish current truth

Use this precedence when sources disagree:

1. Executable tests and JSON Schemas
2. Current Python/Rust implementation and checked-in stage manifests
3. `docs/stage-contract.md` and `docs/writing-a-stage.md`
4. Root and component READMEs
5. `plan_docs/`

Call out a discrepancy instead of silently implementing an older plan. Preserve unrelated worktree and submodule changes.

## Work by change type

### Change the framework

1. Locate the owning package from [architecture.md](references/architecture.md).
2. Read its focused unit test before editing.
3. Preserve the contract boundary: the orchestrator invokes standalone stages; it does not import their implementation.
4. Update typed models, serializers, schemas, docs, examples, and tests together when a public shape changes.
5. Run the narrow test, then the default unit suite and static checks appropriate to the change.

### Change or add a stage

1. Read [contracts-and-stages.md](references/contracts-and-stages.md) and the authoritative contract docs.
2. Start from `stages/example-stage/` for a framework-free stage or `stages/example-llm-stage/` for shared LLM infrastructure.
3. Declare exact requirements and products in `stage.toml`.
4. Read inputs only, use `framework.workdir` for scratch data, create outputs only at envelope destinations, and always emit a valid failure envelope on errors.
5. Build and test transformation output inside the stage. Update `proctor.toml` when a non-local transformation changes persisted wrapper relationships.
6. Add manifest/contract tests and at least one direct stage or orchestrated test.

### Change an adapter

Keep the upstream `stages/c2rust/` and `stages/crat/` submodules unmodified unless the task explicitly targets those repositories. Put envelope translation, environment setup, caching, tool invocation, and output normalization in the corresponding `*-adapter/`.

### Change local transformation

1. Read [contracts-and-stages.md](references/contracts-and-stages.md), then use the `$crat` skill for changes that cross into `crat-tool` or shared pointer analysis.
2. Keep SCC scheduling, prompts, LLM repair, transactional Cargo builds, dependency preparation, artifacts, and usage accounting in `stages/local-transformation/`; keep compiler-resolved structure, replacement, trusted `printf` templates, observations, and rules in Crat.
3. Preserve the dual skeleton views: try optional rule applications through the applied view, but fall back to the baseline view for the whole SCC after a rule-involved candidate fails to compile.
4. Treat the input rule set as read-only. Keep replacement correspondence in stage state rather than `proctor.toml`; the stage produces a Rust project and diagnostic/learning artifacts, not a new rule set.
5. Accept a replacement only after structural validation when LLM output is used and a transactional `cargo build` succeeds. Let SCCs whose selected views need no LLM work—because rules and/or mechanical conversions completed them—bypass the LLM and validator, but not replacement or build acceptance. Extract observations only from accepted transform regions.
6. Run `tests/test_local_transformation.py` and the focused Crat `tools` tests for the Rust surface you changed.

### Change the stage contract

Treat this as high-risk, cross-repository work:

1. Decide whether the change is additive or breaking.
2. Update dataclasses, JSON Schemas, validation, artifact wiring, docs, templates/adapters, and conformance tests as one change.
3. Bump `schema_version` only for a breaking change.
4. Preserve the rule that readers ignore unknown additive fields and reject newer schema versions.

### Diagnose a run

Inspect in this order:

1. `run.json` for provenance and recorded inputs
2. `run.toml` for the resolved configuration
3. `events.jsonl` for orchestration order and gating
4. `stages/<NN>-<id>/stage_input.json`
5. `stage_output.json`, `.checkpoint`, `stdout.log`, `stderr.log`
6. `out/artifacts/` and per-stage `usage.jsonl`

Replay the stage from its saved envelope only after redirecting or clearing output destinations safely. Use `proctor resume <run>` for normal recovery and `--from <stage>` after intentional intermediate edits.

## Preserve core invariants

- Keep every stage a single-case standalone program invoked with `--input` and `--output`.
- Never mutate an input artifact; copy then modify.
- Keep paths in envelopes absolute and paths recorded inside run metadata relocatable where intended.
- Let `skipped` forward existing state without copying an output.
- Keep provider credentials in environment variables, never config or run records.
- Report every LLM attempt, including errors, to the stage usage log when using the shared client.
- Treat `proctor.toml` as the Rust-project compatibility manifest after CRAT.
- Keep default tests free of real toolchain, network, and API-key requirements; mark real-toolchain coverage `e2e`.

## Verify proportionally

Use [testing-and-status.md](references/testing-and-status.md) to choose focused tests. The normal ladder is:

```bash
uv run pytest <focused-test>
uv run pytest
uv run ruff check .
uv run ruff format --check .
uv run mypy proctor
```

Run `uv run pytest -m e2e` only when the affected adapter, real CRAT/C2Rust flow, Rust index, or TRACTOR vector harness requires it and prerequisites are available. Validate configs with `uv run proctor validate -c <config>` before expensive runs.

## Avoid stale assumptions

Do not present these planned items as implemented: discipline repair, test-vector-to-test-package conversion or `make-tests`, per-stage `gate_tests`, test packages as stage-produced artifacts, chained/merge bench rule-set policies, automatic intermediate pruning, token-rate or budget enforcement, `prompts.lock`, or `check-stage`.

Do not describe abstraction recovery as an unimplemented scaffold. A narrow direct-Claude implementation exists for function-local manual dynamic arrays, although its older default config comments and full-pipeline E2E still assume scaffold-style skipping.

Do not assume the global post-stage test gate is safe after C2Rust: executable cases are still library-shaped until CRAT's `bin` pass.
