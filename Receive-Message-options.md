If you need to set specific SQS [receive_message](https://docs.aws.amazon.com/sdkforruby/api/Aws/SQS/Client.html#receive_message-instance_method) options, such as `wait_time_seconds` ([[Long Polling]]), you can set them in an initializer as follows:

```ruby
Shoryuken.sqs_client_receive_message_opts = { 
  wait_time_seconds: 20, 
  max_number_of_messages: 1 
}
```

this configuration applies to the 'default' group.

For other [groups](https://github.com/phstc/shoryuken/wiki/Processing-Groups) you need to set them per group.
```ruby
Shoryuken.sqs_client_receive_message_opts['group-name'] = { wait_time_seconds: 20 }
```

You can also set it per queue.
```ruby
Shoryuken.sqs_client_receive_message_opts['queue-name'] = { wait_time_seconds: 20 }

```

See [API Reference/ReceiveMessage](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/APIReference/API_ReceiveMessage.html).