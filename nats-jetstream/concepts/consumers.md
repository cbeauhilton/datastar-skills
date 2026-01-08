# JetStream Consumers

A consumer is a stateful view of a stream that tracks message delivery and acknowledgments. Consumers provide the read interface to streams.

## Core Concept

Consumers:
- Track position (cursor) in the stream
- Manage acknowledgments and redelivery
- Filter messages by subject
- Control replay behavior

Multiple consumers can read from the same stream independently, each with their own position and configuration.

## Consumer Types

### Durable Consumers
- Explicitly named
- Persisted across server restarts
- Survive client disconnections
- Must be explicitly deleted

```go
cons, _ := stream.CreateOrUpdateConsumer(ctx, jetstream.ConsumerConfig{
    Durable: "order-processor",
})
```

### Ephemeral Consumers
- Auto-generated name (or omit `Durable`)
- Memory-only, no fault tolerance
- Auto-deleted after inactivity period

```go
cons, _ := stream.CreateOrUpdateConsumer(ctx, jetstream.ConsumerConfig{
    InactiveThreshold: 5 * time.Minute,
})
```

### Ordered Consumers
Convenient default type with:
- Always ephemeral
- No acknowledgments required
- Automatic flow control
- Single active subscriber
- Automatic recovery from disconnects

```go
cons, _ := stream.OrderedConsumer(ctx, jetstream.OrderedConsumerConfig{
    FilterSubjects: []string{"events.>"},
})
```

## Dispatch Types

### Pull Consumers (Recommended)
Client requests messages on demand. Provides natural backpressure and horizontal scalability.

```go
// Fetch batch
msgs, _ := cons.Fetch(10)
for msg := range msgs.Messages() {
    // Process
    msg.Ack()
}

// Continuous consumption
iter, _ := cons.Messages()
for {
    msg, _ := iter.Next()
    // Process
    msg.Ack()
}

// Callback-based
cons.Consume(func(msg jetstream.Msg) {
    // Process
    msg.Ack()
})
```

### Push Consumers (Legacy)
Server pushes messages to a delivery subject. Simpler but less control.

```go
jetstream.ConsumerConfig{
    DeliverSubject: "my.delivery.subject",
}
```

## Delivery Policies

Where to start reading:

| Policy | Description |
|--------|-------------|
| `DeliverAll` | From the beginning (default) |
| `DeliverLast` | Last message only |
| `DeliverLastPerSubject` | Last message per filtered subject |
| `DeliverNew` | Only new messages after consumer creation |
| `DeliverByStartSequence` | From specific sequence number |
| `DeliverByStartTime` | From specific timestamp |

```go
jetstream.ConsumerConfig{
    DeliverPolicy: jetstream.DeliverNewPolicy,
}

// Or with specific start point
jetstream.ConsumerConfig{
    DeliverPolicy: jetstream.DeliverByStartSequencePolicy,
    OptStartSeq:   1000,
}
```

## Replay Policies

How fast to deliver during replay:

| Policy | Description |
|--------|-------------|
| `ReplayInstant` | As fast as possible (default) |
| `ReplayOriginal` | At original publication rate |

```go
jetstream.ConsumerConfig{
    ReplayPolicy: jetstream.ReplayOriginalPolicy,
}
```

## Subject Filtering

Consumers can filter which subjects they receive:

```go
// Single filter
jetstream.ConsumerConfig{
    FilterSubject: "orders.us.>",
}

// Multiple filters (v2.10+)
jetstream.ConsumerConfig{
    FilterSubjects: []string{"orders.us.>", "orders.ca.>"},
}
```

## Flow Control

### MaxAckPending
Limits outstanding unacknowledged messages across all subscribers.

```go
jetstream.ConsumerConfig{
    MaxAckPending: 1000, // Default
}
```

Increase for high-throughput scenarios; decrease for ordered processing.

### AckWait
Time to wait for acknowledgment before redelivery.

```go
jetstream.ConsumerConfig{
    AckWait: 30 * time.Second,
}
```

### MaxDeliver
Maximum delivery attempts before giving up.

```go
jetstream.ConsumerConfig{
    MaxDeliver: 5,
}
```

### BackOff
Custom redelivery delay sequence:

```go
jetstream.ConsumerConfig{
    BackOff: []time.Duration{
        1 * time.Second,
        5 * time.Second,
        30 * time.Second,
    },
}
```

## Headers Only Mode

Receive only headers, not payload (for routing decisions):

```go
jetstream.ConsumerConfig{
    HeadersOnly: true,
}
```

## Creating Consumers (Go)

```go
// Create durable pull consumer
cons, _ := stream.CreateOrUpdateConsumer(ctx, jetstream.ConsumerConfig{
    Durable:       "processor",
    AckPolicy:     jetstream.AckExplicitPolicy,
    DeliverPolicy: jetstream.DeliverAllPolicy,
    FilterSubject: "orders.>",
    MaxAckPending: 1000,
    AckWait:       30 * time.Second,
})

// Get existing consumer
cons, _ := stream.Consumer(ctx, "processor")

// Consumer info
info, _ := cons.Info(ctx)
fmt.Printf("Pending: %d, Redelivered: %d\n",
    info.NumPending, info.NumRedelivered)

// Delete consumer
stream.DeleteConsumer(ctx, "processor")
```

## CLI Commands

```bash
# Create pull consumer
nats consumer add ORDERS processor --pull --ack explicit --deliver all

# List consumers
nats consumer ls ORDERS

# Consumer info
nats consumer info ORDERS processor

# Get next message
nats consumer next ORDERS processor

# Delete consumer
nats consumer rm ORDERS processor -f
```
