If you use standard Shoryuken workers by `include Shoryuken::Worker`, and do not use ActiveJob, [a commit in version 3.2.3](https://github.com/phstc/shoryuken/commit/bbfa813939f88e4e64beb178df85d9868bbff15f) stopped eager loading the entire application. In this case, you must use the `-r` option to require your jobs specifically while starting Shoryuken.

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

