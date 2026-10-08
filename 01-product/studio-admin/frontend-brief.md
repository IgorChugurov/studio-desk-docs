# Frontend brief — Studio admin

> **Ownership.** This brief is written by the project owner from the canon of the topic and the agreed API contract. The frontend implementer does not edit it. Deviations, better approaches, and implementation discoveries are recorded in `frontend-brief-feedback.md` in the same folder. Only the owner changes canon.
>
> **Design.** On 2026-10-08 the owner skipped the design assignment for this surface. There is no `design-brief.md` and no Figma. Behaviour and layout are `overview.md`. Components come from the starter kit, as on the platform admin.

**Derived from:** `overview.md`, `api-contract.md`, `../../03-architecture/authentication.md`, `../../03-architecture/api-conventions.md`, `../../02-design/flow-design.md`
**Figma:** none

## Mission

Build the studio admin at `app.studio-desk.axondigital.xyz`: sign-in by e-mail and code, choosing a studio when the person has several, the section tabs, studio settings, and staff. When this works, an owner and the staff of a studio use the browser, on a computer and on a phone, and see only the sections their role allows.

## Source-of-truth order

1. `overview.md` (behaviour, layout, fixed texts), then `../../03-architecture/authentication.md` and `../../03-architecture/api-conventions.md`
2. Components already ported into `studio-desk-frontend/src/shared/ui`, then the starter kit `opie-admins-pack/starter-kit/src/components/figma/` for anything still missing. The platform admin shell is `src/platform/shell`.
3. `api-contract.md` for the API this UI consumes
4. This brief, for sequencing

Canon wins over the visual source. Where `overview.md` and `api-contract.md` disagree, report it in feedback instead of choosing.

## Surfaces and placement

The studio admin, in the `studio-desk-frontend` project. The host is chosen in `src/proxy.ts` (`APP_HOST`, locally `app.localhost:3001`). API base `https://api.studio-desk.axondigital.xyz/api/studio` (`api-contract.md`). Code of this surface lives in `src/studio/` and `src/app/studio/` and may import `src/shared/`. It does not import `src/platform/`. The platform admin and the public site stay as they are.

## Current frontend baseline

Verified 2026-10-08 against `studio-desk-frontend`.

- The platform admin is built: sign-in, shell, studio list, studio page. Its host is rewritten to `src/app/platform/`. `src/proxy.ts` answers 404 for every other host, including the studio admin host.
- `src/studio/` has no pages. The README states the studio admin is not built.
- The kit used by the platform admin is in `src/shared/ui` (button, field, input, table, pagination, modal, dropdown menu, avatar, breadcrumbs, toaster). Those files were ported from `opie-admins-pack/starter-kit/src/components/figma/`. The tab row and the select are not in `src/shared/ui` yet. In the starter kit they are `FigmaTab.tsx` and `FigmaSelect.tsx`.
- The platform client calls `NEXT_PUBLIC_API_URL`, which points at `/api/platform`. Sessions use the platform refresh cookie. The studio API uses a different base path and the cookie `sd_studio_refresh` (`api-contract.md`).

The platform admin's screens, tests, and host must keep working unchanged.

## Target behaviour

Screens, flows, and fixed texts are in `overview.md`. Data is in `api-contract.md`. These are the invariants a plausible-looking implementation could break silently.

1. **Access token lives only in page memory.** Never in `localStorage`, `sessionStorage`, a cookie, or an address. The refresh token is the `HttpOnly` cookie `sd_studio_refresh`, which the page cannot read (`authentication.md`).
2. **Every page load starts with one token refresh** of the studio API. No cookie (`UNAUTHORIZED`) shows the sign-in screen with no message. `SESSION_EXPIRED` shows it with «Your session has expired. Sign in again».
3. **One refresh serves all waiting requests.** On a `401` with an expired access token, a single refresh runs, then the waiting requests repeat. If the refresh fails, the sign-in screen is shown.
4. **Browser contract of the auth calls:** requests to `/auth/*` include credentials. `POST /auth/refresh`, `POST /auth/sign-out`, `POST /auth/switch-studio`, and `POST /auth/impersonation/exchange` send `X-Requested-With`.
5. **A studio session is never rewritten into another studio.** Choosing a studio and switching studio replace the session (`api-contract.md`). The previous session is revoked on a switch.
6. **Texts come from `code` and `field`.** The screen text is chosen by the table in `api-contract.md`, «error codes and screen texts». `message` is never shown. A field error is shown at the field named in `field`.
7. **Types come from the OpenAPI document** of the studio API, not written by hand.
8. **Countdowns use the seconds from the server**, not the clock of the user's computer.
9. **The sign-in answer does not reveal** whether the e-mail has a studio. An e-mail with no active studio is shown as a wrong code.
10. **Several studios at sign-in show the selection page and create no session.** One studio signs in at once. `INVALID_SELECTION_TICKET` returns to the sign-in screen.
11. **There is no home page.** After sign-in, after a switch, after exchanging the handoff code, and on a click of the studio name, the first section in the `overview.md` order that `GET /me` lists in `sections` is open, and that tab is selected.
12. **The tab row lists only `sections` from `GET /me`.** A direct address of a section that is not in the list shows «You don't have access to this page». Schedule, Catalogs, Clients and bookings, Subscriptions, and Accounting and reports are empty pages with their tab selected.
13. **The interface language** is `interfaceLanguage` from `GET /me` (`null` means English). Changing it calls `PATCH /me/language`, the screen updates without a reload, and the page writes the cookie copy after the response. The server does not set that cookie. The sign-in and studio-selection screens read the copy and have no language control.
14. **Studio settings** send only fields that changed. «Save» is enabled only when a value has changed. There is no Back on that page.
15. **Staff in this brief** adds and edits `administrator` and `accountant` only. The trainer role, the Trainer column, and the trainer field wait for the catalogs topic (`overview.md`, `api-contract.md`). Remove deletes the row. There is no disabled state.
16. **Interface texts come from language dictionaries.** No phrase is written into the code. English in `overview.md` is the source. Slovak and Ukrainian are prepared in the implementation and checked by the owner. The platform admin stays English only.
17. **The sign-out** calls the API and always returns to the sign-in screen, also when the call fails.
18. **On a narrow screen** the tab row scrolls horizontally and does not reflow. Breadcrumbs stay in the navbar. The studio name hides when they do not fit.

## Cross-cutting constraints

Sessions and cookie protection: `../../03-architecture/authentication.md`. Errors, lists, OpenAPI: `../../03-architecture/api-conventions.md`, section «Агент — фронтенд». Fixed texts, tabs, and flows: `overview.md`. Lists and forms: `../../02-design/flow-design.md`.

## Decisions left to the frontend

Project structure and routing inside the studio surface, rendering mode, state and data fetching, the API client, form handling, the name and attributes of the language cookie the page writes, how the studio client addresses `/api/studio` without changing `NEXT_PUBLIC_API_URL` of the platform client, auto-closing time of notifications, and the dictionary mechanism. Product behaviour above must not change because of these decisions.

A control that is already in `src/shared/ui` is used as it is. A control this surface needs and that folder does not have is ported from `opie-admins-pack/starter-kit/src/components/figma/` into `src/shared/ui`, as source of this project, the same way the platform admin kit was ported. The starter-kit engines `universal-list`, `form-generation`, `entity-config`, and `sdk` are not ported. A component is not invented.

## Screen mapping (before any UI code)

Presented to the owner for approval before UI code is written. For each block (sign-in, studio selection, navbar, language items, switch-studio window, tab row, empty section, studio settings, staff list, staff form, remove window, notifications):

- the shared component it resolves to, or the starter-kit file that will be ported into `src/shared/ui`
- the states it must cover (only those that apply)

Country, currency, time zone, and the studio language use the select ported from starter-kit `FigmaSelect.tsx`. The section tab row is the port of starter-kit `FigmaTab.tsx`. No production UI code until the owner approves the mapping.

## Implementation and verification

- Do not invent spacing, colours, or typography. Use the tokens already in `src/shared/styles`.
- Use a component from `src/shared/ui`. If it is missing, port it from the starter kit into that folder before using it. Do not write a new visual component.
- A control that `overview.md` shows as enabled is not disabled merely because its handler is missing.
- Before calling the work done, compare the studio shell with the platform admin shell and any newly ported component with its starter-kit source, on desktop and on a phone, and report every mismatch in feedback.

## Phases

1. Studio host and API client: `src/proxy.ts` serves the studio surface, session handling for `sd_studio_refresh` (invariants 1–5, 17). No production UI. Platform admin stays green. Needs nothing else.
2. Sign-in, studio selection, handoff exchange (`/impersonate`). Needs phase 1 and the approved screen mapping.
3. Navbar, tab row, language menu, switch-studio window, empty sections, first section on entry. Needs phase 2.
4. Studio settings. Needs phase 3.
5. Staff list, add, edit, remove. Needs phase 3. Can follow phase 4 or precede it.

## Required tests

Invariants 1–18. Every sign-in error code shows its text. One studio signs in at once; several studios show the selection page and no session. A switch opens the first allowed section of the new studio and drops the previous session. Tabs follow `sections`. A section outside `sections` shows the forbidden text. Settings «Save» stays disabled until a value changes. Staff cannot assign `trainer`. Remove drops the row. Slovak and Ukrainian dictionaries contain every English source string. The existing platform admin tests stay green.

## Acceptance criteria

1. All flows of `overview.md` for this surface work on desktop and on a phone against the studio API.
2. Every fixed text appears as written in `overview.md`, in the person's interface language.
3. After a reload the signed-in person stays signed in. After Sign out they do not.
4. Two tabs refreshing at once keep the session.
5. A server field error appears at the right field, with the right text.
6. The shell matches the platform admin shell, and ported components match the starter kit, or the mismatch is listed in feedback.
7. No access token in `localStorage`, `sessionStorage`, a cookie the page can read, or an address.
8. The platform admin still passes its tests and still answers on its own host.
9. The owner has checked the Slovak and Ukrainian texts.

## Open questions

None.

## Required feedback

`frontend-brief-feedback.md` contains:

1. The approved screen mapping.
2. Mismatches between the delivered API and `api-contract.md`.
3. Parts of `overview.md` that could not be implemented as specified, and why.
4. Behaviour decided in the UI because neither `overview.md` nor the API defined it.
5. Which items the implementer believes change the canon (opinion only; canon is not edited).
6. Which documents were updated.

## Out of scope

- The insides of Schedule, Catalogs, Clients and bookings, Subscriptions, and Accounting and reports. Their tabs are empty pages.
- The trainer role, the Trainer column, and the link to a catalog trainer.
- The platform admin, except the host split that must keep it working.
- The public site and client sign-in.
- A design assignment and Figma for this surface.
- Deployment of the frontend.

## Quality checklist

- [ ] Mission is understandable without reading the code
- [ ] Baseline is verified against the code
- [ ] Source-of-truth order names the canon and the API contract
- [ ] Screen mapping step is required before code
- [ ] Every acceptance criterion is checkable
- [ ] No open question hides a product decision
- [ ] No canon is restated
