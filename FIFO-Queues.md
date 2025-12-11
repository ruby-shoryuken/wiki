If you have a queue configured as a [FIFO Queue](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/FIFO-queues.html), Shoryuken will auto detect it, and just work ✨ 

## How does that work?

If you send a message using [`perform_async`](https://github.com/phstc/shoryuken/wiki/Sending-a-message) or [ActiveJob](https://github.com/phstc/shoryuken/wiki/Rails-Integration-Active-Job), Shoryuken will automatically set the Message Group ID to `ShoryukenMessage` and the Message Deduplication ID to a SHA-256 hash based on the message body.

If you want to set specific values to Message Group ID and Message Deduplication ID, 
you should use `send_message` instead:

```ruby
MyWorker.perform_async(
  'body',
  message_group_id: 'id',
  message_deduplication_id: 'id'
)

Shoryuken::Client.queues('queue.fifo').send_message(
  message_body: 'body', 
  message_group_id: 'id', 
  message_deduplication_id: 'id'
)
```

*Note:* If you are fetching from a FIFO queue and not using the batch option for workers, the fetcher will fetch a maximum of one message at a time, in order to ensure that the messages for a particular Message Group ID are executed in FIFO order ([see here](https://github.com/phstc/shoryuken/blob/17e9df8d80d86485c1b80ef4465350a8c40fa90d/lib/shoryuken/fetcher.rb#L62-L71)). It is better to keep `concurrency > 1` in this case, so that your worker can service jobs from other Message Group IDs at the same time.

You might also consider using the `batch` option to fetch more messages at once, but process them in sequence. SQS will not release the next message for a particular Message Group ID unless the currently-received ones have been deleted. [See the AWS docs](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/FIFO-queues.html#FIFO-queues-understanding-logic) (section entitled "Retrying multiple times") for more info.

## Receiving messages

There's no special configuration needed to receive FIFO messages, you only need to configure the queue in the [same way you would if it was a standard queue](https://github.com/phstc/shoryuken/wiki/Shoryuken-options#queues).

### Process one by one

If you want to receive and process messages one by one for a Message Group, configure `max_number_of_messages: 1 
` as follows:

```ruby
Shoryuken.sqs_client_receive_message_opts = { 
  max_number_of_messages: 1 
}
```

See [[Receive Message options]].