# API contract — Platform admin

**Scope:** the HTTP API of the platform admin (`/api/platform`): sign-in, presets, organizations, deactivation and activation, log in as organization; plus the one organization API endpoint that log in as organization needs (handoff code exchange).
**Derived from:** `overview.md`, `backend-brief.md`, `../../03-architecture/api-conventions.md`, `../../03-architecture/authentication.md`

## Essay — what this is

This is the agreed contract between the platform admin backend and its frontend. It names endpoints, payloads, and error codes. It does not restate product behaviour (`overview.md`), the common error and list formats (`api-conventions.md`), or how sessions work (`authentication.md`); it links to them.

After implementation, the live API is the OpenAPI document of the platform API (`/api/platform/docs/json`). Differences from this contract are recorded in `backend-brief-feedback.md`.

## Essay — general

- **Base URL:** `https://api.studio-desk.axondigital.xyz/api/platform`. Paths below are relative to it.
- **Authorization:** every endpoint requires `Authorization: Bearer <accessToken>` of a platform session, except those marked **public**. A missing, invalid, or revoked token returns `401 UNAUTHORIZED`. An organization session is not accepted.
- **Cookie:** the refresh token is the cookie `sd_platform_refresh` with `HttpOnly; Secure; SameSite=Strict; Path=/api/platform/auth`. The frontend calls `/auth/*` with credentials included (`credentials: "include"`), because `admin.` and `api.` are different origins.
- **`X-Requested-With`:** required on `POST /auth/refresh` and `POST /auth/sign-out`. Without it: `403 FORBIDDEN`.
- **Formats:** dates are ISO 8601 strings in UTC. Lifetimes (`…ExpiresIn`, `…AvailableIn`) are seconds from now. IDs are UUIDs. Errors use the common format; field errors use `VALIDATION_ERROR` (`api-conventions.md`). Text fields are trimmed. E-mails, subdomains, and custom domains are lowercased before the format check; an uppercase letter in them is corrected, not rejected. A name keeps its case. Empty optional values clear the field (`api-conventions.md`, «пустые значения»).
- **Several errors:** format errors of all fields come together in one `VALIDATION_ERROR`. Checks that need the database (reserved, taken) run only when the format of all fields is valid, and return one error at a time in the order of the endpoint's table.

## Essay — sign-in

### `POST /auth/code` — request a sign-in code (public)

Body:

```json
{ "email": "admin@example.com" }
```

`200`, the same for any e-mail, whether or not it has access:

```json
{
  "codeExpiresIn": 600,
  "resendAvailableIn": 60
}
```

Both values are seconds from now. The code is always 6 digits. The same endpoint is used to resend; a new code replaces the previous one and gives a full set of attempts, also after `TOO_MANY_ATTEMPTS`. A code is e-mailed only if the e-mail is the platform administrator's.

| Status | `code` | `field` | When |
|---|---|---|---|
| 400 | `VALIDATION_ERROR` (`INVALID_FORMAT`) | `email` | e-mail format is invalid |
| 429 | `RESEND_TOO_EARLY` | — | called again before the resend wait is over; applies to any e-mail |

### `POST /auth/sign-in` — sign in with the code (public)

Body:

```json
{ "email": "admin@example.com", "code": "123456" }
```

`200`, and `Set-Cookie: sd_platform_refresh=…`:

```json
{
  "accessToken": "eyJ…",
  "accessTokenExpiresIn": 900,
  "user": { "email": "admin@example.com" }
}
```

Errors, in check order:

| Status | `code` | When |
|---|---|---|
| 400 | `VALIDATION_ERROR` | e-mail or code format is invalid |
| 429 | `TOO_MANY_ATTEMPTS` | attempt limit reached for this code |
| 400 | `CODE_EXPIRED` | the code has expired |
| 400 | `INVALID_CODE` | the code is wrong, or there is no code for this e-mail |

For an e-mail without access, the answers must be indistinguishable from those for the administrator's e-mail with a wrong code (including `CODE_EXPIRED` and `TOO_MANY_ATTEMPTS` timing).

### `POST /auth/refresh` — new tokens (public, cookie)

No body. Requires the cookie and `X-Requested-With`.

`200`, the same body as sign-in, and a new `Set-Cookie`.

| Status | `code` | When |
|---|---|---|
| 401 | `UNAUTHORIZED` | no cookie: the user has not signed in; the frontend shows the sign-in screen without a message |
| 401 | `SESSION_EXPIRED` | session expired, revoked, or the refresh token was reused; the cookie is cleared |

### `POST /auth/sign-out` — sign out (public, cookie)

No body. Requires `X-Requested-With`. Revokes the session of the cookie and clears the cookie. `204` always, also without a cookie or for an already revoked session.

## Essay — presets

### Objects

`LocalizedLabel`:

```json
{
  "en": { "singular": "Hall", "plural": "Halls" },
  "sk": { "singular": "Sála", "plural": "Sály" },
  "uk": { "singular": "Зал", "plural": "Зали" }
}
```

`Section`: `schedule`, `catalogs`, `clients`, `subscriptions`, `accounting`, `organization-settings`, `staff`.

`SlotStatus`: `planned`, `confirmed`, `in_progress`, `completed`, `cancelled`, `no_show`. `planned` and `cancelled` are always present.

`SettlementMode`: `subscription`, `direct_check`, `external_order`.

`ConflictMode`: `strict`, `warning`, `none`.

`ServiceConfig`:

```json
{
  "labels": {
    "en": { "singular": "Class type", "plural": "Class types" },
    "sk": { "singular": "Typ lekcie", "plural": "Typy lekcií" },
    "uk": { "singular": "Вид заняття", "plural": "Види занять" }
  },
  "audienceEnabled": true,
  "formatEnabled": true,
  "statuses": ["planned", "cancelled"],
  "settlementMode": "subscription"
}
```

`ResourceTypeConfig`:

```json
{
  "key": "place",
  "role": "place",
  "labels": {
    "en": { "singular": "Hall", "plural": "Halls" },
    "sk": { "singular": "Sála", "plural": "Sály" },
    "uk": { "singular": "Зал", "plural": "Зали" }
  },
  "showcase": true,
  "geography": true,
  "person": false,
  "skills": false,
  "required": true,
  "multiple": false,
  "conflictMode": "strict"
}
```

- Every preset has at least one `ResourceTypeConfig` and **exactly one** item with `role: "place"` (`geography: true`, `required: true`, `multiple: false`).
- Items with `role: "resource"` have `geography: false`.

`Preset`:

```json
{
  "id": "p1a2…",
  "name": "Fitness Studio",
  "description": "Group and personal classes in halls and online",
  "sections": ["schedule", "catalogs", "clients", "subscriptions", "accounting", "organization-settings", "staff"],
  "serviceConfig": { "…": "…" },
  "resourceTypes": [{ "…": "…" }],
  "createdAt": "2026-10-27T10:00:00Z",
  "updatedAt": "2026-10-27T10:00:00Z"
}
```

### `GET /presets` — list

Query, in addition to the common list parameters (`api-conventions.md`):
- `search`: case-insensitive substring of `name`.
- `sortBy`: `createdAt` (default) or `name`.

`200`: `{ "items": Preset[], "meta": … }`.

### `POST /presets` — create

Body: `name`, `description` (optional), `sections`, `serviceConfig`, `resourceTypes`.

`201`: `Preset`. Screen text: «Preset created».

| Status | `code` | `field` | Fixed text in the canon |
|---|---|---|---|
| 400 | `VALIDATION_ERROR` (`REQUIRED` / `INVALID_FORMAT`) | `name` | Use 2–100 characters |
| 400 | `VALIDATION_ERROR` (`REQUIRED`) | `sections` | Select at least one section |
| 400 | `VALIDATION_ERROR` (`REQUIRED`) | `serviceConfig.labels`, `resourceTypes[].labels` | Enter all language labels |
| 400 | `VALIDATION_ERROR` (`INVALID_VALUE`) | `resourceTypes` | Exactly one place resource type is required |
| 409 | `PRESET_NAME_TAKEN` | `name` | This preset name is already taken |

### `GET /presets/{id}` — one preset

`200`: `Preset`. `404 NOT_FOUND` if there is no such preset.

### `PATCH /presets/{id}` — update

Body: any subset of the create body. Updating a preset does not modify already created organizations. `200`: `Preset`. Screen text: «Preset updated». Errors: as for create, plus `404 NOT_FOUND`.

### `POST /presets/{id}/duplicate` — duplicate

No body. Creates a copy of the preset with a unique name (`{name} (copy)`, or `{name} (copy 2)` if taken). `201`: `Preset`. Screen text: «Preset duplicated». `404 NOT_FOUND` if there is no such preset.

### `DELETE /presets/{id}` — delete

`204`. Organizations created from this preset keep their copied configuration (`presetId` on those organizations is set to `null`, while `presetName` snapshot is kept). Screen text: «Preset deleted». `404 NOT_FOUND` if there is no such preset.

## Essay — organizations

### Objects

`Organization`:

```json
{
  "id": "6f1c…",
  "name": "Yoga Space",
  "subdomain": "yoga-space",
  "customDomain": null,
  "owner": { "email": "owner@example.com" },
  "preset": { "id": "p1a2…", "name": "Fitness Studio" },
  "sections": ["schedule", "catalogs", "clients", "subscriptions", "accounting", "organization-settings", "staff"],
  "status": "active",
  "createdAt": "2026-10-03T10:00:00Z",
  "updatedAt": "2026-10-03T10:00:00Z"
}
```

- `status`: `active` or `deactivated`.
- `customDomain`: string or `null`.
- `preset`: `{ "id": string | null, "name": string }` — read-only reference to the preset used when the organization was created.

`OrganizationListItem`: `id`, `name`, `preset`, `subdomain`, `customDomain`, `owner`, `status`, `createdAt`.

### `GET /organizations` — list

Query, in addition to the common list parameters (`api-conventions.md`):

- `status`: `active` (default) or `deactivated`.
- `search`: case-insensitive substring of name, subdomain, custom domain, or owner e-mail.
- `sortBy`: `createdAt` only.

`200`: `{ "items": OrganizationListItem[], "meta": … }` in the common list format.

### `POST /organizations` — create

Body:

```json
{
  "name": "Yoga Space",
  "subdomain": "yoga-space",
  "customDomain": null,
  "owner": { "email": "owner@example.com" },
  "presetId": "p1a2…"
}
```

`customDomain` is optional. The organization, its owner, and its copied configuration (sections, `serviceConfig`, `resourceTypes` from the chosen preset) are created together in one transaction; the new organization is `active`. `201`: `Organization`. Screen text: «Organization created».

Errors. All `VALIDATION_ERROR` field errors come together; the `409` checks follow in table order:

| Status | `code` | `field` | Fixed text in the canon |
|---|---|---|---|
| 400 | `VALIDATION_ERROR` (`REQUIRED`) | `name`, `subdomain`, `owner.email`, `presetId` | — (`presetId`: Select a preset) |
| 400 | `VALIDATION_ERROR` (`INVALID_FORMAT`) | `name` | Use 2–100 characters |
| 400 | `VALIDATION_ERROR` (`INVALID_FORMAT`) | `subdomain` | Use 3–20 lowercase letters, digits or hyphens, starting with a letter |
| 400 | `VALIDATION_ERROR` (`INVALID_FORMAT`) | `customDomain` | Enter a valid domain, for example yogaspace.com |
| 400 | `VALIDATION_ERROR` (`INVALID_FORMAT`) | `owner.email` | Enter a valid e-mail address |
| 400 | `VALIDATION_ERROR` (`INVALID_VALUE`) | `presetId` | Select a preset (when preset does not exist) |
| 409 | `SUBDOMAIN_RESERVED` | `subdomain` | This subdomain is reserved |
| 409 | `SUBDOMAIN_TAKEN` | `subdomain` | This subdomain is already taken |
| 409 | `DOMAIN_TAKEN` | `customDomain` | This domain is already used by another organization |

Uniqueness counts organizations in any state. A custom domain on the platform domain `studio-desk.axondigital.xyz` is an invalid format.

### `GET /organizations/{id}` — one organization

`200`: `Organization`. `404 NOT_FOUND` if there is no such organization.

### `PATCH /organizations/{id}` — update

Body: any subset of `name`, `subdomain`, `customDomain`, `owner`, and `sections`. `presetId` cannot be changed on an existing organization. A field that is not sent is not changed; an empty `customDomain` removes the custom domain. Works in any organization state.

- A new `owner.email` makes the person with that e-mail the owner. All sessions of the previous owner in this organization are revoked (`authentication.md`).
- `sections` updates the organization's enabled sections (`organization_section`); at least one section must remain enabled.

`200`: `Organization`. Screen text: «Organization updated». Errors: as for create (plus `REQUIRED` on empty `sections`: «Select at least one section»), `404 NOT_FOUND`, and:

| Status | `code` | `field` | Fixed text in the canon |
|---|---|---|---|
| 409 | `OWNER_IS_STAFF` | `owner.email` | This e-mail is already a staff member of this organization |

### `POST /organizations/{id}/deactivate` and `POST /organizations/{id}/activate`

No body. `200`: `Organization` with the new status. Deactivation revokes the organization's sessions as described in `authentication.md`. `404 NOT_FOUND` if there is no such organization.

### `POST /organizations/{id}/impersonate` — log in as organization

No body. Works in any organization state. `200`:

```json
{ "code": "k3J…", "expiresIn": 60 }
```

On click, the frontend opens an empty new tab at once and, after this response, sends it to the organization admin with the handoff code after `#`. Flow: `authentication.md`. `404 NOT_FOUND` if there is no such organization.

### `POST /api/organization/auth/impersonation/exchange` — exchange the handoff code (organization API, public)

Full path, in the organization API. Called by the organization admin. Requires `X-Requested-With`.

Body:

```json
{ "code": "k3J…" }
```

`200`, and `Set-Cookie: sd_organization_refresh=…` (`HttpOnly; Secure; SameSite=Strict; Path=/api/organization/auth`):

```json
{
  "accessToken": "eyJ…",
  "accessTokenExpiresIn": 900,
  "user": { "email": "owner@example.com" },
  "organization": { "id": "6f1c…", "name": "Yoga Space" }
}
```

The session is an organization API session of the current owner with `impersonated_by` set (`authentication.md`).

| Status | `code` | When |
|---|---|---|
| 400 | `INVALID_HANDOFF_CODE` | the code is unknown, expired, or already used |

## Essay — error codes and screen texts

The frontend picks the text by `code` (and `field`). Fixed texts come from `overview.md`.

| `code` | Text |
|---|---|
| `VALIDATION_ERROR` on `email` / `owner.email` | Enter a valid e-mail address |
| `VALIDATION_ERROR` on `name` | Use 2–100 characters |
| `VALIDATION_ERROR` on `subdomain` | Use 3–20 lowercase letters, digits or hyphens, starting with a letter |
| `VALIDATION_ERROR` on `customDomain` | Enter a valid domain, for example yogaspace.com |
| `VALIDATION_ERROR` on `presetId` | Select a preset |
| `VALIDATION_ERROR` on `sections` | Select at least one section |
| `VALIDATION_ERROR` on `labels` | Enter all language labels |
| `PRESET_NAME_TAKEN` | This preset name is already taken |
| `RESEND_TOO_EARLY` | Please wait before requesting a new code |
| `TOO_MANY_ATTEMPTS` | Too many attempts. Try again later |
| `CODE_EXPIRED` | The code has expired. Request a new one |
| `INVALID_CODE` | Invalid or expired code |
| `SESSION_EXPIRED` | Your session has expired. Sign in again |
| `UNAUTHORIZED` from `POST /auth/refresh` | no text, sign-in screen |
| `SUBDOMAIN_RESERVED` | This subdomain is reserved |
| `SUBDOMAIN_TAKEN` | This subdomain is already taken |
| `DOMAIN_TAKEN` | This domain is already used by another organization |
| `OWNER_IS_STAFF` | This e-mail is already a staff member of this organization |
| any other error of an action or of loading the list | Something went wrong. Try again |

## Essay — how it connects

- Product behaviour and texts: `overview.md`.
- Requirements and phases: `backend-brief.md`.
- Errors, lists, OpenAPI: `../../03-architecture/api-conventions.md`.
- Sessions, cookie, handoff code: `../../03-architecture/authentication.md`.

## Agent — rules

- Every endpoint, code, and field here appears in the OpenAPI document of the platform API.
- Renaming a `code` or a field is a contract change: agree it first.
- No endpoint physically deletes an organization or manages platform administrators.
- Creating an organization copies the chosen preset into the organization's own configuration tables in one transaction; `presetId` is not changeable on `PATCH /organizations/{id}`.
- The response to `POST /auth/code` and the errors of `POST /auth/sign-in` never reveal whether an e-mail has access.

## Agent — acceptance checklist

- [ ] Each endpoint matches its path, method, body, response, and status codes
- [ ] Presets CRUD and duplicate (`/presets`) validate `sections`, `serviceConfig`, and `resourceTypes` (exactly one `place` resource type and all three language labels `en/sk/uk`)
- [ ] `POST /organizations` requires `presetId` and copies the preset configuration into the organization
- [ ] `PATCH /organizations/{id}` allows updating `sections` and does not allow overwriting the preset
- [ ] Format errors of all fields come in one response; `409` checks follow in table order
- [ ] E-mails, subdomains, and custom domains are lowercased on save and search; e-mails are lowercased on sign-in
- [ ] Cookie name and attributes, `X-Requested-With`, and CORS match `authentication.md`
- [ ] Identical answers for an e-mail with and without access
- [ ] Uniqueness checks include deactivated organizations
- [ ] An e-mail of a current staff member of the organization cannot become its owner (`OWNER_IS_STAFF`)

## Open questions

None.
