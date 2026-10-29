The Shoryuken ActiveJob adapter supports `ActiveJob.perform_all_later`. It sends the jobs with the SQS `SendMessageBatch` API instead of one `SendMessage` call per job.

```ruby
jobs = users.map { |user| NotifyUserJob.new(user.id) }
ActiveJob.perform_all_later(jobs)
```

## How it works

* Jobs are grouped by queue name, and each group is sent in batches of up to 10 messages (the SQS batch limit).
* After the call, `job.successfully_enqueued?` tells you whether SQS accepted each job. Failed entries are logged.
* For FIFO queues, the adapter generates the `message_deduplication_id` as for `perform_later`. See [[Rails Integration Active Job]].

## SQS parameters

The SQS parameters you set with `.set` (`message_group_id`, `message_deduplication_id`, `message_attributes` and `message_system_attributes`) do not apply to bulk enqueuing. `JobClass.set(...)` has no `perform_all_later`, and `job.set(...)` on a job instance ignores these keys. On a FIFO queue, such a job goes to the default message group `ShoryukenMessage` (see [[FIFO Queues]]).

To send them in bulk, put them in each job's `sqs_send_message_parameters` before the call:

```ruby
jobs = users.map do |user|
  job = NotifyUserJob.new(user.id)
  job.sqs_send_message_parameters[:message_group_id] = "user-#{user.id}"
  job
end
ActiveJob.perform_all_later(jobs)
```

A `message_deduplication_id` set this way replaces the generated one.

## Delayed jobs

Bulk enqueuing sends every message without a delay: a `wait` or `wait_until` set on a job is not applied. Enqueue delayed jobs with `perform_later`.

```ruby
NotifyUserJob.set(wait: 5.minutes).perform_later(user.id)
```
