# Crat local-transformation tools

## Scope and ownership

- Start with `src/bin/crat-tool.rs` for CLI arguments and file I/O, and `crates/tools/src/lib.rs` for the public Rust API.
- Keep `crat-tool` thin. Put Rust parsing, compiler analysis, validation, rewriting, observation extraction, and rule logic in `crates/tools`.
- Keep SCC scheduling, prompt/context construction, LLM requests, transactional Cargo builds, repair attempts, and usage accounting in PROCTOR's Python local-transformation stage. Do not move that orchestration into `crat-tool`.
- Treat tests and current source as authoritative; the prototype plans contain useful rationale but may describe later or superseded behavior.

## CLI operations

### MakeSkeleton

- Command: `crat-tool make-skeleton --output <records.json> [--rules <rules.json>] <input-project>`.
- Locate the project's library with `utils::find_lib_path`, compile it with `run_compiler_on_path`, call `tools::make_skeletons_with_rules`, and serialize the result with `skeletons_to_json`.
- Parse the optional schema-version-1 rule document strictly. Emit both a baseline view and an applied view; without rules, the views are equivalent.
- Keep file-system work in the binary; keep record construction in `crates/tools/src/skeleton.rs`.

### Validate

- Command: `crat-tool validate --input <request.json> --output <response.json>`.
- Parse a schema-version-1 `ValidationRequest`, validate the returned Rust structurally, and always serialize `valid`, `invalid`, or `setup_error` JSON through `validate_json`.
- Do not compile a Cargo project or modify source in this operation.

### NormalizeSafety

- Command: `crat-tool normalize-safety --output <normalized.rs> <input.rs>`.
- Parse one source file and make every source-defined free function except `main` unsafe.
- Run this after skeleton generation and before replacement; replacement rejects a current target that was not normalized.

### Replace

- Command: `crat-tool replace --request <request.json> --output <candidate.rs> --statement-pairs-output <pairs.json> --observation-source-output <observation.rs> --observation-metadata-output <metadata.json> <current-project>`.
- Parse a schema-version-1 `ReplacementRequest`, locate the current library, compile it for resolution, and call `tools::replace_items_with_observations`.
- Publish the candidate and all three sidecars together. Keep output paths distinct and leave the current project untouched.

### Observations and rules

- Use `extract-observations --metadata <metadata.json> --output <observations.json> <observation.rs>` to compile the synthetic observation source and recover typed source/target expression regions after a candidate has been accepted.
- Use `merge-observations --output <observations.json> [inputs...]` to validate and concatenate observation documents. An empty input list produces an empty schema-version-1 document.
- Use `synthesize-rules --output <rules.json> <observations...>` to deduplicate observations and deterministically synthesize generalized rules from compatible pairs. A lone observation does not produce a rule; a repeated identical observation can.
- Use `pretty-print-rules --output <rules.md> <rules.json>` for a reviewable Markdown rendering. Keep JSON as the machine-readable source of truth.

## Skeleton generation

- Use `crates/tools/src/skeleton.rs` for deterministic `ItemRecord` generation. Include source-defined free functions except `main`, plus contextual statics, constants, type aliases, enums, structs, and unions; omit modules, uses, foreign items, and other unsupported item kinds as records.
- Preserve recursive inline-module source order and assign numeric IDs from that order. Use crate-relative paths to distinguish identical final names.
- For functions, emit `annotated_source`, source/target signatures, direct and signature dependencies, resolved foreign function names, and `baseline`/`applied` `SkeletonView` values.
- Keep each view self-contained: include its skeleton, `needs_transformation`, recursive statement dispositions (`preserve`, `preserve_shell`, `transform`, or `rule_applied`), and statement-pair metadata for remaining transform labels.
- Sanitize prompt-facing ABI and `no_mangle`, display non-`ref` bindings as mutable, label source statements, and make the target skeleton unsafe.
- Preserve statements whose compiler-resolved types and expressions remain valid. Replace required payloads with parseable placeholders; in the applied view, replace a transform region only when one rule covers every selected expression in that region and the result passes structural checks.
- Return `GenerationError` for unsupported body shapes such as empty statements, function-local items, non-block match arms, invalid nested controls, or an AST/HIR mapping mismatch.

## Validation and preservation

- Use `crates/tools/src/validator.rs` for request parsing, expected-skeleton setup checks, deterministic failure ordering, and repair-oriented diagnostics.
- Require exactly the requested top-level functions. Compare lifetime declarations, parameter names/types, return types, existing bindings, and explicit local types structurally rather than by raw text.
- Enforce canonical `#[proctor(N)]` statement groups, control roles and descendants, preserved regions, and generated temporary names of the form `proctor_temp_var_<n>`.
- Reject function-local items, explicit unsafe blocks, unexpected body attributes, misplaced or duplicated labels, and temporaries escaping their expansion group.
- Distinguish malformed requests or inconsistent expected skeletons as `setup_error`; report LLM-transformable failures as `invalid`.
- Keep shared label-tree validation and canonical restoration in `crates/tools/src/preservation.rs`. The replacer must restore preserved groups independently instead of trusting that validation already ran.

## Item replacement

- Use `crates/tools/src/item_replacer.rs` to resolve targets by full crate-relative path against the current HIR-mapped surface AST.
- Compose the accepted target lifetime declaration, signature, and body with the current function's visibility and metadata; remove PROCTOR labels before emitting source.
- When parameter or return types change, keep the transformed implementation at its original path, create a collision-free sibling compatibility wrapper, and redirect untransformed external callers through compiler-resolved call sites.
- Keep calls within the current SCC direct. Reject required rewrites hidden in macro token input.
- Preserve export behavior by moving `no_mangle`/`export_name` responsibility to a generated wrapper when required.
- Handle the special two-argument `main_0` boundary mechanically instead of generating a normal compatibility wrapper.
- Limit wrapper conversions to the explicit raw-pointer/reference/slice/box cases implemented by `input_conversion` and `output_conversion`; return `UnsupportedConversion` for other pairs.

## Observation and rule mechanics

- Use `crates/tools/src/observation.rs` for replacement metadata, callable correspondence, typed expression extraction, pointer anchors, semantic identities, source-region selection, and canonical observation documents.
- Keep extraction downstream of successful candidate compilation. Validate sidecar digests and accepted/current item correspondence before trusting the synthetic observation source.
- Use `crates/tools/src/rule.rs` for strict schema-version-1 observation/rule documents, deterministic merging and synthesis, rule matching, specificity, substitution cost, and semantic canonicalization; use `rule/markdown.rs` only for presentation.
- Keep rule application inside skeleton generation. A rule-applied candidate remains provisional: PROCTOR may fall back from the applied view to the baseline view when the candidate does not compile.

## Focused verification

- Run `cargo test -p tools skeleton::tests` for record, annotation, target-type, or preservation-classification changes.
- Run `cargo test -p tools validator::tests` for validation or diagnostic changes.
- Run `cargo test -p tools item_replacer::tests` for normalization, replacement, wrappers, conversions, or call rewriting.
- Run `cargo test -p tools observation::tests` for observation source, metadata, correspondence, region selection, or extraction changes.
- Run `cargo test -p tools rule::tests` for document validation, synthesis, matching, ordering, or Markdown changes.
- Run `cargo test -p tools` for shared preservation or public tools API changes.
- When changing the CLI contract or Python integration, also run PROCTOR's focused local-transformation protocol/tooling tests.
