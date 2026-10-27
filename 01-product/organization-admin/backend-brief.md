# Backend brief — Studio admin

> **Ownership.** This brief is written by the project owner from the canon of the topic and the approved product decisions. The backend implementer does not edit it. Deviations, better approaches, and implementation discoveries are recorded in `backend-brief-feedback.md` in the same folder. Only the owner changes canon.

**Derived from:** `01-product/studio-admin/overview.md`

## Mission

Build the backend of the studio admin (`/api/studio`): the place where the owner and the staff of one studio sign in by e-mail and code, choose and switch between their studios, manage the studio settings and the staff, and work under a role that decides which sections they can use. When this works, an owner can sign in to a studio created in the platform admin, change its settings, add administrators and accountants, and every later section (schedule, catalogs, clients, subscriptions, accounting) can be protected by the same access mechanism.

## Sequencing

Depends on the finished platform admin backend (`01-product/platform-admin/backend-brief.md`, phases 1–4): studios, owners, `handoff_code`, deactivation. Some of its deferred parts are part of this brief (phase 3). No dependency on other briefs. The catalogs topic (trainers) is not started; this brief does not wait for it.

Order inside this brief: first the API contract of the studio API (`api-contract.md`, phase 1), then implementation. The frontend brief is issued after the contract is agreed and the design is accepted.

## Source-of-truth order

1. `01-product/studio-admin/overview.md` and `01-product/profile.md`
2. `03-architecture/` (`backend-structure.md`, `api-conventions.md`, `authentication.md`) and `04-engineering-rules/backend.md`
3. `01-product/platform-admin/api-contract.md`, where the platform API and the studio API meet (handoff code exchange, `OWNER_IS_STAFF`)
4. The approved design, only where it affects the API
5. This brief, for sequencing

Canon wins over this brief. Where they appear to disagree, raise it in feedback instead of choosing.

## Current backend baseline

Verified 2026-10-06 against `studio-desk-backend`:

- The studio API is a stub: `src/apis/studio/studio-api.module.ts` connects the database as `studio_api`; `studio-auth.guard.ts` rejects every request with `401`. There are no controllers.
- Sign-in, tokens, cookies, mailer, and the common error format exist for the platform API (`src/apis/platform/auth/`, `src/common/errors/`).
- Tables: `studio` (name, subdomain, custom domain, owner e-mail, status), `platform_administrator`, `sign_in_code` (one row per e-mail), `session` (**bound to `platform_administrator_id NOT NULL`**, with `api` in `platform|studio`, no `studio_id`, no subject for owners and staff, no `impersonated_by`), `handoff_code`, `foundation_check`.
- Platform endpoints `deactivate`, `activate`, `impersonate` exist; deactivation changes only the status (no session revocation); the studio API has no handoff exchange. `PATCH /studios/{id}` has no `OWNER_IS_STAFF` check yet.
- Grants per API user: `src/database/expected-grants.ts`, checked by a test.

No tables for staff, studio settings, currencies, or interface language. The existing `session` and `sign_in_code` shapes do not fit studio sign-in as they are; how to change them is the backend's decision.

## Product behaviour (mandatory)

Product terms and flows are in the canon. These are the invariants a plausible-looking implementation could break silently.

1. **Sign-in** is the same mechanism as the platform's: e-mail, code, tokens. An e-mail can sign in to the studio API if it is the owner of an active studio or a staff member of an active studio. The code request answer is identical for every e-mail. An e-mail without any active studio is indistinguishable from a wrong code.
2. **A studio session is bound to one studio** and never changes studio. One studio: sign-in creates the session at once. Several studios: sign-in creates no session and returns the person's studios and a one-time, short-lived selection ticket; choosing a studio exchanges the ticket for a session. Switching studio creates a new session and revokes the current one, without a new code. A studio the e-mail is not linked to, or a deactivated one, cannot be obtained. Mechanism: `authentication.md`.
3. **Owner and staff do not combine.** An e-mail that is the owner of a studio cannot be added as its staff member. The platform API rejects an owner change to a current staff e-mail of that studio with `OWNER_IS_STAFF` (`409`, field `owner.email`); the owner is not changed. One e-mail can be owner of one studio and staff of another.
4. **Roles and access.** Four fixed roles; one role per person per studio; one owner per studio, changed only through the platform API. The table «role → sections» lives in the code and is built so it can later become configurable per person and gain finer rights. Access to a section is only «yes» or «no». The role and the rights are read from the database on every request, not taken from the token. A request to a section without access is rejected by the backend. Staff and studio settings are owner-only. The access mechanism must be reusable by the endpoints of the later section topics (it is not tied to staff and settings).
5. **Staff.** Added by e-mail (lowercased, unique within a studio). Roles that can be assigned now: administrator and accountant. The trainer role and the link to the trainers catalog are not part of this brief; the role table must allow adding them later. Role changes act from the next request. Removing a staff member revokes all their sessions in this studio immediately; the same e-mail in other studios is not affected. There is no «disabled» state. The owner is not in the staff list.
6. **Studio settings.** Language of the studio for clients, country, currency, time zone. They are always filled: a new studio is created with the defaults of the canon, and studios that already exist receive the same defaults in the migration that introduces the settings. There is no first-sign-in step. Only the owner changes them. Country is validated by ISO 3166-1, time zone by IANA, currency by the table of allowed currencies in the database, filled by a migration with the list from the canon. Editing that list is not part of this brief.
7. **Interface language of a person** is stored by e-mail and applies in all the person's studios. Not set means English.
8. **Deactivated studio.** New sign-in of its owner and staff is rejected; the sessions of the studio are revoked at deactivation, except log-in-as-studio sessions. Activation reverses the rejection. Data is kept.
9. **Owner change** (platform API) revokes all sessions of the previous owner in that studio, including log-in-as-studio sessions.
10. **Log in as studio.** The studio API exchanges the handoff code for an owner-level session with `impersonated_by` set, for a studio in any state including Deactivated. The code lives 60 seconds and works once. Contract of the exchange: `platform-admin/api-contract.md`.
11. **No physical delete** of studios through any of this.

## Cross-cutting constraints

Stack, the three APIs, database users, and grants: `03-architecture/backend-structure.md`. Errors, validation, lists, OpenAPI: `03-architecture/api-conventions.md`. Sessions, cookies, revocation, handoff code, studio selection: `03-architecture/authentication.md`. Rules of the repository: `04-engineering-rules/backend.md`. Raise conflicts in feedback.

Data structure is agreed with the owner before any table is created or changed (`00-governance/feature-workflow.md`).

## Decisions left to the backend

Schema and naming (including the shape of sessions for studio users, staff, settings, currencies, language, selection ticket), endpoints and methods, DTOs, error codes of the studio API, indexes and constraints, transaction strategy, module structure. Also: lifetime of the selection ticket, whether sign-in codes of the two APIs share storage, the exact set of countries and time zones supported by the chosen standard data source. Product behaviour above must not change because of these decisions.

## Required research before implementation

None. E-mail delivery uses the provider already chosen for the platform (`platform-admin/backend-brief-feedback.md`).

## API contract

`api-contract.md` in this folder, written by the backend in phase 1 and agreed by the owner. This brief states the requirements it must satisfy and does not define endpoints or payloads. The contract must make it possible for the frontend to know after sign-in: the person's e-mail, role, studio name and id, the sections the person can use, the interface language, whether the session is a log-in-as-studio session, and the list of the person's studios (for the selection page and the «Switch studio» menu item).

## Phases

1. API contract of the studio API. Needs nothing.
2. Studio sessions and sign-in: codes, the shape of sessions for owners and staff, studio selection and switching, refresh, sign-out, the guard. Needs phase 1.
3. Platform-side leftovers: handoff code exchange, rejection of sign-in for a deactivated studio, revocation of studio sessions on deactivation and on owner change, `OWNER_IS_STAFF`. Needs phase 2.
4. Studio settings: defaults for new and existing studios, the currency table, reading and changing settings. Needs phase 2.
5. Roles, access by section, and staff: list, add, edit, remove, revocation on removal. Needs phase 2.
6. Interface language of a person. Needs phase 2.

Phases 4, 5, and 6 are independent of each other.

## Required tests

Each invariant above is proven by a test, in particular: identical answer for known and unknown e-mail; an e-mail without an active studio gets the answer of a wrong code; one studio signs in at once, several studios give a selection ticket and no session; selection ticket is single-use and expires; switching creates a new session and revokes the old one; a session works only for its own studio; owner e-mail cannot be added as staff; `OWNER_IS_STAFF` on owner change and the owner stays unchanged; section access matrix for each role, including a rejected request to a section without access; role and rights are read from the database (a role change acts on the next request with the same token); staff removal revokes the person's sessions in this studio only; staff and settings endpoints rejected for non-owners; new and existing studios get the default settings; country, time zone, and currency outside the allowed sets are rejected; interface language is shared across studios of one e-mail; deactivated studio rejects sign-in and loses its sessions except log-in-as-studio ones; owner change revokes the previous owner's sessions including log-in-as-studio ones; handoff exchange works for an Active and a Deactivated studio, and fails for an unknown, expired, or reused code; log-in-as-studio sessions are marked `impersonated_by`; grants test passes for all new tables.

## Acceptance criteria

1. `api-contract.md` exists and is agreed before implementation of phase 2 starts.
2. An owner created in the platform admin signs in to the studio admin by e-mail and code, and can read and change the studio settings.
3. A person with several studios chooses one and can switch to another without a new code; each session works only for its own studio.
4. An e-mail without an active studio cannot sign in and gets the same answer as for a wrong code.
5. The owner adds, edits, and removes administrators and accountants; a removed person's sessions in that studio stop working at once.
6. Every request is checked against the role and sections read from the database; a section without access is rejected by the backend.
7. A new studio has the default settings from the canon; studios that existed before have them after the migration.
8. Country, time zone, and currency are accepted only from their allowed sets; the currency set is read from the database.
9. A person's interface language is stored by e-mail and is the same in all their studios.
10. A deactivated studio rejects sign-in and has no active sessions except log-in-as-studio ones; log in as studio works for it.
11. The platform administrator can open any studio as its owner through the handoff code, and the session is marked `impersonated_by`.
12. An owner change to a current staff e-mail of the studio is rejected with `OWNER_IS_STAFF`.
13. All new tables have explicit grants and an entry in the expected-grants file; the grants test passes.
14. Every new endpoint passes the checklist of `api-conventions.md`, and the Postman collection of the repository contains the new requests.

## Open questions

Points this brief deliberately leaves to implementation. Answer by number in `backend-brief-feedback.md`.

1. Sign-in codes of the platform API and the studio API: the same e-mail can request a code in both. How the two are kept from overwriting each other.
2. Where the person's interface language is stored, if not as a separate table by e-mail, and why.
3. The lifetime of the selection ticket.

## Required feedback

`backend-brief-feedback.md` contains:
1. Answers to every numbered open question, by number (including ones accepted as proposed).
2. Deviations from this brief and why.
3. Behaviour decided in the backend because neither canon nor this brief defined it.
4. Which items the implementer believes change the canon (opinion only; canon is not edited).
5. Which documents were updated.

## Out of scope

- The trainer role and the link between staff and the trainers catalog (invariants 1–8 of «Сущности» in the canon): the catalogs topic.
- Content of the sections other than staff and studio settings: schedule, catalogs, clients and bookings, subscriptions, accounting (own topics).
- Public site, client sign-in, and host-to-studio lookup (public API).
- The e-mail to the owner after studio creation.
- Editing the list of allowed currencies in the platform admin.
- The platform admin frontend change for `OWNER_IS_STAFF`: a light-route change after the backend is done.
- All interface texts, translations, and form behaviour.

## Quality checklist

- [x] Mission is understandable without reading the code
- [x] Baseline is verified against the code
- [x] Mandatory behaviour and decisions left to the backend are clearly separated
- [x] Every acceptance criterion is checkable
- [x] No open question hides a product decision
- [x] No canon is restated
