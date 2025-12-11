# CurrentAttributes

Shoryuken can automatically persist Rails `ActiveSupport::CurrentAttributes` from the code that enqueues a job to the job's execution. This is useful for maintaining request context like current user, tenant, or locale across background job processing.

## Overview

When you enqueue a job, Shoryuken serializes your CurrentAttributes into the message payload. When the job executes, these attributes are restored before `perform` runs and reset afterward.

```
┌─────────────────────────────────────────────────────────────┐
│                    Request Context Flow                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  HTTP Request       →    Enqueue Job    →    Execute Job     │
│  Current.user = X       (serialize X)       Current.user = X │
│  Current.tenant = Y     (serialize Y)       Current.tenant= Y│
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Setup

### 1. Define Your CurrentAttributes

```ruby
# app/models/current.rb
class Current < ActiveSupport::CurrentAttributes
  attribute :user
  attribute :tenant
  attribute :request_id
  attribute :locale
end
```

### 2. Configure Shoryuken to Persist Attributes

```ruby
# config/initializers/shoryuken.rb
require 'shoryuken/active_job/current_attributes'
Shoryuken::ActiveJob::CurrentAttributes.persist('Current')
```

### 3. Set Attributes in Your Controller

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  before_action :set_current_attributes

  private

  def set_current_attributes
    Current.user = current_user
    Current.tenant = current_tenant
    Current.request_id = request.request_id
    Current.locale = I18n.locale
  end
end
```

### 4. Access Attributes in Your Jobs

```ruby
# app/jobs/audit_job.rb
class AuditJob < ApplicationJob
  queue_as :audit

  def perform(action:, resource:)
    AuditLog.create!(
      user: Current.user,
      tenant: Current.tenant,
      action: action,
      resource: resource,
      request_id: Current.request_id
    )
  end
end
```

## Multiple CurrentAttributes Classes

You can persist multiple CurrentAttributes classes:

```ruby
# app/models/current.rb
class Current < ActiveSupport::CurrentAttributes
  attribute :user, :request_id
end

# app/models/request_context.rb
class RequestContext < ActiveSupport::CurrentAttributes
  attribute :ip_address, :user_agent
end

# config/initializers/shoryuken.rb
require 'shoryuken/active_job/current_attributes'
Shoryuken::ActiveJob::CurrentAttributes.persist('Current', 'RequestContext')
```

Both classes will be serialized and restored.

## Serialization

CurrentAttributes are serialized using `ActiveJob::Arguments`, which supports:

- Primitive types (String, Integer, Float, etc.)
- Symbols
- Date, Time, DateTime
- Hash, Array
- GlobalID-compatible objects (ActiveRecord models)

### Example with ActiveRecord

```ruby
class Current < ActiveSupport::CurrentAttributes
  attribute :user  # ActiveRecord model
end

# In your controller
Current.user = User.find(123)

# In your job - the user is restored via GlobalID
def perform
  Current.user  # => #<User id: 123, ...>
end
```

### Handling Complex Objects

For objects that don't support GlobalID, store identifiers instead:

```ruby
class Current < ActiveSupport::CurrentAttributes
  attribute :user_id, :tenant_id

  def user
    @user ||= User.find(user_id) if user_id
  end

  def tenant
    @tenant ||= Tenant.find(tenant_id) if tenant_id
  end
end
```

## Error Handling

If an attribute fails to deserialize (e.g., the object was deleted), Shoryuken logs a warning but continues with the job:

```
WARN: Failed to restore CurrentAttributes Current: Couldn't find User with 'id'=123
```

The job still executes, but that attribute will be `nil`.

## Testing

### RSpec

```ruby
RSpec.describe AuditJob do
  let(:user) { create(:user) }
  let(:tenant) { create(:tenant) }

  before do
    Current.user = user
    Current.tenant = tenant
  end

  after do
    Current.reset
  end

  it 'creates audit log with current user' do
    expect {
      described_class.perform_now(action: 'login', resource: user)
    }.to change(AuditLog, :count).by(1)

    log = AuditLog.last
    expect(log.user).to eq(user)
    expect(log.tenant).to eq(tenant)
  end
end
```

### Minitest

```ruby
class AuditJobTest < ActiveJob::TestCase
  setup do
    @user = users(:one)
    @tenant = tenants(:one)
    Current.user = @user
    Current.tenant = @tenant
  end

  teardown do
    Current.reset
  end

  test 'creates audit log with current attributes' do
    AuditJob.perform_now(action: 'update', resource: @user)

    log = AuditLog.last
    assert_equal @user, log.user
    assert_equal @tenant, log.tenant
  end
end
```

## Common Use Cases

### Multi-Tenant Applications

```ruby
class Current < ActiveSupport::CurrentAttributes
  attribute :tenant

  def tenant=(value)
    super
    # Optionally set tenant on connection when restored
    ActiveRecord::Base.connection.execute("SET app.current_tenant = '#{value.id}'") if value
  end
end
```

### Request Tracing

```ruby
class Current < ActiveSupport::CurrentAttributes
  attribute :request_id, :correlation_id

  def request_id
    super || SecureRandom.uuid
  end
end

# In your job
class ProcessOrderJob < ApplicationJob
  def perform(order)
    Rails.logger.tagged(Current.request_id) do
      # All logs include the request_id
      order.process!
    end
  end
end
```

### Localization

```ruby
class Current < ActiveSupport::CurrentAttributes
  attribute :locale

  def locale=(value)
    super
    I18n.locale = value if value
  end
end

# Emails sent from jobs use the original request's locale
class WelcomeEmailJob < ApplicationJob
  def perform(user)
    I18n.with_locale(Current.locale || user.preferred_locale) do
      UserMailer.welcome(user).deliver_now
    end
  end
end
```

## Implementation Details

### How It Works

1. **On Enqueue**: The `Persistence` module prepends to `ShoryukenAdapter#message` and adds current attributes to the message body.

2. **On Execute**: The `Loading` module prepends to `JobWrapper#perform` and restores attributes before calling the original perform method.

3. **After Execute**: Attributes are reset in an `ensure` block to prevent leaking between jobs.

### Message Format

```json
{
  "job_class": "AuditJob",
  "arguments": ["action", "resource"],
  "cattr": {
    "user": { "_aj_globalid": "gid://app/User/123" },
    "tenant": { "_aj_globalid": "gid://app/Tenant/456" },
    "request_id": "abc-123"
  }
}
```

## Comparison with Sidekiq

This implementation is based on [Sidekiq's CurrentAttributes middleware](https://github.com/sidekiq/sidekiq/blob/main/lib/sidekiq/middleware/current_attributes.rb), using the same `ActiveJob::Arguments` serialization approach.

## Related

- [[Rails Integration Active Job]] - Basic ActiveJob setup
- [Rails CurrentAttributes Guide](https://api.rubyonrails.org/classes/ActiveSupport/CurrentAttributes.html)
