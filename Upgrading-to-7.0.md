# Upgrading to Shoryuken 7.0

This guide covers the breaking changes and new features introduced in Shoryuken 7.0.

## Table of Contents

- [Ruby and Rails Version Requirements](#ruby-and-rails-version-requirements)
- [Breaking Changes](#breaking-changes)
  - [New Error Class Hierarchy](#new-error-class-hierarchy)
  - [Shoryuken::Shutdown Removed](#shoryukenshutdown-removed)
  - [FIFO Queue Delay Validation](#fifo-queue-delay-validation)
  - [Core Class Extensions Removed](#core-class-extensions-removed)
  - [Concurrent-Ruby Replaced](#concurrent-ruby-replaced)
- [New Features](#new-features)
  - [Bulk ActiveJob Enqueuing](#bulk-activejob-enqueuing)
  - [ActiveJob Continuations Support](#activejob-continuations-support)
  - [CurrentAttributes Persistence](#currentattributes-persistence)
  - [Fiber-Local Storage for Logging](#fiber-local-storage-for-logging)
- [Changes in 7.0.x Patch Releases](#changes-in-70x-patch-releases)
- [Upgrade Steps](#upgrade-steps)

## Ruby and Rails Version Requirements

### Minimum Ruby Version: 3.2.0

**Breaking:** Shoryuken 7.0 requires Ruby 3.2.0 or higher. Ruby 3.1 reached end-of-life in March 2025 and is no longer supported.

**Supported Ruby versions:**
- Ruby 3.2
- Ruby 3.3
- Ruby 3.4
- Ruby 4.0

**Action Required:** If you're using Ruby 3.1 or older, you must upgrade to Ruby 3.2+ or remain on Shoryuken 6.x.

**Note:** Shoryuken 7.0.x supports Ruby 3.2. Releases after 7.0.x require Ruby 3.3.0 or higher. On Ruby 3.2, upgrade Ruby or stay on Shoryuken 7.0.x.

### Minimum Rails Version: 7.2

**Breaking:** Shoryuken 7.0 requires Rails 7.2 or higher. Rails 7.0 and 7.1 reached end-of-life in April 2025.

**Supported Rails versions:**
- Rails 7.2
- Rails 8.0
- Rails 8.1

**Action Required:** If you're using Rails 7.1 or older, you must upgrade to Rails 7.2+ or remain on Shoryuken 6.x.

## Breaking Changes

### New Error Class Hierarchy

**Breaking:** Generic Ruby exceptions (`ArgumentError`, `RuntimeError`) have been replaced with domain-specific error classes under the `Shoryuken::Errors` module.

#### New Error Classes

All Shoryuken errors now inherit from `Shoryuken::Errors::BaseError`:

- `Shoryuken::Errors::QueueNotFoundError` - Raised when a queue doesn't exist or is inaccessible
- `Shoryuken::Errors::InvalidConfigurationError` - Raised for configuration validation failures
- `Shoryuken::Errors::InvalidWorkerRegistrationError` - Raised for worker registration conflicts
- `Shoryuken::Errors::InvalidPollingStrategyError` - Raised for invalid polling strategy configuration
- `Shoryuken::Errors::InvalidEventError` - Raised for invalid lifecycle event names
- `Shoryuken::Errors::InvalidDelayError` - Raised by `perform_in` and `perform_at` when a delay exceeds the SQS 15-minute maximum (the ActiveJob adapter still raises a `RuntimeError` for this)
- `Shoryuken::Errors::FifoDelayNotSupportedError` - Raised when a delay is used with a FIFO queue (since 7.0.1)
- `Shoryuken::Errors::InvalidArnError` - Raised for invalid ARN format

#### Migration Guide

**Before (Shoryuken 6.x):**
```ruby
begin
  Shoryuken::Client.queues(queue_name)
rescue ArgumentError => e
  # Handle error
end
```

**After (Shoryuken 7.0):**
```ruby
begin
  Shoryuken::Client.queues(queue_name)
rescue Shoryuken::Errors::QueueNotFoundError => e
  # Handle error
end
```

**Action Required:** Update your error handling to catch the new specific error classes. If you want to catch all Shoryuken errors, you can rescue `Shoryuken::Errors::BaseError`.

```ruby
begin
  # Shoryuken operations
rescue Shoryuken::Errors::BaseError => e
  # Handle all Shoryuken-specific errors
end
```

### Shoryuken::Shutdown Removed

**Breaking:** The `Shoryuken::Shutdown` class has been removed.

This class was unused since the Celluloid removal in 2016. It was originally used to raise on worker threads during hard shutdown but is no longer needed. The current shutdown flow uses `Interrupt` and executor-level shutdown instead.

**Action Required:** If your code references `Shoryuken::Shutdown`, remove those references. The shutdown behavior is now handled internally through the `Interrupt` exception and the launcher's shutdown lifecycle.

### FIFO Queue Delay Validation

**Breaking:** Using `delay` with FIFO queues now raises an error. In 7.0.0 it is an `ArgumentError`. Since 7.0.1 it is `Shoryuken::Errors::FifoDelayNotSupportedError` ([#957](https://github.com/ruby-shoryuken/shoryuken/pull/957)).

FIFO queues do not support per-message `DelaySeconds`. Previously, this would cause confusing AWS errors, especially with ActiveJob's `retry_on` (Rails 6.1+ defaults to `wait: 3.seconds`).

**Before (Shoryuken 6.x):**
```ruby
# Would silently fail or produce confusing AWS errors
MyJob.set(wait: 3.seconds).perform_later
```

**After (Shoryuken 7.0):**
```ruby
# Raises a clear error with guidance
MyJob.set(wait: 3.seconds).perform_later
# => Shoryuken::Errors::FifoDelayNotSupportedError: FIFO queue 'my_queue.fifo' does not support
#    per-message delays. When using ActiveJob retry_on with FIFO queues, set `wait: 0`.
```

**Action Required:**
1. If using FIFO queues with ActiveJob's `retry_on`, set `wait: 0`:
   ```ruby
   retry_on SomeException, wait: 0, attempts: 3
   ```

2. If you need delayed retries with FIFO queues, implement a custom retry strategy using SQS dead-letter queues or application-level retry logic. A dead-letter queue breaks the order of a message group, see [FIFO Queues](https://github.com/ruby-shoryuken/shoryuken/wiki/FIFO-Queues#failures-and-dead-letter-queues).

**Fixes:** [#924](https://github.com/ruby-shoryuken/shoryuken/issues/924)

### Core Class Extensions Removed

**Breaking:** All monkey-patching of core Ruby classes (`Hash` and `String`) has been removed.

Shoryuken no longer extends core classes. Instead, it provides utility modules:
- `Shoryuken::Helpers::HashUtils.deep_symbolize_keys` - Replaces `Hash#deep_symbolize_keys`
- `Shoryuken::Helpers::StringUtils.constantize` - Replaces `String#constantize`

**Action Required:** This should not affect most users, as these were internal APIs. If you were relying on Shoryuken adding these methods to core classes, you'll need to:

1. Use ActiveSupport directly if you need these methods globally
2. Use Shoryuken's utility methods directly:
   ```ruby
   Shoryuken::Helpers::HashUtils.deep_symbolize_keys(my_hash)
   Shoryuken::Helpers::StringUtils.constantize('MyClass')
   ```

### Concurrent-Ruby Replaced

**Breaking:** Internal concurrent data structures from `concurrent-ruby` have been replaced with pure Ruby implementations.

This is mostly an internal change, but if you were directly accessing these internal structures, they've been replaced:
- `Concurrent::AtomicFixnum` → `Shoryuken::Helpers::AtomicCounter`
- `Concurrent::AtomicBoolean` → `Shoryuken::Helpers::AtomicBoolean`
- `Concurrent::Hash` → `Shoryuken::Helpers::AtomicHash`

**Note:** The `concurrent-ruby` gem is still a dependency and used for other purposes (like the thread pool executor).

**Action Required:** If you were accessing Shoryuken's internal concurrent data structures (unlikely for most users), update your references to use the new helper classes.

## New Features

### Bulk ActiveJob Enqueuing

**New in Rails 7.1+:** Shoryuken now supports efficient bulk job enqueuing through `ActiveJob.perform_all_later`.

The `enqueue_all` method uses SQS's `send_message_batch` API to enqueue multiple jobs efficiently:
- Automatically batches jobs in groups of 10 (SQS limit)
- Groups jobs by queue name for efficient multi-queue handling

**Usage:**
```ruby
# Enqueue multiple jobs at once
ActiveJob.perform_all_later(
  MyJob.new(arg1),
  MyJob.new(arg2),
  AnotherJob.new(arg3)
)
```

This is significantly more efficient than enqueueing jobs individually.

See [[Bulk Enqueuing]] for details, including how it handles delayed jobs.

### ActiveJob Continuations Support

**New in Rails 8.1+:** Shoryuken now supports ActiveJob Continuations, allowing jobs to checkpoint their progress and resume after interruption.

When Shoryuken receives a shutdown signal, the `stopping?` method in the ActiveJob adapter signals to jobs that they should checkpoint and reschedule themselves.

**How it works:**
1. Shoryuken tracks shutdown state in the Launcher via the `stopping?` flag
2. Jobs can check if a shutdown is in progress and checkpoint their work
3. Jobs can reschedule themselves to resume after the interruption
4. Handles past timestamps correctly (SQS treats negative delays as immediate delivery)

**Usage:** Refer to Rails PR [#55127](https://github.com/rails/rails/pull/55127) for details on implementing job continuations.

See [[ActiveJob Continuations]].

### CurrentAttributes Persistence

**New:** Rails `ActiveSupport::CurrentAttributes` can now flow from enqueue to job execution.

See [[CurrentAttributes]] for details.

This feature automatically serializes current attributes into the job payload when enqueuing and restores them before execution.

**Setup:**
```ruby
# In an initializer (e.g., config/initializers/shoryuken.rb)
require 'shoryuken/active_job/current_attributes'

# Specify which CurrentAttributes classes to persist
Shoryuken::ActiveJob::CurrentAttributes.persist('MyApp::Current')
Shoryuken::ActiveJob::CurrentAttributes.persist('AnotherApp::Current')
```

**Example:**
```ruby
class Current < ActiveSupport::CurrentAttributes
  attribute :user_id, :request_id
end

# When enqueueing
Current.user_id = 123
Current.request_id = 'abc-def'
MyJob.perform_later(args)

# When job executes, these will be restored:
# Current.user_id => 123
# Current.request_id => 'abc-def'
```

**Note:** This feature is based on Sidekiq's approach and uses `ActiveJob::Arguments` for serialization.

### Fiber-Local Storage for Logging

**Enhancement:** Logging context now uses fiber-local storage instead of thread-local storage.

This ensures logging context doesn't leak between fibers in the same thread, which is important for async environments. Uses Ruby 3.2+ fiber-local storage API.

**Action Required:** None. This is a transparent improvement that ensures better isolation in concurrent scenarios.

### Other Enhancements

- **Zeitwerk Autoloading:** Replaces manual require statements with Zeitwerk-based autoloading, reducing startup overhead
- **SendMessageBatch Limit:** Increased to 1MB to align with AWS limits ([#864](https://github.com/ruby-shoryuken/shoryuken/pull/864))
- **Server-Side Logging:** Configurable server-side logging ([#844](https://github.com/ruby-shoryuken/shoryuken/pull/844))
- **Thread Priority:** Uses `-1` as thread priority ([#825](https://github.com/ruby-shoryuken/shoryuken/pull/825))
- **InlineExecutor:** Added support for `message_attributes` ([#835](https://github.com/ruby-shoryuken/shoryuken/pull/835))
- **Rails 7.2 Compatibility:** Added `enqueue_after_transaction_commit?` ([#777](https://github.com/ruby-shoryuken/shoryuken/pull/777))
- **YARD Documentation:** Comprehensive YARD documentation with 100% coverage
- **Trusted Publishing:** Improved gem security with trusted publishing ([#840](https://github.com/ruby-shoryuken/shoryuken/pull/840))

## Changes in 7.0.x Patch Releases

The 7.0.x patch releases are bug-fix releases. They also add the opt-in settings and options below, whose defaults keep the 7.0.0 behavior. One change is not opt-in: since 7.0.1 the FIFO delay error is `Shoryuken::Errors::FifoDelayNotSupportedError` instead of `ArgumentError`, so a `rescue ArgumentError` written for 7.0.0 no longer catches it. Rescue `Shoryuken::Errors::FifoDelayNotSupportedError` (or `Shoryuken::Errors::BaseError`) instead. The option validation in the next release after 7.0.4 is not opt-in either.

- **7.0.1:** The `non_retryable_exceptions` worker option deletes a message at once when the worker raises one of the listed exceptions, instead of retrying it ([#958](https://github.com/ruby-shoryuken/shoryuken/pull/958)). See [[Worker options]].
- **7.0.1:** The FIFO delay error is now `Shoryuken::Errors::FifoDelayNotSupportedError` (see [FIFO Queue Delay Validation](#fifo-queue-delay-validation)).
- **7.0.2:** A custom polling strategy can be set per processing group with `Shoryuken.add_group(group, concurrency, polling_strategy: MyStrategy)`. See [[Polling strategies]].
- **7.0.3:** `Shoryuken.active_job_fifo_message_deduplication = false` stops the ActiveJob adapter from generating a `message_deduplication_id` for FIFO queues. Default: `true`.
- **7.0.4:** `Shoryuken.fifo_message_deduplication = false` stops `perform_async`, `Queue#send_message` and `Queue#send_messages` from generating a `message_deduplication_id` (a SHA-256 of the body) for FIFO queues. Default: `true`.
- **Next release after 7.0.4:** `shoryuken start` validates its options before it loads queues and workers. A configuration that started before can now fail to start with `Shoryuken::Errors::InvalidConfigurationError`, for example `concurrency: 0` or a malformed `queues` entry. Fix the option that the error names by its key path (`groups.<name>.<key>` inside a group). An unknown key only logs a warning: check it for a typo. See [Option validation](https://github.com/ruby-shoryuken/shoryuken/wiki/Shoryuken-options#option-validation). Worker options are stricter: a worker whose `shoryuken_options` has an unknown key, for example a mistyped `auto_delte` or the formerly documented `sqs` key, now raises `Shoryuken::Errors::InvalidConfigurationError` while its class loads, so the worker does not load. Remove or fix the key that the error names (`<WorkerClass>.<key>`). See [Option validation](https://github.com/ruby-shoryuken/shoryuken/wiki/Worker-options#option-validation) in [[Worker options]].

With these settings at `true`, two sends of an identical body to a FIFO queue within the SQS 5-minute deduplication window are deduplicated, and SQS drops the second one. If you set them to `false`, give a `message_deduplication_id` yourself or enable `ContentBasedDeduplication` on the queue. See [[FIFO Queues]].

For the full list of fixes, see the [CHANGELOG](https://github.com/ruby-shoryuken/shoryuken/blob/main/CHANGELOG.md).

## Upgrade Steps

### 1. Check Ruby and Rails Versions

Ensure you're running:
- Ruby 3.2.0 or higher
- Rails 7.2 or higher (if using Rails)

```bash
ruby -v
# Should show 3.2.0 or higher

bundle exec rails -v
# Should show 7.2.0 or higher
```

### 2. Update Gemfile

```ruby
gem 'shoryuken', '~> 7.0'
```

Shoryuken 7.0 requires `aws-sdk-sqs` 1.66.0 or later.

### 3. Update Dependencies

```bash
bundle update shoryuken aws-sdk-sqs
```

### 4. Update Error Handling

Search your codebase for error handling around Shoryuken operations:

```bash
# Find potential error handling that needs updating
grep -r "rescue ArgumentError" app/
grep -r "rescue RuntimeError" app/
```

Update to use the new error classes from `Shoryuken::Errors`.

### 5. Fix FIFO Queue Delays (if applicable)

If using FIFO queues with ActiveJob, update any `retry_on` or `wait` configurations:

```ruby
# Before
retry_on SomeException, wait: 3.seconds

# After (for FIFO queues)
retry_on SomeException, wait: 0
```

### 6. Remove References to Shoryuken::Shutdown (if applicable)

Search for and remove any references:

```bash
grep -r "Shoryuken::Shutdown" app/
```

### 7. Test Your Application

Run your test suite to identify any issues:

```bash
bundle exec rspec
# or
bundle exec rake test
```

Test in a staging environment before deploying to production.

### 8. Optional: Enable CurrentAttributes Persistence

If you want to use CurrentAttributes persistence:

```ruby
# config/initializers/shoryuken.rb
require 'shoryuken/active_job/current_attributes'
Shoryuken::ActiveJob::CurrentAttributes.persist('YourApp::Current')
```

### 9. Optional: Use Bulk Enqueuing

Update code that enqueues multiple jobs to use `perform_all_later` (Rails 7.1+):

```ruby
# Before
jobs.each { |args| MyJob.perform_later(args) }

# After
ActiveJob.perform_all_later(*jobs.map { |args| MyJob.new(args) })
```

## Need Help?

If you encounter issues during the upgrade:

1. Check the [GitHub Issues](https://github.com/ruby-shoryuken/shoryuken/issues) for known problems
2. Review the [full CHANGELOG](https://github.com/ruby-shoryuken/shoryuken/blob/main/CHANGELOG.md)
3. Join the discussion on GitHub or open a new issue

## Summary

Shoryuken 7.0 modernizes the codebase with Ruby 3.2+ and Rails 7.2+ support, introduces a cleaner error hierarchy, removes deprecated code, and adds powerful new features like bulk enqueuing and ActiveJob continuations. The upgrade should be straightforward for most applications, with the main action items being:

1. Upgrade Ruby to 3.2+ and Rails to 7.2+
2. Update error handling to use new error classes
3. Fix FIFO queue delay configurations (if applicable)

Happy upgrading!
