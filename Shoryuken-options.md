# Shoryuken Options

Shoryuken can be configured via CLI options, a YAML configuration file, or programmatically.

## Configuration Methods

### CLI Options

```shell
shoryuken -q queue1 -q queue2 -c 25 -R -C config/shoryuken.yml
```

### Configuration File

```yaml
# config/shoryuken.yml
concurrency: 25
timeout: 25
queues:
  - [default, 2]
  - [low_priority, 1]
```

Start with the config file:

```shell
shoryuken -C config/shoryuken.yml -R
```

### Programmatic Configuration

```ruby
# config/initializers/shoryuken.rb
Shoryuken.configure_server do |config|
  # Server-side configuration
end

Shoryuken.configure_client do |config|
  # Client-side configuration (when enqueuing jobs)
end
```

---

## CLI Options

| Option | Description |
|--------|-------------|
| `-C, --config FILE` | Path to YAML configuration file |
| `-R, --rails` | Load Rails application |
| `-r, --require FILE` | Require a file or directory |
| `-q, --queue QUEUE[,WEIGHT]` | Queue to process (can specify multiple) |
| `-c, --concurrency INT` | Number of worker threads |
| `-t, --timeout INT` | Shutdown timeout in seconds |
| `-L, --logfile FILE` | Log file path |
| `-P, --pidfile FILE` | PID file path |
| `-d, --daemon` | Daemonize process |

### Examples

```shell
# Basic usage with Rails
shoryuken -R -C config/shoryuken.yml

# Multiple queues with weights
shoryuken -R -q critical,3 -q default,2 -q low,1

# With logging and PID file
shoryuken -R -C config/shoryuken.yml -L log/shoryuken.log -P tmp/pids/shoryuken.pid

# Daemonize
shoryuken -R -C config/shoryuken.yml -d
```

For all CLI options:

```shell
shoryuken help start
```

---

## Configuration File Options

### concurrency

Number of worker threads. Default: `25`

```yaml
concurrency: 25
```

**Note:** Set your database connection pool size to at least this value.

### timeout

Shutdown timeout in seconds. Default: `8`

```yaml
timeout: 25
```

Workers that don't finish within this time during SIGTERM are forcefully terminated.

### delay

Seconds to pause before re-polling an empty queue. Default: `0`

```yaml
delay: 25
```

Setting `delay: 0` polls continuously (faster but more expensive).

### queues

Queues to process with optional weights:

```yaml
# Simple list
queues:
  - queue1
  - queue2

# With weights (higher = more priority)
queues:
  - [critical, 3]
  - [default, 2]
  - [low, 1]

# Using URLs (for cross-account/cross-region)
queues:
  - https://sqs.us-east-1.amazonaws.com/123456789/myqueue

# Using ARNs
queues:
  - arn:aws:sqs:us-east-1:123456789:myqueue
```

### logfile

Path to log file:

```yaml
logfile: log/shoryuken.log
```

### pidfile

Path to PID file:

```yaml
pidfile: tmp/pids/shoryuken.pid
```

---

## Processing Groups

Isolate queues with separate concurrency settings:

```yaml
groups:
  critical:
    concurrency: 10
    delay: 0
    queues:
      - [payments, 1]
  default:
    concurrency: 25
    delay: 5
    queues:
      - [emails, 2]
      - [reports, 1]
```

See [[Processing Groups]] for more details.

---

## AWS Configuration

Configure AWS credentials in the config file:

```yaml
aws:
  access_key_id: <%= ENV['AWS_ACCESS_KEY_ID'] %>
  secret_access_key: <%= ENV['AWS_SECRET_ACCESS_KEY'] %>
  region: us-east-1
```

**Recommended:** Use IAM roles instead of access keys. See [[Configure the AWS Client]].

---

## Programmatic Options

### Server Configuration

```ruby
Shoryuken.configure_server do |config|
  # Logger
  config.logger = Rails.logger
  config.logger.level = Logger::INFO

  # Middleware
  config.server_middleware do |chain|
    chain.add MyMiddleware
  end

  # Lifecycle events
  config.on(:startup) { puts "Starting" }
  config.on(:shutdown) { puts "Stopping" }

  # Exception handler
  config.on_exception do |ex, queue, sqs_msg|
    Sentry.capture_exception(ex)
  end
end
```

### Client Configuration

```ruby
Shoryuken.configure_client do |config|
  # SQS client configuration
  config.sqs_client = Aws::SQS::Client.new(region: 'us-east-1')
end
```

### Dynamic Queue Registration

```ruby
# Add a queue at runtime
Shoryuken.add_queue('new_queue', 1, 'default')

# Add a group
Shoryuken.add_group('critical', 10)
Shoryuken.add_queue('payments', 1, 'critical')
```

---

## Other Options

### active_job_queue_name_prefixing

Honor ActiveJob queue name prefixes:

```ruby
# config/initializers/shoryuken.rb
Shoryuken.active_job_queue_name_prefixing = true
```

### cache_visibility_timeout

Cache queue visibility timeout to reduce SQS API calls:

```ruby
Shoryuken.cache_visibility_timeout = true
```

**Note:** With caching enabled, restart Shoryuken after changing visibility timeout in SQS.

---

## ERB in Configuration Files

Configuration files support ERB:

```yaml
# config/shoryuken.yml
concurrency: <%= ENV.fetch('SHORYUKEN_CONCURRENCY', 25) %>
timeout: <%= ENV.fetch('SHORYUKEN_TIMEOUT', 25) %>
queues:
  - <%= ENV.fetch('QUEUE_NAME', 'default') %>
```

---

## Environment-Specific Configuration

```yaml
# config/shoryuken.yml
<% if Rails.env.production? %>
concurrency: 50
<% else %>
concurrency: 5
<% end %>

queues:
  - <%= "#{Rails.env}_default" %>
```

---

## Complete Example

```yaml
# config/shoryuken.yml
concurrency: <%= ENV.fetch('SHORYUKEN_CONCURRENCY', 25) %>
timeout: 25
delay: 5
logfile: <%= Rails.root.join('log', 'shoryuken.log') %>
pidfile: <%= Rails.root.join('tmp', 'pids', 'shoryuken.pid') %>

groups:
  critical:
    concurrency: 10
    delay: 0
    queues:
      - [payments, 1]
      - [webhooks, 1]
  default:
    concurrency: 25
    delay: 5
    queues:
      - [emails, 3]
      - [reports, 2]
      - [cleanup, 1]
```

```ruby
# config/initializers/shoryuken.rb
Shoryuken.configure_server do |config|
  Rails.logger = Shoryuken::Logging.logger
  Rails.logger.level = Rails.application.config.log_level

  config.on(:startup) do
    Rails.logger.info "Shoryuken starting with concurrency #{Shoryuken.options[:concurrency]}"
  end

  config.on(:shutdown) do
    Rails.logger.info "Shoryuken shutting down"
  end
end
```

---

## Related

- [[Worker options]] - Per-worker configuration
- [[Processing Groups]] - Queue isolation
- [[Configure the AWS Client]] - AWS credentials
- [[Polling strategies]] - Queue polling behavior
