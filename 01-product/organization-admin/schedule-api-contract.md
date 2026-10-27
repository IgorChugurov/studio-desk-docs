# API contract — Schedule

**Scope:** week templates, schedule slots (`service` and `block`), statuses, conflict modes (`strict`, `warning`, `none`), and the three week actions in the organization admin API (`/api/organization`). A place time-zone change that would violate a `strict` resource conflict is refused here too.
**Derived from:** `schedule.md`, `catalogs.md`, `catalogs-api-contract.md`, `api-contract.md`, `../../03-architecture/api-conventions.md`

## Essay — what this is

This is the contract between the organization admin backend and its frontend for `Schedule`. It names endpoints, payloads, and error codes. It does not restate product behaviour (`schedule.md`) or the common error and list formats (`api-conventions.md`).

A staff member with the role `specialist` is linked to one catalog resource (`staff_resource`, `api-contract.md`). That link is how this API knows which slots are theirs.

## Essay — general

- **Base URL:** `https://api.studio-desk.axondigital.xyz/api/organization`. Paths below are relative to it.
- **Authorization:** every endpoint requires `Authorization: Bearer <accessToken>` of an organization session. A missing, invalid, or revoked token, and a platform session, return `401 UNAUTHORIZED`.
- **Section:** `schedule` (checked via `organization_section` ∩ `role_section`). The owner and an administrator can do everything in this contract. A `specialist` can read slots on which their linked catalog resource is one of the resources (plus organization-wide `blocksAll` blocks), and nothing else: no templates, no week actions, no create, update, cancel, restore, delete, or replace. An `accountant` receives `403 FORBIDDEN`.
- **Another organization** is `404 NOT_FOUND` on read, update, and delete. A `specialist` who asks for a slot they are not on also receives `404 NOT_FOUND`.
- **Formats:** IDs are UUIDs. `createdAt` and `updatedAt` are ISO 8601 strings in UTC. A slot `date` and a week date are `YYYY-MM-DD` by the place resource's clock, not a UTC instant. A `startTime` is `HH:MM`, 24-hour. Duration is `5..1440` minutes in steps of `5`, ending at or before `24:00` of the place's clock.
- **No view time zone.** The client converts the grid into the place's clock before sending a time, and back after reading one.

A week is Monday–Sunday. `from` and `to` are widened to the Monday–Sunday weeks that contain them. The current week is the week that contains today in the organization's time zone. The slot list range may not be longer than 366 days.

### Objects

`SlotKind`: `service` or `block`.

`SlotStatus`:
- occupies resources (`occupiesResources: true`): `planned`, `confirmed`, `in_progress`, `completed`;
- frees resources (`occupiesResources: false`): `cancelled`, `no_show`.

`Audience`: `group` or `one_person`. `Format`: `offline`, `offline_and_online`, or `online_only`. `Weekday`: `monday`..`sunday`.

`ScheduleSlot`:

```json
{
  "id": "c1…",
  "kind": "service",
  "placeResourceId": "9c2e…",
  "resourceIds": ["4a1b…"],
  "date": "2026-10-20",
  "startTime": "18:00",
  "durationMinutes": 60,
  "serviceTypeId": "7c3d…",
  "audience": "group",
  "format": "offline",
  "places": 12,
  "note": null,
  "blocksAll": false,
  "status": "planned",
  "occupiesResources": true,
  "createdAt": "2026-10-16T18:00:00Z",
  "updatedAt": "2026-10-16T18:00:00Z"
}
```

- `placeResourceId` is the id of a resource whose resource type has `role = "place"`. If the organization has only one `place` resource and `placeResourceId` is omitted on create, the backend uses that single place resource automatically.
- `resourceIds` holds the selected resources of the organization's other `resourceTypes` (`role = "resource"`).
- For `kind = "service"`:
  - `serviceTypeId` is required; `note` is `null`; `blocksAll` is `false`.
  - If `serviceConfig.audienceEnabled` is `false`, `audience` defaults to `one_person` and `places` is `null`.
  - If `serviceConfig.formatEnabled` is `false`, `format` defaults to `offline` (or `online_only` in a virtual place).
  - In a virtual place, `format` is always `online_only` and `places` is `null`.
- For `kind = "block"`:
  - `serviceTypeId`, `audience`, `format`, and `places` are `null`.
  - `note` is optional trimmed text (`null` when blank).
  - `blocksAll` is boolean (`false` by default). When `blocksAll` is `false`, at least one resource (`placeResourceId` or in `resourceIds`) is blocked.

`TemplateRow` has the same fields as `ScheduleSlot` except `weekday` instead of `date`, and no `status` or `occupiesResources`.

`Template`:

```json
{
  "id": "t1…",
  "name": "Morning week",
  "rows": [],
  "createdAt": "2026-10-16T18:00:00Z",
  "updatedAt": "2026-10-16T18:00:00Z"
}
```

## Essay — templates (`/week-templates`)

The owner and an administrator only (`403 FORBIDDEN` for `specialist` and `accountant`).

- `GET /week-templates`: list with `search`, `sortBy` (`createdAt` or `name`), `order`.
- `POST /week-templates`: `{ "name": "Morning week" }` → `201 Template` (`409 TEMPLATE_NAME_TAKEN` if duplicate in the organization).
- `GET /week-templates/{id}`: `200 Template`.
- `PATCH /week-templates/{id}`: `{ "name": "Evening week" }` → `200 Template`.
- `DELETE /week-templates/{id}`: `204`.
- `POST /week-templates/{id}/rows`: add a `service` or `block` row (`durationMinutes` `5..1440` in steps of `5`). Strict overlaps (`conflictMode = "strict"`) on the same `weekday` by clock digits return `409 RESOURCE_BUSY`.
- `PATCH /week-templates/{id}/rows/{rowId}` and `DELETE /week-templates/{id}/rows/{rowId}`.

## Essay — schedule slots (`/schedule-slots`)

### `GET /schedule-slots` — slots in a range

Query:
- `from`, `to`: required `YYYY-MM-DD` (widened to Monday–Sunday weeks, span <= 366 days).
- `resourceId`: optional UUID (filters slots where `placeResourceId = resourceId`, `resourceIds` contains `resourceId`, or `blocksAll = true`).

`200`: `{ "items": ScheduleSlot[] }`.

### `GET /schedule-slots/{id}` — one slot

`200`: `ScheduleSlot`.

### `POST /schedule-slots` — create

Body:

```json
{
  "kind": "service",
  "placeResourceId": "9c2e…",
  "resourceIds": ["4a1b…"],
  "date": "2026-10-20",
  "startTime": "18:00",
  "durationMinutes": 60,
  "serviceTypeId": "7c3d…",
  "audience": "group",
  "format": "offline",
  "places": 12,
  "note": null,
  "blocksAll": false,
  "confirmOverlap": false
}
```

- Start must be strictly in the future (`409 SLOT_ALREADY_STARTED` otherwise).
- Created with `status: "planned"` and `occupiesResources: true`.
- **Conflict checking:**
  1. If any involved resource has `conflictMode = "strict"` and is already occupied by an active slot (`occupiesResources = true`) in `[starts_at, ends_at)`, the request is refused with `409 RESOURCE_BUSY` (enforced by PostgreSQL `EXCLUDE USING gist`). `confirmOverlap` cannot bypass a `strict` conflict.
  2. If there is no `strict` conflict, but an involved resource has `conflictMode = "warning"` and overlaps an active slot, OR the new slot overlaps an active `block` slot (on a shared resource or with `blocksAll = true`):
     - when `confirmOverlap` is omitted or `false`, nothing is written and `409 OVERLAP_WARNING` is returned with the list of conflicting slots;
     - when `confirmOverlap` is `true`, the slot is saved (`201 ScheduleSlot`).

| Status | `code` | `field` | Text / behaviour |
|---|---|---|---|
| 400 | `VALIDATION_ERROR` (`REQUIRED`) | `date`, required resource types, `serviceTypeId` (for `service`) | Select a date / Select a {singular} |
| 400 | `VALIDATION_ERROR` (`INVALID_VALUE`) | `startTime` | Enter a time in steps of 5 minutes / A slot must end by midnight |
| 400 | `VALIDATION_ERROR` (`REQUIRED` or `INVALID_VALUE`) | `durationMinutes` | Enter a duration from 5 to 1440 minutes in steps of 5 |
| 409 | `SLOT_ALREADY_STARTED` | — | slot start is in the past |
| 409 | `RESOURCE_BUSY` | — | strict conflict on a resource (`conflict` payload includes `resourceId`, `resourceName`, `start`, `end`, `title`) |
| 409 | `OVERLAP_WARNING` | — | warning conflict or overlap with a `block` (`conflicts` array returned; user can confirm with `confirmOverlap: true`) |

### `PATCH /schedule-slots/{id}` — update

Body: any subset of `placeResourceId`, `resourceIds`, `date`, `startTime`, `durationMinutes`, `audience`, `format`, `places`, `note`, `blocksAll`, `status`, `confirmOverlap`.
- For a future active slot: time, resources, details, and `status` (among statuses enabled in `serviceConfig.statuses`) can be updated. `serviceTypeId` and `kind` are not changed here (use `replace` to change `serviceTypeId`).
- For a past `service` slot: time and resources are frozen, but `status` may be updated to `completed` or `no_show` (if enabled in `serviceConfig.statuses`).

`200`: `ScheduleSlot`.

### `POST /schedule-slots/{id}/cancel` and `POST /schedule-slots/{id}/restore`

- `cancel`: sets `status = "cancelled"`, `occupiesResources = false`. `200`: `ScheduleSlot`.
- `restore`: accepts optional `{ "confirmOverlap": boolean }`, sets `status = "planned"`, `occupiesResources = true` (checking `RESOURCE_BUSY` and `OVERLAP_WARNING`). `200`: `ScheduleSlot`.

### `DELETE /schedule-slots/{id}`

`204`. Deletes a future slot. A past slot returns `409 SLOT_ALREADY_STARTED`.

### `POST /schedule-slots/{id}/replace`

Body: the create body for a `service` slot, including the new `serviceTypeId` (and optional `confirmOverlap`). In one transaction the old slot is set to `status = "cancelled"` (`occupiesResources = false`) and the new slot is created. `201`: the new `ScheduleSlot`.

## Essay — week actions (`/weeks/apply`, `/weeks/clear`, `/weeks/replace`)

The owner and an administrator only. Work in one transaction on not-yet-started slots from the current week onwards, supporting both `service` and `block` template rows:
- `POST /weeks/apply`: `{ "templateId", "from", "to", "choices" }`
- `POST /weeks/clear`: `{ "from", "to", "placeResourceIds", "preview" }`
- `POST /weeks/replace`: `{ "templateId", "from", "to", "clear": "template_places" | "all_places", "preview", "choices" }`

## Essay — tables

`week_template`: `id`, `organization_id`, `name`, `created_at`, `updated_at`.

`week_template_row`: `id`, `organization_id`, `template_id`, `kind` (`'service' | 'block'`), `weekday`, `start_time`, `duration_minutes` (`5..1440`), `place_resource_id` (FK `resource.id`), `service_type_id` (nullable FK `service_type.id`), `audience` (nullable), `format` (nullable), `places` (nullable), `note` (nullable), `blocks_all` (boolean), `created_at`, `updated_at`.

`week_template_row_resource`: `organization_id`, `row_id`, `resource_id`, `created_at`. Primary key `(row_id, resource_id)`.

`schedule_slot`:
- `id` (uuid, PK), `organization_id` (uuid, FK `organization.id`), `kind` (`'service' | 'block'`), `place_resource_id` (uuid, FK `resource.id`), `service_type_id` (nullable uuid, FK `service_type.id`), `date` (date), `start_time` (time), `duration_minutes` (integer `5..1440`), `audience` (nullable text), `format` (nullable text), `places` (nullable integer), `note` (nullable text), `blocks_all` (boolean), `status` (`'planned' | 'confirmed' | 'in_progress' | 'completed' | 'cancelled' | 'no_show'`), `occupies_resources` (boolean), `starts_at` (timestamptz), `ends_at` (timestamptz), `created_at`, `updated_at`.

`schedule_slot_resource`:
- `organization_id` (uuid, FK `organization.id`), `slot_id` (uuid, FK `schedule_slot.id`, on delete cascade), `resource_id` (uuid, FK `resource.id`), `role` (`'place' | 'resource'`), `enforce_no_overlap` (boolean), `during` (`tstzrange` — `[starts_at, ends_at)` when `occupies_resources = true`, `NULL` when `occupies_resources = false`), `created_at`.
- Primary key `(slot_id, resource_id)`.
- Every slot writes its `place_resource_id` and each of its `resourceIds` into `schedule_slot_resource` with `enforce_no_overlap = (resource_type.conflict_mode = 'strict')`.
- **Single unified database exclusion constraint:**
  `EXCLUDE USING gist (resource_id WITH =, during WITH &&) WHERE (during IS NOT NULL AND enforce_no_overlap = true)`
  This single constraint guarantees at the database level that no resource with `conflictMode = 'strict'` can ever be double-booked by concurrent requests, while allowing confirmed overlaps (`confirmOverlap = true`) for resources with `conflictMode = 'warning'` or `none`.

Grants: `organization_api` has `SELECT, INSERT, UPDATE, DELETE` on `week_template`, `week_template_row`, `week_template_row_resource`, `schedule_slot`, and `schedule_slot_resource`.

## Agent — acceptance checklist

- [ ] Both `service` and `block` slots (including `blocksAll`) work in `/schedule-slots` and `/week-templates`
- [ ] Duration supports `5..1440` minutes in steps of 5, ending by `24:00` local place time
- [ ] Single place auto-selection works when the organization has only one `place` resource
- [ ] Statuses update `occupiesResources` (`true` for `planned`, `confirmed`, `in_progress`, `completed`; `false` for `cancelled`, `no_show`)
- [ ] `strict` conflicts are rejected by the DB exclusion constraint (`409 RESOURCE_BUSY`)
- [ ] `warning` conflicts and `block` overlaps return `409 OVERLAP_WARNING` and succeed when `confirmOverlap: true` is sent
- [ ] `specialist` can only read slots where their linked resource participates (or `blocksAll = true`) and receives `403` on writes

## Open questions

None.
