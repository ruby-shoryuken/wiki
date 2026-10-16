Shoryuken fires process lifecycle events when starting up and shutting down:

```ruby
Shoryuken.configure_server do |config|
  config.on(:startup) do
    # called when it's about to start fetching messages, after validating queues, params etc
  end

  config.on(:quiet) do
    # called for soft shutdown (kill -USR1 or kill -TSTP), before calling shutdown
  end

  config.on(:dispatch) do |event|
    # called every time it tries to fetch messages from SQS
    # event[:queue_name] is the queue it fetches from
  end

  config.on(:utilization_update) do |event|
    # called when a processing group starts and when its number of busy processors changes
    # event[:group], event[:max_processors], event[:busy_processors]
  end

  config.on(:shutdown) do
    # called when shut-downing
  end

  config.on(:stopped) do
    # called after executor stopped
  end
end
```

Each handler receives one Hash argument. It is empty for `startup`, `quiet`, `shutdown` and `stopped`.

Which events a signal fires is listed in [[Signals]].

## Several handlers for one event

You can register more than one handler for an event. Shoryuken calls them in the order you registered them, except `quiet` and `shutdown`, which call them in reverse order.

An error raised by a handler is logged as a warning and does not stop the other handlers.

`config.on` raises `Shoryuken::Errors::InvalidEventError` when the event name is not a Symbol or not one of the events above.

## Utilization example

```ruby
Shoryuken.configure_server do |config|
  config.on(:utilization_update) do |event|
    utilization = event[:busy_processors].to_f / event[:max_processors]
    Shoryuken.logger.info "#{event[:group]} utilization: #{(utilization * 100).round}%"
  end
end
```
