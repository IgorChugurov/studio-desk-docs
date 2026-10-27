# Backend brief feedback — Halls

The owner decides canon. This file records answers and decisions for `catalogs-backend-brief.md`.

1. **Open questions**
   1. Which image and video types are accepted, and what the maximum size of one file is. The owner decided on 2026-10-08: jpeg, png, webp, and gif up to 10 MB; mp4 and webm up to 100 MB. The type is read from the file bytes. An image and a video both remain possible.

2. **Deviations from the brief.** None. On 2026-10-08 the owner corrected the file address, and the brief was updated to match: hall files are public static files at `/files/halls/{hallId}/{fileId}` plus the type extension. Closing a file behind a session is a later topic.

3. **Behaviour the backend decided, because neither canon nor the brief defined it.**
   - A hall or file that is not in the session's studio is `404 NOT_FOUND`. A trainer and an accountant are `403 FORBIDDEN` because they have no `catalogs` section.
   - List sort is `createdAt` descending by default. `name` is also allowed. Pagination is the common list format.
   - Upload is `POST /halls/{id}/files` as `multipart/form-data`, field `files`. Delete is `DELETE /halls/{id}/files/{fileId}`. Reorder is `PUT /halls/{id}/files/order` with every id of that hall. Delete and reorder return the hall. One failed part rejects the whole upload and stores nothing.
   - A YouTube or Vimeo link is recognized by the host and path rules in `catalogs-api-contract.md`. The trimmed URL is stored as sent.
   - Tables `hall` and `hall_file`, agreed by the owner on 2026-10-08. The studio is not copied onto `hall_file`. `hall_file.created_at` stays. Migrated in `0009_halls`.
   - `GET /halls` returns `images` on every hall. The owner confirmed this on 2026-10-08. It was already in the contract.
   - Create and update of a hall, confirmed by the owner on 2026-10-08. File upload, delete, and reorder are implemented. The public address is stored on the file row and served as a static file from `FILE_STORAGE_DIR`.
   - A trainer cannot be stored: `studio_staff` allows only `administrator` and `accountant`. The catalogs refusal is tested with an accountant. A trainer hits the same section check once that role can be saved.

4. **What the implementer believes changes the canon.** The owner decided on 2026-10-08 that current files are public. `files.md` and `catalogs.md` were updated to say so. Session-closed files stay a later topic.

5. **Documents updated.** `catalogs-backend-brief.md`, `catalogs-frontend-brief.md`, `catalogs-api-contract.md`, `catalogs.md`, `../../03-architecture/files.md`, this file, and work item `2026-10-08-studio-admin-catalogs-backend`.
