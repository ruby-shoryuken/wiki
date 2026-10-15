# Shoryuken Wiki

Shoryuken is a super-efficient AWS SQS thread-based message processor for Ruby.

**Requirements:** Ruby 3.2+ | Rails 7.2+ | aws-sdk-sqs >= 1.66

---

## Getting Started

- [[Getting Started]] - Installation and basic setup
- [[Configure the AWS Client]] - AWS credentials configuration
- [[Complete Shoryuken Setup for Rails]] - Full Rails integration walkthrough

## Rails & ActiveJob

- [[Rails Integration Active Job]] - ActiveJob adapter setup and usage
- [[ActiveJob Continuations]] - Graceful job interruption for long-running jobs (Rails 8.1+)
- [[CurrentAttributes]] - Persist Rails CurrentAttributes across job execution
- [[Bulk Enqueuing]] - Efficient batch enqueuing with `perform_all_later`

## Workers & Queues

- [[Worker options]] - Worker configuration (`shoryuken_options`)
- [[Shoryuken options]] - Global configuration and CLI options
- [[Processing Groups]] - Group queues with separate concurrency
- [[FIFO Queues]] - FIFO queue support and configuration
- [[Queue URL]] - Multi-region and cross-account queue access

## Message Handling

- [[Sending a message]] - Enqueuing messages and jobs
- [[Retrying a message]] - Retry strategies and exponential backoff
- [[Middleware]] - Server middleware for cross-cutting concerns
- [[Lifecycle Events]] - Hooks for startup, shutdown, and processing events

## Polling & Performance

- [[Polling strategies]] - WeightedRoundRobin and StrictPriority
- [[Long Polling]] - Reduce SQS costs with long polling
- [[Receive Message options]] - Fine-tune SQS receive parameters

## Operations

- [[Deployment]] - Deploy to Heroku, ECS, Kubernetes, and more
- [[Signals]] - Process control signals (USR1, TSTP, TTIN)
- [[Health Checks]] - Health check API for load balancers and orchestrators

## Monitoring & Logging

- [[Logging]] - Configure logging and log levels
- [[Sentry.io Integration]] - Error tracking with Sentry
- [[Honeybadger Integration]] - Error tracking with Honeybadger

## Testing & Development

- [[Testing]] - Test strategies for Shoryuken workers
- [[Shoryuken Inline adapter]] - Synchronous execution for testing
- [[Using a local mock SQS server]] - LocalStack and moto for local development
- [[Enable hot reload for rails dev environment]] - Development auto-reload

## Security & IAM

- [[Amazon SQS IAM Policy for Shoryuken]] - Minimum required IAM permissions

## Migration & Upgrade

- [[Upgrade Guide 7.0]] - Upgrading to Shoryuken 7.0
- [[From Sidekiq to Shoryuken]] - Migration guide from Sidekiq
- [[Best Practices]] - Idempotency and reliability patterns

## Advanced Topics

- [[Worker registry]] - Custom worker registration
- [[Multiple ways of configuring queues in Shoryuken]] - Queue configuration options
- [[Scheduled Jobs]] - Running jobs on a schedule with EventBridge/Lambda

---

## Quick Links

- [GitHub Repository](https://github.com/ruby-shoryuken/shoryuken)
- [RubyGems](https://rubygems.org/gems/shoryuken)
- [CHANGELOG](https://github.com/ruby-shoryuken/shoryuken/blob/master/CHANGELOG.md)
