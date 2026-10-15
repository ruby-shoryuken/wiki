# Complete Shoryuken Setup for Rails

This guide provides a step-by-step walkthrough for setting up Shoryuken with Rails and ActiveJob.

## Requirements

- Ruby 3.2+
- Rails 7.2+
- AWS account with SQS access

## 1. Create SQS Queue

### Using AWS Console

1. Log in to AWS Console
2. Navigate to SQS
3. Click "Create queue"
4. Choose Standard or FIFO
5. Configure settings (visibility timeout, retention)
6. Create queue

### Using Shoryuken CLI

```bash
shoryuken sqs create myapp-default
shoryuken sqs create myapp-critical
```

See `shoryuken sqs help create` for options.

---

## 2. Install Gems

```ruby
# Gemfile
gem 'shoryuken'
gem 'aws-sdk-sqs', '>= 1.66'
```

```bash
bundle install
```

---

## 3. Configure Rails

```ruby
# config/application.rb
module MyApp
  class Application < Rails::Application
    config.active_job.queue_adapter = :shoryuken
  end
end
```

**Note:** With Zeitwerk (Rails 7+), jobs in `app/jobs` are autoloaded automatically.

---

## 4. Configure AWS Credentials

Choose one method:

### Environment Variables (Development)

```bash
export AWS_ACCESS_KEY_ID=your_key
export AWS_SECRET_ACCESS_KEY=your_secret
export AWS_REGION=us-east-1
```

### AWS Credentials File

```ini
# ~/.aws/credentials
[default]
aws_access_key_id = your_key
aws_secret_access_key = your_secret
```

### IAM Role (Production)

Use Instance Profiles (EC2), Task Roles (ECS), or IRSA (EKS). See [[Configure the AWS Client]].

---

## 5. Create Configuration File

```yaml
# config/shoryuken.yml
concurrency: 25
timeout: 25
queues:
  - [myapp-critical, 3]
  - [myapp-default, 1]
```

---

## 6. Create Initializer

```ruby
# config/initializers/shoryuken.rb
Shoryuken.configure_server do |config|
  # Use Rails logger
  Rails.logger = Shoryuken::Logging.logger
  Rails.logger.level = Rails.application.config.log_level

  # Lifecycle events
  config.on(:startup) do
    Rails.logger.info "Shoryuken starting..."
  end

  config.on(:shutdown) do
    Rails.logger.info "Shoryuken shutting down..."
  end
end

# Enable ActiveJob queue name prefixing (optional)
Shoryuken.active_job_queue_name_prefixing = true
```

---

## 7. Create Jobs

```ruby
# app/jobs/application_job.rb
class ApplicationJob < ActiveJob::Base
  # Retry on transient failures
  retry_on StandardError, wait: :polynomially_longer, attempts: 5

  # Discard jobs with invalid records
  discard_on ActiveJob::DeserializationError
end
```

```ruby
# app/jobs/process_order_job.rb
class ProcessOrderJob < ApplicationJob
  queue_as :myapp_default

  def perform(order_id)
    order = Order.find(order_id)
    order.process!
  end
end
```

---

## 8. Enqueue Jobs

```ruby
# Enqueue for later processing
ProcessOrderJob.perform_later(order.id)

# Enqueue with delay
ProcessOrderJob.set(wait: 5.minutes).perform_later(order.id)

# Enqueue to specific queue
ProcessOrderJob.set(queue: :myapp_critical).perform_later(order.id)
```

---

## 9. Run Workers

### Development

```bash
bundle exec shoryuken -R -C config/shoryuken.yml
```

### With Foreman

```
# Procfile.dev
web: bin/rails server
worker: bundle exec shoryuken -R -C config/shoryuken.yml
```

```bash
foreman start -f Procfile.dev
```

---

## 10. Configure Database Pool

Ensure your database connection pool is at least equal to Shoryuken concurrency:

```yaml
# config/database.yml
production:
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 25 } %>
```

---

## Complete Example

### Gemfile

```ruby
gem 'rails', '~> 8.0'
gem 'shoryuken'
gem 'aws-sdk-sqs', '>= 1.66'
```

### config/shoryuken.yml

```yaml
concurrency: <%= ENV.fetch('SHORYUKEN_CONCURRENCY', 25) %>
timeout: 25
queues:
  - [<%= Rails.env %>_critical, 3]
  - [<%= Rails.env %>_default, 1]
```

### config/initializers/shoryuken.rb

```ruby
Shoryuken.configure_server do |config|
  Rails.logger = Shoryuken::Logging.logger
  Rails.logger.level = Rails.application.config.log_level

  config.on(:startup) do
    Rails.logger.info "[Shoryuken] Starting with #{Shoryuken.options[:concurrency]} threads"
  end
end

# Use environment-based queue prefixes
Shoryuken.active_job_queue_name_prefixing = true
```

### config/environments/production.rb

```ruby
# Enable ActiveJob queue prefix
config.active_job.queue_name_prefix = Rails.env
```

### app/jobs/application_job.rb

```ruby
class ApplicationJob < ActiveJob::Base
  retry_on StandardError, wait: :polynomially_longer, attempts: 5
  discard_on ActiveJob::DeserializationError
end
```

### app/jobs/send_notification_job.rb

```ruby
class SendNotificationJob < ApplicationJob
  queue_as :default

  def perform(user_id, message)
    user = User.find(user_id)
    NotificationService.send(user, message)
  end
end
```

### Usage

```ruby
# In a controller or service
SendNotificationJob.perform_later(current_user.id, "Welcome!")
```

---

## Next Steps

- [[Rails Integration Active Job]] - Advanced ActiveJob features
- [[ActiveJob Continuations]] - Handle worker shutdowns
- [[CurrentAttributes]] - Pass request context to jobs
- [[Deployment]] - Production deployment
- [[Health Checks]] - Health monitoring

---

## Troubleshooting

### "No worker found for queue"

Ensure queue names in `shoryuken.yml` match your job's `queue_as` setting.

### Connection Pool Exhaustion

Increase database pool to match concurrency:

```yaml
pool: <%= ENV.fetch("SHORYUKEN_CONCURRENCY", 25) %>
```

### AWS Credentials Not Found

Verify credentials are set:

```bash
aws sts get-caller-identity
```

See [[Configure the AWS Client]] for credential options.
