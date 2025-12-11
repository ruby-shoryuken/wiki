# Amazon SQS IAM Policy for Shoryuken

This guide provides IAM policy examples for different Shoryuken use cases.

## Minimum Required Policy

Basic policy for consuming and producing messages:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ShoryukenAccess",
      "Effect": "Allow",
      "Action": [
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage",
        "sqs:DeleteMessageBatch",
        "sqs:GetQueueAttributes",
        "sqs:GetQueueUrl",
        "sqs:SendMessage",
        "sqs:SendMessageBatch",
        "sqs:ChangeMessageVisibility",
        "sqs:ChangeMessageVisibilityBatch"
      ],
      "Resource": "arn:aws:sqs:us-east-1:123456789:myapp-*"
    }
  ]
}
```

---

## Role-Specific Policies

### Consumer Only (Worker)

For workers that only process messages:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ShoryukenConsumer",
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

### Producer Only (Web App)

For applications that only enqueue messages:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ShoryukenProducer",
      "Effect": "Allow",
      "Action": [
        "sqs:SendMessage",
        "sqs:SendMessageBatch",
        "sqs:GetQueueUrl"
      ],
      "Resource": "arn:aws:sqs:us-east-1:123456789:myapp-*"
    }
  ]
}
```

### Admin (CLI Tools)

For using `shoryuken sqs` commands:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ShoryukenAdmin",
      "Effect": "Allow",
      "Action": [
        "sqs:ListQueues",
        "sqs:CreateQueue",
        "sqs:DeleteQueue",
        "sqs:PurgeQueue",
        "sqs:GetQueueAttributes",
        "sqs:GetQueueUrl",
        "sqs:ReceiveMessage",
        "sqs:SendMessage",
        "sqs:SendMessageBatch",
        "sqs:DeleteMessage",
        "sqs:DeleteMessageBatch"
      ],
      "Resource": "*"
    }
  ]
}
```

---

## Feature-Specific Permissions

### FIFO Queues

FIFO queues use the same permissions as standard queues. No additional permissions required.

### Dead Letter Queues

Add permissions for the DLQ:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "MainQueue",
      "Effect": "Allow",
      "Action": [
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage",
        "sqs:GetQueueAttributes",
        "sqs:GetQueueUrl"
      ],
      "Resource": "arn:aws:sqs:us-east-1:123456789:myapp-jobs"
    },
    {
      "Sid": "DeadLetterQueue",
      "Effect": "Allow",
      "Action": [
        "sqs:SendMessage",
        "sqs:GetQueueUrl"
      ],
      "Resource": "arn:aws:sqs:us-east-1:123456789:myapp-jobs-dlq"
    }
  ]
}
```

### KMS Encrypted Queues

For queues using SSE-KMS:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "SQSAccess",
      "Effect": "Allow",
      "Action": [
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage",
        "sqs:SendMessage",
        "sqs:GetQueueAttributes",
        "sqs:GetQueueUrl"
      ],
      "Resource": "arn:aws:sqs:us-east-1:123456789:myapp-*"
    },
    {
      "Sid": "KMSAccess",
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt",
        "kms:GenerateDataKey"
      ],
      "Resource": "arn:aws:kms:us-east-1:123456789:key/12345678-1234-1234-1234-123456789012",
      "Condition": {
        "StringEquals": {
          "kms:ViaService": "sqs.us-east-1.amazonaws.com"
        }
      }
    }
  ]
}
```

---

## Resource Restrictions

### By Queue Name Pattern

```json
{
  "Resource": "arn:aws:sqs:us-east-1:123456789:myapp-*"
}
```

### By Environment

```json
{
  "Resource": [
    "arn:aws:sqs:us-east-1:123456789:production-*",
    "arn:aws:sqs:us-east-1:123456789:staging-*"
  ]
}
```

### Single Queue

```json
{
  "Resource": "arn:aws:sqs:us-east-1:123456789:myapp-jobs"
}
```

---

## Cross-Account Access

### Producer Account Policy

Allow sending to queues in another account:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "sqs:SendMessage",
        "sqs:GetQueueUrl"
      ],
      "Resource": "arn:aws:sqs:us-east-1:OTHER_ACCOUNT:shared-queue"
    }
  ]
}
```

### Queue Policy (Owner Account)

Allow access from another account:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::PRODUCER_ACCOUNT:root"
      },
      "Action": "sqs:SendMessage",
      "Resource": "arn:aws:sqs:us-east-1:OWNER_ACCOUNT:shared-queue"
    }
  ]
}
```

---

## IAM Role Examples

### ECS Task Role

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

Trust policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ecs-tasks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

### EKS Service Account (IRSA)

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

Trust policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE:sub": "system:serviceaccount:default:shoryuken-worker"
        }
      }
    }
  ]
}
```

---

## Troubleshooting

### Common Errors

**AccessDenied on ReceiveMessage:**
- Check queue ARN in policy matches actual queue
- Verify IAM role is attached to instance/task
- Check for condition keys that might restrict access

**AccessDenied on SendMessage:**
- Verify producer has `sqs:SendMessage` permission
- Check queue policy allows the principal

**KMS Access Denied:**
- Add `kms:Decrypt` and `kms:GenerateDataKey` permissions
- Ensure KMS key policy allows the IAM role

### Testing Permissions

Use AWS CLI to verify:

```bash
# Test receive
aws sqs receive-message --queue-url https://sqs.us-east-1.amazonaws.com/123456789/myqueue

# Test send
aws sqs send-message --queue-url https://sqs.us-east-1.amazonaws.com/123456789/myqueue --message-body "test"
```

---

## Related

- [[Security Best Practices]] - Comprehensive security guide
- [[Configure the AWS Client]] - Credential configuration
- [AWS SQS IAM Documentation](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-authentication-and-access-control.html)
