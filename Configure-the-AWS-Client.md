# Configure the AWS Client

This guide covers configuring AWS credentials for Shoryuken.

## Credential Resolution Order

The AWS SDK resolves credentials in this order:

1. Explicit credentials in code
2. Environment variables
3. Shared credentials file (`~/.aws/credentials`)
4. IAM instance profile (EC2) / Task role (ECS) / IRSA (EKS)

---

## Environment Variables

```bash
export AWS_ACCESS_KEY_ID=your_access_key
export AWS_SECRET_ACCESS_KEY=your_secret_key
export AWS_REGION=us-east-1

# Optional: session token for temporary credentials
export AWS_SESSION_TOKEN=your_session_token
```

---

## Shared Credentials File

```ini
# ~/.aws/credentials
[default]
aws_access_key_id = your_access_key
aws_secret_access_key = your_secret_key

[production]
aws_access_key_id = prod_access_key
aws_secret_access_key = prod_secret_key
```

```ini
# ~/.aws/config
[default]
region = us-east-1

[profile production]
region = eu-west-1
```

Use a named profile:

```bash
export AWS_PROFILE=production
```

---

## Programmatic Configuration

### Configure SQS Client

```ruby
# config/initializers/shoryuken.rb
sqs_client = Aws::SQS::Client.new(
  region: 'us-east-1',
  access_key_id: ENV['AWS_ACCESS_KEY_ID'],
  secret_access_key: ENV['AWS_SECRET_ACCESS_KEY']
)

Shoryuken.configure_client do |config|
  config.sqs_client = sqs_client
end

Shoryuken.configure_server do |config|
  config.sqs_client = sqs_client
end
```

### With AWS Configuration

```ruby
# Set global AWS config
Aws.config.update(
  region: 'us-east-1',
  credentials: Aws::Credentials.new(
    ENV['AWS_ACCESS_KEY_ID'],
    ENV['AWS_SECRET_ACCESS_KEY']
  )
)
```

---

## IAM Roles (Recommended for Production)

### EC2 Instance Profile

1. Create an IAM role with SQS permissions
2. Attach the role to your EC2 instance
3. No code changes needed - SDK auto-discovers credentials

```ruby
# Just works - no explicit credentials
sqs_client = Aws::SQS::Client.new(region: 'us-east-1')
```

### ECS Task Role

1. Create an IAM role with SQS permissions
2. Configure the role in your task definition:

```json
{
  "family": "shoryuken-worker",
  "taskRoleArn": "arn:aws:iam::123456789:role/shoryuken-task-role",
  "containerDefinitions": [...]
}
```

### EKS IRSA (IAM Roles for Service Accounts)

1. Create an IAM role with SQS permissions
2. Configure the OIDC trust relationship
3. Annotate the Kubernetes service account:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: shoryuken-worker
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/shoryuken-worker-role
```

4. Use the service account in your deployment:

```yaml
spec:
  serviceAccountName: shoryuken-worker
```

---

## AWS SSO

For development with AWS SSO:

```bash
# Configure SSO
aws configure sso

# Login
aws sso login --profile your-profile

# Export profile
export AWS_PROFILE=your-profile
```

---

## Assume Role

```ruby
# config/initializers/shoryuken.rb
sts_client = Aws::STS::Client.new(region: 'us-east-1')

credentials = Aws::AssumeRoleCredentials.new(
  client: sts_client,
  role_arn: 'arn:aws:iam::123456789:role/ShoryukenRole',
  role_session_name: 'shoryuken-session'
)

sqs_client = Aws::SQS::Client.new(
  region: 'us-east-1',
  credentials: credentials
)

Shoryuken.configure_server do |config|
  config.sqs_client = sqs_client
end
```

---

## Cross-Account Access

Access queues in another AWS account:

```ruby
# Assume role in target account
credentials = Aws::AssumeRoleCredentials.new(
  role_arn: 'arn:aws:iam::TARGET_ACCOUNT:role/CrossAccountSQS',
  role_session_name: 'shoryuken'
)

sqs_client = Aws::SQS::Client.new(
  region: 'us-east-1',
  credentials: credentials
)

Shoryuken.configure_server do |config|
  config.sqs_client = sqs_client
end
```

The target account role needs:
1. Trust policy allowing your account to assume the role
2. SQS permissions for the queues

---

## Custom Endpoint (LocalStack/ElasticMQ)

```ruby
# For local development
sqs_client = Aws::SQS::Client.new(
  endpoint: ENV.fetch('SQS_ENDPOINT', 'http://localhost:4566'),
  region: 'us-east-1',
  access_key_id: 'test',
  secret_access_key: 'test'
)

Shoryuken.configure_client do |config|
  config.sqs_client = sqs_client
end
```

---

## Configuration File (YAML)

```yaml
# config/shoryuken.yml
aws:
  access_key_id: <%= ENV['AWS_ACCESS_KEY_ID'] %>
  secret_access_key: <%= ENV['AWS_SECRET_ACCESS_KEY'] %>
  region: us-east-1
```

**Note:** Prefer IAM roles over access keys in the config file.

---

## Verifying Configuration

```bash
# Verify AWS credentials
aws sts get-caller-identity

# List SQS queues
aws sqs list-queues

# Or via Shoryuken CLI
shoryuken sqs ls
```

---

## Troubleshooting

### Credentials Not Found

```
Aws::Errors::MissingCredentialsError
```

**Solutions:**
1. Set environment variables
2. Configure `~/.aws/credentials`
3. Attach IAM role (EC2/ECS/EKS)

### Invalid Credentials

```
Aws::SQS::Errors::InvalidClientTokenId
```

**Solutions:**
1. Verify credentials are correct
2. Check credentials haven't expired
3. Ensure IAM user/role is active

### Region Not Specified

```
Aws::Errors::MissingRegionError
```

**Solutions:**
1. Set `AWS_REGION` environment variable
2. Configure region in `~/.aws/config`
3. Specify region in code

### Access Denied

```
Aws::SQS::Errors::AccessDenied
```

**Solutions:**
1. Verify IAM permissions (see [[Amazon SQS IAM Policy for Shoryuken]])
2. Check queue policy allows access
3. Verify assume role permissions

---

## Best Practices

1. **Use IAM roles** in production instead of access keys
2. **Rotate credentials** regularly if using access keys
3. **Use least privilege** - only grant necessary permissions
4. **Separate environments** - use different credentials/roles per environment
5. **Never commit credentials** - use environment variables or secrets management

---

## Related

- [[Amazon SQS IAM Policy for Shoryuken]] - IAM permissions
- [[Security Best Practices]] - Security guidelines
- [[Using a local mock SQS server]] - Local development
- [AWS SDK for Ruby Documentation](https://docs.aws.amazon.com/sdk-for-ruby/v3/api/)
