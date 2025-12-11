### Web server environment 
To restart (or just start) Shoryuken on each deployment to Beanstalk, prepare the configuration files as described below. 
Make sure the selected EC2 instance type has enough memory to ensure the functioning of both the web server and Shoryuken.

#### Amazon Linux 2 and 2023 with Procfile

Generally, it is recommended to run more complex projects on AWS EB with your own Procfile, it gives greater control over the version of the web server and other parameters.
The easiest way to add a shoryuken worker in Elastic Beanstalk is to simply add the command to run it to the Procfile as `worker:`

```
web: bundle exec puma -C /opt/elasticbeanstalk/config/private/pumaconf.rb --pidfile /var/app/current/tmp/pids/server.pid
worker: bundle exec shoryuken -R -P /var/app/current/tmp/pids/shoryuken.pid -L /var/app/current/log/shoryuken.log
```

#### Amazon Linux 2 with hooks (deprecated)

`.platform/hooks/postdeploy/restart_shoryuken.sh`

```sh
#!/bin/bash

# NOTE: get-config platformconfig returns paths WITH the trailing slash .../
APP_DEPLOY_DIR=$(/opt/elasticbeanstalk/bin/get-config platformconfig -k AppDeployDir)
LOG_DIR="${APP_DEPLOY_DIR}log"
PID_DIR="${APP_DEPLOY_DIR}tmp/pids"
EB_APP_USER=$(/opt/elasticbeanstalk/bin/get-config platformconfig -k AppUser)

mkdir -m 777 -p $PID_DIR

if [ -f $PID_DIR/shoryuken.pid ]
then
  kill -TERM `cat $PID_DIR/shoryuken.pid` || echo "The shoryuken process was not running."
  rm -rf $PID_DIR/shoryuken.pid
fi

sleep 10

cd $APP_DEPLOY_DIR

# NOTE: in local development environment, run `bundle exec dotenv shoryuken \`
su -s /bin/bash -c "bundle exec shoryuken \
  -R \
  -P $PID_DIR/shoryuken.pid \
  -C ${APP_DEPLOY_DIR}config/shoryuken.yml \
  -r ${APP_DEPLOY_DIR}app/jobs \
  -L $LOG_DIR/shoryuken.log \
  -d" $EB_APP_USER

exit 0
```

----

`.platform/hooks/predeploy/mute_shoryuken.sh`

```sh
#!/bin/bash

APP_DEPLOY_DIR=$(/opt/elasticbeanstalk/bin/get-config platformconfig -k AppDeployDir)
PID_DIR="${APP_DEPLOY_DIR}tmp/pids"
EB_APP_USER=$(/opt/elasticbeanstalk/bin/get-config platformconfig -k AppUser)

if [ -f $PID_DIR/shoryuken.pid ]
then
  su -s /bin/bash -c "kill -USR1 `cat $PID_DIR/shoryuken.pid`" $EB_APP_USER || echo "The shoryuken process was not running."
fi

exit 0
```

#### Amazon Linux 1 (deprecated platform)
`.ebextensions/shoryuken.config` inside your repo.

``` yaml

# .ebextensions/shoryuken.config
# Based on the conversation in https://github.com/phstc/shoryuken/issues/48

files:
  "/opt/elasticbeanstalk/hooks/appdeploy/post/50_restart_shoryuken":
    mode: "000777"
    content: |
      APP_DEPLOY_DIR=$(/opt/elasticbeanstalk/bin/get-config container -k app_deploy_dir)
      LOG_DIR=$(/opt/elasticbeanstalk/bin/get-config container -k app_log_dir)
      PID_DIR=$(/opt/elasticbeanstalk/bin/get-config container -k app_pid_dir)

      EB_SCRIPT_DIR=$(/opt/elasticbeanstalk/bin/get-config container -k script_dir)
      EB_SUPPORT_DIR=$(/opt/elasticbeanstalk/bin/get-config container -k support_dir) 
      . $EB_SUPPORT_DIR/envvars
      . $EB_SCRIPT_DIR/use-app-ruby.sh

      if [ -f $PID_DIR/shoryuken.pid ]
      then
        kill -TERM `cat $PID_DIR/shoryuken.pid` || echo "The shoryuken process was not running."
        rm -rf $PID_DIR/shoryuken.pid
      fi

      sleep 10

      cd $APP_DEPLOY_DIR

      bundle exec shoryuken \
        -R \
        -P $PID_DIR/shoryuken.pid \
        -C $APP_DEPLOY_DIR/config/shoryuken.yml \
        -L $LOG_DIR/shoryuken.log \
        -d

  "/opt/elasticbeanstalk/hooks/appdeploy/pre/03_mute_shoryuken":
    mode: "000777"
    content: |
      PID_DIR=$(/opt/elasticbeanstalk/bin/get-config container -k app_pid_dir)
      if [ -f $PID_DIR/shoryuken.pid ]
      then
        kill -USR1 `cat $PID_DIR/shoryuken.pid` || echo "The shoryuken process was not running."
      fi
```

### Worker environment
For information on deploying to a worker environment see the thread in https://github.com/phstc/shoryuken/issues/48