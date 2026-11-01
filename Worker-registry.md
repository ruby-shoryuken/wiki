For retrieving which worker to call for each queue/message, Shoryuken uses [`DefaultWorkerRegistry`](https://github.com/ruby-shoryuken/shoryuken/blob/main/lib/shoryuken/default_worker_registry.rb), this is the default behavior, but if you have a special need, you can set a custom one.

## Extending the default registry

The simplest custom registry subclasses `Shoryuken::DefaultWorkerRegistry` and overrides only `fetch_worker`. Call `super` to keep the default lookup: the class in the `shoryuken_class` message attribute, else the worker registered for the queue.

```ruby
class CustomWorkerRegistry < Shoryuken::DefaultWorkerRegistry
  def fetch_worker(queue, message)
    super || FallbackWorker.new
  end
end

class FallbackWorker
  include Shoryuken::Worker

  def perform(sqs_msg, body)
    # ...
  end
end
```

`fetch_worker` gets the queue name and a `Shoryuken::Message`, or an Array of messages for a `batch: true` worker. It returns a worker instance. When it returns `nil`, Shoryuken logs `No worker found for <queue>` and calls no worker.

## Writing a registry from scratch

A registry that does not subclass `DefaultWorkerRegistry` must implement every method of `Shoryuken::WorkerRegistry`. The base class raises `NotImplementedError` for each one.

- `batch_receive_messages?(queue)` - returns `true` when the queue's worker processes messages in batches. Shoryuken calls it for each fetch.
- `clear` - removes all worker registrations.
- `fetch_worker(queue, message)` - returns a worker instance for the message, or `nil`. Shoryuken calls it for each message or batch.
- `queues` - returns the names of the queues that have a registered worker. When ActiveJob is not loaded, Shoryuken calls it at start to log `No worker supplied for <queue>`.
- `register_worker(queue, clazz)` - registers a worker class for a queue. Each `shoryuken_options` call and the ActiveJob adapter call it.
- `workers(queue)` - returns the worker classes registered for the queue, or `[]`.

## Setting the registry

Register it in your initializer:

```ruby
# config/initializers/shoryuken.rb
Shoryuken.worker_registry = CustomWorkerRegistry.new
```

Set it before your worker classes load: each `shoryuken_options` call registers the worker in the registry that is set at that moment.
