# Backend brief — Stage 2: Unified Catalogs (`/resources`) and `staff_resource`

**Derived from:**
- `01-product/profile.md`
- `01-product/organization-admin/overview.md`
- `01-product/organization-admin/api-contract.md`
- `01-product/organization-admin/catalogs.md`
- `01-product/organization-admin/catalogs-api-contract.md`
- `03-architecture/files.md`

## Mission

Replace the legacy split catalog tables (`hall`, `hall_file`, `trainer`, `trainer_file`, `class_type`, `class_type_file`, `trainer_class_type`, `staff_trainer`) and legacy endpoints (`/halls`, `/trainers`, `/class-types`) with the single unified `Resource` model:
1. Tables: `resource`, `resource_file`, `resource_relation`, and `staff_resource`.
2. Endpoints: `/api/organization/resources` (`GET`, `POST`, `GET /:id`, `PATCH /:id`, `POST /:id/files`, `DELETE /:id/files/:fileId`, `PUT /:id/files/order`), serving every catalog tab (`role: 'place' | 'resource' | 'offering'`) according to the cubes enabled on its `organization_resource_type`.
3. Update `/api/organization/staff` so `role: 'specialist'` uses `resourceId` (referencing a `resource` whose `resourceType` has `person: true`, stored in `staff_resource`) and returns `409 SPECIALIST_ALREADY_LINKED` / `Select a specialist`.

## IMPORTANT — DEVELOPMENT MODE (NO DATA MIGRATION NEEDED)

There is NO production data to preserve. Do NOT write `ALTER TABLE` or data migration scripts from `hall`/`trainer`/`class_type`. Update `0000_bumpy_nighthawk.sql` (or recreate `studio_desk` and `studio_desk_test` cleanly from the updated schema) so the database directly has the unified `resource` tables.
For the schedule tables (`scheduled_class`, `week_template_row`, etc.) that Stage 3 will replace next: either point their foreign keys at `resource.id` temporarily so existing schedule tests compile, or keep them minimal until Stage 3 rewrites `schedule_slot`.

## Deliverables

### 1. Database schema (`src/database/schema/`) & Grants (`expected-grants.ts`)

Follow `01-product/organization-admin/catalogs-api-contract.md` and `api-contract.md`:
1. Drop legacy `hall`, `hall_file`, `trainer`, `trainer_file`, `class_type`, `class_type_file`, `trainer_class_type`, `staff_trainer`.
2. Create `resource`:
   - `id` (uuid, PK), `organization_id` (uuid, FK `organization.id`), `resource_type_id` (uuid, FK `organization_resource_type.id`), `name` (text, not null, trimmed non-empty),
   - `type` (`'ordinary' | 'virtual'`, nullable — non-null when `geography = true`), `address` (text, nullable — `'Virtual place'` when `type = 'virtual'`), `time_zone` (text, nullable),
   - `default_duration` (integer, nullable — `between 5 and 1440 and % 5 = 0` when `duration = true`),
   - `description` (text, nullable), `video_link` (text, nullable), `instagram` (text, nullable), `tiktok` (text, nullable),
   - `created_at`, `updated_at`.
3. Create `resource_file`:
   - `id` (uuid, PK), `resource_id` (uuid, FK `resource.id` on delete cascade), `storage_path` (text, `/files/resources/{resourceId}/{fileId}.ext`), `content_type` (text), `index` (integer >= 0), `created_at`. Unique `(resource_id, index)`.
4. Create `resource_relation`:
   - `organization_id` (uuid, FK `organization.id`), `resource_a_id` (uuid, FK `resource.id`), `resource_b_id` (uuid, FK `resource.id`), `created_at`.
   - Primary key `(resource_a_id, resource_b_id)` with check `resource_a_id < resource_b_id` (symmetric many-to-many relation between resources of different `skills`-enabled resource types in the same organization).
5. Create `staff_resource`:
   - `organization_id` (uuid, FK `organization.id`), `staff_id` (uuid, PK, FK `organization_staff.id` on delete cascade), `resource_id` (uuid, unique, FK `resource.id` on delete restrict), `created_at`.
6. Update `RESOURCE_TYPE_IN_USE` check in `PATCH /api/organization/settings`:
   - Refuses deleting an `organization_resource_type` that has rows in `resource`, or disabling `person` on a resource type that has a resource linked in `staff_resource`.
7. Update `expected-grants.ts`:
   - `studio_api` has `SELECT, INSERT, UPDATE` on `resource`, `SELECT, INSERT, UPDATE, DELETE` on `resource_file`, `SELECT, INSERT, DELETE` on `resource_relation`, and `SELECT, INSERT, UPDATE, DELETE` on `staff_resource`.

### 2. Unified Catalogs API (`/api/organization/resources`)

Follow `01-product/organization-admin/catalogs-api-contract.md`:
- `GET /resources`: optional `resourceTypeId` filter, `search` on `name`, `sortBy` (`createdAt` or `name`), `order`, pagination. Returns `Resource[]` (each with `relatedResourceIds` and sorted `images`).
- `POST /resources`: validates `resourceTypeId` belongs to the organization and validates fields according to the cubes enabled on that `resourceType`:
  - `geography: true` (`role = 'place'`): `type` defaults to `'ordinary'`; `'ordinary'` requires `address` («Enter an address»); `'virtual'` sets `address = 'Virtual place'`; `timeZone` defaults to organization's `timeZone` and must be a valid IANA zone;
  - `duration: true` (`role = 'offering'`): `defaultDuration` required, whole integer `5..1440` in steps of `5` («Enter a duration from 5 to 1440 minutes in steps of 5»);
  - `showcase: true`: optional `description`, `videoLink` (YouTube/Vimeo), `instagram`, `tiktok`;
  - `skills: true`: optional `relatedResourceIds` (each must be a resource of another `skills`-enabled resource type in the same organization).
  - Fields for disabled cubes on that `resourceType` are ignored and stored as `null` / `[]`.
- `GET /resources/:id` and `PATCH /resources/:id` (changing `type` to `'virtual'` sets `'Virtual place'`; changing `'virtual'` to `'ordinary'` requires `address`; replacing `relatedResourceIds` updates `resource_relation` symmetrically so both sides see the link).
- File endpoints: `POST /resources/:id/files`, `DELETE /resources/:id/files/:fileId`, `PUT /resources/:id/files/order` (allowed when `showcase: true` on the resource's type; stores files at `/files/resources/{resourceId}/{fileId}.ext`).

### 3. Staff API (`/api/organization/staff`)

Follow `01-product/organization-admin/api-contract.md`:
- Replace `trainerId` with `resourceId` on `StaffMember`, `POST /staff`, and `PATCH /staff/:id`.
- `role: 'specialist'` requires `resourceId` («Select a specialist» on `REQUIRED`).
- `resourceId` must belong to a resource of this organization whose `resourceType` has `person: true` (`INVALID_VALUE` otherwise, or when sent with `administrator` / `accountant`).
- Linking a resource already linked to another staff member returns `409 SPECIALIST_ALREADY_LINKED` (`field: 'resourceId'`, message: «This specialist is already linked to another staff member»).

## Required tests & Acceptance criteria

1. `pnpm lint`, `pnpm format:check`, `pnpm test`, and `pnpm test:e2e` all pass with 0 errors.
2. E2E tests verify `/api/organization/resources` for all three roles (`place`, `resource`, `offering`), cube validations (`geography` with `Virtual place`, `duration` `5..1440` in steps of 5, `showcase` links and files `/files/resources/...`, symmetric `relatedResourceIds` across `resource_relation`).
3. E2E tests verify `/api/organization/staff` with `role: 'specialist'`, `resourceId`, and `SPECIALIST_ALREADY_LINKED`.
4. Record implementation notes in `01-product/organization-admin/catalogs-unified-backend-brief-feedback.md`.
