# Backend brief — Hall type and time zone, class-type duration, trainer link

> **Ownership.** This brief is written by the project owner from the canon of the topic and the approved product decisions. The backend implementer does not edit it. Deviations, better approaches, and implementation discoveries are recorded in `catalogs-schedule-fields-backend-brief-feedback.md` in the same folder. Only the owner changes canon.
>
> `catalogs-backend-brief.md` and `catalogs-trainers-backend-brief.md` are finished. This brief does not replace them. It adds the catalog fields that `schedule-backend-brief.md` depends on.

**Derived from:** `catalogs.md`, `catalogs-api-contract.md`

## Mission

Add the missing catalog fields: a hall's type and time zone, a class type's default duration, and the link between a trainer and the class types they teach. When this works, a class can be stored against a hall that has a time zone, and the schedule brief can start.

## Sequencing

Depends on the finished halls, trainers, and class types (`catalogs-backend-brief.md`, `catalogs-trainers-backend-brief.md`). No dependency on schedule. The schedule brief waits for this one.

Order: extend `catalogs-api-contract.md` (phase 1), then implementation. New tables and new columns are shown to the owner and agreed before any migration. The schedule grid on the hall and trainer edit pages is not part of this brief.

## Source-of-truth order

1. `catalogs.md`
2. `catalogs-api-contract.md` for the halls, trainers, and class types that already exist
3. `overview.md` for who may open Catalogs
4. `api-contract.md` for the studio time-zone set
5. `../../03-architecture/` (`backend-structure.md`, `api-conventions.md`, `authentication.md`) and `../../04-engineering-rules/backend.md`
6. This brief, for sequencing

Canon wins over this brief. Where they appear to disagree, raise it in feedback instead of choosing.

## Current backend baseline

Verified 2026-10-14 against `studio-desk-backend` and `catalogs-api-contract.md`:

- Halls, trainers, and class types exist, including files, search, create, read, and update. None of them can be deleted.
- A hall has `name`, `address`, `description`, `videoLink`, and `images`. It has no type and no time zone (`src/database/schema/hall.ts`).
- A class type has `name`, `description`, and `images`. It has no default duration (`src/database/schema/catalog.ts`).
- A trainer has `name`, `description`, `instagram`, `tiktok`, and `images`. There is no trainer–class type table.
- The studio has `timeZone`. The allowed values are the IANA names from `Intl.supportedValuesOf('timeZone')` (`src/apis/studio/settings/studio-settings.ts`). An unknown value is `VALIDATION_ERROR` (`INVALID_VALUE`).

Existing hall, trainer, and class-type behaviour, other than the fields below, must keep working unchanged.

## Product behaviour (mandatory)

Flows and fixed texts are in `catalogs.md`. These are the invariants a plausible-looking implementation could break silently.

1. **Access.** Same as halls: the owner and an administrator of the session's studio. A trainer and an accountant are refused. A record of another studio is not readable or writable.
2. **Hall type.** Ordinary or virtual. Chosen on create and changeable later. Omitted on create means ordinary. A virtual hall's address is exactly `Virtual hall`, on create and after a change to virtual. While the hall stays virtual, an update that changes the address is refused. A change to ordinary requires a new address in the same request. An ordinary hall keeps today's address rules. Existing halls become ordinary.
3. **Hall time zone.** An IANA name from the same set as studio settings. Omitted on create means the studio's time zone. A value outside that set is refused the same way as a studio settings time zone. The zone can be changed later. This brief stores the new zone and does not rewrite anything else: there are no classes yet. The refusal when a change would double-book a trainer is `schedule-backend-brief.md`, phase 5, and is not built here. Existing halls receive their studio's time zone.
4. **Class-type duration.** Required. Whole minutes, from 5 to 240, in steps of 5. A missing value, 0, 4, 7, and 241 are refused. The fixed text is «Enter a duration from 5 to 240 minutes in steps of 5». The form prefills 60; the backend does not invent 60 for a missing value. Existing class types are set to 60. There is still no capacity, no group-or-individual flag, and no price.
5. **Trainer–class type link.** One link, many to many. It is written from the trainer and from the class type, and both writes change the same rows. The list sent for a record replaces that record's links. An empty list is allowed. A trainer or a class type of another studio is refused. A link can always be removed. Saving it does not change file indexes. The link does not constrain classes. There is no hall–class type link.
6. **Unchanged.** Names may still be duplicated. Nothing in this brief deletes a hall, a trainer, or a class type. Search stays a case-insensitive substring of the name. File indexes are unchanged by saving these fields.

## Cross-cutting constraints

Stack, the three APIs, database users, and grants: `../../03-architecture/backend-structure.md`. Errors, validation, lists, OpenAPI: `../../03-architecture/api-conventions.md`. Sessions: `../../03-architecture/authentication.md`. Rules of the repository: `../../04-engineering-rules/backend.md`.

## Decisions left to the backend

Schema and naming, how the link is updated from either side, DTOs, error codes, indexes and constraints, and the migration order. The type values, the duration bounds, the address `Virtual hall`, and the time-zone set are fixed above. Product behaviour must not change because of these decisions.

## Required research before implementation

None.

## API contract

Extend `catalogs-api-contract.md` in phase 1. Agree the extension with the owner before phase 2. This brief does not define payloads. The extension must:

- add `type` and `timeZone` to a hall on create, read, and update;
- add `defaultDuration` to a class type on create, read, and update;
- add the class types of a trainer and the trainers of a class type on create, read, and update;
- keep every screen text already fixed in `catalogs.md`, and add «Enter a duration from 5 to 240 minutes in steps of 5».

## Phases

1. Contract extension. Needs nothing.
2. Hall type and time zone, including the backfill. Needs phase 1 and the owner's agreement on the columns.
3. Class-type duration, including the backfill to 60, and the trainer–class type link. Needs phase 1 and the owner's agreement on the column and the link table.

Phases 2 and 3 do not depend on each other.

## Required tests

Existing hall, trainer, and class-type tests stay green. In particular:

- a hall created without a type is ordinary; a hall created without a time zone receives the studio's time zone; an unknown time zone is refused;
- after create, the type can be changed: virtual stores address `Virtual hall`; ordinary without a new address is refused; a virtual hall's later address change is refused; an ordinary hall still requires an address;
- existing halls are ordinary and have their studio's time zone;
- a class type refuses a missing duration and refuses 0, 4, 7, and 241; 5, 60, and 240 are accepted; existing class types are 60;
- setting class types on a trainer and setting trainers on a class type writes the same link; an empty list clears it; another studio's id is refused; saving the link does not change file indexes;
- a trainer role and an accountant are refused;
- the grants test passes for every new table.

## Acceptance criteria

1. The contract extension is agreed before phase 2 starts.
2. A hall has an editable type and an editable IANA time zone. Existing halls are ordinary, in the studio's time zone.
3. A virtual hall's address is `Virtual hall` and cannot be edited. An ordinary hall is unchanged apart from the new fields.
4. A class type has a required default duration of 5 to 240 minutes, step 5. Existing class types are 60.
5. The trainer–class type link is one relation, editable from both records, and may be empty.
6. No request in this brief deletes a hall, a trainer, or a class type, or checks a class overlap.

## Open questions

None.

## Required feedback

`catalogs-schedule-fields-backend-brief-feedback.md` contains:

1. Answers to every numbered open question, by number. There are none until the owner adds one.
2. Deviations from this brief and why.
3. Behaviour decided in the backend because neither canon nor this brief defined it.
4. Which items the implementer believes change the canon (opinion only; canon is not edited).
5. Which documents were updated.

## Out of scope

- Classes, templates, week actions, and the refusal when a hall time-zone change would double-book a trainer. `schedule-backend-brief.md`.
- The schedule grid on the hall and trainer edit pages. `schedule.md`.
- Capacity, group or one person, and price. `catalogs.md`.
- Deleting a hall, a trainer, or a class type. `catalogs.md`.
- The staff role «trainer» and its link to a catalog trainer. `overview.md`.

## Quality checklist

- [x] Mission is understandable without reading the code
- [x] Baseline is verified against the code
- [x] Mandatory behaviour and decisions left to the backend are clearly separated
- [x] Every acceptance criterion is checkable
- [x] No open question hides a product decision
- [x] No canon is restated beyond the invariants an implementation could break silently
