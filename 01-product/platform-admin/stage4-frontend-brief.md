# Frontend brief — Stage 4: Platform Admin (`Organizations` & `Presets`) + Organization Admin shell, `Staff`, and `Organization settings`

**Derived from:**
- `01-product/profile.md`
- `01-product/platform-admin/overview.md`
- `01-product/platform-admin/api-contract.md`
- `01-product/organization-admin/overview.md`
- `01-product/organization-admin/api-contract.md`
- `02-design/flow-design.md`

## Mission

Update `studio-desk-frontend` to match the completed Stage 1–3 backend for:
1. **Platform Admin (`/platform`)**:
   - Add the top tab bar under the navbar with two tabs: **`Organizations`** and **`Presets`**.
   - Evolve the studios list and form into **`Organizations`** (`/api/platform/organizations`): show `Preset` column in the table, require selecting a `Preset` when creating an organization, show `Preset` read-only and allow editing `Enabled sections` (`sections` checkboxes) when editing an organization, update all fixed texts from `Studio / Studios` to `Organization / Organizations` (`Create organization`, `New organization`, `Organization created`, `Organization updated`, `Deactivate organization?`, `Organization deactivated`, `Organization activated`, `Log in as organization`, `No organizations found`, `This domain is already used by another organization`, `This e-mail is already a staff member of this organization`).
   - Build the **`Presets`** tab and pages (`/api/platform/presets`): list with search, `Create preset`, row menu `Edit`, `Duplicate`, `Delete` (with confirmation modal `Delete preset?`), and the preset create/edit form (`General & Modules`, `Schedule & Billing configuration`, `Catalog resource types` cards with `en/sk/uk` singular & plural labels, cubes `geography`, `duration`, `showcase`, `person`, `skills`, `required`, `multiple`, `conflictMode`).
2. **Organization Admin shell, `Staff`, and `Organization settings`**:
   - Point the client at `/api/organization` (and update `next.config.ts` local rewrite from `/api/studio/:path*` to `/api/organization/:path*`, `public-env.ts`).
   - Update auth/session payloads (`organization`, `organizations`, `organizationId`, `organization-selection`, `/auth/select-organization`, `/auth/switch-organization`, `config` on `GET /me` with `catalogConfig`, `scheduleConfig`, `billingConfig`).
   - Update section ID `'studio-settings'` → `'organization-settings'` (`Organization settings` in `i18n/language.ts`, `Switch organization`).
   - Update **`Staff`** (`/staff`) to use `role: 'specialist'` and `resourceId` (fetching `person: true` resources from `GET /resources?resourceTypeId=...`), dynamically displaying the localized label of the `person` resource type (e.g. `Trainer`, `Master`, `Mechanic`, or `Specialist` if multiple `person` types exist), and handling `SPECIALIST_ALREADY_LINKED` / `Select a specialist`.
   - Update **`Organization settings`** (`/settings`) so the owner can edit both the 4 base fields (`language`, `country`, `currency`, `timeZone`) and the organization's `catalogConfig.resourceTypes`, `scheduleConfig`, and `billingConfig` (`Organization settings saved`, `409 RESOURCE_TYPE_IN_USE`).

## Sequencing

This is **Stage 4 of 5**.
- Stage 4 updates Platform Admin (`Organizations` & `Presets`) and Organization Admin shell, `auth`, `me`, `i18n`, `Staff`, and `Organization settings`.
- Keep existing `Catalogs` and `Schedule` files compiling (update any broken imports from `session` / `api` / `sections` if needed) — **Stage 5** will rewrite `Catalogs` and `Schedule` UI to the unified `/resources` and `/schedule-slots` endpoints.
- **Note on internal Next.js folder path:** You may either keep the internal Next rewrite folder `src/app/studio` (since `src/proxy.ts` rewrites `app.*` host to it internally and the browser URL never shows `/studio`) or rename it, as long as external browser URLs (`/sign-in`, `/select`, `/schedule`, `/catalogs`, `/settings`, `/staff`, `/impersonate`) and API paths (`/api/organization/*`) work seamlessly. Keeping `src/app/studio` and `src/studio` as internal folder names avoids unnecessary file-move churn.

## IMPORTANT — Frontend CI rules (`studio-desk-frontend/AGENTS.md`)

Before finishing, you MUST run and pass all three commands in `studio-desk-frontend`:
1. `pnpm lint` (`oxlint --type-aware`) — do NOT call `setState` synchronously inside `useEffect` (`react(set-state-in-effect)`); do NOT call `String(obj)` on non-string values.
2. `pnpm format:check` (run `pnpm exec prettier --write` on all added/edited files).
3. `pnpm test` (Vitest).

## Deliverables

### 1. Platform Admin (`/platform`)

Follow `01-product/platform-admin/overview.md` and `01-product/platform-admin/api-contract.md`:
1. **Tabs under navbar:** `Organizations` and `Presets`.
2. **Organizations list & form:**
   - Calls `/api/platform/organizations` (`GET`, `POST`, `GET /:id`, `PATCH /:id`, `POST /:id/deactivate`, `POST /:id/activate`, `POST /:id/impersonate`).
   - Table columns: `Name`, `Preset` (`preset.name`), `Subdomain`, `Custom domain`, `Owner (e-mail)`, `Status`, `Created`, `⋮` menu (`Edit`, `Deactivate` / `Activate`, `Log in as organization`).
   - Create form (`New organization`): fields `Name`, `Subdomain`, `Custom domain (optional)`, `Owner e-mail`, `Preset` (select loaded from `GET /presets`, defaulting to first preset; error `Select a preset`).
   - Edit form: `Preset` shown read-only; `Enabled sections` checkboxes (`schedule`, `catalogs`, `clients`, `subscriptions`, `accounting`, `organization-settings`, `staff`; at least one required: `Select at least one section`).
   - Confirm changes dialog and Deactivate dialog updated to `organization` texts.
3. **Presets list & form:**
   - Calls `/api/platform/presets` (`GET`, `POST`, `GET /:id`, `PATCH /:id`, `POST /:id/duplicate`, `DELETE /:id`).
   - List: search by name, `Create preset` button, columns `Name`, `Resource types` (comma-separated English plural labels), `Settlement mode`, `Created`, `⋮` menu (`Edit`, `Duplicate`, `Delete`).
   - Empty states: `No presets yet` / `No presets found` with `Reset search`.
   - Duplicate action: `POST /presets/:id/duplicate`, toast `Preset duplicated`, reloads list.
   - Delete action: modal `Delete preset?`, text `Preset "{name}" will be deleted. Existing organizations created from it will keep their configuration`, buttons `Delete` and `Cancel`, toast `Preset deleted`.
   - Preset create/edit form:
     - `Name`, `Description (optional)`, `Enabled sections` checkboxes;
     - `Schedule & Billing configuration`: `Audience & places (Group / One person)`, `Format (Offline / Online)`, `Statuses` checkboxes (`planned` and `cancelled` always checked and disabled; `confirmed`, `in_progress`, `completed`, `no_show` toggleable), `Settlement mode` select (`subscription`, `direct_check`, `external_order`);
     - `Catalog resource types`: list of cards. First card is `role: 'place'` (cannot be removed). `Add resource type` adds a card with role `'resource'` or `'offering'`. Each card edits `en`, `sk`, `uk` singular & plural labels, cubes (`geography` locked true on `place`, `duration` locked true on `offering`, `showcase`, `person` on `resource`, `skills`), `required`, `multiple`, `conflictMode` (`strict`, `warning`, `none`; locked `none` on `offering`).
     - Validation & error texts: `Use 2–100 characters`, `This preset name is already taken`, `Select at least one section`, `Enter all language labels`.

### 2. Organization Admin shell, `Staff`, and `Organization settings`

Follow `01-product/organization-admin/overview.md` and `01-product/organization-admin/api-contract.md`:
1. **API & Session (`src/shared/config/public-env.ts`, `next.config.ts`, `src/studio/api/*`):**
   - Derive `/api/organization` instead of `/api/studio`; rewrite `/api/organization/:path*` in `next.config.ts`.
   - Update `auth-types.ts`, `create-api.ts`, `session-store.ts` to parse `organization`, `organizations`, `organization-selection`, `/auth/select-organization` (`{ selectionTicket, organizationId }`), `/auth/switch-organization` (`{ organizationId }`), and store `config` (`catalogConfig`, `scheduleConfig`, `billingConfig`) on the signed-in session state.
2. **Sections & i18n (`src/studio/sections.ts`, `src/studio/i18n/language.ts`, `src/studio/shell/*`, `src/studio/select/*`, `src/studio/impersonate/*`):**
   - Replace `'studio-settings'` with `'organization-settings'`.
   - Update UI strings in `en`, `sk`, `uk` (`Organization settings`, `Switch organization`, `This link has expired. Open the organization again from the platform admin`, etc.).
3. **Staff (`src/studio/staff/*`):**
   - Role is `'specialist'` (instead of `'trainer'`) and field is `resourceId` (instead of `trainerId`).
   - Load resources from `GET /resources` filtered to resource types that have `person: true` in `session.config.catalogConfig.resourceTypes`.
   - Use the localized singular label of the `person` resource type for the role option and field label (or `Specialist` if multiple `person` types exist). If no resource type has `person: true`, only `administrator` and `accountant` roles are shown.
   - Error texts: `This e-mail is already added to the organization`, `This e-mail belongs to the organization owner`, `Select a specialist`, `This specialist is already linked to another staff member`.
4. **Organization settings (`src/studio/settings/*`):**
   - Edits `language`, `country`, `currency`, `timeZone` AND `catalogConfig`, `scheduleConfig`, `billingConfig` via `GET /settings` and `PATCH /settings`.
   - Success toast: `Organization settings saved`.

## Acceptance criteria

1. `pnpm lint`, `pnpm format:check`, and `pnpm test` in `studio-desk-frontend` all pass with 0 errors.
2. All unit/component tests for Platform Admin (`Organizations` and `Presets`) and Organization Admin (`create-api`, `app-shell`, `sign-in`, `settings`, `staff`) are updated and green.
3. Record implementation notes in `01-product/platform-admin/stage4-frontend-brief-feedback.md`.
