# API contract — Catalogs (Unified Resources)

**Scope:** unified catalog resources (places, specialists, equipment, and offerings/services/class types) and their files in the organization admin API (`/api/organization`). A file address is on the API host, not under `/api/organization`.
**Derived from:** `catalogs.md`, `../../03-architecture/files.md`, `../../03-architecture/api-conventions.md`, `api-contract.md`

## Essay — what this is

This is the contract between the organization admin backend and its frontend for `Catalogs`. Every tab in `Catalogs` — whether it represents places (`Halls`, `Bays`, `Rooms`), specialists/equipment (`Trainers`, `Masters`, `Mechanics`), or offerings (`Class types`, `Services`, `Work types`) — is backed by the single unified `Resource` entity and the `/resources` endpoints, driven by the organization's `catalogConfig.resourceTypes`.

## Essay — general

- **Base URL:** `https://api.studio-desk.axondigital.xyz/api/organization`. Paths below are relative to it. A catalog file is a public static file at `/files/resources/…` on the API host.
- **Authorization:** every endpoint requires `Authorization: Bearer <accessToken>` of an organization session. A missing, invalid, or revoked token, and a platform session, return `401 UNAUTHORIZED`.
- **Section:** `catalogs` (checked via the double filter `organization_section` ∩ `role_section`). The owner and an administrator pass. A `specialist` and an `accountant` receive `403 FORBIDDEN`.
- **Formats:** IDs are UUIDs. Dates are ISO 8601 strings in UTC. Text fields are trimmed. Errors and lists follow `api-conventions.md`. Empty optional text is stored as `null`, never as `""`.
- **A record of another organization** is `404 NOT_FOUND` on read, update, upload, delete, and reorder.

### Objects

`ResourceFile`:

```json
{
  "id": "a31c…",
  "index": 0,
  "kind": "image",
  "contentType": "image/jpeg",
  "url": "/files/resources/9c2e…/a31c….jpg"
}
```

- `kind` is `image` or `video`, taken from `contentType`.
- `url` is the public address of the file on the API host: `/files/resources/{resourceId}/{fileId}` plus `.jpg`, `.png`, `.webp`, `.gif`, `.mp4`, or `.webm`.
- `index` is an integer, zero or greater.

`Resource`:

```json
{
  "id": "9c2e…",
  "resourceTypeId": "rt1…",
  "name": "Main hall",
  "type": "ordinary",
  "address": "Hlavná 1, Bratislava",
  "timeZone": "Europe/Bratislava",
  "defaultDuration": null,
  "description": null,
  "videoLink": null,
  "instagram": null,
  "tiktok": null,
  "relatedResourceIds": [],
  "images": [],
  "createdAt": "2026-10-08T18:00:00Z",
  "updatedAt": "2026-10-08T18:00:00Z"
}
```

- `resourceTypeId` references one of the organization's `catalogConfig.resourceTypes` (`api-contract.md`).
- **Cube `geography` (`role = "place"`):**
  - `type` is `ordinary` or `virtual`. A virtual place's `address` is always `Virtual place`. Changing `type` to `virtual` stores `Virtual place`; changing it to `ordinary` requires a new `address`.
  - `timeZone` is an IANA time-zone name from the same set as organization settings.
  - When `geography: false`, `type`, `address`, and `timeZone` are `null`.
- **Cube `duration` (`role = "offering"`):**
  - `defaultDuration` is whole minutes, **from 5 to 1440 in steps of 5**.
  - When `duration: false`, `defaultDuration` is `null`.
- **Cube `showcase`:**
  - `description`, `videoLink` (YouTube/Vimeo URL), `instagram` (Instagram profile URL), and `tiktok` (TikTok profile URL) are accepted (`null` when empty), and `images` holds the resource's uploaded files sorted by `index`.
- **Cube `skills`:**
  - `relatedResourceIds` holds the array of linked resources of this organization across other resource types that have `skills: true` (symmetric relation stored in `resource_relation`).

## Essay — resources (`/resources`)

### `GET /resources` — list

Query, in addition to the common list parameters (`api-conventions.md`):

- `resourceTypeId`: optional UUID. Filters resources by a specific resource type of this organization.
- `search`: case-insensitive substring of `name`.
- `sortBy`: `createdAt` (default) or `name`.
- `order`: `DESC` (default) or `ASC`.

`200`: `{ "items": Resource[], "meta": … }`.

### `POST /resources` — create

Body:

```json
{
  "resourceTypeId": "rt1…",
  "name": "Yoga",
  "type": "ordinary",
  "address": "Hlavná 1, Bratislava",
  "timeZone": "Europe/Bratislava",
  "defaultDuration": 60,
  "description": "",
  "videoLink": "",
  "instagram": "",
  "tiktok": "",
  "relatedResourceIds": []
}
```

- `resourceTypeId` and `name` are always required.
- If the resource type has `geography: true` (`role = "place"`): `type` defaults to `ordinary`; for `ordinary`, `address` is required; for `virtual`, `address` is stored as `Virtual place`; `timeZone` defaults to the organization's time zone when omitted or empty.
- If the resource type has `duration: true` (`role = "offering"`): `defaultDuration` is required (whole minutes `5..1440` in steps of `5`).
- Fields belonging to disabled cubes on that `resourceType` are ignored and stored as `null` / `[]`.

`201`: `Resource` with `images: []`.

| Status | `code` | `field` | When |
|---|---|---|---|
| 400 | `VALIDATION_ERROR` (`REQUIRED`) | `resourceTypeId`, `name` | missing, `null`, or blank |
| 400 | `VALIDATION_ERROR` (`REQUIRED`) | `address` | `geography: true`, `ordinary` place, and `address` is missing or blank |
| 400 | `VALIDATION_ERROR` (`REQUIRED` or `INVALID_VALUE`) | `defaultDuration` | `duration: true`, and not a whole number from 5 to 1440 in steps of 5 |
| 400 | `VALIDATION_ERROR` (`INVALID_VALUE`) | `resourceTypeId` | not a resource type of this organization |
| 400 | `VALIDATION_ERROR` (`INVALID_VALUE`) | `type` | not `ordinary` or `virtual` |
| 400 | `VALIDATION_ERROR` (`INVALID_VALUE`) | `timeZone` | not in the allowed time-zone set |
| 400 | `VALIDATION_ERROR` (`INVALID_FORMAT`) | `videoLink` | non-empty and not a YouTube or Vimeo URL |
| 400 | `VALIDATION_ERROR` (`INVALID_FORMAT`) | `instagram` | non-empty and not an Instagram URL |
| 400 | `VALIDATION_ERROR` (`INVALID_FORMAT`) | `tiktok` | non-empty and not a TikTok URL |
| 400 | `VALIDATION_ERROR` (`INVALID_VALUE`) | `relatedResourceIds` | an id is not a `skills`-enabled resource of another resource type in this organization |

### `GET /resources/{id}` — one resource

`200`: `Resource`. `404 NOT_FOUND` when this organization has no such resource.

### `PATCH /resources/{id}` — update

Body: any subset of `name`, `type`, `address`, `timeZone`, `defaultDuration`, `description`, `videoLink`, `instagram`, `tiktok`, `relatedResourceIds`. `resourceTypeId` cannot be changed.

Changing `timeZone` on a `place` resource recalculates the real instants (`starts_at`, `ends_at`) of its schedule slots without changing their local clock `date` or `startTime`. If that causes a `strict` conflict on another resource (`schedule-api-contract.md`), the update is refused with `409 RESOURCE_BUSY`.

`200`: `Resource`.

### Files of a resource (`/resources/{id}/files`)

`POST /resources/{id}/files`, `DELETE /resources/{id}/files/{fileId}`, and `PUT /resources/{id}/files/order` work when the resource's type has `showcase: true`.
- Accepted types: `image/jpeg`, `image/png`, `image/webp`, `image/gif` (up to 10 MB); `video/mp4`, `video/webm` (up to 100 MB).
- Public URL: `/files/resources/{resourceId}/{fileId}.ext`.

## Essay — tables

`organization_resource_type`:
- `id` (uuid, PK), `organization_id` (uuid, FK `organization.id`), `sort_order` (integer), `role` (`'place' | 'resource' | 'offering'`), `labels` (jsonb `en/sk/uk`), `geography` (boolean), `duration` (boolean), `showcase` (boolean), `person` (boolean), `skills` (boolean), `required` (boolean), `multiple` (boolean), `conflict_mode` (`'strict' | 'warning' | 'none'`), `created_at`, `updated_at`.

`resource`:
- `id` (uuid, PK), `organization_id` (uuid, FK `organization.id`), `resource_type_id` (uuid, FK `organization_resource_type.id`), `name` (text, not null), `type` (`'ordinary' | 'virtual'`, null when not `geography`), `address` (text, null when not `geography`), `time_zone` (text, null when not `geography`), `default_duration` (integer, null when not `duration`, check `between 5 and 1440 and value % 5 = 0`), `description` (text, null), `video_link` (text, null), `instagram` (text, null), `tiktok` (text, null), `created_at`, `updated_at`.

`resource_file`:
- `id` (uuid, PK), `resource_id` (uuid, FK `resource.id`), `storage_path` (text), `content_type` (text), `index` (integer), `created_at`. Unique `(resource_id, index)`.

`resource_relation`:
- `organization_id` (uuid, FK `organization.id`), `resource_a_id` (uuid, FK `resource.id`), `resource_b_id` (uuid, FK `resource.id`), `created_at`. Primary key `(resource_a_id, resource_b_id)` with check `resource_a_id < resource_b_id` so every pair is stored once and queried symmetrically from either side.

Grants: `organization_api` has `SELECT, INSERT, UPDATE` on `resource`, `SELECT, INSERT, UPDATE, DELETE` on `resource_file`, and `SELECT, INSERT, DELETE` on `resource_relation`.

## Essay — error codes and screen texts

| `code` | Text |
|---|---|
| `VALIDATION_ERROR` (`REQUIRED`) on `name` | Enter a name |
| `VALIDATION_ERROR` (`REQUIRED`) on `address` | Enter an address |
| `VALIDATION_ERROR` (`REQUIRED` or `INVALID_VALUE`) on `defaultDuration` | Enter a duration from 5 to 1440 minutes in steps of 5 |
| `VALIDATION_ERROR` (`INVALID_FORMAT`) on `videoLink` | Enter a YouTube or Vimeo link |
| `VALIDATION_ERROR` (`INVALID_FORMAT`) on `instagram` | Enter an Instagram link |
| `VALIDATION_ERROR` (`INVALID_FORMAT`) on `tiktok` | Enter a TikTok link |
| `FORBIDDEN` on this section | You don't have access to this page |
| any other error of an action or of loading | Something went wrong. Try again |

## Agent — acceptance checklist

- [ ] `/resources` handles all catalog tabs (`place`, `resource`, and `offering`) according to the cubes of `resourceTypeId`
- [ ] `place` resources validate `type` (`ordinary` / `virtual`), `address` (`Virtual place` for virtual), and `timeZone`
- [ ] `offering` resources (`duration: true`) validate `defaultDuration` in `5..1440` in steps of 5
- [ ] Symmetric `resource_relation` links work from either side for `skills`-enabled resource types
- [ ] File upload, delete, and reorder preserve sequential `0..n-1` indexes at `/files/resources/{resourceId}/{fileId}.ext`

## Open questions

None.
