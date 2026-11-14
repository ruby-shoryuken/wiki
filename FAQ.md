Short answers to common questions. Each answer links to the page with the details.

## Why is the same message processed again while the first run is still busy?

The job runs longer than the message's visibility timeout. SQS then makes the message visible again, and Shoryuken runs it again in another thread while the first run is still busy. See [Keep the visibility timeout longer than your longest job](https://github.com/ruby-shoryuken/shoryuken/wiki/Best-Practices#keep-the-visibility-timeout-longer-than-your-longest-job).

To keep the message hidden while the job runs, do one of these:

- Set the visibility timeout above the worst-case run time of the job. That is the queue's `VisibilityTimeout`, or the `visibility_timeout` in [[Receive Message options]] when you set one there for the queue or its group.
- Set [`auto_visibility_timeout: true`](https://github.com/ruby-shoryuken/shoryuken/wiki/Worker-options#auto_visibility_timeout) in the worker's `shoryuken_options`, so that Shoryuken extends the timeout while the job runs. It does not apply to `batch: true` workers or to ActiveJob jobs: for them, use the first fix.

If ActiveRecord also raises `could not obtain a connection from the pool`, the database pool is smaller than the threads that use it. See the note under [`concurrency`](https://github.com/ruby-shoryuken/shoryuken/wiki/Shoryuken-options#concurrency).

SQS can also deliver a message more than once for other reasons, so make the job safe to run twice. See [Design your workers to be idempotent](https://github.com/ruby-shoryuken/shoryuken/wiki/Best-Practices#design-your-workers-to-be-idempotent).

## How do I retry one failed message by hand?

SQS has no call that retries one message. Store the message's body and message attributes, and send them again as a new message when you want to retry it. [Retrying a message by hand](https://github.com/ruby-shoryuken/shoryuken/wiki/Retrying-a-message#retrying-a-message-by-hand) shows the example and what to pass: the message attributes, and on a FIFO queue the original group ID and a new deduplication ID.

The original message stays in the queue unless something deletes it, so it also runs again when its visibility timeout ends. To retry it only by hand, call `sqs_msg.delete` after you store it.

## Why does the first `perform_later` to a queue take seconds?

The first enqueue to a queue does setup work that later calls skip:

- When the app does not set `sqs_client`, the first SQS call builds the client with `Aws::SQS::Client.new`, and the AWS SDK looks up the credentials. When no earlier source in its credential chain has credentials, it tries the EC2 instance metadata service last. Outside EC2, that request waits for its timeout. Set the credentials explicitly: see [[Configure the AWS Client]].
- Shoryuken looks up the queue URL by name (`GetQueueUrl`) and the queue attributes (`GetQueueAttributes`). It caches the queue object per process.

To move this work to boot, call `Shoryuken::Client.queues('my_queue')` in an initializer. After the first call, each `perform_later` sends one `SendMessage` request, so if later calls are also slow, check the network path to SQS.

## Why does enqueuing fail with "The specified queue default does not exist"?

Shoryuken's Active Job adapter sends the job to the queue named by the job's `queue_name`. A job class without `queue_as` uses Active Job's default queue name, `default` (unless you change `config.active_job.default_queue_name`). A mailer's `deliver_later` also uses it when `config.action_mailer.deliver_later_queue_name` is `nil`, which `load_defaults "6.1"` and later set. So the name in the error is the queue that the failing job asked for, not always the queue you expect.

Find the job class that was enqueued, then set its queue with `queue_as` (for mailers, `deliver_later_queue_name`), or create an SQS queue with that name. Check the AWS permissions only when the name in the error is the right one. See [[Rails Integration Active Job]].

## Should I poll SQS from a job in another job system, or run Shoryuken?

Run Shoryuken as its own process: it polls the queues and runs your workers in threads.

```shell
bundle exec shoryuken -R -C config/shoryuken.yml
```

- A worker gets every message on its queue, also from producers that do not use Shoryuken. Without the `shoryuken_class` attribute, Shoryuken uses the worker registered for the queue. Use [`body_parser`](https://github.com/ruby-shoryuken/shoryuken/wiki/Worker-options#body_parser) to parse the body.
- Shoryuken does not set `wait_time_seconds` by default. To use long polling, see [[Long Polling]].
- To scale, set [`concurrency`](https://github.com/ruby-shoryuken/shoryuken/wiki/Shoryuken-options#concurrency) and run more processes. To poll queues independently of each other, use [[Processing Groups]].
- To run code around each message, use [[Middleware]].
