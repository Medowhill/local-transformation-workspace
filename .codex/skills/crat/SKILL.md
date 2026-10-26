---
name: crat
description: "Understand and change Crat compiler analyses, transformation passes, shared pointer inference, and crat-tool skeleton, validation, replacement, observation, and rule mechanics. Use for compiler-side work; use local-transformation for the Python stage that coordinates these tools."
---

# Crat

Crat improves C2Rust translations through compiler analysis and Rust rewrites. Its ordinary pass pipeline and its local-transformation tools share analysis code but have different transformation contracts.

## Work at the Crat root

- In this workspace the Crat root is `proctor/stages/crat/`. In a standalone checkout, locate it by its workspace `Cargo.toml` and `src/bin/crat.rs`. All implementation paths below and in the references are relative to that root.
- Treat the located Crat tree as an independent git repository. Inspect its own status and preserve unrelated parent-workspace and submodule changes.
- Prefer current implementation and tests over plans or older documentation when they disagree.
- Keep changes surgical. Crat relies on rustc-private AST/HIR/MIR mappings and many local invariants.

## Route the task

Read only the references needed for the task:

- [crat.md](references/crat.md): ordinary `crat` passes and analyses, compiler setup, configuration and dependency publication, plus `crat-finder` and `crat-merge`.
- [pointer.md](references/pointer.md): shared pointer inference and the distinction between initial skeleton decisions and ordinary expression rewriting. Combine it with the reference for the consumer being changed.
- [crat-tool.md](references/crat-tool.md): skeleton views, preservation and validation, frozen SCC replacement and assembly, finalization, deterministic `printf` lowering, observations, and reusable rules.

Use `$proctor` for stage adapters and pipeline integration, and `$local-transformation` for the Python stage's scheduling, prompts, LLM repair, build acceptance, and artifacts. The supported local-stage input contract belongs there; low-level compiler tests can intentionally cover a broader language.

## Preserve compiler boundaries

- Keep CLI argument parsing and filesystem publication in `src/bin/`; implement compiler behavior in the owning library crate.
- Reuse compiler-resolved identity and the shared AST/HIR/MIR mapping helpers. Surface spelling alone does not identify calls, bindings, types, or module paths.
- Preserve the distinction between the ordinary pointer rewrite and `crat-tool`'s fixed initial target types; shared changes need coverage for both consumers.
- Keep run-internal tool formats strict and unversioned. Reusable observation and rule documents retain their schema version; avoid adding compatibility readers for old intermediates.
- Structural validity, successful compilation, behavioral equivalence, and C ABI compatibility are separate claims. The tools do not establish the latter two.

## Verify changes

- Crat tests must not exercise the `crat-tool` CLI, change filesystem state, or use a project-root `tests/` directory.
- Keep planning labels such as phase and amendment only in planning documents, never in PROCTOR or Crat implementation surfaces.
- Run `cargo fmt` and `cargo clippy --workspace --all-targets` from the Crat root after modifying Rust source.
- Resolve Clippy warnings. Address `len_without_is_empty`, `too_many_arguments`, and `type_complexity` with targeted `#[allow(clippy::...)]` attributes rather than reshaping code for those lints.
- Run the focused package or module tests named in the selected reference. Run `cargo test --workspace` for cross-cutting changes when practical.

## Maintain this skill

Read the complete affected entrypoint or reference, then verify changed claims against the current compiler implementation and focused tests. Revise the section that owns the behavior and remove obsolete statements or repetition; describe current behavior without implementation chronology. Keep ordinary passes, shared analyses, and tool mechanics at comparable detail for their importance, regardless of recency. Move Python orchestration detail to `$local-transformation` and repository-wide context to `$proctor`. Reread the result as a standalone guide, checking scope, source paths, links, and proportional coverage.
