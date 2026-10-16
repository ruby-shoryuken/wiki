Processing Groups is a way to set specific configurations, such as `queues`, `concurrency`, `long polling` per queue(s).

Given this configuration:

```yaml
concurrency: 25
queues:
  - queue1
  - queue2
  - queue3
```

Shoryuken will fetch messages from `queue1`, then `queue2`, then `queue3`, then repeat. See [[Polling strategies]].

If you don't want to wait on fetching for `queue1` to fetch for `queue2`, you can move them into separate groups:

```yaml
groups:
  group1:
    concurrency: 10
    queues:
      - queue1
  group2:
    concurrency: 10
    delay: 10
    queues:
      - queue2
```

To process only one message at a time for `queue1`, but keep `concurrency: 25` for `queue2` and `queue3`:

```yaml
concurrency: 25
queues:
  - queue2
  - queue3

groups:
  group1:
    concurrency: 1
    queues:
      - queue1
```

*Note:* If you want to make sure to process only one message at time for a given queue while running multiple Shoryuken processes, have a look at [[FIFO Queues]].

### Polling strategy per group


```yaml
groups:
  group1:
    polling_strategy: StrictPriority
    queues:
      - queue1
  group2:
    polling_strategy: WeightedRoundRobin # Default, declaration optional
    queues:
      - queue2
```

See [[Polling strategies]].

### Long polling per group

```ruby
# config/initializers/shoryuken.rb
Shoryuken.sqs_client_receive_message_opts[:group1] = { wait_time_seconds: 20 }
```

Group names from `shoryuken.yml` are Symbols, so use a Symbol key (`:group1`).

See [[Long Polling]].