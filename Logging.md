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

See [available levels](https://ruby-doc.org/stdlib-2.3.1/libdoc/logger/rdoc/Logger/Severity.html).

```ruby
Shoryuken.logger.level = Logger::FATAL
```

## Logging to a file

```shell
shoryuken -L ./shoryuken.log
```