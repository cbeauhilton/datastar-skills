# Consumer Configuration Reference

Complete reference for JetStream consumer configuration options.

## ConsumerConfig Struct (Go)

```go
type ConsumerConfig struct {
    Name               string
    Durable            string
    Description        string
    DeliverPolicy      DeliverPolicy
    OptStartSeq        uint64
    OptStartTime       *time.Time
    AckPolicy          AckPolicy
    AckWait            time.Duration
    MaxDeliver         int
    BackOff            []time.Duration
    FilterSubject      string
    FilterSubjects     []string
    ReplayPolicy       ReplayPolicy
    RateLimit          uint64
    SampleFrequency    string
    MaxWaiting         int
    MaxAckPending      int
    HeadersOnly        bool
    MaxRequestBatch    int
    MaxRequestExpires  time.Duration
    MaxRequestMaxBytes int
    InactiveThreshold  time.Duration
    Replicas           int
    MemoryStorage      bool
    Metadata           map[string]string

    // Push-specific
    DeliverSubject string
    DeliverGroup   string
    FlowControl    bool
    IdleHeartbeat  time.Duration
}
```

## Identity

### Name
Consumer identifier (auto-generated if not set).

```go
Name: "order-processor"
```

### Durable
Makes consumer persistent. If set, consumer survives restarts.

```go
Durable: "order-processor"
```

### Description
Human-readable description.

```go
Description: "Processes incoming orders"
```

## Delivery

### DeliverPolicy

| Value | Description |
|-------|-------------|
| `DeliverAllPolicy` | From beginning (default) |
| `DeliverLastPolicy` | Last message only |
| `DeliverLastPerSubjectPolicy` | Last per filtered subject |
| `DeliverNewPolicy` | Only new messages |
| `DeliverByStartSequencePolicy` | From sequence number |
| `DeliverByStartTimePolicy` | From timestamp |

```go
DeliverPolicy: jetstream.DeliverAllPolicy
```

### OptStartSeq
Start sequence for `DeliverByStartSequencePolicy`.

```go
DeliverPolicy: jetstream.DeliverByStartSequencePolicy,
OptStartSeq:   1000,
```

### OptStartTime
Start time for `DeliverByStartTimePolicy`.

```go
startTime := time.Now().Add(-1 * time.Hour)
DeliverPolicy: jetstream.DeliverByStartTimePolicy,
OptStartTime:  &startTime,
```

### ReplayPolicy

| Value | Description |
|-------|-------------|
| `ReplayInstantPolicy` | As fast as possible (default) |
| `ReplayOriginalPolicy` | At original publish rate |

```go
ReplayPolicy: jetstream.ReplayInstantPolicy
```

## Filtering

### FilterSubject
Single subject filter.

```go
FilterSubject: "orders.us.>"
```

### FilterSubjects
Multiple subject filters (NATS 2.10+).

```go
FilterSubjects: []string{"orders.us.>", "orders.ca.>"}
```

## Acknowledgment

### AckPolicy

| Value | Description |
|-------|-------------|
| `AckExplicitPolicy` | Each message must be acked (default) |
| `AckAllPolicy` | Ack last = ack all prior |
| `AckNonePolicy` | No ack required |

```go
AckPolicy: jetstream.AckExplicitPolicy
```

### AckWait
Time before redelivery if unacked.

```go
AckWait: 30 * time.Second // Default: 30s
```

### MaxDeliver
Maximum delivery attempts. -1 for unlimited.

```go
MaxDeliver: 5
```

### BackOff
Custom redelivery delay sequence.

```go
BackOff: []time.Duration{
    1 * time.Second,
    5 * time.Second,
    30 * time.Second,
    2 * time.Minute,
}
```

## Flow Control

### MaxAckPending
Maximum outstanding unacked messages.

```go
MaxAckPending: 1000 // Default
```

### MaxWaiting
Maximum pending pull requests.

```go
MaxWaiting: 512 // Default
```

### MaxRequestBatch
Maximum messages per pull request.

```go
MaxRequestBatch: 100
```

### MaxRequestExpires
Maximum pull request wait time.

```go
MaxRequestExpires: 30 * time.Second
```

### MaxRequestMaxBytes
Maximum bytes per pull request.

```go
MaxRequestMaxBytes: 1024 * 1024 // 1MB
```

### RateLimit
Bits per second rate limit (push consumers).

```go
RateLimit: 1024 * 1024 // 1 Mbps
```

## Special Modes

### HeadersOnly
Receive only headers, not payload.

```go
HeadersOnly: true
```

### SampleFrequency
Sampling percentage for observability.

```go
SampleFrequency: "50%" // Sample 50% of acks
```

## Push Consumer Settings

### DeliverSubject
Subject to deliver messages (makes it a push consumer).

```go
DeliverSubject: "deliver.orders"
```

### DeliverGroup
Queue group for load balancing.

```go
DeliverGroup: "workers"
```

### FlowControl
Enable flow control for push consumers.

```go
FlowControl: true
```

### IdleHeartbeat
Heartbeat interval for push consumers.

```go
IdleHeartbeat: 5 * time.Second
```

## Lifecycle

### InactiveThreshold
Delete ephemeral consumer after inactivity.

```go
InactiveThreshold: 5 * time.Minute
```

## Storage

### MemoryStorage
Store consumer state in memory only.

```go
MemoryStorage: true
```

### Replicas
Number of replicas for consumer state.

```go
Replicas: 3
```

## Metadata

### Metadata
Custom key-value metadata.

```go
Metadata: map[string]string{
    "team": "orders",
}
```

## CLI Create Options

```bash
# Pull consumer
nats consumer add ORDERS processor \
    --pull \
    --ack explicit \
    --deliver all \
    --filter "orders.>" \
    --max-deliver 5 \
    --max-pending 1000 \
    --wait 30s \
    --backoff "1s,5s,30s,2m"

# Push consumer
nats consumer add ORDERS monitor \
    --target deliver.orders \
    --ack none \
    --deliver last \
    --replay instant
```

## Common Configurations

### Reliable Pull Consumer
```go
jetstream.ConsumerConfig{
    Durable:       "order-processor",
    AckPolicy:     jetstream.AckExplicitPolicy,
    DeliverPolicy: jetstream.DeliverAllPolicy,
    AckWait:       30 * time.Second,
    MaxDeliver:    5,
    MaxAckPending: 1000,
    BackOff: []time.Duration{
        1 * time.Second,
        5 * time.Second,
        30 * time.Second,
    },
}
```

### Real-Time Monitor
```go
jetstream.ConsumerConfig{
    DeliverPolicy: jetstream.DeliverNewPolicy,
    AckPolicy:     jetstream.AckNonePolicy,
    // Ephemeral - no Durable set
    InactiveThreshold: 5 * time.Minute,
}
```

### Filtered Partitioned Worker
```go
jetstream.ConsumerConfig{
    Durable:       "worker-us",
    FilterSubject: "orders.us.>",
    AckPolicy:     jetstream.AckExplicitPolicy,
    MaxAckPending: 500,
}
```

### Replay Consumer
```go
startTime := time.Now().Add(-24 * time.Hour)
jetstream.ConsumerConfig{
    DeliverPolicy: jetstream.DeliverByStartTimePolicy,
    OptStartTime:  &startTime,
    ReplayPolicy:  jetstream.ReplayOriginalPolicy,
    AckPolicy:     jetstream.AckNonePolicy,
}
```
