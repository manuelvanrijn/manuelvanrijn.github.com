---
layout: post
title: "Scalable Heroku worker for Sidekiq with RedisToGo Nano"
date: 2012-11-09 16:34
comments: true
categories: []
published: true
---

[{% img right /images/posts/yeoman-logo.png 200 200 Yeoman %}](/blog/2012/06/22/integrate-travis-ci-into-grunt/) In this post I'd like to show you guys how to deploy your Rails application to Heroku with a Sidekiq worker process.

Our goal is to create a worker on Heroku only when there's work to do and to destroy this worker after work is done because we don't like to pay for a lazy worker of corse.

<!-- more -->

## Why this post

Today I've been struggeling a lot with getting the Sidekiq to work probably with Redis to Go Nano on Heroku. The main problem was I was having difficulties with the amount of connection's being created to the Redis server. Because the Nano variant is free but large enough for handling small queue's of work, we have to face the limitation of 10 connection's.

## The final code

### Sidekiq worker

The worker is very basic. It just performs some heavy task and sends a email after it finished.

``` ruby app/workers/my_worker.rb
class MyWorker
  include Sidekiq::Worker

  def perform(id)
    object = Model.find(id)
    # perform some heavy task
    UserMailer.download_is_ready(object).deliver
  end
end
```

### Sidekiq initializer

The initializer is a bit different than the samples on the Sidekiq wiki. This is mainly because I'm using the autoscaler gem which is a Sidekiq middleware that will create and destory a Heroku worker based on the amount of tasks in a Sidekiq queue.

The only thing this gem require's is having these two enviroment variables defined on your heroku instance to create and destory a worker:

``` bash
# API KEY can be found on https://dashboard.heroku.com/account
heroku config:add HEROKU_API_KEY=123-your-key-456
heroku config:add HEROKU_APP=your_heroku_app_name
```

Here's my final code for the Sidekiq initializer:

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
    config.redis = { :size => 1 }
    config.client_middleware do |chain|
      chain.add Autoscaler::Sidekiq::Client, 'default' => heroku
    end
  end
end

Sidekiq.configure_server do |config|
  config.redis = { :size => 2 } if heroku
  config.server_middleware do |chain|
    if heroku
      p "[Sidekiq] Running on Heroku, autoscaler is used"
      chain.add(Autoscaler::Sidekiq::Server, heroku, 60)
    else
      p "[Sidekiq] Running locally, so autoscaler isn't used"
    end
  end
end
```

``` yaml app/config/sidekiq.yml
:concurrency: 2
```

``` ruby Gemfile
# genereren van permit request middels sidekiq
gem 'sidekiq'
gem 'sinatra', require: false
gem 'slim'

# sidekiq middleware for auto-scaling workers
gem 'autoscaler'
```

``` bash Procfile
web: bundle exec unicorn -p $PORT -c ./config/unicorn.rb
worker: bundle exec sidekiq -e production -C config/sidekiq.yml
```

``` ruby app/config/unicorn.rb
worker_processes 3 # amaount of unicorn workers to spin up
```

## The Sum!

``` bash
connection limit: 10
unicorn processes: 3
web dynos: 2
worker dynos (sidekiq server processes): 1
client pool size: 1
server pool size: 2
concurrency: 2
```

concurrency 2 == 2 connections
2 reserved connections == 2 connections
1 * 6 unicorn processes == 6 connections
total = 10
