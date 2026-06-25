# EpubReader Sync Server — Client Integration Guide

> **Audience**: developers / agents building Windows desktop and Web (PWA) clients that share the **same self-hosted sync server** with the existing Android client.
>
> **Last updated**: 2026-06-25. After Android Phase D-2.
>
> Everything below is **production-ready and exercised**. Two physical devices (a real Android phone + a redroid emulator) have completed full end-to-end sync. The server hosts 2 books / 2 progress records as of this writing; the integration model is proven, not theoretical.

---

## 0. Quickstart (smoke in 90 seconds)

```bash
# 1. Health (anonymous)
curl https://us.guyii.net/healthz
# → {"status":"ok"}

# 2. Login as test admin (DEV password; change for prod)
RESP=$(curl -sS -X POST https://us.guyii.net/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"admin","password":"<your-password>"}')
ACCESS=$(echo "$RESP" | jq -r .accessToken)
REFRESH=$(echo "$RESP" | jq -r .refreshToken)
USER_ID=$(echo "$RESP" | jq -r .userId)

# 3. Register your client device (any UUID you generate — keep it stable)
DEV_ID=$(uuidgen | tr A-Z a-z)
curl -sS -X POST https://us.guyii.net/api/v1/auth/devices \
  -H "Authorization: Bearer $ACCESS" \
  -H 'Content-Type: application/json' \
  -d "{\"deviceId\":\"$DEV_ID\",\"name\":\"My Windows PC\",\"platform\":\"WIN\"}"

# 4. Pull everything (first sync uses since=0)
curl -sS -H "Authorization: Bearer $ACCESS" -H "X-Device-Id: $DEV_ID" \
  "https://us.guyii.net/api/v1/books?since=0" | jq

# 5. Download an EPUB file (use the id from step 4)
curl -sS -H "Authorization: Bearer $ACCESS" -H "X-Device-Id: $DEV_ID" \
  -o /tmp/book.epub \
  "https://us.guyii.net/api/v1/books/<id>/file"
```

If all 5 work, you have a viable client foundation.

---

## 1. System overview

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│ Android      │    │ Windows      │    │ Web (PWA)    │    │ iOS (future) │
│ (Kotlin)     │    │ (you)        │    │ (you)        │    │              │
└──────┬───────┘    └──────┬───────┘    └──────┬───────┘    └──────┬───────┘
       │                   │                   │                   │
       └───────────────────┴───────────────────┴───────────────────┘
                                   │
                              HTTPS only
                                   │
                          ┌────────▼────────┐
                          │ us.guyii.net    │
                          │  (Caddy :443)   │
                          └────────┬────────┘
                                   │
                          ┌────────▼────────────┐
                          │ epub-reader-server  │
                          │ (Kotlin/Ktor :8090) │
                          │ Docker container    │
                          └────────┬────────────┘
                                   │
                          ┌────────▼────────────┐
                          │ SQLite + /data/books│
                          │ (Linode US VM)      │
                          └─────────────────────┘
```

### Design philosophy

- **Local-first**: every client keeps a complete copy of all the user's data. Network failure does not block the user. The local database is the user's data; the server is a coordination point.
- **Server is source of truth for conflicts**: when two clients disagree, the server decides (LWW by `updatedAt`, tie-broken by `deviceId` lexicographic).
- **Idempotent server**: re-sending the same content collapses cleanly. Server side dedup by SHA-256 means re-uploading a known book returns 200 (not 201) with the existing row.
- **No transparent retries on the wire**: the protocol does not assume a working WebSocket / push channel. Clients pull on a schedule; the server has no ability to push to a client that isn't actively connected.

### Tech stack (server)

| Layer | Choice |
|---|---|
| Language | Kotlin 1.9 |
| HTTP framework | Ktor 2.3 |
| ORM | Exposed 0.46 |
| Database | SQLite (via `xerial:sqlite-jdbc`) |
| JSON | kotlinx.serialization |
| Password hashing | bcrypt cost 12 |
| Auth | JWT HS256 (15 min access + 30 day refresh, refresh rotated on every use) |
| Container | distroless-ish Eclipse Temurin 17 |
| Reverse proxy | Caddy (TLS terminator) |

### Hosting

- **Production URL**: `https://us.guyii.net` (TLS via Let's Encrypt through Caddy)
- **VM**: Linode US-east, Ubuntu 24.04, public IPv4 `50.116.41.49`
- **Container**: bound to `127.0.0.1:8090`; Caddy reverse-proxies `/healthz` + `/api/*` paths to it
- **Disk**: `/opt/epub-reader/data/` (SQLite + EPUB files)
- **Source code**: `/opt/epub-reader/source/` (mirrors `https://github.com/guyiicn/epub_reader_server`)

---

## 2. Authentication

### 2.1 Login

```http
POST /api/v1/auth/login
Content-Type: application/json

{ "username": "admin", "password": "<your-password>" }
```

Response:
```json
{
  "userId":  "bd1ebb49-bf45-416b-8b78-aaa8f35ccd51",
  "accessToken":  "eyJhbGciOi...",   // 15-min HS256 JWT
  "refreshToken": "eyJhbGciOi...",   // 30-day HS256 JWT, single-use
  "accessTokenExpiresAt": 1782353391421
}
```

Errors: `401 invalid credentials`. After 5 failed attempts in 15 min, exponential backoff (1–30 s). After 10 failed attempts, account is locked for 1 hour (`423 account_locked`).

### 2.2 Refresh

```http
POST /api/v1/auth/refresh
Content-Type: application/json

{ "refreshToken": "<the refresh JWT>" }
```

Returns the same shape as login. **The old refresh token is revoked.** Reusing a revoked refresh token is detected as a replay attack:
- Server response: `401 refresh_replay`
- Server side effect: **all** refresh tokens for the affected user are revoked. The user must log in again from every device.

→ Clients **must** treat refresh tokens as one-shot and not retry-refresh on a half-failed previous attempt.

### 2.3 Device registration

```http
POST /api/v1/auth/devices
Authorization: Bearer <access>
Content-Type: application/json

{
  "deviceId":  "<UUID you generated>",
  "name":      "Guyi's Surface Pro",
  "platform":  "WIN"     // one of: ANDROID, IOS, WEB, WIN, MAC
}
```

- The `deviceId` is **client-generated** and must be **stable** across launches of the same client install. Reinstalling the client = new `deviceId` (new "device").
- Idempotent: calling it again with the same `deviceId` is fine — updates `lastSeenAt` and `name`.
- Must be a UUID (lowercase, with dashes, e.g. `0d3b6f8c-19b3-4c50-8ad4-37c0cf7f7e88`).

### 2.4 Logout

```http
POST /api/v1/auth/logout
Authorization: Bearer <access>
Content-Type: application/json

{ "refreshToken": "<the current refresh token>" }
```

Revokes that refresh token only. The access token stays valid until it expires (15 min) — there's no central revocation list for access tokens. **Treat logout as best-effort.**

### 2.5 Headers required on every business call

```
Authorization: Bearer <access>
X-Device-Id:   <the deviceId you registered>
```

Missing `X-Device-Id` → `400 missing_device_id`. Unknown `X-Device-Id` (not registered for this user) → `403 unknown_device`. **Both header values are required on every endpoint under `/api/v1/{books,progress,bookmarks,annotations}`** — the `/auth/*` family and `/healthz` are the only exceptions.

---

## 3. Data model

### 3.1 Book

| Field | Type | Notes |
|---|---|---|
| `id` | UUID v7 (string) | **Server-assigned** on first upload. On dedup the server may return a different id than the client suggested. |
| `contentHash` | hex SHA-256 (string) | Hash of the EPUB bytes. Server computes this on upload. **Dedup key.** |
| `title` | string | From client metadata |
| `author` | string | From client metadata |
| `format` | enum | `EPUB`, `MOBI`, `MARKDOWN`, `TXT` |
| `fileSize` | long | Bytes |
| `totalChapters` | int | From client metadata |
| `coverHash` | hex SHA-256, nullable | If server extracted/stored a cover |
| `addedAt` | long (ms epoch) | Server clock |
| `addedBy` | string (deviceId) | First device to add |
| `updatedAt` | long (ms epoch) | LWW comparator |
| `deletedAt` | long (ms epoch), nullable | Tombstone marker. 30-day server-side GC removes the file later. |

### 3.2 Progress

One row per `(userId, bookId)`. Last-write-wins on `updatedAt`.

| Field | Type | Notes |
|---|---|---|
| `bookId` | UUID | FK to books |
| `locator` | string | Opaque to server. Android stores a Readium `Locator.toJSON()` string. **Web/Win can use whatever locator format they want, but must accept the value as opaque if it arrived from a different client.** See §7.1 for the practical rule. |
| `totalProgression` | double (0..1) | Useful for UI display when locator format is foreign |
| `updatedAt` | long (ms epoch) | Client clock |
| `updatedBy` | string (deviceId) | Tie-breaker for LWW |

### 3.3 Bookmark

| Field | Type | Notes |
|---|---|---|
| `id` | UUID | **Client-assigned**. Idempotent POST: re-sending the same id returns 200 + existing row, not 409. |
| `bookId` | UUID | |
| `locator` | string | Same opacity rule as Progress |
| `chapterTitle` | string, nullable | For UX |
| `note` | string, nullable | User text |
| `color` | hex color (e.g. `#FFC107`) | UI hint |
| `createdAt` | long (ms epoch) | Client clock |
| `createdBy` | string (deviceId) | |
| `updatedAt` | long (ms epoch) | Bumped on edit/delete |
| `deletedAt` | long (ms epoch), nullable | Tombstone |

### 3.4 Annotation (highlight + note)

Same shape as Bookmark plus:

| Field | Type | Notes |
|---|---|---|
| `selectedText` | string | The highlighted passage |

---

## 4. REST API

Base: `https://us.guyii.net/api/v1`. All bodies JSON unless noted. All endpoints require `Authorization` + `X-Device-Id` except `/auth/*` and `/healthz`.

### 4.1 Healthcheck

- `GET /healthz` (no auth) → `{ "status": "ok" }` 200
- `GET /healthz` (with auth) → full payload with `db`, `disk`, `schemaVersion`, `uptimeMs`, `version`

### 4.2 Incremental list (the `?since=` pattern)

Used on every business list endpoint:

```
GET /books?since=<ts>&limit=<n>
GET /progress?since=<ts>&limit=<n>
GET /bookmarks?since=<ts>&limit=<n>
GET /annotations?since=<ts>&limit=<n>
```

- `since`: ms epoch. Returns rows where `updatedAt > since`. On first sync, pass `0`.
- `limit`: max 500. Silently clamped to that range.
- Response body: JSON array.
- **`X-Sync-Cursor` response header**: the `updatedAt` of the most recent row in this response, or `now()` if the response was empty. **Save this and pass it as `since` on the next call** to do incremental sync.

Tombstones (rows with `deletedAt != null`) are included in the list — clients must remove them locally (or mark them tombstoned and propagate to other clients).

### 4.3 Books

#### `POST /books` — upload

`multipart/form-data` with two parts:

- `file`: the binary EPUB. Max 100 MB. Server validates ZIP magic + checks `mimetype` entry equals `application/epub+zip` + counts entries (max 10k) + bounds uncompressed total to 500 MB + rejects entries with `..` or absolute paths (zip-slip protection).
- `metadata`: JSON string with `{ title, author, format, totalChapters, addedAt }`.

Response:
- `201 Created`: new book row, full BookDto.
- `200 OK`: dedup hit. The body is the **existing** book row. The `id` field is the server's canonical id and may differ from anything the client picked. **Treat this as the authoritative id from now on.** If your local row had a different id, rewrite all your local `progress`/`bookmarks`/`annotations` to reference the new id.
- `413 Payload Too Large`: > 100 MB
- `422 Unprocessable Entity`: malformed EPUB

#### `GET /books?since=...`
List + tombstones, paginated. See §4.2.

#### `GET /books/{id}` → single book
404 if not yours.

#### `DELETE /books/{id}` → soft delete
204. Sets `deletedAt`. Server keeps the file for 30 days (per `SECURITY.md`), then GC's it.

#### `GET /books/{id}/file`
Stream the EPUB. `Content-Type: application/epub+zip`. **Range supported** (use `Range: bytes=N-` for resumable downloads, also for streaming readers that just want a slice).

#### `GET /books/{id}/cover`
Stream JPG or PNG. Server attempts cover extraction at upload time; some books may have no cover (returns 404).

### 4.4 Progress

#### `GET /progress?since=...`
Standard.

#### `PUT /progress/{bookId}`
Body:
```json
{
  "locator":          "...opaque...",
  "totalProgression": 0.1858736,
  "updatedAt":        1782359919234,
  "deviceId":         "<your X-Device-Id>"
}
```

LWW rules:
- If existing server row's `updatedAt < incoming.updatedAt` → accept. Return 200 + new row.
- If existing `updatedAt > incoming.updatedAt` → **conflict, server wins**. Return `409 Conflict` with body = the server's current row. **The client must overwrite its local copy with that.**
- If `updatedAt == incoming.updatedAt` and `incoming.deviceId < server.updatedBy` (lexicographic) → 409 same shape.

### 4.5 Bookmarks

#### `GET /bookmarks?since=...`
Standard.

#### `POST /bookmarks`
Body: full Bookmark row with **client-assigned `id`**.
- New row: 201
- Same `id` already exists (for the same user, e.g. retry of a previous request): 200, body is the existing row. **Use this for safe retries.**
- Same `id` already exists for a different user: 400 `id_collision`. (Client bug — pick a fresh UUID.)

#### `DELETE /bookmarks/{id}`
204. Soft delete; row reappears in subsequent `?since` lists with `deletedAt` set.

### 4.6 Annotations

Mirror of Bookmarks. Same endpoints, same semantics. `selectedText` is the highlighted passage; `note` is the optional user note.

### 4.7 Error format

RFC 7807-style Problem JSON on all non-2xx:

```json
{
  "type":     "about:blank",
  "title":    "Unauthorized",
  "status":   401,
  "detail":   "invalid credentials",
  "instance": "/api/v1/auth/login"
}
```

The `title` is stable and short — use it for branching client logic. The `detail` is human-readable; safe to show users.

### 4.8 OpenAPI spec

Authoritative machine-readable contract lives at:

```
/home/guyi/code/epub_android/docs/server/openapi.yaml
```

Or via GitHub: <https://github.com/guyiicn/epub_android/blob/main/docs/server/openapi.yaml>

Schema bumps to `1.0.1` when X-Device-Id became a documented requirement (it had been undocumented but enforced since Phase B). Generate client stubs from this — don't hand-roll DTOs.

---

## 5. Sync state machine

This is the **most important section.** Get it wrong and your client appears to work but actually drops data silently — we hit this exact bug in the Android client three times before getting the model right.

### 5.1 Book sync state (derived, never stored)

Every book in the client's local DB has a sync state computed from two hashes:

| State | Condition | Meaning |
|---|---|---|
| **SYNCED** | `contentHash == serverHash` | In sync with server |
| **LOCAL_ONLY** | `serverHash IS NULL` | Never uploaded — push it |
| **REMOTE_ONLY** | local file missing (or never downloaded) | Known to server, not in local FS |
| **OPTED_OUT** | user flagged this book "do not sync" | Skip on push, ignore inbound tombstones |
| **CONFLICT** | both hashes set, `contentHash != serverHash` | Local file was modified after upload. Reupload (server dedups, no harm) or surface to user. Rare. |

### 5.2 Why two hashes (`contentHash` vs `serverHash`)

This caused us three bugs. Internalize this distinction:

- `contentHash` = SHA-256 of the **local file bytes**. Computed by the client at import time. It is a fingerprint of "what's on disk right now". It exists regardless of server state.
- `serverHash` = SHA-256 the **server told us it stores** the last time we successfully uploaded or pulled this row. It exists only after a successful round-trip with the server.

You CANNOT use `contentHash IS NULL` to mean "needs upload" — your client may compute the hash at import. You CANNOT use a "last push timestamp" cursor to derive what to upload — a buggy push can advance the cursor without uploading anything and poison every existing book forever.

**The correct query** for "what books need to be uploaded":

```sql
SELECT * FROM books
WHERE deletedAt IS NULL
  AND remoteOnly = 0
  AND syncOptOut = 0
  AND (serverHash IS NULL OR contentHash != serverHash)
```

And after a successful upload, set `serverHash = dto.contentHash` on the local row.

### 5.3 Pull flow

```
For table in [books, progress, bookmarks, annotations]:
    since   = stored cursor for this table (or 0 first time)
    rows, X-Sync-Cursor = GET /<table>?since=<since>&limit=500
    apply(rows)
    if X-Sync-Cursor is set, save it as the new cursor for this table
```

**`apply()` semantics**:
- For each incoming row, upsert into local DB by `id`.
- If `deletedAt != null` (tombstone): apply tombstone (soft-delete the local row + delete the local file if it's a book).
- For Book rows specifically: **set local `serverHash = row.contentHash`**. The server told you about this row, so by definition it stores this hash.
- If your local Book row has no local file (you just pulled metadata), set `remoteOnly = true`.

Pull is **idempotent**: running it twice in a row is harmless (the second run sees an empty result because the cursor has advanced).

### 5.4 Push flow

```
# 1. Upload books that need it
toUpload = local Books where (serverHash IS NULL OR contentHash != serverHash)
                              AND deletedAt IS NULL AND remoteOnly = false AND syncOptOut = false
for book in toUpload:
    if book.pushAttempts >= MAX_PUSH_ATTEMPTS:  # default 5
        continue  # cool down; let user manually retry
    try:
        dto = POST /books with file + metadata
        local.update(book.id, serverHash=dto.contentHash, id=dto.id, pushAttempts=0, lastSyncError=null)
        if dto.id != book.id:
            # Server deduped. Rewrite progress/bookmarks/annotations FKs.
            ...
    catch e:
        local.update(book.id, pushAttempts++, lastSyncError=e.message)

# 2. Push deletes
toDelete = local Books where (deletedAt != null AND serverHash != null)
for book in toDelete:
    try:
        DELETE /books/{book.id}
        local.update(book.id, serverHash=null)  # server forgot it now
    catch ...

# 3. Push progress/bookmarks/annotations
# Simple model: send everything modified since the last successful push cursor.
# OR: also gate on per-row dirty flags. The reference client uses cursors.
for row in progressDao.getChangedSince(lastPushAt):
    try:
        PUT /progress/{row.bookId}  with row body
    catch 409:
        # Server's version wins; copy it into local
        local.update(progress with server's row body)
```

**Crucial invariant**: only bump `lastPushAt` cursor if push **actually attempted at least one operation per table.** Better: don't use `lastPushAt` for books (use the `serverHash != contentHash` comparison instead) — only use it for progress/bookmarks/annotations where there's no equivalent confirmation field.

### 5.5 Triggers

When to call sync? Reference client uses:

| Event | Action | Throttle |
|---|---|---|
| Manual button | `sync()` | 5 s |
| App start | `sync()` | 60 s |
| Window returns from background | `sync()` | 60 s |
| User finishes reading session (book closed) | `push()` only, no pull | none |
| New book imported | `push()` for that book only | none |
| First successful login | `sync()` | none |

Web clients without lifecycle hooks can substitute "tab visibilitychange" for foreground. Windows: hook the main window's `Activated` / focus event.

### 5.6 Cursor reset on schema upgrade

Whenever you change your local DB schema in a way that affects "what is a dirty row," **wipe your per-table cursors**. Why: pre-upgrade your push code may have been buggy and your stored cursors are poisoned. Reference Android implementation:

```
suspend fun ensureSchemaVersion(currentVersion: Int) {
    val seen = dataStore.read("schema_version_seen") ?: 0
    if (seen < currentVersion) {
        clearAllCursors()
        dataStore.write("schema_version_seen", currentVersion)
    }
}
```

Call this at the start of every sync. It's a no-op except after a schema bump.

---

## 6. File upload / download specifics

### 6.1 Upload

Use streaming multipart. Don't `read-all-into-buffer + send` — a 50 MB book on a low-memory device will OOM.

Reference Kotlin (ktor-client):
```
val response = httpClient.submitFormWithBinaryData(
    url = "$baseUrl/api/v1/books",
    formData = formData {
        append("metadata", json.encodeToString(metadata), Headers.build {
            append(HttpHeaders.ContentType, "application/json")
        })
        append("file", InputProvider(size = file.length()) { file.inputStream().asInput() }, Headers.build {
            append(HttpHeaders.ContentDisposition, "filename=${file.name}")
        })
    },
)
```

### 6.2 Download

`GET /books/{id}/file` supports `Range: bytes=N-`. For Web (PWA), prefer the Streams API to write to IndexedDB / OPFS in chunks instead of a single `Blob` for very large books.

Save the file to whatever local store you use, hash it locally, and set `serverHash` on the local row from the server's BookDto (not from re-hashing — they're guaranteed equal but you save the CPU).

---

## 7. Edge cases and required behaviors

### 7.1 The locator opacity rule

`locator` strings in Progress, Bookmark, Annotation are **opaque to the server**. But they're NOT cross-platform-compatible by default — Android stores Readium `Locator.toJSON()`, which references EPUB resource paths in Readium's format.

**Pragmatic rule for Web/Win clients**:
1. Round-trip: a locator emitted by Client X must successfully open the right page when received by Client X again. (Easy — just don't transform it.)
2. Cross-client: when Client Y receives a locator from Client X, **try to parse it**. If it's something Y understands (e.g. Y also uses Readium's locator format), use it. If not, **fall back to `totalProgression`** — that's why we ship that field separately. Show "～19%" instead of "Chapter 7 paragraph 3" rather than failing.

Long-term, all clients should converge on the Readium spec for locators. It's well-defined.

### 7.2 Idempotent POST for bookmarks / annotations

Always client-generate the UUID. Always send it on POST. Re-sending the same id is safe (server returns 200 with the existing row instead of 201). **Don't generate a fresh UUID on retry** — that creates a duplicate.

### 7.3 ID rewrite on book dedup

When you `POST /books` and get `200 + existing row`, the server's `id` may not match what you had locally. Three cases:

| Scenario | Action |
|---|---|
| Local row's id == returned id | Just update metadata, mark serverHash |
| Local row's id != returned id, **local id has no dependent rows** | Update the row's id to the server's id. Done. |
| Local row's id != returned id, **local progress/bookmarks/annotations exist for this id** | UPDATE those tables to point at the new id BEFORE deleting the old book row. |

Reference implementation: `SyncEngine.uploadBookInternal` in `/home/guyi/code/epub_android/app/src/main/java/com/guyi/reader/sync/engine/SyncEngine.kt`.

### 7.4 OPTED_OUT is local-only state

When a user toggles "don't sync this book":
- Local: `syncOptOut = true`
- Server: never told. The server doesn't know this state exists.

This means: even if another device deletes the book server-side, an OPTED_OUT local copy stays. That's intentional. The user explicitly chose not to participate in this book's sync.

If the user later un-toggles, the book re-joins the normal flow on next push (with state derived from `serverHash`).

### 7.5 Initial login + auto-sync UX

On the **first** login on a fresh client install, before kicking off the auto-sync, ask the user:

> "Auto-sync future imported books to the cloud?"

- "Yes" → store account-level `autoSyncEnabled = true`. Normal flow.
- "No" → store `autoSyncEnabled = false`. Mark **all currently local books as `syncOptOut = true`**, then sync (pulls remote books, pushes nothing).

We made this UX explicit because users were surprised when their private books got auto-uploaded.

### 7.6 Retry budget

After 5 consecutive push failures for the same book, **stop trying automatically**. Surface to the user (we use a per-book "重新上传" button in the detail sheet) and reset the counter when they explicitly retry. Don't burn bandwidth + battery on a broken file.

### 7.7 Token refresh on 401

Every business endpoint can return 401 if the access token expired. Reference flow:

```
1. Catch the 401
2. POST /auth/refresh with stored refresh token
3a. Success: store the new pair, retry the original request
3b. 401 refresh_replay or 401 invalid: clear all tokens, surface a "please log in" state to the UI
```

If you have multiple in-flight requests and one of them refreshes, **serialize the refresh** — don't trigger 5 refreshes in parallel for 5 concurrent 401s. Reference Android uses a Mutex.

---

## 8. Reference Android implementation map

Read these to crib semantics. All paths are under `/home/guyi/code/epub_android/`.

| Concept | File |
|---|---|
| Auth + session model | `app/src/main/java/com/guyi/reader/sync/credentials/SyncCredentialsStore.kt` |
| Ktor HTTP client + DTOs | `app/src/main/java/com/guyi/reader/sync/api/SyncApiClient.kt` + `Dtos.kt` |
| All push/pull semantics | `app/src/main/java/com/guyi/reader/sync/engine/SyncEngine.kt` |
| Per-table cursors | `app/src/main/java/com/guyi/reader/sync/SyncCursorStore.kt` |
| Trigger logic + throttling | `app/src/main/java/com/guyi/reader/sync/SyncTrigger.kt` |
| Sync state derivation | `app/src/main/java/com/guyi/reader/core/model/Book.kt` (look for `BookSyncState` enum + `syncState` extension) |
| Multi-action menu UX | `app/src/main/java/com/guyi/reader/ui/screens/LibraryScreen.kt` (BookRow / DropdownMenu) |
| Detail sheet | `app/src/main/java/com/guyi/reader/ui/screens/BookDetailSheet.kt` |
| Sync screen with danger zone | `app/src/main/java/com/guyi/reader/ui/screens/SyncScreen.kt` |
| First-login auto-sync dialog | `app/src/main/java/com/guyi/reader/sync/viewmodel/SyncScreenViewModel.kt` (`showAutoSyncPrompt` flow) |

Higher-level design docs (read before coding):

- `docs/MULTI_PLATFORM_PLAN.md` — the umbrella architecture for the 5-client system
- `docs/PHASE_D_LIBRARY_SYNC_V2.md` — the library + sync UX spec (state model, deletes, filter chips). Reference for the UI you'll build.
- `docs/server/openapi.yaml` — machine-readable API contract
- `docs/server/SECURITY.md` — server-side hardening (input caps, throttle, replay detection). Useful to understand what errors you might get.

GitHub mirror: <https://github.com/guyiicn/epub_android>

---

## 9. Not yet implemented (server-side gaps)

If you need any of these for your client, raise it now — they're planned:

| Feature | Status | Server work |
|---|---|---|
| WebSocket `/sync` push channel | **Not started** (Phase D-6). Clients today poll on triggers. | New endpoint, ~1 day |
| `GET /storage` quota endpoint | **Not started**. Clients can't show "you're using X / Y MB". | ~half day |
| Multi-user support | **Not started**. Server is single-user (`admin`). | Significant — adds registration endpoint, ACL on rows, device limits |
| Conflict resolution UI on Book content | **Not started**. Reference Android logs CONFLICT to `lastSyncError` but no UI. | Out of scope; clients should expose. |

---

## 10. Dev / debugging

### 10.1 Test admin credentials (DEV ONLY)

```
Server:    https://us.guyii.net
Username:  admin
Password:  <your-password>
```

This password is intentionally trivial because we're in active dev. **Rotate before any non-dev use.**

To rotate:
```bash
ssh root@us.guyii.net
cd /opt/epub-reader

# Generate a new bcrypt hash off-box, e.g. with htpasswd:
htpasswd -nbB admin 'new-password' | cut -d: -f2

# Stop the container, wipe just the admin row (NOT the whole DB), update .env, restart:
docker compose down
docker run --rm -v /opt/epub-reader/data:/data alpine sh -c \
  'apk add -q sqlite && sqlite3 /data/db.sqlite "DELETE FROM users"'
sed -i 's|^BOOTSTRAP_ADMIN_PASSWORD=.*|BOOTSTRAP_ADMIN_PASSWORD=new-password|' .env
docker compose up -d --force-recreate
```

(`--force-recreate` is required — `restart` does not reload `env_file`.)

### 10.2 Read server logs

```bash
ssh root@us.guyii.net 'docker logs --tail 100 -f epub-server'
```

Application-level events (login, push, GC) are at INFO. Push failures and replay attempts are at WARN.

### 10.3 Read Caddy access log (every HTTPS request)

```bash
ssh root@us.guyii.net 'tail -F /var/log/caddy/access.log | jq -r ".ts|todateiso8601[11:19], .request.method, .request.uri, .status"'
```

Essential when debugging "my client says the request was sent but I see nothing happen" — confirms whether the request reached the edge.

### 10.4 Inspect the database

```bash
ssh root@us.guyii.net
docker run --rm -v /opt/epub-reader/data:/data alpine sh -c \
  'apk add -q sqlite && sqlite3 -header -column /data/db.sqlite "SELECT * FROM books LIMIT 5"'
```

Tables of interest: `books`, `progress`, `bookmarks`, `annotations`, `sync_log`, `devices`. The full schema is in `/home/guyi/code/epub_android/docs/server/schema.sql`.

### 10.5 Audit trail

Every successful write to a business endpoint inserts a row into `sync_log` with `(operation, target_id, happened_at, device_id)`. Use this when debugging "did my client's request actually do anything?".

```sql
SELECT operation, substr(target_id,1,16) as target,
       datetime(happened_at/1000, 'unixepoch', '+8 hours') as at
FROM sync_log ORDER BY happened_at DESC LIMIT 20;
```

### 10.6 Local dev server (if you want to bring up your own)

The server code is at `/opt/epub-reader/source/` on the prod box, or cloned from <https://github.com/guyiicn/epub_reader_server>. Bring up locally:

```bash
git clone https://github.com/guyiicn/epub_reader_server
cd epub_reader_server
export JWT_SECRET="$(openssl rand -base64 48)"
export BOOTSTRAP_ADMIN_USERNAME=admin
export BOOTSTRAP_ADMIN_PASSWORD=<your-password>
./gradlew run
# Listens on 127.0.0.1:8090
```

Hits `docs/DEV_RUNBOOK.md` in that repo for the full smoke walkthrough.

---

## 11. Open questions / sanity checks for your client

Before you commit to architecture decisions, sanity-check against these:

1. **Where do you store the JWT?** Web: HttpOnly cookie ideal, but you're hitting a different origin (CORS+credentials). LocalStorage + bearer header is simpler but XSS-exposed. For an internal-use client this is fine; document the choice. Windows: Windows Credential Manager via Win32 API or DPAPI-encrypted file is the right choice over plain settings file.
2. **Does your local DB support `serverHash` distinct from `contentHash`?** If not, add it. Don't try to overload the same field — you'll hit one of our bugs.
3. **Are your per-table sync cursors invalidated on schema upgrade?** Add a `schema_version_seen` preference and reset cursors when it lags.
4. **Are your triggers actually firing under your platform's lifecycle?** Web: `visibilitychange`. Windows: window `Activated`. Test that "open app, see new books from the other device" happens within 60s without manual sync.
5. **Are 401s automatically retrying with a refresh?** And is the refresh serialized?
6. **Are your uploads streaming?** Don't `fs.readFileSync` a 50 MB EPUB. Web: use ReadableStream from File / Blob.
7. **Are you sending `X-Device-Id` on every business call?** Easy to forget after `/auth/login` succeeds.
8. **Is `deviceId` persisted across launches?** Test by uninstalling and reinstalling — the server should NOT see them as the same device after reinstall (intentional), but it SHOULD see them as the same device across app restarts (less intentional + critical for sync_log audit).

---

## 12. Versioning policy

OpenAPI is at `v1.0.1`. The contract here is stable for v1.x; breaking changes will land at `/api/v2/...` with a deprecation window. If you see your client receive unexpected fields, it's safe to ignore them (forward-compat).

The server reports its build via `version` in the authed `GET /healthz` — not strictly tied to OpenAPI version.

---

## 13. Help, escalation

Server source: <https://github.com/guyiicn/epub_reader_server>
Android client: <https://github.com/guyiicn/epub_android>
Issues: <https://github.com/guyiicn/epub_reader_server/issues>

Dev contact: Guyi (`guyiicn@gmail.com`).

---

**You are now ready to start integrating.** Read §1, §5 (especially §5.2), §7, then write a smoke-test pass before implementing anything UI. The Android client took two days plus a chain of bugs to settle the data model — most of that pain is concentrated in §5.2 and §7. Don't re-derive it.

---

## V2 Addendum (Phase E, schema_version=2)

Server `1.2.0+`. Backwards compatible — V1 clients keep working, just ignoring the new columns and endpoints. New surfaces:

### A. New REST endpoints

| Endpoint | Verb | Purpose |
|---|---|---|
| `/tags` | GET / POST | List (incremental, `?since=`) / create tag |
| `/tags/{id}` | PUT / DELETE | Rename or recolor / soft-delete |
| `/book-tags` | GET | Incremental list of (bookId, tagId) associations |
| `/books/{bookId}/tags/{tagId}` | PUT / DELETE | Attach / detach |
| `/books/{id}/reading-status` | PUT | Set `NONE/WANT_TO_READ/READING/FINISHED/ABANDONED` |
| `/sessions` | GET / POST | List sessions / create on reading start |
| `/sessions/{id}` | PUT | Close session: `endedAt + durationMs + endProgression` |
| `/stats/overview` | GET | Aggregates — total ms, sessions, books opened/finished, streaks |
| `/stats/books/{id}` | GET | Per-book aggregates |
| `/stats/daily?since=&until=` | GET | UTC daily reading-time series |
| `/storage/quota` | GET | Used bytes, file count, disk free, per-format breakdown |
| `/sync` | GET (Upgrade) | WebSocket real-time push (see §B) |

Same auth as V1: `Authorization: Bearer <accessToken>` and `X-Device-Id: <deviceId>`.

`Book` DTO now carries a `readingStatus` field (default `"NONE"`). Old clients can keep ignoring it.

### B. WebSocket `/sync` (real-time invalidate)

**URL** `wss://us.guyii.net/api/v1/sync` (or local `ws://`).

**Auth** — pick one:
- `?token=<accessJwt>` query parameter (mobile / desktop friendly)
- `Sec-WebSocket-Protocol: bearer.<accessJwt>` subprotocol (browser friendly; no token in URL or proxy logs)

**Required header** `X-Device-Id: <registered device id>`.

**On accept** — server immediately sends `{"type":"hello","ts":1234567890123}`. Treat this as "auth ok, you may proceed."

**Push frames** — every REST write fires:

```json
{
  "type": "invalidate",
  "table": "tags",
  "cursor": 1782366729398,
  "originDeviceId": "ca0a9df81760f32a",
  "targetId": "251bbf61-b919-402d-b036-1f44a0d6686d",
  "ts": 1782366729401
}
```

- `table` ∈ {`books`, `progress`, `bookmarks`, `annotations`, `tags`, `book_tags`, `sessions`}
- `originDeviceId` is the device that performed the write. If it matches your own deviceId, **ignore the event** (you already have the change locally).
- `cursor` is the row's new `updated_at`. Recover the row by calling the matching `?since=cursor-1` endpoint (subtract 1 to keep `>=` semantics correct).

**Heartbeat** — client may send `{"type":"ping"}`; server replies `{"type":"pong","ts":...}`. Server itself pings every 25 s and times out after 75 s of silence.

**Reconnect** — exponential back-off (1 s → 2 s → 4 s ... cap 30 s). On reconnect, **immediately run a full incremental pull** (every V1 + V2 table) using persisted cursors so events missed during disconnect are caught up.

**Close codes**: 1000 normal, 1011 server error, 4001 auth failure, 4003 device not registered.

### C. Suggested client sync state model

V1 introduced `BookSyncState ∈ {SYNCED, LOCAL_ONLY, REMOTE_ONLY, OPTED_OUT, CONFLICT}` derived from two hashes per book. V2 adds three independent sync streams: `tags`, `book_tags`, `sessions`. Each gets its own `SyncCursor` row keyed by table name. The `/sync` WebSocket only signals *which* cursor to advance; you still do the actual fetch over REST. Stats endpoints are **read-only computed server-side** — there is no `stats` cursor, the client just refetches when displaying.

### D. Reading sessions — client flow

1. **On reading start** (book opens, page visible): `POST /sessions` with a fresh UUID, `bookId`, `startedAt = now()`, `startProgression = current progression`. Persist the id locally.
2. **On page turn**: nothing for the server — track locally.
3. **On idle ≥ 5 minutes** *or* book close *or* app backgrounded: `PUT /sessions/{id}` with `endedAt = now()`, `durationMs = endedAt - startedAt - paused_time`, `endProgression`, `pagesTurned`. The 5-minute idle threshold is a shared convention across clients.
4. **If the app dies mid-session** (no PUT close), the open session row has `endedAt = null`. On next start: if `startedAt < now - 24h`, treat as abandoned and close it with `durationMs = 0` (or skip — the server stats logic also ignores open sessions older than 24h).

### E. Reading status semantics

- Typical UX transitions: NONE → WANT_TO_READ → READING → FINISHED, plus the side branch ABANDONED.
- Server enforces no transition rules — clients pick. PUT bumps `books.updatedAt`, so existing `/books?since=` pulls already surface the change.
- Status is global per user (not per device).
