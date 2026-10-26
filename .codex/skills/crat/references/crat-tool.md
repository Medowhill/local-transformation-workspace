# Crat local-transformation tools

## Scope and ownership

- Start with `src/bin/crat-tool.rs` for CLI arguments and file I/O, and `crates/tools/src/lib.rs` for the public Rust API.
- All implementation paths are relative to the Crat root, `proctor/stages/crat/` in this workspace. Compiler-backed operations use the pinned toolchain and `deps_crate` setup described in [crat.md](crat.md#entrypoint-and-compiler-utilities).
- Keep `crat-tool` thin. Put Rust parsing, compiler analysis, validation, rewriting, deterministic `printf` lowering, observation extraction, and rule logic in `crates/tools`.
- Keep SCC scheduling, prompt/context construction, LLM requests, transactional Cargo builds, repair attempts, and usage accounting in PROCTOR's Python local-transformation stage. Do not move that orchestration into `crat-tool`.
- Use `$local-transformation` for stage-level input assumptions and acceptance policy. Compiler parsing, structural validation, and a successful Cargo build do not establish semantic equivalence or preservation of an exported C ABI.

## CLI operations

- `make-skeleton --output <records.json> [--frozen-output <frozen-project.json>] [--rules <rules.json>] <analysis-project>` compiles the library and emits records with baseline and applied views. The optional frozen bundle captures normalized analysis source, the target stub template, function records, and resolved observation helpers in that compiler session. The optional rule document is parsed strictly, including `rules` and `printf_rules`.
- `validate --input <request.json> --output <response.json>` structurally checks returned Rust and emits `valid`, `invalid`, or `setup_error`; it does not build a project.
- `normalize-safety --output <normalized.rs> <input.rs>` makes source-defined free functions except `main` unsafe. Run it after skeleton generation on the analysis source, before assembling target functions.
- `make-initial [--frozen <frozen-project.json>] --output <initial.rs> <analysis-project>` creates the target template. With `--frozen`, it checks the analysis source and emits the frozen target signatures with `todo!()` bodies. Without it, the retained parser-only path emits zero-argument pending stubs. Both retain non-function context and omit `main`.
- `add-functions --frozen <frozen-project.json> --request <request.json> --current-project <worker-project> --output <candidate.rs> --statement-pairs-output <pairs.json> --observation-source-output <observation.rs> --observation-metadata-output <metadata.json> <analysis-project>` requires the frozen bundle, checks the analysis source and worker template against it, installs one SCC's functions, and publishes candidate plus sidecars together. Outputs must be distinct and outside both input projects.
- `assemble-project --frozen <frozen-project.json> --request <accepted-sccs.json> --output <assembled.rs>` combines explicitly accepted SCC members from their independent candidate sources. The request is an array of `AcceptedScc { members, source }` records.
- `finalize-project [--frozen <frozen-project.json>] --manifest <proctor.toml> --current-project <assembled-project> --output <final.rs> --manifest-output <final-proctor.toml> <analysis-project>` restores `main`, including the supported two-argument `main_0` boundary. The frozen path uses stored compiler facts; the nonfrozen path recompiles the analysis source. The manifest must declare `target_kind`, `target_name`, `api_functions`, and an empty `wrappers` array; output preserves its contents. The Python stage builds the finalized project.
- `extract-observations --metadata <metadata.json> --output <observations.json> <observation.rs>` compiles the synthetic observation source and recovers typed expression regions after candidate acceptance.
- `merge-observations --output <observations.json> [inputs...]` validates and concatenates observation documents; no inputs produce an empty schema-version-1 document. `synthesize-rules --output <rules.json> <observations...>` deduplicates observations and synthesizes rules from compatible pairs; a lone observation does not produce a rule, but a repeated identical one can. `pretty-print-rules --output <rules.md> <rules.json>` renders rules for review.

Skeleton records, the frozen bundle, validation requests/responses, replacement
and assembly requests, statement-pairs sidecars, and replacement-observation
metadata use strict unversioned shapes. Do not accept older format variants.
Reusable observation and rule documents retain schema version 1.

## Skeleton generation

- Use `crates/tools/src/skeleton.rs` for deterministic `ItemRecord` generation. Include source-defined free functions except `main`, plus contextual statics, constants, type aliases, enums, structs, and unions; omit modules, uses, foreign items, and other unsupported item kinds as records.
- Preserve recursive inline-module source order and assign numeric IDs from that order. Use crate-relative paths to distinguish identical final names.
- For functions, emit `annotated_source`, source/target signatures, direct and signature dependencies, advisory format/proctor-libc/foreign-symbol metadata, and `baseline`/`applied` `SkeletonView` values.
- Keep each view self-contained: include its skeleton, `needs_transformation`, recursive statement dispositions (`preserve`, `preserve_shell`, `transform`, `rule_applied`, or `mechanical`), and statement-pair metadata for reportable transform/mechanical labels.
- Sanitize prompt-facing ABI and `no_mangle`, display non-`ref` bindings as mutable, label source statements, and make the target skeleton unsafe.
- Preserve original binding identities, patterns, control roles, and label-tree structure. Closures and restricted ternary-style `if` expressions are opaque regions, so their interiors receive no independent labels or target binding annotations.
- Preserve statements whose compiler-resolved types and expressions remain valid. Replace required payloads with parseable placeholders; in the applied view, select rules independently for expression regions and install the statement atomically only when every selected region is covered and structural checks pass.
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

## Frozen replacement and assembly

- `crates/tools/src/item_replacer.rs` owns common request validation, canonical preservation, metadata composition, and retained in-memory additive APIs. `item_replacer/frozen.rs` owns `freeze_project`, `transform_frozen_scc`, `assemble_frozen_project`, and `finalize_frozen_project`, which implement the stage's independent-worker path.
- Freeze once during skeleton compilation: store safety-normalized analysis source, a target template with fixed signatures and stub bodies, function records, and compiler-resolved source-copy/stub helpers. Worker replacement and final assembly parse these frozen facts without recompiling the analysis crate or recomputing pointer decisions.
- Each independent worker starts from the exact frozen target template. Require an empty `accepted_correspondence`, an unchanged analysis source after safety normalization, matching non-function context, and a requested view equal to the recorded baseline or applied view. Install only the requested SCC's bodies; other functions retain typed target-interface stubs.
- Keep observation-only source separate from the Cargo candidate. It contains labeled current implementations, current original-source copies, and original-signature helper stubs for resolved source calls, including recursion. Metadata lists other frozen functions as dependency interfaces rather than accepted transformations. Required rewrites hidden in source macro tokens are unsupported.
- Assemble each accepted member exactly once, in original item order, preserving frozen headers and non-function context. Reject missing or duplicate members and accepted functions that still equal their frozen whole-function stub. Finalization restores the original `main`, or the fixed boundary for supported two-argument `main_0`; it validates manifest target/API names and requires no wrappers. It does not verify original exported signatures.

The retained `add_functions_with_observations` API handles a growing partial
target and prior accepted correspondence. Keep that API's behavior distinct from
the frozen CLI path. Both retain logical and implementation paths and use no
persistent compatibility wrappers; source-copy helpers belong only to observation
extraction, never the published target.

## Observation and rule mechanics

- Use `crates/tools/src/observation.rs` for replacement metadata, callable correspondence, typed expression extraction, pointer anchors, semantic identities, source-region selection, and canonical observation documents.
- Keep extraction downstream of successful candidate compilation. The single replacement-metadata shape requires candidate, sidecar, observation-source, and frozen-bundle digests, explicit dependency correspondence, current item correspondence, source stubs, and `unfilled_stub_items`. Dependencies must not be reported as accepted transformations. Validate digests and correspondence before trusting the synthetic observation source.
- Use `crates/tools/src/rule.rs` for strict schema-version-1 observation/rule documents, deterministic merging and synthesis, rule matching, specificity, substitution cost, and semantic canonicalization. Keep resolved `scanf`/`fscanf`/`sscanf` and `xj_scanf` format literals rigid during synthesis; use `rule/markdown.rs` only for presentation. Keep the ordinary `observations`/`rules` and format-specific `printf_observations`/`printf_rules` arrays present and distinct.
- Keep rule application inside skeleton generation. A rule-applied candidate remains provisional: PROCTOR may fall back from the applied view to the baseline view when the candidate does not compile.

## Focused verification

- Run `cargo test -p tools skeleton::tests` for record, annotation, target-type, or preservation-classification changes.
- Run `cargo test -p tools validator::tests` for validation or diagnostic changes.
- Run `cargo test -p tools item_replacer::tests` for initial-source construction, additive assembly, finalization, or call rewriting.
- Run `cargo test -p tools item_replacer::frozen::tests` for frozen interfaces, independent replacement, accepted-member assembly, or frozen finalization.
- Run `cargo test -p tools printf::tests` for `printf` recognition, format conversion, or print-template validation.
- Run `cargo test -p tools observation::tests` for observation source, metadata, correspondence, region selection, or extraction changes.
- Run `cargo test -p tools rule::tests` for document validation, synthesis, matching, ordering, or Markdown changes.
- Run `cargo test -p tools` for shared preservation or public tools API changes.
- When changing the CLI contract or Python integration, also run PROCTOR's focused local-transformation protocol/tooling tests.
