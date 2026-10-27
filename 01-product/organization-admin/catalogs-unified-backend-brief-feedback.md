# Feedback — Stage 2: unified catalogs and staff_resource

Implementation notes for `catalogs-unified-backend-brief.md`. Canon in `studio-desk-docs` was not changed. Applied on 2026-10-27 in development: `studio_desk` and `studio_desk_test` were recreated from the edited base migration. There is no data migration from `hall`, `trainer`, or `class_type`.

## 1. What shipped

- Tables `resource`, `resource_file`, `resource_relation`, `staff_resource` replace `hall`, `hall_file`, `trainer`, `trainer_file`, `class_type`, `class_type_file`, `trainer_class_type`, `staff_trainer`. The change is in `src/database/schema/` and in `0000_bumpy_nighthawk.sql`.
- `/api/organization/resources` serves `place`, `resource`, and `offering` from the cubes on `organization_resource_type`. Disabled-cube fields are ignored and stored as `null` or `[]`.
- Geography: ordinary requires an address («Enter an address»). Virtual stores `Virtual place` and ignores a sent address. Omitted `timeZone` on create uses the organization's time zone.
- Duration: whole minutes 5..1440, step 5 («Enter a duration from 5 to 1440 minutes in steps of 5»).
- Showcase links use the existing YouTube/Vimeo, Instagram, and TikTok checks. Files are stored at `/files/resources/{resourceId}/{fileId}.ext` and are refused when `showcase` is false.
- Skills links are stored once with `resource_a_id < resource_b_id`. Both sides return `relatedResourceIds`. A self link, the same resource type, a type with `skills` off, or another organization is one `INVALID_VALUE` after format checks.
- Staff `role: 'specialist'` uses `resourceId` of a resource whose type has `person: true`, stored in `staff_resource`. Required text is «Select a specialist». A second link is `409 SPECIALIST_ALREADY_LINKED` on `resourceId`: «This specialist is already linked to another staff member».
- `PATCH /api/organization/settings` returns `409 RESOURCE_TYPE_IN_USE` when a removed type still has a `resource` row, or when `person` is turned off on a type that still has a `staff_resource` link. A resource of that type with no staff link does not block turning `person` off.
- Grants for `studio_api` (the existing login; the role was not renamed): `resource` SELECT/INSERT/UPDATE, `resource_file` SELECT/INSERT/UPDATE/DELETE, `resource_relation` SELECT/INSERT/DELETE, `staff_resource` SELECT/INSERT/UPDATE/DELETE.

## 2. Left for Stage 3

Schedule HTTP still uses `hallId`, `trainerIds`, and `classTypeId`. Columns `hall_id`, `class_type_id`, and `trainer_id` now reference `resource.id`. Place lookups require `type is not null`, offerings require `default_duration is not null`, and the specialist schedule filter still reads `staff_resource.resource_id` under the name `trainer_id`. A time-zone change on `PATCH /resources/:id` that conflicts with a class returns `409 RESOURCE_BUSY`. Creating a class still returns `HALL_BUSY` or `TRAINER_BUSY`.

## 3. Schema note

`src/database/migrations/meta/0000_snapshot.json` was not regenerated. The applied schema is the edited SQL. The next `drizzle-kit generate` will diff that stale snapshot and should not be used to emit a follow-up migration until the snapshot is rebuilt from this SQL.

Resource types are updated in place. Sort order is parked at `+1000` inside the transaction, then rewritten, and only removed types are deleted. Deleting every type and inserting it again would break the foreign key from `resource`.

## 4. Checks

`pnpm lint`, `pnpm format:check`, `pnpm test` (16), and `pnpm test:e2e` (116) passed. Lint still prints the existing `no-misused-spread` warning in `src/apis/organization/halls/file-type.ts` and exits 0.
