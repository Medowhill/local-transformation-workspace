# Contracts and stages

## Contents

- Contract files
- `stage.toml`
- Input and output envelopes
- Project manifest
- Status semantics
- Current stage implementations
- Stage-change checklist

## Contract files

Treat these files as one public surface:

- `docs/stage-contract.md`: authoritative contributor-facing behavior
- `proctor/contracts/stage_io.py`: Python dataclasses and validation
- `proctor/contracts/schemas/stage_input.v1.json`
- `proctor/contracts/schemas/stage_output.v1.json`
- `proctor/contracts/stage_manifest.py`: `stage.toml`
- `proctor/contracts/manifest.py`: Rust-project `proctor.toml`
- `tests/test_stage_io.py`, `tests/test_schemas.py`, `tests/test_validate.py`

Keep the code serializers and JSON Schemas synchronized. Version 1 allows additive unknown fields. Reject a newer version; bump only for breaking changes.

## `stage.toml`

Require each stage root to contain:

```toml
id = "my_stage"
version = "0.1.0"
description = "..."
exec = ["python3", "main.py"]
warmup = ["python3", "main.py", "--build-only"] # optional

[requires]
c_project = "optional"      # required | optional | unused
rust_project = "required"
test_package = "required"
rule_set = "unused"

[produces]
rust_project = true
rule_set = false

[config]
option = { type = "integer", default = 3, doc = "..." }
```

Use lowercase letters, digits, underscores, or hyphens for the manifest id. The config table documents settings but does not enforce them. Unlisted requirements default to `unused`.

Only `rust_project` and `rule_set` are currently producible. Extend the manifest model, destination model, schemas, orchestrator state, validation, docs, and tests together before adding another producible kind.

## Input envelope

The orchestrator invokes:

```text
<exec...> --input /abs/stage_input.json --output /abs/stage_output.json
```

Use these `StageInput` groups:

- Identity: `schema_version`, `run_id`, `stage_id`, `stage_index`, optional `item`
- `inputs`: read-only absolute paths for all artifact kinds
- `outputs`: destination paths for Rust, rule set, and extra artifacts
- `config`: opaque resolved stage config
- `framework`: advisory LLM settings, usage-log path, prompt-library path, scratch workdir, budget, timeout

Do not resolve artifact paths against the stage working directory. The orchestrator sets cwd to the stage root only so relative executables and stage-local resources work.

## Output envelope

Use `StageOutput` to report:

- `status`: `success`, `failure`, or `skipped`
- matching `stage_id` and optional stage version
- actual output paths on success
- `config_used`
- LLM models, aggregate usage, and prompt ids/versions when applicable
- flat metrics, artifact-relative logs, free-form metadata
- error text only on failure

The framework accepts missing optional fields, but a well-behaved stage should report reproducibility fields honestly. Always write an envelope even after caught exceptions. Return nonzero for failure; return zero for success or skip.

## Status semantics

Use exact meanings:

- `success`: create every declared output and let the orchestrator update artifact state.
- `skipped`: create no output; let the orchestrator forward the prior state.
- `failure`: provide a nonempty `error`; do not claim a usable output.

The orchestrator rejects a declared Rust output that is not a directory and a declared rule set that is not a file. A nonzero process exit overrides a reported success/skip. A missing or malformed output is a contract failure.

With `[run] on_stage_failure = "continue"`, later stages run using the most recent valid state. Do not assume a failed transformer partially updates state.

## Project manifest

Read and update `proctor.toml` inside Rust projects after the Translation component:

```toml
target_kind = "library" # or "executable"
target_name = "example"
api_functions = ["foo"]
wrappers = [
  { wrapped = "implementation::foo_impl", wrapper = "api::foo" },
]
```

Enforce:

- Executables have no `api_functions`.
- Library API names identify functions whose external signatures must remain stable.
- Wrapper paths are crate-relative full paths.
- Inspect existing wrapper entries before introducing another wrapper.
- Copy the manifest with the project and update it when non-local transformations change wrapper relationships.

CRAT emits the initial manifest with an empty wrapper list. C2Rust output carries `config.toml`, not `proctor.toml`.

## Current stage implementations

### C2Rust adapter

Use `stages/c2rust-adapter/`, not the upstream submodule, for framework integration.

The adapter:

1. Finds the TRACTOR project root containing `CMakeLists.txt`.
2. Runs CMake file-api configuration and reads the codemodel.
3. Discovers public library functions from project headers through libclang.
4. Filters the compilation database to real executable/shared-library targets.
5. Locates `c2rust-transpile` on PATH, in `PROCTOR_CACHE_DIR`, or builds the submodule.
6. Transpiles and applies Cargo/link fixups.
7. Writes CRAT `config.toml` and `libs.json`.
8. Optionally runs `cargo build`.

Its output is library-shaped even for eventual executables. Do not run executable test vectors until CRAT's `bin` pass.

### CRAT adapter

Use `stages/crat-adapter/` for framework integration. It builds the pinned rustc-private CRAT tool once per submodule commit, constructs its sysroot/Z3 environment, and runs a cumulative pass chain ending at configurable `final_pass` (default `bin`).

The defined chain is:

```text
expand -> extern -> preprocess -> outparam -> punning -> enum ->
pointer -> io -> libc -> static -> simpl -> interface -> unsafe ->
unexpand -> split -> bin
```

After copying the final pass output, emit `proctor.toml` from CRAT `config.toml` and Cargo metadata.

### Abstraction recovery

Treat `stages/abstraction-recovery/` as a native-stage scaffold, not a completed transformer. Candidate discovery returns no candidates, so the stage currently reports `skipped`. The intended fail-open loop is present, but identification, transformation, LLM repair, and usage reporting remain TODO.

### Examples and fakes

- Copy `stages/example-stage/` for a framework-free Python stage.
- Copy patterns from `stages/example-llm-stage/` for typed envelopes, prompts, LLM calls, pricing, and usage tracking.
- Use `tests/fake_stages/fake/` to test sequencing, failure, skip, timeout, malformed output, and checkpoint behavior.

## Stage-change checklist

1. Update `stage.toml` requirements/products accurately.
2. Keep input trees immutable and refuse accidental destination overwrite.
3. Put scratch/build intermediates in `framework.workdir`; put persistent reports under `artifacts_dir`.
4. Validate the output Cargo project and test it when a package is provided.
5. Remove bulky build targets before copying/finalizing when appropriate.
6. Preserve or update `proctor.toml`.
7. Record stage-specific metrics and log names.
8. Track every LLM attempt and return aggregate usage/models/prompts.
9. Add direct stage tests and pipeline wiring tests.
10. Run `proctor validate` before an expensive end-to-end invocation.
