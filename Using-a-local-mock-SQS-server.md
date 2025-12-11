It is possible to configure Shoryuken to use a local mock SQS server implementation in place of Amazon's real SQS. This is useful, for example, when doing local development without an Internet connection.

This guide will help you to configure one such mock SQS service implementation, [`moto`](https://github.com/spulec/moto). We'll set up `aws-sdk-ruby` and `shoryuken` to use `moto` instead of the real SQS provided by AWS.

# Install and run `moto` stand alone server mode

[Install moto](https://github.com/spulec/moto#stand-alone-server-mode) with:

```
pip install moto[server]
```

To run it:

```
moto_server sqs -p 4576 -H localhost
```

# Upgrade `aws-sdk`

To use a local SQS server, you must be running `aws-sdk` version 2.2.15 or later. If your version of `aws-sdk` is older, update it.

```
bundle update aws-sdk
```

# Configure Shoryuken to use `moto`

We'll need to change the `:sqs_endpoint` key of Shoryuken's AWS config to point to your local `moto` instance. We'll also have to disable MD5 checks from `send_message*` calls in the AWS SDK.

## Configure Client

```ruby
Shoryuken.configure_client do |config|
  config.sqs_client = Aws::SQS::Client.new(
    region: ENV["AWS_REGION"],
    access_key_id: ENV["AWS_ACCESS_KEY_ID"],
    secret_access_key: ENV["AWS_SECRET_ACCESS_KEY"],
    endpoint: 'http://localhost:4576',
    verify_checksums: false
  )
end
```

## Configure Server

```ruby
Shoryuken.configure_server do |config|
  config.sqs_client = Aws::SQS::Client.new(
    region: ENV["AWS_REGION"],
    access_key_id: ENV["AWS_ACCESS_KEY_ID"],
    secret_access_key: ENV["AWS_SECRET_ACCESS_KEY"],
    endpoint: 'http://localhost:4576',
    verify_checksums: false
  )
end
```

## Running SQS commands locally

If you want to use `shoryuken sqs` command with a different endpoint you can pass an `--endpoint` option to the command, example:

```bash
shoryuken sqs ls --endpoint=http://localhost:4576
```

if you don't want to pass the `--endpoint` option to all `sqs` commands you can set an env var for it in the application `SHORYUKEN_SQS_ENDPOINT=http://localhost:4576`.