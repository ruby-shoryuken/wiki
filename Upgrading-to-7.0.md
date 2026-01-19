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
- [Upgrade Steps](#upgrade-steps)

## Ruby and Rails Version Requirements

### Minimum Ruby Version: 3.2.0

**Breaking:** Shoryuken 7.0 requires Ruby 3.2.0 or higher. Ruby 3.1 reached end-of-life in March 2025 and is no longer supported.

**Supported Ruby versions:**
- Ruby 3.2
- Ruby 3.3
- Ruby 3.4

**Action Required:** If you're using Ruby 3.1 or older, you must upgrade to Ruby 3.2+ or remain on Shoryuken 6.x.

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
- `Shoryuken::Errors::InvalidDelayError` - Raised when delays exceed SQS 15-minute maximum
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

**Breaking:** Using `delay` with FIFO queues now raises an `ArgumentError`.

FIFO queues do not support per-message `DelaySeconds`. Previously, this would cause confusing AWS errors, especially with ActiveJob's `retry_on` (Rails 6.1+ defaults to `wait: 3.seconds`).

**Before (Shoryuken 6.x):**
```ruby
# Would silently fail or produce confusing AWS errors
MyJob.set(wait: 3.seconds).perform_later
```

**After (Shoryuken 7.0):**
```ruby
# Raises clear ArgumentError with guidance
MyJob.set(wait: 3.seconds).perform_later
# => ArgumentError: FIFO queues do not support DelaySeconds. Use wait: 0
```

**Action Required:**
1. If using FIFO queues with ActiveJob's `retry_on`, set `wait: 0`:
   ```ruby
   retry_on SomeException, wait: 0, attempts: 3
   ```

2. If you need delayed retries with FIFO queues, implement a custom retry strategy using SQS dead-letter queues or application-level retry logic.

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

### ActiveJob Continuations Support

**New in Rails 8.1+:** Shoryuken now supports ActiveJob Continuations, allowing jobs to checkpoint their progress and resume after interruption.

When Shoryuken receives a shutdown signal, the `stopping?` method in the ActiveJob adapter signals to jobs that they should checkpoint and reschedule themselves.

**How it works:**
1. Shoryuken tracks shutdown state in the Launcher via the `stopping?` flag
2. Jobs can check if a shutdown is in progress and checkpoint their work
3. Jobs can reschedule themselves to resume after the interruption
4. Handles past timestamps correctly (SQS treats negative delays as immediate delivery)

**Usage:** Refer to Rails PR [#55127](https://github.com/rails/rails/pull/55127) for details on implementing job continuations.

### CurrentAttributes Persistence

**New:** Rails `ActiveSupport::CurrentAttributes` can now flow from enqueue to job execution.

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

### 3. Update Dependencies

```bash
bundle update shoryuken
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
