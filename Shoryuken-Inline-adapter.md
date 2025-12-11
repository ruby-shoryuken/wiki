Shoryuken also supports inline jobs, which can be useful for testing and development environments. When using the inline adapter `perform_async` and `perform_in` execute the workers right away without going to SQS.

*Note:* Shoryuken Inline Adapter is only available in Shoryuken workers. It isn't supported in [Active Job jobs](https://github.com/phstc/shoryuken/wiki/Rails-Integration-Active-Job). For Active Job refer to [Active Job Inline adapter
](http://api.rubyonrails.org/classes/ActiveJob/QueueAdapters/InlineAdapter.html).

For enabling inline adapter you need to change the `worker_executor` as follows:

```ruby
Shoryuken.worker_executor = Shoryuken::Worker::InlineExecutor
```

For returning to the default one (the one that goes to SQS):

```ruby
Shoryuken.worker_executor = Shoryuken::Worker::DefaultExecutor
```

To run all jobs `inline` in `test` just add this to `config/environments/test.rb` `Shoryuken.worker_executor = Shoryuken::Worker::InlineExecutor`
