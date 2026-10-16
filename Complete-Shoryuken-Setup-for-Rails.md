This guide sets up Shoryuken 7.0 with ActiveJob in a Rails 7.2+ application.

## Create SQS queue

Login to your Amazon AWS Console and go to SQS to create the queue or use the Shoryuken CLI.

```shell
shoryuken sqs create QUEUE-NAME
```

See `shoryuken sqs help create`.

## Install gem

```ruby
gem 'shoryuken'
```

**Note:** `aws-sdk-sqs` is a dependency of `shoryuken`. The full `aws-sdk` gem is not necessary.

## Configure your rails application to use Shoryuken

```ruby
# config/application.rb
module YourRailsApp
  class Application < Rails::Application
    config.active_job.queue_adapter = :shoryuken
  end
end
```

## Create your job

```ruby
# app/jobs/your_job.rb
class YourJob < ApplicationJob
  queue_as 'name_of_your_sqs_queue'

  def perform(resource_id)
    resource   = Resource.find(resource_id)
    # perform your task on resource ...
  end
end
```

Trigger your job where required:

```ruby
YourJob.perform_later(resource.id)
```


## Configure shoryuken.yml

Make sure it's located in `config/shoryuken.yml`. This will be used by the worker to consume and process the messages sent to the queue in SQS.

```yaml
concurrency: 15  # keep your database pool at least this large
queues:
  - [name_of_your_queue_in_sqs, 1]
  - [<%= ENV['OTHER_QUEUE'] %>, 1]
```

You can use [dotenv](https://github.com/bkeepers/dotenv) to setup your environment variables.

The number `1` above after the queue name is the queue weight, please refer to [Polling strategies
](https://github.com/ruby-shoryuken/shoryuken/wiki/Polling-strategies) for more information.

## Configure AWS credentials

See [[Configure the AWS Client]] and [[Amazon SQS IAM Policy for Shoryuken]].

## Run the following command to start consuming the SQS queue

```shell
bundle exec shoryuken -R -C config/shoryuken.yml
```