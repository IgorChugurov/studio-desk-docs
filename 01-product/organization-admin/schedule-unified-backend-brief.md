# Backend brief — Stage 3: Universal Schedule (`/schedule-slots`, `/week-templates`, `/weeks`)

**Derived from:**
- `01-product/profile.md`
- `01-product/organization-admin/schedule.md`
- `01-product/organization-admin/schedule-api-contract.md`
- `01-product/organization-admin/catalogs-api-contract.md`
- `01-product/organization-admin/api-contract.md`

## Mission

Complete the backend evolution by replacing the legacy `scheduled_class`, `scheduled_class_trainer`, and `/api/organization/classes` with the universal schedule engine:
1. Tables: `schedule_slot`, `schedule_slot_resource`, `week_template`, `week_template_row`, and `week_template_row_resource`.
2. Single unified PostgreSQL exclusion constraint on `schedule_slot_resource` (`EXCLUDE USING gist (resource_id WITH =, during WITH &&) WHERE (during IS NOT NULL AND enforce_no_overlap = true)`).
3. Endpoints `/api/organization/schedule-slots`, `/api/organization/week-templates`, and `/api/organization/weeks` supporting:
   - both slot kinds: `service` (work/service/class type + specialists/equipment) and `block` (blocked time with `note` and `blocksAll`);
   - duration `5..1440` minutes in steps of `5`;
   - single `place` auto-selection when the organization has only one `place` resource;
   - 6 statuses (`planned`, `confirmed`, `in_progress`, `completed` with `occupiesResources: true`; `cancelled`, `no_show` with `occupiesResources: false`);
   - conflict modes per resource type (`strict` → `409 RESOURCE_BUSY`; `warning` or overlap with an active `block` → `409 OVERLAP_WARNING` unless `confirmOverlap: true`; `none` → no conflict).

## IMPORTANT — DEVELOPMENT MODE (NO DATA MIGRATION NEEDED)

There is NO production data to preserve. Do NOT write `ALTER TABLE` data migrations from `scheduled_class`. Edit `0000_bumpy_nighthawk.sql` (and `src/database/schema/schedule.ts`) directly and recreate `studio_desk` and `studio_desk_test` cleanly.

## Deliverables

### 1. Database schema (`src/database/schema/schedule.ts` & `0000_bumpy_nighthawk.sql`) & Grants

Follow `01-product/organization-admin/schedule-api-contract.md`:
1. Drop `scheduled_class`, `scheduled_class_trainer`, `week_template_row_trainer`.
2. `week_template`: `id` (uuid, PK), `organization_id` (uuid, FK `organization.id`), `name` (text, unique per organization), `created_at`, `updated_at`.
3. `week_template_row`:
   - `id` (uuid, PK), `organization_id` (uuid, FK `organization.id`), `template_id` (uuid, FK `week_template.id` on delete cascade),
   - `kind` (`'service' | 'block'`), `weekday` (`'monday'..'sunday'`), `start_time` (time), `duration_minutes` (integer `5..1440` step 5, ending <= 1440 minutes of day),
   - `place_resource_id` (uuid, FK `resource.id`),
   - `audience` (`'group' | 'one_person'`, nullable), `format` (`'offline' | 'offline_and_online' | 'online_only'`, nullable), `places` (integer >= 1, nullable),
   - `note` (text, nullable), `blocks_all` (boolean, not null, default `false`),
   - `created_at`, `updated_at`.
4. `week_template_row_resource`:
   - `organization_id` (uuid, FK `organization.id`), `row_id` (uuid, FK `week_template_row.id` on delete cascade), `resource_id` (uuid, FK `resource.id`), `created_at`.
   - Primary key `(row_id, resource_id)`. Holds all selected non-place resources (`role = 'offering'` and `role = 'resource'`) for the row.
5. `schedule_slot`:
   - `id` (uuid, PK), `organization_id` (uuid, FK `organization.id`),
   - `kind` (`'service' | 'block'`), `place_resource_id` (uuid, FK `resource.id`),
   - `date` (date), `start_time` (time), `duration_minutes` (integer `5..1440` step 5, ending <= 1440 minutes of day),
   - `audience` (nullable), `format` (nullable), `places` (nullable),
   - `note` (text, nullable), `blocks_all` (boolean, not null, default `false`),
   - `status` (`'planned' | 'confirmed' | 'in_progress' | 'completed' | 'cancelled' | 'no_show'`, not null, default `'planned'`),
   - `occupies_resources` (boolean, not null, default `true`),
   - `starts_at` (timestamptz, not null), `ends_at` (timestamptz, not null),
   - `created_at`, `updated_at`.
6. `schedule_slot_resource`:
   - `organization_id` (uuid, FK `organization.id`), `slot_id` (uuid, FK `schedule_slot.id` on delete cascade), `resource_id` (uuid, FK `resource.id`),
   - `role` (`'place' | 'resource' | 'offering'`), `enforce_no_overlap` (boolean, not null), `during` (`tstzrange`, nullable), `created_at`.
   - Primary key `(slot_id, resource_id)`.
   - Every slot inserts a row for its `place_resource_id` (`role = 'place'`) plus a row for each id in its `resourceIds` (`role = 'resource' | 'offering'`), setting `enforce_no_overlap = (resource_type.conflict_mode = 'strict')` and `during = tstzrange(starts_at, ends_at, '[)')` when `occupies_resources = true` (or `NULL` when `occupies_resources = false`).
   - Exclusion constraint in SQL:
     `ALTER TABLE "schedule_slot_resource" ADD CONSTRAINT "schedule_slot_resource_no_overlap" EXCLUDE USING gist ("resource_id" WITH =, "during" WITH &&) WHERE ("during" IS NOT NULL AND "enforce_no_overlap" = true);`
7. Update `expected-grants.ts`:
   - `studio_api` has `SELECT, INSERT, UPDATE, DELETE` on `week_template`, `week_template_row`, `week_template_row_resource`, `schedule_slot`, `schedule_slot_resource`.

### 2. Schedule Slots API (`/api/organization/schedule-slots`)

Follow `01-product/organization-admin/schedule-api-contract.md`:
- `GET /schedule-slots`:
  - Query: `from`, `to` (`YYYY-MM-DD`, widened to Monday–Sunday weeks, span <= 366 days), optional `resourceId` (matches `place_resource_id = resourceId`, a row in `schedule_slot_resource`, or `blocks_all = true`).
  - `specialist` receives only slots where their linked `staff_resource.resource_id` participates (or `blocks_all = true`); asking for a different `resourceId` returns `403 FORBIDDEN`.
- `GET /schedule-slots/:id`: returns `ScheduleSlot` (`404 NOT_FOUND` if another organization or if `specialist` is not on the slot and `blocksAll = false`).
- `POST /schedule-slots`:
  - Supports `kind: 'service' | 'block'`.
  - If `placeResourceId` is omitted and the organization has exactly one resource of its `place` resource type, use that resource automatically; otherwise require `placeResourceId`.
  - For `kind = 'service'`:
    - Validate `resourceIds` against `catalogConfig.resourceTypes` (`required` and `multiple` rules for each `resource` and `offering` type).
    - Respect `scheduleConfig.audienceEnabled` (when `false`, default `audience = 'one_person'`, `places = null`) and `scheduleConfig.formatEnabled` (when `false`, default `format = 'offline'`, or `'online_only'` if virtual place).
  - For `kind = 'block'`:
    - `audience`, `format`, `places` are `null`; no `offering` resource allowed in `resourceIds`; `note` is optional trimmed text; `blocksAll` is boolean.
  - Conflict checking:
    1. Any overlap on an involved resource whose type has `conflictMode = 'strict'` (with an active slot `occupiesResources = true`) returns `409 RESOURCE_BUSY` (with `conflict: { resourceId, resourceName, start, end, title, slotId }`). `confirmOverlap` cannot bypass `strict`.
    2. If no `strict` conflict exists, check for overlaps on involved resources with `conflictMode = 'warning'` AND overlaps with any active `kind = 'block'` slot (sharing a resource or having `blocksAll = true`). Also if the new slot is itself `kind = 'block'` with `blocksAll = true`, check if any active slot exists in the organization in `[starts_at, ends_at)` (if a `strict` conflict is preferred or warning — per canon, `block` overlaps use `OVERLAP_WARNING` when confirmed with `confirmOverlap: true`, except `strict` resource exclusion on the same resource).
       - If `confirmOverlap` is omitted or `false`, return `409 OVERLAP_WARNING` with `conflicts: [...]`.
       - If `confirmOverlap` is `true`, save the slot (`201 ScheduleSlot`).
- `PATCH /schedule-slots/:id`:
  - Future active slot: updates `placeResourceId`, `resourceIds` (for `service`, `offering` resources in `resourceIds` cannot be changed via `PATCH` — return `400 VALIDATION_ERROR` `INVALID_VALUE` on `resourceIds`), `date`, `startTime`, `durationMinutes`, `audience`, `format`, `places`, `note`, `blocksAll`, `status` (must be in `scheduleConfig.statuses`), with `confirmOverlap`.
  - Past `service` slot: time and resources are frozen (`409 SLOT_ALREADY_STARTED` if changed), but `status` may be updated to `completed` or `no_show` (if enabled in `scheduleConfig.statuses`), updating `occupies_resources` and `schedule_slot_resource.during` accordingly.
  - Future inactive slot (`occupiesResources = false`): returns `409 SLOT_INACTIVE` (or `SLOT_CANCELLED`) unless restored via `/restore`.
- `POST /schedule-slots/:id/cancel`: sets `status = 'cancelled'`, `occupies_resources = false`, `during = null`.
- `POST /schedule-slots/:id/restore`: accepts optional `{ confirmOverlap?: boolean }`, sets `status = 'planned'`, `occupies_resources = true`, checking `RESOURCE_BUSY` and `OVERLAP_WARNING`.
- `DELETE /schedule-slots/:id`: `204` on future slot; `409 SLOT_ALREADY_STARTED` on past slot.
- `POST /schedule-slots/:id/replace`: cancels old `service` slot and creates new `service` slot (allowing new `offering` resource in `resourceIds` and optional `confirmOverlap`) in one transaction (`201`).

### 3. Week Templates (`/api/organization/week-templates`) & Week Actions (`/api/organization/weeks`)

- Update `/week-templates` rows to use `kind`, `placeResourceId`, `resourceIds`, `durationMinutes` (`5..1440`), `audience`, `format`, `places`, `note`, `blocksAll`. Check same-weekday clock-digit overlaps on `strict` resources (`409 RESOURCE_BUSY`).
- Update `/weeks/apply`, `/weeks/clear` (`placeResourceIds`), and `/weeks/replace` (`clear: 'template_places' | 'all_places'`) to work with `schedule_slot` and `schedule_slot_resource`.
- Ensure `PATCH /api/organization/resources/:id` time-zone change on a `place` resource recalculates `starts_at`/`ends_at`/`during` on `schedule_slot` and `schedule_slot_resource` and returns `409 RESOURCE_BUSY` if a `strict` resource overlaps.

## Required tests & Acceptance criteria

1. `pnpm lint`, `pnpm format:check`, `pnpm test`, and `pnpm test:e2e` all pass with 0 errors.
2. E2E tests cover:
   - `service` and `block` (including `blocksAll: true`) slots in `/schedule-slots` and `/week-templates`;
   - duration up to `1440` minutes;
   - single `place` auto-selection when `placeResourceId` is omitted;
   - all 6 statuses and `occupiesResources` (`true` vs `false`), including marking a past slot `completed` or `no_show`;
   - `strict` conflict (`409 RESOURCE_BUSY`) vs `warning` conflict and `block` overlap (`409 OVERLAP_WARNING` → `201` with `confirmOverlap: true`);
   - `/weeks/apply`, `/weeks/clear`, `/weeks/replace`;
   - `specialist` read-only access to their own slots and `blocksAll` slots.
3. Record implementation notes in `01-product/organization-admin/schedule-unified-backend-brief-feedback.md`.
