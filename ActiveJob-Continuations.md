Shoryuken supports ActiveJob Continuations (Rails 8.1+). A continuable job runs in steps. When the worker shuts down, the job stops at the next step boundary and is enqueued again, and the new run resumes where it stopped. For the job API, see Rails PR [#55127](https://github.com/rails/rails/pull/55127) and the Rails guides.

## What Shoryuken provides

Rails asks the queue adapter whether the worker is shutting down. The Shoryuken adapter answers with `stopping?`, which is `true` once the Shoryuken launcher starts to stop: after `USR1`, `TSTP`, `TERM` or `INT` (see [[Signals]]).

When the job is enqueued again to resume, the adapter sends it like any other job. A resume time in the past is delivered at once.

## Things to keep in mind

* A step must finish within the shutdown time. With `TERM`, `INT`, `USR1` and `TSTP`, busy workers are killed after `timeout` seconds (see [[Shoryuken options]]). Keep each step shorter than that.
* On a FIFO queue, a resume with a delay raises `Shoryuken::Errors::FifoDelayNotSupportedError`, because FIFO queues do not support per-message delays (see [[Rails Integration Active Job]]). By default, Rails resumes a job with `wait: 5.seconds`, so on a FIFO queue every resume raises. SQS then delivers the message again after the visibility timeout, and the job starts again from its first step: the steps that finished run again. For jobs on FIFO queues, resume without a delay:

  ```ruby
  class ImportJob < ApplicationJob
    include ActiveJob::Continuable

    self.resume_options = { wait: 0 }
  end
  ```
