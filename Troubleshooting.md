# Troubleshooting

Common issues and solutions when running Shoryuken.

## Worker Issues

### "No worker found for queue"

**Cause:** Shoryuken can't find a worker class registered for the queue.

**Solutions:**

1. **For Rails with ActiveJob:** Use the `-R` flag:

```bash
bundle exec shoryuken -R -C config/shoryuken.yml
```

2. **For plain Ruby:** Use `-r` to require your workers:

```bash
bundle exec shoryuken -r ./app/workers -C config/shoryuken.yml
```

3. **Ensure queue names match:** The queue in `shoryuken.yml` must match the worker's queue:

```yaml
# config/shoryuken.yml
queues:
  - my_queue
```

```ruby
class MyWorker
  include Shoryuken::Worker
  shoryuken_options queue: 'my_queue'  # Must match
end
```

4. **Check Zeitwerk autoloading:** With Rails 7+, ensure workers are in `app/workers` or `app/jobs`.

---

### Workers Not Processing Messages

**Check these in order:**

1. **Is Shoryuken running?**
```bash
ps aux | grep shoryuken
```

2. **Are there messages in the queue?**
```bash
aws sqs get-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789/myqueue \
  --attribute-names ApproximateNumberOfMessagesVisible
```

3. **Is the worker registered?**
```bash
bundle exec shoryuken -R -C config/shoryuken.yml
# Look for "Registered workers" in startup output
```

4. **Check logs:**
```bash
tail -f log/shoryuken.log
```

---

### Jobs Running Multiple Times

**Causes:**
- Visibility timeout too short
- Job taking longer than visibility timeout
- Not deleting messages

**Solutions:**

1. **Increase visibility timeout** on the queue (AWS Console or CLI)

2. **Enable auto_visibility_timeout** for long jobs:
```ruby
shoryuken_options auto_visibility_timeout: true
```

3. **Ensure auto_delete is enabled:**
```ruby
shoryuken_options auto_delete: true
```

4. **Make jobs idempotent** - design jobs to handle duplicate execution safely

---

## AWS/SQS Issues

### AWS Credentials Not Found

```
Aws::Errors::MissingCredentialsError
```

**Solutions:**

1. Set environment variables:
```bash
export AWS_ACCESS_KEY_ID=xxx
export AWS_SECRET_ACCESS_KEY=xxx
export AWS_REGION=us-east-1
```

2. Configure `~/.aws/credentials`

3. Use IAM role (EC2/ECS/EKS)

See [[Configure the AWS Client]] for details.

---

### Access Denied

```
Aws::SQS::Errors::AccessDenied
```

**Solutions:**

1. Check IAM policy includes required permissions:
```json
{
  "Action": [
    "sqs:ReceiveMessage",
    "sqs:DeleteMessage",
    "sqs:GetQueueAttributes",
    "sqs:GetQueueUrl"
  ]
}
```

2. Verify queue policy allows access

3. Check resource ARN matches your queue

See [[Amazon SQS IAM Policy for Shoryuken]].

---

### Queue Not Found

```
Aws::SQS::Errors::NonExistentQueue
```

**Solutions:**

1. Verify queue exists:
```bash
aws sqs list-queues
```

2. Check queue name spelling in `shoryuken.yml`

3. Verify region is correct

4. Create the queue:
```bash
shoryuken sqs create my_queue
```

---

### Region Not Specified

```
Aws::Errors::MissingRegionError
```

**Solutions:**

1. Set environment variable:
```bash
export AWS_REGION=us-east-1
```

2. Configure in `~/.aws/config`

3. Set in code:
```ruby
Aws.config[:region] = 'us-east-1'
```

---

## Logging Issues

### Too Many "receive_message(...)" Log Messages

```
[Aws::SQS::Client 200 0.452184 0 retries] receive_message(...)
```

**Cause:** AWS SDK logging is set to debug level.

**Solution:** Set a specific log level for the SQS client:

```ruby
Shoryuken.configure_server do |config|
  config.sqs_client = Aws::SQS::Client.new(log_level: :info)
end
```

---

### No Log Output

**Solutions:**

1. Check log file path:
```bash
bundle exec shoryuken -R -C config/shoryuken.yml -L log/shoryuken.log
```

2. Configure logger level:
```ruby
Shoryuken.configure_server do |config|
  config.logger.level = Logger::DEBUG
end
```

3. For Rails, sync logger with Rails:
```ruby
Shoryuken.configure_server do |config|
  Rails.logger = Shoryuken::Logging.logger
end
```

---

## Connection Issues

### Database Connection Pool Exhausted

```
ActiveRecord::ConnectionTimeoutError: could not obtain a database connection
```

**Solution:** Increase pool size to match concurrency:

```yaml
# config/database.yml
production:
  pool: <%= ENV.fetch("SHORYUKEN_CONCURRENCY", 25) %>
```

Or reduce Shoryuken concurrency:

```yaml
# config/shoryuken.yml
concurrency: 10
```

---

### Redis Connection Issues (if using Redis)

**Solution:** Configure connection pool:

```ruby
Shoryuken.configure_server do |config|
  config.on(:startup) do
    Redis.current = Redis.new(url: ENV['REDIS_URL'])
  end
end
```

---

## Shutdown Issues

### Jobs Not Completing on Shutdown

**Cause:** Timeout too short.

**Solution:** Increase shutdown timeout:

```yaml
# config/shoryuken.yml
timeout: 60  # seconds
```

Or for Kubernetes:

```yaml
spec:
  terminationGracePeriodSeconds: 300
```

---

### Stuck on Shutdown

**Diagnosis:** Send TTIN signal to get thread dump:

```bash
kill -TTIN $(cat tmp/pids/shoryuken.pid)
```

Check logs for stuck threads.

**Solution:** Use graceful shutdown signal:

```bash
kill -USR1 $(cat tmp/pids/shoryuken.pid)
```

---

## LocalStack/Development Issues

### Connection Refused to LocalStack

```
Aws::SQS::Errors::InternalServerError: Connection refused
```

**Solutions:**

1. Verify LocalStack is running:
```bash
docker-compose ps
```

2. Check endpoint URL:
```ruby
Aws::SQS::Client.new(endpoint: 'http://localhost:4566')
```

3. Wait for LocalStack to be ready before starting Shoryuken

---

### MD5 Checksum Errors with Mock SQS

**Solution:** Disable checksum verification:

```ruby
Aws::SQS::Client.new(
  endpoint: 'http://localhost:4566',
  verify_checksums: false
)
```

---

## Debug Mode

Enable verbose logging:

```ruby
Shoryuken.configure_server do |config|
  config.logger.level = Logger::DEBUG
end
```

Or via environment:

```bash
SHORYUKEN_LOG_LEVEL=debug bundle exec shoryuken -R -C config/shoryuken.yml
```

---

## Getting Help

1. Check [GitHub Issues](https://github.com/phstc/shoryuken/issues)
2. Search existing issues before creating new ones
3. Include:
   - Shoryuken version
   - Ruby/Rails version
   - Full error message and backtrace
   - Relevant configuration

---

## Related

- [[Configure the AWS Client]] - AWS credential setup
- [[Amazon SQS IAM Policy for Shoryuken]] - IAM permissions
- [[Signals]] - Process control
- [[Logging]] - Log configuration
