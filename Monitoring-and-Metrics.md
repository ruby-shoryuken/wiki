# Monitoring and Metrics

This guide covers monitoring Shoryuken workers and collecting metrics for observability.

## Lifecycle Events for Metrics

Use lifecycle events to emit metrics:

```ruby
# config/initializers/shoryuken.rb
Shoryuken.configure_server do |config|
  config.on(:startup) do
    Rails.logger.info "Shoryuken started"
    StatsD.increment('shoryuken.started')
  end

  config.on(:shutdown) do
    Rails.logger.info "Shoryuken stopped"
    StatsD.increment('shoryuken.stopped')
  end

  config.on(:utilization_update) do |group, utilization|
    StatsD.gauge("shoryuken.utilization.#{group}", utilization)
  end
end
```

---

## Key Metrics

### Worker Metrics

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| `jobs.processed` | Jobs completed | N/A (track rate) |
| `jobs.failed` | Jobs that raised errors | > 1% of processed |
| `jobs.duration` | Processing time | p99 > SLA |
| `utilization` | Thread usage (0-1) | Sustained > 0.9 |
| `memory_mb` | Process memory | > 80% of limit |

### Queue Metrics (SQS)

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| `ApproximateNumberOfMessagesVisible` | Queue depth | Growing trend |
| `ApproximateAgeOfOldestMessage` | Processing delay | > acceptable latency |
| `NumberOfMessagesReceived` | Throughput | Sudden drop |
| `NumberOfMessagesDeleted` | Completion rate | Diverging from received |

---

## Metrics Middleware

### StatsD / Datadog

```ruby
class MetricsMiddleware
  def call(worker_instance, queue, sqs_msg, body)
    start_time = Process.clock_gettime(Process::CLOCK_MONOTONIC)
    success = false

    yield
    success = true
  rescue => e
    StatsD.increment('shoryuken.jobs.failed', tags: [
      "queue:#{queue}",
      "worker:#{worker_instance.class.name}",
      "error:#{e.class.name}"
    ])
    raise
  ensure
    duration = Process.clock_gettime(Process::CLOCK_MONOTONIC) - start_time

    StatsD.timing('shoryuken.jobs.duration', duration * 1000, tags: [
      "queue:#{queue}",
      "worker:#{worker_instance.class.name}",
      "success:#{success}"
    ])

    StatsD.increment('shoryuken.jobs.processed', tags: [
      "queue:#{queue}",
      "success:#{success}"
    ])
  end
end

# Register
Shoryuken.configure_server do |config|
  config.server_middleware do |chain|
    chain.add MetricsMiddleware
  end
end
```

### Prometheus

```ruby
require 'prometheus/client'

class PrometheusMiddleware
  def initialize
    @registry = Prometheus::Client.registry

    @jobs_processed = @registry.counter(
      :shoryuken_jobs_processed_total,
      docstring: 'Total jobs processed',
      labels: [:queue, :worker, :status]
    )

    @job_duration = @registry.histogram(
      :shoryuken_job_duration_seconds,
      docstring: 'Job processing duration',
      labels: [:queue, :worker],
      buckets: [0.1, 0.5, 1, 5, 10, 30, 60]
    )
  end

  def call(worker_instance, queue, sqs_msg, body)
    start_time = Process.clock_gettime(Process::CLOCK_MONOTONIC)
    status = 'success'

    yield
  rescue => e
    status = 'failed'
    raise
  ensure
    duration = Process.clock_gettime(Process::CLOCK_MONOTONIC) - start_time
    worker = worker_instance.class.name

    @jobs_processed.increment(labels: { queue: queue, worker: worker, status: status })
    @job_duration.observe(duration, labels: { queue: queue, worker: worker })
  end
end
```

---

## CloudWatch Integration

### Custom Metrics

```ruby
require 'aws-sdk-cloudwatch'

class CloudWatchMetrics
  def initialize
    @cloudwatch = Aws::CloudWatch::Client.new
    @namespace = 'Shoryuken'
  end

  def record_job(queue:, duration:, success:)
    @cloudwatch.put_metric_data(
      namespace: @namespace,
      metric_data: [
        {
          metric_name: 'JobDuration',
          dimensions: [{ name: 'Queue', value: queue }],
          value: duration,
          unit: 'Seconds'
        },
        {
          metric_name: success ? 'JobsSucceeded' : 'JobsFailed',
          dimensions: [{ name: 'Queue', value: queue }],
          value: 1,
          unit: 'Count'
        }
      ]
    )
  end
end
```

### SQS Queue Metrics

SQS automatically publishes metrics to CloudWatch:

- `ApproximateNumberOfMessagesVisible`
- `ApproximateNumberOfMessagesNotVisible`
- `ApproximateAgeOfOldestMessage`
- `NumberOfMessagesReceived`
- `NumberOfMessagesDeleted`
- `NumberOfMessagesSent`

Access via CloudWatch console or API.

---

## Logging for Observability

### Structured Logging

```ruby
# config/initializers/shoryuken.rb
Shoryuken.configure_server do |config|
  config.on(:startup) do
    Shoryuken.logger.formatter = proc do |severity, datetime, progname, msg|
      {
        timestamp: datetime.iso8601(3),
        level: severity,
        message: msg,
        service: 'shoryuken',
        environment: Rails.env
      }.to_json + "\n"
    end
  end
end
```

### Job Context Logging

```ruby
class LoggingMiddleware
  def call(worker_instance, queue, sqs_msg, body)
    job_context = {
      queue: queue,
      worker: worker_instance.class.name,
      message_id: sqs_msg.message_id
    }

    Rails.logger.tagged(job_context.to_json) do
      Rails.logger.info "Job started"
      yield
      Rails.logger.info "Job completed"
    end
  rescue => e
    Rails.logger.error "Job failed: #{e.message}"
    raise
  end
end
```

---

## Health Monitoring

### Health Check Endpoint

```ruby
# config/initializers/shoryuken.rb
Shoryuken.configure_server do |config|
  config.on(:startup) do
    Thread.new do
      require 'webrick'
      server = WEBrick::HTTPServer.new(Port: 3001, Logger: WEBrick::Log.new("/dev/null"))

      server.mount_proc '/health' do |req, res|
        launcher = Shoryuken::Runner.instance.launcher
        if launcher&.healthy?
          res.status = 200
          res.body = JSON.generate(status: 'healthy')
        else
          res.status = 503
          res.body = JSON.generate(status: 'unhealthy')
        end
        res['Content-Type'] = 'application/json'
      end

      server.mount_proc '/metrics' do |req, res|
        # Expose Prometheus metrics
        res.status = 200
        res.body = Prometheus::Client::Formats::Text.marshal(Prometheus::Client.registry)
        res['Content-Type'] = 'text/plain'
      end

      server.start
    end
  end
end
```

See [[Health Checks]] for more details.

---

## Alerting

### Recommended Alerts

| Alert | Condition | Severity |
|-------|-----------|----------|
| High error rate | Failed jobs > 5% in 5 min | Critical |
| Queue backlog | Queue depth > 10,000 | Warning |
| Old messages | Oldest message > 1 hour | Warning |
| Worker down | No heartbeat in 5 min | Critical |
| High latency | p99 duration > 60s | Warning |
| Memory pressure | Memory > 90% limit | Warning |

### Example: Datadog Alert

```yaml
# Datadog monitor definition
name: "Shoryuken High Error Rate"
type: "metric alert"
query: "sum(last_5m):sum:shoryuken.jobs.failed{*} / sum:shoryuken.jobs.processed{*} > 0.05"
message: "Shoryuken error rate is above 5%"
thresholds:
  critical: 0.05
  warning: 0.02
```

### Example: CloudWatch Alarm

```json
{
  "AlarmName": "SQS-Queue-Depth-High",
  "MetricName": "ApproximateNumberOfMessagesVisible",
  "Namespace": "AWS/SQS",
  "Dimensions": [
    {"Name": "QueueName", "Value": "myapp-default"}
  ],
  "Statistic": "Average",
  "Period": 300,
  "EvaluationPeriods": 2,
  "Threshold": 10000,
  "ComparisonOperator": "GreaterThanThreshold"
}
```

---

## Dashboards

### Key Panels

1. **Throughput** - Jobs processed per minute
2. **Error Rate** - Failed / Total percentage
3. **Latency** - p50, p95, p99 duration
4. **Queue Depth** - Messages waiting per queue
5. **Worker Utilization** - Thread usage
6. **Memory/CPU** - Resource consumption

### Grafana Dashboard Example

```json
{
  "panels": [
    {
      "title": "Jobs Processed",
      "type": "graph",
      "targets": [
        {
          "expr": "rate(shoryuken_jobs_processed_total[5m])",
          "legendFormat": "{{queue}} - {{status}}"
        }
      ]
    },
    {
      "title": "Job Duration (p99)",
      "type": "graph",
      "targets": [
        {
          "expr": "histogram_quantile(0.99, rate(shoryuken_job_duration_seconds_bucket[5m]))",
          "legendFormat": "{{queue}}"
        }
      ]
    }
  ]
}
```

---

## Distributed Tracing

### OpenTelemetry

```ruby
require 'opentelemetry-sdk'

class TracingMiddleware
  def call(worker_instance, queue, sqs_msg, body)
    tracer = OpenTelemetry.tracer_provider.tracer('shoryuken')

    tracer.in_span("#{worker_instance.class.name}#perform") do |span|
      span.set_attribute('messaging.system', 'aws_sqs')
      span.set_attribute('messaging.destination', queue)
      span.set_attribute('messaging.message_id', sqs_msg.message_id)

      yield
    end
  end
end
```

---

## Related

- [[Lifecycle Events]] - Event hooks
- [[Health Checks]] - Health endpoints
- [[Performance Tuning]] - Optimization
- [[Logging]] - Log configuration
