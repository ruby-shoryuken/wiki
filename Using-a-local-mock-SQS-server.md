It is possible to configure Shoryuken to use a local mock SQS server implementation in place of Amazon's real SQS. This is useful, for example, when doing local development without an Internet connection.

This guide shows two such servers: [ElasticMQ](https://github.com/softwaremill/elasticmq), which Shoryuken's own integration tests use, and [`moto`](https://github.com/spulec/moto).

## ElasticMQ

Shoryuken's repository runs ElasticMQ with this `docker-compose.yml`:

```yaml
services:
  elasticmq:
    image: softwaremill/elasticmq-native:1.7.1
    container_name: elasticmq
    ports:
      - "9324:9324"
```

Point the SQS client at it. The credentials are placeholders, as in Shoryuken's test setup:

```ruby
sqs_client = Aws::SQS::Client.new(
  region: 'us-east-1',
  endpoint: 'http://localhost:9324',
  access_key_id: 'fake',
  secret_access_key: 'fake'
)

Shoryuken.configure_server { |config| config.sqs_client = sqs_client }
Shoryuken.configure_client { |config| config.sqs_client = sqs_client }
```

## moto

### Install and run `moto` stand alone server mode

[Install moto](https://github.com/spulec/moto#stand-alone-server-mode) with:

```
pip install moto[server]
```

To run it:

```
moto_server sqs -p 4576 -H localhost
```

### Configure Shoryuken to use `moto`

Set the `endpoint` of the SQS client to your local `moto` instance. We'll also have to disable MD5 checks from `send_message*` calls in the AWS SDK.

#### Configure Client

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

#### Configure Server

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