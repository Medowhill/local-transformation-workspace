# Historical Plan for the Local-Transformation Prototype

## Introduction

This document is a concise guide to the work that produced the
local-transformation prototype. It records implementation history, not the
current component contract.

Keep the prose summary under each phase or amendment heading to at most five
Markdown source lines; the following list of links to other files is excluded.

Start with [prototype-desc.md](prototype-desc.md) to understand current
behavior. Current tests, schemas, manifests, and implementation take precedence
over every historical plan when they disagree.

Phases 1--5 and Amendments 1--5 denote historical sets of tasks performed
together. Ownership follows when work was done, not which component was
affected. A fix to skeleton generation made while implementing a later phase
therefore remains part of that later phase.

Phase 6 denotes the completed rule-synthesis task set. Phase 7 denotes the
planned rule-application task set.

Substantive text from the former consolidated plan is preserved in the linked
detailed files. Imperative and future-tense wording in those files is
historical. Material shared by several task sets is preserved in
[prototype-plan-misc.md](prototype-plan-misc.md).

The broader [research plan](research-plan.md) and
[component specification](proctor-spec.md) describe the intended research
system. [unsupported.md](unsupported.md) consolidates the prototype's
conceptual input restrictions.

The former end-to-end evaluation phase was not executed as planned and is not
part of this history. Validation now follows a separate, currently
undocumented plan.

## Phase 1: Skeleton JSON generation

Phase 1 added the Crat tools library and thin `crat-tool make-skeleton`
operation. It reused Crat's pointer analysis to emit deterministic function and
context records, source and target signatures, statement labels, target local
types, dependencies, and parseable function skeletons without rewriting the
input project.

- [Detailed Phase 1 plan](phase-1-plan.md)
- [Phase 1 test plan](phase-1-test-plan.md)

## Phase 2: Structural validator

Phase 2 reorganized the tools library and implemented structural validation of
LLM-returned functions, labels, controls, declarations, target types, generated
temporaries, unsafe blocks, and attributes. It also performed the assigned
generator changes for binding mutability, unsafe target headers, `main`
exclusion, and the two-argument `main_0` target type.

- [Detailed Phase 2 plan](phase-2-plan.md)
- [Phase 2 test plan](phase-2-test-plan.md)

## Phase 3: Item replacement and integration

Phase 3 added safety normalization, atomic item replacement, compatibility
wrappers, export handling, call-site rewriting, and the mechanical executable
boundary. It also implemented corrections to earlier generator and validator
behavior for lifetime declarations, function-local items, and arity-only
`main_0` recognition.

- [Detailed Phase 3 plan](phase-3-plan.md)
- [Phase 3 test plan](phase-3-test-plan.md)

## Phase 4: Python orchestration

Phase 4 implemented the standalone PROCTOR stage: project preparation,
skeleton loading, graph and SCC scheduling, prompt construction, shared LLM
integration, validation/replacement invocation, transactional builds, repair,
usage reporting, and final project copying. It also corrected Crat's Expand
cleanup and normalized binding presentation in skeleton records.

- [Detailed Phase 4 plan](phase-4-plan.md)
- [Phase 4 test plan](phase-4-test-plan.md)

## Amendment 1

This work allowed a strict C-conditional-like `if` expression beneath an
otherwise non-control expression payload. Accepted nested conditionals remain
opaque to statement labeling and form one enclosing transformation region
unless the complete statement can be preserved.

- [Detailed Amendment 1 plan](amendment-1-plan.md)
- No separate test-plan document was created.

## Amendment 2

This work added conservative classification of statements as either preserved
or transformation-required. It extended skeleton metadata, validation,
replacement, orchestration, and prompting so proven-independent statements can
be restored mechanically and an entirely preserved SCC can avoid an LLM call.

- [Detailed Amendment 2 plan](amendment-2-plan.md)
- [Amendment 2 test plan](amendment-2-test-plan.md)

## Amendment 3

This work added resolved foreign-function names as per-function prompt guidance
and simplified prompt presentation to kind and final item names. The metadata
is advisory and does not change graph, validation, replacement, or compilation
semantics.

- [Detailed Amendment 3 plan](amendment-3-plan.md)
- [Amendment 3 test plan](amendment-3-test-plan.md)

## Amendment 4

This work made synthesized target types valid in the module containing each
function. It introduced scope-aware name selection, source-syntax reuse,
validated local or absolute fallbacks, safe `Option` and `Box` constructor
selection, and atomic errors when a type cannot be spelled correctly.

- [Detailed Amendment 4 plan](amendment-4-plan.md)
- [Amendment 4 test plan](amendment-4-test-plan.md)

## Amendment 5

This work added a human-readable diagnostic pairing prompt-facing source
statements with build-accepted canonical target groups and relevant raw-pointer
binding types. It preserves statement labels and reports incomplete compiler
mappings, but does not extract or apply reusable rules.

- [Detailed Amendment 5 plan](amendment-5-plan.md)
- [Amendment 5 test plan](amendment-5-test-plan.md)

## Phase 5: Validated expression observation collection

Phase 5 added normalized, typed source/target expression observations from
build-accepted SCCs using a separate labeled observation source and retained
logical-function correspondence. It left candidate and statement-pair bytes
unchanged, published versioned `observations.json`, and did not synthesize or
apply rules.

- [Detailed Phase 5 plan](phase-5-plan.md)
- [Phase 5 test plan](phase-5-test-plan.md)

## Phase 6: Rule synthesis

Phase 6 added deterministic offline synthesis of candidate expression rules
from one or more observation documents. It uses coupled, grammar-aware
anti-unification in a reusable Python library, keeps application and semantic
validation out of scope, and widens field observations to retain their owner.

- [Detailed Phase 6 plan](phase-6-plan.md)
- [Phase 6 test plan](phase-6-test-plan.md)

## Phase 7: Rule application

Phase 7 moves observation and rule ownership into Crat, applies an optional
fixed rule set during skeleton generation, and emits baseline and rule-applied
views. PROCTOR tries the applied view first and falls back to the baseline view
for bounded LLM repair after a rule-involved build failure.

- [Detailed Phase 7 plan](phase-7-plan.md)
- [Phase 7 test plan](phase-7-test-plan.md)
