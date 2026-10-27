# API contract — Catalogs (Resources and Service types)

**Scope:** resources (places, specialists, equipment) and service types, and their files, in the organization admin API (`/api/organization`). A file address is on the API host, not under `/api/organization`.
**Derived from:** `catalogs.md`, `../../03-architecture/files.md`, `../../03-architecture/api-conventions.md`, `api-contract.md`

## Essay — what this is

This is the contract between the organization admin backend and its frontend for `Catalogs`. It names endpoints, payloads, and error codes. It does not restate product behaviour (`catalogs.md`), the common error and list formats (`api-conventions.md`), or sessions (`../../03-architecture/authentication.md`).

## Essay — general

- **Base URL:** `https://api.studio-desk.axondigital.xyz/api/organization`. Paths below are relative to it. A catalog file is a public static file at `/files/…` on the API host.
- **Authorization:** every endpoint requires `Authorization: Bearer <accessToken>` of an organization session. A missing, invalid, or revoked token, and a platform session, return `401 UNAUTHORIZED`.
- **Section:** `catalogs` (checked via the double filter `organization_section` ∩ `role_section`). The owner and an administrator pass. A `specialist` and an `accountant` receive `403 FORBIDDEN`.
- **Formats:** IDs are UUIDs. Dates are ISO 8601 strings in UTC. Text fields are trimmed. Errors and lists follow `api-conventions.md`. Empty optional text is stored as `null`, never as `""`.
- **A record of another organization** is `404 NOT_FOUND` on read, update, upload, delete, and reorder. The same answer is used when the id does not exist.

### Objects

`CatalogFile`:

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
- `url` is the public address of the file on the API host: `/files/resources/{resourceId}/{fileId}` or `/files/service-types/{serviceTypeId}/{fileId}` plus `.jpg`, `.png`, `.webp`, `.gif`, `.mp4`, or `.webm`.
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
  "description": null,
  "videoLink": null,
  "instagram": null,
  "tiktok": null,
  "serviceTypeIds": [],
  "images": [],
  "createdAt": "2026-10-08T18:00:00Z",
  "updatedAt": "2026-10-08T18:00:00Z"
}
```

- `resourceTypeId` references one of the organization's `resourceTypes` (`api-contract.md`).
- When the resource type has `geography: true` (`role = "place"`):
  - `type` is `ordinary` or `virtual`. A virtual place's `address` is always `Virtual place`. Changing `type` to `virtual` stores `Virtual place`; changing it to `ordinary` requires a new `address`.
  - `timeZone` is an IANA time-zone name from the same set as organization settings.
- When `geography: false`, `type`, `address`, and `timeZone` are `null`.
- When `showcase: true`, `description`, `videoLink` (YouTube/Vimeo URL), `instagram` (Instagram profile URL), and `tiktok` (TikTok profile URL) are accepted (`null` when empty), and `images` holds the resource's uploaded files sorted by `index`.
- When `skills: true`, `serviceTypeIds` holds the linked service types of this organization (the same relation as `resourceIds` on `ServiceType`).

`ServiceType`:

```json
{
  "id": "7c3d…",
  "name": "Yoga",
  "description": null,
  "defaultDuration": 60,
  "resourceIds": ["4a1b…"],
  "images": [],
  "createdAt": "2026-10-09T16:00:00Z",
  "updatedAt": "2026-10-09T16:00:00Z"
}
```

- `defaultDuration` is whole minutes, **from 5 to 1440 in steps of 5**.
- `resourceIds` is the array of linked resources of this organization (across any resource types that have `skills: true`). It is the same relation as `serviceTypeIds` on `Resource`.

## Essay — resources

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
  "name": "Main hall",
  "type": "ordinary",
  "address": "Hlavná 1, Bratislava",
  "timeZone": "Europe/Bratislava",
  "description": "",
  "videoLink": "",
  "instagram": "",
  "tiktok": "",
  "serviceTypeIds": []
}
```

- `resourceTypeId` and `name` are always required.
- If the resource type has `geography: true`: `type` defaults to `ordinary`; for `ordinary`, `address` is required; for `virtual`, `address` is stored as `Virtual place`; `timeZone` defaults to the organization's time zone when omitted or empty.
- Fields belonging to disabled cubes (`geography`, `showcase`, `skills`) are ignored or stored as `null` / `[]`.

`201`: `Resource` with `images: []`.

| Status | `code` | `field` | When |
|---|---|---|---|
| 400 | `VALIDATION_ERROR` (`REQUIRED`) | `resourceTypeId`, `name` | missing, `null`, or blank |
| 400 | `VALIDATION_ERROR` (`REQUIRED`) | `address` | `geography: true`, `ordinary` place, and `address` is missing or blank |
| 400 | `VALIDATION_ERROR` (`INVALID_VALUE`) | `resourceTypeId` | not a resource type of this organization |
| 400 | `VALIDATION_ERROR` (`INVALID_VALUE`) | `type` | not `ordinary` or `virtual` |
| 400 | `VALIDATION_ERROR` (`INVALID_VALUE`) | `timeZone` | not in the allowed time-zone set |
| 400 | `VALIDATION_ERROR` (`INVALID_FORMAT`) | `videoLink` | non-empty and not a YouTube or Vimeo URL |
| 400 | `VALIDATION_ERROR` (`INVALID_FORMAT`) | `instagram` | non-empty and not an Instagram URL |
| 400 | `VALIDATION_ERROR` (`INVALID_FORMAT`) | `tiktok` | non-empty and not a TikTok URL |
| 400 | `VALIDATION_ERROR` (`INVALID_VALUE`) | `serviceTypeIds` | an id is not a service type of this organization |

### `GET /resources/{id}` — one resource

`200`: `Resource`. `404 NOT_FOUND` when this organization has no such resource.

### `PATCH /resources/{id}` — update

Body: any subset of `name`, `type`, `address`, `timeZone`, `description`, `videoLink`, `instagram`, `tiktok`, `serviceTypeIds`. `resourceTypeId` cannot be changed.

Changing `timeZone` on a `place` resource recalculates the real instants (`starts_at`, `ends_at`) of its schedule slots without changing their local clock `date` or `startTime`. If that causes a `strict` conflict on another resource (`schedule-api-contract.md`), the update is refused with `409 RESOURCE_BUSY`.

`200`: `Resource`.

### Files of a resource

`POST /resources/{id}/files`, `DELETE /resources/{id}/files/{fileId}`, and `PUT /resources/{id}/files/order` work when the resource's type has `showcase: true`.
- Accepted types: `image/jpeg`, `image/png`, `image/webp`, `image/gif` (up to 10 MB); `video/mp4`, `video/webm` (up to 100 MB).
- Public URL: `/files/resources/{resourceId}/{fileId}.ext`.

## Essay — service types

### `GET /service-types` — list

Query: `search` (on `name`), `sortBy` (`createdAt` or `name`), `order`, pagination. `200`: `{ "items": ServiceType[], "meta": … }`.

### `POST /service-types` — create

Body:

```json
{
  "name": "Yoga",
  "description": "",
  "defaultDuration": 60,
  "resourceIds": []
}
```

`name` and `defaultDuration` (whole minutes from `5` to `1440` in steps of `5`) are required. Each id in `resourceIds` must belong to a resource of this organization whose resource type has `skills: true`.

`201`: `ServiceType` with `images: []`.

| Status | `code` | `field` | When |
|---|---|---|---|
| 400 | `VALIDATION_ERROR` (`REQUIRED`) | `name` | missing, `null`, or blank |
| 400 | `VALIDATION_ERROR` (`REQUIRED` or `INVALID_VALUE`) | `defaultDuration` | not a whole number from 5 to 1440 in steps of 5 |
| 400 | `VALIDATION_ERROR` (`INVALID_VALUE`) | `resourceIds` | an id is not a `skills`-enabled resource of this organization |

### `GET /service-types/{id}` and `PATCH /service-types/{id}`

Read (`200`: `ServiceType`) and partial update (`200`: `ServiceType`) of `name`, `description`, `defaultDuration`, `resourceIds`.

### Files of a service type

`POST /service-types/{id}/files`, `DELETE /service-types/{id}/files/{fileId}`, and `PUT /service-types/{id}/files/order` follow the same rules as resource files. Public URL: `/files/service-types/{serviceTypeId}/{fileId}.ext`.

## Essay — tables

`organization_resource_type`:
- `id` (uuid, PK), `organization_id` (uuid, FK `organization.id`), `sort_order` (integer), `role` (`'place' | 'resource'`), `labels` (jsonb `en/sk/uk`), `showcase` (boolean), `geography` (boolean), `person` (boolean), `skills` (boolean), `required` (boolean), `multiple` (boolean), `conflict_mode` (`'strict' | 'warning' | 'none'`), `created_at`, `updated_at`.

`resource`:
- `id` (uuid, PK), `organization_id` (uuid, FK `organization.id`), `resource_type_id` (uuid, FK `organization_resource_type.id`), `name` (text, not null), `type` (`'ordinary' | 'virtual'`, null when not `place`), `address` (text, null when not `place`), `time_zone` (text, null when not `place`), `description` (text, null), `video_link` (text, null), `instagram` (text, null), `tiktok` (text, null), `created_at`, `updated_at`.

`resource_file`:
- `id` (uuid, PK), `resource_id` (uuid, FK `resource.id`), `storage_path` (text), `content_type` (text), `index` (integer), `created_at`. Unique `(resource_id, index)`.

`service_type`:
- `id` (uuid, PK), `organization_id` (uuid, FK `organization.id`), `name` (text, not null), `description` (text, null), `default_duration` (integer, check `between 5 and 1440 and value % 5 = 0`), `created_at`, `updated_at`.

`service_type_file`:
- `id` (uuid, PK), `service_type_id` (uuid, FK `service_type.id`), `storage_path` (text), `content_type` (text), `index` (integer), `created_at`. Unique `(service_type_id, index)`.

`resource_service_type`:
- `organization_id` (uuid, FK `organization.id`), `resource_id` (uuid, FK `resource.id`), `service_type_id` (uuid, FK `service_type.id`), `created_at`. Primary key `(resource_id, service_type_id)`.

Grants: `organization_api` has `SELECT, INSERT, UPDATE` on `resource` and `service_type`, `SELECT, INSERT, UPDATE, DELETE` on `resource_file` and `service_type_file`, and `SELECT, INSERT, DELETE` on `resource_service_type`.

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

## Essay — how it connects

- Product behaviour: `catalogs.md`.
- Files: `../../03-architecture/files.md`.
- Errors, lists, OpenAPI: `../../03-architecture/api-conventions.md`.
- Sessions and organization configuration: `api-contract.md`.

## Agent — rules

- Every catalog endpoint appears in the OpenAPI document of the organization API.
- Create does not store files. `PATCH` of fields does not change file indexes.
- `defaultDuration` is validated between 5 and 1440 in steps of 5.
- Both sides of `resource_service_type` read and write the same link table.

## Agent — acceptance checklist

- [ ] `/resources` and `/service-types` match the paths, bodies, and status codes above
- [ ] A `specialist` and an `accountant` receive `403`; another organization's record receives `404`
- [ ] `place` resources validate `type` (`ordinary` / `virtual`), `address` (`Virtual place` for virtual), and `timeZone`
- [ ] `defaultDuration` accepts 5..1440 in steps of 5
- [ ] Bidirectional `resource_service_type` links work from both `/resources` and `/service-types`
- [ ] File upload, delete, and reorder preserve sequential `0..n-1` indexes

## Open questions

None.
