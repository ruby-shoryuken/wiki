By default Shoryuken does not set `wait_time_seconds`, so SQS uses the queue's `ReceiveMessageWaitTimeSeconds` attribute. To use [Long Polling](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-short-and-long-polling.html) for every queue, configure it as follows:

```ruby
# config/initializers/shoryuken.rb
Shoryuken.sqs_client_receive_message_opts = { wait_time_seconds: 20 }
```

For a group, see [Long polling per group](https://github.com/ruby-shoryuken/shoryuken/wiki/Processing-Groups#long-polling-per-group).

If you are not using Rails `--rails`, which auto loads initializers, just add that configuration to the file you require `--require` for starting Shoryuken.

See [[Receive Message options]].