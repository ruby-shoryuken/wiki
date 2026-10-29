Lines like this one come from the AWS SDK, not from Shoryuken:

```
[Aws::SQS::Client 200 0.452184 0 retries] receive_message(max_number_of_messages:1,message_attribute_names:["All"],attribute_names:["All"],queue_url:"https://sqs.us-east-1.amazonaws.com/000011112222/my-queue")
```

The AWS SDK for Ruby (v3) logs every request when its client has a `logger`. Your application or another gem most likely set one, for example for all clients with `Aws.config[:logger]`. The SDK writes these lines at the client's `log_level`, which is `:info` by default.

To remove them from the Shoryuken process, give Shoryuken its own SQS client without a logger. Options passed to the client constructor take priority over `Aws.config`:

```ruby
Shoryuken.configure_server do |config|
  # https://docs.aws.amazon.com/sdk-for-ruby/v3/api/Aws/SQS/Client.html#initialize-instance_method
  config.sqs_client = Aws::SQS::Client.new(logger: nil)
end
```

To keep the lines only for debugging, keep the logger and set `log_level: :debug` on the client instead. The lines then appear only when that logger's level is `DEBUG`.

See [[Configure the AWS Client]] for the other client options and [[Logging]] for the Shoryuken logger.
