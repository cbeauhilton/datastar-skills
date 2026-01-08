# NATS Subjects

Subjects are the addressing mechanism in NATS. They're strings that publishers send to and subscribers listen on. JetStream streams capture messages by subject patterns.

## Subject Anatomy

Subjects are dot-separated tokens:
```
orders.us.created
│      │  └── action
│      └── region
└── domain
```

## Wildcards

### Single Token: `*`
Matches exactly one token at that position.

```
orders.*        → matches orders.us, orders.eu
orders.*.created → matches orders.us.created, orders.eu.created
                 → NOT orders.us.west.created
```

### Multi Token: `>`
Matches one or more tokens. Must be last token.

```
orders.>        → matches orders.us, orders.us.created, orders.us.west.created
events.>        → matches anything starting with events.
```

## Stream Subject Configuration

```go
jetstream.StreamConfig{
    Subjects: []string{
        "orders.>",           // All order events
        "payments.*",         // Direct payment events
        "users.*.profile",    // User profile updates
    },
}
```

### Overlapping Subjects
A single message can be captured by multiple subject patterns in one stream, but it's stored only once.

### Multiple Streams
Different streams can capture the same subjects. Use this for:
- Different retention policies
- Regional distribution
- Archival vs. operational stores

## Consumer Filtering

Consumers filter the stream's messages by subject:

```go
// Stream captures orders.>
// Consumer only wants US orders
jetstream.ConsumerConfig{
    FilterSubject: "orders.us.>",
}

// Multiple filters (NATS 2.10+)
jetstream.ConsumerConfig{
    FilterSubjects: []string{
        "orders.us.>",
        "orders.ca.>",
    },
}
```

## Subject Naming Best Practices

### Hierarchical Structure
```
<domain>.<entity>.<action>
<domain>.<region>.<entity>.<action>
```

Examples:
```
orders.created
orders.us.created
orders.us.west.created
users.profile.updated
payments.stripe.completed
```

### Conventions
- Use lowercase
- Use dots for hierarchy
- Use nouns for entities, past tense for events
- Avoid special characters except dots

### For Work Queues
Design subjects for partitioning:
```
tasks.region.priority
tasks.us.high
tasks.eu.low
```

This enables filtered consumers for load distribution.

## Subject Transforms

Streams can transform subjects before storage:

```go
jetstream.StreamConfig{
    SubjectTransform: &jetstream.SubjectTransformConfig{
        Source:      "raw.>",
        Destination: "processed.{{wildcard(1)}}",
    },
}
```

## Republish

Streams can republish to different subjects:

```go
jetstream.StreamConfig{
    RePublish: &jetstream.RePublish{
        Source:      ">",
        Destination: "archive.{{subject}}",
    },
}
```

## Subject Mapping (Server)

NATS servers can map subjects for routing:
- `foo` → `bar`
- `foo.*` → `bar.$1`
- Weighted distribution for load balancing

This is server configuration, not JetStream-specific.

## Common Patterns

### Event Sourcing
```
<aggregate>.<id>.<event>
orders.123.created
orders.123.item_added
orders.123.shipped
```

### CQRS
```
commands.<aggregate>.<command>
events.<aggregate>.<event>
commands.order.create
events.order.created
```

### Multi-Tenant
```
tenant.<tenant_id>.<domain>.>
tenant.acme.orders.created
tenant.globex.orders.created
```

### Regional
```
<region>.<domain>.<action>
us-east.orders.created
eu-west.orders.created
```
