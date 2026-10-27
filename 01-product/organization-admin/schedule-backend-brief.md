# Backend brief — Schedule

> **Ownership.** This brief is written by the project owner from the canon of the topic and the approved product decisions. The backend implementer does not edit it. Deviations, better approaches, and implementation discoveries are recorded in `schedule-backend-brief-feedback.md` in the same folder. Only the owner changes canon.
>
> This file is not `backend-brief.md`. That file is the studio admin shell and stays as it is.

**Derived from:** `schedule.md`, `catalogs.md`, `overview.md`

## Mission

Build the studio admin schedule API (`/api/studio`): week templates, classes, the week grid, and the three week actions (apply a template, clear weeks, replace weeks with a template). When this works, an owner or an administrator can keep the studio's classes, and a trainer can read their own classes and nothing else.

## Sequencing

Depends on the finished studio admin shell (`backend-brief.md`) and on catalogs (`catalogs-backend-brief.md`, `catalogs-trainers-backend-brief.md`). The section `schedule` is already granted to the owner, the administrator, and the trainer.

This brief also depends on `catalogs-schedule-fields-backend-brief.md`: a hall's type and time zone, a class type's default duration, and the trainer–class type link. This brief does not add them. No phase that stores a class or a template row starts until that brief is done, because a class is meaningless without the hall's time zone.

No dependency on the design. The frontend brief waits until `schedule-api-contract.md` is agreed. A finished implementation is not required for that brief to start.

Order: the API contract (phase 1) may be written now, and it names the catalog dependency. Phases 2–5 wait for the catalog fields and for the owner's agreement on the new tables (`00-governance/feature-workflow.md`).

## Source-of-truth order

1. `schedule.md`, then `catalogs.md` for the hall, the trainer, the class type, and the trainer–class type link
2. `overview.md` for who may open Schedule
3. `../../03-architecture/` (`backend-structure.md`, `api-conventions.md`, `authentication.md`) and `../../04-engineering-rules/backend.md`
4. This brief, for sequencing

Canon wins over this brief. Where they appear to disagree, raise it in feedback instead of choosing.

## Current backend baseline

Verified 2026-10-14 against `studio-desk-backend`:

- Studio sessions and section access exist. `schedule` is granted to `owner`, `administrator`, and `trainer` (`src/database/schema/role-section.ts`, migration `0007_mighty_ezekiel.sql`). `accountant` is not granted `schedule`.
- A hall exists (`src/database/schema/hall.ts`): `name`, `address`, `description`, `videoLink`, files. It has no type and no time zone. There is no hall delete.
- A trainer exists (`src/database/schema/catalog.ts`): `name`, `description`, `instagram`, `tiktok`, files. There is no trainer delete.
- A class type exists: `name`, `description`, files. It has no default duration. There is no class type delete.
- There is no trainer–class type table.
- The studio has `timeZone` (`src/database/schema/studio.ts`), default `Europe/Bratislava`.
- There is no template table, no class table, and no schedule endpoint.

Sign-in, staff, studio settings, halls, trainers, and class types must keep working unchanged, except the hall update rule in phase 5.

## Product behaviour (mandatory)

Flows, fixed texts, and the screen rules are in `schedule.md`. These are the invariants a plausible-looking implementation could break silently.

1. **Access.** The owner and an administrator of the session's studio can do everything in this brief. A trainer can read classes on which they are one of the trainers, and cannot create, update, cancel, restore, delete, replace, or run a week action, and cannot read or write templates. An accountant is refused. A record of another studio is not readable or writable. A trainer does not receive another trainer's classes.
2. **What is stored.** A class stores the hall, the date and start time by that hall's clock, the duration, the trainers, the class type, group or one person, the format, the number of places, and a cancelled flag. Future and past are not stored. A class is past when its real start is at or before the current time. The real start is computed from the hall's time zone on that date. The hall-clock date and time plus the hall's time zone are the source of truth.
3. **Time.** The start time and the duration are multiples of 5 minutes. The duration is from 5 to 240 minutes. The class ends by 24:00 on the hall's clock. A class does not cross midnight. The same bounds apply to a template row and to a class type's default duration.
4. **Occupancy.** At one moment a hall has at most one class that is not cancelled, and each trainer has at most one such class in any hall. Back-to-back is allowed: one may end at 19:00 and the next start at 19:00. A cancelled class occupies nothing. The check uses the real moment, not the clock digits. The database enforces it, so two saves that race cannot both succeed.
5. **Past is frozen.** A class that has already started cannot be updated, cancelled, restored, deleted, or replaced, whether or not it is cancelled. Week actions do not create, change, or delete a class that has already started. A manual create is allowed on any day that has not started, including far ahead. A create whose start is already past is refused.
6. **Edit and replace.** An update of a future class may change the time, the date, the hall, the duration, the trainers, the format, and the places. It cannot change the class type. Changing the class type is replace: the old class is cancelled and a new class is created in one action. If the new class fails the check, the old class stays as it was. The new class stores no link to the old one. Replace is refused on a past class and on an already cancelled class.
7. **Cancel, restore, delete.** Cancel sets the flag and frees the hall and the trainers. Restore clears the flag and is refused when the class would overlap a hall or a trainer. Delete removes the class. On a future cancelled class the only changes are restore and delete. There are no bookings in this brief, so delete has nothing else to consider.
8. **Format and places.** The format is one of: offline, offline and online, online only. In a virtual hall the format is online only, and places are absent; anything else is refused. Online only in an ordinary hall still occupies the hall. Places are a whole number from 1, and only for a group that is not online only. One person has no places, in any format. The trainer–class type link does not constrain the trainers on a class or a template row: any trainer of the studio is accepted. At least one trainer is required.
9. **Template.** A template has a name, unique in the studio after trimming, and rows. A row has a weekday, a start time by the hall's clock, a duration, a hall, trainers, a class type, group or one person, a format, and places. The same occupancy rules as a class apply inside one weekday. Row times are compared as clock digits in one zone. Editing or deleting a template does not change classes already created. Editing or deleting a class does not change the template. Deleting a row is allowed. There is no delete restriction from classes. The halls of a template are the halls of its rows, not a separate stored list.
10. **Week actions.** Apply, clear, and replace affect only classes that have not started, and only weeks from the current week on. Apply skips a class that is already the same on that date. "The same" is every stored field except the cancelled flag: a cancelled class that matches is skipped and stays cancelled. A conflict is a template class that overlaps one or more existing classes by hall or by trainer. The caller sends one choice per template class. "Apply new" deletes every existing class that overlaps that template class, then creates it. "Keep existing" creates nothing for it. The whole action is one transaction: all of it is applied, or none. Clear deletes the not-yet-started classes in the chosen weeks and halls. Replace is clear and apply in one action; a trainer overlap with a class in a hall that was not cleared is the same conflict choice as apply. An empty template adds 0 classes and still succeeds. A range with nothing to delete succeeds and reports 0.
11. **Hall time zone.** Changing a hall's time zone does not rewrite the clock time of its classes. The grid follows the new zone because the real moment is computed from it. If the change would put one trainer in two classes at one moment, the hall is not saved, and the error names the conflicting class. Classes of that hall move together, so a hall cannot overlap itself this way.

The view time zone is not stored and is not a request field. The API accepts and returns the hall-clock date and time, and the hall's time zone is available on the hall, so the client can place a class in any view zone.

## Cross-cutting constraints

Stack, the three APIs, database users, and grants: `../../03-architecture/backend-structure.md`. Errors, validation, lists, OpenAPI: `../../03-architecture/api-conventions.md`. Sessions: `../../03-architecture/authentication.md`. Rules of the repository: `../../04-engineering-rules/backend.md`.

## Decisions left to the backend

Schema and naming, endpoints and methods, DTOs, error codes, indexes, the exact database constraint that enforces invariant 4, transaction boundaries inside a single action, and module structure. Whether a computed instant is stored beside the hall-clock fields. How two template rows in different hall time zones are compared as clock digits in one zone. Product behaviour above must not change because of these decisions. Lists, the error format, and sessions are fixed in `03-architecture/` and are not listed here.

## Required research before implementation

Before the migration of phase 3, record in the feedback how the database rejects an overlap of two non-cancelled classes on one hall and on one trainer, allows back-to-back, ignores cancelled classes, and still rejects the loser of two concurrent saves. The hall-clock fields stay the source of truth.

## API contract

`schedule-api-contract.md` in this folder, written in phase 1 and agreed by the owner before phase 2. This brief does not define endpoints or payloads. The contract must let the client:

- list, create, read, update, and delete templates and their rows;
- list classes for a week, for one hall, and for one trainer;
- create, update, cancel, restore, delete, and replace a class;
- apply a template, clear weeks, and replace weeks, including the per-class conflict choice and the counts in the result;
- receive, on a hall time-zone change that conflicts, the class that caused the refusal.

Field errors, conflict errors, and result texts are the fixed texts of `schedule.md`, copied verbatim, including the singular and plural forms (`{n} class added` / `{n} classes added`, and the same pair for delete and for replace). Times in those texts are `HH:MM`, 24-hour.

## Phases

1. **API contract.** Needs nothing. Names the catalog dependency. No migration.
2. **Templates.** List, create, read, update, delete, rows, the in-template overlap check, unique name. Needs phase 1, the catalog fields, and the owner's agreement on the tables.
3. **Classes.** Create, read, update, cancel, restore, delete, replace, the week read for a hall and for a trainer, the database overlap check, the past freeze. Needs phase 1 and the same agreement. Does not need phase 2.
4. **Week actions.** Apply, clear, replace, each in one transaction. Needs phases 2 and 3.
5. **Hall time zone.** On the existing hall update, reject a time-zone change that causes a trainer overlap, without rewriting clock times. Needs phase 3 and the hall time-zone field.

## Required tests

Each invariant above is proven, in particular:

- an accountant is refused; a trainer can read only their own class and cannot write; another studio's class is refused;
- a class stores hall-clock time; future and past are not columns; a class that has started refuses update, cancel, restore, delete, and replace;
- duration 0, 4, 241, and 7 minutes are refused; 5 and 240 are accepted; a class that would end after midnight is refused; back-to-back is accepted;
- two classes in one hall at one moment are refused by the database; the same for one trainer across two halls; a cancelled class does not cause either refusal; two concurrent overlapping saves do not both succeed;
- replace that fails leaves the old class unchanged and stores no link; an update cannot change the class type;
- a virtual hall refuses any format other than online only, and refuses places; a group class without places and places of 0 are refused; one person with places is refused; a trainer with no link to the class type is still accepted;
- a second template with the same name is refused; editing a template does not change an existing class; deleting a class does not change the template;
- apply skips an identical class, including a cancelled one; "apply new" deletes every overlapping class and creates the template class; "keep existing" creates nothing; a failure on one class rolls back the whole action; a started class in the chosen weeks is untouched; an empty template reports 0;
- changing a hall's time zone keeps `18:00` as `18:00` and is refused when a trainer would overlap, with that class named in the error;
- the grants test passes for every new table.

## Acceptance criteria

1. `schedule-api-contract.md` exists and is agreed before phase 2 starts.
2. The catalog dependency is recorded, and phases 2–5 have not started without those fields.
3. The owner and an administrator can keep templates and classes of their studio. A trainer can only read their own classes. An accountant cannot.
4. Overlaps are rejected by the database. Back-to-back passes. A cancelled class occupies nothing.
5. A started class cannot be changed by any action in this brief.
6. Replace is one action. A failed replace leaves the old class. The new class has no link to the old one.
7. Apply, clear, and replace are each one transaction, skip started classes, and report the fixed texts including the singular form.
8. A hall time-zone change does not rewrite clock times and is refused when it would double-book a trainer.
9. No request in this brief deletes a hall, a trainer, or a class type, creates a booking, or extends weeks automatically.

## Open questions

None. The points in "Decisions left to the backend" are answered in the feedback as decisions, not as product questions.

## Required feedback

`schedule-backend-brief-feedback.md` contains:

1. The overlap mechanism from "Required research", and the other decisions from "Decisions left to the backend".
2. Deviations from this brief and why.
3. Behaviour decided in the backend because neither canon nor this brief defined it.
4. Which items the implementer believes change the canon (opinion only; canon is not edited).
5. Which documents were updated.

## Out of scope

- Hall type, hall time zone, class-type default duration, and the trainer–class type link as fields. Owned by `catalogs-schedule-fields-backend-brief.md`. The check on time-zone change is in scope (phase 5); adding the field is not.
- Bookings, attendance, subscription deduction, trainer pay, notifications to booked clients. `schedule.md`, "Не входит", and `../../plan.md`.
- What a trainer may do to their own class beyond reading. `schedule.md`, open questions.
- The public schedule page. `schedule.md`. The read model must not block it: a class carries the hall-clock time and the hall's time zone.
- Dragging a class on the grid, and automatic extension of weeks. `schedule.md`.
- Deleting a hall, a trainer, or a class type. `catalogs.md`.

## Quality checklist

- [x] Mission is understandable without reading the code
- [x] Baseline is verified against the code
- [x] Mandatory behaviour and decisions left to the backend are clearly separated
- [x] Every acceptance criterion is checkable
- [x] No open question hides a product decision
- [x] No canon is restated beyond the invariants an implementation could break silently
