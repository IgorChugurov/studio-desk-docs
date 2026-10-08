# Backend brief — Halls

> **Ownership.** This brief is written by the project owner from the canon of the topic and the approved product decisions. The backend implementer does not edit it. Deviations, better approaches, and implementation discoveries are recorded in `catalogs-backend-brief-feedback.md` in the same folder. Only the owner changes canon.
>
> This file is not `backend-brief.md`. That file is the studio admin shell (sign-in, settings, staff) and stays as it is.

**Derived from:** `catalogs.md`, `../../03-architecture/files.md`

## Mission

Build halls inside the studio admin API (`/api/studio`). An owner or an administrator can list, create, and edit halls of their studio, search them by name, and on an existing hall add, remove, and reorder image and video files. When this works, the studio admin can show a hall with its gallery in index order.

## Sequencing

Depends on the finished studio admin backend (`backend-brief.md`): studio sessions and section access. The section `catalogs` is already granted to the owner and the administrator. No dependency on trainers, class types, schedule, or the public site.

Order: the API contract of halls (phase 1), then implementation. The structure of new tables is shown to the owner and agreed before any migration (`00-governance/feature-workflow.md`). The frontend brief is `catalogs-frontend-brief.md`; calls to this API wait until the contract is agreed.

## Source-of-truth order

1. `catalogs.md`, then `../../03-architecture/files.md`
2. `overview.md` for who may open Catalogs
3. `../../03-architecture/` (`backend-structure.md`, `api-conventions.md`, `authentication.md`) and `../../04-engineering-rules/backend.md`
4. This brief, for sequencing

Canon wins over this brief. Where they appear to disagree, raise it in feedback instead of choosing.

## Current backend baseline

Verified 2026-10-08 against `studio-desk-backend`:

- Studio sessions, the studio guard, and section access exist. `catalogs` is one of the section ids in `src/database/schema/role-section.ts` and `src/apis/studio/access/studio-access.service.ts`.
- There is no hall table, no file table, and no file storage. No hall or file endpoint exists.

Sign-in, staff, and studio settings must keep working unchanged.

## Product behaviour (mandatory)

Flows and fixed texts are in `catalogs.md`. File storage and the read address are in `files.md`. These are the invariants a plausible-looking implementation could break silently.

1. **Access.** Only the owner and an administrator of the studio. Every hall and file request is refused for anyone else, including a trainer and an accountant. The studio is the one in the session. A hall of another studio is not readable or writable.
2. **Hall.** Name and address are required. The video link may be empty. A non-empty video link is a YouTube or Vimeo address; anything else is rejected. Two halls of one studio may have the same name. There is no delete of a hall.
3. **List.** Search is a case-insensitive substring of the name. The owner is not a special row; every hall of the studio is listed.
4. **Files only after the hall exists.** Create does not accept files. Upload, delete, and reorder apply to a saved hall.
5. **Index.** A new file's index equals the number of files on that hall at that moment. Several files in one upload receive consecutive indexes in upload order. Delete rewrites the remaining indexes from zero. Reorder is its own request and rewrites indexes from zero. Saving the hall's name, address, or video link does not change indexes.
6. **Response.** A hall includes `images`: the file records of that hall, sorted by index. An image and a video are both allowed. The read address of a file is `/api/files/{id}` on the API host (`files.md`).
7. **Who can read a file in this topic.** The owner or an administrator of the studio can read a file of a hall of that studio. The address is not made public. Visitors of the public site are outside this brief.

## Cross-cutting constraints

Stack, the three APIs, database users, and grants: `../../03-architecture/backend-structure.md`. Errors, validation, lists, OpenAPI: `../../03-architecture/api-conventions.md`. Sessions: `../../03-architecture/authentication.md`. Where the file bytes live: `../../03-architecture/files.md`. Rules of the repository: `../../04-engineering-rules/backend.md`.

## Decisions left to the backend

Schema and naming, endpoints and methods except the read address fixed in `files.md`, DTOs, error codes, indexes and constraints, transaction strategy, module structure, how YouTube and Vimeo addresses are recognized. Product behaviour above must not change because of these decisions.

## Required research before implementation

None.

## API contract

`catalogs-api-contract.md` in this folder, written by the backend in phase 1 and agreed by the owner before phase 2. This brief does not define endpoints or payloads. The contract must let the frontend list and search halls, create a hall, read and update its name, address, and video link, receive `images` sorted by index, upload files to an existing hall, delete a file, and reorder files. Screen texts on success are «Hall added» and «Hall updated». Field texts are «Enter a name», «Enter an address», «Enter a YouTube or Vimeo link».

## Phases

1. API contract of halls and their files. Needs nothing.
2. Halls: list with search, create, read, update. No delete. Needs phase 1 and the owner's agreement on the table structure.
3. Files of a hall: storage as in `files.md`, upload, delete with reindexing, reorder, `images` on the hall. Needs phase 2.

## Required tests

Each invariant above is proven by a test, in particular: a trainer and an accountant are refused; a hall of another studio is refused; empty name and empty address are refused; an empty video link is kept; a video link that is not YouTube or Vimeo is refused; two halls may share a name; search is a case-insensitive substring of the name; create does not store files; the first file has index 0; a second file has index 1; three files uploaded together receive 0, 1, 2 in order; deleting the middle file leaves 0 and 1; reorder persists the new order; updating the name does not change indexes; `images` comes back sorted by index; the grants test passes for every new table.

## Acceptance criteria

1. `catalogs-api-contract.md` exists and is agreed before phase 2 starts.
2. The owner and an administrator can list, search, create, and edit halls of their studio. A trainer and an accountant cannot.
3. Name and address are required. The video link is optional and, when set, is a YouTube or Vimeo address.
4. A hall cannot be deleted by any request in this brief.
5. Files are uploaded, deleted, and reordered only on a saved hall. Indexes follow `catalogs.md`. Saving the hall fields does not change indexes.
6. The hall response includes `images` sorted by index. A file is readable at `/api/files/{id}` by the owner or an administrator of that studio, and is not public.
7. An image file and a video file can both be stored.
8. All new tables have explicit grants and an entry in the expected-grants file; the grants test passes.
9. Every new endpoint passes the checklist of `api-conventions.md`.

## Open questions

1. Which image and video types are accepted, and what the maximum size of one file is. Both an image and a video must remain possible.

## Required feedback

`catalogs-backend-brief-feedback.md` contains:
1. Answers to every numbered open question, by number (including ones accepted as proposed).
2. Deviations from this brief and why.
3. Behaviour decided in the backend because neither canon nor this brief defined it.
4. Which items the implementer believes change the canon (opinion only; canon is not edited).
5. Which documents were updated.

## Out of scope

- Trainers, class types, and deleting a hall (`catalogs.md`, open questions).
- The staff search change in `overview.md`. It is not this brief.
- Schedule, clients, subscriptions, accounting, the public site, and client sign-in.
- Making `/api/files/{id}` readable without a studio session. `files.md` still leaves that open for the public site.

## Quality checklist

- [x] Mission is understandable without reading the code
- [x] Baseline is verified against the code
- [x] Mandatory behaviour and decisions left to the backend are clearly separated
- [x] Every acceptance criterion is checkable
- [x] No open question hides a product decision
- [x] No canon is restated
