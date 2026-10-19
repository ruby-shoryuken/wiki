## Prerequisites

Shoryuken requires:

- **Ruby 3.3+**
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

## AWS configuration

Shoryuken uses the AWS SDK for Ruby and its usual credential sources. See [[Configure the AWS Client]].

---

## Plain Ruby

### 1. Create a worker

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

### 2. Create a queue

```shell
bundle exec shoryuken sqs create hello
```

### 3. Start Shoryuken

```shell
bundle exec shoryuken -q hello -r ./hello_worker.rb
```

### 4. Enqueue a message

```ruby
require 'shoryuken'
require_relative 'hello_worker'

HelloWorker.perform_async('Ken')
```

---

## Rails

### 1. Create a job

```ruby
# app/jobs/hello_job.rb
class HelloJob < ActiveJob::Base
  queue_as 'hello'

  def perform(name)
    puts "Hello, #{name}"
  end
end
```

### 2. Create a queue

```shell
bundle exec shoryuken sqs create hello
```

### 3. Configure the queue adapter

```ruby
# config/application.rb
module YourApp
  class Application < Rails::Application
    config.active_job.queue_adapter = :shoryuken
  end
end
```

### 4. Start Shoryuken

```shell
bundle exec shoryuken -q hello -R
```

The `-R` flag loads your Rails application. For middleware and other settings, see the initializer in [[Rails Integration Active Job]].

### 5. Enqueue a message

```ruby
HelloJob.perform_later('Ken')
```

---

## Configuration file

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

