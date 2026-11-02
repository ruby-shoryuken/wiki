For retrieving which worker to call for each queue/message, Shoryuken uses [`DefaultWorkerRegistry`](https://github.com/ruby-shoryuken/shoryuken/blob/main/lib/shoryuken/default_worker_registry.rb), this is the default behavior, but if you have a special need, you can set a custom one.

## Extending the default registry

A custom registry subclasses `Shoryuken::DefaultWorkerRegistry` and overrides the methods it needs. The example below overrides `fetch_worker` and calls `super` to keep the default lookup: the class in the `shoryuken_class` message attribute, else the worker registered for the queue.

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

## Setting the registry

Register it in your initializer:

```ruby
# config/initializers/shoryuken.rb
Shoryuken.worker_registry = CustomWorkerRegistry.new
```

Set it before your worker classes load: each `shoryuken_options` call registers the worker in the registry that is set at that moment.
