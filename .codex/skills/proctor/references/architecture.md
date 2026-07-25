# Architecture and ownership

## Contents

- Purpose and current pipeline
- Artifact flow
- Package ownership
- Run lifecycle
- Source-of-truth hierarchy
- Architectural invariants

## Purpose and current pipeline

Treat Proctor as shared infrastructure for experiments that turn a TRACTOR-format C project into progressively safer Rust. The framework supplies orchestration, reproducibility, a vendor-neutral LLM client, usage accounting, prompt management, Rust context retrieval, and test-package execution. Each transformation stage remains an independently executable program and may use none of the shared libraries.

The checked-in full pipeline is:

```text
C project + test package
        |
        v
c2rust-adapter -> unsafe Rust + config.toml
        |
        v
crat-adapter   -> symbolic pass chain + proctor.toml
        |
        v
abstraction-recovery -> currently skips because candidate discovery is a scaffold
```

The design documents describe later discipline-repair and local-transformation stages. They are not present in the current configured pipeline.

Although early plans describe one Translation component, the implementation exposes C2Rust and CRAT as two stages. CRAT completes the component by creating `proctor.toml`.

## Artifact flow

Use these four framework artifact kinds:

| Kind | Meaning | Runner/stage behavior |
|---|---|---|
| `c_project` | TRACTOR-format C project | Normally supplied by the runner; never produced |
| `rust_project` | Cargo project moving through transformations | May be supplied or produced |
| `test_package` | Executable `run_test.sh` plus `test_data/` | Normally supplied; not currently producible |
| `rule_set` | Opaque local-transformation rules file | May be supplied or produced |

`stage.toml` defines whether each kind is `required`, `optional`, or `unused`, and whether a stage produces a Rust project or rule set. `proctor validate` walks enabled stages in order and tracks availability.

Keep `proctor.toml` separate from the JSON stage envelope:

- The envelope connects the orchestrator to one stage invocation.
- `proctor.toml` travels inside a Rust project and records target identity, public API functions, and wrapper relationships for downstream transformations.

## Package ownership

Use this map to find the smallest owning surface:

| Path | Ownership |
|---|---|
| `proctor/contracts/` | Stage envelopes, stage manifests, project manifest, artifact wrappers, JSON Schemas |
| `proctor/config/` | TOML overlay loading, `--set`, typed config, stage resolution |
| `proctor/orchestrator/run.py` | Run creation, input recording, artifact state, sequencing, gating, resume |
| `proctor/orchestrator/invoke.py` | Stage command construction and process-group timeout |
| `proctor/orchestrator/checkpoint.py` | Stage/config fingerprints and hash-chain checkpoints |
| `proctor/orchestrator/validate.py` | Preflight artifact-wiring validation |
| `proctor/orchestrator/bench.py` | Corpus discovery and parallel independent runs |
| `proctor/orchestrator/events.py` | Append-only `events.jsonl` |
| `proctor/orchestrator/record.py` | `run.json` provenance |
| `proctor/cli.py` | CLI parsing and verb dispatch |
| `proctor/llm/` | Request/response types, retry, rate limiting, truncation, providers |
| `proctor/usage/` | Per-attempt JSONL, pricing, aggregation/rendering |
| `proctor/prompts/` | Strict versioned Jinja templates and content hashes |
| `proctor/context/` | Rust index cache, target resolution, retrieval strategies |
| `crates/proctor-rust-index/` | `syn`-based Rust item/edge indexer |
| `proctor/testing/runner.py` | Cargo build, artifact inference, test-package invocation |
| `stages/*-adapter/` | Envelope shims around pinned upstream tools |
| `stages/example-stage/` | Minimal framework-independent stage template |
| `stages/example-llm-stage/` | Shared LLM/tracker/prompt integration example |
| `tests/fake_stages/fake/` | Deterministic orchestration test double |

Keep `proctor/contracts/` dependency-light. It is the load-bearing public model used by other framework packages and optionally by stages.

## Run lifecycle

Follow this sequence in `start_run`/`execute_run`:

1. Load and type-check merged TOML.
2. Validate enabled stages and their artifact chain.
3. Create `runs/<run_id>/`, save fully resolved `run.toml`, and record provenance in `run.json`.
4. Copy supplied inputs into `inputs/{c,rust,tests,rule_set}`.
5. For each enabled stage:
   - derive a stage fingerprint and chained checkpoint key;
   - create `stages/<NN>-<id>/work` and `out/artifacts`;
   - write an absolute-path `stage_input.json`;
   - run the stage, through its own `uv` project when it has `pyproject.toml`;
   - capture process output in stage-local logs;
   - validate `stage_output.json`;
   - optionally gate a produced Rust project with the test package;
   - update artifact state only after success;
   - forward previous state on `skipped`;
   - write `.checkpoint` and events.
6. Return the current artifact state as the run result.

A run directory is intended to be self-contained and auditable:

```text
run.toml
run.json
events.jsonl
inputs/
stages/00-.../
  stage_input.json
  stage_output.json
  stdout.log
  stderr.log
  usage.jsonl
  .checkpoint
  work/
  out/{rust,rule_set,artifacts}/
```

## Source-of-truth hierarchy

Resolve discrepancies in this order:

1. Tests and versioned JSON Schemas
2. Current implementation and manifests/configs
3. `docs/stage-contract.md` and `docs/writing-a-stage.md`
4. READMEs
5. Plans

Plans explain why the code has its shape but contain milestones and examples that are not all implemented. Update current-facing docs when implementation changes; retain plans as historical/design material unless the task explicitly revises them.

## Architectural invariants

- Keep orchestration language-agnostic and file-based.
- Keep batch parallelism outside stages: one stage invocation handles one case.
- Pass stage-specific config opaquely; make the stage validate it.
- Use environment variables for credentials and host/toolchain integration.
- Make failures reproducible from saved envelopes and logs.
- Isolate stage Python dependencies with `uv run --project <stage-dir>`.
- Kill the entire spawned process group on timeout because stages launch Cargo/rustc children.
- Record upstream tool pins through git submodules; keep upstream repositories separate from adapter code.
- Keep normal unit tests independent from external APIs, real CRAT/C2Rust builds, and network access.
