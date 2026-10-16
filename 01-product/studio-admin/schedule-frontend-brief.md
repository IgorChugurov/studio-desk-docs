# Frontend brief — Schedule

> **Ownership.** This brief is written by the project owner from the canon of the topic. The frontend implementer does not edit it. Deviations, better approaches, and implementation discoveries are recorded in `schedule-frontend-brief-feedback.md` in the same folder. Only the owner changes canon.
>
> **Design.** The owner did not ask for a design assignment. There is no design brief and no Figma. Behaviour, layout, and fixed texts are `schedule.md`. Lists, forms, and confirmations follow `../../02-design/flow-design.md`. Components and tokens come from the kit already used by the studio admin. What `schedule.md` leaves to a designer (colour, size, type, the look of a card, a strip, a day header, and the conflict and cancelled windows) is decided by that kit, not by a new drawing.
>
> **API.** `schedule-api-contract.md` does not exist yet. It is phase 1 of `schedule-backend-brief.md`. No screen that calls the schedule API is built before the owner has agreed that contract. A finished schedule backend is not required for the screen mapping.

**Derived from:** `schedule.md`, `catalogs.md`, `overview.md`, `../../02-design/flow-design.md`
**Figma:** none

## Mission

Build the Schedule section of the studio admin. An owner or an administrator sees the week, creates and edits future classes, keeps week templates, and applies, clears, or replaces weeks. A trainer sees only their own classes and cannot change them. When this works, the studio's week is kept on the same shell as Catalogs.

## Source-of-truth order

1. `schedule.md` (behaviour, layout, fixed texts), then `catalogs.md` for the hall, the trainer, and the class type, then `overview.md` for who sees Schedule
2. `../../02-design/flow-design.md` for lists, forms, and confirmations
3. Components already in `studio-desk-frontend/src/shared/ui`, as used by `src/studio/catalogs/` and `src/studio/staff/`
4. `schedule-api-contract.md`, once the owner has agreed it, for the schedule API. The catalogs API, including hall type, hall time zone, class-type duration, and the trainer link once `catalogs-schedule-fields-backend-brief.md` is in the catalogs contract, for the form's selects
5. This brief, for sequencing

Canon wins over the visual source. Where `schedule.md` and the agreed contract disagree, report it in feedback instead of choosing.

## Surfaces and placement

The studio admin, in `studio-desk-frontend`. Code lives in `src/studio/` and `src/app/studio/` and may import `src/shared/`. It does not import `src/platform/`. The platform admin, staff, settings, sign-in, and the existing catalog screens stay as they are, except the schedule block added under the gallery on the hall and trainer edit pages.

The schedule section is already `/schedule` (`src/studio/sections.ts`). That address opens the Schedule tab. Templates is the other tab of this section, not a new entry in the top tab row.

## Current frontend baseline

Verified 2026-10-14 against `studio-desk-frontend`:

- The studio shell, sign-in, settings, staff, and catalogs exist. The top tab row includes Schedule.
- There is no `src/app/studio/(app)/schedule/` page. `/schedule` is the empty section page `src/app/studio/(app)/[section]/page.tsx`, which renders nothing. The shell draws the tab.
- Catalogs is the pattern to follow for a section with a second tab row, a list, and add and edit pages: `src/app/studio/(app)/catalogs/` and `src/studio/catalogs/`.
- Staff is the pattern for a list and a form: `src/studio/staff/`.
- The hall edit page and the trainer edit page have no schedule under the gallery.
- There is no week grid in this project.

Sign-in, staff, settings, and the existing catalog screens must keep working unchanged.

## Target behaviour

Screens, flows, and fixed texts are in `schedule.md`. Data is in `schedule-api-contract.md` after it is agreed. These are the invariants a plausible-looking implementation could break silently.

1. The top tab row is unchanged. Schedule stays selected on the week, on Templates, and on template add and edit. A trainer sees Schedule and does not see the week-action buttons or Templates. An accountant does not see Schedule.
2. Under it, a second row: Schedule and Templates. Breadcrumbs follow `schedule.md`.
3. The week opens on the current week, scrolled to today where the days run downward. Hours 08:00–22:00 are visible. The night bands collapse and expand, and that choice is remembered for the signed-in person. A whole day collapses from its header and is expanded again on the next week. Filters are one value or All, survive a week change, and reset when the page is left. One hall or one trainer switches to the week grid. Hall and trainer together show that hall's week grid with only that trainer's classes.
4. A class is created from an empty cell or from an expanded night band, only when that start has not passed. The form shows the grid's time. The client converts it to the hall's clock. A past class, including a cancelled one, opens read-only. A future cancelled class offers only Restore and Delete.
5. Fixed texts from `schedule.md` are inserted as written, including `HH:MM`, the singular and plural class counts, and «Back» on every window except «Cancel class».
6. On the hall edit page the grid is that hall's week, with a trainer filter and the hall's time zone. On the trainer edit page it is that trainer's week, with a hall filter and the studio's time zone. Neither page has the week-action buttons or the Halls / Trainers switch.
7. Template add and edit follow the catalog form pattern: Back, Save, Update, a unique name, and the same class fields on a row. Deleting a row asks for confirmation.

## Cross-cutting constraints

Lists and forms: `../../02-design/flow-design.md`. Studio host and session: the existing studio admin. Errors and lists: `../../03-architecture/api-conventions.md`. Do not restate them.

## Screen mapping (before any UI code)

There is no Figma. Before production UI code, name for the week page, the class window, the template list and editor, and the three week-action windows:

- the shared component it uses, taken from the catalogs and staff screens
- what is built for this brief because the kit has no week grid, and which kit tokens it uses
- the states it covers (empty week, loading, field error, conflict list, cancelled strip, cancelled card, past read-only)

No production UI code until the owner approves the mapping.

## Implementation and verification

- Use kit tokens and components. Do not invent spacing, colours, or typography.
- Fixed texts from `schedule.md` are inserted as written.
- A control that the canon shows as usable is not left disabled because its handler is missing.
- Before calling the work done, compare the screens with the catalog list and form, on a computer and on a phone, and report every mismatch with `schedule.md` in feedback.

## Phases

1. Screen mapping. Needs nothing. No production UI code.
2. Week page and the class window: views, filters, create, edit, cancel, restore, delete, replace. Needs phase 1 and the agreed `schedule-api-contract.md`.
3. Templates: list, add, edit, delete. Needs phase 1 and that contract. Does not need phase 2.
4. Apply, clear, and replace, including the conflict window. Needs phases 2 and 3.
5. The same week grid on the hall edit page and the trainer edit page. Needs phase 2.

## Required tests

The invariants in Target behaviour are proven, in particular: Schedule stays the selected top tab; a trainer sees only their classes and no week actions; a past cell does not create a class; a past class has no actions; fixed texts match `schedule.md`, including one class and many classes; night-band state survives a reload; filters reset on leaving the page; the platform admin and the existing staff, settings, and catalog tests stay green.

## Acceptance criteria

1. The owner approved the screen mapping before production UI code.
2. No schedule screen calls the API before `schedule-api-contract.md` is agreed.
3. An owner or an administrator can keep the week and the templates. The fixed texts match `schedule.md`.
4. A trainer can only look at their own classes. An accountant does not open Schedule.
5. A class that has already started cannot be created or changed from the UI.
6. The hall and trainer edit pages show their week grid without the week-action buttons.
7. Existing platform admin, staff, settings, and catalog tests stay green.

## Open questions

None.

## Required feedback

`schedule-frontend-brief-feedback.md` contains:

1. Answers to every numbered open question, by number. There are none until the owner adds one.
2. Mismatches between the delivered API and `schedule-api-contract.md`.
3. Parts of `schedule.md` that could not be implemented as specified, and why.
4. Behaviour decided in the UI because neither the canon nor the contract defined it.
5. Which items the implementer believes change the canon (opinion only; canon is not edited).
6. Which documents were updated.

## Out of scope

- A design brief and Figma.
- The schedule API and the catalog fields it depends on. `schedule-backend-brief.md` and `catalogs-schedule-fields-backend-brief.md`.
- Bookings, attendance, subscriptions, trainer pay, and the public schedule page. `schedule.md`.
- What a trainer may do beyond reading. `schedule.md`, open questions.
- Dragging a class, and automatic extension of weeks. `schedule.md`.

## Quality checklist

- [x] Mission is understandable without reading the code
- [x] Baseline is verified against the code
- [x] Source-of-truth order names the canon and the API contract
- [x] Screen mapping step is required before code
- [x] Every acceptance criterion is checkable
- [x] No open question hides a product decision
- [x] No canon is restated beyond the invariants an implementation could break silently
