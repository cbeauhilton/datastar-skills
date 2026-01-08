# Work Queue Pattern

Distribute work across multiple competing consumers where each message is processed exactly once.

## When to Use

- Task distribution (background jobs, async processing)
- Load balancing across workers
- Exactly-once consumption semantics
- Scaling processing horizontally

## Stream Configuration

```go
stream, _ := js.CreateOrUpdateStream(ctx, jetstream.StreamConfig{
    Name:      "TASKS",
    Subjects:  []string{"tasks.>"},
    Retention: jetstream.WorkQueuePolicy, // Key: delete after ack
    Storage:   jetstream.FileStorage,
})
```

**WorkQueuePolicy** deletes messages after acknowledgment, ensuring each message is processed once.

## Basic Pattern

### Publisher
```go
// Publish tasks
js.Publish(ctx, "tasks.email", []byte(`{"to": "user@example.com"}`))
js.Publish(ctx, "tasks.resize", []byte(`{"image": "photo.jpg"}`))
```

### Worker
```go
cons, _ := stream.CreateOrUpdateConsumer(ctx, jetstream.ConsumerConfig{
    Durable:   "worker",
    AckPolicy: jetstream.AckExplicitPolicy,
})

// Multiple instances share the same durable name
msgs, _ := cons.Fetch(10)
for msg := range msgs.Messages() {
    err := processTask(msg.Data())
    if err != nil {
        msg.Nak() // Retry
    } else {
        msg.Ack() // Done, removed from queue
    }
}
```

## Partitioned Work Queue

For high throughput, partition by subject and use filtered consumers:

### Subject Design
```
tasks.<partition>.<type>
tasks.0.email
tasks.1.resize
tasks.2.notify
```

### Partitioned Consumers
```go
// Worker for partition 0
cons0, _ := stream.CreateOrUpdateConsumer(ctx, jetstream.ConsumerConfig{
    Durable:       "worker-0",
    FilterSubject: "tasks.0.>",
})

// Worker for partition 1
cons1, _ := stream.CreateOrUpdateConsumer(ctx, jetstream.ConsumerConfig{
    Durable:       "worker-1",
    FilterSubject: "tasks.1.>",
})
```

**Critical**: Work queue streams require non-overlapping `FilterSubject` values between consumers. Multiple unfiltered consumers will error.

### Publisher with Partitioning
```go
func publishTask(ctx context.Context, taskType string, data []byte) {
    // Hash-based partitioning
    partition := hash(taskType) % numPartitions
    subject := fmt.Sprintf("tasks.%d.%s", partition, taskType)
    js.Publish(ctx, subject, data)
}
```

## Priority Queues

Use subjects for priority levels:

```go
// High priority stream with shorter retention
highStream, _ := js.CreateOrUpdateStream(ctx, jetstream.StreamConfig{
    Name:      "TASKS_HIGH",
    Subjects:  []string{"tasks.high.>"},
    Retention: jetstream.WorkQueuePolicy,
})

// Normal priority
normalStream, _ := js.CreateOrUpdateStream(ctx, jetstream.StreamConfig{
    Name:      "TASKS_NORMAL",
    Subjects:  []string{"tasks.normal.>"},
    Retention: jetstream.WorkQueuePolicy,
})
```

Workers poll high-priority first:
```go
for {
    // Try high priority first
    msgs, _ := highConsumer.Fetch(10, jetstream.FetchMaxWait(100*time.Millisecond))
    if msgs.Error() == nil {
        processMessages(msgs)
        continue
    }

    // Fall back to normal
    msgs, _ = normalConsumer.Fetch(10)
    processMessages(msgs)
}
```

## Delayed/Scheduled Tasks

Use `NakWithDelay` for delayed retry or republish with future timestamp:

```go
// Immediate retry with backoff
msg.NakWithDelay(30 * time.Second)

// Or use scheduled subject pattern
func scheduleTask(ctx context.Context, at time.Time, data []byte) {
    // External scheduler republishes when time arrives
    js.Publish(ctx, "scheduled.tasks", data,
        jetstream.WithMsgHeader("X-Execute-At", at.Format(time.RFC3339)))
}
```

## Best Practices

1. **Use explicit acks**: Always `AckExplicit` for work queues
2. **Set reasonable timeouts**: Configure `AckWait` based on task duration
3. **Handle poison messages**: Use `MaxDeliver` and `Term()` for unprocessable messages
4. **Partition for scale**: Use filtered consumers to parallelize
5. **Idempotent processing**: Design tasks to handle potential redelivery

## Anti-Patterns

1. **Multiple unfiltered consumers**: Work queue streams don't allow this
2. **Very long AckWait**: Delays failure detection
3. **No MaxDeliver**: Infinite retries can block queue
4. **Stateful workers**: Workers should be stateless and interchangeable

## CLI Testing

```bash
# Create work queue stream
nats stream add TASKS --subjects "tasks.>" --retention work

# Publish task
nats pub tasks.email '{"to": "test@example.com"}'

# Create worker consumer
nats consumer add TASKS worker --pull --ack explicit

# Process one task
nats consumer next TASKS worker
```
