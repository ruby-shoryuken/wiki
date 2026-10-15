# Signals

Shoryuken responds to Unix signals for process control. Understanding these signals is essential for graceful deployments and debugging.

## Signal Summary

| Signal | Effect | Use Case |
|--------|--------|----------|
| `TTIN` | Print debug info | Debugging, monitoring |
| `TSTP` | Quiet mode (stop fetching) | Pre-shutdown, deployments |
| `USR1` | Graceful shutdown | Deployments, scaling down |
| `TERM` | Hard shutdown with timeout | Container termination |
| `INT` | Same as TERM | Ctrl+C in terminal |

---

## TTIN - Debug Information

Prints thread backtraces, processor stats, and queue weights to the log.

```bash
kill -TTIN $(cat tmp/pids/shoryuken.pid)
```

### Output Includes

- Backtraces for all threads
- Number of busy/ready processors
- Current queue weights
- Processing group status

### Use Cases

- Debugging hung workers
- Identifying slow jobs
- Monitoring thread utilization

---

## TSTP - Quiet Mode

Stops fetching new messages but continues processing in-flight messages. The process remains running.

```bash
kill -TSTP $(cat tmp/pids/shoryuken.pid)
```

### Behavior

1. Stops sending requests to SQS
2. Processes all messages already fetched
3. Messages remain visible (not lost)
4. Process stays alive

### Use Cases

- Pre-deployment pause
- Manual traffic drain
- Debugging without termination

### Notes

- Does **not** exit the process
- Requires `TERM` or `USR1` to fully stop
- Fires the `quiet` lifecycle event

---

## USR1 - Graceful Shutdown

Initiates graceful shutdown: stops fetching and exits after processing all in-flight messages.

```bash
kill -USR1 $(cat tmp/pids/shoryuken.pid)
```

### Behavior

1. Stops fetching new messages
2. Waits for all in-flight messages to complete
3. Fires lifecycle events (`quiet`, `shutdown`, `stopped`)
4. Exits cleanly

### Use Cases

- Deployments
- Scaling down
- Graceful restarts

### Lifecycle Events

```
USR1 received
     │
     ▼
quiet event ──────► Stop accepting new work
     │
     ▼
Process in-flight messages
     │
     ▼
shutdown event ───► Close connections
     │
     ▼
stopped event ────► Final cleanup
     │
     ▼
Exit(0)
```

---

## TERM - Hard Shutdown

Initiates shutdown with a timeout. Workers that don't finish within the timeout are terminated.

```bash
kill -TERM $(cat tmp/pids/shoryuken.pid)
```

### Behavior

1. Stops fetching new messages
2. Waits up to `timeout` seconds for workers
3. Forcefully terminates remaining workers
4. Unfinished messages return to SQS (visibility timeout)
5. Exits

### Timeout Configuration

```yaml
# config/shoryuken.yml
timeout: 25  # seconds
```

Or via CLI:

```bash
shoryuken -t 25 ...
```

### Default Timeout

The default timeout is 8 seconds (designed for Heroku's 10-second limit).

### Use Cases

- Container termination (Docker, Kubernetes)
- Heroku dyno restarts
- Emergency shutdown

---

## INT - Interrupt

Same behavior as TERM. Typically sent when pressing Ctrl+C in a terminal.

```bash
# Ctrl+C in terminal
# or
kill -INT $(cat tmp/pids/shoryuken.pid)
```

---

## Container Orchestration

### Kubernetes

Kubernetes sends SIGTERM on pod termination:

```yaml
spec:
  terminationGracePeriodSeconds: 300  # 5 minutes
  containers:
    - name: worker
      # ...
```

Set Shoryuken timeout lower than grace period:

```yaml
# config/shoryuken.yml
timeout: 280  # Leave 20s buffer
```

### Docker / ECS

Configure stop timeout in task definition:

```json
{
  "stopTimeout": 300
}
```

### Heroku

Heroku allows 30 seconds for SIGTERM shutdown:

```yaml
# config/shoryuken.yml
timeout: 25  # Leave 5s buffer
```

---

## ActiveJob Continuations

With Rails 8.1+, jobs can check `stopping?` to checkpoint progress:

```ruby
class LongRunningJob < ApplicationJob
  def perform(data)
    data.each do |item|
      if stopping?
        # Save progress and re-enqueue
        self.class.perform_later(remaining_data)
        return
      end
      process(item)
    end
  end
end
```

The `stopping?` flag is set when:
- `USR1` is received
- `TERM` is received
- `stop` or `stop!` is called on the launcher

See [[ActiveJob Continuations]] for more details.

---

## Health Check Integration

During shutdown:

| Phase | `stopping?` | `healthy?` |
|-------|-------------|------------|
| Normal operation | `false` | `true` |
| After TERM/USR1 | `true` | `true` |
| Workers stopping | `true` | `true` |
| Workers stopped | `true` | `false` |

Use `stopping?` for readiness probes (stop accepting new connections):

```ruby
# Readiness check
if launcher&.stopping?
  [503, {}, ['Shutting down']]
else
  [200, {}, ['Ready']]
end
```

See [[Health Checks]] for more details.

---

## Signal Handling in Scripts

### Deployment Script

```bash
#!/bin/bash

PID_FILE="tmp/pids/shoryuken.pid"

if [ -f "$PID_FILE" ]; then
  echo "Sending USR1 to Shoryuken..."
  kill -USR1 $(cat $PID_FILE)

  # Wait for graceful shutdown
  while [ -f "$PID_FILE" ]; do
    sleep 1
  done

  echo "Shoryuken stopped"
fi

# Start new worker
bundle exec shoryuken -R -C config/shoryuken.yml -P $PID_FILE -d
echo "Shoryuken started"
```

### systemd Integration

```ini
[Service]
ExecReload=/bin/kill -USR1 $MAINPID
TimeoutStopSec=300
KillSignal=SIGTERM
```

---

## Debugging with Signals

### Finding Stuck Jobs

```bash
# Send TTIN to get thread dump
kill -TTIN $(cat tmp/pids/shoryuken.pid)

# Check logs for backtraces
tail -100 log/shoryuken.log
```

### Manual Drain

```bash
# Enter quiet mode
kill -TSTP $(cat tmp/pids/shoryuken.pid)

# Monitor queue processing
watch "cat tmp/pids/shoryuken.pid | xargs ps -o pid,stat,time"

# When ready, terminate
kill -USR1 $(cat tmp/pids/shoryuken.pid)
```

---

## Related

- [[Lifecycle Events]] - Event hooks during shutdown
- [[ActiveJob Continuations]] - Job checkpointing
- [[Health Checks]] - Health monitoring during shutdown
- [[Deployment]] - Deployment strategies
