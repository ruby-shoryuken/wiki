# AWS Elastic Beanstalk Configuration

This guide covers running Shoryuken workers on AWS Elastic Beanstalk.

## Recommended: Procfile Method

The simplest and most reliable approach is using a Procfile. This works with Amazon Linux 2 and Amazon Linux 2023.

### Setup

Create a `Procfile` in your application root:

```
web: bundle exec puma -C /opt/elasticbeanstalk/config/private/pumaconf.rb --pidfile /var/app/current/tmp/pids/server.pid
worker: bundle exec shoryuken -R -C config/shoryuken.yml -L /var/app/current/log/shoryuken.log
```

### With Health Checks

```
web: bundle exec puma -C /opt/elasticbeanstalk/config/private/pumaconf.rb
worker: bundle exec shoryuken -R -C config/shoryuken.yml
```

Configure health check via Shoryuken initializer (see [[Health Checks]]).

### Configuration File

```yaml
# config/shoryuken.yml
concurrency: 25
timeout: 25
queues:
  - [default, 1]
```

### IAM Role

Ensure your Elastic Beanstalk instance role has SQS permissions:

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
        "sqs:SendMessage",
        "sqs:SendMessageBatch",
        "sqs:ChangeMessageVisibility",
        "sqs:ChangeMessageVisibilityBatch"
      ],
      "Resource": "arn:aws:sqs:*:*:myapp-*"
    }
  ]
}
```

---

## Instance Sizing

When running both web and worker on the same instance, ensure adequate resources:

| Instance Type | Web + Worker | Notes |
|---------------|--------------|-------|
| t3.small | Light workloads | 2 GB RAM |
| t3.medium | Medium workloads | 4 GB RAM |
| t3.large | Heavy workloads | 8 GB RAM |

### Memory Allocation

```yaml
# config/shoryuken.yml
concurrency: 10  # Reduce if sharing with web server
```

---

## Environment Variables

Set via Elastic Beanstalk console or `.ebextensions`:

```yaml
# .ebextensions/env.config
option_settings:
  aws:elasticbeanstalk:application:environment:
    RAILS_ENV: production
    AWS_REGION: us-east-1
```

---

## Alternative: Platform Hooks (Deprecated)

For more control, you can use platform hooks. This approach is more complex and is deprecated in favor of the Procfile method.

### Amazon Linux 2/2023 Hooks

Create `.platform/hooks/postdeploy/restart_shoryuken.sh`:

```bash
#!/bin/bash

APP_DEPLOY_DIR=$(/opt/elasticbeanstalk/bin/get-config platformconfig -k AppDeployDir)
LOG_DIR="${APP_DEPLOY_DIR}log"
PID_DIR="${APP_DEPLOY_DIR}tmp/pids"
EB_APP_USER=$(/opt/elasticbeanstalk/bin/get-config platformconfig -k AppUser)

mkdir -m 777 -p $PID_DIR

if [ -f $PID_DIR/shoryuken.pid ]; then
  kill -TERM $(cat $PID_DIR/shoryuken.pid) || echo "Shoryuken was not running"
  rm -rf $PID_DIR/shoryuken.pid
fi

sleep 10

cd $APP_DEPLOY_DIR

su -s /bin/bash -c "bundle exec shoryuken \
  -R \
  -P $PID_DIR/shoryuken.pid \
  -C ${APP_DEPLOY_DIR}config/shoryuken.yml \
  -L $LOG_DIR/shoryuken.log \
  -d" $EB_APP_USER

exit 0
```

Create `.platform/hooks/predeploy/mute_shoryuken.sh`:

```bash
#!/bin/bash

APP_DEPLOY_DIR=$(/opt/elasticbeanstalk/bin/get-config platformconfig -k AppDeployDir)
PID_DIR="${APP_DEPLOY_DIR}tmp/pids"
EB_APP_USER=$(/opt/elasticbeanstalk/bin/get-config platformconfig -k AppUser)

if [ -f $PID_DIR/shoryuken.pid ]; then
  su -s /bin/bash -c "kill -USR1 $(cat $PID_DIR/shoryuken.pid)" $EB_APP_USER || echo "Shoryuken was not running"
fi

exit 0
```

Make scripts executable:

```bash
chmod +x .platform/hooks/postdeploy/restart_shoryuken.sh
chmod +x .platform/hooks/predeploy/mute_shoryuken.sh
```

---

## Worker Environment

For dedicated worker environments (no web tier), use a Worker Environment:

1. Create a Worker environment in Elastic Beanstalk
2. Configure the worker queue in the console
3. Use a simple Procfile:

```
worker: bundle exec shoryuken -R -C config/shoryuken.yml
```

Note: Worker environments use SQS daemon, which may conflict with Shoryuken. Consider using a Web environment with `web=0` scaled down instead.

---

## Troubleshooting

### Logs Location

```
/var/app/current/log/shoryuken.log
/var/log/eb-engine.log
```

### View Logs

```bash
eb logs
# or
eb ssh
tail -f /var/app/current/log/shoryuken.log
```

### Common Issues

**Worker not starting:**
- Check IAM permissions
- Verify config file path
- Check log file for errors

**Out of memory:**
- Reduce concurrency
- Upgrade instance type
- Check for memory leaks in jobs

**Permission errors:**
- Ensure hooks are executable
- Check file ownership

---

## Related

- [[Deployment]] - General deployment guide
- [[Amazon SQS IAM Policy for Shoryuken]] - IAM permissions
- [[Health Checks]] - Health monitoring
