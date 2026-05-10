# Long-running Tasks

## Problem

Use cases: report generation, ML inference jobs, loan application processing.

When each message takes a long time to process:

- **Message lag grows** because the consumer can't keep up.
- **Consumer misses heartbeats** → Kafka considers it dead → triggers rebalance, even though it's still working.

The best practice is: **consumers should process data as fast as possible** and offload heavy work.

## Approach 1: Asynchronous processing

Consume and commit immediately, then process asynchronously via a thread pool or work queue.

**Trade-offs:**

- ✅ Very low consume latency — offsets advance immediately.
- ✅ Simple consumer code.
- ❌ **At-most-once delivery** — if the async task fails, the message is already committed and won't be retried from Kafka.
- ❌ Error handling is complex — need a separate dead-letter or retry mechanism.
- ❌ Processing order is not guaranteed (async tasks complete in arbitrary order).

> Use this only when occasional message loss is acceptable and ordering doesn't matter.

## Approach 2: Liveness tuning

Adjust Kafka consumer settings to tolerate longer processing times without triggering a rebalance.

### How Kafka detects failed consumers

A consumer is considered dead if:

- `poll()` is not called within `max.poll.interval.ms` (default: **5 minutes**)
- No heartbeat is sent within `session.timeout.ms` (default: **45 seconds**)

### Tuning parameters

```properties
# Reduce records per poll batch → faster per-batch processing
max.poll.records=250              # default: 500

# Allow more time between poll() calls
max.poll.interval.ms=600000       # increase from default 300000

# Allow more time before session timeout
session.timeout.ms=60000          # increase from default 45000
```

**Trade-offs:**

- ✅ No code changes to the consumer — config-only fix.
- ❌ **Unreliable** — processing time can vary unpredictably, so you're always guessing the right timeout values. A spike in processing time will still trigger rebalance.

> This is a stopgap, not a solution. Use the pipeline approach below for tasks with unpredictable or long processing times.

## Approach 3: Pause/resume pattern

Pause the partition while processing a long task, keep calling `poll()` to send heartbeats, then resume when done.

```
Consumer.poll() → receive message from partition P0
Consumer.pause(P0)                        # stop fetching from P0
  ↓
  [Process long-running task]
  while processing: Consumer.poll()       # returns empty, but sends heartbeats
  ↓
Consumer.resume(P0)                       # resume fetching from P0
Consumer.commitSync()                     # commit offset
```

**Trade-offs:**

- ✅ Keeps the consumer in the group — no rebalances.
- ✅ **At-least-once delivery** — offset is committed only after processing.
- ✅ No extra topics or infrastructure required.
- ❌ Consumer throughput drops to zero on the paused partition during processing.
- ❌ Must manage a background thread or async loop to keep calling `poll()` while the task runs.

> Good middle ground when task duration is long but bounded (minutes, not hours) and you don't want the complexity of a full pipeline.

## Approach 4: Pipeline (recommended for complex tasks)

Break the long process into multiple short stages, each with its own Kafka topic.

### Architecture

```
Command Topic → [Stage 0: Validate] → Stage 1 Topic
                                           ↓
                    [Stage 1: Transform] → Stage 2 Topic
                                           ↓
                            [Stage 2: Persist] → Done
```

Each stage is a fast consumer that does one thing and produces to the next topic.

### Single-topic variant (loop pattern)

For processes where stages are dynamic, use one Stage Topic that the processor reads from and writes back to with an incremented stage number:

```
Command Topic → Stage Topic ←──── [Produce next stage]
                    │                      ↑
                    ↓                      │
              [Initializer] → [Stage Processor]
```

**Trade-offs:**

- ✅ Low per-stage latency — each stage is fast enough to avoid heartbeat issues.
- ✅ **At-least-once delivery** — each stage commits only after completing its work.
- ✅ **Traceable** — you can see which stage a message is at by inspecting the stage topic.
- ✅ **Decoupled** — each stage can be scaled, deployed, and debugged independently.
- ✅ Stages within one process execute in order.
- ❌ More complex topology — multiple topics and consumer groups.
- ❌ **Cross-process ordering is not guaranteed** — process A's stage 2 may run before process B's stage 1.

### Choosing an approach

| Criterion | Async | Liveness Tuning | Pause/Resume | Pipeline |
|-----------|-------|----------------|-------------|----------|
| Delivery guarantee | At-most-once | Depends on config | At-least-once | At-least-once |
| Code changes | Minimal | None | Moderate | Significant |
| Predictable processing time | Not needed | Required | Not needed | Not needed |
| Ordering | No | Yes (within partition) | Yes (within partition) | Per-process yes, cross-process no |
| Production readiness | Low | Medium | Medium-High | High |
