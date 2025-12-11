# Security Best Practices

This guide covers security considerations for Shoryuken deployments with AWS SQS.

## IAM Permissions

### Principle of Least Privilege

Grant only the permissions required for your use case:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage",
        "sqs:DeleteMessageBatch",
        "sqs:GetQueueAttributes",
        "sqs:GetQueueUrl",
        "sqs:ChangeMessageVisibility",
        "sqs:ChangeMessageVisibilityBatch"
      ],
      "Resource": "arn:aws:sqs:us-east-1:123456789:myapp-*"
    }
  ]
}
```

### Separate Producer and Consumer Roles

**Consumer (Worker) Role:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage",
        "sqs:DeleteMessageBatch",
        "sqs:GetQueueAttributes",
        "sqs:GetQueueUrl",
        "sqs:ChangeMessageVisibility"
      ],
      "Resource": "arn:aws:sqs:*:*:myapp-*"
    }
  ]
}
```

**Producer (Web App) Role:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "sqs:SendMessage",
        "sqs:SendMessageBatch",
        "sqs:GetQueueUrl"
      ],
      "Resource": "arn:aws:sqs:*:*:myapp-*"
    }
  ]
}
```

See [[Amazon SQS IAM Policy for Shoryuken]] for detailed policy examples.

---

## Credential Management

### Prefer IAM Roles Over Access Keys

| Environment | Recommended Method |
|-------------|-------------------|
| EC2 | Instance Profile |
| ECS | Task Role |
| EKS | IRSA (IAM Roles for Service Accounts) |
| Lambda | Execution Role |
| Local Dev | AWS SSO or temporary credentials |

### Never Hardcode Credentials

```ruby
# BAD - Never do this
Aws.config.update(
  access_key_id: 'AKIAXXXXXXXX',
  secret_access_key: 'xxxxx'
)

# GOOD - Use environment or IAM roles
# Credentials are automatically loaded from:
# 1. Environment variables
# 2. Shared credentials file (~/.aws/credentials)
# 3. IAM instance profile/task role
```

### Rotate Access Keys Regularly

If you must use access keys:
1. Create new key pair
2. Update application configuration
3. Deploy and verify
4. Deactivate old key
5. Delete old key after grace period

---

## Queue Security

### Server-Side Encryption (SSE)

Enable SSE-SQS or SSE-KMS for encryption at rest:

```json
{
  "QueueName": "myapp-jobs",
  "Attributes": {
    "SqsManagedSseEnabled": "true"
  }
}
```

For SSE-KMS (customer-managed key):

```json
{
  "Attributes": {
    "KmsMasterKeyId": "alias/myapp-sqs-key"
  }
}
```

**IAM permissions for KMS:**
```json
{
  "Effect": "Allow",
  "Action": [
    "kms:Decrypt",
    "kms:GenerateDataKey"
  ],
  "Resource": "arn:aws:kms:us-east-1:123456789:key/xxx"
}
```

### VPC Endpoints

Use VPC endpoints to keep traffic within AWS:

```hcl
# Terraform example
resource "aws_vpc_endpoint" "sqs" {
  vpc_id            = aws_vpc.main.id
  service_name      = "com.amazonaws.us-east-1.sqs"
  vpc_endpoint_type = "Interface"

  security_group_ids = [aws_security_group.sqs_endpoint.id]
  subnet_ids         = aws_subnet.private[*].id

  private_dns_enabled = true
}
```

### Queue Policies

Restrict queue access by account/role:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789:role/myapp-worker"
      },
      "Action": [
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage"
      ],
      "Resource": "arn:aws:sqs:us-east-1:123456789:myapp-jobs"
    }
  ]
}
```

---

## Message Security

### Sensitive Data in Messages

**Avoid storing sensitive data directly in messages:**

```ruby
# BAD - Sensitive data in message
UserNotificationJob.perform_later(
  email: user.email,
  ssn: user.ssn  # Never include PII
)

# GOOD - Reference by ID
UserNotificationJob.perform_later(user_id: user.id)
```

### Encryption for Sensitive Payloads

If you must include sensitive data, encrypt it:

```ruby
class SecureJob < ApplicationJob
  def perform(encrypted_data)
    data = decrypt(encrypted_data)
    # Process data
  end

  private

  def decrypt(encrypted)
    cipher = OpenSSL::Cipher.new('AES-256-GCM')
    cipher.decrypt
    cipher.key = Rails.application.credentials.encryption_key
    # ... decryption logic
  end
end

# When enqueuing
encrypted = encrypt(sensitive_data)
SecureJob.perform_later(encrypted)
```

### Message Validation

Validate messages before processing:

```ruby
class ValidationMiddleware
  def call(worker_instance, queue, sqs_msg, body)
    unless valid_signature?(sqs_msg)
      Shoryuken.logger.warn "Invalid message signature"
      sqs_msg.delete
      return
    end

    yield
  end

  private

  def valid_signature?(sqs_msg)
    # Verify message integrity
  end
end
```

---

## Network Security

### Security Groups

Restrict outbound traffic to only required AWS services:

```hcl
resource "aws_security_group" "worker" {
  name = "shoryuken-worker"

  # SQS via VPC endpoint
  egress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    prefix_list_ids = [aws_vpc_endpoint.sqs.prefix_list_id]
  }

  # Database
  egress {
    from_port   = 5432
    to_port     = 5432
    protocol    = "tcp"
    security_groups = [aws_security_group.database.id]
  }
}
```

### Private Subnets

Deploy workers in private subnets without public IPs:

```yaml
# ECS Task Definition
networkMode: awsvpc
networkConfiguration:
  awsvpcConfiguration:
    subnets:
      - subnet-private-1
      - subnet-private-2
    assignPublicIp: DISABLED
```

---

## Secrets Management

### AWS Secrets Manager

```ruby
# config/initializers/shoryuken.rb
require 'aws-sdk-secretsmanager'

Shoryuken.configure_server do |config|
  config.on(:startup) do
    client = Aws::SecretsManager::Client.new
    secret = client.get_secret_value(secret_id: 'myapp/database')

    ENV['DATABASE_URL'] = JSON.parse(secret.secret_string)['url']
  end
end
```

### Environment Variables

Use encrypted environment variables in container platforms:

```yaml
# ECS Task Definition
secrets:
  - name: DATABASE_URL
    valueFrom: arn:aws:secretsmanager:us-east-1:123456789:secret:myapp/database
```

---

## Audit Logging

### CloudTrail

Enable CloudTrail for SQS API calls:

```json
{
  "eventSource": "sqs.amazonaws.com",
  "eventName": "ReceiveMessage",
  "sourceIPAddress": "10.0.1.50",
  "userAgent": "aws-sdk-ruby3/3.0.0"
}
```

### Application Logging

Log job execution for audit trails:

```ruby
class AuditMiddleware
  def call(worker_instance, queue, sqs_msg, body)
    Rails.logger.info({
      event: 'job_started',
      job: worker_instance.class.name,
      queue: queue,
      message_id: sqs_msg.message_id,
      timestamp: Time.current.iso8601
    }.to_json)

    yield

    Rails.logger.info({
      event: 'job_completed',
      job: worker_instance.class.name,
      message_id: sqs_msg.message_id,
      timestamp: Time.current.iso8601
    }.to_json)
  rescue => e
    Rails.logger.error({
      event: 'job_failed',
      job: worker_instance.class.name,
      message_id: sqs_msg.message_id,
      error: e.message,
      timestamp: Time.current.iso8601
    }.to_json)
    raise
  end
end
```

---

## Dead Letter Queue Security

### Protect DLQ Contents

Dead letter queues may contain sensitive data from failed jobs:

1. Apply same encryption as main queue
2. Restrict access to operators only
3. Set appropriate retention period
4. Monitor DLQ depth for anomalies

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789:role/myapp-operator"
      },
      "Action": [
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage",
        "sqs:GetQueueAttributes"
      ],
      "Resource": "arn:aws:sqs:us-east-1:123456789:myapp-jobs-dlq"
    }
  ]
}
```

---

## Security Checklist

- [ ] IAM roles use least privilege
- [ ] No hardcoded credentials
- [ ] Server-side encryption enabled
- [ ] VPC endpoints configured (if applicable)
- [ ] Workers in private subnets
- [ ] Sensitive data not stored in messages
- [ ] CloudTrail enabled for SQS
- [ ] DLQ access restricted
- [ ] Security groups properly configured
- [ ] Secrets managed securely

---

## Related

- [[Amazon SQS IAM Policy for Shoryuken]] - IAM policy examples
- [[Configure the AWS Client]] - Credential configuration
- [[Container Deployment]] - Secure container deployments
