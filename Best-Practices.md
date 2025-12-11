# 

## Design your workers to be idempotent

SQS provides [at-least-once message delivery](http://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/standard-queues.html#standard-queues-at-least-once-delivery) by default, which means each job you published could end up being delivered more than once by SQS. That's why it is very important that you design your workers to be idempotent (they should be aware that the job might be worked on or already completed by another worker). Shoryuken can NOT handle this case because different workers don't share the knowledge across different jobs so there is no way to detect the job duplication at shoryuken's level. 

If idempotency is not an option in your application, have a look at [FIFO queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/FIFO-queues.html), these calls may be used to reduce the likelihood of duplicate jobs.

Some references on FIFO queues:

* [FIFO Queues · phstc/shoryuken Wiki · GitHub](https://github.com/phstc/shoryuken/wiki/FIFO-Queues)
* [Using Ruby and Amazon SQS FIFO Queues · Pablo Cantero](http://www.pablocantero.com/blog/2017/08/06/using-ruby-and-amazon-sqs-fifo-queues/)