---
name: crat
description: "Domain guide for Crat's `crat` pass/analysis CLI and `crat-tool` local-transformation utilities, including rustc-private APIs and shared pointer analysis. Use when modifying or understanding Crat passes, configs, analyses, pointer decisions or rewrites, skeleton generation, structural validation, safety normalization, item replacement, or generated Rust transformations."
---

# Crat

## Route the task

Read only the references required for the task:

- Read [crat.md](references/crat.md) for the `crat` CLI, pass pipeline, standalone analyses, configs, dependency side effects, or non-pointer transformation code.
- Read [crat-tool.md](references/crat-tool.md) for `crat-tool`, `crates/tools`, skeleton records, preservation labels, structural validation, safety normalization, or item replacement.
- Read [pointer.md](references/pointer.md) together with `crat.md` for the ordinary pointer pass or `pointer_replacer` work.
- Read [pointer.md](references/pointer.md) together with `crat-tool.md` for skeleton target types, initial pointer decisions, or changes to shared pointer analysis used by the tools.

Do not load `crat.md` for tool-only work or `crat-tool.md` for pass-only work.

## Work at the Crat root

- Locate the Crat root by its workspace `Cargo.toml` and `src/bin/crat.rs`; it may be `crat/` in a standalone checkout or `proctor/stages/crat/` in PROCTOR.
- Treat the located Crat tree as an independent git repository. Inspect its own status and preserve unrelated parent-workspace and submodule changes.
- Prefer current implementation and tests over plans or older documentation when they disagree.
- Keep changes surgical. Crat relies on rustc-private AST/HIR/MIR mappings and many local invariants.

## Verify changes

- Crat tests must not exercise the `crat-tool` CLI, change filesystem state, or use a project-root `tests/` directory.
- Keep planning labels such as phase and amendment only in planning documents, never in PROCTOR or Crat implementation surfaces.
- Run `cargo fmt` and `cargo clippy --workspace --all-targets` from the Crat root after modifying Rust source.
- Resolve Clippy warnings; use targeted `#[allow(clippy::...)]` only for `len_without_is_empty`, `too_many_arguments`, or `type_complexity` when necessary.
- Run the focused package or module tests named in the selected reference. Run `cargo test --workspace` for cross-cutting changes when practical.
