Shoryuken fires process lifecycle events when starting up and shutting down:

```ruby
Shoryuken.configure_server do |config|
  config.on(:startup) do
    # called when it's about to start fetching messages, after validating queues, params etc
  end

  config.on(:quiet) do
    # called for soft shutdown kill -USR1, before calling shutdown
  end

  config.on(:dispatch) do
    # called every time it tries to fetch messages from SQS
  end

  config.on(:shutdown) do
    # called when shut-downing
  end

  config.on(:stopped) do
    # called after executor stopped
  end
end
```