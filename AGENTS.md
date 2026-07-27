You are an expert in Rust and Python.

# Project-Specific Instructions

* Read `research-plan.md` whenever you need to understand the goal of the entire
  research project.
* PROCTOR is the name of the entire research project, and `./proctor` is the
  top-level pipeline implementation.
* You can use the `$proctor` skill for quick on-boarding of the PROCTOR
  implementation; note that this skill was written when the PROCTOR
  implementation is the repository root, not under `./proctor`, so be aware of
  paths.
* While PROCTOR includes various components, the purpose of this workspace is to
  focus on implementing the local transformation part.
* Read `proctor-spec.md` whenever you need to understand the specification of
  each component.
* Treat planning-specific labels such as "phase", "amendment", `P1`, and `A1`
  as planning-document terminology only. Do not use them in the PROCTOR or Crat
  implementation, including code, comments, identifiers, tests, fixtures,
  diagnostics, configuration values, or file names; use names that describe
  current behavior instead.
* Keep `./proctor` self-contained as a standalone repository. Its code, tests,
  build configuration, fixtures, and checked-in tooling must not read, import,
  execute, or otherwise depend on files outside `./proctor`; the enclosing
  workspace root and its research or planning documents are development context
  only.
* Crat is our tool to improve C2Rust's translation through several passes mainly
  using static analysis. Its source code is under `./proctor/stages/crat`. While
  some part of Crat is irrelevant to the current project, many parts are
  valuable, and you may extend it.
* Run `cargo fmt` and `cargo clippy` under `./proctor/stages/crat` after
  modifying its source code. While you should resolve all Clippy warnings, the
  following warnings can be addressed only by inserting `#[allow(clippy::...)]`
  in the source code:
  - `len_without_is_empty`
  - `too_many_arguments`
  - `type_complexity`

# General Rules

## Think Before Coding

**Don't assume. Don't hide confusion. Surface tradeoffs.**

Before implementing:
- State your assumptions explicitly. If uncertain, ask.
- If multiple interpretations exist, present them - don't pick silently.
- If a simpler approach exists, say so. Push back when warranted.
- If something is unclear, stop. Name what's confusing. Ask.

## Simplicity First

**Minimum code that solves the problem. Nothing speculative.**

- No features beyond what was asked.
- No abstractions for single-use code.
- No "flexibility" or "configurability" that wasn't requested.
- No error handling for impossible scenarios.
- If you write 200 lines and it could be 50, rewrite it.

Ask yourself: "Would a senior engineer say this is overcomplicated?" If yes, simplify.

## Surgical Changes

**Touch only what you must. Clean up only your own mess.**

When editing existing code:
- Don't "improve" adjacent code, comments, or formatting.
- Don't refactor things that aren't broken.
- Match existing style, even if you'd do it differently.
- If you notice unrelated dead code, mention it - don't delete it.

When your changes create orphans:
- Remove imports/variables/functions that YOUR changes made unused.
- Don't remove pre-existing dead code unless asked.

The test: Every changed line should trace directly to the user's request.
