# Upgrading to Shoryuken 7.0

This guide covers the changes required when upgrading from Shoryuken 6.x to 7.0.

## Requirements

Shoryuken 7.0 requires:

- **Ruby 3.2+** (dropped support for Ruby 3.1 and earlier)
- **Rails 7.2+** (dropped support for Rails 7.0 and 7.1)
- **aws-sdk-sqs >= 1.66**

If you cannot upgrade to these versions, remain on Shoryuken 6.x.

## Breaking Changes

### 1. Ruby and Rails Version Requirements

Shoryuken 7.0 drops support for older Ruby and Rails versions:

| Shoryuken | Ruby | Rails |
|-----------|------|-------|
| 7.0+ | 3.2, 3.3, 3.4 | 7.2, 8.0, 8.1 |
| 6.x | 2.7+ | 6.0+ |

### 2. Removed concurrent-ruby Dependency

Shoryuken 7.0 removes the `concurrent-ruby` dependency for core atomic operations. The gem now uses pure Ruby implementations:

- `Concurrent::AtomicFixnum` → `Shoryuken::Helpers::AtomicCounter`
- `Concurrent::AtomicBoolean` → `Shoryuken::Helpers::AtomicBoolean`
- `Concurrent::Hash` → `Shoryuken::Helpers::AtomicHash`

**Impact**: This change is transparent for most users. Your code should work without modification unless you were directly using these concurrent-ruby classes through Shoryuken's internals.

**Note**: The `concurrent-ruby` gem is still used by `ShoryukenConcurrentSendAdapter` for `Concurrent::Promises`.

### 3. Removed Core Extensions

Shoryuken 7.0 removes monkey-patching of Ruby core classes. If your code relied on these extensions being globally available, you'll need to update it:

**Before (6.x):**
```ruby
# Hash and String were monkey-patched with these methods
some_hash.deep_symbolize_keys
"SomeClass".constantize
```

**After (7.0):**
```ruby
# Use the helper modules directly
Shoryuken::Helpers::HashUtils.deep_symbolize_keys(some_hash)
Shoryuken::Helpers::StringUtils.constantize("SomeClass")
```

### 4. Zeitwerk Autoloading

Shoryuken 7.0 uses Zeitwerk for autoloading. This change is mostly transparent but affects how polling strategy files are organized:

- `Shoryuken::Polling::BaseStrategy` (was nested in polling.rb)
- `Shoryuken::Polling::QueueConfiguration` (was nested in polling.rb)

**Impact**: If you have custom polling strategies that inherit from Shoryuken classes, ensure your `require` statements are updated.

### 5. aws-sdk-sqs Version Requirement

The minimum required version of `aws-sdk-sqs` is now 1.66. Update your Gemfile:

```ruby
gem 'aws-sdk-sqs', '>= 1.66'
```

### 6. SendMessageBatch Size Limit

The `SendMessageBatch` size limit has been increased from 256KB to 1MB, aligning with AWS SQS limits. This allows for larger batch operations.

## New Features

Shoryuken 7.0 introduces several new features. See the individual wiki pages for details:

- [[ActiveJob Continuations]] - Graceful job interruption (Rails 8.1+)
- [[CurrentAttributes]] - Persist Rails CurrentAttributes across job execution
- [[Bulk Enqueuing]] - Efficient batch enqueuing with `perform_all_later`
- [[Health Checks]] - Health check API via `launcher.healthy?`
- New `utilization_update` lifecycle event

## Upgrade Checklist

1. **Update Ruby version** to 3.2 or later
2. **Update Rails version** to 7.2 or later
3. **Update Gemfile dependencies:**
   ```ruby
   gem 'shoryuken', '~> 7.0'
   gem 'aws-sdk-sqs', '>= 1.66'
   ```
4. **Run `bundle update shoryuken aws-sdk-sqs`**
5. **Search for core extension usage** in your codebase:
   - `deep_symbolize_keys` calls on strings/hashes that might rely on Shoryuken's patches
   - `constantize` calls on strings
6. **Test your workers** in development/staging before deploying
7. **Review custom polling strategies** for compatibility with Zeitwerk autoloading

## Step-by-Step Migration

### For Rails Applications

1. Update your `Gemfile`:
   ```ruby
   # Gemfile
   gem 'shoryuken', '~> 7.0'
   ```

2. Update Rails queue adapter configuration (if needed):
   ```ruby
   # config/application.rb
   config.active_job.queue_adapter = :shoryuken
   ```

3. Workers in `app/jobs` are auto-loaded via Zeitwerk - no changes needed.

4. If using `ActiveJob::Base` directly, update to `ApplicationJob`:
   ```ruby
   # Before
   class MyJob < ActiveJob::Base
   end

   # After
   class MyJob < ApplicationJob
   end
   ```

### For Standalone Shoryuken (Non-Rails)

1. Update your `Gemfile`:
   ```ruby
   gem 'shoryuken', '~> 7.0'
   ```

2. Ensure workers are required correctly. With Zeitwerk, files must match class names:
   ```
   lib/
     my_worker.rb  # defines MyWorker
   ```

3. Start Shoryuken with your configuration:
   ```bash
   bundle exec shoryuken -r ./lib/my_worker.rb -C config/shoryuken.yml
   ```

## Getting Help

If you encounter issues during the upgrade:

1. Check the [GitHub Issues](https://github.com/ruby-shoryuken/shoryuken/issues)
2. Review the full [CHANGELOG](https://github.com/ruby-shoryuken/shoryuken/blob/master/CHANGELOG.md)
