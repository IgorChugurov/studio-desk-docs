# Frontend brief — Staff trainer link

> **Ownership.** This brief is written from the canon and the agreed staff contract. The frontend implementer does not edit it. Deviations and discoveries go in `staff-trainer-frontend-brief-feedback.md` in the same folder. Only the owner changes canon.
>
> `frontend-brief.md` is finished. This brief does not replace it. It adds the trainer role on the staff screens that brief left out.
>
> **Design.** No new design assignment and no Figma. Layout and fixed texts are `overview.md`. Components are the ones the staff screens already use.

**Derived from:** `overview.md`, `api-contract.md`, `../../02-design/flow-design.md`
**Figma:** none

## Mission

Let the owner add and edit a staff member whose role is trainer, linked to one catalog trainer, and see that trainer in the staff list. When this works, that person can sign in and open Schedule. The schedule screens themselves are not part of this brief.

## Source-of-truth order

1. `overview.md` for the staff list, the add and edit flows, and the fixed texts
2. The staff screens already in `studio-desk-frontend/src/studio/staff/`, and the select and table they use from `src/shared/ui`
3. `api-contract.md`, section staff, for `trainerId`, the role `trainer`, and the error codes
4. This brief, for sequencing

Canon wins. Where `overview.md` and `api-contract.md` disagree, report it in feedback instead of choosing.

## Surfaces and placement

The studio admin, in `studio-desk-frontend`. Code stays in `src/studio/staff/` and `src/studio/i18n/language.ts`. It may use `src/studio/catalogs/catalog-options.ts` to read trainer names. It does not import `src/platform/`. Sign-in, settings, catalogs, and the rest of the shell stay as they are.

## Current frontend baseline

Verified 2026-10-16 against `studio-desk-frontend`.

- The staff list is `src/studio/staff/staff-list.tsx`. Columns are E-mail, Role, Added. There is no Trainer column.
- The staff form is `src/studio/staff/staff-form.tsx`. The role select has `administrator` and `accountant` only. There is no trainer field. On edit, Save stays disabled until the role changes.
- `src/studio/staff/staff-types.ts` accepts only those two roles and has no `trainerId`.
- The dictionaries in `src/studio/i18n/language.ts` have Administrator and Accountant. They do not have the trainer role label, «Select a trainer», or «This trainer is already linked to another staff member».
- `loadNamedRecords('/trainers')` in `src/studio/catalogs/catalog-options.ts` already reads every trainer page.
- The staff API returns `trainerId` (`api-contract.md`). `GET /me` does not.

Administrator and accountant add, edit, and remove must keep working unchanged.

## Target behaviour

Flows and fixed texts are in `overview.md`, staff list, add, and edit. Data is in `api-contract.md`. These are the invariants a plausible-looking implementation could break silently.

1. The list columns are E-mail, Role, Trainer, Added. Trainer shows the catalog trainer's name for that `trainerId`. An administrator or an accountant has an empty Trainer cell. The name is read from `GET /trainers`, not stored on the staff member.
2. The role select adds Trainer. The Trainer field is shown only while the role is Trainer. Its options are the catalog trainers who are not already linked, plus the current card when editing. The form reads every staff page with `currentPage` and `perPage` from the contract, and every trainer page, and subtracts the linked ids. It does not use only the page visible in the table.
3. Changing the role away from Trainer hides the field and does not send `trainerId`. Changing the role to Trainer requires a card before save. Save is enabled when the card changes, also when the role does not.
4. A missing card shows «Select a trainer» at the Trainer field (`VALIDATION_ERROR` `REQUIRED` on `trainerId`). A card already linked shows «This trainer is already linked to another staff member» at that field (`TRAINER_ALREADY_LINKED`). `message` is not shown.
5. New phrases are in the English, Slovak, and Ukrainian dictionaries. No phrase is written into the component. English is the source from `overview.md`. Slovak and Ukrainian are prepared in the implementation and checked by the owner.

## Cross-cutting constraints

Lists and forms: `../../02-design/flow-design.md`. Errors and lists: `../../03-architecture/api-conventions.md`. Do not restate them.

## Screen mapping

The screens already exist. This brief adds to them. No new visual component.

| Block | Component already on the screen | States |
|---|---|---|
| Trainer column | The staff table, a new column between Role and Added | Empty cell for a non-trainer. Name when `trainerId` is set. The name is missing from the loaded trainers: the cell stays empty |
| Role select | The existing role `Select` | Administrator, Accountant, Trainer |
| Trainer field | The same `Field` and `Select` as Role, under Role | Hidden unless the role is Trainer. Options are the free cards, with the current card kept on edit. Loading while the lists load. Field error for a missing card and for a card already linked |

No production UI code until the owner accepts this brief.

## Implementation and verification

- Use the staff screen's existing components and tokens. Do not invent spacing, colours, or typography.
- Fixed texts from `overview.md` are inserted as written.
- Before calling the work done, exercise add, edit, and the list on a computer and on a phone: a trainer with a card, an administrator without one, a refused missing card, and a refused card that is already linked.

## Phases

1. This brief. Needs the agreed staff contract. No UI code.
2. Staff list column, form field, and the three dictionaries. Needs the owner to accept this brief.

## Required tests

- An administrator and an accountant still have an empty Trainer cell and no Trainer field.
- Choosing Trainer without a card shows «Select a trainer» and does not save.
- A catalog trainer already linked to another staff member shows «This trainer is already linked to another staff member».
- Saving a trainer staff member shows that trainer's name in the list.
- Changing the role away from Trainer clears the link on the next read. Changing only the card saves the new card.
- The current card stays selectable while editing. A card linked to someone else does not.
- Slovak and Ukrainian contain the new English strings. Existing staff tests stay green.

## Acceptance criteria

1. The owner can add, edit, and remove a staff member with the role Trainer and one catalog trainer.
2. The Trainer column and field match `overview.md`, including the two fixed error texts.
3. Administrator and accountant staff screens behave as they do now, plus the empty Trainer column.
4. The owner has checked the Slovak and Ukrainian texts.
5. No screen in this brief opens or edits the schedule.

## Open questions

None.

## Required feedback

`staff-trainer-frontend-brief-feedback.md` contains:

1. Deviations from this brief and why.
2. Behaviour decided in the UI because neither `overview.md` nor the contract defined it.
3. Which items the implementer believes change the canon (opinion only; canon is not edited).
4. Which documents were updated.

## Out of scope

- Schedule screens, week templates, classes, and what a signed-in trainer can do with a class. `schedule.md` and `schedule-frontend-brief.md`.
- Deleting a catalog trainer. `catalogs.md`.
- Sign-in, studio settings, and the shell. `frontend-brief.md`.

## Quality checklist

- [x] Mission is understandable without reading the code
- [x] Baseline is verified against the code
- [x] Source-of-truth order names the canon and the API contract
- [x] Screen mapping step is required before code
- [x] Every acceptance criterion is checkable
- [x] No open question hides a product decision
- [x] No canon is restated beyond the invariants an implementation could break silently
