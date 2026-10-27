# Backend brief feedback — Trainers, class types, hall description

The owner decides canon. This file records answers and decisions for `catalogs-trainers-backend-brief.md`. The owner agreed the contract extension and the tables on 2026-10-09.

1. **Open questions.** None.

2. **Deviations from the brief.** None.

3. **Behaviour the backend decided, because neither canon nor the brief defined it.**
   - An Instagram address is an `http` or `https` URL on `instagram.com` or `www.instagram.com` with one username in the path. A TikTok address is an `http` or `https` URL on `tiktok.com` or `www.tiktok.com` with a path `/@` plus one username. The username is letters, digits, `.` and `_`. The trimmed URL is stored as sent.
   - Endpoints follow halls: `/trainers` and `/class-types`, with the same list, create, read, update, upload, delete-file, and reorder shapes.
   - List sort is `createdAt` descending by default. `name` is also allowed.
   - Tables agreed and migrated: `description` on `hall`; `trainer`, `trainer_file`, `class_type`, `class_type_file`.

4. **What the implementer believes changes the canon.** Nothing. The public file paths are the ones already fixed in the brief.

5. **Documents updated.** `catalogs-api-contract.md`, this file, and work item `2026-10-09-studio-admin-catalogs-trainers-backend`.
