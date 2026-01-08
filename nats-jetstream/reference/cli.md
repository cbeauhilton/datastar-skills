# NATS CLI Reference

Essential `nats` CLI commands for JetStream operations.

## Installation

```bash
# macOS
brew install nats-io/nats-tools/nats

# Linux (download binary)
curl -sf https://binaries.nats.dev/nats-io/natscli/nats@latest | sh

# Go install
go install github.com/nats-io/natscli/nats@latest
```

## Connection

```bash
# Default (localhost:4222)
nats stream ls

# Specific server
nats -s nats://localhost:4222 stream ls

# With credentials
nats -s nats://user:pass@server:4222 stream ls
nats --creds /path/to/user.creds stream ls

# Context (save connection settings)
nats context add local --server localhost:4222
nats context select local
```

## Stream Commands

### Create Stream
```bash
# Interactive
nats stream add

# Non-interactive
nats stream add ORDERS \
    --subjects "orders.>" \
    --storage file \
    --retention limits \
    --max-msgs 1000000 \
    --max-bytes 1GB \
    --max-age 7d \
    --replicas 1
```

### List Streams
```bash
nats stream ls
nats stream ls -j  # JSON output
```

### Stream Info
```bash
nats stream info ORDERS
nats stream info ORDERS -j  # JSON output
```

### Update Stream
```bash
nats stream edit ORDERS --max-age 30d
```

### Delete Stream
```bash
nats stream rm ORDERS -f
```

### Purge Messages
```bash
# Purge all
nats stream purge ORDERS -f

# Purge by subject
nats stream purge ORDERS --subject "orders.old.*" -f

# Keep last N
nats stream purge ORDERS --keep 100 -f
```

### View Messages
```bash
# Get specific message
nats stream get ORDERS 1

# Get last message
nats stream get ORDERS --last

# Get last message for subject
nats stream get ORDERS --last-for "orders.123"
```

### Copy/Backup
```bash
# Copy stream
nats stream copy ORDERS ORDERS_BACKUP

# Backup to file
nats stream backup ORDERS /path/to/backup

# Restore from backup
nats stream restore ORDERS /path/to/backup
```

## Consumer Commands

### Create Consumer
```bash
# Interactive
nats consumer add ORDERS

# Pull consumer
nats consumer add ORDERS processor \
    --pull \
    --ack explicit \
    --deliver all \
    --filter "orders.>" \
    --max-deliver 5 \
    --max-pending 1000

# Push consumer
nats consumer add ORDERS monitor \
    --target deliver.orders \
    --ack none \
    --deliver last
```

### List Consumers
```bash
nats consumer ls ORDERS
```

### Consumer Info
```bash
nats consumer info ORDERS processor
```

### Delete Consumer
```bash
nats consumer rm ORDERS processor -f
```

### Get Next Message (Pull)
```bash
# Single message
nats consumer next ORDERS processor

# Multiple messages
nats consumer next ORDERS processor --count 10
```

### Subscribe (Push/Continuous)
```bash
nats consumer sub ORDERS processor
```

## Publish/Subscribe

### Publish to JetStream
```bash
# Simple publish
nats pub orders.created '{"id": 123}'

# With headers
nats pub orders.created '{"id": 123}' \
    -H "Nats-Msg-Id:order-123-v1"

# From file
nats pub orders.created --file order.json
```

### Request (sync publish with ack)
```bash
nats req orders.created '{"id": 123}'
```

### Subscribe (Core NATS)
```bash
nats sub "orders.>"
nats sub "orders.>" --queue workers
```

## Monitoring

### Server Info
```bash
nats server info
nats server report jetstream
```

### Stream Stats
```bash
nats stream report
nats stream report --subjects  # Per-subject breakdown
```

### Consumer Stats
```bash
nats consumer report ORDERS
```

### Account Info
```bash
nats account info
```

## Key-Value Store

```bash
# Create bucket
nats kv add CONFIG

# Put value
nats kv put CONFIG app.setting "value"

# Get value
nats kv get CONFIG app.setting

# Watch for changes
nats kv watch CONFIG

# List keys
nats kv ls CONFIG

# Delete key
nats kv del CONFIG app.setting

# Delete bucket
nats kv rm CONFIG -f
```

## Object Store

```bash
# Create bucket
nats object add FILES

# Put file
nats object put FILES /path/to/file.pdf

# Get file
nats object get FILES file.pdf

# List objects
nats object ls FILES

# Delete object
nats object del FILES file.pdf
```

## Debugging

### Events
```bash
# JetStream advisories
nats event --js-advisory

# All events
nats event
```

### Latency Test
```bash
nats latency --server-b nats://other:4222
```

### Benchmarks
```bash
# Pub/sub benchmark
nats bench test --pub 1 --sub 1 --msgs 100000

# JetStream benchmark
nats bench test --js --pub 1 --sub 1 --msgs 100000
```

## Common Workflows

### Debug Message Flow
```bash
# 1. Check stream exists and has messages
nats stream info ORDERS

# 2. Check consumer state
nats consumer info ORDERS processor

# 3. View recent messages
nats stream get ORDERS --last

# 4. Try consuming
nats consumer next ORDERS processor
```

### Reset Consumer
```bash
# Delete and recreate consumer
nats consumer rm ORDERS processor -f
nats consumer add ORDERS processor --pull --ack explicit --deliver all
```

### Drain Stream
```bash
# Stop new messages, let consumers drain
# Then purge or delete
nats stream purge ORDERS -f
```

## Aliases

Common aliases for faster CLI usage:

```bash
alias ns='nats stream'
alias nc='nats consumer'
alias np='nats pub'
alias nsub='nats sub'
```

```bash
# Now use:
ns ls
nc ls ORDERS
np orders.test '{"msg": "hello"}'
```
