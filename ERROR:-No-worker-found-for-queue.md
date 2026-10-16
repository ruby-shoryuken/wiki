If you use standard Shoryuken workers by `include Shoryuken::Worker`, and do not use ActiveJob, [a commit in version 3.2.3](https://github.com/ruby-shoryuken/shoryuken/commit/bbfa813939f88e4e64beb178df85d9868bbff15f) stopped eager loading the entire application. With `-R`, Rails eager loads only when its `config.eager_load` is true. Otherwise, use the `-r` option to require your workers when you start Shoryuken.

Example:

```sh
bundle exec shoryuken \
        -R \
        -P /var/app/containerfiles/pids/shoryuken.pid \
        -C /var/app/current/config/shoryuken.yml \
        -L /var/app/containerfiles/logs/shoryuken.log \
        -r /var/app/current/app/jobs \
        -d
```

Also check that the queue name in `shoryuken.yml` matches the worker's `shoryuken_options queue:`. When ActiveJob is not loaded, Shoryuken logs `No worker supplied for <queue>` at start for each queue that has no worker.
