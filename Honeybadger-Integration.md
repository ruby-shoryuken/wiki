The [honeybadger gem](https://github.com/honeybadger-io/honeybadger-ruby/) integrates with Shoryuken automatically to report failed jobs to [Honeybadger](https://www.honeybadger.io/).

To report a failure only after a number of receives, set `shoryuken.attempt_threshold` in your Honeybadger configuration. The gem compares it with the message's `ApproximateReceiveCount`.
