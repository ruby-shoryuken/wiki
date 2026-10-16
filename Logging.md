Shoryuken's default logger level is `INFO`.

## Logging in application
```ruby
Shoryuken.logger.info("Info-level log message")
```

## Changing to DEBUG (verbose)

You can pass `-v` while starting Shoryuken.

```shell
shoryuken -v
```

### Other levels 

See [available levels](https://ruby-doc.org/3.4/stdlibs/logger/Logger/Severity.html).

```ruby
Shoryuken.logger.level = Logger::FATAL
```

## Logging to a file

```shell
shoryuken -L ./shoryuken.log
```

## Using your own logger

```ruby
Shoryuken.logger = Rails.logger
```

## Log format

The default formatter writes the UTC time, the process ID, the thread ID and the level. While a worker runs, each line also has `<worker>/<queue>/<message_id>`.
