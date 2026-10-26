# Configuration and runtime behavior

All paths and commands here are relative to the PROCTOR root (`./proctor/`).

## Configuration

Repeated `-c` files are deep-merged in order: tables merge, arrays/scalars
replace. `--set dotted.path=value` applies last, parsing TOML values where
possible and otherwise accepting a plain string. A dotted override cannot
traverse a scalar.

`PipelineConfig` types framework-owned fields:

- `[run]`: output directory, failure policy, keep-intermediates flag, provided
  artifact kinds;
- `[pipeline].order`: unique ordered configured stage ids;
- `[testing]`: global post-stage test-package gate and debug/release profile;
- `[llm]`: shared defaults;
- `[stages.<id>]`: `uses`, enabled flag, timeout, opaque config, LLM override.

Stage paths resolve relative to CLI `--root` unless absolute. Disabled stages
are removed and enabled indexes renumbered. Global LLM defaults merge with
stage LLM overrides; stage-specific `config` is passed to its validator without
framework application of `stage.toml` documented defaults.

### Checked-in compositions

| Config | Composition/purpose |
|---|---|
| `configs/example.toml`, `configs/llm_example.toml` | Copy-through or shared-LLM examples |
| `configs/c2rust.toml`, `configs/c2rust_crat.toml` | Translation core |
| `configs/c2rust_crat_absrec.toml`, `configs/c2rust_crat_absrec_llm.toml` | Translation plus agentic or direct-HTTP abstraction recovery |
| `configs/c2rust_crat_local.toml` | Prepared translation plus local transformation |
| `configs/c2rust_crat_testgen_disrep.toml` | Translation, generated oracle, discipline repair |
| `configs/c2rust_crat_testgen_absrec_disrep.toml` | Adds abstraction recovery before discipline repair |
| `configs/c2rust_crat_testgen_absrec_disrep_local.toml` | Adds local transformation before final oracle run |
| `configs/bench.toml`, `configs/bench_vectors.toml` | Independent corpus runs, optionally with vector comparison |
| `tests/e2e/crat_smoke.toml`, `tests/e2e/translation_smoke.toml` | Real-toolchain smoke fixtures |
| `configs/full_pipeline.toml` | Older core/recovery wiring; its scaffold comments are stale |

Use small overlays for experiments. Replacing `pipeline.order` requires the
complete desired array. Model/provider choices in examples are configuration,
not capability guarantees; independent agent stages may read `config.model`
and their own credentials instead of shared `[llm]` settings.

## CLI and preflight

```text
proctor validate -c CFG [--set ...] [--root ...]
proctor stages   -c CFG
proctor run      -c CFG --input-c/--input-rust/--tests/--rule-set
proctor resume   RUN_DIR [--from STAGE]
proctor bench    -c CFG --corpus DIR [--jobs N] [--match REGEX] [--name NAME]
proctor report   PATH... [--group-by stage,model] [--format table|csv|json]
proctor warmup   -c CFG
```

Each kind in `[run].provides` requires a corresponding input flag. Supplying a
kind outside `provides` warns and does not make it available to preflight.
`validate` checks artifact wiring, not tool availability or stage config
semantics. `warmup` syncs configured stage environments, runs their declared
warmup commands, and builds the Rust indexer. It may install/build prerequisites.
The CLI's `make-tests` and `check-stage` entries remain stubs despite the
implemented test-infrastructure stage.

## Runs, diagnosis, and resume

A case has its own run directory containing fully resolved `run.toml`,
provenance in `run.json`, copied `inputs/`, chronological `events.jsonl`, and
numbered stage directories. Stage directories retain envelopes, process logs,
`.checkpoint`, work files, output artifacts, and shared-client usage when used.

For diagnosis, read provenance/config, events, the failing input/output
envelopes, process logs, then artifacts and component telemetry. Generated run
content is evidence for an explicitly requested run investigation; do not use
ignored runs as source material for ordinary repository study. Replaying a
saved stage requires safe fresh output destinations. `resume` is normal
recovery; `--from <stage>` forces rerunning a known boundary.

`on_stage_failure = "stop"` is default. `continue` attempts later stages with
the last valid artifact state. Skipped stages also preserve prior state.
`keep_intermediates` is parsed but currently does not prune.

A checkpoint key hashes stage id, stage code fingerprint, stage config/LLM/
timeout, envelope schema version, and the preceding stage key. Git-backed stage
fingerprints include the commit and dirty-file content; fallback tree hashing
excludes common build/cache directories. Resume reuses consecutive successful
or skipped checkpoints until a mismatch, then reruns that stage and downstream.
It does not hash supplied artifact contents or all global testing settings.
Force reruns when changing those or manually editing intermediates. Recorded
input destinations are reused on resume, not overwritten from arbitrary input
paths.

## Bench and evaluation

Bench discovers one case per directory with all required provided layouts:
`c_project -> c`, `rust_project -> c2rust`, `test_package -> tests`,
`rule_set -> rules`. Override under `[bench.layout]`; `[bench].jobs` sets
parallel cases. Each is an ordinary independent run; `bench.json` aggregates
outcomes and isolates failures. Only `rule_set_policy = "independent"` is
implemented; chained/merge-per-round policies are rejected.

`verify_vectors = true` verifies final Rust outputs using the vendored TRACTOR
harness; `verify_all_stages = true` compares intermediate outputs. Library cases
need a writable copy of the containing corpus Cargo workspace. Vector failures
are recorded separately: `BenchResult.ok` and the bench CLI status reflect
pipeline success, so inspect `vectors_ok` and reports.

`proctor.testing.vector_harness` also supports the
`tools/test_runner --no-falco` corpus harness through Nix. It skips file-change
vectors and folds config/build pseudo-results into build status. This is
separate from the vendored `runtests.rust` path. The
`proctor.testing.no_falco_bench` module stages cases, verifies outputs, and can
retain a baseline when recovery regresses build/vector results.

Measurement utilities are separate module entrypoints:

```bash
uv run python -m proctor.testing.metrics --help
uv run python -m proctor.testing.no_falco_bench --help
uv run python -m proctor.testing.suite_report --help
```

`unsafe_eval.py` drives the vendored syntactic unsafety scorer;
`idiomaticity_eval.py` drives the vendored Clippy-based metric. Compilation,
behavioral vector results, unsafety counts, and lint measurements are distinct
evidence; report them separately.

## Shared LLM client and usage

`proctor.llm.client.LlmClient` accepts settings from `framework.llm`. Providers are
Anthropic Messages, OpenAI Chat Completions/compatible servers (`base_url`), and
request-hashed offline replay cassettes. The API is text-only. Input usage
includes cached input, with cached tokens also recorded as a subset; reasoning
tokens are preserved when supplied by the provider.

Shared `[llm]` defaults do not configure every stage:

| Stage | LLM execution and model selection |
|---|---|
| C2Rust | No LLM |
| CRAT | No LLM |
| Test infrastructure | Independent coding-agent drivers; `config.model` selects the model |
| Abstraction recovery | Independent backends: Claude Code uses its own default; OpenCode uses `config.model` or `config.llm_model`; direct HTTP uses `config.llm_model` or `OPENAI_MODEL` and `OPENAI_API_KEY` |
| Discipline repair | Independent coding-agent drivers; uses `config.model`, falling back to `framework.llm.model` when the key is absent |
| Local transformation | Shared `LlmClient` with resolved `framework.llm` settings |

`stages/example-llm-stage/` also demonstrates the shared client. Independent
agent stages own their provider setup and telemetry.

The client retries rate limits, server/transport failures with backoff and
records every attempt through a supplied `UsageTracker`. Authentication and
deterministic non-429 client errors do not retry. Context overflow defaults to
an error, with bounded head/middle truncation available. Request-rate limiting
exists; token-rate limiting and budget enforcement do not.

Use environment credentials, including configured `api_key_env`; never write
secrets to TOML or run records. Compatible local servers may use no key.

The tracker writes JSONL with run/stage/item, provider/model, prompt metadata,
tokens, latency, finish reason, error, and cost. Prices under
`[llm.pricing."provider/model"]` are dollars per million input/cached/output
tokens. Unknown pricing yields null cost and a warning, not zero.
`proctor report` collects `usage.jsonl` recursively. Abstraction recovery,
discipline repair, and test infrastructure have independent backend/agent
telemetry; their envelope summaries are not automatically shared-client logs.

## Prompts and context

Prompt templates are Markdown with `+++`-delimited TOML frontmatter: stable id,
positive integer version, declared variables, and Jinja body. Strict rendering
rejects missing/unknown variables. Record rendered id, version, SHA-256;
`PromptLibrary.get(id)` chooses the latest version unless pinned.
`prompts.lock` enforcement remains absent.

`proctor.context.api.retrieve_context` uses a best-effort `syn` Rust index from
`crates/proctor-rust-index/`. Cache by Rust-source hash under `PROCTOR_CACHE_DIR`
(default `~/.cache/proctor`). Resolve exact paths before unique suffixes;
ambiguous suffixes fail and unresolved edges are omitted. Strategies are
`target_only` and `target_plus_types`; the latter follows type references, not
callees. A token budget retains at least the target even if its estimate is
larger than the budget. This syntactic index is distinct from Crat's
compiler-resolved local-transformation analysis.
