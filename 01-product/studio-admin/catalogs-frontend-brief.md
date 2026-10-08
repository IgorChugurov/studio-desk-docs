# Frontend brief — Halls

> **Ownership.** This brief is written by the project owner from the canon of the topic and the agreed API contract. The frontend implementer does not edit it. Deviations, better approaches, and implementation discoveries are recorded in `catalogs-frontend-brief-feedback.md` in the same folder. Only the owner changes canon.
>
> **Design.** The owner did not ask for a design assignment for halls. There is no design brief and no Figma. Behaviour and layout are `catalogs.md`. Lists and forms follow `../../02-design/flow-design.md`. Components come from the kit already used by the studio admin.
>
> **API.** `catalogs-api-contract.md` does not exist yet. It is phase 1 of `catalogs-backend-brief.md`. No screen that calls the halls API is built before the owner has agreed that contract.

**Derived from:** `catalogs.md`, `../../02-design/flow-design.md`, `../../03-architecture/files.md`
**Figma:** none

## Mission

Build halls in the studio admin. An owner or an administrator opens Catalogs, sees Halls, Trainers, and Class types, and can list, search, create, and edit halls, including a gallery of images and short videos on the edit page. Trainers and Class types stay empty.

## Source-of-truth order

1. `catalogs.md` (behaviour, layout, fixed texts), then `../../02-design/flow-design.md` and `../../03-architecture/files.md`
2. Components already in `studio-desk-frontend/src/shared/ui`. A control this surface needs and that folder does not have is ported from `opie-admins-pack/starter-kit/src/components/figma/` into `src/shared/ui`. The starter-kit upload engines (`universal-list`, `form-generation`, `entity-config`, `sdk`) are not ported.
3. `catalogs-api-contract.md`, once the owner has agreed it, for the API this UI consumes
4. This brief, for sequencing

Canon wins over the visual source. Where `catalogs.md` and the agreed contract disagree, report it in feedback instead of choosing.

## Surfaces and placement

The studio admin, in `studio-desk-frontend`. Code lives in `src/studio/` and `src/app/studio/` and may import `src/shared/`. It does not import `src/platform/`. The platform admin, staff, settings, and sign-in stay as they are.

The catalogs section is already `/catalogs` (`src/studio/sections.ts`). That address opens Halls. Trainers and Class types are the other two tabs of this section, not new entries in the top tab row.

## Current frontend baseline

Verified 2026-10-08 against `studio-desk-frontend`:

- The studio shell, sign-in, settings, and staff exist. The top tab row includes Catalogs.
- `src/app/studio/(app)/[section]/page.tsx` renders nothing. `/catalogs` is that empty page.
- Staff is the list pattern to follow: `src/app/studio/(app)/staff/page.tsx`, `staff/new/page.tsx`, `staff/[id]/page.tsx`, and `src/studio/staff/`.
- There is no gallery and no file upload in this project.

## Target behaviour

Screens, flows, and fixed texts are in `catalogs.md`. Data is in `catalogs-api-contract.md` after it is agreed. These are the invariants a plausible-looking implementation could break silently.

1. The top tab row is unchanged. Catalogs stays selected on the list, on both empty tabs, and on add and edit.
2. Under it, a second row: Halls, Trainers, Class types. Halls is selected after opening Catalogs. The other two are empty pages with their tab selected.
3. The halls list has search on the left and «Add hall» on the right. Search is by name. No halls: «No halls yet». A search with no hits: «No halls found», the hint, and «Reset search». There is no Back on the list.
4. Create has Name, Address, and Video link, Back, and Save. No gallery. Empty name and empty address show their texts. An empty video link is saved. A bad video link shows «Enter a YouTube or Vimeo link». Success returns to the list with «Hall added».
5. Edit has the same fields, Back, and Update. Success returns to the list with «Hall updated». The gallery is below the fields and shows `images` in index order. An image is an image. A video file is a `video` element whose source is `/api/files/{id}`. The add control says «Add file». Dragging reorders and then calls the reorder request. It does not save the name, address, or video link.
6. Breadcrumbs: studio name, Catalogs, Halls. On add and edit the hall name is last. On add, until the hall has a name, the last crumb follows the staff add page's pattern for a page that is not saved yet.

## Cross-cutting constraints

Lists and forms: `../../02-design/flow-design.md`. File address: `../../03-architecture/files.md`. Studio host and session: the existing studio admin. Do not restate them.

## Screen mapping (before any UI code)

There is no Figma. Before production UI code, name for each block of the halls list, the hall form, and the gallery:

- the shared component it uses, or the starter-kit file that will be ported into `src/shared/ui`
- the states it covers (empty list, empty search, field error, gallery with an image, gallery with a video)

The gallery is not in `src/shared/ui`. It is built for this brief. It is not a port of the starter-kit SDK upload. No production UI code until the owner approves the mapping.

## Implementation and verification

- Use kit tokens and components. Do not invent spacing, colours, or typography.
- Fixed texts from `catalogs.md` are inserted as written.
- A control that the canon shows as usable is not left disabled because its handler is missing.
- Before calling the work done, compare the halls screens with the staff list and form, on a computer and on a phone, and report every mismatch with `catalogs.md` in feedback.

## Phases

1. Screen mapping. Needs nothing. No production UI code.
2. Halls list, the second tab row, the two empty tabs, create, and edit of the three fields. Needs phase 1 and the agreed `catalogs-api-contract.md`.
3. Gallery: add, delete, reorder, image and video display. Needs phase 2 and the file requests in that contract.

## Required tests

The invariants in Target behaviour are proven, in particular: Catalogs stays the selected top tab; Trainers and Class types render empty; create has no gallery; a video file renders as `video`; reorder does not submit the hall form; the fixed texts are the canon strings; the platform admin and the existing staff and settings tests stay green.

## Acceptance criteria

1. The owner approved the screen mapping before production UI code.
2. An owner or an administrator can list, search, create, and edit a hall. The fixed texts match `catalogs.md`.
3. Trainers and Class types are empty tabs. The top tab row is unchanged.
4. The gallery exists only on edit, shows files in index order, uses «Add file», shows a video file with a `video` element, and reorders through the reorder request.
5. Existing platform admin, staff, and settings tests stay green.

## Open questions

None. The file types and the maximum size are decided in the API contract (`catalogs-backend-brief.md`, open question 1). The UI accepts what that contract allows.

## Required feedback

`catalogs-frontend-brief-feedback.md` contains:
1. Answers to every numbered open question, by number. There are none until the owner adds one.
2. Mismatches between the delivered API and `catalogs-api-contract.md`.
3. Parts of `catalogs.md` that could not be implemented as specified, and why.
4. Behaviour decided in the UI because neither the canon nor the contract defined it.
5. Which items the implementer believes change the canon (opinion only; canon is not edited).
6. Which documents were updated.

## Out of scope

- A design brief and Figma.
- Trainers, class types beyond the empty tab, and deleting a hall.
- The staff search described in `overview.md`.
- The public site and `Image` optimization. `files.md` is unchanged: the file is still read from `/api/files/{id}`.
- Porting the starter-kit form and upload engines.

## Quality checklist

- [x] Mission is understandable without reading the code
- [x] Baseline is verified against the code
- [x] Source-of-truth order names the canon and the API contract
- [x] Screen mapping step is required before code
- [x] Every acceptance criterion is checkable
- [x] No open question hides a product decision
- [x] No canon is restated
