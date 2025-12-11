Shoryuken does not direct support recurring jobs, but you can use Lambda or even CloudWatch for automating that, both can work in combination with SQS.

For some guidance:

- https://docs.aws.amazon.com/lambda/latest/dg/with-scheduled-events.html
- https://aws.amazon.com/pt/about-aws/whats-new/2016/03/cloudwatch-events-now-supports-amazon-sqs-queue-targets/

In case you want to discuss some alternatives, please follow up in [this thread](https://github.com/phstc/shoryuken/issues/251#issuecomment-271660396).