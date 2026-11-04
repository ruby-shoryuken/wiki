Shoryuken 7.0 can carry Rails `ActiveSupport::CurrentAttributes` from the code that enqueues an ActiveJob job to the job when it runs. It works only with the Shoryuken ActiveJob adapter, not with `Shoryuken::Worker` workers.

## Setup

```ruby
# config/initializers/shoryuken.rb
require 'shoryuken/active_job/current_attributes'

Shoryuken::ActiveJob::CurrentAttributes.persist('Current')
# Several classes:
# Shoryuken::ActiveJob::CurrentAttributes.persist('Current', 'RequestContext')
```

`persist` accepts class names or classes. You can call it more than once.

## Example

```ruby
class Current < ActiveSupport::CurrentAttributes
  attribute :user, :request_id
end

# In a controller
Current.user = current_user
Current.request_id = request.request_id
AuditJob.perform_later('login')

# In the job
class AuditJob < ActiveJob::Base
  def perform(action)
    Current.user       # the user from the enqueue
    Current.request_id # the request id from the enqueue
  end
end
```

## How it behaves

* On enqueue, the attributes of each registered class are serialized into the message body with `ActiveJob::Arguments`, so Symbols and GlobalID objects (for example ActiveRecord models) work. A class with no attributes set is not added.
* Before the job runs, the attributes are restored. An attribute the class no longer defines is skipped. If restoring fails, Shoryuken logs a warning and runs the job anyway.
* After every job, Shoryuken resets all registered classes, so values do not leak to the next job on the same thread.
* `ShoryukenConcurrentSendAdapter` keeps the attributes too: it reads them on the thread that enqueues the job and sets them for the send. Before [#1144](https://github.com/ruby-shoryuken/shoryuken/pull/1144), it built the message on its executor thread, so every job was sent without them.

## Fiber-based apps

If your app runs its work as fibers on the same thread (for example on Falcon), set `ActiveSupport::IsolatedExecutionState.isolation_level = :fiber` (in Rails: `config.active_support.isolation_level = :fiber`). With the default, `:thread`, all fibers of a thread share one set of CurrentAttributes, so a job can get the attributes of another fiber. See [[Fibers Async]].
