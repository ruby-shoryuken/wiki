You can tell Shoryuken to load your Rails application by passing the `-R` or `--rails` flag to the `shoryuken` command.

If you load Rails, assuming your workers are located in the `app/jobs` directory, they will be auto-loaded. This means you don't need to require them explicitly with `-r`.

For middleware and other configurations, you might want to create an initializer:

```ruby
# config/initializers/shoryuken.rb

Shoryuken.configure_server do |config|
  # Replace Rails logger so messages are logged wherever Shoryuken is logging
  # Note: this entire block is only run by the processor, so we don't overwrite
  #       the logger when the app is running as usual.

  Rails.logger = Shoryuken::Logging.logger
  Rails.logger.level = Rails.application.config.log_level

  # config.server_middleware do |chain|
  #  chain.add Shoryuken::MyMiddleware
  # end

  # For dynamically adding queues prefixed by Rails.env 
  # Shoryuken.add_group('default', 25)
  # %w(queue1 queue2).each do |name|
  #   Shoryuken.add_queue("#{Rails.env}_#{name}", 1, 'default')
  # end
end
```

*Note:* In the above case, since we are replacing the Rails logger, it's desired that this initializer runs before other initializers (in case they themselves use the logger). Since, by Rails conventions, initializers are executed in alphabetical order, this can be achieved by prepending the initializer filename with `00_` (assuming no other initializers alphabetically precede this one).

### Active Job Support

Yes, Shoryuken supports [Active Job](http://edgeguides.rubyonrails.org/active_job_basics.html)! This means that you can put your jobs in processor-agnostic `ActiveJob::Base` subclasses, and change processors whenever you want (or better yet, switch to Shoryuken from another processor easily!).

It works as expected once your [queuing backend is set](http://edgeguides.rubyonrails.org/active_job_basics.html#setting-the-backend) to `:shoryuken`. Just put your job in `app/jobs`. Here's an example:

```ruby
# app/jobs/process_photo_job.rb
class ProcessPhotoJob < ActiveJob::Base
  queue_as :default

  rescue_from ActiveJob::DeserializationError do |ex|
    Shoryuken.logger.error ex
    Shoryuken.logger.error ex.backtrace.join("\n")
  end

  def perform(photo)
    photo.process_image!
  end
end
```

*Note:* When queueing jobs to be performed in the future (e.g. when setting the `wait` or `wait_until` Active Job options), SQS limits the amount of time to 15 minutes (see [`delay_seconds`](https://docs.aws.amazon.com/sdkforruby/api/Aws/SQS/Client.html#send_message-instance_method)). Shoryuken will raise an exception if you attempt to schedule a job further into the future than this limit.


#### Active Job Queue Name Support

*Note:* Active Job allows you to [prefix the queue names](http://edgeguides.rubyonrails.org/active_job_basics.html#queues) of all jobs. Shoryuken supports this behavior natively. By default, though, queue names defined in the config file (or passed to the CLI), are not prefixed similarly. To have Shoryuken honor Active Job prefixes, you must enable that option explicitly. A good place to do that in Rails is in an initializer:

```ruby
# config/initializers/shoryuken.rb
Shoryuken.active_job_queue_name_prefixing = true
```


#### Active Job Queue Adapters

Shoryuken has two queue adapters, the default uses a synchronous producer, that is set up normally in your rails config as:

```ruby
# config/application.rb
config.active_job.queue_adapter = :shoryuken
```

There is also a second option to use an async producer:

```ruby
# config/application.rb
config.active_job.queue_adapter = ActiveJob::QueueAdapters::ShoryukenConcurrentSendAdapter.new
```

If you have a lot of jobs to produce, consider using the `ShoryukenConcurrentSendAdapter`. Another hint for quick job enqueuing for `ActiveRecord` objects is to use a query that only selects the `id`:

```ruby
Rails.application.config.active_job.queue_adapter = ActiveJob::QueueAdapters::ShoryukenConcurrentSendAdapter.new
MyModel.expensive_scope.select(:id).load.each { |obj| MyJob.perform_later(obj) }
```

#### Set SQS SendMessage parameters with `.set`

```ruby
MyJob.set(message_group_id: 'abc123',
          message_deduplication_id: 'dedupe',
          message_attributes: {
            'key' => {
              string_value: 'myvalue',
              data_type: 'String'
            }
          })
     .perform_later(123, 'abc')
```

### How to use [`retry_on`](https://edgeguides.rubyonrails.org/active_job_basics.html#retrying-or-discarding-failed-jobs)

Please follow the steps detailed [here](https://github.com/phstc/shoryuken/issues/553#issuecomment-465813111) to make `retry_on` work with Shoryuken.