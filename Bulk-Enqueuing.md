Since 7.0, the Shoryuken ActiveJob adapter supports `ActiveJob.perform_all_later`. It sends the jobs with the SQS `SendMessageBatch` API instead of one `SendMessage` call per job.

```ruby
jobs = users.map { |user| NotifyUserJob.new(user.id) }
ActiveJob.perform_all_later(jobs)
```

## How it works

* Jobs are grouped by queue name, and each group is sent in batches of up to 10 messages (the SQS batch limit).
* After the call, `job.successfully_enqueued?` tells you whether SQS accepted each job. Failed entries are logged.
* FIFO deduplication IDs and `.set` parameters (for example `message_group_id`) apply as for `perform_later`. See [[Rails Integration Active Job]].

## Delayed jobs

Bulk enqueuing sends every message without a delay: a `wait` or `wait_until` set on a job is not applied. Enqueue delayed jobs with `perform_later`.

```ruby
NotifyUserJob.set(wait: 5.minutes).perform_later(user.id)
```
