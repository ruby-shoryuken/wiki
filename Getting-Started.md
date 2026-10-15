# Getting Started

## Prerequisites

Shoryuken requires:

- **Ruby 3.2+**
- **Rails 7.2+** (for Rails/ActiveJob integration)
- **aws-sdk-sqs >= 1.66**

## Installation

Add Shoryuken to your Gemfile:

```ruby
gem 'shoryuken'
```

Run bundler:

```shell
bundle install
```

## AWS Configuration

Shoryuken uses the AWS SDK for Ruby. Configure your AWS credentials using one of these methods:

1. **Environment variables** (recommended for development):
   ```shell
   export AWS_ACCESS_KEY_ID=your_access_key
   export AWS_SECRET_ACCESS_KEY=your_secret_key
   export AWS_REGION=us-east-1
   ```

2. **AWS credentials file** (`~/.aws/credentials`):
   ```ini
   [default]
   aws_access_key_id = your_access_key
   aws_secret_access_key = your_secret_key
   ```

3. **IAM role** (recommended for production on AWS):
   - EC2 instance profile
   - ECS task role
   - EKS service account

See [[Configure the AWS Client]] for more details.

---

## Plain Ruby

### 1. Create a Worker

```ruby
# hello_worker.rb
class HelloWorker
  include Shoryuken::Worker

  shoryuken_options queue: 'hello', auto_delete: true

  def perform(sqs_msg, name)
    puts "Hello, #{name}"
  end
end
```

### 2. Create a Queue

```shell
bundle exec shoryuken sqs create hello
```

### 3. Start Shoryuken

```shell
bundle exec shoryuken -q hello -r ./hello_worker.rb
```

### 4. Enqueue a Message

```ruby
require_relative 'hello_worker'

HelloWorker.perform_async('Ken')
```

---

## Rails

### 1. Create an ApplicationJob

```ruby
# app/jobs/application_job.rb
class ApplicationJob < ActiveJob::Base
  # Common job configuration can go here
end
```

### 2. Create a Job

```ruby
# app/jobs/hello_job.rb
class HelloJob < ApplicationJob
  queue_as 'hello'

  def perform(name)
    puts "Hello, #{name}"
  end
end
```

### 3. Create a Queue

```shell
bundle exec shoryuken sqs create hello
```

### 4. Configure the Queue Adapter

```ruby
# config/application.rb
module YourApp
  class Application < Rails::Application
    config.active_job.queue_adapter = :shoryuken
  end
end
```

### 5. Create a Shoryuken Initializer (Optional)

```ruby
# config/initializers/shoryuken.rb
Shoryuken.configure_server do |config|
  # Replace Rails logger so messages are logged to Shoryuken's log
  Rails.logger = Shoryuken::Logging.logger
  Rails.logger.level = Rails.application.config.log_level

  # Add server middleware
  # config.server_middleware do |chain|
  #   chain.add MyMiddleware
  # end
end

Shoryuken.configure_client do |config|
  # Client-side configuration (used when enqueuing jobs)
end
```

### 6. Start Shoryuken

```shell
bundle exec shoryuken -q hello -R
```

The `-R` flag loads your Rails application. Workers in `app/jobs` are auto-loaded via Zeitwerk.

### 7. Enqueue a Message

```ruby
HelloJob.perform_later('Ken')
```

---

## Configuration File

For more complex setups, create a configuration file:

```yaml
# config/shoryuken.yml
concurrency: 25
delay: 0
queues:
  - [default, 1]
  - [high_priority, 2]
```

Start Shoryuken with the configuration file:

```shell
bundle exec shoryuken -R -C config/shoryuken.yml
```

See [[Shoryuken options]] for all available configuration options.

---

## Next Steps

- [[Rails Integration Active Job]] - Full ActiveJob integration guide
- [[Worker options]] - Configure worker behavior
- [[Middleware]] - Add cross-cutting concerns
- [[Deployment]] - Deploy to production
