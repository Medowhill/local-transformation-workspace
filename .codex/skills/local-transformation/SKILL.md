---
name: local-transformation
description: Implementation guidance for PROCTOR's local-transformation stage. Use when changing or reviewing its skeleton consumption, SCC workers, prompts and repair, candidate acceptance, observation artifacts, or supported input boundary; use proctor for general pipeline work and crat for compiler internals.
---

# Local Transformation

Use this skill for the local type-directed transformation stage that is this
workspace's implementation focus. It describes the current stage and preserves
relevant research intent separately from implemented behavior.

## Work at the correct root

- Locate the standalone PROCTOR root by its `pyproject.toml`, `proctor/` package,
  and `stages/` directory. In this workspace that root is `./proctor`.
- Implementation paths below are relative to that root. The stage lives in
  `stages/local-transformation/`; Crat is the independent repository at
  `stages/crat/`. Inspect the status of each repository affected by a change.
- Keep PROCTOR self-contained. Its implementation, fixtures, tests, and tooling
  must not depend on this workspace's enclosing files or skills.
- Study current git-visible source and tests; respect git ignores rather than
  scanning generated runs, caches, or build trees.

## Route the task

Read only the references relevant to the change:

- [Stage flow](references/stage-flow.md) maps ownership, preparation,
  independent SCC execution, repair, build transactions, publication, and tests.
- [Contracts and prompts](references/contracts-and-prompts.md) covers the
  envelope, configuration, dual skeleton views, constrained output, context,
  sidecar acceptance, and observation/rule boundary.
- [Supportedness and intent](references/supportedness-and-intent.md) defines
  the input assumptions, limitations, and research goals. Read it before
  expanding accepted input or interpreting a successful build as correctness.

Use `$proctor` for orchestration, general stage contracts, shared LLM services,
and the full pipeline. Use `$crat` when changing pointer decisions, compiler
mapping, skeleton generation, structural validation, frozen source assembly,
observation extraction, or rule mechanics. Keep the Python stage responsible
for scheduling and acceptance; keep Rust-aware operations in Crat's tools
library and filesystem dispatch in its thin CLI.

## Preserve the stage boundary

- Inputs and the optional rule set are read-only. All preparation and mutation
  happen in private working projects; publication requires a successful full
  Cargo build.
- Freeze whole-program decisions once. Each SCC builds independently against
  the same target-signature stubs, with private project state and Cargo cache;
  previously accepted bodies are assembled only after all jobs succeed.
- Keep each SCC's chosen skeleton views consistent across prompting,
  validation, insertion, observations, and reporting. A rule-involved build
  failure switches the whole SCC once to `baseline` within the existing budget.
- Accept only canonical, structurally admitted functions whose sidecars match
  frozen identities and whose installed candidate builds. Extract observations
  only from the accepted `transform` regions.
- Treat observation documents as opaque in Python. Crat validates, extracts,
  merges, synthesizes, and applies them; the stage emits observations rather
  than a new rule-set output.
- Preserve the distinction between compilation, structural conformance, and
  semantic or C-ABI correctness. The local stage performs the first two and
  does not run a test package or create API compatibility wrappers.

## Verify changes

From the PROCTOR root, start with the affected cases in
`tests/test_local_transformation.py`, then run that focused file when stage
behavior changes. These tests inject fake tools and LLM clients; they do not
require live providers or the real Crat toolchain.

```bash
uv run pytest tests/test_local_transformation.py
uv run ruff check stages/local-transformation tests/test_local_transformation.py
uv run ruff format --check stages/local-transformation tests/test_local_transformation.py
```

Use the general PROCTOR checks for shared framework changes and the Crat
skill's focused package tests, formatting, and Clippy requirements for Rust
source changes. Validate affected pipeline configs before expensive real runs;
do not substitute an API-backed translation for focused unit coverage.

## Maintain this skill

Read the complete affected entrypoint or reference before revising it. Verify
claims against current code, tests, manifests, and schemas. Integrate material
changes into the sections that own them, remove obsolete or repeated claims,
and keep research intent explicitly separate. Match the existing conceptual
density: analysis, translation, validation, compilation, and learning deserve
detail according to their importance, never their recency. Omit implementation
chronology and minor mechanics that do not change the stage's mental model.
Reread the result for coherence, balanced coverage, working links, and correct
root-relative paths; check overlap with the `$proctor` and `$crat` skills.
