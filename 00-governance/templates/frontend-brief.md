# Frontend brief — <topic>

> **Template.** The sections below are recommended, not mandatory. Omit sections that do not apply
> to the topic. The filled brief lives at `01-product/<topic>/frontend-brief.md`.
>
> **Ownership.** This brief is written by the project owner from the canon of the topic, the
> approved design, and the agreed API contract. The frontend implementer does not edit it.
> Deviations, better approaches, and implementation discoveries are recorded in
> `frontend-brief-feedback.md` in the same folder. Only the owner changes canon.
>
> **When issued.** Issued after `api-contract.md` is agreed and any new invariants from backend implementation (`*-backend-brief-feedback.md`) or design are synchronized back into the topic canon (`01-product/<topic>/`). When Figma is used for the slice, the design frames must be approved and their `Figma node ID`s (plus any `design error` notes where a frame diverges from canon) bound to the step-by-step flows. When Figma is not created for the slice, the layout, states, and fixed copy come directly from the topic canon's step-by-step flows (`Потоки по шагам`) and `02-design/flow-design.md`.

**Derived from:** <links to the topic canon, design brief (if any), and api-contract.md>
**Figma:** <links to files and nodes, or "none — driven by step-by-step flows in topic canon">

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

## Required tests (`Test plan`, 1:1 to Acceptance criteria)
For each acceptance criterion (`AC-1..AC-N`), state one check: **how it is verified** (`pnpm test`, API request inspection, or manual step along the flow) and **what counts as a fail** (fail condition). Include the invariants from Target behaviour and bind screen/flow checks to the flow steps (and `Figma node ID` when present).

## Acceptance criteria (`Specify`)
Numbered, observable outcomes (`AC-1..AC-N`) by which each slice/phase can be accepted, including that existing behaviour and CI checks (`pnpm lint`, `pnpm format:check`, `pnpm test`) stay green.

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
