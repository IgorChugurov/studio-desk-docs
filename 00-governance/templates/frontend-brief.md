# Frontend brief — <topic>

> **Template.** The sections below are recommended, not mandatory. Omit sections that do not apply
> to the topic. The filled brief lives at `01-product/<topic>/frontend-brief.md`.
>
> **Ownership.** This brief is written by the project owner from the canon of the topic, the
> approved design, and the agreed API contract. The frontend implementer does not edit it.
> Deviations, better approaches, and implementation discoveries are recorded in
> `frontend-brief-feedback.md` in the same folder. Only the owner changes canon.
>
> **When issued.** Not before the design is approved and `api-contract.md` is agreed, because this
> brief derives from both. A finished backend implementation is not required.

**Derived from:** <links to the topic canon, design brief, and api-contract.md>
**Figma:** <links to files and nodes>

## Mission
One paragraph: which surface is being built and what becomes possible when it works.

## Source-of-truth order
1. the governing canon documents, in order
2. the approved design — name the design brief and the Figma nodes
3. `api-contract.md` for the API this UI consumes
4. this brief for sequencing

Canon wins over the design; the design wins over personal preference. Where the design and the
agreed API disagree, report it in feedback instead of choosing.

## Surfaces and placement
Which surface(s) are affected (platform admin, studio admin, public site) and where they live in
the project. Do not restate architecture; link it.

## Current frontend baseline
Verified against the code, with paths: routes, pages, components, state handling, and the API
client layer this work extends. Name what must keep working unchanged.
Write "none — new development" when nothing exists. Do not guess.

## Target behaviour
What the UI must do, in product terms. Layout comes from the design, data from `api-contract.md`.
State the invariants that hold regardless of visual choices — the ones a plausible-looking
implementation could break silently.

## Cross-cutting constraints
Links to the governing architecture documents that apply (for example rendering mode, tenant
resolution, authentication). Do not restate them here.

## Screen mapping (before any UI code)
Presented to the owner for approval before code is written. For each block of the Figma frame:
- the kit component or token it resolves to (or "dynamic" / "local" with the reason)
- the states it must cover (default, hover, focus, disabled, empty, loading, error, selected/open —
  only those that apply)
- where the design differs from the kit, and the proposed handling

Choose the page pattern and the scroll owner before inventing layout. No production UI code until
the owner approves the mapping.

## Implementation and verification
- Figma / generated code is a reference only; it is never pasted as production UI.
- Use kit tokens and components; do not invent spacing, colours, or typography.
- A control that appears enabled in the design is not disabled merely because its handler is
  missing.
- Before calling the work done, compare Figma Inspect with the implementation (text style, colour,
  spacing, control size, radius and border, scroll owner, states) and report every mismatch.

## Phases
Ordered, each independently reviewable, each naming what it depends on.

## Required tests
What must be proven, not how. Include the invariants from Target behaviour.

## Acceptance criteria
Numbered, checkable statements, including that existing behaviour stays green.

## Open questions
Points this brief deliberately leaves to implementation. Answer by number in
`frontend-brief-feedback.md`. An unanswered item cannot be carried back into canon.

## Required feedback
`frontend-brief-feedback.md` contains:
1. Answers to every numbered open question, by number (including ones accepted as proposed).
2. Mismatches between the delivered API and `api-contract.md`.
3. Parts of the design that could not be implemented as specified, and why.
4. Behaviour decided in the UI because neither the design nor the API defined it.
5. Which items the implementer believes change the canon (opinion only; canon is not edited).
6. Which documents were updated.

## Out of scope
Each excluded item points at the brief or canon that owns it.

## Quality checklist
- [ ] Mission is understandable without reading the code
- [ ] Baseline is verified against the code, or explicitly "none"
- [ ] Source-of-truth order names the design and the API contract
- [ ] Screen mapping step is required before code
- [ ] Every acceptance criterion is checkable
- [ ] No open question hides a product decision
- [ ] No canon is restated
