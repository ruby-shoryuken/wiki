Shoryuken is a drop-in replacement for Sidekiq, the code changes should be minor `s/sidekiq/shoryuken`. But as Shoryuken "reads" messages from SQS, instead of Redis, you will probably need a three steps migration: 

* Stop sending jobs to Sidekiq
* Start using Shoryuken
* Keep Sidekiq running until it consumes all pending jobs.


## Worker

### Sidekiq

```ruby
class MyWorker
  include Sidekiq::Worker

  sidekiq_options queue: 'my_queue'

  def perform(arg)
    # ...
  end
end
```

### Shoryuken

```ruby
class MyWorker
  include Shoryuken::Worker

  shoryuken_options queue: 'my_queue', auto_delete: true

  def perform(sqs_msg, arg)
    # ...
  end
end
```

Note that if your Sidekiq worker has more than 1 argument you will need to configure message body parser for Shoryuken worker and define only one argument in a `#perform` method:

```ruby
class YourWorker
  include Shoryuken::Worker

  shoryuken_options queue: "queue", auto_delete: true, body_parser: JSON

  def perform(_sqs_message, args)
    args.fetch("user_id")
    args.fetch("post_id")
    # ...
  end
end

YourWorker.perform_async(user_id: "XXX", post_id: "YYY")
```


## Configuration file

### sidekiq.yml

```yaml
concurrency: 25
pidfile: tmp/pids/sidekiq.pid
queues:
  - default
  - [myqueue, 2]
```

### shoryuken.yml

```yaml
concurrency: 25
pidfile: tmp/pids/shoryuken.pid
queues:
  - default
  - [myqueue, 2]
```

## Sending messages

### Sidekiq

```ruby
MyWorker.perform_async('test')
```

### Shoryuken

```ruby
MyWorker.perform_async('test')
```

## To get it running

### Sidekiq

```shell
bundle exec sidekiq -r ./my_worker.rb -q my_queue
```

### Shoryuken

```shell
bundle exec shoryuken -r ./my_worker.rb -q my_queue
```