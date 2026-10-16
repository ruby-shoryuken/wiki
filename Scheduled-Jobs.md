Shoryuken does not support recurring jobs directly. You can use Lambda or CloudWatch Events to send messages to SQS on a schedule.

For some guidance:

- https://docs.aws.amazon.com/lambda/latest/dg/with-scheduled-events.html
- https://aws.amazon.com/about-aws/whats-new/2016/03/cloudwatch-events-now-supports-amazon-sqs-queue-targets/

In case you want to discuss some alternatives, please follow up in [this thread](https://github.com/ruby-shoryuken/shoryuken/issues/251#issuecomment-271660396).