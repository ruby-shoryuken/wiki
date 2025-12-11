# Health Checks

Shoryuken provides a health check API to monitor the status of processing groups. This is useful for load balancer health checks, Kubernetes probes, and monitoring systems.

## The `healthy?` Method

The `Launcher` class provides a `healthy?` method that returns `true` when all processing groups are running normally:

```ruby
launcher = Shoryuken::Runner.instance.launcher
launcher.healthy?  # => true or false
```

### What It Checks

The `healthy?` method verifies that:
- All configured processing groups have a running manager
- Each manager's `running?` method returns `true`

```ruby
def healthy?
  Shoryuken.groups.keys.all? do |group|
    manager = @managers.find { |m| m.group == group }
    manager && manager.running?
  end
end
```

## HTTP Health Endpoint

Shoryuken doesn't include a built-in HTTP server, but you can easily add one using a lifecycle event:

### Using WEBrick

```ruby
# config/initializers/shoryuken.rb
require 'webrick'

Shoryuken.configure_server do |config|
  config.on(:startup) do
    @health_server = WEBrick::HTTPServer.new(Port: 3001, Logger: WEBrick::Log.new("/dev/null"))

    @health_server.mount_proc '/health' do |req, res|
      launcher = Shoryuken::Runner.instance.launcher

      if launcher&.healthy?
        res.status = 200
        res.body = 'OK'
      else
        res.status = 503
        res.body = 'Unhealthy'
      end
    end

    Thread.new { @health_server.start }
  end

  config.on(:shutdown) do
    @health_server&.shutdown
  end
end
```

### Using Rack

```ruby
# config/initializers/shoryuken.rb
require 'rack'
require 'rack/handler/webrick'

Shoryuken.configure_server do |config|
  config.on(:startup) do
    health_app = lambda do |env|
      launcher = Shoryuken::Runner.instance.launcher

      if launcher&.healthy?
        [200, { 'Content-Type' => 'text/plain' }, ['OK']]
      else
        [503, { 'Content-Type' => 'text/plain' }, ['Unhealthy']]
      end
    end

    Thread.new do
      Rack::Handler::WEBrick.run(health_app, Port: 3001, Logger: WEBrick::Log.new("/dev/null"))
    end
  end
end
```

### Using Puma (Production Recommended)

For production, consider using a minimal Puma server:

```ruby
# lib/shoryuken_health_server.rb
require 'puma'
require 'rack'

class ShoryukenHealthServer
  def initialize(port: 3001)
    @port = port
  end

  def start
    app = self

    @server = Puma::Server.new(app)
    @server.add_tcp_listener('0.0.0.0', @port)
    @server.run
  end

  def stop
    @server&.stop(true)
  end

  def call(env)
    case env['PATH_INFO']
    when '/health', '/healthz'
      health_check
    when '/ready', '/readiness'
      readiness_check
    else
      [404, {}, ['Not Found']]
    end
  end

  private

  def health_check
    launcher = Shoryuken::Runner.instance.launcher

    if launcher&.healthy?
      [200, { 'Content-Type' => 'application/json' }, [{ status: 'healthy' }.to_json]]
    else
      [503, { 'Content-Type' => 'application/json' }, [{ status: 'unhealthy' }.to_json]]
    end
  end

  def readiness_check
    launcher = Shoryuken::Runner.instance.launcher

    if launcher && !launcher.stopping?
      [200, { 'Content-Type' => 'application/json' }, [{ status: 'ready' }.to_json]]
    else
      [503, { 'Content-Type' => 'application/json' }, [{ status: 'not_ready' }.to_json]]
    end
  end
end

# config/initializers/shoryuken.rb
require_relative '../../lib/shoryuken_health_server'

Shoryuken.configure_server do |config|
  config.on(:startup) do
    @health_server = ShoryukenHealthServer.new(port: ENV.fetch('HEALTH_PORT', 3001))
    Thread.new { @health_server.start }
  end

  config.on(:shutdown) do
    @health_server&.stop
  end
end
```

## Kubernetes Integration

### Liveness Probe

Use for determining if the container should be restarted:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shoryuken-worker
spec:
  template:
    spec:
      containers:
        - name: worker
          image: myapp:latest
          command: ["bundle", "exec", "shoryuken", "-R", "-C", "config/shoryuken.yml"]
          ports:
            - containerPort: 3001
              name: health
          livenessProbe:
            httpGet:
              path: /health
              port: health
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
```

### Readiness Probe

Use for determining if the container should receive traffic (useful if you have HTTP endpoints):

```yaml
          readinessProbe:
            httpGet:
              path: /ready
              port: health
            initialDelaySeconds: 5
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 2
```

### Complete Example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shoryuken-worker
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
      terminationGracePeriodSeconds: 300  # 5 minutes for graceful shutdown
      containers:
        - name: worker
          image: myapp:latest
          command: ["bundle", "exec", "shoryuken", "-R", "-C", "config/shoryuken.yml"]
          env:
            - name: HEALTH_PORT
              value: "3001"
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
```

## AWS ECS Integration

### Task Definition Health Check

```json
{
  "containerDefinitions": [
    {
      "name": "shoryuken-worker",
      "image": "myapp:latest",
      "command": ["bundle", "exec", "shoryuken", "-R", "-C", "config/shoryuken.yml"],
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:3001/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 60
      },
      "portMappings": [
        {
          "containerPort": 3001,
          "protocol": "tcp"
        }
      ]
    }
  ]
}
```

## Load Balancer Health Checks

### AWS ALB/NLB

```yaml
# CloudFormation
TargetGroup:
  Type: AWS::ElasticLoadBalancingV2::TargetGroup
  Properties:
    HealthCheckPath: /health
    HealthCheckPort: "3001"
    HealthCheckProtocol: HTTP
    HealthCheckIntervalSeconds: 30
    HealthCheckTimeoutSeconds: 10
    HealthyThresholdCount: 2
    UnhealthyThresholdCount: 3
```

## Health Check During Shutdown

During graceful shutdown:

1. `stopping?` becomes `true` immediately
2. `healthy?` remains `true` while managers are running
3. `healthy?` becomes `false` when managers stop

This allows load balancers to drain connections while the worker processes remaining messages.

### Recommended Shutdown Flow

```
SIGTERM received
     │
     ▼
stopping? = true  ─────► Readiness probe fails (stop accepting new work)
     │
     ▼
Process in-flight messages
     │
     ▼
Managers stop
     │
     ▼
healthy? = false  ─────► Liveness probe fails (container can be removed)
```

## Monitoring Integration

### Prometheus Metrics

```ruby
# config/initializers/shoryuken.rb
require 'prometheus/client'

prometheus = Prometheus::Client.registry

shoryuken_healthy = Prometheus::Client::Gauge.new(
  :shoryuken_healthy,
  docstring: 'Whether Shoryuken is healthy',
  labels: [:group]
)
prometheus.register(shoryuken_healthy)

Shoryuken.configure_server do |config|
  config.on(:startup) do
    Thread.new do
      loop do
        launcher = Shoryuken::Runner.instance.launcher

        Shoryuken.groups.keys.each do |group|
          manager = launcher&.instance_variable_get(:@managers)&.find { |m| m.group == group }
          healthy = manager&.running? ? 1 : 0
          shoryuken_healthy.set(healthy, labels: { group: group })
        end

        sleep 10
      end
    end
  end
end
```

### DataDog

```ruby
# config/initializers/shoryuken.rb
Shoryuken.configure_server do |config|
  config.on(:startup) do
    Thread.new do
      loop do
        launcher = Shoryuken::Runner.instance.launcher
        healthy = launcher&.healthy? ? 1 : 0

        Datadog::Statsd.instance.gauge('shoryuken.healthy', healthy)
        sleep 10
      end
    end
  end
end
```

## Related

- [[Lifecycle Events]] - Startup and shutdown hooks
- [[Signals]] - Understanding shutdown signals
- [[Deployment]] - Production deployment guides
