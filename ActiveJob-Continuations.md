# ActiveJob Continuations

ActiveJob Continuations allow long-running jobs to gracefully checkpoint their progress during shutdown and resume after restart.

**Requires:** Rails 8.1+

## Overview

When Shoryuken receives a shutdown signal (SIGTERM, SIGINT), it sets a `stopping?` flag. Jobs can check this flag and:

1. Save their current progress
2. Re-enqueue themselves to continue later
3. Exit gracefully

This prevents jobs from being interrupted mid-process and losing work.

## How It Works

```
┌─────────────────────────────────────────────────────────────┐
│                    Job Execution Flow                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Job starts → Process items → Check stopping? → Continue    │
│                     │                  │                     │
│                     ▼                  ▼                     │
│              Process more        Save progress               │
│                     │            Re-enqueue job              │
│                     ▼            Exit gracefully             │
│                Complete                                      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Basic Usage

```ruby
class DataExportJob < ApplicationJob
  queue_as :exports

  def perform(export)
    export.records_to_process.find_each do |record|
      # Check if Shoryuken is shutting down
      if stopping?
        # Checkpoint progress
        export.update!(last_processed_id: record.id)

        # Re-enqueue to continue from checkpoint
        self.class.perform_later(export)
        return
      end

      process_record(record)
    end

    export.mark_completed!
  end

  private

  def process_record(record)
    # Your processing logic
  end
end
```

## The `stopping?` Method

The `stopping?` method returns `true` when:

1. Shoryuken receives `SIGTERM` (graceful shutdown)
2. Shoryuken receives `SIGINT` (Ctrl+C)
3. The `stop` or `stop!` method is called on the launcher

### Implementation Details

Internally, Shoryuken tracks the stopping state in the `Launcher`:

```ruby
# In the adapter
def stopping?
  launcher = Shoryuken::Runner.instance.launcher
  launcher&.stopping? || false
end
```

This state is propagated to all ActiveJob jobs via Rails' Continuations API.

## Patterns

### Batch Processing with Checkpoints

```ruby
class BulkImportJob < ApplicationJob
  BATCH_SIZE = 100

  def perform(import, offset: 0)
    records = import.source_records.offset(offset).limit(BATCH_SIZE)

    records.each_with_index do |record, index|
      if stopping?
        # Save checkpoint and re-enqueue
        self.class.perform_later(import, offset: offset + index)
        return
      end

      import_record(record)
    end

    if records.size == BATCH_SIZE
      # More records to process
      self.class.perform_later(import, offset: offset + BATCH_SIZE)
    else
      import.mark_completed!
    end
  end
end
```

### Progress Tracking

```ruby
class VideoTranscodeJob < ApplicationJob
  def perform(video, start_frame: 0)
    total_frames = video.frame_count
    current_frame = start_frame

    while current_frame < total_frames
      if stopping?
        video.update!(transcoding_progress: current_frame)
        self.class.perform_later(video, start_frame: current_frame)
        return
      end

      transcode_frame(video, current_frame)
      current_frame += 1

      # Update progress periodically
      if current_frame % 1000 == 0
        video.update!(transcoding_progress: current_frame)
      end
    end

    video.finalize_transcode!
  end
end
```

### External API Pagination

```ruby
class SyncUsersJob < ApplicationJob
  def perform(page: 1)
    response = ExternalApi.fetch_users(page: page)

    response.users.each_with_index do |user_data, index|
      if stopping?
        # Can't checkpoint mid-page, re-enqueue current page
        self.class.perform_later(page: page)
        return
      end

      User.upsert(user_data)
    end

    if response.has_more_pages?
      self.class.perform_later(page: page + 1)
    end
  end
end
```

## Best Practices

### 1. Check `stopping?` at Natural Boundaries

Check at logical breakpoints in your processing:

```ruby
# Good: Check between records
records.each do |record|
  return if stopping?
  process(record)
end

# Avoid: Checking too frequently adds overhead
# Bad
data.each_char do |char|
  return if stopping?  # Too granular
  process(char)
end
```

### 2. Keep Checkpoint Data Small

Store only essential progress information:

```ruby
# Good: Store IDs and offsets
export.update!(last_processed_id: record.id)

# Avoid: Storing large data structures
export.update!(remaining_records: unprocessed_records.to_json)  # Bad
```

### 3. Make Jobs Resumable

Design jobs to resume from checkpoints:

```ruby
class ReportJob < ApplicationJob
  def perform(report, last_id: nil)
    scope = report.data_source
    scope = scope.where('id > ?', last_id) if last_id

    scope.find_each do |record|
      if stopping?
        self.class.perform_later(report, last_id: record.id)
        return
      end

      process(record)
    end
  end
end
```

### 4. Handle Idempotency

Ensure processing is idempotent in case a job restarts:

```ruby
class ProcessOrderJob < ApplicationJob
  def perform(order, processed_items: [])
    order.line_items.each do |item|
      next if processed_items.include?(item.id)

      if stopping?
        self.class.perform_later(order, processed_items: processed_items)
        return
      end

      fulfill_item(item)
      processed_items << item.id
    end

    order.mark_fulfilled!
  end
end
```

## Testing Continuations

### RSpec Example

```ruby
RSpec.describe DataExportJob do
  describe '#perform' do
    let(:export) { create(:export, :with_records) }

    context 'when stopping? is false' do
      it 'processes all records' do
        expect { described_class.perform_now(export) }
          .to change { export.reload.status }.to('completed')
      end
    end

    context 'when stopping? becomes true' do
      before do
        allow_any_instance_of(described_class)
          .to receive(:stopping?).and_return(false, false, true)
      end

      it 'checkpoints progress and re-enqueues' do
        expect(described_class).to receive(:perform_later)
          .with(export)

        described_class.perform_now(export)

        expect(export.reload.last_processed_id).to be_present
      end
    end
  end
end
```

### Minitest Example

```ruby
class DataExportJobTest < ActiveJob::TestCase
  test 'checkpoints and re-enqueues on shutdown' do
    export = exports(:large_export)

    # Simulate stopping after 2 records
    job = DataExportJob.new(export)
    call_count = 0
    job.define_singleton_method(:stopping?) do
      call_count += 1
      call_count > 2
    end

    job.perform_now

    assert_enqueued_jobs 1, only: DataExportJob
    assert_not_nil export.reload.last_processed_id
  end
end
```

## Limitations

### SQS Delay Limit

When re-enqueueing, be aware of SQS's 15-minute delay limit:

```ruby
# This works
self.class.set(wait: 5.minutes).perform_later(args)

# This raises an error
self.class.set(wait: 20.minutes).perform_later(args)  # > 15 minutes
```

For longer delays, use EventBridge Scheduler or a similar service.

### Visibility Timeout

Ensure your visibility timeout is long enough for checkpoint + re-enqueue operations:

```yaml
# config/shoryuken.yml
queues:
  - [exports, 1]

# Set visibility_timeout on the SQS queue to accommodate processing time
```

## Related

- [[Rails Integration Active Job]] - Basic ActiveJob setup
- [[Signals]] - Understanding shutdown signals
- [[Lifecycle Events]] - Hooks for shutdown events
- [Rails PR #55127](https://github.com/rails/rails/pull/55127) - Original Rails implementation
