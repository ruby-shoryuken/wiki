# Architecture Internals

This guide explains Shoryuken's internal architecture for contributors and advanced users.

## Overview

```
┌─────────────────────────────────────────────────────────┐
│                        Runner                           │
│  (Entry point, signal handling, Rails initialization)   │
└──────────────────────────┬──────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│                       Launcher                          │
│  (Lifecycle management, health checks, managers)        │
└──────────────────────────┬──────────────────────────────┘
                           │
              ┌────────────┴────────────┐
              ▼                         ▼
┌─────────────────────────┐  ┌─────────────────────────┐
│   Manager (Group 1)     │  │   Manager (Group 2)     │
│  ┌─────────────────┐    │  │  ┌─────────────────┐    │
│  │    Fetcher      │    │  │  │    Fetcher      │    │
│  └────────┬────────┘    │  │  └────────┬────────┘    │
│           │             │  │           │             │
│  ┌────────▼────────┐    │  │  ┌────────▼────────┐    │
│  │ Polling Strategy│    │  │  │ Polling Strategy│    │
│  └────────┬────────┘    │  │  └────────┬────────┘    │
│           │             │  │           │             │
│  ┌────────▼────────┐    │  │  ┌────────▼────────┐    │
│  │   Processor(s)  │    │  │  │   Processor(s)  │    │
│  └─────────────────┘    │  │  └─────────────────┘    │
└─────────────────────────┘  └─────────────────────────┘
```

---

## Core Components

### Runner

Entry point for the Shoryuken process.

**Location:** `lib/shoryuken/runner.rb`

**Responsibilities:**
- Parse CLI options
- Load Rails/application
- Initialize Launcher
- Handle Unix signals (TERM, INT, USR1, TTIN, TSTP)

```ruby
# Signal handling
Signal.trap('TERM') { launcher.stop! }
Signal.trap('USR1') { launcher.stop }
Signal.trap('TTIN') { print_debug_info }
```

### Launcher

Coordinates the lifecycle of all processing managers.

**Location:** `lib/shoryuken/launcher.rb`

**Responsibilities:**
- Create managers for each processing group
- Start/stop managers
- Health check coordination
- Fire lifecycle events

**Key Methods:**
```ruby
def start         # Starts all managers
def stop          # Graceful shutdown (waits for jobs)
def stop!         # Hard shutdown (with timeout)
def stopping?     # Check if shutting down
def healthy?      # All managers running?
```

### Manager

Manages message dispatching for a single processing group.

**Location:** `lib/shoryuken/manager.rb`

**Responsibilities:**
- Dispatch loop (fetch → assign → process)
- Track busy/ready processor count
- Coordinate with polling strategy
- Fire utilization events

**Key Attributes:**
```ruby
@group              # Processing group name
@fetcher            # Fetcher instance
@polling_strategy   # Queue selection strategy
@max_processors     # Concurrency limit
@busy_processors    # AtomicCounter
```

**Dispatch Flow:**
```
dispatch_loop
    │
    ├── Check: stop requested? not running? → exit
    │
    ├── Check: ready processors > 0?
    │   └── No → sleep(0.1)
    │
    ├── Get next queue from polling strategy
    │   └── None available → sleep(0.1)
    │
    ├── Fetch messages from queue
    │
    └── Assign to processor(s)
        │
        └── Loop back
```

### Fetcher

Retrieves messages from SQS.

**Location:** `lib/shoryuken/fetcher.rb`

**Responsibilities:**
- Call SQS receive_message API
- Handle long polling
- Apply message limits

### Processor

Executes worker logic on messages.

**Location:** `lib/shoryuken/processor.rb`

**Responsibilities:**
- Invoke middleware chain
- Instantiate worker
- Call `worker.perform(sqs_msg, body)`
- Handle exceptions

---

## Polling Strategies

### BaseStrategy

**Location:** `lib/shoryuken/polling/base_strategy.rb`

Abstract base for queue selection.

### WeightedRoundRobin (Default)

**Location:** `lib/shoryuken/polling/weighted_round_robin.rb`

Selects queues based on configured weights.

```ruby
queues:
  - [critical, 3]  # Selected 3× more often
  - [default, 1]
```

### StrictPriority

**Location:** `lib/shoryuken/polling/strict_priority.rb`

Always processes higher-priority queues first.

```ruby
polling: :strict_priority
queues:
  - critical  # Always first
  - default   # Only when critical is empty
```

---

## Thread Safety

### Atomic Helpers

Shoryuken v7 uses custom thread-safe primitives (no concurrent-ruby dependency).

**Location:** `lib/shoryuken/helpers/`

```ruby
# AtomicCounter - thread-safe integer
counter = Shoryuken::Helpers::AtomicCounter.new(0)
counter.increment
counter.decrement
counter.value

# AtomicBoolean - thread-safe boolean
flag = Shoryuken::Helpers::AtomicBoolean.new(false)
flag.make_true
flag.make_false
flag.true?

# AtomicHash - thread-safe hash
hash = Shoryuken::Helpers::AtomicHash.new
hash[:key] = value
```

### Executor

Uses `Concurrent::Promise` for async processing:

```ruby
Concurrent::Promise
  .execute(executor: @executor) { Processor.process(queue, msg) }
  .then { processor_done(queue) }
  .rescue { processor_done(queue) }
```

---

## Middleware Chain

**Location:** `lib/shoryuken/middleware/chain.rb`

```ruby
chain.entries  # Array of middleware classes
chain.add(Middleware)
chain.insert_before(Existing, New)
chain.insert_after(Existing, New)
chain.remove(Middleware)
chain.clear
```

**Execution:**
```ruby
def invoke(worker, queue, sqs_msg, body)
  chain = @entries.dup
  traverse_chain = lambda do
    if chain.empty?
      yield  # Worker.perform
    else
      middleware = chain.shift
      middleware.new.call(worker, queue, sqs_msg, body, &traverse_chain)
    end
  end
  traverse_chain.call
end
```

---

## Lifecycle Events

**Location:** `lib/shoryuken/options.rb`

Events fired during operation:

| Event | When | Parameters |
|-------|------|------------|
| `startup` | Process starts | None |
| `dispatch` | Before fetching | `queue_name` |
| `utilization_update` | Processor assigned/freed | `group`, `max_processors`, `busy_processors` |
| `quiet` | Stop requested | None |
| `shutdown` | Shutting down | None |
| `stopped` | Fully stopped | None |

**Registration:**
```ruby
Shoryuken.configure_server do |config|
  config.on(:startup) { puts "Starting" }
  config.on(:utilization_update) do |group, utilization|
    # utilization = busy / max (0.0 - 1.0)
  end
end
```

---

## Worker Registry

**Location:** `lib/shoryuken/worker_registry.rb`

Maps queues to worker classes.

```ruby
Shoryuken.worker_registry.register('my_queue', MyWorker)
Shoryuken.worker_registry.fetch('my_queue')  # => MyWorker
Shoryuken.worker_registry.workers           # => { 'my_queue' => MyWorker }
```

---

## ActiveJob Integration

### Adapter

**Location:** `lib/active_job/queue_adapters/shoryuken_adapter.rb`

```ruby
def enqueue(job, options = {})
  # Build SQS message
  # Send to queue
end

def enqueue_at(job, timestamp)
  # Calculate delay_seconds
  # Call enqueue with delay
end

def enqueue_all(jobs)
  # Batch enqueue (Rails 7.1+)
end

def stopping?
  # Check launcher state
end
```

### JobWrapper

**Location:** `lib/shoryuken/active_job/job_wrapper.rb`

Worker that wraps ActiveJob execution:

```ruby
class JobWrapper
  include Shoryuken::Worker

  def perform(sqs_msg, body)
    ActiveJob::Base.execute(body)
  end
end
```

### CurrentAttributes

**Location:** `lib/shoryuken/active_job/current_attributes.rb`

Persists Rails CurrentAttributes across job boundaries:

```ruby
module Persistence
  # Prepended to adapter
  def enqueue(job, options = {})
    # Serialize current attributes into job
    super
  end
end

module Loading
  # Prepended to JobWrapper
  def perform(sqs_msg, body)
    # Restore current attributes
    super
  ensure
    # Reset current attributes
  end
end
```

---

## Directory Structure

```
lib/shoryuken/
├── active_job/
│   ├── current_attributes.rb
│   └── job_wrapper.rb
├── helpers/
│   ├── atomic_boolean.rb
│   ├── atomic_counter.rb
│   ├── atomic_hash.rb
│   ├── hash_utils.rb
│   └── string_utils.rb
├── middleware/
│   ├── chain.rb
│   └── server/
│       ├── active_record.rb
│       ├── auto_delete.rb
│       ├── auto_extend_visibility.rb
│       ├── exponential_backoff_retry.rb
│       └── timing.rb
├── polling/
│   ├── base_strategy.rb
│   ├── strict_priority.rb
│   └── weighted_round_robin.rb
├── client.rb
├── extensions/
├── fetcher.rb
├── launcher.rb
├── logging.rb
├── manager.rb
├── message/
├── options.rb
├── processor.rb
├── queue.rb
├── runner.rb
├── util.rb
├── worker.rb
└── worker_registry.rb
```

---

## Zeitwerk Autoloading

Shoryuken v7 uses Zeitwerk for autoloading:

```ruby
# lib/shoryuken.rb
loader = Zeitwerk::Loader.for_gem
loader.setup
```

File/class naming follows Rails conventions:
- `lib/shoryuken/manager.rb` → `Shoryuken::Manager`
- `lib/shoryuken/helpers/atomic_counter.rb` → `Shoryuken::Helpers::AtomicCounter`

---

## Extension Points

### Custom Polling Strategy

```ruby
class MyStrategy < Shoryuken::Polling::BaseStrategy
  def initialize(queues, delay)
    @queues = queues
    @delay = delay
  end

  def next_queue
    # Return queue to poll or nil
  end

  def messages_found(queue, count)
    # Called after fetch
  end

  def active_queues
    # Return currently active queues
  end
end

# Register
Shoryuken.polling_strategy = MyStrategy
```

### Custom Executor

```ruby
Shoryuken.launcher_executor = Concurrent::ThreadPoolExecutor.new(
  min_threads: 5,
  max_threads: 50,
  max_queue: 100
)
```

---

## Related

- [[Performance Tuning]] - Optimization
- [[Middleware]] - Middleware system
- [[Lifecycle Events]] - Event hooks
- [[Processing Groups]] - Group configuration
