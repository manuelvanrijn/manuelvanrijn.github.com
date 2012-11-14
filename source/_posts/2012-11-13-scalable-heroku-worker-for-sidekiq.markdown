---
layout: post
title: "Scalable Heroku worker for Sidekiq"
date: 2012-11-13 09:02
comments: true
categories: [scale, heroku, redis, sidekiq, automating, threading, rails, ruby]
published: true
---

[{% img right /images/posts/more-workers.jpg 200 200 Yeoman %}](/blog/2012/06/22/integrate-travis-ci-into-grunt/) In this post I'd like to show you guys how to deploy your Rails application to Heroku with a Sidekiq worker that only got's initiated when there are tasks in the queue to process.

Our goal is to start a worker when a task is added to the queue and to destroy the worker after it's done processing the queue tasks. This will result in a much lower bill at the end of the month because the worker doesn't have to be up the whole month.

<!-- more -->

## Setting up Sidekiq

First let's take a look at a default configuration of Sidekiq within a Rails project.

### Add the gem

First we need to add Sidekiq to our Gemfile and run `bundle install`

``` ruby Gemfile
gem 'sidekiq'
```

### Create a Sidekiq worker

The worker is very basic. It just performs some heavy task and sends an email after it finished.

``` ruby app/workers/my_worker.rb
class MyWorker
  include Sidekiq::Worker

  def perform(id)
    object = Model.find(id)
    object.generate_download
    UserMailer.download_is_ready(object).deliver
  end
end
```

### Procfile

In our Procfile we have to define the `worker` line to start the Sidekiq server on a Heroku worker dyno.

``` bash Procfile
web: bundle exec unicorn -p $PORT -c ./config/unicorn.rb
worker: bundle exec sidekiq -e production
```

### Move heavy task to the queue

On the controller we probably have a line calling `object.generate_download` which takes to much time. We'll change this line to execute the `MyWorker` instead:

``` ruby app/controllers/download_controller.rb
class DownloadController < ApplicationController
  def generate
    object = Model.find(param[:id])
    # instead of calling object.generate_download we'll do:
    MyWorker.perform_async(object.id)
  end
end
```

**NOTE:** It's recommended not to add the whole object into the worker because this is stored in the Redis database. Also the object might be changed before it's being processed by Sidekiq. Adding the `id` and fetching the object in the worker is a better approuch.

## So far...

At this point we have configured our Rails project to add tasks to Redis and have Sidekiq perform executing the worker when a task is added. If we deploy our Rails app to Heroku we need to add the Redis To Go addon and add a Heroku worker dyno to our application.

### Running locally

To test this locally we need:

- Redis installad locally and running
- Run the Rails app (`bundle exec rails server`)
- Run the Sidekiq server (`bundle exec sidekiq`)

## Adding the autoscaler

At this point the Heroku worker dyno has to be started, to pickup tasks from the queue. It's likely you do not need this worker to run the whole day because the queue might be empty most of the time. Here's where the [autoscaler gem](http://github.com/JustinLove/autoscaler/) come's in handy.

This gem acts as Middleware for Sidekiq and performs the following tasks:

1. When a task is added it checks if a Worker is present.
2. When a worker is already running it does nothing and the worker will pickup the task.
3. If there isn't a worker running it will create a Heroku worker dyno.
4. If the worker finished processing the queue it will keep the worker alive for 60 seconds just in case we add a task to the queue within this time. If no tasks are added it will destroy the worker.

### Add the gem

First we need to add the autoscaler gem to our Gemfile and run `bundle install`

``` ruby Gemfile
gem 'autoscaler'
```

### Adding required ENV variables

The gem requires two enviroment variables to be set on your Heroku application. The `HEROKU_API_KEY` is required to perform creating and removing a Heroku worker dyno and the `HEROKU_APP` to know on which application it has to create/destroy the worker dyno on.

``` bash
# API KEY can be found on https://dashboard.heroku.com/account
heroku config:add HEROKU_API_KEY=123-your-key-456
heroku config:add HEROKU_APP=your_heroku_app_name
```

### Tweaking the Sidekiq initializer

Because this gem acts as Middleware we need to create a `sidekiq.rb` in our initializers folder. This file will checks if we are running on Heroku and if so, activates the autoscaler as Middleware.

``` ruby app/config/initializers/sidekiq.rb
require 'sidekiq'
require 'autoscaler/sidekiq'
require 'autoscaler/heroku_scaler'

heroku = nil
if ENV['HEROKU_APP']
  heroku = Autoscaler::HerokuScaler.new
end

Sidekiq.configure_client do |config|
  if heroku
    config.client_middleware do |chain|
      chain.add Autoscaler::Sidekiq::Client, 'default' => heroku
    end
  end
end

Sidekiq.configure_server do |config|
  config.server_middleware do |chain|
    if heroku
      p "[Sidekiq] Running on Heroku, autoscaler is used"
      chain.add(Autoscaler::Sidekiq::Server, heroku, 60) # 60 seconds timeout
    else
      p "[Sidekiq] Running locally, so autoscaler isn't used"
    end
  end
end
```

## Ready to go!

At this point we're ready to deploy our application to Heroku and let the autscaler automaticly create and destroy a worker dyno whenever it needs to process tasks from the Sidekiq queue.
