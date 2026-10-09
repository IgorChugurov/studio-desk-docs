# API contract — Halls

**Scope:** halls and their files in the studio admin API. The file read address is on the API host, not under `/api/studio`.
**Derived from:** `catalogs.md`, `catalogs-backend-brief.md`, `../../03-architecture/files.md`, `../../03-architecture/api-conventions.md`, `api-contract.md`

## Essay — what this is

This is the contract between the studio admin backend and its frontend for halls. It names endpoints, payloads, and error codes. It does not restate product behaviour (`catalogs.md`), the common error and list formats (`api-conventions.md`), or sessions (`../../03-architecture/authentication.md`).

Phase 2 of `catalogs-backend-brief.md` starts after the owner agrees this contract and the table structure below. Screen texts on success are «Hall added» and «Hall updated». Field texts are «Enter a name», «Enter an address», «Enter a YouTube or Vimeo link».

## Essay — general

- **Base URL:** `https://api.studio-desk.axondigital.xyz/api/studio`. Paths below are relative to it. A hall file is a public static file at `/files/…` on the API host.
- **Authorization:** every endpoint requires `Authorization: Bearer <accessToken>` of a studio session. A missing, invalid, or revoked token, and a platform session, return `401 UNAUTHORIZED`.
- **Section:** `catalogs`. The owner and an administrator pass. A trainer and an accountant receive `403 FORBIDDEN`. The studio is the one in the session.
- **Formats:** IDs are UUIDs. Dates are ISO 8601 strings in UTC. Text fields are trimmed. Errors and lists follow `api-conventions.md`. Empty optional text is stored as `null`, never as `""`.
- **A hall of another studio** is `404 NOT_FOUND` on read, update, upload, delete, and reorder. The same answer is used when the id does not exist.

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
  "videoLink": null,
  "images": [],
  "createdAt": "2026-10-08T18:00:00Z",
  "updatedAt": "2026-10-08T18:00:00Z"
}
```

`images` is every file of this hall, sorted by `index` ascending. The YouTube or Vimeo link is `videoLink`. It is not an item of `images`.

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
  "videoLink": ""
}
```

`name` and `address` are required. `videoLink` may be omitted, empty, or `null`; that stores `null`. The body has no files. A file field is ignored because unknown fields are dropped (`api-conventions.md`).

`201`: `Hall` with `images: []`. Screen text: «Hall added».

| Status | `code` | `field` | When |
|---|---|---|---|
| 400 | `VALIDATION_ERROR` (`REQUIRED`) | `name`, `address` | missing, `null`, or blank |
| 400 | `VALIDATION_ERROR` (`INVALID_FORMAT`) | `videoLink` | non-empty and not a YouTube or Vimeo URL as above |

Field errors of one request come together.

### `GET /halls/{id}` — one hall

`200`: `Hall`. `404 NOT_FOUND` when this studio has no such hall.

### `PATCH /halls/{id}` — update name, address, video link

Body: any subset of `name`, `address`, `videoLink`. A field that is not sent is not changed. `name` and `address` cannot be cleared. `videoLink` sent as `null` or blank becomes `null`.

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

## Essay — error codes and screen texts

The frontend picks the text by `code` and `field`. Fixed texts are from `catalogs.md`.

| `code` | Text |
|---|---|
| `VALIDATION_ERROR` (`REQUIRED`) on `name` | Enter a name |
| `VALIDATION_ERROR` (`REQUIRED`) on `address` | Enter an address |
| `VALIDATION_ERROR` (`INVALID_FORMAT`) on `videoLink` | Enter a YouTube or Vimeo link |
| `FORBIDDEN` on this section | You don't have access to this page |
| any other error of an action or of loading | Something went wrong. Try again |

File type and size have no fixed screen text in the canon. The frontend uses the common failure text until the canon names one.

## Essay — how it connects

- Product behaviour: `catalogs.md`.
- Requirements and phases: `catalogs-backend-brief.md`.
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
- [ ] There is no endpoint that deletes a hall

## Open questions

None. Image and video types and the maximum size are decided: jpeg, png, webp, and gif up to 10 MB; mp4 and webm up to 100 MB.
