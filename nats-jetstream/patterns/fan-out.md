# Fan-Out Pattern

Deliver the same messages to multiple independent consumers. Each consumer receives all messages.

## When to Use

- Multiple services need the same events
- Audit/logging alongside processing
- Real-time dashboards and analytics
- Event-driven microservices

## Stream Configuration

```go
stream, _ := js.CreateOrUpdateStream(ctx, jetstream.StreamConfig{
    Name:      "EVENTS",
    Subjects:  []string{"events.>"},
    Retention: jetstream.LimitsPolicy, // Keep messages for replay
    MaxAge:    7 * 24 * time.Hour,
    Storage:   jetstream.FileStorage,
})
```

**LimitsPolicy** retains messages based on limits, allowing multiple consumers to read the same messages.

## Basic Fan-Out

### Publisher
```go
js.Publish(ctx, "events.order.created", orderData)
```

### Multiple Independent Consumers
```go
// Order processor - starts from where it left off
orderProcessor, _ := stream.CreateOrUpdateConsumer(ctx, jetstream.ConsumerConfig{
    Durable:       "order-processor",
    DeliverPolicy: jetstream.DeliverAllPolicy,
})

// Analytics - starts from where it left off
analytics, _ := stream.CreateOrUpdateConsumer(ctx, jetstream.ConsumerConfig{
    Durable:       "analytics",
    DeliverPolicy: jetstream.DeliverAllPolicy,
})

// Audit log - needs all historical events
audit, _ := stream.CreateOrUpdateConsumer(ctx, jetstream.ConsumerConfig{
    Durable:       "audit",
    DeliverPolicy: jetstream.DeliverAllPolicy,
})
```

Each consumer:
- Has its own cursor position
- Receives all messages (or filtered subset)
- Acknowledges independently
- Can replay from any point

## Filtered Fan-Out

Different consumers interested in different event types:

```go
// Only order events
orderConsumer, _ := stream.CreateOrUpdateConsumer(ctx, jetstream.ConsumerConfig{
    Durable:       "order-service",
    FilterSubject: "events.order.>",
})

// Only payment events
paymentConsumer, _ := stream.CreateOrUpdateConsumer(ctx, jetstream.ConsumerConfig{
    Durable:       "payment-service",
    FilterSubject: "events.payment.>",
})

// All user-related events
userConsumer, _ := stream.CreateOrUpdateConsumer(ctx, jetstream.ConsumerConfig{
    Durable:       "user-service",
    FilterSubjects: []string{"events.user.>", "events.auth.>"},
})
```

## Real-Time + Replay

### Ordered Consumer for Real-Time
For monitoring/dashboards that only need new events:

```go
// Ephemeral, no persistence, auto flow control
monitor, _ := stream.OrderedConsumer(ctx, jetstream.OrderedConsumerConfig{
    DeliverPolicy: jetstream.DeliverNewPolicy,
})
```

### Durable Consumer for Processing
For services that need to catch up after downtime:

```go
processor, _ := stream.CreateOrUpdateConsumer(ctx, jetstream.ConsumerConfig{
    Durable:       "processor",
    DeliverPolicy: jetstream.DeliverAllPolicy, // Resume from last ack
})
```

## Republish for Lightweight Subscribers

For high fan-out to many lightweight subscribers (IoT, mobile), use republish to avoid consumer overhead:

```go
stream, _ := js.CreateOrUpdateStream(ctx, jetstream.StreamConfig{
    Name:      "SENSOR_DATA",
    Subjects:  []string{"sensors.>"},
    RePublish: &jetstream.RePublish{
        Source:      ">",
        Destination: "broadcast.{{subject}}",
    },
})
```

Lightweight subscribers use Core NATS:
```go
nc.Subscribe("broadcast.sensors.>", func(msg *nats.Msg) {
    // Real-time, no ack needed
})
```

## Interest-Based Retention

For fan-out where messages should be deleted after all consumers have processed:

```go
stream, _ := js.CreateOrUpdateStream(ctx, jetstream.StreamConfig{
    Name:      "NOTIFICATIONS",
    Subjects:  []string{"notify.>"},
    Retention: jetstream.InterestPolicy,
})
```

Messages are deleted when all active consumers have acknowledged.

## Scaling Consumers

Each durable consumer can have multiple instances pulling messages:

```go
// Same consumer, multiple worker instances
cons, _ := stream.Consumer(ctx, "order-processor")

// Instance 1
go func() {
    msgs, _ := cons.Fetch(10)
    // Process batch
}()

// Instance 2
go func() {
    msgs, _ := cons.Fetch(10)
    // Process batch
}()
```

Messages are distributed across instances (within `MaxAckPending` limit).

## Best Practices

1. **Use durable consumers**: For reliable processing that survives restarts
2. **Set appropriate MaxAge**: Balance storage vs replay capability
3. **Filter when possible**: Reduce processing overhead
4. **Consider republish**: For many lightweight subscribers
5. **Monitor consumer lag**: Watch `NumPending` to detect slow consumers

## Anti-Patterns

1. **Too many consumers**: Beyond ~100k consumers causes instability
2. **No retention limits**: Unbounded storage growth
3. **Polling consumer info**: Use message metadata instead
4. **Ephemeral for critical processing**: Use durable consumers

## Event-Driven Architecture

Fan-out enables loose coupling:

```
                    ┌──────────────────┐
                    │  Order Service   │
                    │  (order-proc)    │
                    └────────▲─────────┘
                             │
┌─────────┐    ┌─────────────┴─────────────┐
│Publisher│───►│     EVENTS Stream         │
└─────────┘    └─────────────┬─────────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
        ▼                    ▼                    ▼
┌───────────────┐  ┌─────────────────┐  ┌────────────────┐
│ Notification  │  │    Analytics    │  │   Audit Log    │
│   Service     │  │    Service      │  │    Service     │
└───────────────┘  └─────────────────┘  └────────────────┘
```
