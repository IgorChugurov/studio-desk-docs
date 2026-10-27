# Feedback — Stage 3: Universal Schedule

Implementation notes for `schedule-unified-backend-brief.md`. The brief and the canon were not edited.

## 1. What was built

`scheduled_class`, `scheduled_class_trainer`, and `week_template_row_trainer` are gone. `0000_bumpy_nighthawk.sql` and `src/database/schema/schedule.ts` now create `schedule_slot`, `schedule_slot_resource`, `week_template`, `week_template_row`, and `week_template_row_resource`.

One exclusion constraint enforces strict resources:

`EXCLUDE USING gist (resource_id WITH =, during WITH &&) WHERE (during IS NOT NULL AND enforce_no_overlap = true)`

`during` is `[starts_at, ends_at)` while `occupies_resources` is true, and null otherwise. `enforce_no_overlap` is true only when that resource's type has `conflictMode = strict`. `confirmOverlap` cannot pass a strict overlap.

`/api/organization/classes` is replaced by `/api/organization/schedule-slots`. Week templates and week actions use the same slot shape (`kind`, `placeResourceId`, `resourceIds`, `durationMinutes` 5..1440, `audience`, `format`, `places`, `note`, `blocksAll`).

`studio_desk` and `studio_desk_test` were dropped and migrated from the edited base migration. There is no data migration from `scheduled_class`.

## 2. Deviations from the brief

None on the contract. Two backend choices are recorded below because the brief does not spell them out.

## 3. Behaviour decided in the backend

- **Grants.** The database role remains `studio_api` (the name already used by this API). It has `SELECT, INSERT, UPDATE, DELETE` on the five schedule tables. The contract text says `organization_api`; that role does not exist in this database.
- **`blocksAll`.** The flag is stored on the slot. The slot still writes only its place and its `resourceIds` into `schedule_slot_resource`. It does not insert a row for every resource in the organization. A strict overlap is still only on a shared strict resource. Any other overlap with an active `blocksAll` slot, or a new `blocksAll` block over any active slot, is `409 OVERLAP_WARNING` until `confirmOverlap: true`.
- **Conflict body.** `409 RESOURCE_BUSY` returns `conflict`: `{ resourceId, resourceName, start, end, title, slotId }`. `title` is the offering names for a service, or the note (or `Blocked time`) for a block. `409 OVERLAP_WARNING` returns `conflicts` as a list of the same shape, plus `started`.
- **Inactive future slot.** `PATCH` returns `409 SLOT_CANCELLED` when the status is `cancelled`, and `409 SLOT_INACTIVE` for `no_show`. Restore is `POST /schedule-slots/{id}/restore`.
- **Past service.** Time and resources stay frozen (`409 SLOT_ALREADY_STARTED`). `status` may become `completed` or `no_show` when that status is enabled. A past block has no status exception.
- **List range.** `GET /schedule-slots` widens `from`/`to` to Monday–Sunday and refuses a span longer than 366 days. It does not refuse a week before the current week, so a past slot can be marked `completed` or `no_show`. Week actions still start at the current week.
- **Week replace.** Slots that the action is about to delete are not treated as conflicts. Conflicts are only slots that would remain.
- **Concurrent strict inserts.** Two overlapping writes can deadlock (`40P01`) on the GiST index instead of losing with `23P01`. The request is retried once. The loser then answers `409 RESOURCE_BUSY`.
- **Template rows.** Same-weekday overlaps are refused only when the rows share a resource whose type is `strict` (`409 RESOURCE_BUSY`). Warning resources and `blocksAll` do not conflict inside a template.

## 4. Checks

- `pnpm lint` — exit 0. One existing warning in `src/apis/organization/halls/file-type.ts` (`no-misused-spread`), not from this change.
- `pnpm format:check` — exit 0.
- `pnpm test` — 16 passed.
- `pnpm test:e2e` — 117 passed, including `test/studio-schedule.e2e-spec.ts` (service and block, `blocksAll`, duration 1440, single-place auto-select, six statuses, strict vs warning vs block overlap, week apply/clear/replace, specialist read access).

`src/database/migrations/meta/0000_snapshot.json` was not regenerated. Drizzle applies the SQL file, not the snapshot. The snapshot was already behind the hand-edited migration.

The development API was using `studio_desk`. That database was recreated empty, so a running `npm run dev` needs a restart.
