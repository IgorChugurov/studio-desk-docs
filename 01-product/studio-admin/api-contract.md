# API contract — Studio admin

**Scope:** the HTTP API of the studio admin (`/api/studio`): sign-in, studio selection and switching, the person's data and interface language, studio settings, staff. The handoff code exchange is defined in `../platform-admin/api-contract.md` and only referenced here.
**Derived from:** `overview.md`, `backend-brief.md`, `../../03-architecture/api-conventions.md`, `../../03-architecture/authentication.md`, `../platform-admin/api-contract.md`

## Essay — what this is

This is the agreed contract between the studio admin backend and its frontend. The frontend brief is issued against it, and implementation of phases 2–6 of `backend-brief.md` follows it. It names endpoints, payloads, and error codes. It does not restate product behaviour (`overview.md`), the common error and list formats (`api-conventions.md`), or how sessions work (`authentication.md`); it links to them.

After implementation, the live API is the OpenAPI document of the studio API (`/api/studio/docs/json`). Differences from this contract are recorded in `backend-brief-feedback.md`.

## Essay — general

- **Base URL:** `https://api.studio-desk.axondigital.xyz/api/studio`. Paths below are relative to it.
- **Authorization:** every endpoint requires `Authorization: Bearer <accessToken>` of a studio session, except those marked **public**. A missing, invalid, or revoked token returns `401 UNAUTHORIZED`. A platform session is not accepted. A studio session works only for its own studio.
- **Section access:** an endpoint of a section the person has no access to returns `403 FORBIDDEN` (screen text: «You don't have access to this page»). The role and the rights are read from the database on every request. Staff and studio settings are owner-only.
- **Cookie:** the refresh token is the cookie `sd_studio_refresh` with `HttpOnly; Secure; SameSite=Strict; Path=/api/studio/auth`. The frontend calls `/auth/*` with credentials included (`credentials: "include"`), because `app.` and `api.` are different origins.
- **`X-Requested-With`:** required on `POST /auth/refresh`, `POST /auth/sign-out`, `POST /auth/switch-studio`, and `POST /auth/impersonation/exchange`. Without it: `403 FORBIDDEN`.
- **Formats:** dates are ISO 8601 strings in UTC. Lifetimes (`…ExpiresIn`, `…AvailableIn`) are seconds from now. IDs are UUIDs. Errors use the common format; field errors use `VALIDATION_ERROR` (`api-conventions.md`). Text fields are trimmed. E-mails are lowercased before the format check. Empty values are handled as in `api-conventions.md`, «пустые значения».
- **Several errors:** format errors of all fields come together in one `VALIDATION_ERROR`. Checks that need the database run only when the format of all fields is valid, and return one error at a time in the order of the endpoint's table.

### Objects

`StudioRef`: `{ "id": "6f1c…", "name": "Yoga Space" }`.

`SessionBody`, the body of every successful sign-in, studio selection, switch, refresh, and exchange that creates or renews a session:

```json
{
  "accessToken": "eyJ…",
  "accessTokenExpiresIn": 900,
  "user": { "email": "anna@example.com" },
  "studio": { "id": "6f1c…", "name": "Yoga Space" }
}
```

It is the same shape as the exchange response in `../platform-admin/api-contract.md`. Everything else the frontend needs about the person is read with `GET /me`.

`Role`: `owner`, `administrator`, `accountant`, `trainer`.

`Section`: `schedule`, `catalogs`, `clients`, `subscriptions`, `accounting`, `studio-settings`, `staff`.

`Language`: `en`, `sk`, `uk`.

## Essay — sign-in

### `POST /auth/code` — request a sign-in code (public)

Body:

```json
{ "email": "anna@example.com" }
```

`200`, the same for any e-mail, whether or not it has access:

```json
{
  "codeExpiresIn": 600,
  "resendAvailableIn": 60
}
```

Both values are seconds from now. The code is always 6 digits. The same endpoint is used to resend; a new code replaces the previous one and gives a full set of attempts, also after `TOO_MANY_ATTEMPTS`. A code is e-mailed only if the e-mail is the owner of an active studio or a staff member of an active studio. How this endpoint and the platform one keep their codes from overwriting each other is the backend's decision.

| Status | `code` | `field` | When |
|---|---|---|---|
| 400 | `VALIDATION_ERROR` (`INVALID_FORMAT`) | `email` | e-mail format is invalid |
| 429 | `RESEND_TOO_EARLY` | — | called again before the resend wait is over; applies to any e-mail |

### `POST /auth/sign-in` — sign in with the code (public)

Body:

```json
{ "email": "anna@example.com", "code": "123456" }
```

`200` has one of two results, told apart by `result`.

One active studio: the session is created at once. `Set-Cookie: sd_studio_refresh=…`.

```json
{
  "result": "signed-in",
  "accessToken": "eyJ…",
  "accessTokenExpiresIn": 900,
  "user": { "email": "anna@example.com" },
  "studio": { "id": "6f1c…", "name": "Yoga Space" }
}
```

Several active studios: no session and no cookie. The frontend shows the studio selection page.

```json
{
  "result": "studio-selection",
  "selectionTicket": "q8Z…",
  "selectionTicketExpiresIn": 300,
  "studios": [{ "id": "6f1c…", "name": "Yoga Space" }]
}
```

The ticket is single-use and lives 300 seconds.

Errors, in check order:

| Status | `code` | When |
|---|---|---|
| 400 | `VALIDATION_ERROR` | e-mail or code format is invalid |
| 429 | `TOO_MANY_ATTEMPTS` | attempt limit reached for this code |
| 400 | `CODE_EXPIRED` | the code has expired |
| 400 | `INVALID_CODE` | the code is wrong, there is no code for this e-mail, or the e-mail has no active studio |

For an e-mail without any active studio, the answers must be indistinguishable from those for an e-mail with access and a wrong code (including `CODE_EXPIRED` and `TOO_MANY_ATTEMPTS` timing). How this is achieved is the backend's decision.

### `POST /auth/select-studio` — choose a studio (public, by ticket)

Body:

```json
{ "selectionTicket": "q8Z…", "studioId": "6f1c…" }
```

`200` `SessionBody`, and `Set-Cookie: sd_studio_refresh=…`. The ticket is spent.

| Status | `code` | When |
|---|---|---|
| 400 | `VALIDATION_ERROR` | body format is invalid |
| 400 | `INVALID_SELECTION_TICKET` | the ticket is unknown, expired, or already used |
| 404 | `NOT_FOUND` | the studio is not one of the person's active studios |

For `INVALID_SELECTION_TICKET` the frontend returns to the sign-in screen.

### `POST /auth/switch-studio` — switch studio

Requires Bearer and `X-Requested-With`. Body:

```json
{ "studioId": "6f1c…" }
```

`200` `SessionBody` of the new studio, and a new `Set-Cookie`. A new session is created and the current one is revoked, with no new code. If the current session is a log-in-as-studio session (`impersonated_by`), the mark passes to the new session.

| Status | `code` | When |
|---|---|---|
| 400 | `VALIDATION_ERROR` | body format is invalid |
| 404 | `NOT_FOUND` | the studio is not one of the person's active studios |

### `POST /auth/refresh` — new tokens (public, cookie)

No body. Requires the cookie and `X-Requested-With`.

`200` `SessionBody` and a new `Set-Cookie`. Previous refresh token handling, including the 10-second window: `authentication.md`.

| Status | `code` | When |
|---|---|---|
| 401 | `UNAUTHORIZED` | no cookie: the user has not signed in; the frontend shows the sign-in screen without a message |
| 401 | `SESSION_EXPIRED` | session expired, revoked, or the refresh token was reused; the cookie is cleared |

### `POST /auth/sign-out` — sign out (public, cookie)

No body. Requires `X-Requested-With`. Revokes the session of the cookie and clears the cookie. `204` always, also without a cookie or for an already revoked session.

### `POST /auth/impersonation/exchange` — exchange the handoff code (public)

Defined in `../platform-admin/api-contract.md`. It returns `SessionBody` and sets `sd_studio_refresh`.

## Essay — the person

### `GET /me` — who am I

Any studio session. `200`:

```json
{
  "user": { "email": "anna@example.com", "interfaceLanguage": "sk" },
  "studio": { "id": "6f1c…", "name": "Yoga Space" },
  "role": "owner",
  "sections": ["schedule", "catalogs", "clients", "subscriptions", "accounting", "studio-settings", "staff"],
  "impersonated": false,
  "studios": [{ "id": "6f1c…", "name": "Yoga Space" }]
}
```

- `interfaceLanguage`: `en`, `sk`, `uk`, or `null` when the person has not chosen one (the interface is then English).
- `sections`: only the sections this person can use, as they are in the database now. A role change shows here from the next request.
- `impersonated`: `true` for a log-in-as-studio session.
- `studios`: all active studios of the person, ordered by name. The «Switch studio» item is shown if there is more than one.

### `PATCH /me/language` — choose the interface language

Body:

```json
{ "language": "uk" }
```

`200`: `{ "interfaceLanguage": "uk" }`. The choice is stored by e-mail and applies in all the person's studios.

| Status | `code` | `field` | When |
|---|---|---|---|
| 400 | `VALIDATION_ERROR` (`REQUIRED`) | `language` | not sent or empty |
| 400 | `VALIDATION_ERROR` (`INVALID_VALUE`) | `language` | not `en`, `sk`, or `uk` |

## Essay — studio settings

Section `studio-settings`, owner only (`403 FORBIDDEN` for others).

### Objects

`StudioSettings`:

```json
{
  "language": "en",
  "country": "SK",
  "currency": "EUR",
  "timeZone": "Europe/Bratislava"
}
```

- `language`: `en`, `sk`, or `uk`; the language of the studio for clients.
- `country`: ISO 3166-1 alpha-2 code, upper case.
- `currency`: code from the table of allowed currencies in the database (now `EUR`, `UAH`, `USD`).
- `timeZone`: IANA time zone name.

All four fields are always filled. A new studio has the defaults of `overview.md`; existing studios receive them in the migration.

### `GET /settings`

`200`: `StudioSettings`.

### `GET /settings/options`

The allowed values for the selects. `200`:

```json
{
  "countries": ["SK", "UA", "…"],
  "currencies": ["EUR", "UAH", "USD"],
  "timeZones": ["Europe/Bratislava", "…"]
}
```

Names of countries and currencies are shown by the frontend. The exact set of countries and time zones is decided by the backend from a standard data source (`backend-brief.md`).

### `PATCH /settings`

Body: any subset of `StudioSettings`. A field that is not sent is not changed. These fields cannot be cleared: `null` or an empty value is `REQUIRED`. `200`: `StudioSettings`. Screen text on success: «Studio settings saved».

| Status | `code` | `field` | When |
|---|---|---|---|
| 400 | `VALIDATION_ERROR` (`REQUIRED`) | any of the four | sent as `null` or empty |
| 400 | `VALIDATION_ERROR` (`INVALID_VALUE`) | `language`, `country`, `currency`, `timeZone` | value is outside its allowed set |

## Essay — staff

Section `staff`, owner only (`403 FORBIDDEN` for others). The owner is not in the list.

A staff member with the role `trainer` is linked to exactly one trainer card of this studio. The link is what lets that person open Schedule and see only the classes of that card. `GET /me` does not return the link: Schedule reads it from the session on each request. There is no separate list of free trainers. The form builds it from `GET /trainers` minus the `trainerId` values already on `GET /staff`, and keeps the current card when editing.

The owner agreed this link on 2026-10-16.

### Objects

`StaffMember`:

```json
{
  "id": "2b9e…",
  "email": "olga@example.com",
  "role": "administrator",
  "trainerId": null,
  "createdAt": "2026-10-06T10:00:00Z"
}
```

`role` is `administrator`, `accountant`, or `trainer`. `trainerId` is the catalog trainer id when `role` is `trainer`, and `null` otherwise. The trainer's name is not copied onto the staff member; the client reads it from `GET /trainers`.

### `GET /staff` — list

Query, in addition to the common list parameters (`api-conventions.md`):

- `search`: case-insensitive substring of the e-mail.
- `sortBy`: `createdAt` (default) or `email`.

`200`: `{ "items": StaffMember[], "meta": … }` in the common list format. Each item includes `trainerId`.

### `POST /staff` — add

Body:

```json
{ "email": "olga@example.com", "role": "trainer", "trainerId": "4a1b…" }
```

`role` `administrator` or `accountant` omits `trainerId` or sends `null`. `role` `trainer` requires `trainerId`: the id of a trainer card of this studio that is not already linked. A `trainerId` sent with `administrator` or `accountant` is refused and nothing is stored. The staff row and the link are written in one transaction.

`201`: `StaffMember`. Screen text on success: «Staff member added». The person signs in by e-mail and code.

Errors. All `VALIDATION_ERROR` field errors come together. The checks that need the database then run one at a time, in this order:

| Status | `code` | `field` | Fixed text in the canon |
|---|---|---|---|
| 400 | `VALIDATION_ERROR` (`REQUIRED`) | `email`, `role` | — |
| 400 | `VALIDATION_ERROR` (`REQUIRED`) | `trainerId` | Select a trainer |
| 400 | `VALIDATION_ERROR` (`INVALID_FORMAT`) | `email` | Enter a valid e-mail address |
| 400 | `VALIDATION_ERROR` (`INVALID_FORMAT`) | `trainerId` | — (not a UUID) |
| 400 | `VALIDATION_ERROR` (`INVALID_VALUE`) | `role` | — (not `administrator`, `accountant`, or `trainer`) |
| 400 | `VALIDATION_ERROR` (`INVALID_VALUE`) | `trainerId` | sent with `administrator` or `accountant`, or not a trainer of this studio |
| 409 | `STAFF_ALREADY_ADDED` | `email` | This e-mail is already added to the studio |
| 409 | `EMAIL_IS_OWNER` | `email` | This e-mail belongs to the studio owner |
| 409 | `TRAINER_ALREADY_LINKED` | `trainerId` | This trainer is already linked to another staff member |

An e-mail can be a staff member of several studios, and the owner of one studio and a staff member of another. The link belongs to this studio: the same catalog trainer cannot be linked twice here, and a trainer of another studio is refused.

### `GET /staff/{id}` — one staff member

`200`: `StaffMember`. `404 NOT_FOUND` if there is no such staff member in this studio.

### `PATCH /staff/{id}` — change the role and the link

Body: any subset of `role` and `trainerId`. A field that is not sent is not changed. The e-mail does not change. The new role acts from the next request.

- Becoming `trainer` requires `trainerId` in this request.
- Staying `trainer`: an omitted `trainerId` keeps the current card; a sent `trainerId` replaces it; `null` is refused (`REQUIRED`).
- Becoming `administrator` or `accountant` removes the link. A `trainerId` sent on that request is refused and nothing changes.

The staff row and the link are written in one transaction. `200`: `StaffMember`. Screen text on success: «Staff member updated». Errors: the field and `409` errors of create for the fields that apply, and `404 NOT_FOUND`. Linking the card this member already has is not `TRAINER_ALREADY_LINKED`.

### `DELETE /staff/{id}` — remove

`204`. The link row is deleted. The trainer card stays. All sessions of this person in this studio are revoked at once; sessions of the same e-mail in other studios are not affected. The person can be added again. Screen text on success: «Staff member removed». `404 NOT_FOUND` if there is no such staff member in this studio.

### Tables

Agreed by the owner on 2026-10-16. This is the structure the link is stored in. It is not the HTTP contract.

`studio_staff.role` accepts `administrator`, `accountant`, and `trainer`.

`staff_trainer`

| Column | |
|---|---|
| `studio_id` | uuid, not null, references `studio.id` |
| `staff_id` | uuid, primary key, references `studio_staff.id`. One row per staff member. Deleting the staff member deletes this row |
| `trainer_id` | uuid, not null, references `trainer.id`, unique. One catalog trainer has at most one staff member. Deleting that trainer is refused while the row exists |
| `created_at` | timestamptz, not null |

Index on `studio_id`. No copy of the trainer's name. The link is inserted and deleted with the staff member; replacing the card updates `trainer_id`.

Grants: `studio_api` has `SELECT, INSERT, UPDATE, DELETE` on `staff_trainer`. `platform_api` and `public_api` have none. The grants file and the grants test cover the table.

## Essay — error codes and screen texts

The frontend picks the text by `code` (and `field`). Fixed texts come from `overview.md`. The texts of sign-in codes are the same as in `../platform-admin/api-contract.md`.

| `code` | Text |
|---|---|
| `VALIDATION_ERROR` on `email` | Enter a valid e-mail address |
| `RESEND_TOO_EARLY` | Please wait before requesting a new code |
| `TOO_MANY_ATTEMPTS` | Too many attempts. Try again later |
| `CODE_EXPIRED` | The code has expired. Request a new one |
| `INVALID_CODE` | Invalid or expired code |
| `SESSION_EXPIRED` | Your session has expired. Sign in again |
| `UNAUTHORIZED` from `POST /auth/refresh` | no text, sign-in screen |
| `INVALID_SELECTION_TICKET` | no text, sign-in screen |
| `FORBIDDEN` on a section | You don't have access to this page |
| `STAFF_ALREADY_ADDED` | This e-mail is already added to the studio |
| `EMAIL_IS_OWNER` | This e-mail belongs to the studio owner |
| `VALIDATION_ERROR` (`REQUIRED`) on `trainerId` | Select a trainer |
| `TRAINER_ALREADY_LINKED` | This trainer is already linked to another staff member |
| any other error of an action or of loading | Something went wrong. Try again |

## Essay — how it connects

- Product behaviour and texts: `overview.md`.
- Requirements and phases: `backend-brief.md`.
- Errors, lists, OpenAPI: `../../03-architecture/api-conventions.md`.
- Sessions, cookie, handoff code, studio selection and switching: `../../03-architecture/authentication.md`.
- The handoff code exchange and `OWNER_IS_STAFF`: `../platform-admin/api-contract.md`.

## Agent — rules

- Every endpoint, code, and field here appears in the OpenAPI document of the studio API.
- Renaming a `code` or a field is a contract change: agree it first.
- The response to `POST /auth/code` and the errors of `POST /auth/sign-in` never reveal whether an e-mail has access to any studio.
- A session of the studio API never changes its studio: selection and switching create a new session.
- Role and rights are read from the database on every request, not taken from the token.
- No endpoint deletes a studio or changes the owner.
- The `trainer` role is assigned only together with one catalog trainer of this studio. `administrator` and `accountant` have no link.

## Agent — acceptance checklist

- [ ] Each endpoint matches its path, method, body, response, and status codes
- [ ] Format errors of all fields come in one response; `409` checks follow in table order
- [ ] E-mails are lowercased on save, search, and sign-in
- [ ] Cookie name and attributes, `X-Requested-With`, and CORS match `authentication.md`
- [ ] Identical answers for an e-mail with and without an active studio
- [ ] The selection ticket is single-use and expires
- [ ] A switch revokes the previous session and keeps `impersonated_by`
- [ ] `GET /me` shows the role, sections, and studios read from the database
- [ ] Studio settings and staff are rejected for every role except the owner
- [ ] Removing a staff member revokes their sessions in this studio only
- [ ] A `trainer` staff member stores exactly one catalog trainer of this studio; `trainerId` is `null` for the other roles
- [ ] A missing trainer on that role is «Select a trainer». A trainer already linked is «This trainer is already linked to another staff member»
- [ ] Changing the role away from `trainer` removes the link. Removing the staff member removes the link and keeps the trainer card
- [ ] `GET /me` does not return the trainer link

## Open questions

None.
