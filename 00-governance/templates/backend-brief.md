# Backend brief — <topic>

> **Template.** The sections below are recommended, not mandatory. Omit sections that do not apply
> to the topic. The filled brief lives at `01-product/<topic>/backend-brief.md`.
>
> **Ownership.** This brief is written by the project owner from the canon of the topic and the
> approved product decisions. The backend implementer does not edit it. Deviations, better
> approaches, and implementation discoveries are recorded in `backend-brief-feedback.md` in the
> same folder. Only the owner changes canon.

**Derived from:** <link to the topic canon>

## Mission
One paragraph: what is being built and what becomes possible when it works.

## Sequencing
What this brief depends on (other briefs, contracts, phases). Which phases may start immediately
and which wait for something else. State "no dependencies" when true.

## Source-of-truth order
1. the governing canon documents, in order
2. the approved design (if it affects the API)
3. this brief for sequencing

Canon wins over this brief. Where they appear to disagree, raise it in feedback instead of choosing.

## Current backend baseline
Verified against the code, with paths: what already exists and must keep working unchanged.
Write "none — new development" when nothing exists. Do not guess; an unverified baseline makes the
implementer distrust the rest of the document.

## Product behaviour (mandatory)
What the backend must do, in product terms. State the invariants that a plausible-looking
implementation could break silently: lifecycle rules, atomicity, uniqueness, permissions,
failure behaviour. Link to canon; do not restate it.

## Cross-cutting constraints
Links to the governing architecture documents that apply (for example tenant isolation and
authentication). Do not restate them here.

## Decisions left to the backend
Explicit list of what the implementer decides and documents: schema and naming, endpoints and
methods, DTOs, error codes, indexes and constraints, transaction strategy, module structure,
pagination. Product behaviour above must not change because of these decisions.

## Required research before implementation
Points that must be investigated and the recommended solution recorded before the affected part
is implemented. Omit if none.

## API contract
The API contract is a separate file, `api-contract.md`, in the topic folder. This brief links to
it and states the requirements it must satisfy; it does not duplicate endpoint or payload
definitions. The frontend brief is issued against the agreed API contract.

## Phases
Ordered, each independently reviewable, each naming what it depends on.

## Required tests
What must be proven, not how. Include the invariants from Product behaviour.

## Acceptance criteria
Numbered, checkable statements.

## Open questions
Points this brief deliberately leaves to implementation. Answer by number in
`backend-brief-feedback.md`. An unanswered item cannot be carried back into canon.

## Required feedback
`backend-brief-feedback.md` contains:
1. Answers to every numbered open question, by number (including ones accepted as proposed).
2. Deviations from this brief and why.
3. Behaviour decided in the backend because neither canon nor this brief defined it.
4. Which items the implementer believes change the canon (opinion only; canon is not edited).
5. Which documents were updated.

## Out of scope
Each excluded item points at the brief or canon that owns it.

## Quality checklist
- [ ] Mission is understandable without reading the code
- [ ] Baseline is verified against the code, or explicitly "none"
- [ ] Mandatory behaviour and decisions left to the backend are clearly separated
- [ ] Every acceptance criterion is checkable
- [ ] No open question hides a product decision
- [ ] No canon is restated
