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

## `on_start` and `on_stop`

Shoryuken also has `on_start` and `on_stop` (`Shoryuken.on_start { ... }`, or `config.on_start { ... }` inside `configure_server`). Prefer `config.on(:startup)` and `config.on(:shutdown)`: they keep several handlers, log a raised error as a warning instead of stopping the start, and work like the other events.

How `on_start` and `on_stop` differ from `config.on`:

- Each keeps one block. A second call replaces the first.
- The block gets no argument.
- The `on_start` block runs just before the `startup` handlers. The `on_stop` block runs just before the `shutdown` handlers.
- If the `on_start` block raises, the error is not caught: the `startup` handlers do not run and Shoryuken does not start. If the `on_stop` block raises, Shoryuken logs the error and the shutdown goes on.

## Utilization example

```ruby
Shoryuken.configure_server do |config|
  config.on(:utilization_update) do |event|
    utilization = event[:busy_processors].to_f / event[:max_processors]
    Shoryuken.logger.info "#{event[:group]} utilization: #{(utilization * 100).round}%"
  end
end
```
