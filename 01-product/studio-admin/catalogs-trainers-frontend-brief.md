# Frontend brief — Trainers, class types, hall description

> **Ownership.** This brief is written by the project owner from the canon of the topic and the agreed API contract. The frontend implementer does not edit it. Deviations, better approaches, and implementation discoveries are recorded in `catalogs-trainers-frontend-brief-feedback.md` in the same folder. Only the owner changes canon.
>
> **Design.** There is no design brief and no Figma. Behaviour and layout are `catalogs.md`. Lists and forms follow `../../02-design/flow-design.md`. Components are the ones already used for halls.
>
> **API.** Extend `catalogs-api-contract.md` in phase 1 of `catalogs-trainers-backend-brief.md`. No screen that calls the new fields or endpoints is built before the owner has agreed that extension.
>
> `catalogs-frontend-brief.md` is the finished halls work. This brief does not replace it.

**Derived from:** `catalogs.md`, `../../02-design/flow-design.md`, `../../03-architecture/files.md`
**Figma:** none

## Mission

Add a description to the hall form, and build the Trainers and Class types tabs. Each tab lists, searches, creates, and edits records with a name, an optional description, and a gallery on the edit page, the same way halls already work.

## Source-of-truth order

1. `catalogs.md`, then `../../02-design/flow-design.md` and `../../03-architecture/files.md`
2. The halls screens in `src/studio/catalogs/` as the working pattern
3. `catalogs-api-contract.md` after the owner agrees the extension
4. This brief, for sequencing

Canon wins. Where `catalogs.md` and the agreed contract disagree, report it in feedback instead of choosing.

## Surfaces and placement

The studio admin, in `studio-desk-frontend`, under `src/studio/catalogs/` and `src/app/studio/`. Do not import `src/platform/`. The top tab row stays as it is. Catalogs stays the selected top tab.

## Current frontend baseline

Verified 2026-10-09 against `studio-desk-frontend`:

- Halls are built: list, search, create, edit, gallery. The hall form fields are name, address, and video link. There is no description field.
- `src/studio/catalogs/catalog-tabs.tsx` already shows Halls, Trainers, and Class types. Trainers and Class types open an empty page (`empty-catalog-tab.tsx`).

Platform admin, staff, settings, and the existing hall behaviour other than the new description must keep working unchanged.

## Target behaviour

Screens and fixed texts are in `catalogs.md`. These are the invariants a plausible-looking implementation could break silently.

1. The hall form, on create and on edit, has «Description» between address and video link. It may be left empty. Empty does not show an error. The list columns stay Name and Address. Search stays by name.
2. Trainers replace the empty tab. List search, «Add trainer», «No trainers yet», «No trainers found», create, and edit follow `catalogs.md`. Fields are «Name», «Description», «Instagram», and «TikTok», on both create and edit. Description and both links may be empty. An empty link shows no error. A bad Instagram link shows «Enter an Instagram link». A bad TikTok link shows «Enter a TikTok link». The name error is «Enter a name». Success texts are «Trainer added» and «Trainer updated». The add breadcrumb is «New trainer».
3. Class types replace the empty tab the same way. Texts are «Add class type», «No class types yet», «No class types found», «Class type added», «Class type updated», «New class type».
4. The gallery exists only on edit of a saved record. It is the hall gallery: «Add file», image or `video`, remove confirmation, reorder through the reorder request and not through Save. The remove-file sentence is the one fixed for that record in `catalogs.md`.
5. Catalogs stays the selected top tab. The second row selects Halls, Trainers, or Class types.

## Cross-cutting constraints

`../../02-design/flow-design.md`, `../../03-architecture/files.md`, and the existing studio session. Do not restate them.

## Screen mapping (before any UI code)

There is no Figma. Before production UI code, name which existing hall components are reused for the description field, the trainer screens, and the class type screens, and which states they cover. No production UI code until the owner approves that mapping.

## Implementation and verification

- Reuse the hall list, form, and gallery. Do not invent a second visual pattern.
- Fixed texts from `catalogs.md` are inserted as written.
- Before calling the work done, compare trainers and class types with halls, on a computer and on a phone, and report every mismatch with `catalogs.md` in feedback.

## Phases

1. Screen mapping. Needs nothing. No production UI code.
2. Hall description on create and edit. Needs phase 1 and the agreed contract extension.
3. Trainers tab, including the gallery. Needs phase 1 and the contract extension.
4. Class types tab, including the gallery. Needs phase 1 and the contract extension.

Phases 2, 3, and 4 do not depend on each other.

## Required tests

The invariants above are proven, in particular: a hall saves with an empty description; trainer and class type create have no gallery; a video file renders as `video`; reorder does not submit the form; the fixed texts are the canon strings; existing hall tests stay green.

## Acceptance criteria

1. The owner approved the screen mapping before production UI code.
2. The hall form shows «Description» and allows it to be empty.
3. Trainers and Class types are no longer empty tabs. List, search, create, edit, and gallery match `catalogs.md`.
4. Existing platform admin, staff, settings, and hall tests stay green.

## Open questions

None.

## Required feedback

`catalogs-trainers-frontend-brief-feedback.md` contains:
1. Answers to every numbered open question, by number. There are none until the owner adds one.
2. Mismatches between the delivered API and the agreed contract.
3. Parts of `catalogs.md` that could not be implemented as specified, and why.
4. Behaviour decided in the UI because neither the canon nor the contract defined it.
5. Which items the implementer believes change the canon (opinion only; canon is not edited).
6. Which documents were updated.

## Out of scope

- A design brief and Figma.
- Capacity, and group or individual, on a class type.
- The staff form role «trainer».
- Deleting a hall, a trainer, or a class type.
- The public site.

## Quality checklist

- [x] Mission is understandable without reading the code
- [x] Baseline is verified against the code
- [x] Source-of-truth order names the canon and the API contract
- [x] Screen mapping step is required before code
- [x] Every acceptance criterion is checkable
- [x] No open question hides a product decision
- [x] No canon is restated
