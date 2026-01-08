# Go SDK Patterns

Common patterns for using JetStream with the Go client.

## Installation

```bash
go get github.com/nats-io/nats.go
go get github.com/nats-io/nats.go/jetstream
```

## Connection

```go
import (
    "github.com/nats-io/nats.go"
    "github.com/nats-io/nats.go/jetstream"
)

// Basic connection
nc, err := nats.Connect(nats.DefaultURL)
if err != nil {
    log.Fatal(err)
}
defer nc.Close()

// With options
nc, err := nats.Connect(
    "nats://localhost:4222",
    nats.Name("my-service"),
    nats.ReconnectWait(2*time.Second),
    nats.MaxReconnects(-1), // Infinite reconnects
    nats.DisconnectErrHandler(func(nc *nats.Conn, err error) {
        log.Printf("Disconnected: %v", err)
    }),
    nats.ReconnectHandler(func(nc *nats.Conn) {
        log.Printf("Reconnected to %s", nc.ConnectedUrl())
    }),
)

// Get JetStream context
js, err := jetstream.New(nc)
if err != nil {
    log.Fatal(err)
}
```

## Stream Management

```go
ctx := context.Background()

// Create or update stream (idempotent)
stream, err := js.CreateOrUpdateStream(ctx, jetstream.StreamConfig{
    Name:     "EVENTS",
    Subjects: []string{"events.>"},
    Storage:  jetstream.FileStorage,
    MaxAge:   7 * 24 * time.Hour,
})

// Get existing stream
stream, err := js.Stream(ctx, "EVENTS")

// Stream info
info, _ := stream.Info(ctx)
fmt.Printf("Messages: %d, Bytes: %d\n", info.State.Msgs, info.State.Bytes)

// Delete stream
err = js.DeleteStream(ctx, "EVENTS")
```

## Publishing

### Synchronous Publish
```go
// Simple publish
ack, err := js.Publish(ctx, "events.user.created", []byte(`{"id": 1}`))
if err != nil {
    log.Printf("Publish failed: %v", err)
}
fmt.Printf("Sequence: %d\n", ack.Sequence)

// With message ID (deduplication)
ack, err := js.Publish(ctx, "events.user.created", data,
    jetstream.WithMsgID("user-1-created"))

if ack.Duplicate {
    log.Println("Message was a duplicate")
}

// With expected sequence (optimistic concurrency)
ack, err := js.Publish(ctx, "events.order.123", data,
    jetstream.WithExpectLastSubjectSequence(5))
```

### Asynchronous Publish
```go
// Async publish for high throughput
futures := make([]jetstream.PubAckFuture, 0, 1000)

for i := 0; i < 1000; i++ {
    future, err := js.PublishAsync("events.batch", data)
    if err != nil {
        log.Printf("PublishAsync failed: %v", err)
        continue
    }
    futures = append(futures, future)
}

// Wait for all acks
select {
case <-js.PublishAsyncComplete():
    log.Println("All messages acknowledged")
case <-time.After(5 * time.Second):
    log.Println("Timeout waiting for acks")
}

// Or check individual futures
for _, f := range futures {
    select {
    case ack := <-f.Ok():
        fmt.Printf("Acked: %d\n", ack.Sequence)
    case err := <-f.Err():
        log.Printf("Nacked: %v", err)
    }
}
```

## Consumer Creation

```go
// Durable pull consumer
cons, err := stream.CreateOrUpdateConsumer(ctx, jetstream.ConsumerConfig{
    Durable:       "processor",
    AckPolicy:     jetstream.AckExplicitPolicy,
    DeliverPolicy: jetstream.DeliverAllPolicy,
    FilterSubject: "events.>",
    MaxAckPending: 1000,
    AckWait:       30 * time.Second,
})

// Ephemeral consumer
cons, err := stream.CreateOrUpdateConsumer(ctx, jetstream.ConsumerConfig{
    AckPolicy:         jetstream.AckExplicitPolicy,
    InactiveThreshold: 5 * time.Minute,
})

// Ordered consumer (simplest)
cons, err := stream.OrderedConsumer(ctx, jetstream.OrderedConsumerConfig{
    FilterSubjects: []string{"events.>"},
})

// Get existing consumer
cons, err := stream.Consumer(ctx, "processor")

// Delete consumer
err = stream.DeleteConsumer(ctx, "processor")
```

## Consuming Messages

### Fetch (Batch)
```go
// Fetch up to 10 messages, wait up to 5 seconds
msgs, err := cons.Fetch(10, jetstream.FetchMaxWait(5*time.Second))
if err != nil {
    log.Printf("Fetch error: %v", err)
}

for msg := range msgs.Messages() {
    fmt.Printf("Received: %s\n", msg.Subject())

    // Process message
    if err := processMessage(msg); err != nil {
        msg.Nak() // Retry
        continue
    }

    msg.Ack()
}

// Check for fetch errors
if err := msgs.Error(); err != nil {
    log.Printf("Fetch completed with error: %v", err)
}
```

### FetchNoWait (Non-blocking)
```go
// Get available messages immediately, don't wait
msgs, err := cons.FetchNoWait(100)
for msg := range msgs.Messages() {
    // Process
    msg.Ack()
}
```

### Messages (Iterator)
```go
// Continuous iteration
iter, err := cons.Messages()
if err != nil {
    log.Fatal(err)
}
defer iter.Stop()

for {
    msg, err := iter.Next()
    if err != nil {
        if errors.Is(err, jetstream.ErrMsgIteratorClosed) {
            break
        }
        log.Printf("Iterator error: %v", err)
        continue
    }

    // Process
    msg.Ack()
}
```

### Consume (Callback)
```go
// Callback-based consumption
consumeCtx, err := cons.Consume(func(msg jetstream.Msg) {
    fmt.Printf("Received: %s\n", msg.Subject())

    // Process
    if err := processMessage(msg); err != nil {
        msg.Nak()
        return
    }

    msg.Ack()
})
if err != nil {
    log.Fatal(err)
}
defer consumeCtx.Stop()

// Block until shutdown
<-ctx.Done()
```

## Message Handling

```go
func processMessage(msg jetstream.Msg) error {
    // Get metadata
    meta, err := msg.Metadata()
    if err != nil {
        return err
    }

    fmt.Printf("Stream seq: %d, Consumer seq: %d, Delivered: %d\n",
        meta.Sequence.Stream,
        meta.Sequence.Consumer,
        meta.NumDelivered)

    // Get headers
    msgID := msg.Headers().Get("Nats-Msg-Id")

    // Get data
    data := msg.Data()
    subject := msg.Subject()

    // Process...

    return nil
}
```

## Acknowledgment Patterns

```go
// Simple ack
msg.Ack()

// Double ack (wait for confirmation)
if err := msg.DoubleAck(ctx); err != nil {
    log.Printf("Ack not confirmed: %v", err)
}

// Negative ack (redeliver now)
msg.Nak()

// Negative ack with delay
msg.NakWithDelay(30 * time.Second)

// In progress (extend ack deadline)
msg.InProgress()

// Terminate (stop redelivery)
msg.Term()
msg.TermWithReason("invalid message format")
```

## Error Handling

```go
// Check for specific errors
if errors.Is(err, jetstream.ErrStreamNotFound) {
    // Stream doesn't exist
}

if errors.Is(err, jetstream.ErrConsumerNotFound) {
    // Consumer doesn't exist
}

if errors.Is(err, jetstream.ErrNoMessages) {
    // No messages available
}

if errors.Is(err, nats.ErrTimeout) {
    // Operation timed out
}
```

## Complete Worker Example

```go
package main

import (
    "context"
    "encoding/json"
    "log"
    "os"
    "os/signal"
    "syscall"

    "github.com/nats-io/nats.go"
    "github.com/nats-io/nats.go/jetstream"
)

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    // Connect
    nc, err := nats.Connect(nats.DefaultURL)
    if err != nil {
        log.Fatal(err)
    }
    defer nc.Close()

    js, err := jetstream.New(nc)
    if err != nil {
        log.Fatal(err)
    }

    // Ensure stream exists
    stream, err := js.CreateOrUpdateStream(ctx, jetstream.StreamConfig{
        Name:     "TASKS",
        Subjects: []string{"tasks.>"},
    })
    if err != nil {
        log.Fatal(err)
    }

    // Create consumer
    cons, err := stream.CreateOrUpdateConsumer(ctx, jetstream.ConsumerConfig{
        Durable:   "worker",
        AckPolicy: jetstream.AckExplicitPolicy,
    })
    if err != nil {
        log.Fatal(err)
    }

    // Consume messages
    consumeCtx, err := cons.Consume(func(msg jetstream.Msg) {
        meta, _ := msg.Metadata()
        log.Printf("Processing message %d (attempt %d)",
            meta.Sequence.Stream, meta.NumDelivered)

        if err := processTask(msg.Data()); err != nil {
            log.Printf("Error: %v", err)
            msg.Nak()
            return
        }

        msg.Ack()
    })
    if err != nil {
        log.Fatal(err)
    }
    defer consumeCtx.Stop()

    // Wait for shutdown signal
    sig := make(chan os.Signal, 1)
    signal.Notify(sig, syscall.SIGINT, syscall.SIGTERM)
    <-sig

    log.Println("Shutting down...")
}

func processTask(data []byte) error {
    var task map[string]any
    return json.Unmarshal(data, &task)
}
```

## Key-Value Store

```go
// Create or get bucket
kv, err := js.CreateKeyValue(ctx, jetstream.KeyValueConfig{
    Bucket: "CONFIG",
})

// Put value
_, err = kv.Put(ctx, "app.setting", []byte("value"))

// Get value
entry, err := kv.Get(ctx, "app.setting")
fmt.Printf("Value: %s, Revision: %d\n", entry.Value(), entry.Revision())

// Watch for changes
watcher, err := kv.Watch(ctx, "app.>")
for entry := range watcher.Updates() {
    if entry == nil {
        continue // Initial nil indicates caught up
    }
    fmt.Printf("Changed: %s = %s\n", entry.Key(), entry.Value())
}

// Delete key
err = kv.Delete(ctx, "app.setting")
```
