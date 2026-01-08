# JetStream Streams

Streams are append-only message logs that capture and store messages published to NATS subjects. They provide the persistence layer in JetStream.

## Core Concept

A stream:
- Listens for messages on configured subjects
- Stores messages in sequence order
- Provides replay capabilities to consumers
- Manages retention based on policies and limits

## Storage Types

### File Storage (Default)
```go
jetstream.StreamConfig{
    Storage: jetstream.FileStorage,
}
```
- Persists to disk
- Survives server restarts
- Optional Snappy compression (`Compression: jetstream.S2Compression`)

### Memory Storage
```go
jetstream.StreamConfig{
    Storage: jetstream.MemoryStorage,
}
```
- RAM-only, fastest
- Lost on restart
- Use for ephemeral/cache scenarios

## Retention Policies

### LimitsPolicy (Default)
Messages kept until limits exceeded, then oldest removed.

```go
jetstream.StreamConfig{
    Retention: jetstream.LimitsPolicy,
    MaxMsgs:   10000,
    MaxBytes:  1024 * 1024 * 100, // 100MB
    MaxAge:    24 * time.Hour,
}
```

### WorkQueuePolicy
Messages deleted after acknowledgment. Enables exactly-once consumption.

```go
jetstream.StreamConfig{
    Retention: jetstream.WorkQueuePolicy,
}
```

**Constraint**: Multiple consumers must have non-overlapping `FilterSubject` values.

### InterestPolicy
Messages deleted when all active consumers have acknowledged.

```go
jetstream.StreamConfig{
    Retention: jetstream.InterestPolicy,
}
```

## Limits

| Limit | Description |
|-------|-------------|
| `MaxMsgs` | Maximum message count |
| `MaxBytes` | Maximum total size |
| `MaxAge` | Maximum message age |
| `MaxMsgSize` | Maximum single message size |
| `MaxMsgsPerSubject` | Per-subject message limit |

When multiple limits are set, whichever triggers first applies.

## Discard Policies

When limits are reached:

### DiscardOld (Default)
Remove oldest messages to make room.

### DiscardNew
Reject new publishes when full.

### DiscardNewPerSubject
Reject per-subject when that subject hits its limit.

## Replication

```go
jetstream.StreamConfig{
    Replicas: 3, // Max 5
}
```

- Replicas provide fault tolerance in clustered deployments
- Uses NATS-optimized RAFT consensus
- Odd numbers recommended (1, 3, 5)

## Subject Wildcards

Streams capture messages by subject patterns:

```go
jetstream.StreamConfig{
    Subjects: []string{
        "orders.>",        // All orders hierarchy
        "events.*",        // One level under events
        "logs.app.error",  // Exact match
    },
}
```

## Advanced Features

### Deduplication Window
```go
jetstream.StreamConfig{
    Duplicates: 2 * time.Minute,
}
```
Track `Nats-Msg-Id` headers for this duration to prevent duplicates.

### Republish
Automatically republish stored messages to another subject:
```go
jetstream.StreamConfig{
    RePublish: &jetstream.RePublish{
        Source:      ">",
        Destination: "archive.{{subject}}",
    },
}
```

### Source/Mirror
- **Sources**: Aggregate from multiple streams
- **Mirror**: Read-only replica of another stream

## Creating Streams (Go)

```go
js, _ := jetstream.New(nc)

// Create or update idempotently
stream, err := js.CreateOrUpdateStream(ctx, jetstream.StreamConfig{
    Name:      "ORDERS",
    Subjects:  []string{"orders.>"},
    Storage:   jetstream.FileStorage,
    Retention: jetstream.LimitsPolicy,
    MaxAge:    7 * 24 * time.Hour,
    Replicas:  1,
})

// Get existing stream
stream, err := js.Stream(ctx, "ORDERS")

// Stream info
info, _ := stream.Info(ctx)
fmt.Printf("Messages: %d, Bytes: %d\n", info.State.Msgs, info.State.Bytes)
```

## CLI Commands

```bash
# Create stream
nats stream add ORDERS --subjects "orders.>" --storage file --retention limits

# List streams
nats stream ls

# Stream info
nats stream info ORDERS

# Purge all messages
nats stream purge ORDERS -f

# Delete stream
nats stream rm ORDERS -f
```
