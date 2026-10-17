If you need to set specific SQS [receive_message](https://docs.aws.amazon.com/sdk-for-ruby/v3/api/Aws/SQS/Client.html#receive_message-instance_method) options, such as `wait_time_seconds` ([[Long Polling]]), you can set them in an initializer as follows:

```ruby
Shoryuken.sqs_client_receive_message_opts = { 
  wait_time_seconds: 20, 
  max_number_of_messages: 1 
}
```

This configuration applies to the `'default'` group.

For other [groups](https://github.com/ruby-shoryuken/shoryuken/wiki/Processing-Groups), set them per group. Group names from `shoryuken.yml` are Symbols, so use a Symbol key:

```ruby
Shoryuken.sqs_client_receive_message_opts[:group_name] = { wait_time_seconds: 20 }
```

You can also set it per queue. A queue entry replaces the group entry; the two are not merged.

```ruby
Shoryuken.sqs_client_receive_message_opts['queue-name'] = { wait_time_seconds: 20 }
```

`max_number_of_messages` can only lower the batch size. Shoryuken fetches at most 10 messages, and 1 for a FIFO queue without `batch: true`. For a worker without `batch: true`, it also fetches no more than the free processors. Shoryuken always sets `attribute_names` and `message_attribute_names` to `All`.

See [API Reference/ReceiveMessage](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/APIReference/API_ReceiveMessage.html).