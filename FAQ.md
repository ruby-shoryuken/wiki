Short answers to common questions. Each answer links to the page with the details.

## Why is the same message processed again while the first run is still busy?

The job runs longer than the queue's visibility timeout. SQS then makes the message visible again, and Shoryuken runs it again in another thread while the first run is still busy. See [Keep the visibility timeout longer than your longest job](https://github.com/ruby-shoryuken/shoryuken/wiki/Best-Practices#keep-the-visibility-timeout-longer-than-your-longest-job).

To keep the message hidden while the job runs, do one of these:

- Set the queue's `VisibilityTimeout` above the worst-case run time of the job.
- Set [`auto_visibility_timeout: true`](https://github.com/ruby-shoryuken/shoryuken/wiki/Worker-options#auto_visibility_timeout) in the worker's `shoryuken_options`, so that Shoryuken extends the timeout while the job runs. It does not apply to `batch: true` workers or to ActiveJob jobs: for them, use the first fix.

If ActiveRecord also raises `could not obtain a connection from the pool`, the database pool is smaller than the threads that use it. See the note under [`concurrency`](https://github.com/ruby-shoryuken/shoryuken/wiki/Shoryuken-options#concurrency).

SQS can also deliver a message more than once for other reasons, so make the job safe to run twice. See [Design your workers to be idempotent](https://github.com/ruby-shoryuken/shoryuken/wiki/Best-Practices#design-your-workers-to-be-idempotent).

## How do I retry one failed message by hand?

SQS has no call that retries one message. Store the message's body and message attributes, and send them again as a new message when you want to retry it. [Retrying a message by hand](https://github.com/ruby-shoryuken/shoryuken/wiki/Retrying-a-message#retrying-a-message-by-hand) shows the example and what the new message needs: the `shoryuken_class` attribute, and on a FIFO queue the original group ID and a new deduplication ID.

The original message stays in the queue unless something deletes it, so it also runs again when its visibility timeout ends. To retry it only by hand, call `sqs_msg.delete` after you store it.
