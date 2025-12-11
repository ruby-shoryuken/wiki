There are a few ways to configure the AWS client:

* Ensure the `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` env vars are set.
* Create a `~/.aws/credentials` file.
* Set `Aws.config[:credentials]` or `Shoryuken.sqs_client = Aws::SQS::Client.new(...)` from Ruby code (e.g. in a Rails initializer)
* Use the Instance Profiles feature. The IAM role of the targeted machine must have an adequate SQS Policy.

You can read about these in more detail [here](http://docs.aws.amazon.com/sdkforruby/api/Aws/SQS/Client.html).
