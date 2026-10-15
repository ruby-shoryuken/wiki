# Lifecycle Events

Shoryuken fires lifecycle events at key points during processing. Use these events for initialization, cleanup, monitoring, and custom integrations.

## Available Events

| Event | When Fired | Use Cases |
|-------|------------|-----------|
| `startup` | After queues validated, before fetching | Initialize connections, start health server |
| `dispatch` | Each time messages are fetched from SQS | Metrics, heartbeats |
| `utilization_update` | When worker pool utilization changes | Monitoring, auto-scaling signals |
| `quiet` | On soft shutdown (USR1 signal) | Pre-shutdown cleanup |
| `shutdown` | During shutdown sequence | Close connections, flush buffers |
| `stopped` | After executor fully stopped | Final cleanup |

## Event Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    Lifecycle Flow                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Start                                                       │
│    │                                                         │
│    ▼                                                         │
│  startup ──────────► Processing Loop                         │
│                           │                                  │
│                      ┌────┴────┐                             │
│                      ▼         ▼                             │
│                  dispatch   utilization_update               │
│                      │         │                             │
│                      └────┬────┘                             │
│                           │                                  │
│                      (repeat)                                │
│                           │                                  │
│  SIGTERM/USR1 ───────────►│                                  │
│                           ▼                                  │
│                        quiet                                 │
│                           │                                  │
│                           ▼                                  │
│                       shutdown                               │
│                           │                                  │
│                           ▼                                  │
│                       stopped                                │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Basic Usage

```ruby
# config/initializers/shoryuken.rb
Shoryuken.configure_server do |config|
  config.on(:startup) do
    Rails.logger.info "Shoryuken starting up"
    # Initialize resources
  end

  config.on(:dispatch) do
    # Called every time messages are fetched
  end

  config.on(:utilization_update) do |group, utilization|
    # Called when worker pool utilization changes
    Rails.logger.debug "Group #{group} utilization: #{utilization}%"
  end

  config.on(:quiet) do
    Rails.logger.info "Shoryuken entering quiet mode"
    # Prepare for shutdown
  end

  config.on(:shutdown) do
    Rails.logger.info "Shoryuken shutting down"
    # Close connections, flush buffers
  end

  config.on(:stopped) do
    Rails.logger.info "Shoryuken fully stopped"
    # Final cleanup
  end
end
```

## Event Details

### startup

Fired after Shoryuken validates queues and configuration, just before starting to fetch messages.

**Use cases:**
- Initialize database connection pools
- Start health check HTTP server
- Register with service discovery
- Initialize monitoring/metrics

```ruby
config.on(:startup) do
  # Start health check server
  @health_server = WEBrick::HTTPServer.new(Port: 3001)
  Thread.new { @health_server.start }

  # Warm up connections
  ActiveRecord::Base.connection_pool.checkout
  ActiveRecord::Base.connection_pool.checkin

  # Register with service discovery
  ServiceDiscovery.register(name: 'shoryuken-worker', port: 3001)
end
```

### dispatch

Fired each time Shoryuken attempts to fetch messages from SQS. This happens in a continuous loop during normal operation.

**Use cases:**
- Heartbeat to monitoring systems
- Track dispatch frequency
- Log activity

```ruby
config.on(:dispatch) do
  # Send heartbeat
  Heartbeat.ping('shoryuken-worker')

  # Track dispatch metrics
  StatsD.increment('shoryuken.dispatch')
end
```

**Note:** This event fires frequently. Keep handlers lightweight to avoid impacting performance.

### utilization_update

Fired when the worker pool utilization changes. The callback receives the processing group name and current utilization percentage.

**Use cases:**
- Monitor worker saturation
- Trigger auto-scaling
- Performance dashboards

```ruby
config.on(:utilization_update) do |group, utilization|
  # Send to monitoring
  StatsD.gauge("shoryuken.utilization.#{group}", utilization)

  # Log high utilization
  if utilization > 90
    Rails.logger.warn "High utilization in group #{group}: #{utilization}%"
  end

  # Signal auto-scaler
  if utilization > 80
    AutoScaler.request_scale_up(group: group)
  end
end
```

**Utilization calculation:**
- `0%` = No workers busy
- `100%` = All workers busy
- Value changes as workers start/finish processing messages

### quiet

Fired when Shoryuken receives a soft shutdown signal (USR1) before the shutdown sequence begins.

**Use cases:**
- Stop accepting new work
- Deregister from load balancer
- Notify external systems

```ruby
config.on(:quiet) do
  # Deregister from service discovery
  ServiceDiscovery.deregister('shoryuken-worker')

  # Notify monitoring
  PagerDuty.notify('Shoryuken worker entering quiet mode')
end
```

### shutdown

Fired during the shutdown sequence, before the executor is stopped.

**Use cases:**
- Close external connections
- Flush buffers and caches
- Send final metrics

```ruby
config.on(:shutdown) do
  # Flush logs
  Rails.logger.flush

  # Close Redis connections
  Redis.current.quit

  # Send final metrics
  StatsD.flush

  # Stop health server
  @health_server&.shutdown
end
```

### stopped

Fired after the executor has fully stopped and all workers have completed.

**Use cases:**
- Final cleanup
- Remove lock files
- Exit notification

```ruby
config.on(:stopped) do
  # Remove PID file
  File.delete('tmp/pids/shoryuken.pid') if File.exist?('tmp/pids/shoryuken.pid')

  # Final notification
  Rails.logger.info "Shoryuken worker #{Process.pid} has stopped"
end
```

## Complete Example

```ruby
# config/initializers/shoryuken.rb
require 'webrick'

Shoryuken.configure_server do |config|
  # ── Startup ──
  config.on(:startup) do
    Rails.logger.info "[Shoryuken] Starting worker #{Process.pid}"

    # Start health check server
    @health_server = WEBrick::HTTPServer.new(
      Port: ENV.fetch('HEALTH_PORT', 3001),
      Logger: WEBrick::Log.new("/dev/null")
    )

    @health_server.mount_proc '/health' do |req, res|
      launcher = Shoryuken::Runner.instance.launcher
      res.status = launcher&.healthy? ? 200 : 503
      res.body = launcher&.healthy? ? 'OK' : 'Unhealthy'
    end

    Thread.new { @health_server.start }

    # Initialize metrics
    StatsD.increment('shoryuken.startup')
  end

  # ── Dispatch ──
  config.on(:dispatch) do
    StatsD.increment('shoryuken.dispatch')
  end

  # ── Utilization ──
  config.on(:utilization_update) do |group, utilization|
    StatsD.gauge("shoryuken.utilization.#{group}", utilization)
  end

  # ── Quiet ──
  config.on(:quiet) do
    Rails.logger.info "[Shoryuken] Entering quiet mode"
  end

  # ── Shutdown ──
  config.on(:shutdown) do
    Rails.logger.info "[Shoryuken] Shutting down"
    @health_server&.shutdown
    StatsD.flush
  end

  # ── Stopped ──
  config.on(:stopped) do
    Rails.logger.info "[Shoryuken] Worker #{Process.pid} stopped"
    StatsD.increment('shoryuken.stopped')
  end
end
```

## Multiple Handlers

You can register multiple handlers for the same event:

```ruby
config.on(:startup) do
  initialize_database_pool
end

config.on(:startup) do
  start_health_server
end

config.on(:startup) do
  register_with_service_discovery
end
```

Handlers are called in the order they were registered.

## Event Timing Considerations

| Event | Blocking? | Notes |
|-------|-----------|-------|
| `startup` | Yes | Runs before processing starts. Keep initialization quick. |
| `dispatch` | No | Called frequently. Keep handlers very lightweight. |
| `utilization_update` | No | Called on utilization changes. Keep handlers lightweight. |
| `quiet` | Yes | Can block soft shutdown. Keep quick. |
| `shutdown` | Yes | Blocks shutdown. Has timeout limit. |
| `stopped` | Yes | Called after executor stopped. |

## Related

- [[Signals]] - Process control signals
- [[Health Checks]] - Health check API
- [[Middleware]] - Per-message processing hooks
