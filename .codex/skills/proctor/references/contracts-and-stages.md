# Contracts and stages

All implementation paths are relative to the PROCTOR root (`./proctor/`).
Read a stage's entrypoint, config validator, and focused tests before changing
its behavior; `stage.toml` config descriptions are not exhaustive validation.

## Shared contracts

The public framework boundary consists of:

- `proctor/contracts/stage_io.py` and its
  `proctor/contracts/schemas/stage_input.v1.json` /
  `proctor/contracts/schemas/stage_output.v1.json`;
- `proctor/contracts/stage_manifest.py` for `stage.toml`;
- `proctor/contracts/manifest.py` for Rust-project `proctor.toml`;
- `docs/stage-contract.md` and `docs/writing-a-stage.md` as contributor guides;
- `tests/test_stage_io.py`, `test_schemas.py`, `test_manifest.py`, and
  `test_validate.py` for serialization and compatibility behavior.

Envelopes retain schema version 1. Readers ignore unknown additive fields and
reject newer versions; update models, serializers, schemas, consumers, and
tests together when changing a public shape. Independently maintained stage
submodules may have their own envelope models; include those in integration
checks rather than assuming they import the framework model.

### Stage manifests

Each stage root declares `id`, `version`, nonempty `exec`, optional `warmup`,
`requires`, `produces`, and optional config documentation. Requirements are
`required`, `optional`, or `unused`; an omitted requirement is unused. The
framework tracks `c_project`, `rust_project`, `test_package`, and `rule_set`;
only `rust_project` and `rule_set` can currently be produced.

A config's stage id identifies that invocation. It may differ from the
manifest id with a preflight warning, permitting the test-infrastructure
checkout to serve both generation and later validation. Paths in `uses` are
resolved from the configured PROCTOR root. Stage-specific config is opaque to
the orchestrator, so its owning implementation determines accepted keys,
defaults, and failure behavior.

### Input and output envelopes

Invoke every stage as:

```text
<exec...> --input /abs/stage_input.json --output /abs/stage_output.json
```

`StageInput` carries identity (`run_id`, `stage_id`, index, optional item),
read-only absolute artifact paths, output destinations, resolved stage config,
and advisory framework settings (LLM defaults, usage log, prompts, workdir,
timeout, budget). Cwd is the stage root for executables/resources, not a base
for artifact paths.

`StageOutput` reports `success`, `failure`, or `skipped`, outputs, config used,
optional models/usage/prompts, a metrics object, artifact-relative logs, metadata,
and error. Failure requires a nonempty error; other statuses require null
error. Success must create declared artifacts and exit zero; omitted output
paths fall back to the framework-assigned destinations. Skipped exits zero
and forwards previous artifact state; it need not copy an output. Stages should
emit a failure envelope on exceptions instead of leaving only a stack trace.
The runner also treats timeouts, nonzero success exits, missing outputs, and
invalid envelopes as failures.

### Project and test metadata

CRAT creates `proctor.toml` containing `target_kind` (`executable` or `library`),
nonempty Cargo `target_name`, `api_functions`, and crate-relative wrapper
relationships (`wrapped`, `wrapper`). Executables require an empty API list.
The model can express wrappers even though the current local stage requires
none. This is project compatibility metadata, not transformation correspondence.

Test infrastructure stores strategy, `target_kind` (`bin` or `lib`), runner,
and strategy metadata in `testinfra.toml`. Preserve that alongside the project
when composing stages that reuse its oracle. The framework test-package
contract is separately `run_test.sh <test_data-dir> <compiled-artifact>`, from
any cwd, with exit zero for success. Do not substitute one contract for another.

## Stage implementations

### C2Rust translation

- **Purpose and artifacts:** `stages/c2rust-adapter/main.py` consumes a required
  C project and produces a copied Cargo Rust project. It accepts a supported C
  directory/tar layout, configures CMake, discovers target/API information, and
  invokes `stages/c2rust` through a resolved transpiler.
- **Controls and acceptance:** `preset` selects a CMake preset when present;
  `check_build` defaults true. Link/build fixups, `config.toml` (including
  exposed functions and optional binary identity), and `libs.json` accompany
  output. The intermediate remains library-shaped until later CRAT conversion.
- **Ownership and checks:** CMake/tool invocation and output normalization
  belong to the adapter; transpiler internals belong to the submodule. Use
  `tests/test_c2rust_adapter.py` and translation smoke E2E for the handoff.

### CRAT symbolic transformation

- **Purpose and artifacts:** `stages/crat-adapter/main.py` consumes Rust with
  C2Rust's `config.toml`, runs Crat passes in separate scratch outputs, and
  produces a Cargo project with newly emitted `proctor.toml` and empty wrappers.
- **Controls and acceptance:** choose `final_pass` for the canonical predecessor
  chain or an exact `passes` list; they are mutually exclusive. `pass_args`
  replaces each selected pass's default arguments. `prepare` is an explicit
  alternative used in local-stage configs, not part of every default chain.
  `check_build` defaults true.
- **Ownership and checks:** build/cache/environment/pass composition belong to
  the adapter; analyses and rewrites belong to Crat. Use
  `tests/test_crat_adapter.py`, CRAT/translation smoke E2E, and `$crat` for
  compiler changes.

### Test generation and validation

- **Purpose and artifacts:** `stages/test-infrastructure/src/test_infrastructure/`
  consumes Rust and optional C/tests, delivering an augmented Rust project with
  harnesses, vectors, runner metadata, and evidence. It is an implemented stage,
  not the unimplemented framework `make-tests` verb or a `test_package` producer.
- **Controls and acceptance:** `testinfra-stage` dispatches `setup`, `run`
  (default, no agent generation), `generate`, or `teardown`. Strategies are
  `cando2` and `rstest`; generation uses a bounded coding-agent loop with tests,
  optional coverage, and relocation checks. `generate` requires a model unless
  the agent is fake. Generation may accept passing tests with partial coverage
  after exhaustion; default `run` uses its convergence decision.
- **Ownership and checks:** `stage.py`, `core/executor.py`, `core/loop.py`, and
  strategy subpackages own config, acceptance, harness/vector completion, and
  telemetry. Use its own `tests/test_infrastructure/` suite. For a preserved
  final oracle, configs use `command = "run"`, `complete_vectors = false`, and
  disable coverage when it is unnecessary.

### Abstraction recovery

- **Purpose and artifacts:** `stages/abstraction-recovery/main.py` reads Rust
  and `proctor.toml`, identifies collection-recovery candidates, copies the
  project, and replaces manual C-style data structures with Rust abstractions.
  Supported candidate kinds include sequences, maps/sets, heaps, and byte
  buffers; inspect `common.py` and the selected backend for the current scope.
- **Controls and acceptance:** backends are Claude Code (`claude`, default),
  OpenCode (`opencode`), or direct OpenAI HTTP (`llm`). Candidate and repair
  bounds apply; acceptance builds with Cargo, not the supplied test package.
  No candidates or give-up normally skips; `on_give_up = "fail"` fails instead.
  Optional quality refinement retains the last building candidate on failure.
- **Ownership and checks:** shared orchestration, backend modules, and
  `quality.py` own recovery and optional unsafety/idiomaticity feedback. Its
  focused `test_main.py` and `test_llm_abstraction_recovery.py` exercise mocked
  behavior. Backend telemetry is independent of `proctor.llm` usage logs.

### Discipline repair

- **Purpose and artifacts:** `stages/discipline-repair/src/dr_agent/` consumes
  Rust and optional C/tests, copies it into an isolated workspace, instruments
  violations, drives a bounded agent repair loop, and delivers Rust plus
  evidence. `ptr-offset` is the default tool; `axm-violation` handles aliasing
  XOR mutation violations. Recorded reports can replace instrumentation in tests.
- **Controls and acceptance:** `dr-stage` validates `RunConfig`; `tools` selects
  the sequential tool suite and `max_attempts_per_target` bounds abandonment.
  Acceptance requires a build, no blocking test failure, an occurrence removed,
  and no new occurrence. Missing oracle evidence is explicit unverified state;
  leftover occurrences can yield successful partial repair with metrics.
- **Ownership and checks:** `runner.py`, `loop.py`, `gate.py`, `testrunner.py`,
  and `tools/` separate execution, assessment, testinfra oracle integration, and
  analyzers. Rejected edits revert when checkpoints are available. Use its own
  `tests/dr_agent/` suite, especially stage/loop/gate/tool tests.

### Local transformation

- **Purpose and artifacts:** `stages/local-transformation/` consumes Rust and
  an optional read-only rule set. It freezes whole-program analysis and target
  interfaces, transforms SCCs concurrently, then produces Rust and observation,
  alignment, and statistics artifacts. It does not produce an updated rule set.
- **Controls and acceptance:** a positive `workers` count bounds private SCC
  workers; `crat_dir` selects the tool checkout and `dump_llm_exchanges` enables
  prompt/response artifacts. LLM output is structurally checked; SCC candidates
  build privately before final assembly and a full build. Applied rules have
  baseline fallback. The stage requires empty manifest wrappers and does not
  run the behavioral oracle internally.
- **Ownership and checks:** Python owns orchestration, prompts, builds, usage,
  and publication; Crat owns compiler analysis, replacement, validation, and
  rule mechanics. Use `tests/test_local_transformation.py`, the
  `$local-transformation` skill for details, and `$crat` at the Rust boundary.

### Example stages

`stages/example-stage/` is a framework-independent copy-through template.
`stages/example-llm-stage/` demonstrates the shared LLM client, prompt library,
usage tracker while adding translation notes. Both
consume and produce Rust projects; their manifests and tests provide focused
starting points for a new stage rather than production translation behavior.

## Change a stage or contract

Keep inputs read-only, use declared outputs/scratch destinations, preserve
project/test metadata as required by the affected stage, and report honest
validation and telemetry. For public contract changes, coordinate framework
models/schemas, independently maintained envelope models, manifests, adapters,
and conformance tests. An extension to producible artifact kinds also changes
destinations, orchestrator state threading, and preflight validation. Keep
code/test tooling inside the standalone PROCTOR tree.
