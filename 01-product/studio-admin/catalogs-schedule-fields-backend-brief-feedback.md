# Feedback — Hall type and time zone, class-type duration, trainer link

Implementation notes for `catalogs-schedule-fields-backend-brief.md`. On 2026-10-14 the owner changed the hall type from immutable to editable. Canon, the brief, and the contract were updated to match.

## 1. Open questions

The brief has no open questions. None added.

## 2. Deviations from the brief

- **Phase order.** The brief lists the contract extension as phase 1, before implementation. The owner instructed, for this work item, to show the column and link-table structure first, implement after agreement, and then write `catalogs-api-contract.md` to match what was built. The structure was shown and agreed before any migration. The contract was updated after implementation to describe exactly what was built. No behaviour changed because of the order.

## 3. Behaviour decided in the backend

Neither canon nor the brief fixed these; they were decided here and must not change product behaviour.

- **Type values.** `type` is stored as `ordinary` or `virtual`.
- **Hall type is editable (API).** `PATCH /halls/{id}` accepts a new `type`. A change to `virtual` stores `address` as `Virtual hall`. A change to `ordinary` without a new `address` is `400 VALIDATION_ERROR (REQUIRED)` on `address`. The DB check still allows only `ordinary` and `virtual`.
- **Virtual hall address.** On create, a virtual hall's `address` is forced to `Virtual hall` (any sent value is ignored). The same force applies when the type changes to `virtual`. While the hall stays virtual, a sent `address` other than `Virtual hall` is refused with `400 VALIDATION_ERROR (INVALID_VALUE)` on `address`. A DB check (`hall_virtual_address`) is the second layer.
- **Time zone validation.** A sent `timeZone` is validated against the same set as studio settings (`Intl.supportedValuesOf('timeZone')`). An unknown value is `400 VALIDATION_ERROR (INVALID_VALUE)` on `timeZone`, the same shape as a studio-settings time zone. The DB check only enforces non-blank, mirroring `studio.time_zone`. An omitted or blank `timeZone` on create stores the studio's time zone.
- **Duration errors.** A missing or `null` `defaultDuration` is `REQUIRED`; a value that is not a whole number from 5 to 240 in steps of 5 is `INVALID_VALUE`. Both carry the fixed text «Enter a duration from 5 to 240 minutes in steps of 5». The backend never substitutes 60 for a missing value. Non-number input (for example a numeric string) is refused as `INVALID_VALUE`.
- **Trainer–class type link shape.** The relation is one table, `trainer_class_type`, with `studio_id`, `trainer_id`, `class_type_id`, `created_at`, primary key `(trainer_id, class_type_id)`. A trainer returns `classTypeIds`; a class type returns `trainerIds` — the same rows from both sides. A sent list replaces that record's links (delete then insert); an empty list clears them; duplicate ids in the input are de-duplicated.
- **Cross-studio link refusal.** An id in `classTypeIds`/`trainerIds` that is not a record of the session's studio (unknown or another studio) is refused with `400 VALIDATION_ERROR (INVALID_VALUE)` on that field. This was chosen over `404` because the id is an element inside a list on a create/update body, not the resource addressed by the path.
- **Grants.** `trainer_class_type` grants `SELECT, INSERT, DELETE` to `studio_api` (no `UPDATE`: the table has no updatable column). `platform_api` and `public_api` have none.

## 4. Items the implementer believes touch the canon (opinion only)

- The owner decided that hall type is editable after create. That is now in canon. The id-array representation of the trainer–class type link remains a contract detail.

## 5. Documents updated

- `catalogs.md` — hall type can be changed; virtual address still fixed; switch to ordinary clears the address.
- `catalogs-schedule-fields-backend-brief.md` — hall type is no longer immutable.
- `catalogs-api-contract.md` — `PATCH /halls/{id}` accepts a type change; switch to ordinary requires an address.
- This feedback file.

## 6. Test note

The backfill of **pre-existing** halls (type `ordinary`, studio time zone) and class types (duration 60) is a one-time migration step. It cannot be reproduced in the integration suite, because migrations run once in global setup before any test data exists, and the new columns are now required so a pre-migration row cannot be inserted. It is covered by the migration SQL (`0012_catalog_schedule_fields.sql`: `hall.type` default `ordinary`, `UPDATE hall SET time_zone = studio.time_zone`, `class_type.default_duration` added with default 60 then dropped) and by the suite applying all migrations cleanly. The create-path defaults (a hall with no type is `ordinary`; a hall with no time zone receives the studio's time zone) are tested through the API in `test/studio-catalog-schedule-fields.e2e-spec.ts`.
