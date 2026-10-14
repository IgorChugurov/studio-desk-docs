# Feedback — Hall type and time zone, class-type duration, trainer link

Implementation notes for `catalogs-schedule-fields-backend-brief.md`. The brief and the canon were not edited. Only the owner changes canon.

## 1. Open questions

The brief has no open questions. None added.

## 2. Deviations from the brief

- **Phase order.** The brief lists the contract extension as phase 1, before implementation. The owner instructed, for this work item, to show the column and link-table structure first, implement after agreement, and then write `catalogs-api-contract.md` to match what was built. The structure was shown and agreed before any migration. The contract was updated after implementation to describe exactly what was built. No behaviour changed because of the order.

## 3. Behaviour decided in the backend

Neither canon nor the brief fixed these; they were decided here and must not change product behaviour.

- **Type values.** `type` is stored as `ordinary` or `virtual`.
- **Hall type is immutable (API).** `PATCH /halls/{id}` reads the stored row first. A sent `type` different from the stored one is refused with `400 VALIDATION_ERROR (INVALID_VALUE)` on `type`; nothing is written. The DB also keeps the check for allowed values.
- **Virtual hall address.** On create, a virtual hall's `address` is forced to `Virtual hall` (any sent value is ignored). On update, a sent `address` other than `Virtual hall` for a virtual hall is refused with `400 VALIDATION_ERROR (INVALID_VALUE)` on `address`. A DB check (`hall_virtual_address`) is the second layer.
- **Time zone validation.** A sent `timeZone` is validated against the same set as studio settings (`Intl.supportedValuesOf('timeZone')`). An unknown value is `400 VALIDATION_ERROR (INVALID_VALUE)` on `timeZone`, the same shape as a studio-settings time zone. The DB check only enforces non-blank, mirroring `studio.time_zone`. An omitted or blank `timeZone` on create stores the studio's time zone.
- **Duration errors.** A missing or `null` `defaultDuration` is `REQUIRED`; a value that is not a whole number from 5 to 240 in steps of 5 is `INVALID_VALUE`. Both carry the fixed text «Enter a duration from 5 to 240 minutes in steps of 5». The backend never substitutes 60 for a missing value. Non-number input (for example a numeric string) is refused as `INVALID_VALUE`.
- **Trainer–class type link shape.** The relation is one table, `trainer_class_type`, with `studio_id`, `trainer_id`, `class_type_id`, `created_at`, primary key `(trainer_id, class_type_id)`. A trainer returns `classTypeIds`; a class type returns `trainerIds` — the same rows from both sides. A sent list replaces that record's links (delete then insert); an empty list clears them; duplicate ids in the input are de-duplicated.
- **Cross-studio link refusal.** An id in `classTypeIds`/`trainerIds` that is not a record of the session's studio (unknown or another studio) is refused with `400 VALIDATION_ERROR (INVALID_VALUE)` on that field. This was chosen over `404` because the id is an element inside a list on a create/update body, not the resource addressed by the path.
- **Grants.** `trainer_class_type` grants `SELECT, INSERT, DELETE` to `studio_api` (no `UPDATE`: the table has no updatable column). `platform_api` and `public_api` have none.

## 4. Items the implementer believes touch the canon (opinion only; canon not edited)

- None. The canon already describes hall type and time zone, the required default duration of 60 as a form prefill, and the single trainer–class type relation. The implementation follows it. The id-array representation of the link in the API is a contract detail, not canon.

## 5. Documents updated

- `catalogs-api-contract.md` — extended for `hall.type`/`hall.timeZone`, `class_type.defaultDuration`, the trainer/class-type link (`classTypeIds` and `trainerIds`), the new error rows, and the tables/grants section.
- This feedback file — created.
- No canon (`catalogs.md`) or brief changes.

## 6. Test note

The backfill of **pre-existing** halls (type `ordinary`, studio time zone) and class types (duration 60) is a one-time migration step. It cannot be reproduced in the integration suite, because migrations run once in global setup before any test data exists, and the new columns are now required so a pre-migration row cannot be inserted. It is covered by the migration SQL (`0012_catalog_schedule_fields.sql`: `hall.type` default `ordinary`, `UPDATE hall SET time_zone = studio.time_zone`, `class_type.default_duration` added with default 60 then dropped) and by the suite applying all migrations cleanly. The create-path defaults (a hall with no type is `ordinary`; a hall with no time zone receives the studio's time zone) are tested through the API in `test/studio-catalog-schedule-fields.e2e-spec.ts`.
