## Heroku

Add `worker` in your [Procfile](https://devcenter.heroku.com/articles/procfile):

```shell
worker: bundle exec shoryuken ...
```

Then:

```shel
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

See [.ebextension file here](https://github.com/phstc/shoryuken/wiki/AWS-Beanstalk-Config).

## systemd integration

See [systemd integration](https://github.com/phstc/shoryuken/issues/320#issuecomment-282290069).