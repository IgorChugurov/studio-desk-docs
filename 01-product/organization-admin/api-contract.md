# API contract — Organization admin

**Scope:** the HTTP API of the organization admin (`/api/organization`): sign-in, organization selection and switching, the person's data, interface language and organization configuration summary, organization settings, staff. The handoff code exchange is defined in `../platform-admin/api-contract.md` and only referenced here.
**Derived from:** `overview.md`, `backend-brief.md`, `../../03-architecture/api-conventions.md`, `../../03-architecture/authentication.md`, `../platform-admin/api-contract.md`

## Essay — what this is

This is the agreed contract between the organization admin backend and its frontend. It names endpoints, payloads, and error codes. It does not restate product behaviour (`overview.md`), the common error and list formats (`api-conventions.md`), or how sessions work (`authentication.md`); it links to them.

After implementation, the live API is the OpenAPI document of the organization API (`/api/organization/docs/json`). Differences from this contract are recorded in `backend-brief-feedback.md`.

## Essay — general

- **Base URL:** `https://api.studio-desk.axondigital.xyz/api/organization`. Paths below are relative to it.
- **Authorization:** every endpoint requires `Authorization: Bearer <accessToken>` of an organization session, except those marked **public**. A missing, invalid, or revoked token returns `401 UNAUTHORIZED`. A platform session is not accepted. An organization session works only for its own organization.
- **Section access (double filter):** an endpoint of a section returns `403 FORBIDDEN` (screen text: «You don't have access to this page») unless the section is **both enabled for the organization (`organization_section`) and allowed for the person's role (`role_section`)**. The role, enabled sections, and rights are read from the database on every request. Staff and organization settings are owner-only.
- **Cookie:** the refresh token is the cookie `sd_organization_refresh` with `HttpOnly; Secure; SameSite=Strict; Path=/api/organization/auth`. The frontend calls `/auth/*` with credentials included (`credentials: "include"`), because `app.` and `api.` are different origins.
- **`X-Requested-With`:** required on `POST /auth/refresh`, `POST /auth/sign-out`, `POST /auth/switch-organization`, and `POST /auth/impersonation/exchange`. Without it: `403 FORBIDDEN`.
- **Formats:** dates are ISO 8601 strings in UTC. Lifetimes (`…ExpiresIn`, `…AvailableIn`) are seconds from now. IDs are UUIDs. Errors use the common format; field errors use `VALIDATION_ERROR` (`api-conventions.md`). Text fields are trimmed. E-mails are lowercased before the format check.
- **Several errors:** format errors of all fields come together in one `VALIDATION_ERROR`. Checks that need the database run only when the format of all fields is valid, and return one error at a time in the order of the endpoint's table.

### Objects

`OrganizationRef`: `{ "id": "6f1c…", "name": "Yoga Space" }`.

`SessionBody`, the body of every successful sign-in, organization selection, switch, refresh, and exchange that creates or renews a session:

```json
{
  "accessToken": "eyJ…",
  "accessTokenExpiresIn": 900,
  "user": { "email": "anna@example.com" },
  "organization": { "id": "6f1c…", "name": "Yoga Space" }
}
```

It is the same shape as the exchange response in `../platform-admin/api-contract.md`. Everything else the frontend needs about the person and the organization's configuration is read with `GET /me`.

`Role`: `owner`, `administrator`, `accountant`, `specialist`.

`Section`: `schedule`, `catalogs`, `clients`, `subscriptions`, `accounting`, `organization-settings`, `staff`.

`Language`: `en`, `sk`, `uk`.

`OrganizationConfig`, the organization's own modular configuration (copied from its preset on creation and editable in `Organization settings`):

```json
{
  "catalogConfig": {
    "resourceTypes": [
      {
        "id": "rt1…",
        "sortOrder": 0,
        "role": "place",
        "labels": {
          "en": { "singular": "Hall", "plural": "Halls" },
          "sk": { "singular": "Sála", "plural": "Sály" },
          "uk": { "singular": "Зал", "plural": "Зали" }
        },
        "geography": true,
        "duration": false,
        "showcase": true,
        "person": false,
        "skills": false,
        "required": true,
        "multiple": false,
        "conflictMode": "strict"
      },
      {
        "id": "rt2…",
        "sortOrder": 1,
        "role": "resource",
        "labels": {
          "en": { "singular": "Trainer", "plural": "Trainers" },
          "sk": { "singular": "Trenér", "plural": "Trenéri" },
          "uk": { "singular": "Тренер", "plural": "Тренери" }
        },
        "geography": false,
        "duration": false,
        "showcase": true,
        "person": true,
        "skills": true,
        "required": true,
        "multiple": true,
        "conflictMode": "strict"
      },
      {
        "id": "rt3…",
        "sortOrder": 2,
        "role": "offering",
        "labels": {
          "en": { "singular": "Class type", "plural": "Class types" },
          "sk": { "singular": "Typ lekcie", "plural": "Typy lekcií" },
          "uk": { "singular": "Вид заняття", "plural": "Види занять" }
        },
        "geography": false,
        "duration": true,
        "showcase": true,
        "person": false,
        "skills": true,
        "required": true,
        "multiple": false,
        "conflictMode": "none"
      }
    ]
  },
  "scheduleConfig": {
    "audienceEnabled": true,
    "formatEnabled": true,
    "statuses": ["planned", "cancelled"]
  },
  "billingConfig": {
    "settlementMode": "subscription"
  }
}
```

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

Both values are seconds from now. The code is always 6 digits. A code is e-mailed only if the e-mail is the owner of an active organization or a staff member of an active organization.

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

One active organization: the session is created at once. `Set-Cookie: sd_organization_refresh=…`.

```json
{
  "result": "signed-in",
  "accessToken": "eyJ…",
  "accessTokenExpiresIn": 900,
  "user": { "email": "anna@example.com" },
  "organization": { "id": "6f1c…", "name": "Yoga Space" }
}
```

Several active organizations: no session and no cookie. The frontend shows the organization selection page.

```json
{
  "result": "organization-selection",
  "selectionTicket": "q8Z…",
  "selectionTicketExpiresIn": 300,
  "organizations": [{ "id": "6f1c…", "name": "Yoga Space" }]
}
```

The ticket is single-use and lives 300 seconds.

Errors, in check order:

| Status | `code` | When |
|---|---|---|
| 400 | `VALIDATION_ERROR` | e-mail or code format is invalid |
| 429 | `TOO_MANY_ATTEMPTS` | attempt limit reached for this code |
| 400 | `CODE_EXPIRED` | the code has expired |
| 400 | `INVALID_CODE` | the code is wrong, there is no code for this e-mail, or the e-mail has no active organization |

### `POST /auth/select-organization` — choose an organization (public, by ticket)

Body:

```json
{ "selectionTicket": "q8Z…", "organizationId": "6f1c…" }
```

`200` `SessionBody`, and `Set-Cookie: sd_organization_refresh=…`. The ticket is spent.

| Status | `code` | When |
|---|---|---|
| 400 | `VALIDATION_ERROR` | body format is invalid |
| 400 | `INVALID_SELECTION_TICKET` | the ticket is unknown, expired, or already used |
| 404 | `NOT_FOUND` | the organization is not one of the person's active organizations |

### `POST /auth/switch-organization` — switch organization

Requires Bearer and `X-Requested-With`. Body:

```json
{ "organizationId": "6f1c…" }
```

`200` `SessionBody` of the new organization, and a new `Set-Cookie`. A new session is created and the current one is revoked, with no new code. If the current session is a log-in-as-organization session (`impersonated_by`), the mark passes to the new session.

| Status | `code` | When |
|---|---|---|
| 400 | `VALIDATION_ERROR` | body format is invalid |
| 404 | `NOT_FOUND` | the organization is not one of the person's active organizations |

### `POST /auth/refresh` — new tokens (public, cookie)

No body. Requires the cookie and `X-Requested-With`.

`200` `SessionBody` and a new `Set-Cookie`.

| Status | `code` | When |
|---|---|---|
| 401 | `UNAUTHORIZED` | no cookie: the user has not signed in; the frontend shows the sign-in screen without a message |
| 401 | `SESSION_EXPIRED` | session expired, revoked, or the refresh token was reused; the cookie is cleared |

### `POST /auth/sign-out` — sign out (public, cookie)

No body. Requires `X-Requested-With`. Revokes the session of the cookie and clears the cookie. `204` always.

### `POST /auth/impersonation/exchange` — exchange the handoff code (public)

Defined in `../platform-admin/api-contract.md`. It returns `SessionBody` and sets `sd_organization_refresh`.

## Essay — the person

### `GET /me` — who am I

Any organization session. `200`:

```json
{
  "user": { "email": "anna@example.com", "interfaceLanguage": "sk" },
  "organization": { "id": "6f1c…", "name": "Yoga Space" },
  "role": "owner",
  "sections": ["schedule", "catalogs", "clients", "subscriptions", "accounting", "organization-settings", "staff"],
  "config": {
    "catalogConfig": { "resourceTypes": [{ "…": "…" }] },
    "scheduleConfig": { "…": "…" },
    "billingConfig": { "…": "…" }
  },
  "impersonated": false,
  "organizations": [{ "id": "6f1c…", "name": "Yoga Space" }]
}
```

- `interfaceLanguage`: `en`, `sk`, `uk`, or `null` when the person has not chosen one (the interface is then English).
- `sections`: intersection of the sections enabled for this organization (`organization_section`) and the sections allowed for the person's role (`role_section`), in canonical tab order.
- `config`: the organization's `OrganizationConfig` so the UI immediately knows localized resource and service labels.
- `impersonated`: `true` for a log-in-as-organization session.
- `organizations`: all active organizations of the person, ordered by name. The «Switch organization» item is shown if there is more than one.

### `PATCH /me/language` — choose the interface language

Body:

```json
{ "language": "uk" }
```

`200`: `{ "interfaceLanguage": "uk" }`. The choice is stored by e-mail and applies in all the person's organizations.

| Status | `code` | `field` | When |
|---|---|---|---|
| 400 | `VALIDATION_ERROR` (`REQUIRED`) | `language` | not sent or empty |
| 400 | `VALIDATION_ERROR` (`INVALID_VALUE`) | `language` | not `en`, `sk`, or `uk` |

## Essay — organization settings

Section `organization-settings`, owner only (`403 FORBIDDEN` for others).

### Objects

`OrganizationSettings`:

```json
{
  "language": "en",
  "country": "SK",
  "currency": "EUR",
  "timeZone": "Europe/Bratislava",
  "catalogConfig": { "resourceTypes": [{ "…": "…" }] },
  "scheduleConfig": { "…": "…" },
  "billingConfig": { "…": "…" }
}
```

- `language`: `en`, `sk`, or `uk`; the language of the organization for clients.
- `country`: ISO 3166-1 alpha-2 code, upper case.
- `currency`: code from the table of allowed currencies in the database (`EUR`, `UAH`, `USD`).
- `timeZone`: IANA time zone name.
- `catalogConfig`, `scheduleConfig`, and `billingConfig`: the organization's own modular configuration (`OrganizationConfig`).

### `GET /settings`

`200`: `OrganizationSettings`.

### `GET /settings/options`

The allowed values for the selects. `200`:

```json
{
  "countries": ["SK", "UA", "…"],
  "currencies": ["EUR", "UAH", "USD"],
  "timeZones": ["Europe/Bratislava", "…"]
}
```

### `PATCH /settings`

Body: any subset of `OrganizationSettings`. A field that is not sent is not changed. `200`: `OrganizationSettings`. Screen text on success: «Organization settings saved».

| Status | `code` | `field` | When |
|---|---|---|---|
| 400 | `VALIDATION_ERROR` (`REQUIRED`) | `language`, `country`, `currency`, `timeZone` | sent as `null` or empty |
| 400 | `VALIDATION_ERROR` (`INVALID_VALUE`) | `language`, `country`, `currency`, `timeZone`, `catalogConfig`, `scheduleConfig`, `billingConfig` | value is outside its allowed set or violates resource type invariants |
| 409 | `RESOURCE_TYPE_IN_USE` | `catalogConfig.resourceTypes` | attempting to delete a resource type that has resources or disable `person` on a resource type linked to a staff member |

## Essay — staff

Section `staff`, owner only (`403 FORBIDDEN` for others). The owner is not in the list.

A staff member with the role `specialist` is linked to exactly one catalog resource (`resourceId`) of this organization whose resource type has `person: true`. The link is what lets that person open Schedule and see only the slots of that resource. `GET /me` does not return the link: Schedule reads it from the session on each request.

### Objects

`StaffMember`:

```json
{
  "id": "2b9e…",
  "email": "olga@example.com",
  "role": "specialist",
  "resourceId": "4a1b…",
  "createdAt": "2026-10-06T10:00:00Z"
}
```

`role` is `administrator`, `accountant`, or `specialist`. `resourceId` is the catalog resource id when `role` is `specialist`, and `null` otherwise.

### `GET /staff` — list

Query, in addition to the common list parameters (`api-conventions.md`):

- `search`: case-insensitive substring of the e-mail.
- `sortBy`: `createdAt` (default) or `email`.

`200`: `{ "items": StaffMember[], "meta": … }` in the common list format. Each item includes `resourceId`.

### `POST /staff` — add

Body:

```json
{ "email": "olga@example.com", "role": "specialist", "resourceId": "4a1b…" }
```

`role` `administrator` or `accountant` omits `resourceId` or sends `null`. `role` `specialist` requires `resourceId`: the id of a catalog resource of this organization whose resource type has `person: true` and that is not already linked. The staff row and the link are written in one transaction.

`201`: `StaffMember`. Screen text on success: «Staff member added».

Errors. All `VALIDATION_ERROR` field errors come together. The checks that need the database then run one at a time, in this order:

| Status | `code` | `field` | Fixed text in the canon |
|---|---|---|---|
| 400 | `VALIDATION_ERROR` (`REQUIRED`) | `email`, `role` | — |
| 400 | `VALIDATION_ERROR` (`REQUIRED`) | `resourceId` | Select a specialist |
| 400 | `VALIDATION_ERROR` (`INVALID_FORMAT`) | `email` | Enter a valid e-mail address |
| 400 | `VALIDATION_ERROR` (`INVALID_FORMAT`) | `resourceId` | — (not a UUID) |
| 400 | `VALIDATION_ERROR` (`INVALID_VALUE`) | `role` | — (not `administrator`, `accountant`, or `specialist`) |
| 400 | `VALIDATION_ERROR` (`INVALID_VALUE`) | `resourceId` | sent with `administrator` or `accountant`, or not a `person` resource of this organization |
| 409 | `STAFF_ALREADY_ADDED` | `email` | This e-mail is already added to the organization |
| 409 | `EMAIL_IS_OWNER` | `email` | This e-mail belongs to the organization owner |
| 409 | `SPECIALIST_ALREADY_LINKED` | `resourceId` | This specialist is already linked to another staff member |

### `GET /staff/{id}` — one staff member

`200`: `StaffMember`. `404 NOT_FOUND` if there is no such staff member in this organization.

### `PATCH /staff/{id}` — change the role and the link

Body: any subset of `role` and `resourceId`. A field that is not sent is not changed. The e-mail does not change. The new role acts from the next request.

- Becoming `specialist` requires `resourceId` in this request.
- Staying `specialist`: an omitted `resourceId` keeps the current resource; a sent `resourceId` replaces it; `null` is refused (`REQUIRED`).
- Becoming `administrator` or `accountant` removes the link. A `resourceId` sent on that request is refused.

`200`: `StaffMember`. Screen text on success: «Staff member updated».

### `DELETE /staff/{id}` — remove

`204`. The link row is deleted. The catalog resource stays. All sessions of this person in this organization are revoked at once; sessions of the same e-mail in other organizations are not affected. Screen text on success: «Staff member removed». `404 NOT_FOUND` if there is no such staff member in this organization.

### Tables

`organization_staff.role` accepts `administrator`, `accountant`, and `specialist`.

`staff_resource`

| Column | |
|---|---|
| `organization_id` | uuid, not null, references `organization.id` |
| `staff_id` | uuid, primary key, references `organization_staff.id`. One row per staff member. Deleting the staff member deletes this row |
| `resource_id` | uuid, not null, references `resource.id`, unique. One catalog resource has at most one staff member. Deleting that resource is refused while the row exists |
| `created_at` | timestamptz, not null |

Grants: `organization_api` has `SELECT, INSERT, UPDATE, DELETE` on `staff_resource`. `platform_api` and `public_api` have none.

## Essay — error codes and screen texts

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
| `STAFF_ALREADY_ADDED` | This e-mail is already added to the organization |
| `EMAIL_IS_OWNER` | This e-mail belongs to the organization owner |
| `VALIDATION_ERROR` (`REQUIRED`) on `resourceId` | Select a specialist |
| `SPECIALIST_ALREADY_LINKED` | This specialist is already linked to another staff member |
| any other error of an action or of loading | Something went wrong. Try again |

## Essay — how it connects

- Product behaviour and texts: `overview.md`.
- Requirements and phases: `backend-brief.md`.
- Errors, lists, OpenAPI: `../../03-architecture/api-conventions.md`.
- Sessions, cookie, handoff code, organization selection and switching: `../../03-architecture/authentication.md`.
- The handoff code exchange and `OWNER_IS_STAFF`: `../platform-admin/api-contract.md`.

## Agent — rules

- Every endpoint, code, and field here appears in the OpenAPI document of the organization API.
- Renaming a `code` or a field is a contract change: agree it first.
- The response to `POST /auth/code` and the errors of `POST /auth/sign-in` never reveal whether an e-mail has access to any organization.
- A session of the organization API never changes its organization: selection and switching create a new session.
- Role, enabled organization sections, and rights are read from the database on every request, not taken from the token.
- No endpoint deletes an organization or changes the owner.
- The `specialist` role is assigned only together with one catalog resource (`person = true`) of this organization. `administrator` and `accountant` have no link.

## Agent — acceptance checklist

- [ ] Each endpoint matches its path, method, body, response, and status codes
- [ ] Format errors of all fields come in one response; `409` checks follow in table order
- [ ] E-mails are lowercased on save, search, and sign-in
- [ ] Cookie name (`sd_organization_refresh`) and attributes, `X-Requested-With`, and CORS match `authentication.md`
- [ ] Identical answers for an e-mail with and without an active organization
- [ ] The selection ticket is single-use and expires
- [ ] A switch revokes the previous session and keeps `impersonated_by`
- [ ] `GET /me` shows the role, intersection of `organization_section` and `role_section`, `config`, and active organizations read from the database
- [ ] Organization settings and staff are rejected for every role except the owner
- [ ] Removing a staff member revokes their sessions in this organization only
- [ ] A `specialist` staff member stores exactly one catalog resource (`person = true`) of this organization; `resourceId` is `null` for the other roles
- [ ] `GET /me` does not return the specialist resource link

## Open questions

None.
