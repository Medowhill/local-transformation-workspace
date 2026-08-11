---
name: prototype-desc
description: Maintain `docs/prototype-desc.md` as a cohesive, accurate, and proportionate description of the current local-transformation prototype. Use whenever planning, making, or reviewing an edit to that file, especially after prototype behavior changes.
---

# Maintain the Prototype Description

Treat `docs/prototype-desc.md` as the entry point for understanding the current
prototype, not as a record of how it evolved.

## Prepare the edit

1. Read the complete current document before changing it.
2. Verify the affected behavior against the implementation, schemas, manifests,
   and tests. Treat those sources as authoritative.
3. Read only the relevant part of `docs/prototype-plan.md` when implementation
   history or rationale is needed to understand the current behavior.
4. Identify which existing sections already own the changed concepts. Decide
   whether the behavior change is material enough for this overview before
   editing.

## Revise cohesively

- Describe only the prototype's current behavior. Do not record milestones,
  amendments, implementation chronology, or task provenance.
- Integrate new facts into the existing organization. Rewrite or replace nearby
  prose and remove obsolete claims instead of appending an update, exception,
  or change-specific section.
- Give a behavior detail in proportion to its importance to a reader's mental
  model, not its recency, implementation effort, or test volume. A recent change
  must not receive more detail than comparable existing behavior.
- Preserve the document's established tone, conceptual density, and level of
  abstraction. Summarize implementation mechanics unless they define an
  important contract, boundary, invariant, or failure mode.
- Avoid a new heading for a narrow change. Add or reorganize sections only when
  the prototype's conceptual structure has materially changed.
- Eliminate contradictions and unnecessary repetition created by the edit.
  Prefer one well-placed statement over similar statements in several sections.
- Omit documentation changes for minor implementation details that do not
  materially affect the prototype description.

## Review the result

Read the entire edited document as a standalone explanation, then inspect the
diff. Confirm that:

- the changed behavior is accurate and obsolete behavior is gone;
- the prose explains what exists now without referring to the edit or its
  history;
- new detail is no more prominent than equally important existing detail;
- headings and paragraph lengths remain balanced across the document;
- the same fact is not repeated merely to emphasize the latest work; and
- every changed line improves the current description rather than documenting
  the change process.

If the diff gives the latest change disproportionate weight, compress it and
integrate it more tightly before finishing.
