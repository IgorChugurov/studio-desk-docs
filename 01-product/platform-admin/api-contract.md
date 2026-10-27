# API contract — Platform admin

**Scope:** the HTTP API of the platform admin (`/api/platform`): sign-in, modular organization presets, organizations, deactivation and activation, log in as organization; plus the one organization API endpoint that log in as organization needs (handoff code exchange).
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

| Status | `code` | When |
|---|---|---|
| 400 | `VALIDATION_ERROR` | e-mail or code format is invalid |
| 429 | `TOO_MANY_ATTEMPTS` | attempt limit reached for this code |
| 400 | `CODE_EXPIRED` | the code has expired |
| 400 | `INVALID_CODE` | the code is wrong, or there is no code for this e-mail |

### `POST /auth/refresh` — new tokens (public, cookie)

No body. Requires the cookie and `X-Requested-With`. `200`, the same body as sign-in, and a new `Set-Cookie`.

| Status | `code` | When |
|---|---|---|
| 401 | `UNAUTHORIZED` | no cookie |
| 401 | `SESSION_EXPIRED` | session expired, revoked, or refresh token reused |

### `POST /auth/sign-out` — sign out (public, cookie)

No body. Requires `X-Requested-With`. Revokes the session of the cookie and clears the cookie. `204` always.

## Essay — presets (modular organization blueprints)

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

`ResourceRole`:
- `place` — location and time-zone anchor (exactly one per preset);
- `resource` — specialist or equipment occupied during a slot;
- `offering` — service, work type, or class type performed during a slot.

`ConflictMode`: `strict`, `warning`, `none`.

`SlotStatus`: `planned`, `confirmed`, `in_progress`, `completed`, `cancelled`, `no_show`. `planned` and `cancelled` are always present.

`SettlementMode`: `subscription`, `direct_check`, `external_order`.

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
  "geography": true,
  "duration": false,
  "showcase": true,
  "person": false,
  "skills": false,
  "required": true,
  "multiple": false,
  "conflictMode": "strict"
}
```

- Every preset has at least one `ResourceTypeConfig` in `catalogConfig.resourceTypes` and **exactly one** item with `role: "place"` (`geography: true`, `duration: false`, `person: false`, `required: true`, `multiple: false`).
- Items with `role: "resource"` have `geography: false` and `duration: false`.
- Items with `role: "offering"` have `duration: true`, `geography: false`, `person: false`, and `conflictMode: "none"`.

`CatalogConfig`:

```json
{
  "resourceTypes": [{ "…": "…" }]
}
```

`ScheduleConfig`:

```json
{
  "audienceEnabled": true,
  "formatEnabled": true,
  "statuses": ["planned", "cancelled"]
}
```

`BillingConfig`:

```json
{
  "settlementMode": "subscription"
}
```

`Preset`:

```json
{
  "id": "p1a2…",
  "name": "Fitness Studio",
  "description": "Group and personal classes in halls and online",
  "sections": ["schedule", "catalogs", "clients", "subscriptions", "accounting", "organization-settings", "staff"],
  "catalogConfig": { "resourceTypes": [{ "…": "…" }] },
  "scheduleConfig": {
    "audienceEnabled": true,
    "formatEnabled": true,
    "statuses": ["planned", "cancelled"]
  },
  "billingConfig": {
    "settlementMode": "subscription"
  },
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

Body: `name`, `description` (optional), `sections`, `catalogConfig`, `scheduleConfig`, `billingConfig`.

`201`: `Preset`. Screen text: «Preset created».

| Status | `code` | `field` | Fixed text in the canon |
|---|---|---|---|
| 400 | `VALIDATION_ERROR` (`REQUIRED` / `INVALID_FORMAT`) | `name` | Use 2–100 characters |
| 400 | `VALIDATION_ERROR` (`REQUIRED`) | `sections` | Select at least one section |
| 400 | `VALIDATION_ERROR` (`REQUIRED`) | `catalogConfig.resourceTypes[].labels` | Enter all language labels |
| 400 | `VALIDATION_ERROR` (`INVALID_VALUE`) | `catalogConfig.resourceTypes` | Exactly one place resource type is required |
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

`OrganizationListItem`: `id`, `name`, `preset`, `subdomain`, `customDomain`, `owner`, `status`, `createdAt`.

### `GET /organizations` — list

Query, in addition to the common list parameters (`api-conventions.md`):
- `status`: `active` (default) or `deactivated`.
- `search`: case-insensitive substring of name, subdomain, custom domain, or owner e-mail.
- `sortBy`: `createdAt` only.

`200`: `{ "items": OrganizationListItem[], "meta": … }`.

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

The organization, its owner, and its copied modular configuration (`sections`, `catalogConfig`, `scheduleConfig`, `billingConfig` from the chosen preset) are created together in one transaction; the new organization is `active`. `201`: `Organization`. Screen text: «Organization created».

| Status | `code` | `field` | Fixed text in the canon |
|---|---|---|---|
| 400 | `VALIDATION_ERROR` (`REQUIRED`) | `name`, `subdomain`, `owner.email`, `presetId` | — (`presetId`: Select a preset) |
| 400 | `VALIDATION_ERROR` (`INVALID_FORMAT`) | `name` | Use 2–100 characters |
| 400 | `VALIDATION_ERROR` (`INVALID_FORMAT`) | `subdomain` | Use 3–20 lowercase letters, digits or hyphens, starting with a letter |
| 400 | `VALIDATION_ERROR` (`INVALID_FORMAT`) | `customDomain` | Enter a valid domain, for example yogaspace.com |
| 400 | `VALIDATION_ERROR` (`INVALID_FORMAT`) | `owner.email` | Enter a valid e-mail address |
| 400 | `VALIDATION_ERROR` (`INVALID_VALUE`) | `presetId` | Select a preset |
| 409 | `SUBDOMAIN_RESERVED` | `subdomain` | This subdomain is reserved |
| 409 | `SUBDOMAIN_TAKEN` | `subdomain` | This subdomain is already taken |
| 409 | `DOMAIN_TAKEN` | `customDomain` | This domain is already used by another organization |

### `GET /organizations/{id}` and `PATCH /organizations/{id}`

`PATCH /organizations/{id}` accepts any subset of `name`, `subdomain`, `customDomain`, `owner`, and `sections`. `presetId` cannot be changed.

| Status | `code` | `field` | Fixed text in the canon |
|---|---|---|---|
| 409 | `OWNER_IS_STAFF` | `owner.email` | This e-mail is already a staff member of this organization |

### `POST /organizations/{id}/deactivate`, `POST /organizations/{id}/activate`, `POST /organizations/{id}/impersonate`, and `POST /api/organization/auth/impersonation/exchange`

As defined in `overview.md` and `../../03-architecture/authentication.md`.

## Agent — acceptance checklist

- [ ] Each endpoint matches its path, method, body, response, and status codes
- [ ] Presets CRUD and duplicate (`/presets`) validate `sections`, `catalogConfig`, `scheduleConfig`, and `billingConfig` (exactly one `place` resource type and all three language labels `en/sk/uk`)
- [ ] `POST /organizations` requires `presetId` and copies the preset modular configuration into the organization
- [ ] `PATCH /organizations/{id}` allows updating `sections` and does not allow overwriting the preset

## Open questions

None.
