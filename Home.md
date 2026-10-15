Shoryuken is a super-efficient AWS SQS thread-based message processor for Ruby.

**Requirements (7.0):** Ruby 3.2+ | Rails 7.2+ (for ActiveJob) | aws-sdk-sqs >= 1.66

## Getting started

- [[Getting Started]] - Installation and basic setup
- [[Configure the AWS Client]] - AWS credentials configuration
- [[Complete Shoryuken Setup for Rails 5 with ActiveJob]] - Rails integration walkthrough

## Rails & ActiveJob

- [[Rails Integration Active Job]] - ActiveJob adapter setup and usage
- [[CurrentAttributes]] - Carry Rails CurrentAttributes into jobs

## Workers & queues

- [[Worker options]] - Worker configuration (`shoryuken_options`)
- [[Shoryuken options]] - Global configuration and CLI options
- [[Processing Groups]] - Group queues with separate concurrency
- [[FIFO Queues]] - FIFO queue support and configuration
- [[Queue URL]] - Configure queues by URL
- [[Multiple ways of configuring queues in Shoryuken]] - Queue configuration options
- [[Worker registry]] - Custom worker registration

## Message handling

- [[Sending a message]] - Enqueuing messages and jobs
- [[Retrying a message]] - Retry strategies and exponential backoff
- [[Middleware]] - Server middleware for cross-cutting concerns
- [[Lifecycle Events]] - Hooks for startup, shutdown and other events
- [[Scheduled Jobs]] - Recurring jobs with AWS services

## Polling

- [[Polling strategies]] - WeightedRoundRobin and StrictPriority
- [[Long Polling]] - Reduce SQS costs with long polling
- [[Receive Message options]] - Fine-tune SQS receive parameters

## Operations

- [[Deployment]] - Heroku, Capistrano, OpsWorks, Elastic Beanstalk and systemd
- [[AWS Beanstalk Config]] - Run Shoryuken on Elastic Beanstalk
- [[Signals]] - Process control signals (TTIN, TSTP, USR1, TERM)
- [[Amazon SQS IAM Policy for Shoryuken]] - IAM permissions
- [[Shoryuken Serverless - SQS as an Event Source for Lambdas]]

## Monitoring & logging

- [[Logging]] - Configure logging and log levels
- [[Sentry.io Integration]] - Error tracking with Sentry
- [[Honeybadger Integration]] - Error tracking with Honeybadger

## Testing & development

- [[Testing]] - Test strategies for Shoryuken workers
- [[Shoryuken Inline adapter]] - Synchronous execution for testing
- [[Using a local mock SQS server]] - Local development without real SQS
- [[Enable hot reload for rails dev environment]] - Development auto-reload

## Migration & upgrade

- [[Upgrading to 7.0]] - Upgrading to Shoryuken 7.0
- [[From Sidekiq to Shoryuken]] - Migration guide from Sidekiq
- [[Best Practices]] - Idempotency and reliability patterns

## Troubleshooting

- [[ERROR: No worker found for queue]]

## Quick links

- [GitHub Repository](https://github.com/ruby-shoryuken/shoryuken)
- [RubyGems](https://rubygems.org/gems/shoryuken)
- [CHANGELOG](https://github.com/ruby-shoryuken/shoryuken/blob/main/CHANGELOG.md)
