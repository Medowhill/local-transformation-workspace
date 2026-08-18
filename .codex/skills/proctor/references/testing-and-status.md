# Testing, change coverage, and current status

## Contents

- Test layers
- Test ownership map
- Verification by change
- End-to-end prerequisites
- Implemented versus planned
- Known sharp edges

## Test layers

Use this verification ladder:

```bash
# focused behavior
uv run pytest tests/test_<area>.py

# default suite; pyproject excludes tests marked e2e
uv run pytest

# static quality
uv run ruff check .
uv run ruff format --check .
uv run mypy proctor

# opt-in real toolchains
uv run pytest -m e2e
```

The default suite should remain deterministic, offline, API-key-free, and independent from real CRAT/C2Rust builds. Use fakes, pure provider parsing, monkeypatched subprocesses, and cassette replay.

Run `uv run proctor validate -c <config>` for config/stage wiring and a focused CLI command when changing user-facing behavior.

## Test ownership map

| Test | Primary coverage |
|---|---|
| `test_config.py` | Overlay rules, `--set`, typed model, LLM resolution |
| `test_manifest.py` | `proctor.toml` round trip and invariants |
| `test_stage_io.py` | Envelope model, compatibility, status and usage checks |
| `test_schemas.py` | Dataclass serialization against JSON Schemas |
| `test_validate.py` | Artifact availability through stage chains |
| `test_orchestrator.py` | Sequencing, state forwarding, failures, timeouts, gating, resume |
| `test_cli.py` | Config-driven CLI verbs |
| `test_bench.py` | Discovery, parallel cases, summaries, failure isolation |
| `test_c2rust_adapter.py`, `test_crat_adapter.py` | Adapter input preparation, config/pass planning, and manifest helpers |
| `test_warmup.py` | Stage warmup commands and error propagation |
| `test_llm.py` | Retry/error behavior, provider mapping, truncation, cassette replay |
| `test_usage.py` | Pricing, recursive collection, aggregation, CLI reporting |
| `test_prompts.py` | Frontmatter, strict variables, versions, hashes |
| `test_context.py` | Syntactic resolution, strategies, budgets |
| `test_testing.py` | Target inference and cargo/test-package flow |
| `test_local_transformation.py` | Skeleton protocol, SCC scheduling, LLM repair, rule fallback, replacement transactions, observations, statistics, and publication |
| `test_vector_harness.py`, `test_vector_compare.py`, `test_vector_inplace.py` | TRACTOR JUnit parsing, final/per-stage comparison, bench recording, and library workspace handling |
| `test_tractor_runtests_provenance.py` | Integrity of the vendored TRACTOR harness copy |
| `test_example_stage.py` | Direct real subprocess contract behavior |
| `test_scaffold.py` | Package import/version sanity only; it does not cover abstraction recovery |
| `tests/fake_stages/fake/` | Orchestration modes without toolchains |

E2E tests:

| Test | Flow |
|---|---|
| `test_crat_smoke.py` | Vendored C2Rust fixture -> real CRAT -> build/run -> resume |
| `test_translation_smoke.py` | C fixture -> real C2Rust -> real CRAT -> manifest/tests/resume |
| `test_full_pipeline.py` | Translation plus abstraction-recovery skip |
| `test_index_e2e.py` | Build real indexer, index fixture, verify cache |
| `test_vector_harness_e2e.py` | Real translated executable through TRACTOR vectors |
| `test_vector_lib_e2e.py` | Real library/cando vector verification in a corpus workspace |

## Verification by change

### Contract or artifact change

Run at least:

```bash
uv run pytest tests/test_stage_io.py tests/test_schemas.py \
  tests/test_validate.py tests/test_orchestrator.py
```

Also update both contract docs, examples, stage manifests/adapters, and any affected config tests.

### Config or CLI change

Run:

```bash
uv run pytest tests/test_config.py tests/test_cli.py \
  tests/test_validate.py tests/test_bench.py
```

Validate each checked-in relevant config.

### Orchestrator, checkpoint, invocation, or gating change

Run:

```bash
uv run pytest tests/test_orchestrator.py tests/test_warmup.py \
  tests/test_bench.py
```

Add a fake-stage behavior when the edge case needs a controlled subprocess.

### LLM/provider/usage change

Run:

```bash
uv run pytest tests/test_llm.py tests/test_usage.py tests/test_prompts.py
```

Test payload and response mapping through pure functions. Do not require live APIs.

### Context/index change

Run `tests/test_context.py`. Run `tests/e2e/test_index_e2e.py -m e2e` when changing the Rust indexer or on-disk integration.

### Test runner change

Run `tests/test_testing.py` and orchestrator gating tests. Exercise executable and cdylib inference; CRAT executable Cargo output contains both library and binary tables, and the binary must win.

### C2Rust or CRAT adapter change

Run focused unit tests for extracted pure logic when available, then the relevant e2e smoke test. Run full translation e2e for changes to adapter handoff, `config.toml`, target discovery, manifest emission, or tool environment.

### Local-transformation change

Run `uv run pytest tests/test_local_transformation.py`. For changes to the Rust protocol or semantics, also run the matching Crat `cargo test -p tools <module>::tests` filter and `cargo test -p tools` when the public tools surface crosses modules. The current suite uses fake tools and LLM clients; no checked-in e2e test runs the complete local-transformation stage with a live model.

### Abstraction-recovery change

Do not treat `tests/test_scaffold.py` as stage coverage. Add or run mocked direct-stage tests for Claude invocation, fail-open/fail behavior, and build repair. The existing `test_full_pipeline.py` still describes and asserts the old scaffold-style skip path even though the stage now invokes Claude.

### TRACTOR vector change

Run `tests/test_vector_harness.py`, `tests/test_vector_compare.py`, `tests/test_vector_inplace.py`, and the provenance test as applicable. Run the executable and library vector E2E tests only when their toolchain/corpus prerequisites are available.

## End-to-end prerequisites

Expect e2e tests to require:

- initialized `stages/crat` and/or `stages/c2rust` submodules;
- rustup and CRAT's pinned nightly/components;
- libclang and Z3 development dependencies for CRAT;
- CMake and Ninja/Make for C2Rust;
- a `c2rust-transpile` on PATH, in `PROCTOR_CACHE_DIR/c2rust`, or buildable from the submodule.
- the vendored `tools/tractor_runtests` harness for vector tests, plus the enclosing corpus Cargo workspace for library/cando cases;
- `claude` and `ANTHROPIC_API_KEY` only for live abstraction-recovery runs.

Read `tests/e2e/README.md` for the current no-sudo cache recipe. First builds can take minutes; use `proctor warmup` for configured stages.

Keep the global test gate off in the C2Rust+CRAT translation smoke config. C2Rust's intermediate artifact is not yet an executable even when the final target is.

## Implemented versus planned

Treat these as implemented and unit-tested unless noted:

- version-1 stage envelopes and JSON Schemas;
- stage/project manifest models;
- TOML overlays and `--set`;
- preflight artifact-chain validation;
- run directories, artifact threading, failure policy, timeout, checkpoint/resume;
- global post-stage test gating;
- bench discovery and parallel independent cases;
- Anthropic, OpenAI-compatible, and replay LLM providers;
- per-attempt usage, pricing, reports;
- versioned strict prompts;
- syntactic Rust index and two retrieval strategies;
- test-package build/run logic;
- TRACTOR vector verification for final or per-stage bench outputs, including library workspace copies;
- C2Rust and CRAT adapters;
- local transformation with optional rule application, SCC-scoped LLM repair, structural validation, transactional Cargo-build acceptance, observation extraction, and statistics;
- narrow direct-Claude dynamic-array-to-Vec abstraction recovery (implemented, but without focused unit coverage and with stale scaffold-oriented config/E2E assumptions).

Treat these as planned or partial:

- discipline repair;
- automatic rule-set synthesis or update inside the local-transformation stage;
- broader abstraction families, shared-client integration, and test-backed abstraction-recovery validation;
- test-vector-to-test-package conversion and `proctor make-tests`;
- bench test-package auto-synthesis;
- stage-produced `test_package`;
- per-stage `gate_tests`;
- chained or merge-per-round rule-set policies;
- automatic pruning for `keep_intermediates = false`;
- token-per-minute limiting and budget enforcement;
- prompt lock/version enforcement;
- `proctor check-stage`;
- richer context strategies and semantic name resolution.

When implementing a planned feature, read its focused plan document, but reconcile it with current models and tests before editing.

## Known sharp edges

- Checkpoint keys do not hash input artifact contents or global testing settings. Force reruns when those change.
- `record_inputs` reuses an existing recorded destination, which is correct for resume but means the run directory is not a generic overwrite target.
- `keep_intermediates` is parsed but does not prune.
- The framework-wide gate cannot distinguish unsuitable intermediate stages.
- Stage output schema requires only a small core; rely on code/tests and review to enforce honest reproducibility reporting.
- Context indexing is syntactic. Ambiguous suffixes fail; unresolved edges are omitted.
- `target_plus_types` does not include callees.
- OpenAI-compatible local servers may use no API key; Anthropic always requires its configured key env.
- Abstraction recovery's compile-only validation is its own minimal implementation and should migrate to `proctor.testing.runner` if the stage takes a framework dependency.
- Local transformation requires `[lib].path` to name one root-level source file after preparation, builds but does not run the test package, and leaves rule synthesis outside the stage.
- Local transformation adds or raises crates.io requirements for `bytemuck`, `xj_scanf`, and `proctor-libc` before preparation, but preserves existing git/path/workspace dependencies.
- Vector-enabled bench records vector failures but does not include them in `BenchResult.ok` or the CLI exit status; inspect `bench.json`/`vectors_ok` explicitly.
- The vector harness uses the vendored pre-Falco `runtests.rust` path. Newer Falco/file-change semantics remain outside this integration.
- Abstraction recovery calls Claude Code directly, validates with `cargo build --release` only, and does not use the shared per-attempt usage tracker. Its default full-pipeline comments and skip-only E2E are stale.
- Plans and README examples may name future models or milestones. Keep model names and prices in config, not code.
