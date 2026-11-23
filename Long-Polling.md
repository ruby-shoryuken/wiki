By default Shoryuken does not set `wait_time_seconds`, so SQS uses the queue's `ReceiveMessageWaitTimeSeconds` attribute. A `wait_time_seconds` that you set replaces that attribute. To use [Long Polling](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-short-and-long-polling.html) for the `default` group, configure it as follows:

```ruby
# config/initializers/shoryuken.rb
Shoryuken.sqs_client_receive_message_opts = { wait_time_seconds: 20 }
```

For a group, see [Long polling per group](https://github.com/ruby-shoryuken/shoryuken/wiki/Processing-Groups#long-polling-per-group).

If you are not using Rails `--rails`, which auto loads initializers, just add that configuration to the file you require `--require` for starting Shoryuken.

## Idle queues

Use long polling for a queue that is often empty. The SQS default for `ReceiveMessageWaitTimeSeconds` is 0 and Shoryuken's [`delay`](https://github.com/ruby-shoryuken/shoryuken/wiki/Shoryuken-options#delay) defaults to 0. With both at 0, an idle group calls `ReceiveMessage` again as soon as the previous call returns: about one call per network round trip. SQS bills each call as a request (see [Amazon SQS pricing](https://aws.amazon.com/sqs/pricing/)).

In a test against ElasticMQ with a simulated round trip of 20 ms, one idle queue got 43.6 `ReceiveMessage` calls per second without long polling (about 113 million in 30 days) and 0.07 calls per second with `wait_time_seconds: 20`.

A long poll returns as soon as a message arrives, so in a group with one queue it does not delay messages.

## Groups with more than one queue

A group fetches its queues one after another, so a long poll on an empty queue blocks the other queues of the group for up to `wait_time_seconds`. In the same test, a busy queue and an empty queue in one group with `wait_time_seconds: 20` processed 10 of 500 messages in 15 seconds. In two groups, they processed all 500.

Put an idle queue in its own group and set long polling for that group only. See [[Processing Groups]].

## Shutdown

A graceful stop (`USR1`, `TSTP`) first waits for the running `ReceiveMessage` call, so with long polling it can take up to `wait_time_seconds` in addition to the `timeout` for busy workers. See [[Signals]].

See [[Receive Message options]].