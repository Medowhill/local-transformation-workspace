---
name: proctor
description: Repository guide for the PROCTOR C-to-Rust pipeline. Use for general onboarding, stage composition and contracts, orchestration, configuration, shared LLM/context infrastructure, testing, and evaluation. Use local-transformation for detailed local-stage work and crat for compiler internals.
---

# PROCTOR

PROCTOR is the research project and its standalone orchestration repository.
This skill explains the whole pipeline at a consistent level; detailed local
transformation and Crat implementation guidance have separate skills.

## Work at the PROCTOR root

- Locate the root containing `pyproject.toml`, the `proctor/` Python package,
  `configs/`, and `stages/`. In this workspace it is `./proctor/`.
- All implementation paths and commands below and in this skill's references
  are relative to that PROCTOR root, unless another root is explicitly named.
- Inspect its git/submodule status and preserve unrelated changes. The stage
  checkouts listed in `.gitmodules` are independent repositories.
- Use git-aware searches such as `rg` and `git ls-files`; exclude ignored run,
  build, cache, and generated content from repository study.
- Keep PROCTOR self-contained: code, tests, configs, fixtures, and tooling must
  not depend on the enclosing workspace or its skill files.

## Route the task

Read only the references relevant to the request:

- [architecture.md](references/architecture.md): pipeline, artifact flow,
  package ownership, and run lifecycle.
- [contracts-and-stages.md](references/contracts-and-stages.md): envelope and
  manifest contracts, all stages, their validation, and their boundaries.
- [configuration-and-runs.md](references/configuration-and-runs.md): config,
  CLI, checkpoint/resume, bench, LLM usage, prompts, and context retrieval.
- [testing-and-status.md](references/testing-and-status.md): test ownership,
  component environments, real-toolchain coverage, and current limitations.
- [research-intent.md](references/research-intent.md): project motivation,
  intended component interfaces, and goals beyond the current implementation.

Use `$local-transformation` for the local stage's scheduling, prompting,
worker builds, observations, and publication. Use `$crat` for compiler passes,
analysis, `crat-tool`, skeletons, validation, and rule mechanics. Read both when
changing the Python/Rust boundary.

## Preserve boundaries

- Treat current code, tests, schemas, and stage validators together as evidence
  of implemented behavior. READMEs, manifest comments, and research intent may
  lag them; identify discrepancies before changing behavior.
- Keep stages independently executable, single-case programs invoked through
  absolute-path JSON envelopes with `--input` and `--output`. They may use
  shared libraries but the orchestrator must not import their implementations.
- Read input artifacts without modifying them; write transformed copies only
  to output destinations and scratch data to the supplied work directory.
- Keep envelope translation and upstream-tool invocation in adapters; change
  the relevant tool checkout when the task targets compiler behavior.
- Coordinate public shape changes across models, serializers, JSON Schemas,
  manifests, artifact wiring, fixtures, and affected tests. Breaking envelope
  changes require a schema-version decision; additive readers ignore unknown
  fields and reject newer versions.
- Treat `proctor.toml` as Rust-project target/API metadata after CRAT, distinct
  from stage envelopes and the test stage's `testinfra.toml`.
- Keep credentials in environment variables. Shared-client stages record every
  LLM attempt, including errors, in the supplied usage log. Independent agent
  stages have their own telemetry; do not assume `proctor report` sees it.
- Do not version intermediate formats produced and consumed within one run by
  the same implementation checkout. Use one current shape with strict
  field/type validation and identity/digest checks. Do not version diagnostic
  or human-facing report artifacts without automated consumers. Retain versions
  for independently consumed reusable contracts, such as stage envelopes and
  observation/rule documents, and numeric prompt versions for provenance.

## Verify changes

Choose focused checks from [testing-and-status.md](references/testing-and-status.md).
From the PROCTOR root, the framework checks are:

```bash
uv run pytest tests/test_<area>.py
uv run pytest
uv run ruff check .
uv run ruff format --check .
uv run mypy proctor
```

Stage submodules have separate environments and suites; framework pytest does
not collect all stage-local tests. Run real-toolchain `e2e` coverage only when
relevant and prerequisites are available. Use `uv run proctor validate -c CFG`
for inexpensive pipeline wiring checks before an authorized run.

## Maintain this skill

Read the complete affected entrypoint/reference and verify material claims
against current code, tests, schemas, and manifests. Revise the section that
owns the behavior, removing obsolete claims and duplication; do not append
change history. Keep the general pipeline and all stages at comparable depth,
move detailed local/compiler mechanics to their owning skills, and label
research intent separately from implemented behavior. Give new and existing
features detail according to their importance, not recency or test volume.
Review the complete result and diff for accurate paths, consistent terminology,
balanced coverage, and working reference links.
