# Research purpose and intended interfaces

This reference preserves project goals and useful design intent. It describes
why the pipeline exists and what future work should support; current code,
schemas, stage validators, and tests determine implemented behavior.
Implementation paths here are relative to the PROCTOR root (`./proctor/`).

## Research goal

A purely symbolic C-to-Rust translator faces both non-local ownership/data
representation problems and a large number of local expression/type rewrite
combinations. Neural approaches can recover abstractions and rewrite bodies
but introduce runtime and monetary cost. PROCTOR aims for correctness near or
better than symbolic approaches, idiomaticity near neural approaches, and cost
near symbolic approaches by combining them.

The research pipeline starts from C2Rust-generated unsafe Rust. Crat's symbolic
preparation resolves internal externs, normalizes troublesome syntax, removes
dead code, and simplifies expressions before neural transformations. Non-local
abstraction recovery and discipline repair address changes to representation,
control flow, ownership, or signatures. Whole-program analysis then fixes local
target types/signatures so constrained body translation does not independently
redesign global interfaces.

Successful aligned translations supply typed, context-constrained rules:
source syntax plus constraints on types, semantic properties, and callee
interfaces yields target syntax. Applying known rules before LLM generation
should reduce remaining holes and amortize cost across functions and programs.
This is a goal to evaluate, not an assumption that compilation proves semantic
correctness or that the local stage already updates a rule library.

## Component intentions and current boundaries

| Component | Intended contribution | Current boundary |
|---|---|---|
| Translation | Cargo-buildable Rust with target/API metadata | Implemented by C2Rust and CRAT adapters; CRAT creates `proctor.toml` |
| Test generation | Reusable behavioral oracle for translation and repair | Test infrastructure delivers harnesses/vectors within Rust projects; the framework's separate test-package kind is still input-only |
| Abstraction recovery | Replace manually implemented abstractions with standard Rust or suitable crate abstractions | Multiple collection kinds/backends exist; built-in acceptance is a Cargo build, with optional quality refinement |
| Discipline repair | Localize ownership/aliasing/access patterns requiring non-local repair | Implemented instrumentation tools cover pointer offsets and aliasing violations; acceptance/reporting remain evidence-dependent |
| Local transformation | Analysis-fixed types, constrained aligned bodies, reusable learned rules | Fixed-analysis concurrent SCC transformation, structural validation, builds, and observation extraction exist; rule synthesis is an explicit Crat operation outside the stage |

The original shared assumptions are Linux, one target and build configuration
per project, Cargo builds, and function-only library exports, with libraries
represented as `cdylib`. Treat these as the experimental interface scope;
inspect adapters and manifests for actual supported layouts and toolchains.

## API compatibility and validation intent

The intended distinction is between executable internals, whose signatures may
change, and library exports, whose external API should remain stable. A
compatibility wrapper may preserve an exported signature while delegating to a
rewritten implementation. `proctor.toml` retains `api_functions` and
crate-relative wrapper relationships to represent such boundaries; non-local
stages should account for existing relationships before introducing nested ones.

The current local prototype requires an empty wrapper list, does not introduce
persistent compatibility wrappers, and does not establish exported-signature
compatibility through finalization. Earlier proposals for temporary migration
wrappers and API-preserving output must not be read as implemented local-stage
behavior. Use the local-transformation and Crat skills for current details.

Compilation, behavioral tests, and symbolic/dynamic checks serve different
purposes. The research design calls for all available evidence and bounded
repair with precise diagnostics. Current stages provide different validation:
C2Rust/CRAT optionally build, abstraction recovery builds, discipline repair
checks builds/tests/instrumentation, and local transformation structurally
checks LLM output and builds. Pipeline configs may explicitly run the generated
oracle after the final transformation. Keep these distinctions visible when
reporting correctness or evaluating broader abstraction/repair coverage.
