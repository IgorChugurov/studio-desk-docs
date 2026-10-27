# Stage 1 feedback — organization presets

The organization API still logs in as the PostgreSQL role `studio_api` (`DB_STUDIO_API_PASSWORD`). `01-init.sh` and `.env` were not changed.

There is one clean migration, `0000_bumpy_nighthawk.sql`. It creates the `organization` and `preset` schema directly. It does not rename or backfill `studio` rows. `studio_desk` and `studio_desk_test` were recreated from this file. An existing database that still has the old migration history must be recreated the same way; applying this file on top of the old tables will not work.

Drizzle Kit emitted two statements that PostgreSQL rejects. The migration file inlines the subdomain and domain checks (Kit wrote `$1` with no parameter) and creates `platform_administrator_one_row` as `((true))`. Grants, `btree_gist`, the two schedule exclusion constraints, currency, `role_section`, and the five starter presets are appended after the generated SQL. Generating migrations again drops that appendix; it has to be put back.

`expected-grants.ts` lists table privileges only. The organization API also has a column grant:

`GRANT UPDATE (language, country, currency, time_zone, schedule_config, billing_config, updated_at) ON organization TO studio_api`

That column grant does not appear as table `UPDATE` in `information_schema.role_table_grants`.

Starter presets use fixed ids `00000000-0000-4000-8000-000000000001` through `...0005`. Their jsonb uses the API's camelCase (`conflictMode`, `audienceEnabled`, `settlementMode`). Tests truncate `preset` and insert that seed block again. `currency` and `role_section` are not truncated.

Staff role `trainer` is now `specialist` in `role_section` and `organization_staff`. The catalog table `trainer`, `trainerId`, `staff_trainer`, and `TRAINER_ALREADY_LINKED` stay until stages 2–3.

`RESOURCE_TYPE_IN_USE` is the stage-1 stand-in for resource links that do not exist yet. Removing the place type while a hall exists, the offering type while a class type exists, or a person type while a trainer or `staff_trainer` row exists returns 409. Turning `person` from true to false while a `staff_trainer` row exists returns 409.

A blank `presetId` is `REQUIRED`. A value that is not an id of a preset is `INVALID_VALUE`. Both messages are `Select a preset`. `PATCH` ignores `presetId`.

Duplicate preset names are `{name} (copy)`, then `{name} (copy 2)`, and so on. The base is shortened so the result stays within 100 characters.

`sections` on an organization are returned in canonical order: `schedule`, `catalogs`, `clients`, `subscriptions`, `accounting`, `organization-settings`, `staff`.

The schedule e2e date moved from `2026-10-20` to `2026-11-03`. `2026-10-20` is before the current week, so class creation was rejected as already started.
