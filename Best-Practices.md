## Design your workers to be idempotent

SQS provides [at-least-once message delivery](http://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/standard-queues.html#standard-queues-at-least-once-delivery) by default, which means each job you published could end up being delivered more than once by SQS. That's why it is very important that you design your workers to be idempotent (they should be aware that the job might be worked on or already completed by another worker). Shoryuken can NOT handle this case because different workers don't share the knowledge across different jobs so there is no way to detect the job duplication at shoryuken's level. 

If idempotency is not an option in your application, have a look at [FIFO queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/FIFO-queues.html), they reduce the likelihood of duplicate jobs.

Some references on FIFO queues:

* [[FIFO Queues]]
* [Using Ruby and Amazon SQS FIFO Queues · Pablo Cantero](http://www.pablocantero.com/blog/2017/08/06/using-ruby-and-amazon-sqs-fifo-queues/)

## Keep the visibility timeout longer than your longest job

When a job runs longer than the queue's [visibility timeout](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/AboutVT.html), SQS makes the message visible again while the first run is still busy, and Shoryuken receives it again. The same message then runs in more than one thread at once. Each copy takes a thread and, with ActiveRecord, a database connection. When the database pool is smaller than [`concurrency`](https://github.com/ruby-shoryuken/shoryuken/wiki/Shoryuken-options#concurrency), the threads can need more connections than the pool holds, and ActiveRecord raises `could not obtain a connection from the pool`.

To keep the message hidden while the job runs, either:

* set the queue's `VisibilityTimeout` above the worst-case run time of the job, or
* enable [`auto_visibility_timeout`](https://github.com/ruby-shoryuken/shoryuken/wiki/Worker-options#auto_visibility_timeout) in the worker's `shoryuken_options`, so that Shoryuken extends the timeout while the job runs. This is for Shoryuken workers only: it does not apply to batch workers, and ActiveJob jobs have no `shoryuken_options`, so for them use the first fix.

To avoid the error, keep the database pool at least as large as `concurrency`.
