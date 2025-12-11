## Basic middleware

```ruby
class MyMiddleware
  def call(worker_instance, queue, sqs_msg, body)
    puts 'Before work'
    yield
    puts 'After work'
  end
end
```

### Registering a global middleware

```ruby
Shoryuken.configure_server do |config|
  config.server_middleware do |chain|
    chain.add MyMiddleware
    # chain.remove MyMiddleware
    # chain.add MyMiddleware, foo: 1, bar: 2
    # chain.insert_before MyMiddleware, MyMiddlewareNew
    # chain.insert_after MyMiddleware, MyMiddlewareNew
  end
end
```

### Registering per worker middleware

```ruby
class MyWorker
  include Shoryuken::Worker

  def perform(sqs_mg, body)
    # ...
  end

  server_middleware do |chain|
    # This will join all "global" middleware with `MyWorkerSpecificMiddleware`
    # if you want to run only `MyWorkerSpecificMiddleware` for this worker
    # you can `chain.clear`
    # or to remove specific middleware for this worker you can `chain.remove OtherMiddleware`
    chain.add MyWorkerSpecificMiddleware
  end
end
```

## Rejecting messages with a middleware

If you don't `yield` in a Middleware, no other middleware or worker will process it.

```ruby
class RejectInvalidMessagesMiddleware
  def call(worker_instance, queue, sqs_msg, body)
    if valid?(sqs_msg)
      # will consume the message
      yield
    else
      # will not consume the message
      Shoryuken.logger.info "sqs_msg '#{sqs_msg.id}' is invalid and was rejected"
      sqs_msg.delete
    end
  end
end
```


*Note:* When [batch is enabled](https://github.com/phstc/shoryuken/wiki/Worker-options#batch), the `sqs_msg` and `body` arguments are arrays.

