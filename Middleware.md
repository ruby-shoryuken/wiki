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

Arguments after the class are passed to its `new` as positional arguments, so `chain.add MyMiddleware, foo: 1, bar: 2` calls `MyMiddleware.new({ foo: 1, bar: 2 })`. Define `initialize(options)`, not keyword arguments. `chain.add` ignores a class that is already in the chain. `chain.prepend MyMiddleware` adds it at the start of the chain.

Middleware runs in chain order: the first one in the chain wraps all the others and the worker.

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
      Shoryuken.logger.info "sqs_msg '#{sqs_msg.message_id}' is invalid and was rejected"
      sqs_msg.delete
    end
  end
end
```


*Note:* When [batch is enabled](https://github.com/ruby-shoryuken/shoryuken/wiki/Worker-options#batch), the `sqs_msg` and `body` arguments are arrays.

## Built-in middleware

The default server chain holds, in this order:

| Middleware | What it does |
|------------|--------------|
| `Shoryuken::Middleware::Server::Timing` | Logs when processing starts and how long it took |
| `Shoryuken::Middleware::Server::NonRetryableException` | Deletes the message when the worker raises one of its [`non_retryable_exceptions`](https://github.com/ruby-shoryuken/shoryuken/wiki/Worker-options#non_retryable_exceptions) |
| `Shoryuken::Middleware::Server::ExponentialBackoffRetry` | Applies the worker's [`retry_intervals`](https://github.com/ruby-shoryuken/shoryuken/wiki/Worker-options#retry_intervals) |
| `Shoryuken::Middleware::Server::AutoDelete` | Deletes the message after success when [`auto_delete`](https://github.com/ruby-shoryuken/shoryuken/wiki/Worker-options#auto_delete) is on |
| `Shoryuken::Middleware::Server::AutoExtendVisibility` | Extends the visibility timeout when [`auto_visibility_timeout`](https://github.com/ruby-shoryuken/shoryuken/wiki/Worker-options#auto_visibility_timeout) is on |
| `Shoryuken::Middleware::Server::ActiveRecord` | Returns ActiveRecord connections to the pool after each message. Added only when `ActiveRecord::Base` is defined |

The four option-based middleware do nothing for a worker that does not use the option. Use `chain.remove` or `chain.insert_before` with these class names to change the default chain.
