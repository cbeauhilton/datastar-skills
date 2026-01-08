---
name: nats-jetstream
description: NATS JetStream persistence layer for building distributed systems with durable messaging. Use this skill for stream configuration, consumer patterns, exactly-once delivery, work queues, and event sourcing with NATS. Covers Go SDK patterns and CLI usage.
version: 1.0.0
---

# NATS JetStream

JetStream is NATS's built-in persistence engine enabling message storage and replay. Unlike Core NATS (which requires active subscriptions), JetStream captures messages and replays them to consumers as needed.

## Core Mental Model

**Streams store messages. Consumers read them.**

- **Streams** = append-only logs that capture messages by subject
- **Consumers** = cursors/views into streams that track position and can replay

This separation allows flexible deployment: one stream can have many consumers with different starting points, filters, and delivery patterns.

## When to Use JetStream

Use JetStream when you need:
- **Temporal decoupling**: Producers and consumers operating at different times
- **Message replay**: Historical record of events
- **At-least-once delivery**: Guaranteed message processing
- **Exactly-once semantics**: Deduplication via message IDs
- **Work queues**: Distribute work across competing consumers

Stick with Core NATS for:
- Tightly coupled request-reply
- Low-TTL ephemeral data
- Control plane messages where durability isn't needed

## Quick Start (Go)

```go
import (
    "github.com/nats-io/nats.go"
    "github.com/nats-io/nats.go/jetstream"
)

// Connect
nc, _ := nats.Connect(nats.DefaultURL)
js, _ := jetstream.New(nc)

// Create stream
stream, _ := js.CreateStream(ctx, jetstream.StreamConfig{
    Name:     "EVENTS",
    Subjects: []string{"events.>"},
})

// Publish (with ack)
js.Publish(ctx, "events.user.created", []byte(`{"id": 1}`))

// Create consumer and consume
cons, _ := stream.CreateOrUpdateConsumer(ctx, jetstream.ConsumerConfig{
    Durable: "my-consumer",
})

msgs, _ := cons.Fetch(10)
for msg := range msgs.Messages() {
    // Process message
    msg.Ack()
}
```

## Key Concepts

### 1. Streams Are Append-Only Logs

Messages published to matching subjects are stored in sequence. Streams define:
- Which subjects to capture (wildcards supported)
- How long to keep messages (retention policy)
- Storage limits (count, bytes, age)

### 2. Consumers Are Cursors

Consumers track position and provide replay capabilities:
- **Durable**: Named, survives disconnects, explicitly deleted
- **Ephemeral**: Unnamed, auto-deleted after inactivity
- **Ordered**: Ephemeral with automatic flow control (simplest)

### 3. Acknowledgment Is Critical

| Policy | Use Case |
|--------|----------|
| `AckExplicit` | Default. Each message requires individual ack |
| `AckAll` | Ack final message = ack all prior |
| `AckNone` | Fire-and-forget (no redelivery) |

**Ack Types:**
- `Ack()` - Success, remove from pending
- `Nak()` - Failed, redeliver immediately
- `InProgress()` - Extend processing deadline
- `Term()` - Stop redelivery (poison message)

### 4. Pull vs Push Consumers

**Pull** (recommended for new code):
- Client requests batches on demand
- Natural backpressure
- Horizontally scalable

**Push** (legacy):
- Server delivers to a subject
- Simpler for some patterns
- Less control over flow

### 5. Subject Filtering

Consumers can filter stream subjects:
```go
jetstream.ConsumerConfig{
    FilterSubject: "events.us.>",  // Only US events
}
```

### 6. Retention Policies

| Policy | Behavior |
|--------|----------|
| `LimitsPolicy` | Keep until limits exceeded (default) |
| `WorkQueuePolicy` | Delete after ack (exactly-once) |
| `InterestPolicy` | Delete when all consumers ack |

## Common Gotchas

1. **Work queue streams require non-overlapping consumers**: Multiple unfiltered consumers on a work queue stream will error. Use `FilterSubject` to partition.

2. **Durable consumers persist**: They don't auto-delete. Clean them up explicitly with `DeleteConsumer()`.

3. **JetStream publish vs Core publish**: Use `js.Publish()` for durability guarantees. Core NATS `nc.Publish()` won't wait for storage confirmation.

4. **MaxAckPending limits parallelism**: Default is 1000. Increase for high-throughput consumers.

5. **Message IDs for deduplication**: Set `Nats-Msg-Id` header for exactly-once publishing within the deduplication window.

## Skill Contents

- `concepts/` - Deep dives on streams, consumers, subjects, acknowledgment
- `patterns/` - Work queues, fan-out, exactly-once, event sourcing
- `reference/` - Stream config, consumer config, CLI commands
- `sdks/` - Go SDK patterns

## Links

- [NATS Documentation](https://docs.nats.io)
- [JetStream Concepts](https://docs.nats.io/nats-concepts/jetstream)
- [NATS by Example](https://natsbyexample.com)
- [Go Client](https://github.com/nats-io/nats.go)
