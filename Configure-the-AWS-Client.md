There are a few ways to configure the AWS client:

* Ensure the `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` env vars are set.
* Create a `~/.aws/credentials` file.
* Set `Aws.config[:credentials]` or `Shoryuken.sqs_client = Aws::SQS::Client.new(...)` from Ruby code (e.g. in a Rails initializer)
* Use the Instance Profiles feature. The IAM role of the targeted machine must have an adequate SQS Policy.

You can read about these in more detail [here](http://docs.aws.amazon.com/sdkforruby/api/Aws/SQS/Client.html).

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

To use ElasticMQ, LocalStack or another SQS-compatible server, pass its `endpoint` to `Aws::SQS::Client.new`. See [[Using a local mock SQS server]].

The `shoryuken sqs` CLI commands take the endpoint with `-e`/`--endpoint`, or from the `SHORYUKEN_SQS_ENDPOINT` env var:

```shell
SHORYUKEN_SQS_ENDPOINT=http://localhost:9324 bundle exec shoryuken sqs ls
```
