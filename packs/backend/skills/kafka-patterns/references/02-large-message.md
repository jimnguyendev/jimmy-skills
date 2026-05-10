# Large Messages

## Problem

- Producers need to send large payloads (e.g., report files) through Kafka.
- The **default max Kafka record size is 1 MB**.
- Real files often exceed this limit.

## Solution 1: Increase max message size (anti-pattern)

Configure `max.message.bytes` at topic level and `message.max.bytes` at broker level (in `server.properties`).

**Why this is an anti-pattern:**

- Practical ceiling of **~10 MB** — beyond this, replication pressure, memory usage, and GC pauses become severe. Kafka has no hard-coded limit, but production clusters rarely go higher.
- Reduces overall topic throughput — large records occupy batch space.
- Slows down consumers — each record takes longer to deserialize and process.
- Wastes broker resources — Kafka is optimized for small, uniform messages.
- Must also adjust `fetch.max.bytes` on the consumer side, otherwise large messages will be silently dropped.

> Consider `compression.type` (lz4, zstd) first — it can bring messages near the 1 MB boundary back under the default limit with no architecture change. Only increase `max.message.bytes` when messages slightly exceed 1 MB and compression is not enough.

## Solution 2: External storage (recommended)

Store the file externally and pass a reference through Kafka.

**Flow:**

1. Producer uploads the file to **object storage** (S3, GCS, Azure Blob).
2. Storage returns the **object URL**.
3. Producer sends a Kafka message containing the **URL instead of the file**.
4. Consumer reads the message, **downloads the file** via the URL, then processes it.

**Trade-offs:**

- ✅ Simple to implement — no Kafka config changes needed.
- ✅ Better performance — Kafka handles only small reference messages.
- ❌ Introduces a dependency on external storage.
- ❌ Must handle storage failures, permissions, and URL expiration (use pre-signed URLs with sufficient TTL).

> This is the standard pattern used by most production Kafka deployments for payloads over 1 MB.
