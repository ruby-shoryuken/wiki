The preferred way to configure queues in Shoryuken is by name. Shoryuken also accepts queue URLs and ARNs, for example `arn:aws:sqs:us-east-1:account-id:queue-name`:

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

Queue URLs and ARNs let you consume messages from multiple regions (and accounts).