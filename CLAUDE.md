# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Asynq is a Go library for distributed task queue processing backed by Redis. It provides guaranteed at-least-once execution, task scheduling, retries, and horizontal scaling.

## Development Commands

### Testing

```bash
# Run all tests for core module
go test -race -v ./...

# Run tests with coverage
go test -race -v -coverprofile=coverage.txt -covermode=atomic ./...

# Run tests for x module (extensions)
cd x && go test -race -v ./... && cd ..

# Run tests for tools module (CLI)
cd tools && go test -race -v ./... && cd ..

# Run benchmarks
go test -run=^$ -bench=. -loglevel=debug ./...

# Run single test
go test -v -run TestName ./...

# Run tests with Redis cluster (requires cluster setup)
go test -race -v -redis_cluster ./...
```

### Building

```bash
# Build core module
go build -v ./...

# Build x module
cd x && go build -v ./... && cd ..

# Build tools module (CLI)
cd tools && go build -v ./... && cd ..

# Install CLI tool
go install github.com/JK-97/asynq/tools/asynq@latest
```

### Linting

```bash
# Run linter
make lint
# OR
golangci-lint run
```

### Protocol Buffers

```bash
# Regenerate protobuf files (if you modify internal/proto/asynq.proto)
make proto
```

## Architecture

### Core Components

Asynq has three main public APIs:

1. **Client** ([client.go](client.go)) - Enqueues tasks to Redis for background processing
2. **Server** ([server.go](server.go)) - Pulls tasks from queues and processes them with worker goroutines
3. **Inspector** ([inspector.go](inspector.go)) - Provides monitoring and management of queues/tasks

Additional components:
- **Scheduler** ([scheduler.go](scheduler.go)) - Manages periodic task execution (cron-like)
- **ServeMux** ([servemux.go](servemux.go)) - Routes task types to handlers (similar to http.ServeMux)

### Internal Architecture

#### Key Internal Packages

- **[internal/base](internal/base/)** - Foundational types, Redis key naming, `TaskMessage`, `TaskState`, `Broker` interface
- **[internal/rdb](internal/rdb/)** - Redis operations via Lua scripts, implements `Broker` interface
- **[internal/proto](internal/proto/)** - Protocol Buffer definitions for serialization
- **[internal/context](internal/context/)** - Task-specific context values (retry count, task ID, etc.)
- **[internal/log](internal/log/)** - Logging abstraction

#### Background Processes (Server Components)

The Server coordinates multiple background goroutines:

- **Processor** ([processor.go](processor.go)) - Core worker pool manager, pulls tasks from queues respecting priorities, enforces concurrency limits via semaphore
- **Heartbeater** ([heartbeat.go](heartbeat.go)) - Maintains liveness signals to Redis (~1s interval), extends task leases, tracks worker info
- **Recoverer** ([recoverer.go](recoverer.go)) - Detects tasks with expired leases (crashed workers), moves them to retry/archive queues
- **Scheduler** ([scheduler.go](scheduler.go)) - Moves scheduled tasks to pending queue when their time comes
- **Janitor** ([janitor.go](janitor.go)) - Deletes expired completed tasks to prevent Redis memory bloat
- **Aggregator** ([aggregator.go](aggregator.go)) - Batches grouped tasks when grace period/max size/max delay conditions are met
- **Syncer** ([syncer.go](syncer.go)) - Retries failed Redis operations for eventual consistency
- **Subscriber** ([subscriber.go](subscriber.go)) - Listens for task cancellation signals via Redis Pub/Sub
- **Forwarder** ([forwarder.go](forwarder.go)) - Moves scheduled/retry tasks to pending queue when ready
- **HealthChecker** ([healthcheck.go](healthcheck.go)) - Periodic Redis connection health checks

### Task Lifecycle

1. **Enqueue**: Client pushes task to pending queue (or scheduled/aggregating queue)
2. **Pull**: Processor pulls from pending queue respecting priority (weighted or strict)
3. **Active**: Task moved to active state with 30-second lease
4. **Process**: Worker goroutine executes handler with context (deadline/timeout)
5. **Complete**:
   - Success: moved to completed (if retention configured) or deleted
   - Failure: moved to retry queue with exponential backoff
   - Max retries exhausted: moved to archived queue
6. **Recovery**: Recoverer detects expired leases and recovers orphaned tasks

### Task States

Tasks transition through these states (defined in [internal/base/base.go](internal/base/base.go:44-54)):

- `pending` - Ready to process
- `active` - Currently being processed (has lease)
- `scheduled` - Scheduled for future processing
- `retry` - Failed, waiting for retry with backoff
- `archived` - Exhausted all retries (dead letter queue)
- `completed` - Successfully processed (retention period applies)
- `aggregating` - Waiting in group to be batched

### Queue Priority System

Two priority modes (configured in `Config.Queues` map):

- **Weighted Priority** (default): Tasks sampled from higher priority queues more frequently (proportional to weight)
- **Strict Priority**: Lower priority queues only processed when higher priority queues are empty

### Lease-Based Safety

Critical for automatic crash recovery:
- Each active task has a 30-second lease ([internal/rdb/rdb.go](internal/rdb/rdb.go))
- Heartbeater extends leases for active tasks every ~1s
- Recoverer checks for expired leases and recovers orphaned tasks
- Provides automatic recovery without manual intervention

### Unique Tasks

- Uniqueness based on combination of queue name, task type, and payload
- TTL-based uniqueness window
- Returns `ErrDuplicateTask` if duplicate detected during TTL window

### Group Aggregation

- Tasks with same group name accumulate in aggregation sets
- Aggregated when: grace period expires OR max size reached OR max delay reached
- User provides aggregation logic via `GroupAggregator` interface
- Aggregator process checks groups periodically (limited to 3 concurrent checks)

## Testing Patterns

### Test Setup

Tests use a centralized setup function that:
- Configures Redis connection (standalone or cluster based on flags)
- Supports flags: `-redis_addr`, `-redis_db`, `-redis_cluster`, `-redis_cluster_addrs`
- Clears Redis with `FlushDB()` between tests

### Test Utilities

Located in [internal/testutil](internal/testutil/):
- Sorting transformers for slice comparison: `SortMsgOpt`, `SortZSetEntryOpt`
- `EquateInt64Approx()` for fuzzy timestamp comparison
- Uses `google/go-cmp` for deep equality

### Redis Cluster Testing

Per [CONTRIBUTING.md](CONTRIBUTING.md:48), run tests with `--redis_cluster` flag to ensure Redis cluster compatibility. Some Lua scripts may not be fully compatible with Redis cluster.

## Code Conventions

### Interface-Based Design

- `Handler` interface: `ProcessTask(context.Context, *Task) error`
- `HandlerFunc` adapter for function handlers
- `Broker` interface abstracts all Redis operations ([internal/base/base.go](internal/base/base.go))

### Task Options Pattern

- Options passed at task creation or enqueue time (enqueue-time overrides)
- Options implement common interface: `String()`, `Type()`, `Value()`
- Defaults: `MaxRetry=25`, `Timeout=30min`, `Queue="default"`

### Middleware Support

- `ServeMux` supports middleware via `MiddlewareFunc`
- Middleware chaining for cross-cutting concerns (logging, metrics, auth)
- Longest pattern wins for task type routing (like `net/http`)

### Context Usage

- Handlers receive context with deadline/timeout from task options
- Context values provide metadata: `context.GetRetryCount()`, `context.GetTaskID()`, etc.
- Context cancellation via Redis Pub/Sub

### Error Handling

- `IsFailure` predicate distinguishes failures from expected errors
- Wrap errors with `asynq.SkipRetry` to prevent retry
- `IsPanicError()` detects panics in handlers

### Lua Scripts for Atomicity

- Critical Redis operations use Lua scripts to prevent race conditions
- Examples: enqueue, dequeue, schedule, retry operations
- Located in [internal/rdb](internal/rdb/)

### Clock Abstraction

- `timeutil.Clock` interface for testability
- Tests can use simulated time
- Production uses real clock

## Project Structure

```
.
├── *.go                    # Public API (Client, Server, Inspector, etc.)
├── *_test.go              # Tests for public API
├── internal/              # Internal packages (not importable)
│   ├── base/             # Foundational types, Broker interface
│   ├── rdb/              # Redis operations and Lua scripts
│   ├── proto/            # Protocol Buffer definitions
│   ├── context/          # Task context utilities
│   ├── log/              # Logging abstraction
│   ├── errors/           # Error types
│   ├── testutil/         # Test helpers
│   ├── testbroker/       # Mock broker for tests
│   └── timeutil/         # Clock abstraction
├── tools/                # CLI tool (asynq command)
│   └── asynq/           # CLI implementation
├── x/                    # Extension modules
└── docs/                 # Documentation
```

## CLI Tool

The `asynq` CLI tool provides queue/task inspection and management:

```bash
# Install
go install github.com/JK-97/asynq/tools/asynq@latest

# View dashboard
asynq dash

# View statistics
asynq stats

# Queue operations
asynq queue ls
asynq queue inspect <queue>
asynq queue pause <queue>
asynq queue unpause <queue>

# Task operations
asynq task ls <queue> <state>
asynq task cancel <task_id>
asynq task archive <task_id>
asynq task run <task_id>

# Connection flags
asynq dash -u localhost:6379 -n 0 -p <password>
asynq dash --cluster --cluster_addrs "host1:6379,host2:6379"
```

Configuration file supported at `$HOME/.asynq.yaml`:

```yaml
uri: 127.0.0.1:6379
db: 2
password: mypassword
```

## Important Notes

### Redis Compatibility

- Requires Redis 4.0 or higher
- Some Lua scripts may not be fully compatible with Redis Cluster
- Supports standalone, sentinel, and cluster deployments via `redis.UniversalClient`

### Version Stability

- Current version is v0.x.x (pre-1.0)
- Public API may change without major version bump before v1.0.0
- Library is in moderate development with less frequent breaking changes

### Graceful Shutdown

- Server stops accepting new tasks
- Waits for in-flight tasks up to shutdown timeout (default 8s)
- Processor drains semaphore to ensure workers complete

### Two Redis Connection Modes

- **Managed**: Asynq creates and closes connection
- **Shared**: Caller provides connection via `NewClientFromRedisClient()`, Asynq doesn't close it

## Related Tools

- **Asynqmon** - Web UI for monitoring and administrating queues (separate repository)
- **Prometheus Integration** - Built-in metrics support via `x/metrics` module
