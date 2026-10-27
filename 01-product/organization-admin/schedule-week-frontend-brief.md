# Frontend brief — Schedule week

> **Ownership.** This brief is the week slice of `schedule-frontend-brief.md`. The frontend implementer does not edit it. Deviations go in `schedule-frontend-brief-feedback.md`. Only the owner changes canon.
>
> `schedule-frontend-brief.md` stays the brief for the whole section. This file is its phase 2, cut down on 2026-10-17: the week grid and the class window on the Schedule tab only.
>
> **Design.** No Figma. Layout and fixed texts are `schedule.md`. Components and tokens are the kit already used by the studio admin.
>
> **API.** `schedule-api-contract.md` is agreed. No screen in this brief is built against a different shape.

**Derived from:** `schedule.md`, `schedule-api-contract.md`, `schedule-frontend-brief.md`, `../../02-design/flow-design.md`
**Figma:** none

## Mission

Show the studio week on the Schedule tab and let an owner or an administrator create and change future classes from that grid. A trainer sees only their own classes and cannot change them. When this works, the week can be checked against the schedule API, including overlaps, a past class, cancel, and the view time zone.

## Source-of-truth order

1. `schedule.md` for the grid, the class window, and the fixed texts
2. `schedule-api-contract.md` for classes, halls, trainers, and class types
3. `src/shared/ui` as used by `src/studio/catalogs/` and `src/studio/staff/`
4. This brief, for what this slice leaves out

Canon wins. Where `schedule.md` and `schedule-api-contract.md` disagree, report it in feedback instead of choosing.

## Surfaces and placement

The studio admin, in `studio-desk-frontend`. The page is `/schedule`, which today is the empty section page. Code lives in `src/studio/schedule/` and `src/app/studio/(app)/schedule/`. It may import `src/shared/` and `src/studio/catalogs/catalog-options.ts`. It does not import `src/platform/`.

The top tab row is unchanged. Schedule stays selected. There is no second tab row in this slice. Sign-in, staff, settings, and catalogs stay as they are.

## Current frontend baseline

Verified 2026-10-17 against `studio-desk-frontend`.

- `/schedule` renders the empty section page `src/app/studio/(app)/[section]/page.tsx`. The shell already draws the Schedule tab.
- There is no week grid.
- Halls, trainers, and class types are loaded by `loadNamedRecords` in `src/studio/catalogs/catalog-options.ts`. A hall from `GET /halls` has `timeZone` and `type`.
- The kit used here: `Button`, `Field`, `Input`, `Select`, `FilterMenu`, `Modal`, `Toaster`. Confirmations follow `src/studio/staff/remove-staff-dialog.tsx`.
- The schedule API returns a class with hall-clock `date` and `startTime`, `durationMinutes`, `hallId`, `trainerIds`, `classTypeId`, `audience`, `format`, `places`, and `cancelled`. A week is loaded with `GET /classes?from=&to=`. The server widens those dates to Monday–Sunday.

## Target behaviour

This slice is the Schedule tab only. These are the invariants a plausible-looking implementation could break silently.

1. The page opens on the current week, in the studio time zone. Arrows and a calendar move to any week, including far ahead. The view time zone can be switched. Filters are one hall, one trainer, one class type, or All. Filters and the view time zone survive a week change and reset when the page is left.
2. While hall and trainer are All, columns are halls or trainers and the week runs downward: Monday, then Tuesday, and so on. The page scrolls to today. One hall, or one trainer, switches to the week grid: columns are days, rows are hours. Hall and trainer together show that hall's week grid with only that trainer's classes. Today's column is marked. A class with several trainers appears in each trainer's column.
3. Hours 08:00–22:00 are visible. 00:00–08:00 and 22:00–24:00 are thin bands. A click expands that band for the whole week. If the week has a class in those hours, the band starts open. The browser remembers the bands for this person. A day in the downward view collapses from its date header, starts open, and is open again on the next week. Collapsed days are not remembered.
4. A class is placed by converting its hall-clock date and start into the view time zone. The stored digits are not rewritten. On a card: the class type, the start and end, the trainers when columns are halls, the hall when columns are trainers, the format when it is not ordinary offline, and «one person» when the class is not a group.
5. An empty cell, including an expanded night band, opens «New class» only when that view-time start has not passed. The form shows the view time. Save sends the hall's clock. A past cell does not open the form. A future class can change time, date, hall, duration, trainers, format, and places. The class type is read-only. A past class, including a cancelled one, is read-only. A future cancelled class offers only Restore and Delete.
6. Where a live class occupies a slot, cancelled classes at that slot are one thin strip. Where it does not, the slot is one card marked «Cancelled». A click opens the list of cancelled classes that overlap that slot. Each row shows the class type, time, hall, trainers, format, and places. A future row has Restore and Delete. A past row has none.
7. A trainer sees only their classes, with no create and no action buttons. An accountant does not open Schedule. Fixed texts from `schedule.md` are inserted as written, including `HH:MM`.

## Cross-cutting constraints

Lists and forms: `../../02-design/flow-design.md`. Errors: `../../03-architecture/api-conventions.md`. The class requests and error codes: `schedule-api-contract.md`. Do not restate them.

## Screen mapping

No new visual kit component. The week grid is built for this brief because the kit has none. It uses the existing tokens (`--space-*`, `--text-*`, `--border-*`, `--background-*`, `--radius-*`) and no new colours or type.

| Block | Taken from | States |
|---|---|---|
| Week arrows and calendar | `Button`. The calendar is a date input in a `Modal` | Current week. Another week. Far ahead |
| View time zone | `Select`, values from the studio time-zone set | Studio zone on open. Another zone. Reset on leaving the page |
| Halls / Trainers | `Button` pair, the same idea as a segmented control | Halls as columns. Trainers as columns. Hidden when the week grid is on |
| Hall, trainer, class type | `FilterMenu`, plus All | All. One value. Hall and trainer together. Reset on leaving the page |
| Week grid | New. Hours column stays put on a narrow screen; columns scroll | Empty week. Loading. Downward days. One hall or trainer. Today scrolled into view, or today's column marked. Night bands open and closed. A day collapsed. A class in two trainer columns |
| Class card and cancelled strip | New, inside the grid | Live card. One cancelled strip beside a live class. «Cancelled» card. Past card |
| Class window | `Modal`, `Field`, `Input`, `Select` | New class. Future edit, class type read-only. Past read-only. Future cancelled with Restore and Delete. Field error. Hall busy. Trainer busy. Saving, screen dimmed |
| Cancel, restore, delete | `Modal`, as `remove-staff-dialog.tsx` | Confirm cancel. Confirm delete. Restore error. Success texts «Class added», «Class updated», «Class cancelled», «Class restored», «Class deleted» |
| Cancelled list | `Modal` | Future rows with Restore and Delete. Past rows with none |

Replace is in this slice: the class window's Replace opens the same form as New class, filled from the old class except the class type. Save is `POST /classes/{id}/replace`. The old class stays if the new one is refused. There is no link between them.

## What the client asks

- `GET /classes?from=&to=` for the week on screen. `hallId` and `trainerId` are sent when those filters are not All. The class-type filter is applied on the client: the contract has no class-type query.
- `GET /halls`, `GET /trainers`, and `GET /class-types` for columns, filters, and names. A hall's `timeZone` places the class.
- Create, update, cancel, restore, delete, and replace as in `schedule-api-contract.md`.
- A conflict sentence is built from `conflict`, with times as `HH:MM`. `message` is not shown.

## Out of this slice

- The Templates tab, template rows, and the second tab row. `schedule-frontend-brief.md`, phase 3.
- Apply template, Clear weeks, Replace with template. Phase 4. The buttons are not drawn.
- The week grid on the hall edit page and the trainer edit page. Phase 5.
- Bookings, attendance, dragging a class, and automatic extension of weeks. `schedule.md`.

## Required tests

- The week opens on the current week. Changing the week keeps the filters. Leaving the page clears them.
- A past cell does not open New class. A past class has no actions. A future cancelled class offers only Restore and Delete.
- Hall and trainer together show that hall's week with only that trainer's classes.
- A class in another hall time zone moves on the grid when the view time zone changes, and the request still sends the hall's clock.
- A trainer sees only their classes and cannot save. An accountant does not open Schedule.
- Night-band state survives a reload. A collapsed day does not.
- Existing staff, settings, and catalog tests stay green.

## Acceptance criteria

1. The owner accepts this brief before the week page is built.
2. An owner or an administrator can create, edit, cancel, restore, delete, and replace a future class from the Schedule tab.
3. The grid matches the two layouts in `schedule.md`, including night bands, cancelled classes, and the view time zone.
4. A trainer can only look. An accountant cannot open the page.
5. Templates, week actions, and the hall and trainer pages are unchanged.

## Open questions

None.

## Quality checklist

- [x] Mission is understandable without reading the code
- [x] Baseline is verified against the code
- [x] Source-of-truth order names the canon and the API contract
- [x] Screen mapping is in this file, before code
- [x] Every acceptance criterion is checkable
- [x] No open question hides a product decision
- [x] No canon is restated beyond the invariants an implementation could break silently
