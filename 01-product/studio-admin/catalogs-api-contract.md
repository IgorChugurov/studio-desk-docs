# API contract — Halls

**Scope:** halls, trainers, and class types, and their files, in the studio admin API. A file address is on the API host, not under `/api/studio`.
**Derived from:** `catalogs.md`, `catalogs-backend-brief.md`, `catalogs-trainers-backend-brief.md`, `../../03-architecture/files.md`, `../../03-architecture/api-conventions.md`, `api-contract.md`

## Essay — what this is

This is the contract between the studio admin backend and its frontend for halls. It names endpoints, payloads, and error codes. It does not restate product behaviour (`catalogs.md`), the common error and list formats (`api-conventions.md`), or sessions (`../../03-architecture/authentication.md`).

Halls below are the agreed contract. The owner agreed the extension from `catalogs-trainers-backend-brief.md`, including the trainer and class-type tables, on 2026-10-09.

Screen texts on success are «Hall added», «Hall updated», «Trainer added», «Trainer updated», «Class type added», and «Class type updated». Field texts include «Enter a name», «Enter an address», «Enter a YouTube or Vimeo link», «Enter an Instagram link», and «Enter a TikTok link».

## Essay — general

- **Base URL:** `https://api.studio-desk.axondigital.xyz/api/studio`. Paths below are relative to it. A hall file is a public static file at `/files/…` on the API host.
- **Authorization:** every endpoint requires `Authorization: Bearer <accessToken>` of a studio session. A missing, invalid, or revoked token, and a platform session, return `401 UNAUTHORIZED`.
- **Section:** `catalogs`. The owner and an administrator pass. A trainer and an accountant receive `403 FORBIDDEN`. The studio is the one in the session.
- **Formats:** IDs are UUIDs. Dates are ISO 8601 strings in UTC. Text fields are trimmed. Errors and lists follow `api-conventions.md`. Empty optional text is stored as `null`, never as `""`.
- **A record of another studio** is `404 NOT_FOUND` on read, update, upload, delete, and reorder. The same answer is used when the id does not exist.

### Objects

`HallFile`:

```json
{
  "id": "a31c…",
  "index": 0,
  "kind": "image",
  "contentType": "image/jpeg",
  "url": "/files/halls/9c2e…/a31c….jpg"
}
```

- `kind` is `image` or `video`, taken from `contentType`.
- `url` is the public address of the file on the API host: `/files/halls/{hallId}/{fileId}` plus `.jpg`, `.png`, `.webp`, `.gif`, `.mp4`, or `.webm`. The same value is stored on the file row. No session is required to open it.
- `index` is an integer, zero or greater.

`Hall`:

```json
{
  "id": "9c2e…",
  "name": "Main hall",
  "address": "Hlavná 1, Bratislava",
  "description": null,
  "videoLink": null,
  "images": [],
  "createdAt": "2026-10-08T18:00:00Z",
  "updatedAt": "2026-10-08T18:00:00Z"
}
```

`images` is every file of this hall, sorted by `index` ascending. The YouTube or Vimeo link is `videoLink`. It is not an item of `images`. `description` is optional text, or `null`. Search does not look at it.

`videoLink` is `null` or an `http` or `https` URL of a YouTube or Vimeo video:

- host `youtube.com`, `www.youtube.com`, or `m.youtube.com`, with a path `/watch` and a non-empty `v`, or a path `/embed/{id}`, `/shorts/{id}`, or `/live/{id}`;
- host `youtu.be` with a non-empty path id;
- host `vimeo.com` or `www.vimeo.com` with a numeric video id as the path;
- host `player.vimeo.com` with a path `/video/{id}` and a numeric id.

Any other non-empty value is rejected. The stored text is the trimmed URL the client sent.

## Essay — halls

### `GET /halls` — list

Query, in addition to the common list parameters (`api-conventions.md`):

- `search`: case-insensitive substring of `name`. An empty search returns every hall of the studio.
- `sortBy`: `createdAt` (default) or `name`.
- `order`: `DESC` (default) or `ASC`.

`200`: `{ "items": Hall[], "meta": … }`. Each item includes its `images`. The owner is not a row. Two halls may have the same name.

### `POST /halls` — create

Body:

```json
{
  "name": "Main hall",
  "address": "Hlavná 1, Bratislava",
  "description": "",
  "videoLink": ""
}
```

`name` and `address` are required. `description` and `videoLink` may be omitted, empty, or `null`; that stores `null`. The body has no files. A file field is ignored because unknown fields are dropped (`api-conventions.md`).

`201`: `Hall` with `images: []`. Screen text: «Hall added».

| Status | `code` | `field` | When |
|---|---|---|---|
| 400 | `VALIDATION_ERROR` (`REQUIRED`) | `name`, `address` | missing, `null`, or blank |
| 400 | `VALIDATION_ERROR` (`INVALID_FORMAT`) | `videoLink` | non-empty and not a YouTube or Vimeo URL as above |

Field errors of one request come together.

### `GET /halls/{id}` — one hall

`200`: `Hall`. `404 NOT_FOUND` when this studio has no such hall.

### `PATCH /halls/{id}` — update name, address, description, video link

Body: any subset of `name`, `address`, `description`, `videoLink`. A field that is not sent is not changed. `name` and `address` cannot be cleared. `description` and `videoLink` sent as `null` or blank become `null`.

`200`: `Hall`. Indexes in `images` are unchanged. Screen text: «Hall updated».

Errors: the field errors of create for the fields that were sent, and `404 NOT_FOUND`.

There is no `DELETE /halls/{id}`.

## Essay — files of a hall

Upload, delete, and reorder exist only for a hall that already exists. They are refused with `404 NOT_FOUND` when this studio has no such hall.

Accepted types and the maximum size, decided by the owner on 2026-10-08:

| Kind | `contentType` | Maximum size |
|---|---|---|
| image | `image/jpeg`, `image/png`, `image/webp`, `image/gif` | 10 MB |
| video | `video/mp4`, `video/webm` | 100 MB |

The type is taken from the file bytes, not from the browser's declared type. A file that is neither an accepted image nor an accepted video is rejected. Zero-byte files are rejected.

### `POST /halls/{id}/files` — upload

`multipart/form-data`. The field name is `files`. One request may contain several parts. Parts receive consecutive indexes in part order. The first new index equals the number of files on the hall at the start of the request. The hall fields do not change.

If any part is rejected, nothing from this request is stored and indexes already on the hall stay as they were.

`201`: `Hall`, with `images` sorted by index.

| Status | `code` | `field` | When |
|---|---|---|---|
| 400 | `VALIDATION_ERROR` (`REQUIRED`) | `files` | no part was sent |
| 400 | `VALIDATION_ERROR` (`INVALID_VALUE`) | `files` | a part is not an accepted image or video |
| 400 | `VALIDATION_ERROR` (`TOO_BIG`) | `files` | a part is over its kind's maximum |
| 400 | `VALIDATION_ERROR` (`TOO_SMALL`) | `files` | a part is empty |
| 404 | `NOT_FOUND` | — | this studio has no such hall |

### `DELETE /halls/{id}/files/{fileId}` — delete one file

Deletes that file's record and its bytes. The remaining files of this hall are rewritten to indexes `0…n-1` in their previous order.

`200`: `Hall`. `404 NOT_FOUND` when the hall is not in this studio, or the file is not on this hall.

### `PUT /halls/{id}/files/order` — reorder

Body:

```json
{ "fileIds": ["a31c…", "b42d…"] }
```

`fileIds` is every file id of this hall, each once, in the new order. The request rewrites indexes from zero in that order. It does not add or remove files.

`200`: `Hall`.

| Status | `code` | `field` | When |
|---|---|---|---|
| 400 | `VALIDATION_ERROR` (`INVALID`) | `fileIds` | not exactly the current set of this hall's files |
| 404 | `NOT_FOUND` | — | this studio has no such hall |

Updating the hall with `PATCH /halls/{id}` does not change indexes.

## Essay — trainers

Same access, list shape, file rules, and file types as halls. Search is a case-insensitive substring of `name` only. Two trainers of one studio may share a name. There is no `DELETE /trainers/{id}`.

`Trainer`:

```json
{
  "id": "4a1b…",
  "name": "Anna",
  "description": null,
  "instagram": null,
  "tiktok": null,
  "images": [],
  "createdAt": "2026-10-09T16:00:00Z",
  "updatedAt": "2026-10-09T16:00:00Z"
}
```

`images` uses the same shape as a hall file. `url` is `/files/trainers/{trainerId}/{fileId}` plus the type extension.

`instagram` is `null` or an `http` or `https` URL whose host is `instagram.com` or `www.instagram.com` and whose path is one username. The username is letters, digits, `.` and `_`, and it is not empty. A trailing slash is allowed. Any other non-empty value is rejected. The stored text is the trimmed URL the client sent.

`tiktok` is `null` or an `http` or `https` URL whose host is `tiktok.com` or `www.tiktok.com` and whose path is `/@` plus one username of the same kind. A trailing slash is allowed. Any other non-empty value is rejected. The stored text is the trimmed URL the client sent.

### `GET /trainers` — list

Query: the same as `GET /halls`, with `search` on `name`. `200`: `{ "items": Trainer[], "meta": … }`. Each item includes its `images`.

### `POST /trainers` — create

Body:

```json
{
  "name": "Anna",
  "description": "",
  "instagram": "",
  "tiktok": ""
}
```

`name` is required. `description`, `instagram`, and `tiktok` may be omitted, empty, or `null`; that stores `null`. The body has no files.

`201`: `Trainer` with `images: []`. Screen text: «Trainer added».

| Status | `code` | `field` | When |
|---|---|---|---|
| 400 | `VALIDATION_ERROR` (`REQUIRED`) | `name` | missing, `null`, or blank |
| 400 | `VALIDATION_ERROR` (`INVALID_FORMAT`) | `instagram` | non-empty and not an Instagram URL as above |
| 400 | `VALIDATION_ERROR` (`INVALID_FORMAT`) | `tiktok` | non-empty and not a TikTok URL as above |

Field errors of one request come together.

### `GET /trainers/{id}` — one trainer

`200`: `Trainer`. `404 NOT_FOUND` when this studio has no such trainer.

### `PATCH /trainers/{id}` — update

Body: any subset of `name`, `description`, `instagram`, `tiktok`. A field that is not sent is not changed. `name` cannot be cleared. The optional fields sent as `null` or blank become `null`.

`200`: `Trainer`. Indexes in `images` are unchanged. Screen text: «Trainer updated». Errors: the field errors of create for the fields that were sent, and `404 NOT_FOUND`.

### Files

`POST /trainers/{id}/files`, `DELETE /trainers/{id}/files/{fileId}`, and `PUT /trainers/{id}/files/order` follow the hall file endpoints. The response is `Trainer`. A missing trainer or a file that is not on this trainer is `404 NOT_FOUND`.

## Essay — class types

Same access, list shape, file rules, and file types as halls. Search is a case-insensitive substring of `name` only. Two class types of one studio may share a name. There is no `DELETE /class-types/{id}`. There is no capacity, duration, price, or group-or-individual flag.

`ClassType`:

```json
{
  "id": "7c3d…",
  "name": "Yoga",
  "description": null,
  "images": [],
  "createdAt": "2026-10-09T16:00:00Z",
  "updatedAt": "2026-10-09T16:00:00Z"
}
```

`url` of a file is `/files/class-types/{classTypeId}/{fileId}` plus the type extension.

### `GET /class-types` — list

Query: the same as `GET /halls`, with `search` on `name`. `200`: `{ "items": ClassType[], "meta": … }`. Each item includes its `images`.

### `POST /class-types` — create

Body: `{ "name": "Yoga", "description": "" }`. `name` is required. `description` may be omitted, empty, or `null`; that stores `null`. The body has no files.

`201`: `ClassType` with `images: []`. Screen text: «Class type added». An empty name is `400` `VALIDATION_ERROR` (`REQUIRED`) on `name`.

### `GET /class-types/{id}` — one class type

`200`: `ClassType`. `404 NOT_FOUND` when this studio has no such class type.

### `PATCH /class-types/{id}` — update

Body: any subset of `name` and `description`. A field that is not sent is not changed. `name` cannot be cleared. `description` sent as `null` or blank becomes `null`.

`200`: `ClassType`. Indexes in `images` are unchanged. Screen text: «Class type updated». Errors: the name error of create when `name` was sent, and `404 NOT_FOUND`.

### Files

`POST /class-types/{id}/files`, `DELETE /class-types/{id}/files/{fileId}`, and `PUT /class-types/{id}/files/order` follow the hall file endpoints. The response is `ClassType`. A missing class type or a file that is not on this class type is `404 NOT_FOUND`.

## Essay — reading a file

There is no hall-file endpoint under `/api/studio`. The bytes are the static file at `url`. Nest serves the storage directory at `/files`. A missing file is `404`. Opening it does not require a session. Closing a file behind a session is a later topic.

## Essay — tables

Agreed by the owner on 2026-10-08. This is the structure phase 2 stores. It is not the HTTP contract.

`hall`

| Column | |
|---|---|
| `id` | uuid, primary key |
| `studio_id` | uuid, not null, references `studio.id` |
| `name` | text, not null, trimmed, not empty |
| `address` | text, not null, trimmed, not empty |
| `video_link` | text, null |
| `created_at` | timestamptz, not null |
| `updated_at` | timestamptz, not null |

Index on `studio_id`. No unique constraint on `name`. No delete of a row in this topic.

`hall_file`

| Column | |
|---|---|
| `id` | uuid, primary key |
| `hall_id` | uuid, not null, references `hall.id` |
| `storage_path` | text, not null; the public address `/files/halls/{hallId}/{id}.ext`, also returned as `url` |
| `content_type` | text, not null; one of the accepted types |
| `index` | integer, not null, zero or greater |
| `created_at` | timestamptz, not null |

Unique `(hall_id, index)`. The bytes live on disk, not in the database. The studio is reached through `hall`. It is not copied onto `hall_file`.

The extension adds a column `description` (`text`, null) on `hall`. The owner agreed it on 2026-10-09.

These tables were agreed by the owner on 2026-10-09.

`trainer`: `id`, `studio_id`, `name` (required), `description`, `instagram`, `tiktok` (the last three null), `created_at`, `updated_at`. Index on `studio_id`. No unique constraint on `name`. No delete of a row in this topic.

`trainer_file`: the same columns as `hall_file`, with `trainer_id` instead of `hall_id`. `storage_path` is `/files/trainers/{trainerId}/{id}.ext`. Unique `(trainer_id, index)`.

`class_type`: `id`, `studio_id`, `name` (required), `description` (null), `created_at`, `updated_at`. Index on `studio_id`. No unique constraint on `name`. No delete of a row in this topic.

`class_type_file`: the same columns as `hall_file`, with `class_type_id` instead of `hall_id`. `storage_path` is `/files/class-types/{classTypeId}/{id}.ext`. Unique `(class_type_id, index)`.

## Essay — error codes and screen texts

The frontend picks the text by `code` and `field`. Fixed texts are from `catalogs.md`.

| `code` | Text |
|---|---|
| `VALIDATION_ERROR` (`REQUIRED`) on `name` | Enter a name |
| `VALIDATION_ERROR` (`REQUIRED`) on `address` | Enter an address |
| `VALIDATION_ERROR` (`INVALID_FORMAT`) on `videoLink` | Enter a YouTube or Vimeo link |
| `VALIDATION_ERROR` (`INVALID_FORMAT`) on `instagram` | Enter an Instagram link |
| `VALIDATION_ERROR` (`INVALID_FORMAT`) on `tiktok` | Enter a TikTok link |
| `FORBIDDEN` on this section | You don't have access to this page |
| any other error of an action or of loading | Something went wrong. Try again |

File type and size have no fixed screen text in the canon. The frontend uses the common failure text until the canon names one.

## Essay — how it connects

- Product behaviour: `catalogs.md`.
- Requirements and phases: `catalogs-backend-brief.md` and `catalogs-trainers-backend-brief.md`.
- Where the bytes live and the read address: `../../03-architecture/files.md`.
- Errors, lists, OpenAPI: `../../03-architecture/api-conventions.md`.
- Sessions: `../../03-architecture/authentication.md` and `api-contract.md`.
- Decisions recorded while writing this contract: `catalogs-backend-brief-feedback.md`.

## Agent — rules

- Every hall endpoint appears in the OpenAPI document of the studio API. The static file address is not an API endpoint.
- Renaming a `code` or a field is a contract change.
- Create does not store files. `PATCH` of hall fields does not change indexes.
- A new file's index equals the count of files on that hall. One upload of several files uses consecutive indexes in part order. Delete and reorder rewrite the remaining indexes from zero.
- `images` is sorted by index.
- A trainer, an accountant, and a hall of another studio are refused on the hall endpoints. The file at `url` is public.

## Agent — acceptance checklist

- [ ] List, create, read, and update match the paths, bodies, and status codes above
- [ ] A trainer and an accountant receive `403`. A hall of another studio receives `404`
- [ ] Empty name and empty address are refused. An empty video link is stored as `null`. A non-YouTube, non-Vimeo link is refused
- [ ] Two halls of one studio may share a name
- [ ] Search is a case-insensitive substring of the name
- [ ] Create does not store files
- [ ] The first file has index 0. A second file has index 1. Three files in one upload receive 0, 1, 2 in part order
- [ ] Deleting the middle file leaves indexes 0 and 1. Reorder persists the new order. Updating the name does not change indexes
- [ ] `images` comes back sorted by index
- [ ] An image and a video can both be stored
- [ ] `url` is `/files/halls/{hallId}/{fileId}` plus the type's extension, and that address returns the bytes without a session
- [ ] There is no endpoint that deletes a hall, a trainer, or a class type
- [ ] A hall description may be empty and is stored as `null`. Search does not use it
- [ ] A trainer has `name`, `description`, `instagram`, and `tiktok`. A class type has `name` and `description`. Empty optional text is `null`
- [ ] A non-empty Instagram or TikTok value that is not that network is refused
- [ ] Trainer and class-type files follow the hall index rules. Their public paths are `/files/trainers/…` and `/files/class-types/…`

## Open questions

None. Image and video types and the maximum size are decided: jpeg, png, webp, and gif up to 10 MB; mp4 and webm up to 100 MB.
