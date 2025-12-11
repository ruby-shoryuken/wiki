# Scaling Patterns

This guide covers strategies for scaling Shoryuken workers to handle varying workloads.

## Horizontal Scaling

### Adding More Workers

Each worker process:
- Has its own database connection pool
- Polls SQS independently
- Handles `concurrency` threads

```
Total capacity = Number of workers × Concurrency per worker
```

### Example: 3 Workers × 25 Concurrency = 75 Concurrent Jobs

```yaml
# config/shoryuken.yml
concurrency: 25
```

Deploy multiple instances of your worker process.

---

## Vertical Scaling

### Increase Concurrency

```yaml
# config/shoryuken.yml
concurrency: 50  # Up from default 25
```

### Considerations

| Concurrency | Memory | Database Pool | Best For |
|-------------|--------|---------------|----------|
| 10-25 | Low | 25 | I/O-bound jobs |
| 25-50 | Medium | 50 | Mixed workloads |
| 50-100 | High | 100 | CPU-light, I/O-heavy |

**Limits:**
- Database connection pool must match
- Memory increases with concurrency
- CPU-bound jobs don't benefit beyond core count

---

## Queue-Based Scaling

### Separate Queues by Priority

```yaml
# config/shoryuken.yml
queues:
  - [critical, 10]
  - [default, 3]
  - [low, 1]
```

### Dedicated Workers per Queue

```yaml
# Worker 1: critical-worker.yml
queues:
  - critical

# Worker 2: default-worker.yml
queues:
  - default
  - low
```

Deploy different numbers of each:
- 3× critical workers
- 2× default workers

---

## Processing Groups

Isolate workloads within a single process:

```yaml
# config/shoryuken.yml
groups:
  realtime:
    concurrency: 10
    delay: 0
    queues:
      - [webhooks, 1]
      - [notifications, 1]
  batch:
    concurrency: 5
    delay: 5
    queues:
      - [reports, 1]
      - [exports, 1]
```

See [[Processing Groups]] for details.

---

## Auto-Scaling

### By Queue Depth

Scale workers based on `ApproximateNumberOfMessagesVisible`:

```
Target: 50-100 messages per worker
```

#### Kubernetes HPA with KEDA

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: shoryuken-worker
spec:
  scaleTargetRef:
    name: shoryuken-worker
  minReplicaCount: 2
  maxReplicaCount: 20
  triggers:
    - type: aws-sqs-queue
      metadata:
        queueURL: https://sqs.us-east-1.amazonaws.com/123456789/myqueue
        queueLength: "100"  # Target messages per pod
        awsRegion: us-east-1
```

#### AWS ECS Auto Scaling

```json
{
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

### By Message Age

Scale when messages wait too long:

```yaml
# KEDA trigger
triggers:
  - type: aws-cloudwatch
    metadata:
      namespace: AWS/SQS
      dimensionName: QueueName
      dimensionValue: myqueue
      metricName: ApproximateAgeOfOldestMessage
      targetMetricValue: "300"  # 5 minutes
```

---

## Queue Sharding

### By Customer/Tenant

```ruby
class TenantJob < ApplicationJob
  queue_as -> { "tenant_#{arguments.first[:tenant_id] % 10}" }

  def perform(tenant_id:, data:)
    # Process for specific tenant
  end
end
```

Queues: `tenant_0`, `tenant_1`, ... `tenant_9`

### By Data Type

```ruby
class ProcessDataJob < ApplicationJob
  queue_as -> {
    case arguments.first[:type]
    when 'urgent' then 'data_urgent'
    when 'large' then 'data_large'
    else 'data_default'
    end
  }
end
```

---

## Worker Specialization

### Heavy vs Light Workers

```yaml
# light-worker.yml - Many small jobs
concurrency: 50
queues:
  - notifications
  - emails

# heavy-worker.yml - Few large jobs
concurrency: 5
queues:
  - reports
  - exports
```

### Resource Allocation

| Worker Type | CPU | Memory | Concurrency |
|------------|-----|--------|-------------|
| Light | 0.5 | 512 MB | 50 |
| Medium | 1 | 1 GB | 25 |
| Heavy | 2 | 2 GB | 5 |

---

## Scaling Strategies by Workload

### Bursty Traffic

- Use auto-scaling with aggressive scale-out
- Keep minimum workers running
- Scale in slowly (cooldown period)

```yaml
# KEDA
minReplicaCount: 2
maxReplicaCount: 50
cooldownPeriod: 300
```

### Steady Traffic

- Fixed worker count
- Size based on average throughput
- Manual scaling for growth

### Time-Based

Schedule workers for predictable patterns:

```yaml
# Kubernetes CronJob to scale
apiVersion: batch/v1
kind: CronJob
metadata:
  name: scale-workers-morning
spec:
  schedule: "0 8 * * 1-5"  # 8 AM weekdays
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: kubectl
              command:
                - kubectl
                - scale
                - deployment/shoryuken-worker
                - --replicas=10
```

---

## Cost Optimization

### Right-Sizing

Monitor utilization and adjust:

```ruby
Shoryuken.configure_server do |config|
  config.on(:utilization_update) do |group, utilization|
    # Log for analysis
    Rails.logger.info "Utilization #{group}: #{(utilization * 100).round}%"
  end
end
```

Target: 60-80% utilization

### Spot/Preemptible Instances

Workers are stateless - safe for spot instances:

```yaml
# Kubernetes
spec:
  nodeSelector:
    node-type: spot
  tolerations:
    - key: "spot"
      operator: "Equal"
      value: "true"
```

### Scale to Zero

For dev/staging with KEDA:

```yaml
minReplicaCount: 0
idleReplicaCount: 0
```

---

## Monitoring Scaling

### Key Metrics

| Metric | Healthy Range | Action if Outside |
|--------|---------------|-------------------|
| Queue depth | < 1000 | Scale out |
| Message age | < 5 min | Scale out |
| Utilization | 60-80% | Adjust concurrency |
| Error rate | < 1% | Investigate |

### Dashboard

Track:
- Workers running
- Messages processed/minute
- Queue depth over time
- Scaling events

---

## Related

- [[Processing Groups]] - Workload isolation
- [[Performance Tuning]] - Optimization
- [[Container Deployment]] - Kubernetes/ECS deployment
- [[Monitoring and Metrics]] - Observability
