# Frontend brief — Platform admin

> **Ownership.** This brief is written by the project owner from the canon of the topic, the approved design, and the agreed API contract. The frontend implementer does not edit it. Deviations, better approaches, and implementation discoveries are recorded in `frontend-brief-feedback.md` in the same folder. Only the owner changes canon.

**Derived from:** `overview.md`, `design-brief.md`, `api-contract.md`, `../../03-architecture/authentication.md`, `../../03-architecture/api-conventions.md`
**Figma:** file https://www.figma.com/design/xP9hSVpiaenxKUBUCiuEsN/Platform-Admin (nodes 19-37627, 19-34330, 19-33164, 19-31862); design kit https://www.figma.com/design/eEErngw5E7Tj0W3woDiXiv/StudioDesk-Design-System

## Mission

Build the frontend of the platform admin at `admin.studio-desk.axondigital.xyz`: sign-in by e-mail and code, the studio list, creating and editing a studio, deactivating and activating, and opening a studio's admin as its owner. When this works, the platform administrator runs studios from the browser, on desktop and on a phone.

## Source-of-truth order

1. `overview.md` (behaviour and fixed texts), then `../../03-architecture/authentication.md` and `../../03-architecture/api-conventions.md`
2. The approved design: `design-brief.md` and the Figma frames above
3. `api-contract.md` for the API this UI consumes
4. This brief, for sequencing

Canon wins over the design; the design wins over personal preference. Where the design and the agreed API disagree, report it in feedback instead of choosing.

## Surfaces and placement

The platform admin, in the `studio-desk-frontend` project (Next, `plan.md`). Host `admin.studio-desk.axondigital.xyz`; API `https://api.studio-desk.axondigital.xyz/api/platform` (`../../03-architecture/deployment.md`). The studio admin and the public site are other surfaces and are not built here.

## Current frontend baseline

None — new development. Verified 2026-10-04: the `studio-desk-frontend` repository contains no files besides git metadata.

## Target behaviour

Screens, flows, and fixed texts are in `overview.md`; layout in the design; data in `api-contract.md`. These are the invariants a plausible-looking implementation could break silently.

1. **Access token lives only in page memory.** Never in `localStorage`, `sessionStorage`, a cookie, or an address. The refresh token is an `HttpOnly` cookie the page cannot read (`authentication.md`, «Запреты»).
2. **Every page load starts with one token refresh.** No cookie (`UNAUTHORIZED`) shows the sign-in screen with no message. `SESSION_EXPIRED` shows it with «Your session has expired. Sign in again».
3. **One refresh serves all waiting requests.** On a `401` with an expired access token, a single refresh runs, then the waiting requests repeat. If the refresh fails, the sign-in screen is shown.
4. **Browser contract of the auth calls:** requests to `/auth/*` include credentials; `POST /auth/refresh` and `POST /auth/sign-out` send `X-Requested-With`.
5. **Texts come from `code` and `field`.** The screen text is chosen by the table in `api-contract.md`, «error codes and screen texts». `message` is never shown. A field error is shown at the field named in `field`.
6. **Types come from the OpenAPI document** of the platform API, not written by hand.
7. **Countdowns use the seconds from the server** (`codeExpiresIn`, `resendAvailableIn`), not the clock of the user's computer.
8. **The API is the authority on validation.** The form checks the formats from `overview.md` for early feedback, but shows the server's error when the server disagrees.
9. **The warning window appears only when the subdomain, the custom domain, or the owner e-mail differs from the loaded values.** Its lines are only for what changed. The update request sends only changed fields; clearing the custom domain removes it.
10. **Log in as studio:** an empty tab is opened at once on click and sent to the studio admin with the handoff code after `#` once the API answers (`authentication.md`, «Вход под студией»). The code never appears in an address before `#`.
11. **List state follows the API:** default filter `active`, newest first, server-side pagination (`currentPage`, `perPage`), search sent to the server.
12. **The sign-out** calls the API and always returns to the sign-in screen, also when the call fails.

## Cross-cutting constraints

Sessions and cookie protection: `../../03-architecture/authentication.md`. Errors, lists, OpenAPI: `../../03-architecture/api-conventions.md`, section «Агент — фронтенд». Fixed texts and flows: `overview.md`.

## Decisions left to the frontend

Project structure and routing, rendering mode of the pages, state and data fetching, the API client layer, form handling, the rules for proposing a subdomain from the studio name (editable until saved; the result must satisfy the subdomain format), auto-closing time of notifications, and the path of the studio admin page that receives the handoff code (kept in one setting, because the studio admin does not exist yet). How to develop against endpoints that are not yet deployed: against the OpenAPI document and the contract. Product behaviour above must not change because of these decisions.

## Required research before implementation

How the Figma design kit is implemented in code: UI approach, tokens, and the list of components needed for this surface. Record the recommendation in `frontend-brief-feedback.md` and get the owner's approval before the screen mapping. `opie-admins-pack/template-administaration-fronend` may be used as a reference, not as canon.

## Screen mapping (before any UI code)

Presented to the owner for approval before code is written. For each block of the Figma frames (sign-in, navbar, notifications, studio list, studio page, confirmation and warning windows):
- the kit component or token it resolves to (or "dynamic" / "local" with the reason)
- the states it must cover (default, hover, focus, disabled, empty, loading, error, selected/open — only those that apply)
- where the design differs from the kit, and the proposed handling

Choose the page pattern and the scroll owner before inventing layout. No production UI code until the owner approves the mapping.

## Implementation and verification

- Figma and generated code are references only; they are never pasted as production UI.
- Use kit tokens and components; do not invent spacing, colours, or typography.
- A control that appears enabled in the design is not disabled merely because its handler is missing.
- Before calling the work done, compare Figma Inspect with the implementation (text style, colour, spacing, control size, radius and border, scroll owner, states), on desktop and mobile, and report every mismatch.

## Phases

1. Project foundation: API client, types from OpenAPI, session handling (invariants 1–4, 12). No production UI. Needs the research above.
2. Sign-in screen, navbar, notifications, Sign out. Needs phase 1 and the approved screen mapping.
3. Studio list. Needs phase 2.
4. Studio page: create, edit, and the warning window. Needs phase 3.
5. Deactivate, activate, log in as studio. Needs phase 4.

## Required tests

What must be proven: invariants 1–12; for sign-in, every error code of `api-contract.md` shows its text; the states and countdowns of the code screen; the list filter, search, and pagination; each condition of the warning window; no token in any browser storage.

## Acceptance criteria

1. All flows of `overview.md` work on desktop and mobile against the real platform API.
2. Every fixed text appears as written in `overview.md`.
3. After a page reload the signed-in administrator stays signed in; after Sign out they do not.
4. Two tabs refreshing at once keep the session.
5. A server field error appears at the right field, with the right text.
6. The implementation matches the design on the comparison from «Implementation and verification», or the mismatch is listed in feedback.
7. No token in `localStorage`, `sessionStorage`, a cookie readable by the page, or an address.
8. Log in as studio opens the studio admin address with the handoff code after `#` in a new tab and leaves the platform admin open.

## Open questions

None.

## Required feedback

`frontend-brief-feedback.md` contains:
1. The research result and the owner-approved decision.
2. Mismatches between the delivered API and `api-contract.md`.
3. Parts of the design that could not be implemented as specified, and why.
4. Behaviour decided in the UI because neither the design nor the API defined it.
5. Which items the implementer believes change the canon (opinion only; canon is not edited).
6. Which documents were updated.

## Out of scope

- The studio admin, including the page that exchanges the handoff code (`api-contract.md` names the endpoint; the page belongs to the studio admin).
- The public site and client sign-in.
- Deployment of the frontend: hosting and the DNS record of `admin.studio-desk.axondigital.xyz`.
- The platform administrator action log.
- Changes to the design: they go through `design-brief.md`.

## Quality checklist

- [ ] Mission is understandable without reading the code
- [ ] Baseline is verified against the code
- [ ] Source-of-truth order names the design and the API contract
- [ ] Screen mapping step is required before code
- [ ] Every acceptance criterion is checkable
- [ ] No canon is restated
