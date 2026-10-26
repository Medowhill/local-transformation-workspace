# Testing and implementation status

All paths/commands here are relative to the PROCTOR root (`./proctor/`) unless
a stage working directory is explicitly named.

## Framework verification

```bash
uv run pytest tests/test_<area>.py
uv run pytest
uv run ruff check .
uv run ruff format --check .
uv run mypy proctor
```

The root pytest configuration collects `tests/` and excludes `e2e` by default.
Framework tests should be offline and use fake stages/providers/subprocesses.
Run focused coverage for the change; broaden when behavior crosses owners.
`uv run proctor validate -c CFG` checks config/stage artifact wiring.
Documentation/skill-only edits need content/link validation, not toolchain runs.

| Area | Framework tests under `tests/` |
|---|---|
| Envelopes, project metadata, artifact wiring | `test_stage_io.py`, `test_schemas.py`, `test_manifest.py`, `test_validate.py` |
| Config and CLI | `test_config.py`, `test_cli.py` |
| Sequencing, checkpoint/resume, failure, timeout, gates | `test_orchestrator.py`, `test_warmup.py` |
| Corpus discovery and independent bench | `test_bench.py` |
| C2Rust and CRAT adapters | `test_c2rust_adapter.py`, `test_crat_adapter.py` |
| Shared LLM, usage, prompts | `test_llm.py`, `test_usage.py`, `test_prompts.py` |
| Rust context/index | `test_context.py` |
| Framework test-package build/run | `test_testing.py` |
| Local transformation | `test_local_transformation.py` |
| Vector harnesses and comparison | `test_vector_harness.py`, `test_vector_compare.py`, `test_vector_inplace.py`, `test_vector_no_falco.py`, `test_no_falco_bench.py` |
| Scoring and reporting | `test_unsafe_eval.py`, `test_idiomaticity_eval.py`, `test_metrics.py`, `test_suite_report.py` |
| Vendored harness provenance and examples | `test_tractor_runtests_provenance.py`, `test_example_stage.py` |

Contract changes normally exercise envelope/schema/manifest/wiring and
orchestrator tests together. Config/CLI changes include preflight and relevant
compositions. Adapter handoff changes include target discovery, `config.toml`,
`proctor.toml`, environment, and pass planning. Shared provider tests should
exercise pure payload/response mapping and replay rather than live APIs.

## Stage-local verification

The framework's default pytest does not collect all component-submodule tests.
Use the owning `pyproject.toml` and local test environment; discipline repair and
test infrastructure currently require Python 3.13+, while the framework requires
3.12+. Do not assume their dev tools or dependencies match the root project.

| Stage | Focused coverage and working directory |
|---|---|
| C2Rust | Root `tests/test_c2rust_adapter.py`; translation E2E for real transpiler behavior |
| CRAT | Root `tests/test_crat_adapter.py`; Crat package/module tests from `stages/crat/`, guided by `$crat` |
| Test infrastructure | From `stages/test-infrastructure/`, `uv run pytest tests/test_infrastructure`; cover stage dispatch, strategies, convergence, vectors, and relocation |
| Abstraction recovery | From `stages/abstraction-recovery/`, run `test_main.py` and `test_llm_abstraction_recovery.py` in an environment with pytest; these mock backends/builds |
| Discipline repair | From `stages/discipline-repair/`, `uv run pytest tests/dr_agent`; cover stage/config, repair loop, gate, oracle, and analyzer tools |
| Local transformation | Root `uv run pytest tests/test_local_transformation.py`; matching Crat `tools` tests for protocol/compiler semantics, guided by both specialized skills |

`tests/test_scaffold.py` only checks package sanity. The full-pipeline E2E still
asserts an older abstraction-recovery skip path; it does not prove current live
backend behavior. There is no checked-in live-model local-transformation E2E.
Do not infer missing implementations from older tests/comments when focused
component suites and current entrypoints exist.

Discipline repair's default oracle checkout is its
`src/resources/test_infrastructure`, which is not a checked-in submodule.
Select the sibling test-infrastructure checkout with `config.testinfra_root`
or `PROCTOR_TESTINFRA_ROOT`, or provision it with `setup.sh --testinfra <checkout>` from
the discipline-repair root. Bare warmup skips that setup; a missing oracle
checkout yields `UNVERIFIED` test evidence.

## Real-toolchain coverage

Use `uv run pytest -m e2e` or a specific marked file when a change affects real
toolchain integration. Existing flows cover C2Rust/CRAT translation/build/resume,
Rust indexing/cache, and executable/library TRACTOR vectors. Read
`tests/e2e/README.md` for current setup.

Prerequisites depend on the flow: initialized submodules, pinned rustup
components, libclang/Z3 for Crat, CMake and a generator for C2Rust, available
transpiler, and the appropriate corpus workspace/harness. No-Falco
verification needs its Nix/Docker setup. Live agent stages require their own
installed agents and environment credentials; they are not default unit tests.
`proctor warmup` performs configured preparation and may build/download tools.

Keep the global test-package gate off for a C2Rust intermediate that is still
library-shaped despite an eventual executable target. CRAT's `bin` pass creates
the later executable boundary. An explicit final oracle stage is distinct from
the framework-wide after-each-stage gate.

## Current limits and interpretation

- All six production stage families have implementations: translation through
  C2Rust/CRAT, test infrastructure, abstraction recovery, discipline repair, and
  local transformation. Stage-specific validation differs; see the stage guide.
- Rule synthesis exists in Crat, outside the local stage. Stage-produced
  `test_package`, framework `make-tests`/`check-stage`, per-stage gate controls,
  chained/merged bench rules, intermediate pruning, token-rate/budget
  enforcement, prompt locking, and semantic context resolution remain absent.
- Checkpoints do not hash supplied artifact contents or global testing settings;
  resumed recorded inputs are reused. Force the appropriate rerun boundary.
- Bench pipeline success does not incorporate separately recorded vector
  failures. No-Falco verification skips file-change vectors.
- Abstraction recovery's built-in gate is compilation, discipline repair may
  report partial success, and local transformation validates/builds without
  running its oracle. Do not report those statuses as equivalent evidence of
  semantic correctness or full problem elimination.
- Local transformation has narrower project/wrapper/API support than the
  research intent; its detailed constraints belong to the specialized skill.
- Stage manifests and older configs/READMEs contain stale comments/defaults;
  compare entrypoints, config validators, and focused tests before relying on
  them. Keep models/pricing in configuration rather than implementation rules.
