## Plain Ruby

### Create a worker

```ruby
class HelloWorker
  include Shoryuken::Worker

  shoryuken_options queue: 'hello', auto_delete: true

  def perform(sqs_msg, name)
    puts "Hello, #{name}"
  end
end
```

### Create a queue

```shell
bundle exec shoryuken sqs create hello
```

### Start Shoryuken

```shell
bundle exec shoryuken -q hello -r ./hello_worker.rb
```

### Enqueue a message

```ruby
HelloWorker.perform_async('Ken')
```

## Rails

### Create a job

```ruby
# app/jobs/hello_job.rb
class HelloJob < ActiveJob::Base
  queue_as 'hello'

  def perform(name)
    puts "Hello, #{name}"
  end
end
```

### Create a queue

```shell
bundle exec shoryuken sqs create hello
```

### Set the queue backend

```ruby
# config/application.rb
module YourApp
  class Application < Rails::Application
    config.active_job.queue_adapter = :shoryuken
  end
end
```

### Start Shoryuken

```shell
bundle exec shoryuken -q hello -R
```

### Enqueue a message

```ruby
HelloJob.perform_later('Ken')
```

