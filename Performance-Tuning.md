# Performance Tuning

This guide covers optimizing Shoryuken for throughput, latency, and cost efficiency.

## Concurrency

### Setting Concurrency

```yaml
# config/shoryuken.yml
concurrency: 25  # Default
```

### Guidelines

| Workload Type | Recommended Concurrency | Notes |
|--------------|------------------------|-------|
| CPU-bound | 1-2× CPU cores | Limited by CPU |
| I/O-bound (API calls) | 25-50 | Higher due to waiting |
| Database-heavy | 10-25 | Limited by connection pool |
| Mixed | 15-25 | Balance based on bottleneck |

### Memory Impact

Each thread consumes memory. Monitor and adjust:

```
Memory per worker ≈ Base Rails memory + (concurrency × per-thread overhead)
```

Typical per-thread overhead: 5-20 MB depending on job complexity.

---

## Database Connections

### Connection Pool Sizing

**Rule:** Pool size ≥ Shoryuken concurrency

```yaml
# config/database.yml
production:
  pool: <%= ENV.fetch("DB_POOL") { 25 } %>
```

### Connection Exhaustion Signs

- `ActiveRecord::ConnectionTimeoutError`
- Slow job processing
- Database connection spikes

### Solutions

```ruby
# config/initializers/shoryuken.rb
Shoryuken.configure_server do |config|
  # Add ActiveRecord middleware to return connections after each job
  config.server_middleware do |chain|
    chain.add Shoryuken::Middleware::Server::ActiveRecord
  end
end
```

---

## Polling Configuration

### Delay Setting

```yaml
# config/shoryuken.yml
delay: 0  # Seconds to wait before re-polling empty queue
```

| Delay | Behavior | Cost Impact |
|-------|----------|-------------|
| 0 | Continuous polling | Higher (more API calls) |
| 5-10 | Balanced | Medium |
| 25+ | Cost-optimized | Lower latency |

### Long Polling

SQS long polling (up to 20 seconds) reduces empty responses:

```ruby
# Set via receive_message options
Shoryuken.sqs_client_receive_message_opts = {
  wait_time_seconds: 20  # Long polling
}
```

**Benefits:**
- Fewer API calls (cost savings)
- Reduced empty responses
- Lower latency for new messages

---

## Queue Weights

Prioritize critical queues:

```yaml
# config/shoryuken.yml
queues:
  - [critical, 6]    # 6× more polling
  - [default, 3]     # 3× more polling
  - [low, 1]         # Baseline
```

### Strict Priority

For absolute priority (process all critical before default):

```yaml
# config/shoryuken.yml
polling: :strict_priority
queues:
  - critical
  - default
  - low
```

See [[Polling strategies]] for details.

---

## Processing Groups

Isolate workloads with different characteristics:

```yaml
# config/shoryuken.yml
groups:
  fast:
    concurrency: 50
    delay: 0
    queues:
      - [webhooks, 1]
  slow:
    concurrency: 5
    delay: 5
    queues:
      - [reports, 1]
```

**Use cases:**
- Separate fast/slow jobs
- Isolate resource-intensive jobs
- Different SLA requirements

---

## Batch Processing

### Enable Batch Mode

```ruby
class BatchWorker
  include Shoryuken::Worker

  shoryuken_options queue: 'batch_queue', batch: true

  def perform(sqs_msgs, bodies)
    # Process up to 10 messages at once
    bodies.each do |body|
      process(body)
    end
  end
end
```

### Benefits

- Reduced per-message overhead
- Better for high-volume, fast jobs
- Fewer database round-trips possible

### Considerations

- All-or-nothing visibility handling
- More complex error handling
- Not suitable for long-running jobs

---

## Message Size Optimization

### SQS Limits

| Limit | Value |
|-------|-------|
| Max message size | 256 KB |
| Max batch size | 10 messages |
| Max batch payload | 1 MB |

### Strategies

**Pass IDs, not data:**

```ruby
# BAD - Large payload
ProcessDataJob.perform_later(large_data_hash)

# GOOD - Reference by ID
ProcessDataJob.perform_later(record_id: 123)
```

**Use S3 for large payloads:**

```ruby
class LargeDataJob < ApplicationJob
  def perform(s3_key)
    data = S3Client.get_object(bucket: 'jobs', key: s3_key)
    process(data.body.read)
  end
end
```

---

## Visibility Timeout

### Setting

Configure in SQS console or via API. Default: 30 seconds.

### Guidelines

```
Visibility timeout > Max job execution time + buffer
```

**Example:** If jobs take up to 60 seconds, set visibility to 90-120 seconds.

### Auto-Extend

For variable-length jobs:

```ruby
class LongRunningWorker
  include Shoryuken::Worker

  shoryuken_options queue: 'long_jobs', auto_visibility_timeout: true

  def perform(sqs_msg, body)
    # Visibility automatically extended during processing
    long_operation
  end
end
```

---

## Memory Optimization

### Monitor Memory

```ruby
# config/initializers/shoryuken.rb
Shoryuken.configure_server do |config|
  config.on(:utilization_update) do |group, utilization|
    memory_mb = `ps -o rss= -p #{Process.pid}`.to_i / 1024
    Rails.logger.info "Memory: #{memory_mb} MB, Utilization: #{utilization}"
  end
end
```

### Reduce Memory Usage

1. **Lower concurrency** - Fewer threads = less memory
2. **Process in batches** - Load data incrementally
3. **Clear caches** - Reset memoized data between jobs
4. **Avoid loading large datasets** - Stream or paginate

### Memory Leaks

Watch for growing memory over time:

```ruby
class MemoryCheckMiddleware
  def call(worker, queue, sqs_msg, body)
    yield
  ensure
    GC.start if rand < 0.01  # Occasional GC
  end
end
```

---

## Cost Optimization

### SQS Pricing Factors

- Number of requests (polling, sending, deleting)
- Data transfer (cross-region, internet)

### Reduce Costs

1. **Increase delay** for low-priority queues
2. **Use long polling** (fewer empty responses)
3. **Batch operations** where possible
4. **Same-region deployment** (no cross-region transfer)
5. **Cache visibility timeout** to reduce API calls:

```ruby
Shoryuken.cache_visibility_timeout = true
```

---

## Benchmarking

### Measure Throughput

```ruby
class MetricsMiddleware
  def initialize
    @processed = 0
    @start_time = Time.now
  end

  def call(worker, queue, sqs_msg, body)
    yield
    @processed += 1

    if @processed % 1000 == 0
      elapsed = Time.now - @start_time
      rate = @processed / elapsed
      Rails.logger.info "Throughput: #{rate.round(1)} jobs/sec"
    end
  end
end
```

### Key Metrics

- **Throughput**: Jobs processed per second
- **Latency**: Time from enqueue to completion
- **Queue depth**: Messages waiting
- **Error rate**: Failed jobs percentage
- **Utilization**: Worker thread usage

---

## Common Bottlenecks

| Symptom | Likely Cause | Solution |
|---------|--------------|----------|
| Low throughput, low CPU | I/O bound | Increase concurrency |
| High CPU, moderate throughput | CPU bound | Add workers, optimize code |
| Connection errors | Pool exhausted | Increase pool, add middleware |
| Growing queue depth | Under-provisioned | Add workers |
| Memory growth | Leak or large objects | Profile, reduce concurrency |

---

## Production Checklist

- [ ] Concurrency matches workload type
- [ ] Database pool ≥ concurrency
- [ ] Visibility timeout > max job time
- [ ] Long polling enabled
- [ ] Monitoring in place
- [ ] Memory limits set (containers)
- [ ] Auto-scaling configured

---

## Related

- [[Shoryuken options]] - Configuration reference
- [[Processing Groups]] - Workload isolation
- [[Polling strategies]] - Queue prioritization
- [[Monitoring and Metrics]] - Observability
