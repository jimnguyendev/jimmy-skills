# Storage primitive + media object lifecycle

How user-uploaded media flows through the system: from S3-compatible backend to a tracked, attachable, garbage-collected resource. Sources: ADR-0014 (storage backend), ADR-0016 (lifecycle). Reference implementation: `pkg/storage/`, `internal/feature/media/`.

## Storage primitive (`pkg/storage`)

`pkg/storage` exposes a 7-method, S3-flavoured interface. Concrete backends live in sub-packages.

```go
type Storage interface {
    Upload(ctx context.Context, in UploadInput) (UploadResult, error)
    Download(ctx context.Context, key string) (io.ReadCloser, ObjectInfo, error)
    Delete(ctx context.Context, key string) error
    Copy(ctx context.Context, srcKey, dstKey string) error
    Exists(ctx context.Context, key string) (bool, error)
    Stat(ctx context.Context, key string) (ObjectInfo, error)
    PresignGet(ctx context.Context, key string, ttl time.Duration) (string, error)
    PresignPut(ctx context.Context, key string, contentType string, ttl time.Duration) (string, error)
}
```

Rules:

- **One backend, one SDK.** Production = Cloudflare R2; local dev = RustFS. Both speak S3, so `aws-sdk-go-v2/service/s3` (in `pkg/storage/s3`) is the only SDK that crosses the boundary.
- **Features import only `pkg/storage`.** Do NOT import `aws-sdk-go-v2` from `internal/feature/*`. The interface is the contract; the concrete backend is wiring.
- **`List` is omitted on purpose.** When we need pagination we'll add it then with paging baked in. Do not add it preemptively.
- **Env prefix is `STORAGE_*`** (not `R2_*` or `S3_*`) so the same vars work for RustFS and R2.
- **Default object key scheme:** `media/{kind}/{userID}/{uuid}{ext}`. The owner segment is for `Delete`-by-key safety, not for ownership lookup (use the DB row).

## Lifecycle overview

A user-uploaded file goes through five well-defined moves. The `media_objects` table tracks every state transition.

```
┌────────────────────────────────────────────────────────────────────┐
│ 1. PRESIGN          POST /api/v1/media/presign                     │
│    server: insert media_objects (status=pending), key=staging/...  │
│            return signed PUT URL bound to client-declared MIME     │
│    client: PUT bytes to staging/...                                │
├────────────────────────────────────────────────────────────────────┤
│ 2. COMMIT           POST /api/v1/media/objects/:id/commit          │
│    server: Stat(staging key) — verify size + content-type match    │
│            Copy(staging → media/...)                               │
│            Delete(staging key)                                     │
│            UPDATE media_objects SET status='committed'             │
│            publish "media:committed"                               │
├────────────────────────────────────────────────────────────────────┤
│ 2b. PROXY UPLOAD    POST /api/v1/media/upload                      │
│    server: stream multipart → media/... directly                   │
│            INSERT media_objects (status='committed')               │
│            publish "media:committed"                               │
├────────────────────────────────────────────────────────────────────┤
│ 3. ATTACH           (service-level, no endpoint)                   │
│    other feature calls media.Service.Attach(mediaID, kind, id)     │
│    server: verify ownership + status=committed                     │
│            UPDATE media_objects SET attached_kind, attached_id     │
├────────────────────────────────────────────────────────────────────┤
│ 4. DELETE           DELETE /api/v1/media/objects/:id               │
│    server: ownership check via owner_user_id                       │
│            UPDATE media_objects SET status='deleted', deleted_at   │
│            Delete(media key) — best-effort                         │
│            publish "media:deleted"                                 │
├────────────────────────────────────────────────────────────────────┤
│ 5. CLEANUP          (event listener)                               │
│    media subscribes: vocabulary:card.deleted, deck.deleted,        │
│                      user:avatar.changed                           │
│    on event: SoftDeleteAttached(kind, ids) → step 4 internally     │
└────────────────────────────────────────────────────────────────────┘
```

## Two-prefix object keys

Presigned PUT writes to `staging/{kind}/{userID}/{uuid}{ext}`; committed objects live under `media/{kind}/{userID}/{uuid}{ext}`. Bucket lifecycle rules expire `staging/` after 24h, so abandoned presigns clean themselves up without a server-side sweeper.

```
staging/avatar/<uuid>/<uuid>.jpg   <-- 24h TTL (R2 lifecycle rule)
media/avatar/<uuid>/<uuid>.jpg     <-- permanent (until DELETE)
```

This is why a single-prefix + status-column-only design is rejected: R2 lifecycle filters by prefix, not metadata. Without two prefixes we'd need a server sweeper from day one.

## The `media_objects` table

Every upload has a row. Columns:

| Column | Why |
|---|---|
| `id` (UUID) | The `media_id` other features store — never the raw key or URL |
| `owner_user_id` | Ownership lookup for delete + listing; not derived from key |
| `kind` | Enum: `avatar`, `card_audio`, `deck_cover`, ... |
| `object_key` | Current key (in `staging/` or `media/`) |
| `content_type`, `size_bytes`, `etag` | Captured at presign-declared, verified at commit |
| `status` | `pending` → `committed` → `deleted` |
| `attached_kind`, `attached_id` | What entity claimed this media (e.g. `card`, <card_id>) |
| `created_at`, `committed_at`, `deleted_at` | Lifecycle timestamps |

Rules:

- **`media_id` is the stable handle.** Other features (vocabulary, user) store `media_id`, never the URL or key. We can rotate `STORAGE_PUBLIC_BASE_URL` or move to signed-GET later without rewriting card rows.
- **Ownership is the row's `owner_user_id`, not the key prefix.** `Delete by id` checks the column; never trust the key segment.
- **Soft delete only.** `status='deleted'` + `deleted_at`. The storage object is removed best-effort; if storage fails, a future GC sweep retries. Hard-delete the row only after a long grace period.
- **Per-user quota** lives here: index on `(owner_user_id, status)` and count.

## Commit verifies, not trusts

Presigned PUT is fire-and-forget on the wire — the server never sees the bytes. Commit is the integrity gate.

```go
// In media.Service.Commit:
info, err := s.store.Stat(ctx, row.ObjectKey)              // HEAD the staged object
if err != nil { return ErrStagingMissing }
if info.Size != row.SizeBytes        { return ErrSizeMismatch }
if info.ContentType != row.ContentType { return ErrMimeMismatch }

if err := s.store.Copy(ctx, row.ObjectKey, mediaKey); err != nil { return ... }
_ = s.store.Delete(ctx, row.ObjectKey)                     // best-effort
// flip status, publish media:committed
```

Commit is **idempotent on the same `media_id`** — second call returns the existing committed row. Mobile / web clients can retry without producing duplicates.

## Cross-feature cleanup via event bus

Vocabulary publishes `vocabulary:card.deleted` and `vocabulary:deck.deleted`; user publishes `user:avatar.changed`. The media feature's `listeners.go` subscribes and best-effort-deletes the referenced media objects async via `event.Bus`.

```go
// internal/feature/media/listeners.go
const (
    topicCardDeleted   = "vocabulary:card.deleted"
    topicDeckDeleted   = "vocabulary:deck.deleted"
    topicAvatarChanged = "user:avatar.changed"
)

func registerListeners(bus *event.Bus, svc *Service, lgr logger.Logger) {
    if bus == nil { return }

    bus.Subscribe(topicCardDeleted, "media.cleanup-card", func(ctx context.Context, e event.Event) error {
        p, ok := e.Payload.(CardDeletedPayload)
        if !ok { return nil }
        svc.SoftDeleteAttached(ctx, "card", []string{p.CardID})
        return nil
    })
    // ... deck.deleted, avatar.changed ...
}
```

Rules:

- **Vocabulary / user NEVER import `media`.** The bus is the contract.
- **Topic strings are duplicated, not imported.** Each feature exports its own copy of the topic name; the listener mirrors the string. No package depends on another.
- **Listener payload structs are mirrored** in the subscriber package (e.g. media's `CardDeletedPayload`) — only the fields the cleanup needs.
- **Failures log + continue.** The entity-delete path has already returned 200 to the user; cleanup is best-effort. A failed cleanup leaves an orphan that a future GC job sweeps.

## Attach via consumer-side interface

When vocabulary's `AddCard` accepts an `audio_media_id`, the vocabulary service calls a narrow `MediaResolver` interface:

```go
// In internal/feature/vocabulary/types.go (consumer-side)
type MediaResolver interface {
    AttachToCard(ctx context.Context, mediaID, cardID, ownerUserID string) error
}
```

`media.Service` satisfies it. The wiring lives in `cmd/app/http.go`:

```go
mediaBundle := media.Provide(media.Deps{...})
vocabHandler := vocabulary.Provide(vocabulary.Deps{
    // ...
    MediaResolver: mediaBundle.Service,
})
```

The interface enforces ownership + `status=committed` and stamps `attached_kind/id` in one call. **Vocabulary never imports the `media` package.**

## Topics published

| Topic | Payload | Subscribers (today) |
|---|---|---|
| `media:committed` | `media_id, owner, kind, object_key, content_type, size_bytes, public_url` | (analytics / search / future indexing) |
| `media:deleted` | `media_id, owner, object_key` | (analytics / future GC reconciliation) |

## Topics subscribed

| Topic | Owning feature | Listener handler |
|---|---|---|
| `vocabulary:card.deleted` | vocabulary | `SoftDeleteAttached("card", [card_id])` |
| `vocabulary:deck.deleted` | vocabulary | `SoftDeleteAttached("card", card_ids[])` + `SoftDeleteAttached("deck", [deck_id])` |
| `user:avatar.changed` | user | `Delete(user_id, old_media_id)` (skip if new == old) |

## Endpoints

```
POST   /api/v1/media/upload                — proxy upload (multipart/form-data)
POST   /api/v1/media/presign               — issue signed PUT URL for client-direct upload
POST   /api/v1/media/objects/:id/commit    — finalise a presigned upload
GET    /api/v1/media/objects/:id           — fetch metadata + public URL
DELETE /api/v1/media/objects/:id           — soft-delete an owned object
```

All paths are scoped by `media_id` (server-issued UUID), not by the raw object key. The wildcard `/api/v1/media/object/*key` route is **forbidden** — any future code reintroducing it must justify why owner-by-key-prefix is acceptable.

## Anti-patterns

| Anti-pattern | Fix |
|---|---|
| Feature stores raw `object_key` or public URL of user media | Store `media_id` (UUID); resolve to URL via `media.Service` |
| Importing `aws-sdk-go-v2` from `internal/feature/*` | Use `pkg/storage.Storage`; backend stays in `pkg/storage/s3` |
| `DELETE /media/object/<key>` (key-based delete) | Use `DELETE /media/objects/:id` (id-based, ownership in DB row) |
| Vocabulary calls `media.Service.Delete` directly on card delete | Publish `vocabulary:card.deleted`; media's listener cleans up |
| Vocabulary imports the `media` package for cleanup | Use the event bus; topic string + payload struct mirrored, no import |
| Single-prefix object keys + status-column-only | Use `staging/` + `media/` so R2 lifecycle can expire abandoned presigns |
| Skip the commit step, reconcile lazily on first attach | Push verification cost into hot paths is wrong; commit is the integrity gate |
| Synchronous storage delete inside vocabulary's delete tx | Async via event bus; flaky storage call must not block the user write path |
| Trust the client's declared size / MIME at presign time | Re-`Stat` at commit; reject on mismatch |
| Hard-delete the `media_objects` row on DELETE | Soft-delete (`status='deleted'`); GC job hard-deletes after grace period |
| Per-feature ad-hoc storage interface (`audio.Store`, `avatar.Store`) | One `pkg/storage.Storage` for all object storage; differentiate by `kind` enum |
