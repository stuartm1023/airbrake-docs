---
layout: classic-docs
title: Deploy tracking
categories: [features]
description: deploy tracking
---

*Having issues deploy tracking in your app? Check the [deploy troubleshooting
doc](/docs/airbrake-faq/deploy-tracking-troubleshooting).*

## Why track deploys?

Deploy Tracking with Airbrake gives insight into the relationship between your
deploys and your errors. Here are a few of the benefits deploy tracking gives
you:

- Resolve all errors on a deploy
- Project and error graphs show deploy markers
- Insight into which deploys introduced problems
- Insight into which deploys solved problems
- Generate backtrace links to click through to the offending lines of code
- Filter errors by deploy

### Resolve all errors on a deploy
Triggering a deploy in your Airbrake project resolves all errors in the
specified `environment` this gives you a fresh slate each deploy so you can
tell which errors you fixed and which errors are still an issue.

### Deploys show on your project's error graphs
Deploys are marked with blue circles on your project's graphs
![graph with deploys](/docs/assets/img/docs/airbrake/graph_with_deploys.png)

Hovering over them will activate a tooltip with environment and timestamp
information.

### Deploys create backtrace links
Using deploy tracking creates helpful links in your backtrace so you can
click through to the file/line/revision in GitHub/GitLab.

![backtrace link to github](/docs/assets/img/docs/airbrake/backtrace_link_to_github.png)

### Filter errors by deploy
You can see your deploy activity on your project overview page and you can
filter by a specific deploy in the search dropdown.

![filter by deploy](/docs/assets/img/docs/airbrake/deploy_filter.png)

## Deploy tracking with the API

You can track a deploy with the POST to the /v4/deploys api
endpoint [docs](https://airbrake.io/docs/api/#create-deploy-v4).

You want to trigger a POST each time you deploy, specifying at least your
`PROJECT_ID`, `PROJECT_KEY` and `environment` in the JSON data. If you provide the
other keys this will enable us to build links in backtraces that link to your
repository.

### Example curl command
{% highlight bash %}
curl -X POST -H "Content-Type: application/json" -d '{"environment":"production","username":"john","repository":"https://github.com/USERNAME/REPO","revision":"38748467ea579e7ae64f7815452307c9d05e05c5","version":"v2.0"}' "https://airbrake.io/api/v4/projects/PROJECT_ID/deploys?key=PROJECT_KEY"
{% endhighlight %}

### Example bash script
This is a simple example that registers a deploy with Airbrake. It assumes it is
run from a Git repository of your deployed project code. You can tweak how the
deployed `REVISION` is determined to suite your deploy process. Make sure you
check and modify the variables as appropriate for your project and requirements.

{% highlight bash %}
#!/bin/bash

PROJECT_ID=12345
PROJECT_KEY=ABCDEF123456
ENVIRONMENT=production
REPOSITORY=https://github.com/USERNAME/REPO
REVISION="$(git rev-parse HEAD)"
USERNAME=$(whoami)

curl -X POST \
  -H "Content-Type: application/json" \
  -d '{"environment":"'${ENVIRONMENT}'","username":"'${USERNAME}'","repository":"'${REPOSITORY}'","revision":"'${REVISION}'"}' \
  "https://airbrake.io/api/v4/projects/${PROJECT_ID}/deploys?key=${PROJECT_KEY}"
{% endhighlight %}

### JSON post data description
You can post JSON data with the following keys. there are example

Key | Example
--- | -------
environment | production
username | john
repository | https://github.com/USERNAME/REPO
revision | 38748467ea579e7ae64f7815452307c9d05e05c5
version | v2.0

### Response

The API returns `201 Created` status code on success.

## Deploy tracking with `rake airbrake:deploy`

Deploy tracking requires the latest airbrake notifier. Be sure you have the
latest version installed from [github](https://github.com/airbrake/airbrake).

Once installed, your application will have a new rake task, here are some
examples.

To Mark all Errors as Resolved for the `production` environment you can use the
following command:

{% highlight bash %}
rake airbrake:deploy TO=production
{% endhighlight %}

To Mark all Errors as Resolved for a **specific environment**, use the
following format:

{% highlight bash %}
rake airbrake:deploy RAILS_ENV=<rails environment> TO=rails environment
{% endhighlight %}

For example the following rake command will track a deploy to the `development`
environment.

{% highlight bash %}
rake airbrake:deploy RAILS_ENV=development TO=development
{% endhighlight %}

### Capistrano

If you use capistrano, you shouldn't have to run the rake command by hand.  The
Airbrake notifier also includes a capistrano recipe that runs after
`deploy:cleanup` which automatically triggers that rake task.

## Deploy tracking on Heroku

To implement Deploy Hooks on Heroku, please use one of the following options.

You can manually create the Deploy hook:
{% highlight bash %}
heroku addons:add deployhooks:http \
--url="https://airbrake.io/api/v3/projects/PROJECT_ID/heroku-deploys\
?key=PROJECT_KEY\
&environment=ENVIRONMENT\
&repository=REPOSITORY_URL"
{% endhighlight %}

Or create it using rake:
{% highlight bash %}
rake airbrake:install_heroku_deploy_hook
{% endhighlight %}

Be sure to provide the app name for the rake task if you have multiple Heroku
apps configured. You can send it by setting the `HEROKU_APP` environment
variable.

The repository URL will default to the URL of the `origin` remote. The
`REPOSITORY_URL` environment variable can be used to override this value.

{% highlight bash %}
export HEROKU_APP=your-heroku-app
export REPOSITORY_URL=git@github.com:username/repo.git
rake airbrake:install_heroku_deploy_hook
{% endhighlight %}

## Deploy tracking on Engine Yard Cloud

To notify Airbrake of your deploys to EngineYard cloud, you can use the
following `deploy/after_restart.rb` script:

{% highlight ruby %}
def notify_airbrake?(config)
  %w(staging production integration).include?(config.environment_name) &&
  %w(solo app_master).include?(config.current_role)
end

def track_deploy(config)
  run(
    "cd #{config.current_path} && bundle exec rake airbrake:deploy " \
    "ENVIRONMENT=#{config.environment_name} REVISION=#{config.active_revision} " \
    "USERNAME=#{config.deployed_by} REPOSITORY=#{config.repo}"
  )
end

track_deploy(config) if notify_airbrake?(config)
{% endhighlight %}

You can find more info on `config` and its variables in the <a
href="https://support.cloud.engineyard.com/hc/en-us/articles/205407008-Use-Ruby-Deploy-Hooks"
target="_blank">EngineYard doc on ruby deploy hooks</a>.
