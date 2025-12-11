```
[Aws::SQS::Client 200 0.452184 0 retries] receive_message(max_number_of_messages:1,message_attribute_names:["All"],attribute_names:["All"],queue_url:"https://sqs.us-east-1.amazonaws.com/000011112222/my-queue-Queue-K1AB9G02OU8S")
```

Those messages are not because of Shoryuken, it's most likely you or another gem (see #353) enabled `AWS.config(log_level: :debug)`.


To get rid of them, try to set a specific SQS Client for Shoryuken:

```ruby
Shoryuken.configure_server do |config|
  # https://docs.aws.amazon.com/sdkforruby/api/Aws/SQS/Client.html#initialize-instance_method
  config.sqs_client = Aws::SQS::Client.new(log_level: :info)
end
```