# Deployment

This guide covers deploying Shoryuken to various platforms. For containerized deployments (Docker, Kubernetes, ECS), see [[Container Deployment]].

## Heroku

### Procfile Setup

Add a `worker` process to your [Procfile](https://devcenter.heroku.com/articles/procfile):

```
web: bundle exec puma -C config/puma.rb
worker: bundle exec shoryuken -R -C config/shoryuken.yml
```

### Scale Workers

```shell
# Start worker dyno
heroku ps:scale worker=1

# Scale to multiple workers
heroku ps:scale worker=3

# Disable web if not needed
heroku ps:scale web=0
```

### Configuration

Set AWS credentials via config vars:

```shell
heroku config:set AWS_ACCESS_KEY_ID=xxx
heroku config:set AWS_SECRET_ACCESS_KEY=xxx
heroku config:set AWS_REGION=us-east-1
```

### Graceful Shutdown

Heroku sends SIGTERM and allows 30 seconds for shutdown. Configure timeout accordingly:

```yaml
# config/shoryuken.yml
timeout: 25  # Leave headroom for cleanup
```

---

## systemd

### Service File

Create `/etc/systemd/system/shoryuken.service`:

```ini
[Unit]
Description=Shoryuken background job processor
After=network.target

[Service]
Type=simple
User=deploy
Group=deploy
WorkingDirectory=/var/www/myapp/current

# Environment
Environment=RAILS_ENV=production
EnvironmentFile=/var/www/myapp/shared/.env

# Command
ExecStart=/usr/local/bin/bundle exec shoryuken -R -C config/shoryuken.yml

# Graceful shutdown
ExecReload=/bin/kill -USR1 $MAINPID
TimeoutStopSec=300
KillSignal=SIGTERM

# Restart on failure
Restart=on-failure
RestartSec=5

# Logging
StandardOutput=append:/var/log/shoryuken/shoryuken.log
StandardError=append:/var/log/shoryuken/shoryuken.log

[Install]
WantedBy=multi-user.target
```

### Commands

```shell
# Enable service
sudo systemctl enable shoryuken

# Start
sudo systemctl start shoryuken

# Stop (graceful)
sudo systemctl stop shoryuken

# Reload (soft restart)
sudo systemctl reload shoryuken

# Check status
sudo systemctl status shoryuken

# View logs
sudo journalctl -u shoryuken -f
```

### Multiple Workers

For multiple worker instances, use a template:

```ini
# /etc/systemd/system/shoryuken@.service
[Unit]
Description=Shoryuken worker %i
After=network.target

[Service]
Type=simple
User=deploy
WorkingDirectory=/var/www/myapp/current
Environment=RAILS_ENV=production
ExecStart=/usr/local/bin/bundle exec shoryuken -R -C config/shoryuken.yml
TimeoutStopSec=300

[Install]
WantedBy=multi-user.target
```

Start multiple instances:

```shell
sudo systemctl enable shoryuken@{1..3}
sudo systemctl start shoryuken@{1..3}
```

---

## Capistrano

Use [capistrano-shoryuken](https://github.com/joekhoobyar/capistrano-shoryuken) for deployment integration.

### Gemfile

```ruby
group :development do
  gem 'capistrano-shoryuken', require: false
end
```

### Capfile

```ruby
require 'capistrano/shoryuken'
```

### Deploy Configuration

```ruby
# config/deploy.rb
set :shoryuken_config, -> { "#{current_path}/config/shoryuken.yml" }
set :shoryuken_role, :worker
set :shoryuken_processes, 2
```

---

## AWS Elastic Beanstalk

See [[AWS Beanstalk Config]] for detailed configuration.

### Quick Setup (Procfile Method)

```
# Procfile
web: bundle exec puma -C /opt/elasticbeanstalk/config/private/pumaconf.rb
worker: bundle exec shoryuken -R -C config/shoryuken.yml
```

---

## AWS OpsWorks

Use the [OpsWorks Recipe](https://github.com/carpet/opsworks-shoryuken) for Chef-based deployments.

---

## Production Checklist

### Before Deploying

- [ ] AWS credentials configured (prefer IAM roles)
- [ ] SQS queues created
- [ ] IAM permissions set (see [[Amazon SQS IAM Policy for Shoryuken]])
- [ ] Database connection pool sized ≥ concurrency
- [ ] Shoryuken configuration file ready

### Configuration

```yaml
# config/shoryuken.yml
concurrency: 25
timeout: 25
queues:
  - [critical, 3]
  - [default, 2]
  - [low, 1]
```

### Database Pool

```yaml
# config/database.yml
production:
  pool: <%= ENV.fetch("DB_POOL") { 25 } %>
```

### Health Checks

See [[Health Checks]] for setting up health monitoring.

### Logging

```ruby
# config/initializers/shoryuken.rb
Shoryuken.configure_server do |config|
  Rails.logger = Shoryuken::Logging.logger
  Rails.logger.level = Rails.application.config.log_level
end
```

---

## Monitoring

### Signals for Debugging

```shell
# Print thread backtraces and stats
kill -TTIN $(cat tmp/pids/shoryuken.pid)
```

See [[Signals]] for all available signals.

### CloudWatch Metrics

Monitor these SQS metrics:
- `ApproximateNumberOfMessagesVisible` - Queue depth
- `ApproximateAgeOfOldestMessage` - Processing delay
- `NumberOfMessagesReceived` - Throughput
- `NumberOfMessagesDeleted` - Completion rate

### Error Tracking

See [[Sentry.io Integration]] and [[Honeybadger Integration]].

---

## Scaling Strategies

### Horizontal Scaling

Add more worker processes/containers. Each worker:
- Has its own connection pool
- Polls independently
- Handles `concurrency` simultaneous jobs

### Queue-Based Auto Scaling

Scale based on queue depth:

```
Messages per worker = ApproximateNumberOfMessagesVisible / desired_workers
Target: 50-100 messages per worker
```

### Processing Groups

Isolate critical queues:

```yaml
# config/shoryuken.yml
groups:
  critical:
    concurrency: 10
    queues:
      - [payments, 1]
  default:
    concurrency: 25
    queues:
      - [emails, 2]
      - [reports, 1]
```

---

## Related

- [[Container Deployment]] - Docker, Kubernetes, ECS
- [[AWS Beanstalk Config]] - Elastic Beanstalk specifics
- [[Health Checks]] - Health monitoring
- [[Signals]] - Process control
- [[Processing Groups]] - Queue isolation
