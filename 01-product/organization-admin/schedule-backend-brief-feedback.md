# Feedback — Schedule

Implementation notes for `schedule-backend-brief.md`. The brief and the canon were not edited. Only the owner changes canon.

## 1. Overlap mechanism

PostgreSQL extension `btree_gist` is created in migration `0014_schedule.sql`. It lets one exclusion constraint compare a uuid and a time range.

A class stores `date` and `start_time` by the hall's clock. `starts_at` and `ends_at` are that moment in the hall's time zone. The hall constraint is:

`EXCLUDE USING gist (hall_id WITH =, tstzrange(starts_at, ends_at, '[)') WITH &&) WHERE (NOT cancelled)`

A trainer row stores the same interval in `during`, and `during` is null when the class is cancelled. The trainer constraint is:

`EXCLUDE USING gist (trainer_id WITH =, during WITH &&) WHERE (during IS NOT NULL)`

The range is half-open, so a class that ends at 19:00 and a class that starts at 19:00 both fit. A cancelled class is outside both constraints. Two concurrent overlapping inserts cannot both commit: the loser gets SQLSTATE `23P01`. The request then answers `409 HALL_BUSY` or `409 TRAINER_BUSY`.

Template rows are not in these constraints. Rows of one weekday are compared as clock minutes in the application, so two halls in different time zones still conflict when the digits overlap.

## 2. Deviations from the brief

None. Endpoints, the stored instants, the Monday week in the studio time zone, and the unpaginated class list are the decisions the contract already records. The owner agreed that contract on 2026-10-16.

## 3. Behaviour decided in the backend

- The class list and the week actions refuse a week before the current week. The current week is Monday–Sunday containing today in the studio's time zone.
- A class list is every class in the widened weeks, not a page. The span may not be longer than 366 days.
- Conflict and result sentences are the fixed texts. The screen builds a conflict sentence from `conflict`. A count sentence is returned in `text`. The singular form is used only when the first count is 1.
- `PATCH /halls/{id}` refuses a time-zone change that would put one trainer in two classes. The error names the earliest conflicting class. Clock times are not rewritten.

## 4. Items the implementer believes touch the canon

None. The canon already requires the database to reject overlaps, back-to-back to pass, and a cancelled class to occupy nothing.

## 5. Documents updated

- `schedule-api-contract.md` — written before implementation and agreed.
- This feedback file.
- No canon or brief changes.
