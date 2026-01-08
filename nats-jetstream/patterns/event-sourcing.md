# Event Sourcing Pattern

Store all changes as a sequence of events. Rebuild state by replaying events. JetStream provides the append-only log foundation.

## Core Concept

Instead of storing current state:
```
Order: {id: 123, status: "shipped", items: [...]}
```

Store the sequence of events that led to that state:
```
OrderCreated:   {id: 123, items: [...]}
PaymentReceived: {id: 123, amount: 99.00}
OrderShipped:   {id: 123, tracking: "ABC123"}
```

## Stream Configuration

```go
stream, _ := js.CreateOrUpdateStream(ctx, jetstream.StreamConfig{
    Name:     "ORDERS",
    Subjects: []string{"orders.>"},
    Storage:  jetstream.FileStorage,
    Retention: jetstream.LimitsPolicy,
    MaxAge:   0, // Keep forever (or set based on retention needs)

    // Enable subject-level last message for snapshots
    MaxMsgsPerSubject: -1,
})
```

## Subject Design

Structure subjects for aggregate access:

```
<aggregate>.<id>.<event-type>
orders.123.created
orders.123.item_added
orders.123.payment_received
orders.123.shipped
```

This enables:
- Filter by aggregate: `orders.123.>`
- Filter by event type: `orders.*.shipped`
- Global stream: `orders.>`

## Publishing Events

```go
type OrderEvent struct {
    Type      string    `json:"type"`
    OrderID   string    `json:"order_id"`
    Timestamp time.Time `json:"timestamp"`
    Data      any       `json:"data"`
}

func publishEvent(ctx context.Context, orderID string, eventType string, data any) error {
    event := OrderEvent{
        Type:      eventType,
        OrderID:   orderID,
        Timestamp: time.Now(),
        Data:      data,
    }

    eventData, _ := json.Marshal(event)
    subject := fmt.Sprintf("orders.%s.%s", orderID, eventType)

    // Use deterministic message ID for deduplication
    msgID := fmt.Sprintf("%s-%s-%d", orderID, eventType, time.Now().UnixNano())

    _, err := js.Publish(ctx, subject, eventData,
        jetstream.WithMsgID(msgID))
    return err
}
```

## Rebuilding State

### Load Aggregate

```go
type Order struct {
    ID       string
    Status   string
    Items    []Item
    Total    float64
    Shipped  bool
    Tracking string
}

func loadOrder(ctx context.Context, orderID string) (*Order, error) {
    order := &Order{ID: orderID}

    // Create consumer for this order's events
    cons, _ := stream.CreateOrUpdateConsumer(ctx, jetstream.ConsumerConfig{
        FilterSubject: fmt.Sprintf("orders.%s.>", orderID),
        DeliverPolicy: jetstream.DeliverAllPolicy,
        AckPolicy:     jetstream.AckNonePolicy, // Read-only replay
    })
    defer stream.DeleteConsumer(ctx, cons.CachedInfo().Name)

    // Replay all events
    for {
        msgs, err := cons.Fetch(100, jetstream.FetchMaxWait(100*time.Millisecond))
        if err != nil {
            break
        }

        for msg := range msgs.Messages() {
            var event OrderEvent
            json.Unmarshal(msg.Data(), &event)
            applyEvent(order, &event)
        }

        if msgs.Error() != nil {
            break
        }
    }

    return order, nil
}

func applyEvent(order *Order, event *OrderEvent) {
    switch event.Type {
    case "created":
        data := event.Data.(map[string]any)
        order.Status = "pending"
        // Apply creation data
    case "item_added":
        data := event.Data.(map[string]any)
        order.Items = append(order.Items, Item{...})
    case "payment_received":
        order.Status = "paid"
    case "shipped":
        data := event.Data.(map[string]any)
        order.Status = "shipped"
        order.Shipped = true
        order.Tracking = data["tracking"].(string)
    }
}
```

## Snapshots

For aggregates with many events, periodically store snapshots:

```go
type OrderSnapshot struct {
    Order     Order     `json:"order"`
    Version   uint64    `json:"version"` // Last event sequence
    Timestamp time.Time `json:"timestamp"`
}

func saveSnapshot(ctx context.Context, order *Order, lastSeq uint64) error {
    snapshot := OrderSnapshot{
        Order:     *order,
        Version:   lastSeq,
        Timestamp: time.Now(),
    }
    data, _ := json.Marshal(snapshot)

    // Store in KV or separate stream
    _, err := kv.Put(ctx, fmt.Sprintf("snapshots.orders.%s", order.ID), data)
    return err
}

func loadOrderWithSnapshot(ctx context.Context, orderID string) (*Order, error) {
    // Try to load snapshot first
    entry, err := kv.Get(ctx, fmt.Sprintf("snapshots.orders.%s", orderID))
    if err == nil {
        var snapshot OrderSnapshot
        json.Unmarshal(entry.Value(), &snapshot)

        // Replay events after snapshot
        order := &snapshot.Order
        replayEventsAfter(ctx, order, orderID, snapshot.Version)
        return order, nil
    }

    // No snapshot, full replay
    return loadOrder(ctx, orderID)
}
```

## Projections (Read Models)

Build optimized read models from events:

```go
// Projection consumer - processes all order events
func runOrderProjection(ctx context.Context) {
    cons, _ := stream.CreateOrUpdateConsumer(ctx, jetstream.ConsumerConfig{
        Durable:       "order-projection",
        DeliverPolicy: jetstream.DeliverAllPolicy,
    })

    for {
        msgs, _ := cons.Fetch(100)
        for msg := range msgs.Messages() {
            var event OrderEvent
            json.Unmarshal(msg.Data(), &event)

            // Update read model (e.g., SQL table, search index)
            updateProjection(ctx, &event)

            msg.Ack()
        }
    }
}

func updateProjection(ctx context.Context, event *OrderEvent) {
    switch event.Type {
    case "created":
        db.Exec(`INSERT INTO orders_view (id, status, created_at)
                 VALUES (?, 'pending', ?)`, event.OrderID, event.Timestamp)
    case "shipped":
        data := event.Data.(map[string]any)
        db.Exec(`UPDATE orders_view SET status = 'shipped',
                 tracking = ? WHERE id = ?`,
                 data["tracking"], event.OrderID)
    }
}
```

## DCB (Dynamic Consistency Boundary)

Use conditional appends for consistency without global locks:

```go
// Conditional append: only if expected state
func addItemToOrder(ctx context.Context, orderID string, item Item) error {
    // Load current state
    order, lastSeq := loadOrderWithSequence(ctx, orderID)

    // Business rule check
    if order.Status != "pending" {
        return errors.New("cannot add items to non-pending order")
    }

    // Append with expected sequence (optimistic lock)
    event := OrderEvent{Type: "item_added", ...}
    _, err := js.Publish(ctx, fmt.Sprintf("orders.%s.item_added", orderID),
        eventData,
        jetstream.WithExpectLastSubjectSequence(lastSeq))

    if errors.Is(err, jetstream.ErrWrongLastSequence) {
        // Concurrent modification, retry
        return addItemToOrder(ctx, orderID, item)
    }
    return err
}
```

## Best Practices

1. **Immutable events**: Never modify published events
2. **Event versioning**: Include version in event schema for evolution
3. **Idempotent projections**: Handle replays gracefully
4. **Snapshot regularly**: For aggregates with many events
5. **Separate streams per aggregate type**: Easier management
6. **Include metadata**: Timestamp, correlation ID, user ID

## Subject Hierarchy for Event Sourcing

```
# Per-aggregate events
orders.<order-id>.<event-type>
users.<user-id>.<event-type>
products.<product-id>.<event-type>

# Cross-cutting events
events.<domain>.<aggregate>.<event-type>

# With version
orders.v1.<order-id>.<event-type>
```

## Anti-Patterns

1. **Mutable events**: Events should be immutable facts
2. **Too fine-grained events**: Balance granularity
3. **No snapshots for long-lived aggregates**: Performance degrades
4. **Coupling projections to event schema**: Use versioning
5. **Synchronous projections**: Build async, eventually consistent views
