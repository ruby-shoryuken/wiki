The most common way to configure queues in Shoryuken is by defining them in the `shoryuken.yml`, as documented [here](https://github.com/phstc/shoryuken/wiki/Shoryuken-options#queues).

You can configure them by name, URL, or ARN.

The `shoryuken.yml` is not only a YAML file, before parsing it as a YAML, Shoryuken parses it as an ERB, which supports embedded Ruby.

```erb
queues:
  - <%= ENV['QUEUE_NAME'] %>
```

Another way to add a queue is programmatic with `Shoryuken.add_queue`. 

```ruby
Shoryuken.add_queue('queue-name', 1, 'default')

# queue-name = queue name
# 1 = priority 
# default = processing group name, if not using processing groups, use default. See https://github.com/phstc/shoryuken/wiki/Processing-Groups
```

For more information, [check out the source code](https://github.com/phstc/shoryuken/blob/5abc9ee69946010f47a96c8b01e69e70a2afdd4a/lib/shoryuken/options.rb#L48).
