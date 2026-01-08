# NATS JetStream Skill Plan

Create a Claude Code skill for NATS JetStream, following the same structure as the Datastar skill.

## Target Repo
`cbeauhilton/nats-jetstream-skills` (new repo, or add to existing marketplace)

## Documentation Sources

### Primary
- https://docs.nats.io/nats-concepts/jetstream
- https://docs.nats.io/using-nats/developer
- https://natsbyexample.com (interactive examples)

### Key Pages to Fetch
1. **Core Concepts**
   - Streams (storage, retention, limits)
   - Consumers (push vs pull, durable vs ephemeral)
   - Subjects and wildcards
   - Message acknowledgment patterns

2. **Operations**
   - Stream management (create, update, delete, purge)
   - Consumer management
   - Message publishing
   - Message consumption patterns

3. **Patterns**
   - Work queues
   - Fan-out/Fan-in
   - Request-reply
   - Event sourcing / DCB patterns
   - Exactly-once delivery

4. **Configuration**
   - Stream config options (replicas, storage, retention policies)
   - Consumer config options (ack policies, replay policies, filters)
   - Cluster configuration

5. **CLI Reference**
   - `nats stream` commands
   - `nats consumer` commands
   - `nats pub/sub/req` commands

## Skill Structure

```
nats-jetstream-skills/
├── .claude-plugin/marketplace.json
└── nats-jetstream/
    ├── .claude-plugin/plugin.json
    ├── skills/
    │   └── nats-jetstream.md       # Main skill (overview, quick start)
    ├── concepts/
    │   ├── streams.md              # Stream concepts and config
    │   ├── consumers.md            # Consumer types and patterns
    │   ├── subjects.md             # Subject hierarchy and wildcards
    │   └── acknowledgment.md       # Ack patterns, redelivery
    ├── patterns/
    │   ├── work-queues.md          # Competing consumers
    │   ├── fan-out.md              # Pub/sub patterns
    │   ├── exactly-once.md         # Deduplication, idempotency
    │   └── event-sourcing.md       # DCB-style patterns
    ├── reference/
    │   ├── stream-config.md        # All stream options
    │   ├── consumer-config.md      # All consumer options
    │   └── cli.md                  # nats CLI reference
    └── sdks/
        └── go.md                   # Go SDK patterns (primary)
```

## Main Skill Content Outline

### nats-jetstream.md
- What is JetStream (persistence layer for NATS)
- When to use it (durability, replay, exactly-once)
- Core mental model: Streams store, Consumers read
- Quick start examples (Go)
- Common gotchas
- Links to deeper docs

### Key Concepts to Emphasize
1. **Streams are append-only logs** - messages stored by subject
2. **Consumers are cursors** - track position, can replay
3. **Ack is critical** - understand AckExplicit vs AckAll vs AckNone
4. **Subject filtering** - consumers can filter stream subjects
5. **Pull vs Push** - pull for control, push for simplicity
6. **Durable vs Ephemeral** - durable survives disconnect

### Patterns to Document
1. **Work Queue**: Multiple consumers, each message processed once
2. **Fan-out**: Multiple consumers, each gets all messages
3. **Replay**: Consumer starts from beginning or specific sequence
4. **Exactly-once**: Message deduplication with Nats-Msg-Id header
5. **Request-Reply over JetStream**: For reliable RPC

## Implementation Steps

1. [ ] Fetch docs from docs.nats.io (concepts, config reference)
2. [ ] Fetch examples from natsbyexample.com
3. [ ] Create skill directory structure
4. [ ] Write main skill file (nats-jetstream.md)
5. [ ] Write concept files (streams, consumers, subjects, ack)
6. [ ] Write pattern files (work-queues, fan-out, exactly-once)
7. [ ] Write reference files (stream-config, consumer-config, cli)
8. [ ] Write Go SDK patterns file
9. [ ] Create marketplace.json and plugin.json
10. [ ] Push to GitHub
11. [ ] Install and test

## Notes

- Focus on Go SDK since that's the primary use case
- Include DCB patterns since user has nats-dcb project
- Emphasize operational concerns (monitoring, limits, cleanup)
- Include nats CLI for debugging/exploration

## Allowed Domains (already configured)
- `WebFetch(domain:docs.nats.io)` ✓
- `WebFetch(domain:natsbyexample.com)` ✓
