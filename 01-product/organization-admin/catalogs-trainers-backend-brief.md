# Backend brief — Trainers, class types, hall description

> **Ownership.** This brief is written by the project owner from the canon of the topic and the approved product decisions. The backend implementer does not edit it. Deviations, better approaches, and implementation discoveries are recorded in `catalogs-trainers-backend-brief-feedback.md` in the same folder. Only the owner changes canon.
>
> `catalogs-backend-brief.md` is the finished halls work. This brief does not replace it.

**Derived from:** `catalogs.md`, `catalogs-api-contract.md`, `../../03-architecture/files.md`

## Mission

Add an optional description to a hall, and add trainers and class types to the studio admin API. Each is a catalog record with a name, an optional description, and a gallery of files on the saved record. When this works, the studio admin can keep trainers and class types the same way it already keeps halls.

## Sequencing

Depends on the finished halls backend (`catalogs-backend-brief.md`, closed). No dependency on schedule, subscriptions, or the staff trainer role.

Order: extend `catalogs-api-contract.md` (phase 1), then implementation. New tables are shown to the owner and agreed before any migration. The frontend brief is `catalogs-trainers-frontend-brief.md`. Screens that call the new API wait until the contract change is agreed.

## Source-of-truth order

1. `catalogs.md`, then `../../03-architecture/files.md`
2. `catalogs-api-contract.md` for the halls that already exist
3. `overview.md` for who may open Catalogs
4. `../../03-architecture/` (`backend-structure.md`, `api-conventions.md`, `authentication.md`) and `../../04-engineering-rules/backend.md`
5. This brief, for sequencing

Canon wins over this brief. Where they appear to disagree, raise it in feedback instead of choosing.

## Current backend baseline

Verified 2026-10-09 against `studio-desk-backend` and `catalogs-api-contract.md`:

- Halls exist: list with search, create, read, update, files, delete file, reorder. There is no hall delete.
- A hall has `name`, `address`, `videoLink`, and `images`. It has no description.
- A hall file is public at `/files/halls/{hallId}/{fileId}` plus the type extension. Upload is `POST /api/studio/halls/{id}/files`.
- There are no trainer or class-type tables and no such endpoints.

Existing hall behaviour, other than the new description, must keep working unchanged.

## Product behaviour (mandatory)

Flows and fixed texts are in `catalogs.md`. These are the invariants a plausible-looking implementation could break silently.

1. **Hall description.** Optional text. Empty is stored as `null`, not as `""`. Name, address, and video link stay as they are. Search remains a substring of the name, not of the description. Adding description does not change file indexes.
2. **Trainer.** Name is required. Description, Instagram, and TikTok are optional. Empty is stored as `null`, not as `""`. A non-empty Instagram value is an Instagram address; a non-empty TikTok value is a TikTok address; anything else is rejected. Screen texts: «Enter an Instagram link», «Enter a TikTok link». No e-mail, working hours, pay rule, or promo code. Two trainers of one studio may share a name. There is no delete of a trainer. Saving these links does not change file indexes.
3. **Class type.** Name is required. Description is optional. No capacity, no group-or-individual flag, no duration, no price. Two class types of one studio may share a name. There is no delete of a class type.
4. **Access.** Same as halls: owner and administrator of the session's studio. A trainer and an accountant are refused. Another studio's record is `404 NOT_FOUND`.
5. **Lists.** Search is a case-insensitive substring of the name.
6. **Files.** Same rules as a hall file: only after the record is saved; index starts at 0; delete and reorder rewrite indexes from zero; saving name or description does not change indexes; `images` comes back sorted by index; image and video are both allowed; the public addresses are `/files/trainers/{id}/{fileId}` and `/files/class-types/{id}/{fileId}` plus the type extension; opening the file does not require a session.

## Cross-cutting constraints

`../../03-architecture/backend-structure.md`, `../../03-architecture/api-conventions.md`, `../../03-architecture/authentication.md`, `../../03-architecture/files.md`, `../../04-engineering-rules/backend.md`.

## Decisions left to the backend

Schema and naming, endpoints and methods, DTOs, error codes, indexes and constraints, transaction strategy, module structure. How an Instagram address and a TikTok address are recognized is recorded in the contract extension. The public file paths above are fixed. Product behaviour must not change because of these decisions.

## Required research before implementation

None.

## API contract

Extend `catalogs-api-contract.md` in phase 1. Agree the extension with the owner before phase 2. This brief does not define payloads. The extension must:

- add optional `description` to a hall on create, read, and update;
- list, create, read, and update trainers with `name`, `description`, `instagram`, `tiktok`, and `images`, and class types with `name`, `description`, and `images`;
- upload, delete, and reorder their files;
- keep the screen texts already fixed in `catalogs.md`.

## Phases

1. Contract extension. Needs nothing.
2. Hall description. Needs phase 1. No new table beyond the column on the existing hall.
3. Trainers, including files. Needs phase 1 and the owner's agreement on the table.
4. Class types, including files. Needs phase 1 and the owner's agreement on the table.

Phases 3 and 4 do not depend on each other. Phase 2 does not depend on them.

## Required tests

Halls still pass. In particular: an empty description is `null`; search does not use description; a trainer and a class type require a name; description, Instagram, and TikTok may be empty; a non-empty value that is not that network is rejected; duplicate names are allowed; a trainer role and an accountant are refused; files follow the hall index rules; public URLs match `catalogs.md`; updating the name does not change indexes; the grants test passes for every new table.

## Acceptance criteria

1. The contract extension is agreed before phase 2 starts.
2. A hall can be saved with and without a description. Existing hall tests stay green.
3. The owner and an administrator can list, search, create, and edit trainers and class types. A trainer role and an accountant cannot.
4. Neither a trainer nor a class type can be deleted by any request in this brief.
5. Their files follow `catalogs.md`, including the public paths `/files/trainers/…` and `/files/class-types/…`.
6. New tables have explicit grants and an entry in the expected-grants file.

## Open questions

None.

## Required feedback

`catalogs-trainers-backend-brief-feedback.md` contains:
1. Answers to every numbered open question, by number. There are none until the owner adds one.
2. Deviations from this brief and why.
3. Behaviour decided in the backend because neither canon nor this brief defined it.
4. Which items the implementer believes change the canon (opinion only; canon is not edited).
5. Which documents were updated.

## Out of scope

- Schedule, capacity, and whether a class is a group or individual (`catalogs.md`).
- The staff form role «trainer» and the link to a catalog trainer (`overview.md`). Not this brief.
- Pay, promo codes, subscriptions, and the public site.
- Deleting a hall, a trainer, or a class type.

## Quality checklist

- [x] Mission is understandable without reading the code
- [x] Baseline is verified against the code
- [x] Mandatory behaviour and decisions left to the backend are clearly separated
- [x] Every acceptance criterion is checkable
- [x] No open question hides a product decision
- [x] No canon is restated
