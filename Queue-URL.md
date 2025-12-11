The preferable way of configuring queues in Shoryuken is using their names, but in case you want to configure their URLs, Shoryuken also supports that:

### Via shoryuken.yml

```yaml
concurrency: 25
delay: 10
queues:
  - ['https://sqs.us-east-1.amazonaws.com/account-id/queue-name', 1]
```
### Via CLI

```shell
shoryuken -r ./workers.rb -q https://sqs.us-east-1.amazonaws.com/account-id/queue-name
```

### Worker

```ruby
class Worker
  include Shoryuken::Worker

  shoryuken_options queue: 'https://sqs.us-east-1.amazonaws.com/account-id/queue-name'

  def perform(sqs, body)
    # ...
  end
end
```

Configuring the queue URLs allows you to consume messages from multiple regions (and accounts).