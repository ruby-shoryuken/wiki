`Shoryuken.healthy?` returns `true` when the Shoryuken process is running and every processing group has a running manager. Use it in a liveness check.

```ruby
Shoryuken.healthy? # => true or false
```

It returns `false`:

* before the launcher has started,
* when a processing group's manager stopped after a fatal dispatch error (the error is logged as `Manager failed: ...`).

In the `shoryuken` CLI process, a fatal dispatch error also sends USR1 to the process, so it shuts down gracefully and your process supervisor can restart it (see [[Signals]]).

`Shoryuken.healthy?` reads the CLI runner's launcher (`Shoryuken::Runner.instance.launcher`). If you start a `Shoryuken::Launcher` yourself inside another process, call `healthy?` on that launcher instead.

## HTTP endpoint

Shoryuken has no HTTP server. To expose the check to a load balancer or an orchestrator, start a small server from a [lifecycle event](https://github.com/ruby-shoryuken/shoryuken/wiki/Lifecycle-Events). For example with WEBrick (add the `webrick` gem):

```ruby
# config/initializers/shoryuken.rb
require 'webrick'

Shoryuken.configure_server do |config|
  config.on(:startup) do
    server = WEBrick::HTTPServer.new(Port: 3001, AccessLog: [], Logger: WEBrick::Log.new(File::NULL))

    server.mount_proc '/health' do |_req, res|
      res.status = Shoryuken.healthy? ? 200 : 503
    end

    Thread.new { server.start }
  end
end
```
