# API contract — Schedule

**Scope:** week templates, schedule slots (`service` and `block`), statuses, conflict modes (`strict`, `warning`, `none`), and the three week actions in the organization admin API (`/api/organization`). A place time-zone change that would violate a `strict` resource conflict is refused here too.
**Derived from:** `schedule.md`, `catalogs.md`, `catalogs-api-contract.md`, `api-contract.md`, `../../03-architecture/api-conventions.md`

## Essay — what this is

This is the contract between the organization admin backend and its frontend for `Schedule`. Every catalog item involved in a slot — the location (`role = "place"`), the specialists/equipment (`role = "resource"`), and the performed work/service/class type (`role = "offering"`) — is a `Resource` from `catalogs-api-contract.md`.

## Essay — general

- **Base URL:** `https://api.studio-desk.axondigital.xyz/api/organization`. Paths below are relative to it.
- **Authorization:** every endpoint requires `Authorization: Bearer <accessToken>` of an organization session. A missing, invalid, or revoked token, and a platform session, return `401 UNAUTHORIZED`.
- **Section:** `schedule` (checked via `organization_section` ∩ `role_section`). The owner and an administrator can do everything in this contract. A `specialist` can read slots on which their linked catalog resource (`staff_resource`) is one of the resources (plus organization-wide `blocksAll` blocks), and nothing else. An `accountant` receives `403 FORBIDDEN`.
- **Another organization** is `404 NOT_FOUND` on read, update, and delete. A `specialist` who asks for a slot they are not on also receives `404 NOT_FOUND`.
- **Formats:** IDs are UUIDs. `createdAt` and `updatedAt` are ISO 8601 strings in UTC. A slot `date` and a week date are `YYYY-MM-DD` by the place resource's clock, not a UTC instant. A `startTime` is `HH:MM`, 24-hour. Duration is `5..1440` minutes in steps of `5`, ending at or before `24:00` of the place's clock.
- **No view time zone.** The client converts the grid into the place's clock before sending a time, and back after reading one.

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
  "resourceIds": ["7c3d…", "4a1b…"],
  "date": "2026-10-20",
  "startTime": "18:00",
  "durationMinutes": 60,
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

- `placeResourceId` is the id of the resource whose `resourceType` has `role = "place"`. If the organization has only one `place` resource and `placeResourceId` is omitted on create, the backend uses that single place resource automatically.
- `resourceIds` holds all other selected catalog resources for this slot across the organization's `catalogConfig.resourceTypes` (both `role = "offering"` resources such as the class type / service, and `role = "resource"` items such as trainers, mechanics, or equipment).
- For `kind = "service"`:
  - `resourceIds` is validated against `required` and `multiple` of each `resourceType` in `catalogConfig.resourceTypes`.
  - `note` is `null`; `blocksAll` is `false`.
  - If `scheduleConfig.audienceEnabled` is `false`, `audience` defaults to `one_person` and `places` is `null`.
  - If `scheduleConfig.formatEnabled` is `false`, `format` defaults to `offline` (or `online_only` in a virtual place).
  - In a virtual place, `format` is always `online_only` and `places` is `null`.
- For `kind = "block"`:
  - `audience`, `format`, and `places` are `null`; no `offering` resource is included in `resourceIds`.
  - `note` is optional trimmed text (`null` when blank).
  - `blocksAll` is boolean (`false` by default).

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

- `GET /week-templates`, `POST /week-templates`, `GET /week-templates/{id}`, `PATCH /week-templates/{id}`, `DELETE /week-templates/{id}`.
- `POST /week-templates/{id}/rows`, `PATCH /week-templates/{id}/rows/{rowId}`, `DELETE /week-templates/{id}/rows/{rowId}`.
  - Supports both `service` and `block` rows (`durationMinutes` `5..1440` in steps of `5`).
  - Strict overlaps (`conflictMode = "strict"`) on the same `weekday` by clock digits return `409 RESOURCE_BUSY`.

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
  "resourceIds": ["7c3d…", "4a1b…"],
  "date": "2026-10-20",
  "startTime": "18:00",
  "durationMinutes": 60,
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
  1. If any involved resource (`placeResourceId` or in `resourceIds`) belongs to a resource type with `conflictMode = "strict"` and is already occupied by an active slot (`occupiesResources = true`) in `[starts_at, ends_at)`, the request is refused with `409 RESOURCE_BUSY` (enforced by PostgreSQL `EXCLUDE USING gist`). `confirmOverlap` cannot bypass a `strict` conflict.
  2. If there is no `strict` conflict, but an involved resource has `conflictMode = "warning"` and overlaps an active slot, OR the new slot overlaps an active `block` slot (on a shared resource or with `blocksAll = true`):
     - when `confirmOverlap` is omitted or `false`, nothing is written and `409 OVERLAP_WARNING` is returned with the list of conflicting slots;
     - when `confirmOverlap` is `true`, the slot is saved (`201 ScheduleSlot`).
  3. Resources whose type has `conflictMode = "none"` (including all `role = "offering"` resources) never cause a conflict.

### `PATCH /schedule-slots/{id}` — update

Body: any subset of `placeResourceId`, `resourceIds`, `date`, `startTime`, `durationMinutes`, `audience`, `format`, `places`, `note`, `blocksAll`, `status`, `confirmOverlap`.
- For a future active `service` slot: `offering` resources inside `resourceIds` cannot be changed via `PATCH` (use `POST /schedule-slots/{id}/replace` to change the offering).
- For a past `service` slot: time and resources are frozen, but `status` may be updated to `completed` or `no_show` (if enabled in `scheduleConfig.statuses`).

`200`: `ScheduleSlot`.

### `POST /schedule-slots/{id}/cancel`, `POST /schedule-slots/{id}/restore`, `DELETE /schedule-slots/{id}`, `POST /schedule-slots/{id}/replace`

- `cancel`: sets `status = "cancelled"`, `occupiesResources = false`. `200`: `ScheduleSlot`.
- `restore`: accepts optional `{ "confirmOverlap": boolean }`, sets `status = "planned"`, `occupiesResources = true`. `200`: `ScheduleSlot`.
- `delete`: `204` on a future slot; `409 SLOT_ALREADY_STARTED` on a past slot.
- `replace`: cancels the old slot (`status = "cancelled"`, `occupiesResources = false`) and creates the new `service` slot (with new `offering` resource in `resourceIds`) in one transaction. `201`: the new `ScheduleSlot`.

## Essay — week actions (`/weeks/apply`, `/weeks/clear`, `/weeks/replace`)

The owner and an administrator only. Work in one transaction on not-yet-started slots from the current week onwards, supporting both `service` and `block` template rows:
- `POST /weeks/apply`: `{ "templateId", "from", "to", "choices" }`
- `POST /weeks/clear`: `{ "from", "to", "placeResourceIds", "preview" }`
- `POST /weeks/replace`: `{ "templateId", "from", "to", "clear": "template_places" | "all_places", "preview", "choices" }`

## Essay — tables

`week_template`: `id`, `organization_id`, `name`, `created_at`, `updated_at`.

`week_template_row`: `id`, `organization_id`, `template_id`, `kind` (`'service' | 'block'`), `weekday`, `start_time`, `duration_minutes` (`5..1440`), `place_resource_id` (FK `resource.id`), `audience` (nullable), `format` (nullable), `places` (nullable), `note` (nullable), `blocks_all` (boolean), `created_at`, `updated_at`.

`week_template_row_resource`: `organization_id`, `row_id`, `resource_id`, `created_at`. Primary key `(row_id, resource_id)`. Holds all other resources (`offering` and `resource`) of the template row.

`schedule_slot`:
- `id` (uuid, PK), `organization_id` (uuid, FK `organization.id`), `kind` (`'service' | 'block'`), `place_resource_id` (uuid, FK `resource.id`), `date` (date), `start_time` (time), `duration_minutes` (integer `5..1440`), `audience` (nullable text), `format` (nullable text), `places` (nullable integer), `note` (nullable text), `blocks_all` (boolean), `status` (`'planned' | 'confirmed' | 'in_progress' | 'completed' | 'cancelled' | 'no_show'`), `occupies_resources` (boolean), `starts_at` (timestamptz), `ends_at` (timestamptz), `created_at`, `updated_at`.

`schedule_slot_resource`:
- `organization_id` (uuid, FK `organization.id`), `slot_id` (uuid, FK `schedule_slot.id`, on delete cascade), `resource_id` (uuid, FK `resource.id`), `role` (`'place' | 'resource' | 'offering'`), `enforce_no_overlap` (boolean), `during` (`tstzrange` — `[starts_at, ends_at)` when `occupies_resources = true`, `NULL` when `occupies_resources = false`), `created_at`.
- Primary key `(slot_id, resource_id)`.
- Every slot writes its `place_resource_id` and each of its `resourceIds` into `schedule_slot_resource` with `enforce_no_overlap = (resource_type.conflict_mode = 'strict')`. (For `role = 'offering'`, `conflict_mode` is always `'none'`, so `enforce_no_overlap = false`).
- **Single unified database exclusion constraint:**
  `EXCLUDE USING gist (resource_id WITH =, during WITH &&) WHERE (during IS NOT NULL AND enforce_no_overlap = true)`
  This single constraint guarantees at the database level that no resource with `conflictMode = 'strict'` can ever be double-booked by concurrent requests, while allowing parallel offerings (`conflictMode = 'none'`) and confirmed overlaps (`confirmOverlap = true`) for resources with `conflictMode = 'warning'`.

Grants: `organization_api` has `SELECT, INSERT, UPDATE, DELETE` on `week_template`, `week_template_row`, `week_template_row_resource`, `schedule_slot`, and `schedule_slot_resource`.

## Agent — acceptance checklist

- [ ] Unified `Resource` items (`place`, `resource`, and `offering`) are linked to slots and template rows via `placeResourceId` and `resourceIds`
- [ ] Both `service` and `block` slots (including `blocksAll`) work in `/schedule-slots` and `/week-templates`
- [ ] Duration supports `5..1440` minutes in steps of 5, ending by `24:00` local place time
- [ ] Single place auto-selection works when the organization has only one `place` resource
- [ ] Statuses update `occupiesResources` (`true` for `planned`, `confirmed`, `in_progress`, `completed`; `false` for `cancelled`, `no_show`)
- [ ] `strict` conflicts are rejected by the DB exclusion constraint (`409 RESOURCE_BUSY`)
- [ ] `warning` conflicts and `block` overlaps return `409 OVERLAP_WARNING` and succeed when `confirmOverlap: true` is sent

## Open questions

None.
