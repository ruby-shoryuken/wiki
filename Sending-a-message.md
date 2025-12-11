There are a couple ways to send a message using Shoryuken.

### Using `perform_async`: 

```ruby
MyWorker.perform_async('Pablo')
``` 

it also accepts a Hash as a parameter, which is automatically converted into JSON:

```ruby
MyWorker.perform_async(field: 'test', other_field: 'other')
```

it's also possible to override where the job is queued to with an options hash:

```ruby
MyWorker.perform_async('Pablo')                     # will queue to the default queue
MyWorker.perform_async('Pablo', queue: 'important') # will queue to the 'important' queue
```

### Or using the queue object directly , which wraps [Aws::SQS::Client](https://docs.aws.amazon.com/sdkforruby/api/Aws/SQS/Client.html):

```ruby
# To send a single message
Shoryuken::Client.queues('default').send_message('msg 1')

# To send a single message as a Hash you need to put it in a hash with `message_body` as the key.
Shoryuken::Client.queues('default').send_message(message_body: { example: "data" })

# To send multiple messages
Shoryuken::Client.queues('default').send_messages(['msg 1', 'msg 2'])
```
#### Sending a message directly for ActiveJob workers
In the case that the job message is generated outside of your Rails Shoryuken via ActiveJob environment (like from a serverless function), you can still send a message to be processed by your workers.
```ruby
queue_name = 'my-queue'

job_args = {
  "job_class": "MyActiveJob",
  "job_id": SecureRandom.uuid,
  "provider_job_id": nil,
  "queue_name": queue_name,
  "priority": nil,
  "arguments": [
    {
      "arg1": 'arg1 value',
      "arg2": 'arg2 value',
      "arg3": 'arg3 value',
      "_aj_symbol_keys": [
        'arg1',
        'arg2',
        'arg3',
      ]
    }
  ],
  "executions": 0,
  "locale": "en"
}

message = { 
  message_body: job_args,
  message_attributes: {
    shoryuken_class: {
      string_value: 'ActiveJob::QueueAdapters::ShoryukenAdapter::JobWrapper',
      data_type: "String"
    }
  }
}

Shoryuken::Client.queues(queue_name).send_message(message)
```

## Delaying a message

You can delay a message up to 15 minutes.

```ruby
MyWorker.perform_in(60, 'Pablo') # 60 seconds
MyWorker.perform_at(Time.now, 'Pablo')
```

## AWS credentials

Check [AWS credentials](https://docs.aws.amazon.com/sdk-for-ruby/v3/developer-guide/setup-config.html) to make sure you have your aws-sdk before sending messages.