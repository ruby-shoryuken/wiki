For retrieving which worker to call for each queue/message, Shoryuken uses [`DefaultWorkerRegistry`](https://github.com/phstc/shoryuken/blob/master/lib/shoryuken/default_worker_registry.rb), this is the default behavior, but if you have a special need, you can set a custom one.

```ruby
class CustomWorkerRegistry
  def fetch_worker(queue, message)
    ...
    worker_class = begin
                     worker_class.constantize
                   rescue
                     [your Worker here]
                   end
    ...
  end
end
```
...and register it in your initializer
```ruby
# config/initializers/shoryuken.rb
Shoryuken.worker_registry = CustomWorkerRegistry.new
```