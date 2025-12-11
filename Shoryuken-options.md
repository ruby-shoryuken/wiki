Shoryuken options can be set via CLI:

```shell
shoryuken -q queue1 -L ./shoryuken.log -P ./shoryuken.pid
```
or via configuration file: 

```yaml
# shoryuken.yml
logfile: ./shoryuken.log
pidfile: ./shoryuken.pid
queues: 
  - queue1
```

When using a configuration file, you must set via CLI `shoryuken -C ./shoryuken.yml`, otherwise Shoryuken won't use it.

Some options available in the configuration file, are not available in the CLI. For checking all options available in the CLI: `shoryuken help start`.

## delay

Default: `0`

Delay is the number of seconds to pause fetching from when an empty queue.

Given this configuration:

```yaml
delay: 25
queues:
  - queue1
  - queue2
```

If Shoryuken tries to fetch messages from `queue1` and it has no messages, Shoryuken will pause fetching from `queue1` for 25 seconds.

Usually having a delay is more cost-efficient, but if you want to consume messages as soon as they get in the queue, I would recommend setting `delay: 0` (default value) to stop pausing empty queues. 

[Check the AWS SQS pricing page for more information](https://aws.amazon.com/sqs/pricing/).

## queues

A single Shoryuken process can consume messages from multiple queues. You can define your queues as follows:

```yaml
queues:
  - queue1
  - queue2
```

You can also configure the queue URLs instead of names.

```yaml
queues:
  - https://sqs.eu-west-1.amazonaws.com:000/1111111111/queue1
  - https://sqs.eu-east-1.amazonaws.com:000/2222222222/queue2
```

Or ARN.

```yaml
queues:
  - arn:aws:sqs:us-east-1:123456789012:queue1
  - arn:aws:sqs:us-east-1:123456789012:queue2
```

### Load balancing

Supposing you have `queue1`, which you would like to fetch messages twice as much as `queue2`, you can configure that as follows:

```yaml
queues:
  - [queue1, 8]
  - [queue2, 4]
  - [queue3, 1]
```

The setup above will cause Shoryuken to fetch messages in cycles of `queue1` 8 times, then `queue2` 4 times, then `queue3` once, then repeat.

*Note:* Each fetch can fetch up to 10 messages at a time (SQS limitation). The fetch size will also depend on the available workers at the time of the fetch.

See [Polling strategies](https://github.com/phstc/shoryuken/wiki/Polling-strategies#weightedroundrobin).

## concurrency

Default: `25`

Concurrency is the number of threads available for processing messages at a time. Be careful with changing this value, otherwise, you could crush your machine with I/O.

*Note:* When accessing databases (`ActiveRecord`) or other resources with `pool` support in your workers, make sure to set `pool` size with the same value or higher than the concurrency set in Shoryuken.

```yaml
concurrency: <%= ENV.fetch('YOUR_CONCURRENCY', 25) %>
```

## cache_visibility_timeout

By default, Shoryuken will make requests against SQS for getting the updated queue visibility timeout, so if you change the visibility timeout in SQS, Shoryuken will automatically update it. 

If you want to reduce the number of requests against SQS (therefore reducing your quota usage), you can disable this behavior by:

```ruby
Shoryuken.cache_visibility_timeout = true
```

With `cache_visibility_timeout` enabled, every time you change the visibility timeout in SQS, you will need to restart Shoryuken, otherwise, it won't get updated.