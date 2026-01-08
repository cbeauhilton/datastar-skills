# JetStream Acknowledgment

Acknowledgment is how consumers tell JetStream they've successfully processed a message. It's the foundation of at-least-once delivery guarantees.

## Why Ack Matters

Without acknowledgment:
- Messages might be lost if consumer crashes
- No way to retry failed processing
- No delivery guarantees

With acknowledgment:
- Unacked messages are redelivered
- Failed processing triggers retry
- At-least-once delivery guaranteed

## Ack Policies

### AckExplicit (Default)
Each message must be individually acknowledged.

```go
jetstream.ConsumerConfig{
    AckPolicy: jetstream.AckExplicitPolicy,
}
```

Best for:
- Critical workloads
- Variable processing times
- When you need per-message control

### AckAll
Acknowledging a message acks all messages before it in the sequence.

```go
jetstream.ConsumerConfig{
    AckPolicy: jetstream.AckAllPolicy,
}
```

Best for:
- Batch processing
- Ordered consumption
- When messages are processed sequentially

### AckNone
No acknowledgment required. Fire-and-forget.

```go
jetstream.ConsumerConfig{
    AckPolicy: jetstream.AckNonePolicy,
}
```

Best for:
- Monitoring/logging
- Non-critical data
- When loss is acceptable

## Ack Types

### Ack - Success
Message processed successfully. Remove from pending.

```go
msg.Ack()
```

### DoubleAck - Confirmed Success
Wait for server confirmation of the ack.

```go
err := msg.DoubleAck(ctx)
if err != nil {
    // Ack wasn't confirmed, message may redeliver
}
```

### Nak - Retry Now
Processing failed. Redeliver immediately (respects BackOff if configured).

```go
msg.Nak()

// With delay
msg.NakWithDelay(5 * time.Second)
```

### InProgress - Extend Deadline
Still processing. Reset the ack wait timer.

```go
msg.InProgress()
```

Use for long-running operations to prevent redelivery while still processing.

### Term - Terminate
Stop redelivery permanently. Use for poison messages.

```go
msg.Term()

// With reason (logged server-side)
msg.TermWithReason("invalid message format")
```

## Ack Wait

Time JetStream waits for an ack before redelivering.

```go
jetstream.ConsumerConfig{
    AckWait: 30 * time.Second, // Default: 30s
}
```

Choose based on expected processing time. Too short = unnecessary redeliveries. Too long = slow recovery from crashes.

## Redelivery Control

### MaxDeliver
Maximum delivery attempts. After this, message is dropped (or sent to dead letter if configured).

```go
jetstream.ConsumerConfig{
    MaxDeliver: 5,
}
```

Set to -1 for unlimited retries.

### BackOff
Custom delay sequence between redeliveries.

```go
jetstream.ConsumerConfig{
    BackOff: []time.Duration{
        1 * time.Second,   // 1st retry
        5 * time.Second,   // 2nd retry
        30 * time.Second,  // 3rd retry
        2 * time.Minute,   // 4th+ retry
    },
}
```

If retries exceed BackOff length, last value repeats.

## MaxAckPending

Limits concurrent unacknowledged messages.

```go
jetstream.ConsumerConfig{
    MaxAckPending: 1000, // Default
}
```

This provides backpressure. Lower values = more ordered processing. Higher values = more parallelism.

## Ack Patterns

### Simple Processing
```go
for msg := range msgs.Messages() {
    err := processMessage(msg)
    if err != nil {
        msg.Nak()
        continue
    }
    msg.Ack()
}
```

### With Timeout Handling
```go
for msg := range msgs.Messages() {
    // Start processing
    done := make(chan error)
    go func() {
        done <- processMessage(msg)
    }()

    // Extend deadline while processing
    ticker := time.NewTicker(10 * time.Second)
    defer ticker.Stop()

    for {
        select {
        case err := <-done:
            if err != nil {
                msg.Nak()
            } else {
                msg.Ack()
            }
            break
        case <-ticker.C:
            msg.InProgress()
        }
    }
}
```

### Poison Message Handling
```go
for msg := range msgs.Messages() {
    meta, _ := msg.Metadata()

    // Check delivery count
    if meta.NumDelivered > 3 {
        log.Printf("Poison message detected: %s", msg.Subject())
        msg.Term()
        // Optionally: send to dead letter queue
        continue
    }

    err := processMessage(msg)
    if err != nil {
        msg.Nak()
        continue
    }
    msg.Ack()
}
```

### Batch Ack (AckAll policy)
```go
msgs, _ := cons.Fetch(100)
var lastMsg jetstream.Msg

for msg := range msgs.Messages() {
    processMessage(msg)
    lastMsg = msg
}

// Ack all at once
if lastMsg != nil {
    lastMsg.Ack()
}
```

## Message Metadata

Access delivery information:

```go
meta, _ := msg.Metadata()
meta.Sequence.Stream    // Stream sequence number
meta.Sequence.Consumer  // Consumer sequence number
meta.NumDelivered       // Delivery attempt count
meta.NumPending         // Messages pending for consumer
meta.Timestamp          // Original publish time
```

## Dead Letter Queues

JetStream doesn't have built-in DLQ, but you can implement:

```go
if meta.NumDelivered >= maxDeliveries {
    // Publish to DLQ stream
    js.Publish(ctx, "dlq.orders", msg.Data())
    msg.Term()
}
```
