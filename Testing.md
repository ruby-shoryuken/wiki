# Testing

This guide covers testing strategies for Shoryuken workers and ActiveJob jobs.

## Testing Strategies

| Strategy | Best For | Pros | Cons |
|----------|----------|------|------|
| Unit tests | Worker logic | Fast, isolated | Doesn't test middleware |
| Inline adapter | Full integration | Tests middleware chain | Synchronous only |
| LocalStack | End-to-end | Real SQS behavior | Requires Docker |

---

## Unit Testing Workers

### RSpec

```ruby
# spec/workers/my_worker_spec.rb
require 'rails_helper'

RSpec.describe MyWorker do
  let(:sqs_msg) do
    double(
      message_id: 'fc754df7-9cc2-4c41-96ca-5996a44b771e',
      body: 'test',
      delete: nil
    )
  end
  let(:body) { { 'user_id' => 123 } }

  describe '#perform' do
    subject(:worker) { described_class.new }

    it 'processes the message' do
      user = create(:user, id: 123)

      worker.perform(sqs_msg, body)

      expect(user.reload.processed).to be true
    end

    it 'deletes the message after processing' do
      expect(sqs_msg).to receive(:delete)

      worker.perform(sqs_msg, body)
    end
  end
end
```

### Minitest

```ruby
# test/workers/my_worker_test.rb
require 'test_helper'

class MyWorkerTest < ActiveSupport::TestCase
  def setup
    @sqs_msg = Minitest::Mock.new
    @sqs_msg.expect :message_id, 'test-123'
    @sqs_msg.expect :body, 'test'
    @body = { 'user_id' => 123 }
  end

  test 'processes message correctly' do
    user = users(:one)

    MyWorker.new.perform(@sqs_msg, @body)

    assert user.reload.processed
  end
end
```

---

## Testing ActiveJob Jobs

### RSpec

```ruby
# spec/jobs/process_order_job_spec.rb
require 'rails_helper'

RSpec.describe ProcessOrderJob, type: :job do
  describe '#perform' do
    let(:order) { create(:order) }

    it 'processes the order' do
      described_class.perform_now(order)

      expect(order.reload.status).to eq('processed')
    end

    it 'enqueues to the correct queue' do
      expect {
        described_class.perform_later(order)
      }.to have_enqueued_job.on_queue('orders')
    end
  end
end
```

### ActiveJob Test Helpers

```ruby
# spec/rails_helper.rb
RSpec.configure do |config|
  config.include ActiveJob::TestHelper
end

# In your spec
RSpec.describe OrdersController, type: :controller do
  describe 'POST #create' do
    it 'enqueues a processing job' do
      expect {
        post :create, params: { order: valid_attributes }
      }.to have_enqueued_job(ProcessOrderJob)
    end
  end
end
```

### Testing with perform_enqueued_jobs

```ruby
RSpec.describe 'Order processing', type: :feature do
  it 'processes orders end-to-end' do
    order = create(:order)

    perform_enqueued_jobs do
      ProcessOrderJob.perform_later(order)
    end

    expect(order.reload.status).to eq('completed')
  end
end
```

---

## Testing CurrentAttributes

```ruby
# spec/jobs/audit_job_spec.rb
require 'rails_helper'

RSpec.describe AuditJob, type: :job do
  let(:user) { create(:user) }

  before do
    Current.user = user
  end

  after do
    Current.reset
  end

  it 'uses current attributes' do
    expect {
      described_class.perform_now(action: 'test')
    }.to change(AuditLog, :count).by(1)

    expect(AuditLog.last.user).to eq(user)
  end
end
```

---

## Testing ActiveJob Continuations

```ruby
# spec/jobs/data_export_job_spec.rb
require 'rails_helper'

RSpec.describe DataExportJob, type: :job do
  let(:export) { create(:export, :with_records) }

  describe 'when stopping? is false' do
    it 'processes all records' do
      described_class.perform_now(export)

      expect(export.reload.status).to eq('completed')
    end
  end

  describe 'when stopping? becomes true' do
    it 'checkpoints and re-enqueues' do
      job = described_class.new(export)

      # Simulate shutdown after processing 2 records
      call_count = 0
      allow(job).to receive(:stopping?) do
        call_count += 1
        call_count > 2
      end

      expect {
        job.perform_now
      }.to have_enqueued_job(described_class)

      expect(export.reload.last_processed_id).to be_present
    end
  end
end
```

---

## Inline Adapter

The inline adapter processes messages synchronously, useful for testing:

### For Shoryuken Workers

```ruby
# spec/rails_helper.rb or test environment
Shoryuken.configure_server do |config|
  config.sqs_client = Shoryuken::InlineExecutor
end

# Or per-test
RSpec.describe 'Integration', type: :feature do
  around do |example|
    original = Shoryuken.sqs_client
    Shoryuken.sqs_client = Shoryuken::InlineExecutor
    example.run
    Shoryuken.sqs_client = original
  end

  it 'processes jobs inline' do
    MyWorker.perform_async('test')
    # Job runs immediately
  end
end
```

### For ActiveJob

```ruby
# config/environments/test.rb
config.active_job.queue_adapter = :inline
```

Or use ActiveJob's test adapter:

```ruby
# config/environments/test.rb
config.active_job.queue_adapter = :test
```

---

## Mocking SQS Client

```ruby
# spec/workers/notification_worker_spec.rb
RSpec.describe NotificationWorker do
  let(:queue) { instance_double(Shoryuken::Queue) }

  before do
    allow(Shoryuken::Client).to receive(:queues)
      .with('notifications')
      .and_return(queue)
  end

  it 'sends notification to queue' do
    expect(queue).to receive(:send_message).with(
      hash_including(message_body: hash_including('type' => 'alert'))
    )

    described_class.new.perform(sqs_msg, body)
  end
end
```

---

## Testing with LocalStack

For full integration tests with real SQS behavior, use LocalStack:

```ruby
# spec/support/localstack.rb
RSpec.configure do |config|
  config.before(:suite) do
    Shoryuken.configure_client do |c|
      c.sqs_client = Aws::SQS::Client.new(
        endpoint: 'http://localhost:4566',
        region: 'us-east-1',
        access_key_id: 'test',
        secret_access_key: 'test'
      )
    end

    # Create test queues
    client = Shoryuken.sqs_client
    client.create_queue(queue_name: 'test-queue')
  end
end
```

See [[Using a local mock SQS server]] for LocalStack setup.

---

## Testing Middleware

```ruby
# spec/middleware/logging_middleware_spec.rb
RSpec.describe LoggingMiddleware do
  let(:worker) { MyWorker.new }
  let(:queue) { 'default' }
  let(:sqs_msg) { double(message_id: 'test-123') }
  let(:body) { { 'data' => 'test' } }

  it 'logs before and after processing' do
    expect(Rails.logger).to receive(:info).with(/Starting/)
    expect(Rails.logger).to receive(:info).with(/Completed/)

    described_class.new.call(worker, queue, sqs_msg, body) do
      # Worker perform would run here
    end
  end

  it 'logs errors' do
    expect(Rails.logger).to receive(:error).with(/Failed/)

    expect {
      described_class.new.call(worker, queue, sqs_msg, body) do
        raise 'Test error'
      end
    }.to raise_error('Test error')
  end
end
```

---

## CI Configuration

### GitHub Actions

```yaml
# .github/workflows/test.yml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      localstack:
        image: localstack/localstack
        ports:
          - 4566:4566
        env:
          SERVICES: sqs

    steps:
      - uses: actions/checkout@v4
      - uses: ruby/setup-ruby@v1
        with:
          bundler-cache: true

      - name: Create SQS queues
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

---

## Best Practices

1. **Isolate worker logic** - Keep perform methods simple, extract complex logic to services
2. **Use factories** - Create realistic test data with FactoryBot or fixtures
3. **Test edge cases** - Empty bodies, invalid data, network errors
4. **Mock external services** - Use VCR or WebMock for HTTP calls
5. **Test idempotency** - Ensure workers handle duplicate messages correctly

---

## Related

- [[Shoryuken Inline adapter]] - Synchronous execution
- [[Using a local mock SQS server]] - LocalStack setup
- [[CurrentAttributes]] - Testing with CurrentAttributes
- [[ActiveJob Continuations]] - Testing job interruption
