## Heroku

Add `worker` in your [Procfile](https://devcenter.heroku.com/articles/procfile):

```shell
worker: bundle exec shoryuken ...
```

Then:

```shell
heroku ps:scale worker=1
```

Shoryuken does not require a `web` process, so you can disable `web`, in case you don't need it.

```shell
heroku ps:scale web=0
```

## Capistrano

Use [capistrano-shoryuken](https://github.com/joekhoobyar/capistrano-shoryuken).

## OpsWorks

See [OpsWorks Recipe](https://github.com/carpet/opsworks-shoryuken).

## AWS Elastic Beanstalk

See [.ebextension file here](https://github.com/ruby-shoryuken/shoryuken/wiki/AWS-Beanstalk-Config).

## systemd integration

See [systemd integration](https://github.com/ruby-shoryuken/shoryuken/issues/320#issuecomment-282290069).

A minimal unit file:

```ini
# /etc/systemd/system/shoryuken.service
[Unit]
Description=Shoryuken
After=network.target

[Service]
Type=simple
WorkingDirectory=/var/www/myapp/current
Environment=RAILS_ENV=production
ExecStart=/usr/local/bin/bundle exec shoryuken -R -C config/shoryuken.yml
# systemd stops with SIGTERM: Shoryuken waits up to `timeout` seconds for busy workers
TimeoutStopSec=30
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Keep `TimeoutStopSec` above Shoryuken's `timeout`. Do not map `ExecReload` to USR1: USR1 makes Shoryuken exit (see [[Signals]]).

## Shutdown

Set `timeout` below the time your platform waits before it kills the process. See [[Signals]].
