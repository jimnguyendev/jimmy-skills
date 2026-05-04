# Message Lag

## Definition

**Message lag** = the offset delta between the last produced message and the last committed consumer offset.

```
Time Lag = (Last_Produced - Last_Consumed) / Producer Rate
```

Lag is the **primary KPI** for Kafka operations. It is also a common blind spot — many teams only discover lag problems when they cascade into business failures.

## Consequences of high lag

- **Unacceptable end-to-end latency** for downstream consumers.
- Consumers fail to send heartbeats on time → **unexpected rebalances**.
- Messages **deleted by retention policy** before the consumer reaches them → data loss.
- Cascading failures in dependent services.

## Root cause first — don't jump to scaling

**Do not scale the consumer group as the first response.** Diagnose the root cause first.

Common causes:

- **Producer rate spikes** — sudden burst of traffic.
- **Slow consumer processing** — heavy logic, slow DB writes, external API calls.
- **Long-running tasks** — see `references/04-long-running-task.md`.
- **Network issues** — high latency between consumer and broker.
- **GC pauses** — long JVM garbage collection stops.
- **Rebalance storms** — repeated rebalances consuming processing time.

## Monitoring tools

| Tool | Notes |
|------|-------|
| **Burrow** (recommended) | LinkedIn open-source, consumer lag monitoring with smart evaluation |
| Kafka Lag Exporter | Prometheus-compatible, good for Kubernetes |
| Confluent Control Center | Full monitoring suite (commercial) |
| Custom JMX metrics | `records-lag-max`, `records-lag` per partition |

## When to scale consumers

Only after confirming consumer processing is optimized and the root cause is genuinely insufficient consumer throughput:

1. **Scale out the consumer group** — add more consumer instances.
2. The max number of useful consumers equals the number of partitions. If you've hit that ceiling, **increase partitions first**, then add consumers.
3. See `references/01-topic-width.md` for partition sizing and the repartition procedure.
