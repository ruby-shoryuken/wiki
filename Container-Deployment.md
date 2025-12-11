# Container Deployment

This guide covers deploying Shoryuken workers in containerized environments including Docker, Kubernetes, and AWS ECS/Fargate.

## Docker

### Basic Dockerfile

```dockerfile
FROM ruby:3.3-slim

WORKDIR /app

# Install dependencies
RUN apt-get update -qq && \
    apt-get install -y --no-install-recommends \
    build-essential \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# Install gems
COPY Gemfile Gemfile.lock ./
RUN bundle config set --local deployment 'true' && \
    bundle config set --local without 'development test' && \
    bundle install

# Copy application
COPY . .

# Create non-root user
RUN useradd -m appuser && chown -R appuser:appuser /app
USER appuser

# Health check port
EXPOSE 3001

# Default command
CMD ["bundle", "exec", "shoryuken", "-R", "-C", "config/shoryuken.yml"]
```

### Production Dockerfile (Multi-Stage)

```dockerfile
# Build stage
FROM ruby:3.3-slim AS builder

WORKDIR /app

RUN apt-get update -qq && \
    apt-get install -y --no-install-recommends \
    build-essential \
    libpq-dev \
    git

COPY Gemfile Gemfile.lock ./
RUN bundle config set --local deployment 'true' && \
    bundle config set --local without 'development test' && \
    bundle install && \
    rm -rf ~/.bundle/cache

COPY . .

# Runtime stage
FROM ruby:3.3-slim

WORKDIR /app

RUN apt-get update -qq && \
    apt-get install -y --no-install-recommends \
    libpq5 \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Copy gems and app from builder
COPY --from=builder /app /app
COPY --from=builder /usr/local/bundle /usr/local/bundle

# Create non-root user
RUN useradd -m -u 1000 appuser && chown -R appuser:appuser /app
USER appuser

ENV RAILS_ENV=production
ENV HEALTH_PORT=3001

EXPOSE 3001

HEALTHCHECK --interval=30s --timeout=5s --start-period=30s --retries=3 \
    CMD curl -f http://localhost:3001/health || exit 1

CMD ["bundle", "exec", "shoryuken", "-R", "-C", "config/shoryuken.yml"]
```

### Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  worker:
    build: .
    command: bundle exec shoryuken -R -C config/shoryuken.yml
    environment:
      - RAILS_ENV=production
      - DATABASE_URL=postgres://db:5432/myapp
      - AWS_REGION=us-east-1
      - AWS_ACCESS_KEY_ID
      - AWS_SECRET_ACCESS_KEY
      - HEALTH_PORT=3001
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3001/health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 30s
    depends_on:
      - db
    deploy:
      replicas: 2
      resources:
        limits:
          memory: 512M
          cpus: '1'
        reservations:
          memory: 256M
          cpus: '0.25'

  db:
    image: postgres:15
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=myapp
      - POSTGRES_PASSWORD=secret

volumes:
  postgres_data:
```

---

## Kubernetes

### Deployment

```yaml
# k8s/worker-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shoryuken-worker
  labels:
    app: shoryuken-worker
spec:
  replicas: 3
  selector:
    matchLabels:
      app: shoryuken-worker
  template:
    metadata:
      labels:
        app: shoryuken-worker
    spec:
      serviceAccountName: shoryuken-worker
      terminationGracePeriodSeconds: 300  # 5 minutes for graceful shutdown
      containers:
        - name: worker
          image: myapp:latest
          command: ["bundle", "exec", "shoryuken", "-R", "-C", "config/shoryuken.yml"]
          env:
            - name: RAILS_ENV
              value: "production"
            - name: HEALTH_PORT
              value: "3001"
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: myapp-secrets
                  key: database-url
          envFrom:
            - configMapRef:
                name: myapp-config
          ports:
            - containerPort: 3001
              name: health
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "1000m"
          livenessProbe:
            httpGet:
              path: /health
              port: health
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /ready
              port: health
            initialDelaySeconds: 5
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 2
          lifecycle:
            preStop:
              exec:
                # Give time for load balancer to deregister
                command: ["sleep", "5"]
```

### Service Account with IRSA (AWS)

```yaml
# k8s/service-account.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: shoryuken-worker
  annotations:
    # IAM Roles for Service Accounts
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/shoryuken-worker-role
```

### ConfigMap

```yaml
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
data:
  AWS_REGION: "us-east-1"
  RAILS_LOG_TO_STDOUT: "true"
```

### Horizontal Pod Autoscaler

```yaml
# k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: shoryuken-worker-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: shoryuken-worker
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: External
      external:
        metric:
          name: sqs_approximate_number_of_messages_visible
          selector:
            matchLabels:
              queue_name: myapp-default
        target:
          type: AverageValue
          averageValue: "100"  # Scale up when > 100 messages per pod
```

### Pod Disruption Budget

```yaml
# k8s/pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: shoryuken-worker-pdb
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: shoryuken-worker
```

---

## AWS ECS

### Task Definition

```json
{
  "family": "shoryuken-worker",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::123456789:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::123456789:role/shoryuken-worker-task-role",
  "containerDefinitions": [
    {
      "name": "worker",
      "image": "123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:latest",
      "command": ["bundle", "exec", "shoryuken", "-R", "-C", "config/shoryuken.yml"],
      "essential": true,
      "environment": [
        {"name": "RAILS_ENV", "value": "production"},
        {"name": "HEALTH_PORT", "value": "3001"}
      ],
      "secrets": [
        {
          "name": "DATABASE_URL",
          "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789:secret:myapp/database-url"
        }
      ],
      "portMappings": [
        {
          "containerPort": 3001,
          "protocol": "tcp"
        }
      ],
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:3001/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 60
      },
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/shoryuken-worker",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "worker"
        }
      },
      "stopTimeout": 300
    }
  ]
}
```

### ECS Service

```json
{
  "serviceName": "shoryuken-worker",
  "cluster": "myapp-cluster",
  "taskDefinition": "shoryuken-worker",
  "desiredCount": 3,
  "launchType": "FARGATE",
  "networkConfiguration": {
    "awsvpcConfiguration": {
      "subnets": ["subnet-xxx", "subnet-yyy"],
      "securityGroups": ["sg-xxx"],
      "assignPublicIp": "DISABLED"
    }
  },
  "deploymentConfiguration": {
    "maximumPercent": 200,
    "minimumHealthyPercent": 100
  }
}
```

### ECS Auto Scaling

```json
{
  "ScalableTarget": {
    "ServiceNamespace": "ecs",
    "ResourceId": "service/myapp-cluster/shoryuken-worker",
    "ScalableDimension": "ecs:service:DesiredCount",
    "MinCapacity": 2,
    "MaxCapacity": 10
  },
  "ScalingPolicy": {
    "PolicyName": "sqs-queue-depth",
    "PolicyType": "TargetTrackingScaling",
    "TargetTrackingScalingPolicyConfiguration": {
      "TargetValue": 100,
      "CustomizedMetricSpecification": {
        "MetricName": "ApproximateNumberOfMessagesVisible",
        "Namespace": "AWS/SQS",
        "Dimensions": [
          {"Name": "QueueName", "Value": "myapp-default"}
        ],
        "Statistic": "Average"
      },
      "ScaleInCooldown": 300,
      "ScaleOutCooldown": 60
    }
  }
}
```

---

## Graceful Shutdown

### Understanding Container Signals

When a container is terminated:

1. **SIGTERM** is sent to the main process
2. Container has `terminationGracePeriodSeconds` to shut down
3. If still running, **SIGKILL** is sent

### Shoryuken Shutdown Behavior

```
SIGTERM received
     │
     ▼
stopping? = true  ────► Jobs can checkpoint via Continuations
     │
     ▼
Stop fetching new messages
     │
     ▼
Wait for in-flight messages to complete
     │
     ▼
Fire shutdown events
     │
     ▼
Exit cleanly
```

### Recommended Grace Period

Set `terminationGracePeriodSeconds` based on your longest job:

```yaml
# Kubernetes
spec:
  terminationGracePeriodSeconds: 300  # 5 minutes

# ECS Task Definition
"stopTimeout": 300
```

### Shoryuken Timeout Configuration

```yaml
# config/shoryuken.yml
timeout: 25  # Seconds to wait for workers during shutdown
```

---

## Resource Sizing

### Memory

| Concurrency | Recommended Memory |
|-------------|-------------------|
| 5 | 256 MB |
| 25 (default) | 512 MB |
| 50 | 1 GB |
| 100 | 2 GB |

Add more for Rails applications with large codebases.

### CPU

| Concurrency | Recommended CPU |
|-------------|-----------------|
| 5 | 0.25 vCPU |
| 25 | 0.5-1 vCPU |
| 50 | 1-2 vCPU |

### Database Connections

Set your connection pool size ≥ concurrency:

```yaml
# config/database.yml
production:
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 25 } %>
```

---

## Environment Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `RAILS_ENV` | Rails environment | `production` |
| `AWS_REGION` | AWS region | `us-east-1` |
| `DATABASE_URL` | Database connection | `postgres://...` |
| `HEALTH_PORT` | Health check port | `3001` |
| `RAILS_LOG_TO_STDOUT` | Log to stdout | `true` |

### AWS Credentials

For containers, prefer IAM roles over access keys:

- **EKS**: Use IRSA (IAM Roles for Service Accounts)
- **ECS**: Use Task IAM Role
- **EC2**: Use Instance Profile

---

## Logging

Configure Rails to log to stdout for container log aggregation:

```ruby
# config/environments/production.rb
if ENV["RAILS_LOG_TO_STDOUT"].present?
  logger = ActiveSupport::Logger.new(STDOUT)
  logger.formatter = config.log_formatter
  config.logger = ActiveSupport::TaggedLogging.new(logger)
end
```

### Structured Logging

```ruby
# config/initializers/shoryuken.rb
Shoryuken.configure_server do |config|
  config.on(:startup) do
    Shoryuken.logger.formatter = proc do |severity, datetime, progname, msg|
      {
        timestamp: datetime.iso8601,
        level: severity,
        message: msg,
        service: 'shoryuken-worker'
      }.to_json + "\n"
    end
  end
end
```

---

## Related

- [[Deployment]] - General deployment options
- [[Health Checks]] - Health check configuration
- [[Signals]] - Signal handling
- [[Lifecycle Events]] - Startup and shutdown hooks
