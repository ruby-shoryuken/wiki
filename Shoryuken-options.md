Shoryuken can be configured via CLI options, a YAML configuration file, or Ruby code.

```shell
shoryuken -q queue1 -L ./shoryuken.log -P ./shoryuken.pid
```

or via a configuration file:

```yaml
# config/shoryuken.yml
logfile: ./shoryuken.log
pidfile: ./shoryuken.pid
queues:
  - queue1
```

When using a configuration file, you must pass it via CLI `shoryuken -C config/shoryuken.yml`, otherwise Shoryuken won't use it. When both set the same option, the CLI value wins.

Some options available in the configuration file are not available in the CLI (for example `groups` and `aws`).

## CLI options

| Option | Description |
|--------|-------------|
| `-C, --config FILE` | Path to the YAML configuration file |
| `-R, --rails` | Load the Rails application |
| `-r, --require PATH` | Dir or path of the workers |
| `-q, --queues QUEUE[,WEIGHT] ...` | Queues to process, with optional weights |
| `-c, --concurrency INT` | Processor threads to use |
| `-t, --timeout INT` | Hard shutdown timeout in seconds |
| `-D, --delay SECONDS` | Seconds to pause fetching from an empty queue |
| `-L, --logfile FILE` | Path to the log file |
| `-P, --pidfile FILE` | Path to the PID file |
| `-d, --daemon` | Daemonize the process (requires `--logfile`) |
| `-v, --verbose` | Print more verbose output (debug log level) |

Examples:

```shell
# Rails with a config file
shoryuken -R -C config/shoryuken.yml

# Several queues with weights
shoryuken -R -q critical,3 default,2 low,1

# Daemonize with a log file and a PID file
shoryuken -R -C config/shoryuken.yml -d -L log/shoryuken.log -P tmp/pids/shoryuken.pid
```

To list all CLI options: `shoryuken help start`.

## Configuration file options

### concurrency

Default: `25`

Concurrency is the number of threads available for processing messages at a time. Be careful with changing this value, otherwise, you could crush your machine with I/O.

*Note:* When accessing databases (`ActiveRecord`) or other resources with `pool` support in your workers, make sure to set `pool` size with the same value or higher than the concurrency set in Shoryuken.

```yaml
concurrency: <%= ENV.fetch('YOUR_CONCURRENCY', 25) %>
```

### timeout

Default: `8`

The number of seconds Shoryuken waits for busy workers on shutdown. Workers that do not finish within this time are killed. See [[Signals]].

```yaml
timeout: 25
```

### delay

Default: `0`

Delay is the number of seconds to pause fetching from an empty queue.

Given this configuration:

```yaml
delay: 25
queues:
  - queue1
  - queue2
```

If Shoryuken tries to fetch messages from `queue1` and it has no messages, Shoryuken will pause fetching from `queue1` for 25 seconds.

Usually having a delay is more cost-efficient, but if you want to consume messages as soon as they get in the queue, keep `delay: 0` (default value) to stop pausing empty queues.

[Check the AWS SQS pricing page for more information](https://aws.amazon.com/sqs/pricing/).

### queues

A single Shoryuken process can consume messages from multiple queues. You can define your queues as follows:

```yaml
queues:
  - queue1
  - queue2
```

You can also configure the queue URLs instead of names.

```yaml
queues:
  - https://sqs.eu-west-1.amazonaws.com/1111111111/queue1
  - https://sqs.us-east-1.amazonaws.com/2222222222/queue2
```

Or ARN.

```yaml
queues:
  - arn:aws:sqs:us-east-1:123456789012:queue1
  - arn:aws:sqs:us-east-1:123456789012:queue2
```

#### Load balancing

Supposing you have `queue1`, which you would like to fetch messages twice as much as `queue2`, you can configure that as follows:

```yaml
queues:
  - [queue1, 8]
  - [queue2, 4]
  - [queue3, 1]
```

The setup above will cause Shoryuken to fetch messages in cycles of `queue1` 8 times, then `queue2` 4 times, then `queue3` once, then repeat.

*Note:* Each fetch can fetch up to 10 messages at a time (SQS limitation). The fetch size will also depend on the available workers at the time of the fetch.

See [[Polling strategies]].

### groups

Processing groups give a set of queues its own concurrency, delay and polling strategy:

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
    polling_strategy: StrictPriority
    queues:
      - [emails, 2]
      - [reports, 1]
```

See [[Processing Groups]].

### logfile and pidfile

```yaml
logfile: log/shoryuken.log
pidfile: tmp/pids/shoryuken.pid
```

### aws

Options under `aws` are passed to `Aws::SQS::Client.new` when Shoryuken builds its SQS client:

```yaml
aws:
  region: us-east-1
```

Prefer IAM roles over access keys. See [[Configure the AWS Client]].

### ERB

The configuration file is evaluated with ERB. With `-R`, Rails is loaded before the file is read, so you can use `Rails.env` and `Rails.root`:

```yaml
concurrency: <%= ENV.fetch('SHORYUKEN_CONCURRENCY', 25) %>
queues:
  - <%= "#{Rails.env}_default" %>
```

## Options set in Ruby

These options have no CLI flag or YAML key. Set them in an initializer (for example `config/initializers/shoryuken.rb`).

### cache_visibility_timeout

By default, Shoryuken will make requests against SQS for getting the updated queue visibility timeout, so if you change the visibility timeout in SQS, Shoryuken will automatically update it.

If you want to reduce the number of requests against SQS (therefore reducing your quota usage), you can disable this behavior by:

```ruby
Shoryuken.cache_visibility_timeout = true
```

With `cache_visibility_timeout` enabled, every time you change the visibility timeout in SQS, you will need to restart Shoryuken, otherwise, it won't get updated.

### active_job_queue_name_prefixing

Default: `false`

Set it to `true` to apply the ActiveJob `queue_name_prefix` to the queue names Shoryuken reads from. See [[Rails Integration Active Job]].

```ruby
Shoryuken.active_job_queue_name_prefixing = true
```

### fifo_message_deduplication and active_job_fifo_message_deduplication

Default: `true`

Control whether Shoryuken generates a Message Deduplication ID for messages sent to FIFO queues. See [[FIFO Queues]].

```ruby
Shoryuken.fifo_message_deduplication = false            # since 7.0.4
Shoryuken.active_job_fifo_message_deduplication = false # since 7.0.3
```

### Server and client configuration

`configure_server` runs only in the Shoryuken process. `configure_client` runs only outside it (for example in a web process that enqueues jobs).

```ruby
Shoryuken.configure_server do |config|
  config.server_middleware do |chain|
    chain.add MyMiddleware
  end

  config.on(:startup) { Shoryuken.logger.info 'Starting' }
  config.on(:shutdown) { Shoryuken.logger.info 'Stopping' }
end

Shoryuken.configure_client do |config|
  config.sqs_client = Aws::SQS::Client.new(region: 'us-east-1')
end
```

See [[Middleware]] and [[Lifecycle Events]].

### exception_handlers

Each handler is called with the exception, the queue name and the SQS message when a worker raises. The default list holds `Shoryuken::DefaultExceptionHandler`, which logs the error.

```ruby
Shoryuken.configure_server do |config|
  config.exception_handlers << ->(exception, queue, sqs_msg) do
    # report to your error tracker of choice
  end
end
```

### Adding groups and queues in code

```ruby
Shoryuken.add_group('critical', 10)
Shoryuken.add_queue('payments', 1, 'critical')
```

`add_group` also takes `delay:` and `polling_strategy:` keyword arguments.

