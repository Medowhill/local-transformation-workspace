# Architecture and ownership

All implementation paths in this reference are relative to the PROCTOR root
(`./proctor/` in the enclosing workspace).

## Pipeline and artifact flow

PROCTOR combines symbolic analysis and neural rewriting to turn TRACTOR-format
C projects into safer, more idiomatic Rust. It provides reproducible runs,
file-based stage composition, shared LLM utilities, and evaluation tooling.
The transformation and test stages remain independently executable.

The checked-in compositions support this sequence:

```text
C project -> c2rust-adapter -> crat-adapter
                                  |
                     test-infrastructure (generate)
                                  |
                       abstraction-recovery
                                  |
                         discipline-repair
                                  |
                       local-transformation
                                  |
                     test-infrastructure (run)
```

This is a composition, not a hard-coded runner sequence. Configs can use the
translation core, individual branches, or subsets. Test generation, abstraction
recovery, discipline repair, and local transformation all have implementations;
older scaffold comments do not describe their present status. See
[contracts-and-stages.md](contracts-and-stages.md) for comparable stage details.

The framework tracks four artifact kinds:

| Kind | Meaning | Current framework flow |
|---|---|---|
| `c_project` | TRACTOR C directory; C2Rust also accepts supported tar archives | Supplied input |
| `rust_project` | Cargo project carried between stages | Supplied or produced |
| `test_package` | `run_test.sh` with `test_data/` for the framework test runner | Supplied input |
| `rule_set` | Rules file opaque to the orchestrator | Supplied or producible by contract; current local stage consumes it only |

The test-infrastructure stage delivers its harness, vectors, and
`testinfra.toml` inside `rust_project`; it does not produce a framework
`test_package`. Raw corpus vectors are not a framework artifact kind.

## Package ownership

| Path | Responsibility |
|---|---|
| `proctor/contracts/` | Stage envelopes, schemas, stage manifests, project metadata |
| `proctor/config/` | TOML overlays, typed config, stage/LLM resolution |
| `proctor/cli.py` | CLI parsing and dispatch |
| `proctor/orchestrator/run.py` | Recorded inputs, artifact state, sequencing, gating, resume |
| `proctor/orchestrator/invoke.py` | Commands, per-stage environments, process-group timeouts |
| `proctor/orchestrator/checkpoint.py` | Stage fingerprints and chained checkpoint keys |
| `proctor/orchestrator/validate.py` | Preflight stage/artifact availability |
| `proctor/orchestrator/bench.py` | Corpus discovery and parallel independent cases |
| `proctor/orchestrator/events.py`, `record.py` | Events and provenance |
| `proctor/llm/`, `proctor/usage/` | Shared providers, retries, request limiting, usage and prices |
| `proctor/prompts/` | Strict Jinja templates, numeric versions, content hashes |
| `proctor/context/`, `crates/proctor-rust-index/` | Syntactic Rust indexing/cache and retrieval |
| `proctor/testing/runner.py` | Framework test-package builds and execution |
| `proctor/testing/vector_harness.py`, `vector_compare.py` | TRACTOR harness invocation and stage comparison |
| `proctor/testing/no_falco_bench.py`, `suite_report.py` | No-Falco harness verification, regression gating, scorecards |
| `proctor/testing/unsafe_eval.py`, `idiomaticity_eval.py`, `metrics.py` | Source-unsafety and Clippy-based measurements |
| `stages/c2rust-adapter/`, `stages/crat-adapter/` | Envelope shims around C2Rust and Crat |
| `stages/abstraction-recovery/` | Candidate abstraction identification and bounded rewrite/repair |
| `stages/discipline-repair/` | Instrumentation-guided repair and evidence-based acceptance |
| `stages/test-infrastructure/` | Harness/vector setup, generation, execution, relocation |
| `stages/local-transformation/` | Fixed-analysis SCC transformation and worker orchestration |
| `stages/example-stage/`, `stages/example-llm-stage/` | Minimal independent/shared-LLM examples |
| `tools/` | Vendored harnesses and measurement tools |

`.gitmodules` identifies C2Rust, Crat, abstraction recovery, discipline repair,
and test infrastructure as separate checkouts. Keep their internal behavior
in those repositories and their integration in PROCTOR. `proctor/contracts/`
should remain dependency-light.

## Run lifecycle

`start_run` and `execute_run` resolve configuration, validate enabled stages,
create a run directory, record provenance, and copy supplied inputs into it.
For each stage the runner computes a checkpoint key, writes an absolute-path
envelope, invokes the stage in its own directory/environment, captures logs,
validates its response, and optionally applies the framework test-package gate.
It updates artifact state only for successful outputs; `skipped` forwards the
previous state. Failures stop by default, or continue with the last valid state
when configured.

```text
run.toml / run.json / events.jsonl
inputs/{c,rust,tests,rule_set}
stages/<NN>-<configured-id>/
  stage_input.json / stage_output.json
  stdout.log / stderr.log / .checkpoint
  usage.jsonl                      # shared-client stages
  work/
  out/{rust,rule_set,artifacts}/     # destinations selected by manifest
```

A stage invocation handles one case; bench supplies parallelism across cases,
and individual stages may also parallelize internally. Stage config is opaque
to the framework and validated by the owning stage. Stages with `pyproject.toml`
are invoked through their own `uv` project. Timeouts kill the spawned process
group so Cargo/rustc/agent children do not outlive the stage.

## Source-of-truth boundary

Check code, focused tests, schemas, manifests, and stage config validators
before relying on contributor docs or examples. `docs/stage-contract.md` and
`docs/writing-a-stage.md` are documentation within the standalone PROCTOR
repository and remain useful contract guides. Research goals and intended
interfaces are retained in [research-intent.md](research-intent.md); they do not
replace implemented contracts.
