# Backend brief — Stage 1: Organization core, modular presets, and double-filter access

**Derived from:**
- `01-product/profile.md`
- `01-product/platform-admin/overview.md`
- `01-product/platform-admin/api-contract.md`
- `01-product/organization-admin/overview.md`
- `01-product/organization-admin/api-contract.md`
- `03-architecture/backend-structure.md`
- `03-architecture/authentication.md`
- `03-architecture/api-conventions.md`

## Mission

Evolve the backend foundation from single-purpose `studio` to the modular `organization` core:
1. Build `/api/platform/presets` (modular organization blueprints with `sections`, `catalogConfig`, `scheduleConfig`, `billingConfig` and localized `en/sk/uk` labels) and seed the starter presets in a migration.
2. Evolve `/api/platform/studios` into `/api/platform/organizations`, requiring `presetId` on creation to stamp the preset's modular configuration into the new organization in a single transaction, and allowing `sections` updates on `PATCH /api/platform/organizations/{id}`.
3. Evolve `/api/studio` into `/api/organization` (`sd_organization_refresh`, `select-organization`, `switch-organization`, `organization-settings`, double-filter section access `organization_section` ∩ `role_section`, and `config` in `GET /me` and `GET / PATCH /settings`).

## Sequencing

This is **Stage 1 of 5**. It has no external dependencies.
To keep the repository green on every stage:
- Stage 1 updates the tenant core (`organization`), platform presets/organizations, organization auth, `/me`, `/settings`, and section guards.
- Existing catalog and schedule tables/controllers (`hall`, `trainer`, `class_type`, `scheduled_class`, `week_template`, `staff_trainer`) are kept compiling and passing tests against the renamed `organization` / `/api/organization` base until Stages 2 and 3 replace them with unified `/resources`, `staff_resource`, and `/schedule-slots`.

## Source-of-truth order

1. `01-product/profile.md`
2. `01-product/platform-admin/overview.md` and `01-product/platform-admin/api-contract.md`
3. `01-product/organization-admin/overview.md` and `01-product/organization-admin/api-contract.md`
4. `03-architecture/backend-structure.md`, `03-architecture/authentication.md`, `03-architecture/api-conventions.md`
5. This brief for Stage 1 scoping.

## Current backend baseline

Verified in `studio-desk-backend`:
- `src/database/schema/studio.ts`, `studio-auth.ts`, `role-section.ts`, `hall.ts`, `catalog.ts`, `schedule.ts`
- `src/database/migrations/0000_*.sql` .. `0014_schedule.sql`
- `src/database/expected-grants.ts` and `src/database/db-roles.ts` (note: PostgreSQL role in `docker/postgres/init/01-init.sh` and `.env` is `studio_api` / `DB_STUDIO_API_PASSWORD`; keep the PostgreSQL login role name `studio_api` in `db-roles.ts` unless `01-init.sh` and `.env` are updated together, or alias it cleanly so existing local/CI Postgres containers do not break — record the choice in feedback)
- `src/apis/platform/` (`auth`, `studios`, `version`)
- `src/apis/studio/` (`auth`, `me`, `settings`, `staff`, `access`, `halls`, `schedule`)
- `test/*.e2e-spec.ts` (16 e2e test suites)

## Product behaviour and Stage 1 deliverables

### 1. Database schema & migrations (`src/database/`)

**IMPORTANT — DEVELOPMENT MODE (NO DATA MIGRATION NEEDED):**
This project is in active development. There is NO production data to preserve. Do NOT waste time writing complex `ALTER TABLE ... RENAME` or data-backfilling SQL migrations from old `studio` rows. You may either replace/reset the schema cleanly or drop and recreate the tables from scratch so the schema matches the new `organization` model directly.

1. **Presets table (`preset`):**
   - `id` (uuid, PK), `name` (text, unique, 2..100 chars trimmed), `description` (text, nullable), `sections` (jsonb or text array, non-empty subset of `schedule`, `catalogs`, `clients`, `subscriptions`, `accounting`, `organization-settings`, `staff`), `catalog_config` (jsonb), `schedule_config` (jsonb), `billing_config` (jsonb), `created_at`, `updated_at`.
   - Seed starter presets (at minimum: `Fitness Studio` with `Hall` (`role: 'place'`), `Trainer` (`role: 'resource'`, `person: true`, `skills: true`), `Class type` (`role: 'offering'`, `duration: true`, `skills: true`); plus `Beauty Salon`, `Auto Service`, `Medical Clinic`, `OblikFlow Accounting`) with valid `en`, `sk`, `uk` labels.
2. **Tenant schema (`organization`) and configuration tables:**
   - Use `organization` (`organization_id`, `organization_staff`, `organization_session`, `organization_sign_in_code`, `organization_selection_ticket`) directly.
   - In `role_section`, use `'organization-settings'` (and `'specialist'` if updating staff role now).
   - On `organization`, include `preset_id` (uuid, nullable FK to `preset.id` `ON DELETE SET NULL`) and `preset_name` (text, not null).
   - Include `organization_section` (`organization_id`, `section`, PK `(organization_id, section)`), `schedule_config` and `billing_config`, and `organization_resource_type` (`id`, `organization_id`, `sort_order`, `role` in `'place' | 'resource' | 'offering'`, `labels` jsonb `en/sk/uk`, `geography`, `duration`, `showcase`, `person`, `skills`, `required`, `multiple`, `conflict_mode` in `'strict' | 'warning' | 'none'`, `created_at`, `updated_at`).
3. **Grants:**
   - Update `expected-grants.ts` and grant SQL so `grants.e2e-spec.ts` passes.

### 2. Platform API (`/api/platform`)

Follow `01-product/platform-admin/api-contract.md`:
1. **`/api/platform/presets`:**
   - `GET /presets` (list with `search` on `name`, `sortBy` in `createdAt | name`, `order`, pagination)
   - `POST /presets` (validate `name` 2..100, non-empty `sections`, `catalogConfig.resourceTypes` having **exactly one** `role: 'place'`, valid `en/sk/uk` labels, `409 PRESET_NAME_TAKEN`)
   - `GET /presets/:id` (`404 NOT_FOUND`)
   - `PATCH /presets/:id`
   - `POST /presets/:id/duplicate` (creates `{name} (copy)` or `{name} (copy N)`)
   - `DELETE /presets/:id` (`204`; existing organizations keep `preset_name` and their copied configuration while `preset_id` becomes `null`)
2. **`/api/platform/organizations` (replaces `/api/platform/studios`):**
   - `GET /organizations` (returns `OrganizationListItem` including `preset: { id, name }`)
   - `POST /organizations` (requires `presetId`; stamps `sections`, `scheduleConfig`, `billingConfig`, and `organization_resource_type` rows in one transaction; returns `201 Organization`)
   - `GET /organizations/:id`
   - `PATCH /organizations/:id` (supports updating `name`, `subdomain`, `customDomain`, `owner`, and `sections`; rejects empty `sections`; does not allow changing `presetId`)
   - `POST /organizations/:id/deactivate`, `POST /organizations/:id/activate`, `POST /organizations/:id/impersonate`

### 3. Organization API (`/api/organization`)

Follow `01-product/organization-admin/api-contract.md`:
1. **Routing & Auth:**
   - Mount organization API at `/api/organization` (and OpenAPI at `/api/organization/docs` and `/api/organization/docs/json`).
   - Cookie name: `sd_organization_refresh` with `Path=/api/organization/auth`.
   - Endpoints: `POST /auth/code`, `POST /auth/sign-in` (`result: "signed-in"` with `organization` or `result: "organization-selection"` with `organizations`), `POST /auth/select-organization` (`organizationId`), `POST /auth/switch-organization` (`organizationId`), `POST /auth/refresh`, `POST /auth/sign-out`, `POST /auth/impersonation/exchange`.
2. **Double-filter section access & `/me`:**
   - `GET /me` returns `user`, `organization`, `role`, `sections` (intersection of `organization_section` and `role_section` in canonical order: `schedule`, `catalogs`, `clients`, `subscriptions`, `accounting`, `organization-settings`, `staff`), `config` (`catalogConfig`, `scheduleConfig`, `billingConfig`), `impersonated`, `organizations`.
   - Section access guard/service checks **both** `organization_section` and `role_section`. If a section is disabled for the organization in `/api/platform/organizations/:id`, even `owner` receives `403 FORBIDDEN` on that section's endpoints.
3. **Organization settings (`/settings`):**
   - Section `organization-settings` (owner only).
   - `GET /settings` and `PATCH /settings` read and update `language`, `country`, `currency`, `timeZone`, plus `catalogConfig`, `scheduleConfig`, and `billingConfig` (validating resource type invariants: exactly one `place`, all `en/sk/uk` labels present).

## Required tests

Update existing e2e tests and add coverage so all suites pass (`pnpm lint`, `pnpm format:check`, `pnpm test`, `pnpm test:e2e`):
1. `grants.e2e-spec.ts`, `api-isolation.e2e-spec.ts`, `route-protection.e2e-spec.ts`, `error-format.e2e-spec.ts`, `version-and-cors.e2e-spec.ts`.
2. Platform presets e2e tests: list, create, validation errors (missing labels, 0 or 2 `place` resource types, empty sections, duplicate name), update, duplicate, delete (verifying an organization created from the deleted preset keeps its config and `preset.name`).
3. Platform organizations e2e tests: create with `presetId`, verify copied config in organization, update `sections`, verify double-filter blocks disabled sections in `/api/organization`.
4. Organization auth, `/me`, and `/settings` e2e tests: `sd_organization_refresh`, `select-organization`, `switch-organization`, `GET /me` returning `config` and intersected `sections`, `PATCH /settings` updating `catalogConfig` / `scheduleConfig` / `billingConfig`.

## Acceptance criteria

1. `pnpm lint`, `pnpm format:check`, `pnpm test`, and `pnpm test:e2e` all pass with 0 errors.
2. `/api/platform/presets` and `/api/platform/organizations` match `01-product/platform-admin/api-contract.md`.
3. `/api/organization/auth/*`, `/api/organization/me`, and `/api/organization/settings` match `01-product/organization-admin/api-contract.md`.
4. Double-filter section access (`organization_section` ∩ `role_section`) is enforced.
5. Implementation notes and any edge-case decisions are recorded in `01-product/platform-admin/organization-presets-backend-brief-feedback.md`.
