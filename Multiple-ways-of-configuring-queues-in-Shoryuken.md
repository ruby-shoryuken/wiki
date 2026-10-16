The most common way to configure queues in Shoryuken is by defining them in the `shoryuken.yml`, as documented [here](https://github.com/ruby-shoryuken/shoryuken/wiki/Shoryuken-options#queues).

You can configure them by name, URL, or ARN (see [[Queue URL]]).

Shoryuken evaluates `shoryuken.yml` with ERB before it parses the YAML. See [ERB](https://github.com/ruby-shoryuken/shoryuken/wiki/Shoryuken-options#erb).

Another way to add a queue is programmatic, with `Shoryuken.add_group` and `Shoryuken.add_queue`. See [Adding groups and queues in code](https://github.com/ruby-shoryuken/shoryuken/wiki/Shoryuken-options#adding-groups-and-queues-in-code). `add_queue` needs the group to exist, and `add_group` does not change a group that already exists ([source](https://github.com/ruby-shoryuken/shoryuken/blob/895472b/lib/shoryuken/options.rb#L82-L107)).
