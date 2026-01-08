# Stream Configuration Reference

Complete reference for JetStream stream configuration options.

## StreamConfig Struct (Go)

```go
type StreamConfig struct {
    Name                 string
    Description          string
    Subjects             []string
    Retention            RetentionPolicy
    MaxConsumers         int
    MaxMsgs              int64
    MaxBytes             int64
    MaxAge               time.Duration
    MaxMsgsPerSubject    int64
    MaxMsgSize           int32
    Discard              DiscardPolicy
    DiscardNewPerSubject bool
    Storage              StorageType
    Replicas             int
    NoAck                bool
    Duplicates           time.Duration
    Placement            *Placement
    Mirror               *StreamSource
    Sources              []*StreamSource
    Sealed               bool
    DenyDelete           bool
    DenyPurge            bool
    AllowRollup          bool
    Compression          StoreCompression
    FirstSeq             uint64
    SubjectTransform     *SubjectTransformConfig
    RePublish            *RePublish
    AllowDirect          bool
    MirrorDirect         bool
    Metadata             map[string]string
}
```

## Core Settings

### Name
Stream identifier. Required.

```go
Name: "ORDERS"
```

- Cannot contain: whitespace, `.`, `*`, `>`, path separators
- Convention: UPPERCASE

### Description
Human-readable description.

```go
Description: "Order events for the e-commerce platform"
```

### Subjects
Subject patterns to capture. Required.

```go
Subjects: []string{"orders.>", "payments.*"}
```

Supports wildcards: `*` (single token), `>` (multi-token, must be last).

## Retention

### Retention Policy

| Value | Behavior |
|-------|----------|
| `LimitsPolicy` | Keep until limits exceeded (default) |
| `InterestPolicy` | Delete when all consumers ack |
| `WorkQueuePolicy` | Delete after any consumer acks |

```go
Retention: jetstream.LimitsPolicy
```

### MaxMsgs
Maximum number of messages.

```go
MaxMsgs: 1000000 // 1M messages
```

### MaxBytes
Maximum total size in bytes.

```go
MaxBytes: 1024 * 1024 * 1024 // 1GB
```

### MaxAge
Maximum message age.

```go
MaxAge: 7 * 24 * time.Hour // 7 days
```

### MaxMsgsPerSubject
Per-subject message limit.

```go
MaxMsgsPerSubject: 1000
```

### MaxMsgSize
Maximum individual message size.

```go
MaxMsgSize: 1024 * 1024 // 1MB
```

### Discard Policy

| Value | Behavior |
|-------|----------|
| `DiscardOld` | Remove oldest when limit reached (default) |
| `DiscardNew` | Reject new messages when limit reached |

```go
Discard: jetstream.DiscardOld
```

### DiscardNewPerSubject
Per-subject discard for `DiscardNew`.

```go
DiscardNewPerSubject: true
```

## Storage

### Storage Type

| Value | Description |
|-------|-------------|
| `FileStorage` | Disk-based (default) |
| `MemoryStorage` | RAM-based |

```go
Storage: jetstream.FileStorage
```

### Compression

| Value | Description |
|-------|-------------|
| `NoCompression` | No compression (default) |
| `S2Compression` | Snappy compression |

```go
Compression: jetstream.S2Compression
```

### Replicas
Number of replicas for fault tolerance (clustered mode).

```go
Replicas: 3 // 1, 3, or 5 recommended
```

## Deduplication

### Duplicates
Deduplication window for `Nats-Msg-Id` headers.

```go
Duplicates: 2 * time.Minute // Default
```

## Placement

### Placement
Control stream placement in cluster.

```go
Placement: &jetstream.Placement{
    Cluster: "us-east",
    Tags:    []string{"ssd"},
}
```

## Mirroring & Sourcing

### Mirror
Create read-only replica of another stream.

```go
Mirror: &jetstream.StreamSource{
    Name:          "ORDERS",
    FilterSubject: "orders.us.>",
}
```

### Sources
Aggregate from multiple streams.

```go
Sources: []*jetstream.StreamSource{
    {Name: "ORDERS_US"},
    {Name: "ORDERS_EU"},
}
```

## Subject Transform

### SubjectTransform
Transform subjects before storage.

```go
SubjectTransform: &jetstream.SubjectTransformConfig{
    Source:      "raw.>",
    Destination: "processed.{{wildcard(1)}}",
}
```

## Republish

### RePublish
Republish stored messages to another subject.

```go
RePublish: &jetstream.RePublish{
    Source:      ">",
    Destination: "archive.{{subject}}",
    HeadersOnly: false,
}
```

## Access Control

### MaxConsumers
Maximum number of consumers.

```go
MaxConsumers: 100
```

### Sealed
Prevent further modifications.

```go
Sealed: true
```

### DenyDelete
Prevent message deletion via API.

```go
DenyDelete: true
```

### DenyPurge
Prevent stream purging.

```go
DenyPurge: true
```

### AllowRollup
Enable `Nats-Rollup` header for subject rollup.

```go
AllowRollup: true
```

## Direct Access

### AllowDirect
Enable direct get API for faster reads.

```go
AllowDirect: true
```

### MirrorDirect
Enable direct get on mirrors.

```go
MirrorDirect: true
```

## Metadata

### Metadata
Custom key-value metadata.

```go
Metadata: map[string]string{
    "owner": "order-team",
    "env":   "production",
}
```

## CLI Create Options

```bash
nats stream add ORDERS \
    --subjects "orders.>" \
    --storage file \
    --retention limits \
    --max-msgs 1000000 \
    --max-bytes 1GB \
    --max-age 7d \
    --max-msg-size 1MB \
    --discard old \
    --replicas 3 \
    --compression s2 \
    --dupe-window 2m
```

## Common Configurations

### High-Volume Event Store
```go
jetstream.StreamConfig{
    Name:        "EVENTS",
    Subjects:    []string{"events.>"},
    Storage:     jetstream.FileStorage,
    Retention:   jetstream.LimitsPolicy,
    MaxAge:      30 * 24 * time.Hour,
    MaxBytes:    100 * 1024 * 1024 * 1024, // 100GB
    Replicas:    3,
    Compression: jetstream.S2Compression,
    Duplicates:  5 * time.Minute,
}
```

### Work Queue
```go
jetstream.StreamConfig{
    Name:      "TASKS",
    Subjects:  []string{"tasks.>"},
    Storage:   jetstream.FileStorage,
    Retention: jetstream.WorkQueuePolicy,
    MaxAge:    24 * time.Hour,
    Replicas:  3,
}
```

### Real-Time Cache
```go
jetstream.StreamConfig{
    Name:              "CACHE",
    Subjects:          []string{"cache.>"},
    Storage:           jetstream.MemoryStorage,
    Retention:         jetstream.LimitsPolicy,
    MaxMsgsPerSubject: 1, // Only latest per key
    MaxAge:            5 * time.Minute,
    AllowDirect:       true,
}
```
