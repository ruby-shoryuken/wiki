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
