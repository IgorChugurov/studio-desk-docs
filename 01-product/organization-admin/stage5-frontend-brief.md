# Frontend brief — Stage 5: Unified `Catalogs` (`/resources`) and Universal `Schedule` (`/schedule-slots`, `/week-templates`, `/weeks`)

**Derived from:**
- `01-product/profile.md`
- `01-product/organization-admin/catalogs.md`
- `01-product/organization-admin/catalogs-api-contract.md`
- `01-product/organization-admin/schedule.md`
- `01-product/organization-admin/schedule-api-contract.md`
- `01-product/organization-admin/api-contract.md`
- `02-design/flow-design.md`

## Mission

Complete the frontend by replacing the legacy split `Catalogs` (`Halls`, `Trainers`, `Class types`) and legacy `Schedule` (`/classes`) with the unified resource-driven `Catalogs` and universal `Schedule`:
1. **`Catalogs` (`src/studio/catalogs/*` and `src/app/studio/(app)/catalogs/*`)**:
   - Build the second-level tab bar dynamically from `session.config.catalogConfig.resourceTypes` using the current interface language (`en`, `sk`, `uk`).
   - Replace the separate hall/trainer/class-type lists and forms with one unified Resource list and Resource create/edit form calling `/api/organization/resources` (`GET /resources?resourceTypeId=...`, `POST /resources`, `GET /resources/:id`, `PATCH /resources/:id`, and `/resources/:id/files`).
   - Show form fields and table columns according to the cubes enabled on the active `resourceType`:
     - always: `Name`;
     - `geography: true` (`role === 'place'`): `Type` (`ordinary` / `virtual`), `Address` (auto-filled read-only `Virtual place` when `virtual`; cleared when switched back to `ordinary`; shown as a table column in the list), `Time zone` (defaults to organization's time zone; handles `409 RESOURCE_BUSY` on time-zone change);
     - `duration: true` (`role === 'offering'`): `Default duration` (`5..1440` minutes in steps of `5`, default `60`; shown as a table column in the list; error `Enter a duration from 5 to 1440 minutes in steps of 5`);
     - `showcase: true`: `Description`, `Video link`, `Instagram`, `TikTok`, and on the edit page the file gallery (`/resources/:id/files`, `/resources/:id/files/:fileId`, `/resources/:id/files/order`, modal text `This file will be removed. You can add it again at any time`);
     - `skills: true`: multi-selects of related resources for each other `skills: true` resource type in `catalogConfig.resourceTypes` (`relatedResourceIds`).
   - On the edit page of any resource with `role === 'place'` or `role === 'resource'`, render the embedded weekly schedule component at the bottom of the page filtered to that resource (`schedule.md`).
2. **`Schedule` (`src/studio/schedule/*` and `src/app/studio/(app)/schedule/*`)**:
   - Second-level tabs under `Schedule`: **`Schedule`** and **`Templates`**.
   - Evolve the schedule grid and slot modal from `/classes` to `/schedule-slots` (`GET /schedule-slots`, `POST /schedule-slots`, `PATCH /schedule-slots/:id`, `POST /schedule-slots/:id/cancel`, `POST /schedule-slots/:id/restore`, `DELETE /schedule-slots/:id`, `POST /schedule-slots/:id/replace`):
     - **Dynamic column mode switcher & filters:** built from `catalogConfig.resourceTypes` (`place` and `resource` types for columns; all resource types including `offering` for filters). If the organization has only one `place` resource and has another `resource` type, default the column view to that second resource type.
     - **Single-place auto-selection:** when the organization has only one `place` resource, prefill `placeResourceId` automatically in the slot and template-row forms.
     - **Two slot kinds (`service` and `block`):**
       - `service`: selects resources per `catalogConfig.resourceTypes` (`required`, `multiple`, filtering `skills`-linked resources when an `offering` is chosen, prefilling `durationMinutes` from the chosen `offering`'s `defaultDuration`), `durationMinutes` (`5..1440` step `5`), and conditionally shows `Audience & places` (`group` / `one_person` + `places`) and `Format` (`offline` / `offline_and_online` / `online_only`) based on `scheduleConfig.audienceEnabled` and `scheduleConfig.formatEnabled`.
       - `block` (`Block time` / `Blocked time`): no `offering`; edits `Date`, `Start time`, `Duration`, `Block entire organization` (`blocksAll`), resource selectors (when not `blocksAll`), and `Reason / Note` (`note`). Rendered as a neutral block in the grid (across all columns when `blocksAll = true`).
     - **Statuses & `occupiesResources`:** support the organization's enabled statuses (`scheduleConfig.statuses`: active `planned`, `confirmed`, `in_progress`, `completed`; inactive `cancelled`, `no_show`). Inactive slots (`occupiesResources = false`) render as the side stripe next to an active slot or as an inactive card (`Cancelled` / `No-show`), opening the inactive list modal with `Restore` and `Delete` for future slots. Allow updating status on a past `service` slot to `completed` or `no_show` if enabled.
     - **Conflict handling (`409 RESOURCE_BUSY` vs `409 OVERLAP_WARNING`):**
       - `409 RESOURCE_BUSY` (`strict`): show the conflict error in the modal; cannot be bypassed.
       - `409 OVERLAP_WARNING` (`warning` resource or overlap with a `block`): display the warning list in the modal with a **`Save anyway`** button that re-submits with `confirmOverlap: true`.
     - **Week actions (`Apply template`, `Clear weeks`, `Replace with template`) & `Templates` tab:**
       - `Templates` tab (`/week-templates` and `/week-templates/:id/rows`): list, create, rename, delete template, and edit template rows (`service` and `block` rows by `weekday`) in a dateless week grid.
       - Week action buttons (`Apply template`, `Clear weeks`, `Replace with template` calling `/weeks/apply`, `/weeks/clear`, `/weeks/replace` with preview counts and the `Conflicts` resolution modal `Keep existing` / `Apply new`). Hidden for `specialist` role and on embedded resource pages.

## Frontend repository checks

Before finishing, run the checks in `studio-desk-frontend/.cursor/rules/project.mdc`.

## Acceptance criteria

1. `pnpm lint`, `pnpm format:check`, and `pnpm test` in `studio-desk-frontend` all pass with 0 errors.
2. Legacy `/halls`, `/trainers`, `/class-types`, and `/classes` calls are completely replaced by `/resources`, `/schedule-slots`, `/week-templates`, and `/weeks`.
3. Component tests cover unified `Catalogs` (tabs, cube-conditional fields for `place`, `resource`, and `offering`, gallery, embedded schedule) and universal `Schedule` (dynamic resource columns/filters, `service` and `block` slots, `OVERLAP_WARNING` confirmation with `confirmOverlap: true`, statuses, `Templates`, and week actions).
4. Record implementation notes in `01-product/organization-admin/stage5-frontend-brief-feedback.md`.
