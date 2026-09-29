# Crat local-transformation tools

## Scope and ownership

- Start with `src/bin/crat-tool.rs` for CLI arguments and file I/O, and `crates/tools/src/lib.rs` for the public Rust API.
- Keep `crat-tool` thin. Put Rust parsing, compiler analysis, validation, rewriting, deterministic `printf` lowering, observation extraction, and rule logic in `crates/tools`.
- Keep SCC scheduling, prompt/context construction, LLM requests, transactional Cargo builds, repair attempts, and usage accounting in PROCTOR's Python local-transformation stage. Do not move that orchestration into `crat-tool`.
- Treat tests and current source as authoritative; the prototype plans contain useful rationale but may describe later or superseded behavior.

## CLI operations

- `make-skeleton --output <records.json> [--rules <rules.json>] <analysis-project>` compiles the library and emits schema-version-1 records with baseline and applied views. The optional rule document is parsed strictly, including `rules` and `printf_rules`.
- `validate --input <request.json> --output <response.json>` structurally checks returned Rust and emits `valid`, `invalid`, or `setup_error`; it does not build a project.
- `normalize-safety --output <normalized.rs> <input.rs>` makes source-defined free functions except `main` unsafe. Run it after skeleton generation on the analysis source, before assembling target functions.
- `make-initial --output <initial.rs> <analysis-project>` creates the partial target: it retains non-function context, replaces source-defined function items with same-name, zero-argument pending stubs, and omits `main`. Build this initial library before adding functions.
- `add-functions --request <request.json> --current-project <partial-project> --output <candidate.rs> --statement-pairs-output <pairs.json> --observation-source-output <observation.rs> --observation-metadata-output <metadata.json> <analysis-project>` compiles the complete analysis source, adds an SCC's accepted functions to the partial target, and publishes candidate plus sidecars together. Outputs must be distinct and outside both input projects.
- `finalize-project --manifest <proctor.toml> --current-project <partial-project> --output <final.rs> --manifest-output <final-proctor.toml> <analysis-project>` restores `main` after function acceptance, including the supported two-argument `main_0` boundary. The input manifest must declare `target_kind`, `target_name`, `api_functions`, and an empty `wrappers` array; the output manifest preserves its contents. Build the finalized project.
- `extract-observations --metadata <metadata.json> --output <observations.json> <observation.rs>` compiles the synthetic observation source and recovers typed expression regions after candidate acceptance.
- `merge-observations --output <observations.json> [inputs...]` validates and concatenates observation documents; no inputs produce an empty schema-version-1 document. `synthesize-rules --output <rules.json> <observations...>` deduplicates observations and synthesizes rules from compatible pairs; a lone observation does not produce a rule, but a repeated identical one can. `pretty-print-rules --output <rules.md> <rules.json>` renders rules for review.

## Skeleton generation

- Use `crates/tools/src/skeleton.rs` for deterministic `ItemRecord` generation. Include source-defined free functions except `main`, plus contextual statics, constants, type aliases, enums, structs, and unions; omit modules, uses, foreign items, and other unsupported item kinds as records.
- Preserve recursive inline-module source order and assign numeric IDs from that order. Use crate-relative paths to distinguish identical final names.
- For functions, emit `annotated_source`, source/target signatures, direct and signature dependencies, resolved foreign function and static names, and `baseline`/`applied` `SkeletonView` values.
- Keep each view self-contained: include its skeleton, `needs_transformation`, recursive statement dispositions (`preserve`, `preserve_shell`, `transform`, `rule_applied`, or `mechanical`), and statement-pair metadata for reportable transform/mechanical labels.
- Sanitize prompt-facing ABI and `no_mangle`, display non-`ref` bindings as mutable, label source statements, and make the target skeleton unsafe.
- Preserve statements whose compiler-resolved types and expressions remain valid. Replace required payloads with parseable placeholders; in the applied view, replace a transform region only when one rule covers every selected expression in that region and the result passes structural checks.
- Return `GenerationError` for unsupported body shapes such as empty statements, function-local items, non-block match arms, invalid nested controls, or an AST/HIR mapping mismatch.

## Deterministic `printf` lowering

- Use `crates/tools/src/printf.rs` for compiler-resolved eligibility, C-format conversion, canonical `::std::print!` templates, and template validation.
- Lower only supported calls to the local non-unwinding C-variadic `printf` symbol with a recoverable literal format. Keep unsupported calls on the ordinary transformation path.
- Mark a converted call with no value arguments as `mechanical`; its complete print statement bypasses the LLM but still goes through replacement and Cargo-build acceptance. Keep argument-bearing templates as `transform` regions with trusted format metadata and `todo!()` value holes, or as `rule_applied` when every argument is covered.
- Keep `printf_template` metadata internal to the skeleton/replacement protocol. Do not expose it as editable prompt content or accept an altered format, placeholder count, or mechanical payload.

## Validation and preservation

- Use `crates/tools/src/validator.rs` for request parsing, expected-skeleton setup checks, deterministic failure ordering, and repair-oriented diagnostics.
- Require exactly the requested top-level functions. Compare lifetime declarations, parameter names/types, return types, existing bindings, and explicit local types structurally rather than by raw text.
- Enforce canonical `#[proctor(N)]` statement groups, control roles and descendants, preserved regions, and generated temporary names of the form `proctor_temp_var_<n>`.
- Reject function-local items, explicit unsafe blocks, unexpected body attributes, misplaced or duplicated labels, and temporaries escaping their expansion group.
- Distinguish malformed requests or inconsistent expected skeletons as `setup_error`; report LLM-transformable failures as `invalid`.
- Keep shared label-tree validation and canonical restoration in `crates/tools/src/preservation.rs`. The replacer must restore preserved groups independently instead of trusting that validation already ran.

## Additive function assembly

- Use `crates/tools/src/item_replacer.rs` for `make_initial_source`, `add_functions_with_observations`, and `finalize_additive_source`.
- Keep the complete analysis project separate from the partial target. `add_functions_with_observations` requires matching non-function context, resolves requested functions by full crate-relative path, composes each accepted signature and body with current metadata, and replaces its pending stub at the original path.
- Pass prior accepted correspondence in the schema-version-1 request. Additive correspondence keeps logical and implementation paths equal and `wrapper_path` absent; signature changes do not create persistent compatibility wrappers. The partial candidate receives no generated source copies.
- For observation extraction, synthesize labeled implementations and copies of the original functions; rewrite compiler-resolved calls to those copies. Add temporary source stubs only where earlier accepted signature changes require them. Reject required rewrites hidden in macro token input.
- Finalization restores `main` from the analysis source, or creates the fixed `main` boundary for supported two-argument `main_0`. Validate the manifest target/API names and keep its `wrappers` array empty; finalization does not check exported signature compatibility.

## Observation and rule mechanics

- Use `crates/tools/src/observation.rs` for replacement metadata, callable correspondence, typed expression extraction, pointer anchors, semantic identities, source-region selection, and canonical observation documents.
- Keep extraction downstream of successful candidate compilation. Validate sidecar digests and accepted/current item correspondence before trusting the synthetic observation source. Additive replacement metadata uses schema version 2 and records any temporary source stubs.
- Use `crates/tools/src/rule.rs` for strict schema-version-1 observation/rule documents, deterministic merging and synthesis, rule matching, specificity, substitution cost, and semantic canonicalization. Keep resolved `scanf`/`fscanf`/`sscanf` and `xj_scanf` format literals rigid during synthesis; use `rule/markdown.rs` only for presentation. Keep the ordinary `observations`/`rules` and format-specific `printf_observations`/`printf_rules` arrays present and distinct.
- Keep rule application inside skeleton generation. A rule-applied candidate remains provisional: PROCTOR may fall back from the applied view to the baseline view when the candidate does not compile.

## Focused verification

- Run `cargo test -p tools skeleton::tests` for record, annotation, target-type, or preservation-classification changes.
- Run `cargo test -p tools validator::tests` for validation or diagnostic changes.
- Run `cargo test -p tools item_replacer::tests` for initial-source construction, additive assembly, finalization, or call rewriting.
- Run `cargo test -p tools printf::tests` for `printf` recognition, format conversion, or print-template validation.
- Run `cargo test -p tools observation::tests` for observation source, metadata, correspondence, region selection, or extraction changes.
- Run `cargo test -p tools rule::tests` for document validation, synthesis, matching, ordering, or Markdown changes.
- Run `cargo test -p tools` for shared preservation or public tools API changes.
- When changing the CLI contract or Python integration, also run PROCTOR's focused local-transformation protocol/tooling tests.
