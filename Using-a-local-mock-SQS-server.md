# Using a Local Mock SQS Server

Configure Shoryuken to use a local SQS mock for development and testing without connecting to AWS.

## Options

| Service | Type | Best For |
|---------|------|----------|
| **LocalStack** | Full AWS mock | CI/CD, comprehensive testing |
| **ElasticMQ** | SQS-only | Lightweight, fast |
| **Moto** | Python AWS mock | Python projects, simple setup |

---

## LocalStack (Recommended)

LocalStack is the recommended option and is used in Shoryuken's own CI.

### Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  localstack:
    image: localstack/localstack:latest
    ports:
      - "4566:4566"
    environment:
      - SERVICES=sqs
      - DEBUG=0
      - DOCKER_HOST=unix:///var/run/docker.sock
    volumes:
      - "./localstack-data:/var/lib/localstack"
      - "/var/run/docker.sock:/var/run/docker.sock"
```

Start LocalStack:

```bash
docker-compose up -d localstack
```

### Create Queues

```bash
# Using AWS CLI
aws --endpoint-url=http://localhost:4566 sqs create-queue --queue-name myapp-default
aws --endpoint-url=http://localhost:4566 sqs create-queue --queue-name myapp-critical

# Or using awslocal (LocalStack CLI)
awslocal sqs create-queue --queue-name myapp-default
```

### Configure Shoryuken

```ruby
# config/initializers/shoryuken.rb
if Rails.env.development? || Rails.env.test?
  sqs_client = Aws::SQS::Client.new(
    endpoint: ENV.fetch('SQS_ENDPOINT', 'http://localhost:4566'),
    region: 'us-east-1',
    access_key_id: 'test',
    secret_access_key: 'test'
  )

  Shoryuken.configure_client do |config|
    config.sqs_client = sqs_client
  end

  Shoryuken.configure_server do |config|
    config.sqs_client = sqs_client
  end
end
```

### Startup Script

```bash
#!/bin/bash
# scripts/setup_localstack.sh

echo "Waiting for LocalStack..."
until aws --endpoint-url=http://localhost:4566 sqs list-queues 2>/dev/null; do
  sleep 1
done

echo "Creating queues..."
aws --endpoint-url=http://localhost:4566 sqs create-queue --queue-name myapp-default
aws --endpoint-url=http://localhost:4566 sqs create-queue --queue-name myapp-critical

echo "Queues ready!"
```

---

## ElasticMQ

ElasticMQ is a lightweight, in-memory SQS-compatible server.

### Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  elasticmq:
    image: softwaremill/elasticmq-native:latest
    ports:
      - "9324:9324"
      - "9325:9325"  # UI
    volumes:
      - "./elasticmq.conf:/opt/elasticmq.conf"
```

### Configuration File

```hocon
# elasticmq.conf
include classpath("application.conf")

node-address {
  protocol = http
  host = localhost
  port = 9324
  context-path = ""
}

rest-sqs {
  enabled = true
  bind-port = 9324
  bind-hostname = "0.0.0.0"
}

queues {
  myapp-default {
    defaultVisibilityTimeout = 30 seconds
    delay = 0 seconds
    receiveMessageWait = 0 seconds
  }
  myapp-critical {
    defaultVisibilityTimeout = 30 seconds
    delay = 0 seconds
    receiveMessageWait = 0 seconds
  }
}
```

### Configure Shoryuken

```ruby
# config/initializers/shoryuken.rb
if Rails.env.development?
  sqs_client = Aws::SQS::Client.new(
    endpoint: 'http://localhost:9324',
    region: 'us-east-1',
    access_key_id: 'test',
    secret_access_key: 'test'
  )

  Shoryuken.configure_client do |config|
    config.sqs_client = sqs_client
  end

  Shoryuken.configure_server do |config|
    config.sqs_client = sqs_client
  end
end
```

---

## Moto

Moto is a Python-based AWS mock library with standalone server mode.

### Install and Run

```bash
# Install
pip install moto[server]

# Run
moto_server sqs -p 4576 -H localhost
```

### Docker

```yaml
# docker-compose.yml
services:
  moto:
    image: motoserver/moto:latest
    ports:
      - "4576:4576"
    command: ["-p", "4576"]
```

### Configure Shoryuken

```ruby
# config/initializers/shoryuken.rb
if Rails.env.development?
  sqs_client = Aws::SQS::Client.new(
    endpoint: 'http://localhost:4576',
    region: 'us-east-1',
    access_key_id: 'test',
    secret_access_key: 'test',
    verify_checksums: false
  )

  Shoryuken.configure_client do |config|
    config.sqs_client = sqs_client
  end

  Shoryuken.configure_server do |config|
    config.sqs_client = sqs_client
  end
end
```

---

## Using Shoryuken CLI

### With Environment Variable

```bash
export SHORYUKEN_SQS_ENDPOINT=http://localhost:4566
shoryuken sqs ls
shoryuken sqs create myapp-default
```

### With CLI Option

```bash
shoryuken sqs ls --endpoint=http://localhost:4566
shoryuken sqs create myapp-default --endpoint=http://localhost:4566
```

---

## Development Workflow

### Procfile.dev

```
web: bundle exec rails server
worker: bundle exec shoryuken -R -C config/shoryuken.yml
localstack: docker-compose up localstack
```

### Start Everything

```bash
# Using Foreman
foreman start -f Procfile.dev

# Or separately
docker-compose up -d localstack
bundle exec shoryuken -R -C config/shoryuken.yml
```

---

## Troubleshooting

### Connection Refused

```
Aws::SQS::Errors::InternalServerError: Connection refused
```

**Solution:** Ensure LocalStack/ElasticMQ is running:

```bash
docker-compose ps
docker-compose logs localstack
```

### Queue Not Found

```
Aws::SQS::Errors::NonExistentQueue
```

**Solution:** Create queues before starting workers:

```bash
aws --endpoint-url=http://localhost:4566 sqs create-queue --queue-name myapp-default
```

### MD5 Checksum Errors

Some mock servers don't compute MD5 correctly. Disable verification:

```ruby
Aws::SQS::Client.new(
  endpoint: 'http://localhost:4566',
  verify_checksums: false
)
```

---

## CI Integration

### GitHub Actions

```yaml
# .github/workflows/test.yml
services:
  localstack:
    image: localstack/localstack
    ports:
      - 4566:4566
    env:
      SERVICES: sqs

steps:
  - name: Setup queues
    run: |
      aws --endpoint-url=http://localhost:4566 sqs create-queue --queue-name test-queue
    env:
      AWS_ACCESS_KEY_ID: test
      AWS_SECRET_ACCESS_KEY: test
      AWS_DEFAULT_REGION: us-east-1

  - name: Run tests
    run: bundle exec rspec
    env:
      SQS_ENDPOINT: http://localhost:4566
```

### GitLab CI

```yaml
# .gitlab-ci.yml
test:
  services:
    - name: localstack/localstack
      alias: localstack
  variables:
    AWS_ACCESS_KEY_ID: test
    AWS_SECRET_ACCESS_KEY: test
    AWS_DEFAULT_REGION: us-east-1
    SQS_ENDPOINT: http://localstack:4566
  script:
    - aws --endpoint-url=$SQS_ENDPOINT sqs create-queue --queue-name test-queue
    - bundle exec rspec
```

---

## Related

- [[Testing]] - Testing strategies
- [[Configure the AWS Client]] - AWS client configuration
- [[Getting Started]] - Initial setup
