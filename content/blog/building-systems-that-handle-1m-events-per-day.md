---
title: "Building Systems That Handle 1M Events Per Day: Lessons from the Trenches"
date: 2025-03-24
---

At Phenom, I built a reconciliation framework that processes 1 million events per day. This post is about what it takes to build systems at that scale, the mistakes we made, and the architecture that finally worked.

## The Problem

We had 10+ downstream systems (data warehouses, analytics platforms, CRM, billing systems) consuming data from our integration platform. The problem:

- **Data drift**: System A says "order completed," System B says "order pending." Which is right?
- **Silent failures**: Errors didn't surface until users reported discrepancies.
- **Manual reconciliation**: Engineers spent hours each week manually investigating mismatches.

We needed a reconciliation framework that:
1. Detected mismatches in near real-time
2. Identified root causes (which system is wrong?)
3. Auto-replayed events when possible
4. Scaled to 1M events/day with predictable latency

## First Attempt: Database Triggers (Failed)

Our initial design was simple:

```
Event comes in → Insert into DB → Trigger fires → Compare with downstream → Log mismatch
```

**Why it failed:**

### The Trigger Explosion

Every event inserted triggered 10+ checks (one per downstream system). At 1M events/day, that's 10M+ database operations per day just for reconciliation.

**Problem**: Our PostgreSQL instance wasn't designed for that write load. Database locks caused cascading failures.

### Latency Accumulation

Trigger execution added 50-100ms per event. With 1M events, reconciliation lagged 10+ hours behind.

**Lesson**: Reactive processing (triggers) doesn't scale at high throughput.

## Second Attempt: Batch Jobs (Worked, but...)

We moved to nightly batch jobs:

```
Daily at 2 AM:
  1. Pull all events from yesterday
  2. Query all downstream systems
  3. Compare and report mismatches
```

**Pros**:
- No impact on real-time traffic
- Database load was predictable

**Cons**:
- **24-hour detection window**: Users discovered mismatches a full day later
- **Debugging pain**: If a bug occurred at 3 PM, we couldn't replay until next day
- **Resource spikes**: 2 AM job consumed all CPU for 3 hours, affecting other processes

**Lesson**: Batching trades latency for simplicity, but some systems need near real-time.

## Final Architecture: Streaming with Fallback

Here's what we built:

### High-Level Flow

```
┌─────────────┐
│  Integration │
│   Platform  │
└──────┬──────┘
       │
       ▼
┌─────────────────┐
│  Kafka Topics  │
│  (raw events)  │
└───────┬────────┘
        │
        ▼
┌──────────────────────┐
│  Apache Flink Job     │
│  - Stateful Join     │
│  - Windowed Aggregation│
└───────┬──────────────┘
        │
        ▼
┌─────────────────┐
│  Kafka Topics  │
│  (reconciled)  │
└──────┬────────┘
       │
       ▼
┌────────────────────┐
│  Alert Service  │  ← Pub/Sub
│  + Auto-replay  │
└────────────────────┘
```

### Why Flink?

We evaluated Apache Flink vs. Spark Streaming vs. Akka Streams:

| Criterion | Flink | Spark Streaming | Akka Streams |
|-----------|--------|-----------------|----------------|
| State management | ✅ Built-in | ✅ Built-in | ❌ Manual |
| Windowing | ✅ Flexible | ✅ Flexible | ✅ Flexible |
| Exactly-once | ✅ Mature | ⚠️ Requires tuning | ⚠️ Requires tuning |
| Debugging | ✅ Savepoint | ⚠️ Complex | ✅ Local replay |
| Our team familiarity | ⚠️ New | ✅ Familiar | ✅ Familiar |

We chose Flink because **exactly-once semantics** were critical for financial data.

### Key Design Patterns

**1. Watermarking for Late Events**

Events arrive out of order. Event 100 (order canceled) might arrive before Event 99 (order placed).

Flink watermarks handle this:

```java
DataStream<Event> stream = env
    .addSource(kafkaSource)
    .keyBy(event -> event.orderId)
    .window(EventTimeSessionWindows.withGap(Duration.minutes(5)))
    .process(new KeyedProcessFunction<String, Event, Report>() {
        // Flink guarantees all events for this window have arrived
        // based on watermarks
    });
```

**2. State Backends for Comparison**

We can't query downstream systems for every event—too slow. Instead:

```java
// Build in-memory state of what downstream should have
ValueState<Map<String, Event>> expectedState = runtime
    .getOperator(new ValueStateDescriptor<>(
        "expected-downstream-state",
        Types.MAP
    ));

// Compare incoming events against expected state
if (expectedState.get(event.orderId) == null) {
    // Unexpected event—mismatch!
    emitAlert(event, "UNEXPECTED_EVENT");
}
```

**3. Configurable Replay Strategies**

Not all mismatches should be replayed:

```yaml
replay_rules:
  - rule_name: "temporal_violation"
    action: "ALERT_ONLY"
  - rule_name: "data_quality_issue"
    action: "REPLAY_WITH_BACKUP"
  - rule_name: "schema_mismatch"
    action: "MANUAL_REVIEW"
```

This prevents cascading errors from replaying bad events.

## Performance Tuning

### Challenge 1: Checkpointing Bottlenecks

Our initial Flink checkpoints took 2 minutes every 5 minutes. That's 40% CPU just for checkpointing.

**Fix**:
1. Disabled checkpointing for stateless operators
2. Increased checkpoint interval to 10 minutes
3. Used RocksDB state backend instead of file system

**Result**: Checkpoints dropped to 30 seconds.

**Lesson**: Measure before optimizing. We spent 2 weeks tuning memory allocation before realizing the issue was I/O.

### Challenge 2: Backpressure from Downstreams

When Kafka consumers were slow, Flink backpressured and our integration platform slowed.

**Solution**: Rate limiting per downstream system:

```java
kafkaProducer.send(
    topic = downstream.topic,
    key = event.id,
    value = event,
    headers = Map("downstream", downstream.id),
    partitioner = new DownstreamAwarePartitioner(downstream.throughput_limit)
);
```

If downstream X can handle 1K events/sec, we throttle to 1K/sec for that topic.

**Lesson**: Protect yourself from your dependencies. Don't let slow downstreams starve your producers.

## Observability

We built three monitoring layers:

### 1. Flink Metrics

```java
env.getMetricGroup()
    .counter("events_processed")
    .counter("mismatches_detected")
    .gauge("processing_latency_ms", latency)
    .gauge("checkpoint_duration_ms", checkpointTime);
```

Grafana dashboard:
- Events/sec (per topic, per job)
- Mismatch rate (should be < 1%)
- Processing latency (P50, P95, P99)

### 2. Alerting

```yaml
alerts:
  - name: "high_mismatch_rate"
    condition: "mismatch_rate > 5% for 5min"
    severity: CRITICAL
    action: "page_oncall"
  - name: "processing_lag"
    condition: "event_lag > 1hour"
    severity: WARNING
    action: "slack_alert"
```

### 3. Debug Replay

We built a tool to replay events from Kafka for debugging:

```bash
$ replay-tool \
  --from-kafka raw-events \
  --event-id 12345678 \
  --window 5min \
  --simulate \
  --dry-run
```

This let us debug issues without touching production.

## The Numbers

After 6 months:

| Metric | Before | After |
|--------|---------|--------|
| Mismatch detection | Manual, ~1 day lag | Automatic, < 5 min lag |
| Root cause identification | ~4 hours | ~10 minutes (auto) |
| Replay success rate | N/A | 67% (auto), 23% (manual), 10% (no action) |
| Engineering hours on reconciliation | ~20 hours/week | ~2 hours/week |
| Events processed/day | 1M | 1M (stable) |

## Lessons Learned

### 1. State is Hard

Handling state in distributed systems is deceptively tricky:

- **Event ordering**: Use watermarks, not timestamps
- **State size**: Our state grew to 10GB—needed partitioning
- **Failover**: Checkpoint/recovery took 5 minutes initially

**Lesson**: Budget for state complexity. It's often underestimated.

### 2. Backpressure is Your Friend

When downstream is slow, you must throttle. We saw this repeatedly:

```mermaid
Producer (Fast) → Queue → Consumer (Slow)
                     ↓
                  Backpressure
```

If you don't throttle:
- Queue grows until OOM
- Producer crashes
- Cascading failure

**Lesson**: Respect backpressure. Flow control beats unbounded buffering.

### 3. Observability Drives Velocity

Our debug replay tool saved us weeks. Before building it:

1. Engineer reproduces issue in dev (2 hours)
2. Pushes fix (1 hour)
3. Waits for production event matching (days)

After building the tool:

1. Replay production event locally (5 minutes)
2. Verify fix against same event (10 minutes)
3. Push with confidence (1 hour)

**Lesson**: The fastest way to debug production is... debug production.

### 4. Batch vs. Streaming is a Tradeoff

We tried both. Streaming won for us because:

- Real-time detection mattered (financial data)
- Late events were common
- Windowed aggregation worked well for our use case

But for some use cases, batch is better:
- Cost (streaming infra is expensive)
- Complexity (state management, exactly-once)
- Debuggability (batch is easier to reason about)

**Lesson**: Choose the right tool. Streaming isn't always better.

## What I'd Do Differently

1. **Start with batch**: We jumped straight to streaming. If we had proven batch first, we'd have a baseline for comparison.
2. **Build replay first**: Not as an afterthought. It should have been a core feature.
3. **State partitioning from day 1**: We outgrew our initial state backend and had to migrate mid-project.

## Conclusion

Building systems that handle 1M events/day isn't about scaling one component—it's about:

- Understanding your constraints (latency, throughput, consistency)
- Designing for failure (backpressure, state recovery)
- Observability as a first-class feature (debug tools, not just metrics)

But mostly, it's about **iterating**. Our third architecture worked because the first two failed spectacularly.

**The perfect architecture is the one you ship.**

---

Have you built systems at scale? What challenges did you face? I'd love to hear your war stories.
