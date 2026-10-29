There are a few ways to configure the AWS client:

* Ensure the `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` env vars are set.
* Create a `~/.aws/credentials` file.
* Set `Aws.config[:credentials]` or `Shoryuken.sqs_client = Aws::SQS::Client.new(...)` from Ruby code (e.g. in a Rails initializer)
* Use the Instance Profiles feature. The IAM role of the targeted machine must have an adequate SQS Policy.

You can read about these in more detail [here](https://docs.aws.amazon.com/sdk-for-ruby/v3/api/Aws/SQS/Client.html).

## Setting the SQS client in Ruby

`configure_server` runs only in the Shoryuken process and `configure_client` only outside it, so set the client in both when your web process also enqueues messages:

```ruby
# config/initializers/shoryuken.rb
sqs_client = Aws::SQS::Client.new(region: 'us-east-1')

Shoryuken.configure_server do |config|
  config.sqs_client = sqs_client
end

Shoryuken.configure_client do |config|
  config.sqs_client = sqs_client
end
```

Without `sqs_client`, Shoryuken builds `Aws::SQS::Client.new` with the options under `aws` in the configuration file (see [[Shoryuken options]]).

## Local SQS server

See [[Using a local mock SQS server]].
