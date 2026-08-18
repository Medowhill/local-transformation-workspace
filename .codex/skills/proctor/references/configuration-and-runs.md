# Configuration and runtime behavior

## Contents

- Config loading and typing
- Checked-in configs
- CLI
- Run directories and failure policy
- Checkpoint/resume
- Bench
- LLM, usage, prompts, and context

## Config loading and typing

Build the effective config in this order:

1. Load each repeated `-c` TOML file in command-line order.
2. Deep-merge tables.
3. Replace scalars and arrays; never append arrays.
4. Apply each `--set dotted.path=value` last.

Parse override values as TOML when possible (`8`, `true`, arrays, quoted strings); fall back to an unquoted plain string. Reject a dotted override that traverses a scalar.

Use `PipelineConfig` for known framework fields:

- `[run]`: output directory, failure policy, keep-intermediates flag, initially provided artifacts
- `[pipeline] order`: unique ordered configured stage ids
- `[testing]`: global post-stage gate and debug/release profile
- `[llm]`: opaque shared defaults
- `[stages.<id>]`: `uses`, enabled, timeout, opaque config, LLM override

Resolve stage paths relative to CLI `--root` unless absolute. Drop disabled stages and renumber enabled stage indexes from zero.

Merge global `[llm]` with `[stages.<id>.llm]`, with stage values winning. Do not merge `[stages.<id>.config]` with another default table in the framework; pass it through.

## Checked-in configs

- `configs/example.toml`: one copy-through native stage; starts from Rust + tests.
- `configs/llm_example.toml`: one example LLM stage; starts from Rust.
- `configs/c2rust.toml`: C2Rust only; accepts a C project directory or tar archive.
- `configs/c2rust_crat.toml`: C2Rust plus an explicit reduced CRAT pass list.
- `configs/c2rust_crat_absrec.toml`: C2Rust, CRAT, then the live Claude Code abstraction-recovery stage.
- `configs/full_pipeline.toml`: older C2Rust/CRAT/abstraction-recovery wiring whose scaffold comments are stale; reconcile it before relying on it.
- `configs/c2rust_crat_local.toml`: C2Rust, a reduced CRAT pass chain, then local transformation; starts from C + an input rule set and uses a configured LLM.
- `configs/bench.toml`: C2Rust/CRAT corpus runs without correctness verification.
- `configs/bench_vectors.toml`: the same core flow with TRACTOR vector verification and per-stage comparison.
- `tests/e2e/crat_smoke.toml`: CRAT only over the vendored C2Rust fixture, with test gating.
- `tests/e2e/translation_smoke.toml`: C2Rust then CRAT, global gate disabled and final tests run explicitly.

Use small overlay files for experiments. Remember that replacing `pipeline.order` requires restating the complete desired array.

## CLI

Use the current verbs:

```text
proctor validate -c CFG [--set ...] [--root ...]
proctor stages   -c CFG
proctor run      -c CFG --input-c/--input-rust/--tests/--rule-set
proctor resume   RUN_DIR [--from STAGE]
proctor bench    -c CFG --corpus DIR [--jobs N] [--match REGEX] [--name NAME]
proctor report   PATH... [--group-by stage,model] [--format table|csv|json]
proctor warmup   -c CFG
```

Require one CLI input flag for every kind in `[run] provides`. Supplying a kind that is absent from `provides` only warns; it does not make the artifact available to validation.

Run `validate` and `stages` for cheap preflight. Use `warmup` to sync stage `uv` environments, run manifest warmup commands, and build the Rust indexer.

## Run directories and failure policy

Start each case in its own run directory. Copy supplied input files/trees into standardized `inputs/` locations. Save:

- `run.toml`: fully resolved config used for resume
- `run.json`: framework/stage fingerprints, config hash, inputs, host tools, image id
- `events.jsonl`: chronological run/stage/test events
- one numbered directory per enabled stage

Use `on_stage_failure = "stop"` by default. With `continue`, keep the last valid artifact state and attempt downstream stages. A skipped stage also preserves the prior state.

The typed `keep_intermediates` option exists, but pruning is not currently implemented.

## Checkpoint/resume

Compute each stage key from:

```text
stage id
+ stage code fingerprint
+ stage config/LLM/timeout hash
+ schema version
+ upstream checkpoint key
```

For a git-backed stage, use the commit plus a hash of dirty files under the stage. Outside git, hash the content tree while excluding build/cache directories.

On resume, reuse consecutive successful or skipped checkpoints until the first mismatch. Re-execute that stage and everything downstream. Use `--from <stage>` to force a boundary.

Do not assume checkpoints hash artifact contents or all global runner settings. They do not currently include supplied input trees, post-stage testing settings, or unrelated global config. Force a rerun when those changes matter.

Do not manually edit intermediates and expect automatic invalidation; checkpoint design assumes they remain untouched.

## Bench

Treat bench as parallel ordinary runs, one per discovered case. A case directory must contain the configured layout subpath for every kind in `[run] provides`.

Defaults:

```text
c_project -> c
rust_project -> c2rust
test_package -> tests
rule_set -> rules
```

Override under `[bench.layout]`; select worker count under `[bench] jobs`. Bench writes a parent `bench.json` and keeps each case as a complete run directory. Exceptions and failures remain isolated per case.

Only `rule_set_policy = "independent"` is implemented. Reject `chained` and `merge-per-round`.

Set `[bench] verify_vectors = true` to run the vendored TRACTOR `runtests.rust` harness on the final Rust-producing stage after each case; set `verify_all_stages = true` to compare every Rust-producing stage. Library cases require a writable copy of the containing TRACTOR Cargo workspace, while binary cases can use an isolated case. Treat this as verification and reporting: `bench.json` records `vectors_ok`, but `BenchResult.ok` and the CLI exit code currently reflect pipeline-run success, not vector failures.

## LLM client

Use `proctor.llm.LlmClient` with settings from `stage_input.framework.llm`.

Supported providers:

- `anthropic`: Messages API
- `openai`: Chat Completions, including compatible servers through `base_url`
- `replay`: request-hashed offline cassettes

The v1 API is text-only. Normalize provider usage so input tokens include cached tokens and cached input is also reported as the subset. Track reasoning tokens when a provider reports them.

Retry 429, 5xx, and transport/provider failures with backoff. Do not retry auth failures or deterministic non-429 4xx responses. Track every attempt before returning or raising.

Context overflow behavior is configurable: structured error by default, or bounded head/middle truncation. Request-rate limiting is implemented; token-rate limiting and budget enforcement are not.

Keep API keys in `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or the configured environment-variable name. Never put secrets in TOML.

## Usage and pricing

Write one JSON object per attempt to the envelope's stage-local `usage.jsonl`. Include run/stage/item, provider/model, prompt metadata, token categories, latency, finish reason, cost, and error.

Configure prices under:

```toml
[llm.pricing."provider/model"]
input = 0.0
cached_input = 0.0
output = 0.0
```

Interpret prices as dollars per million tokens. Return unknown cost as null and warn once; never treat unknown pricing as free. `proctor report` recursively collects `usage.jsonl` files and groups by arbitrary record fields.

## Prompts

Use Markdown templates with TOML frontmatter delimited by `+++`. Require a stable id, positive integer version, declared variables, and a Jinja body. Render with `StrictUndefined`; reject missing and unknown variables.

Record rendered prompt id, version, and SHA-256 in usage metadata. `PromptLibrary.get(id)` selects the latest version; pass `version=` for reproducible pinning.

`prompts.lock` enforcement is planned but absent. Reviewers must ensure meaningful content edits get intentional version treatment.

## Context retrieval

Use:

```python
retrieve_context(
    project,
    strategy="target_plus_types",
    target="crate::path::item",
    budget_tokens=8000,
)
```

The Rust `syn` index is syntactic and best-effort. Cache indexes by the Rust-source tree hash under `PROCTOR_CACHE_DIR` (default `~/.cache/proctor`). Resolve exact paths first, then unique suffixes.

Built-in strategies:

- `target_only`
- `target_plus_types`

The latter includes resolved type references, not callees. Token budgets keep at least the target even when it exceeds the requested estimate.
