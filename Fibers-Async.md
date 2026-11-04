Shoryuken runs jobs on threads. You can still send messages from fibers and use fibers inside jobs, for example with the [async](https://github.com/socketry/async) gem or a fiber-based server such as Falcon.

## Sending messages from fibers

`Queue#send_message`, `perform_async`, `perform_in` and the ActiveJob adapter work from fibers, for example from Async tasks:

```ruby
require 'async'

Sync do |task|
  tasks = Array.new(10) { |i| task.async { MyWorker.perform_async("item-#{i}") } }
  tasks.each(&:wait)
end
```

### CurrentAttributes in fiber-based apps

If your app runs its work as fibers on the same thread (for example on Falcon) and uses [[CurrentAttributes]], set:

```ruby
ActiveSupport::IsolatedExecutionState.isolation_level = :fiber
```

In a Rails app, `config.active_support.isolation_level = :fiber` sets it (see [`config.active_support.isolation_level`](https://guides.rubyonrails.org/configuring.html#config-active-support-isolation-level)). With the default, `:thread`, all fibers of a thread share one set of CurrentAttributes, so a job can get the attributes of another fiber.

## Async inside jobs

A job can run its work in Async tasks. Put them in a `Sync` or `Async` block: the block runs a Fiber scheduler for its tasks and removes it when it returns, so the thread has no scheduler when the next job starts.

```ruby
require 'async'
require 'net/http'

class FetchWorker
  include Shoryuken::Worker

  shoryuken_options queue: 'fetch', auto_delete: true

  def perform(_sqs_msg, body)
    Sync do |task|
      body.split(',').map { |url| task.async { Net::HTTP.get(URI(url)) } }.map(&:wait)
    end
  end
end
```

If a job sets a scheduler itself with `Fiber.set_scheduler`, it must remove it (`Fiber.set_scheduler(nil)`) before it returns.

### A job that leaves a Fiber scheduler set

After each job, including one that raises, the Shoryuken process checks the Fiber scheduler of the job's thread. If it is not the one set before the job, Shoryuken sets back the previous one and logs a warning:

```
A job on <queue> left a Fiber scheduler set; Shoryuken set back the previous one after the job
```
