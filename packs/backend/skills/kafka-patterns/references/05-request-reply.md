# Request & Reply

This topic builds progressively through 5 problems and their solutions. Each solution introduces the next problem.

## Problem 1: Two-way communication

Kafka topics are one-directional: Requestor → Topic → Replier. But services often need a response back (e.g., REST API backed by Kafka).

### Solution 1: Request-Reply Pattern

Use two topics — one for requests, one for replies:

```
Requestor ──→ Request Topic ──→ Replier
    ↑                              │
    └──── Reply Topic ←────────────┘
```

Reference: Enterprise Integration Patterns — Request-Reply.

---

## Problem 2: Request-response mapping

All responses land in the same Reply Topic. How does the requestor know which response belongs to which request?

### Solution 2: Message ID

Every message carries two IDs:

- `message_id` — unique per message (UUIDv4).
- `original_message_id` — the `message_id` of the request this is replying to. Null for initial requests.

The requestor matches replies to requests by looking up `original_message_id`. This also enables distributed tracing.

---

## Problem 3: Processing replies

The requestor has a Request Thread (sending requests) and a Reply Thread (consuming responses). How do they coordinate?

### Solution 3: Message Pool

A **local cache** (e.g., Guava Cache) stores either responses or callbacks, keyed by `message_id`. Set a **TTL** on entries to prevent memory leaks from unanswered requests.

**Option A — Synchronous polling:**

1. Request Thread sends a request and saves `message_id`.
2. Reply Thread consumes all responses and pushes them into the pool.
3. Request Thread polls the pool until it finds the matching `message_id`.

Trade-offs: simple but wastes CPU on polling loops, limiting throughput.

**Option B — Asynchronous callback (recommended):**

1. Request Thread sends a request and stores a `CompletableFuture` in the pool (key: `message_id`).
2. Reply Thread consumes a response, looks up the future by `original_message_id`, and completes it.
3. The Request Thread's future resolves, triggering the API response.

Trade-offs: better performance, but more complex code.

---

## Problem 4: Multiple requestor instances

When the requestor service has multiple instances, how does each instance get its replies?

**Option A — One consumer group per instance:**
Each instance creates its own consumer group (random group ID) and consumes **all** responses from the Reply Topic.

- ❌ Every instance wastes resources filtering out responses that belong to other instances.
- ❌ Load on the Reply Topic scales linearly with instance count.

**Option B — One consumer group for all instances:**
All instances share one consumer group. Each instance consumes only a subset of Reply Topic partitions.

- ❌ Instance A sends a request, but Instance B's partition receives the reply.
- ❌ Needs a mechanism to share responses across instances.

### Solution 4: Remote Message Pool

Use **Redis** as a shared message pool:

- Key: `message_id`
- Value: serialized response
- TTL: set an expiry to prevent unbounded growth.

All reply threads push responses to Redis. Any request thread polls Redis for its response.

**Trade-offs:**

- ✅ Works with any number of instances.
- ❌ Adds network latency (consumer → Redis → requestor).
- ❌ Operational overhead for Redis.
- ❌ Redis is not strongly consistent — risk of data loss on failover.

---

## Problem 5: Eliminating the remote pool

The remote message pool adds latency, complexity, and a reliability dependency.

**Failed attempt — Fixed partition routing:**
Route each response to the specific partition that the originating requestor instance is consuming. Hard to implement (need to track partition assignments) and hard to scale (assignments change on rebalance).

### Solution 5: Return Address Pattern

The requestor embeds its **IP address** (or service URL) directly in the request message. The replier sends the response back via a **direct HTTP call** to that address. No Reply Topic needed.

```json
{
  "message_id": "C1",
  "original_message_id": null,
  "ip": "88.23.34.01",
  "payload": {}
}
```

**Flow:**

1. Requestor sends request (with IP) → Request Topic → Replier.
2. Requestor stores a `CompletableFuture` in a **local** message pool.
3. Replier processes the request and POSTs the response to `http://{ip}/reply`.
4. Requestor's reply endpoint receives the response, looks up the callback, completes the future.

**Trade-offs:**

- ✅ Best performance — no Reply Topic, no Redis, no polling.
- ✅ Scales well for the request-reply use case specifically.
- ❌ **Coupling** — replier must know how to call the requestor's API.
- ❌ **Not event-based** — the response is a direct call, not an event on a topic. Other services can't subscribe to it. Not suitable for fan-out or broadcast patterns.

## Summary

| Solution | Best for | Key trade-off |
|----------|----------|--------------|
| Reply Topic + Local Pool | Single instance, simple setup | Polling overhead |
| Remote Pool (Redis) | Multi-instance, shared state needed | Redis latency + ops |
| Return Address | Low-latency request-reply | Coupling, not event-based |

> **Best practice:** Package the request-reply client logic (message ID generation, pool management, callback handling) as a **shared library** so all services use the same implementation.
