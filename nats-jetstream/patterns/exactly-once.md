# Exactly-Once Pattern

Achieve exactly-once message processing through deduplication and idempotent operations.

## Understanding "Exactly-Once"

JetStream provides:
- **At-least-once delivery**: Messages are redelivered until acknowledged
- **Exactly-once semantics**: Through deduplication + idempotent consumers

True exactly-once requires both sides:
1. **Publisher**: Deduplication prevents duplicate storage
2. **Consumer**: Idempotent processing handles potential redeliveries

## Publisher-Side Deduplication

### Message ID Header

Set `Nats-Msg-Id` header for deduplication:

```go
msgID := fmt.Sprintf("order-%s-%d", orderID, version)
js.Publish(ctx, "orders.created", data,
    jetstream.WithMsgID(msgID))
```

Server tracks message IDs within the deduplication window and rejects duplicates.

### Deduplication Window

Configure on the stream:

```go
stream, _ := js.CreateOrUpdateStream(ctx, jetstream.StreamConfig{
    Name:       "ORDERS",
    Subjects:   []string{"orders.>"},
    Duplicates: 2 * time.Minute, // Default: 2 minutes
})
```

Choose window based on:
- Network latency and retry patterns
- Publisher restart time
- Balance memory usage vs. dedup coverage

### Idempotent Publish

```go
func publishOnce(ctx context.Context, subject string, data []byte, msgID string) error {
    ack, err := js.Publish(ctx, subject, data,
        jetstream.WithMsgID(msgID))
    if err != nil {
        return err
    }

    if ack.Duplicate {
        // Already stored, not an error
        log.Printf("Duplicate message: %s", msgID)
    }
    return nil
}
```

## Consumer-Side Idempotency

### Double Ack

Wait for server confirmation of acknowledgment:

```go
err := msg.DoubleAck(ctx)
if err != nil {
    // Ack wasn't confirmed, message may redeliver
    // Don't commit side effects yet
}
```

### Idempotent Processing Pattern

```go
func processMessage(ctx context.Context, msg jetstream.Msg) error {
    meta, _ := msg.Metadata()

    // Extract idempotency key
    msgID := msg.Headers().Get("Nats-Msg-Id")
    if msgID == "" {
        msgID = fmt.Sprintf("%d", meta.Sequence.Stream)
    }

    // Check if already processed (in your database)
    if alreadyProcessed(ctx, msgID) {
        msg.Ack()
        return nil
    }

    // Process within transaction
    tx, _ := db.BeginTx(ctx, nil)
    defer tx.Rollback()

    // Do work
    err := doBusinessLogic(tx, msg.Data())
    if err != nil {
        msg.Nak()
        return err
    }

    // Record as processed (idempotency key)
    err = markProcessed(tx, msgID)
    if err != nil {
        msg.Nak()
        return err
    }

    // Commit
    if err := tx.Commit(); err != nil {
        msg.Nak()
        return err
    }

    // Ack after successful commit
    msg.Ack()
    return nil
}
```

### Outbox Pattern

For exactly-once with external systems:

```go
func processWithOutbox(ctx context.Context, msg jetstream.Msg) error {
    tx, _ := db.BeginTx(ctx, nil)
    defer tx.Rollback()

    // 1. Process message
    result, err := processOrder(tx, msg.Data())
    if err != nil {
        msg.Nak()
        return err
    }

    // 2. Write to outbox (same transaction)
    outboxMsg := OutboxMessage{
        Subject: "events.order.processed",
        Data:    result,
        MsgID:   generateMsgID(),
    }
    err = insertOutbox(tx, outboxMsg)
    if err != nil {
        msg.Nak()
        return err
    }

    // 3. Commit
    if err := tx.Commit(); err != nil {
        msg.Nak()
        return err
    }

    msg.Ack()
    return nil
}

// Separate process publishes from outbox
func publishOutbox(ctx context.Context) {
    for {
        msgs, _ := getUnpublishedOutbox(ctx)
        for _, m := range msgs {
            _, err := js.Publish(ctx, m.Subject, m.Data,
                jetstream.WithMsgID(m.MsgID))
            if err == nil {
                markOutboxPublished(ctx, m.ID)
            }
        }
        time.Sleep(100 * time.Millisecond)
    }
}
```

## Work Queue + Exactly-Once

Combine work queue retention with deduplication:

```go
stream, _ := js.CreateOrUpdateStream(ctx, jetstream.StreamConfig{
    Name:       "TASKS",
    Subjects:   []string{"tasks.>"},
    Retention:  jetstream.WorkQueuePolicy,
    Duplicates: 5 * time.Minute,
})
```

Messages are:
1. Deduplicated on publish
2. Deleted after acknowledgment (exactly-once consumption)

## Expected Sequence Headers

For optimistic concurrency control:

```go
// Expect specific last sequence for stream
js.Publish(ctx, "orders.123", data,
    jetstream.WithExpectLastSequence(expectedSeq))

// Expect specific last sequence for subject
js.Publish(ctx, "orders.123", data,
    jetstream.WithExpectLastSubjectSequence(expectedSubjectSeq))

// Expect specific last message ID
js.Publish(ctx, "orders.123", data,
    jetstream.WithExpectLastMsgID(expectedMsgID))
```

If expectation fails, publish returns error - useful for conditional updates.

## Best Practices

1. **Always use message IDs**: Generate deterministic IDs (e.g., `entity-id-version`)
2. **Size dedup window appropriately**: Cover your retry/restart scenarios
3. **Make consumers idempotent**: Don't rely solely on JetStream dedup
4. **Use DoubleAck for critical paths**: Confirm ack before side effects
5. **Store idempotency keys**: Track processed messages in your database
6. **Use transactions**: Combine processing + idempotency check atomically

## Anti-Patterns

1. **Random message IDs**: Use deterministic IDs based on content/entity
2. **Trusting single ack**: Network issues can cause redelivery
3. **Non-idempotent side effects**: External API calls, emails, etc.
4. **Very short dedup window**: May not cover restart scenarios
5. **Very long dedup window**: Memory overhead on server

## Message ID Generation

```go
// Entity-based (recommended)
msgID := fmt.Sprintf("%s-%s-%d", entityType, entityID, version)

// Content-based hash
hash := sha256.Sum256(data)
msgID := hex.EncodeToString(hash[:16])

// UUID (only if truly unique event)
msgID := uuid.New().String()
```
