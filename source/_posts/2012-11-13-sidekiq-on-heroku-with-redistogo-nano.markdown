---
layout: post
title: "Sidekiq on Heroku with Redis To Go Nano"
date: 2012-11-13 15:42
comments: true
categories: [sidekiq, heroku, redis, redis to go, connections, rails, ruby, threading]
published: true
---

[{% img right /images/posts/redis.png 200 200 Yeoman %}](/blog/2012/06/22/integrate-travis-ci-into-grunt/) As a follow-up of [my previous post](/blog/2012/11/13/scalable-heroku-worker-for-sidekiq/) I want to explain how to get [Sidekiq](http://sidekiq.org/) to work on [Heroku](http://www.heroku.com) with a [Redis To Go](https://addons.heroku.com/redistogo) Nano instance.

Because the Nano instance has some connection limitation you have to make some config changes so you won't get `ERR max number of clients reached` error messages.

<!-- more -->

## Why this post

Today I've been struggling a lot with getting the Sidekiq to work probably with Redis to Go Nano on Heroku. The main problem was I was having difficulties with the amount of connection's being created to the Redis server. Because the Nano variant is free but large enough for handling normal sized queues of work, we have to face the limitation of 10 connections.

## Why ERR max number of clients reached?

The error `Error fetching message: ERR max number of clients reached`, is quite clear isn't it? The actual question is how we can reduce the connections being opened to match the 10 connection limit given by the Nano instance.

After some research I found the factors that you have to tweak in order to reach the magic number of 10.

1. Sidekiq Client connection size
2. Sidekiq Server connection size
3. Sidekiq Server reserved connections (reserved for the Fetcher and Retrier)
4. Sidekiq concurrency size
5. Unicorn worker_process size
6. Heroku Web Dyno count
7. Heroku Worker count

The answer for all our problems could be described in the following sum:

max connections = (Heroku worker count * (concurrency + 2 reserved connections)) + (web dyno count * (client connection size * unicorn worker_process size))

## Don't care why tell me how!

If you just want to know what you need to change/setup you can go directly to a small tool I've built to calculate the number of connections/concurrencies need for a number of workers/web workers etc.

[SidekiqHerokuRedis calculator](http://manuel.manuelles.nl/sidekiq-heroku-redis-calc)

## Connections and concurrencies

If you have a small amount of Redis connections we need to make some modifications in order to get all our processes happy and not throwing connection errors around.

At the bottom of this post I'll put my final configuration files.

### Dynos * (unicorns * client size)

The first thing I changed was the Redis connection size for the client. By default Sidekiq takes **5** connections per client. Because my applications only queries Redis for adding tasks to the Sidekiq queue, one connection should be more than enough.

{% pullquote %}
One mistake I made was forgetting about the number of dynos and Unicorn worker_process size. In my `app/config/unicorn.rb` I had `worker_processes 3` defined which actually means {" 3 workers * dyno count = redis_connections "} taken by the Client.
{% endpullquote %}

#### Examples:

In my case where I've set the connection size to one we could have the following examples

1. **1** web dyno with **2** unicorn worker_processes takes **2 Redis connections**
2. **2** web dyno with **2** unicorn worker_processes takes **4 Redis connections**
3. **5** web dyno with **4** unicorn worker_processes takes **20 Redis connections**

If you up the client size to, for example 3 you should also multiply the Redis connections by this number (so for example 3 you'll have 20 * 3 = 60 connections).

### Worker * (concurrency + 2)

When calculating the number of connections the Sidekiq server needs, we need to modify the number of currencies it initializes on launch. This number represents the number of threads created by the Sidekiq server which will perform queued tasks. Each concurrency/thread takes up 1 Redis connection.

When tweaking this number I found out the Sidekiq server took 2 additional connections upon the concurrency number. This seemed to be default behavior becase Sidekiq server uses these two for the Fetcher and Retrier commands.

#### Examples:

1. **1** worker dyno running Sidekiq, with **1** concurrency takes **3 Redis connections**
2. **1** worker dyno running Sidekiq, with **2** concurrency takes **4 Redis connections**
3. **2** worker dyno running Sidekiq, with **2** concurrency takes **8 Redis connections**

The default concurrency size of Sidekiq is set to 25 which mean that without modifications you need at least 27 Redis connections for the Sidekiq server.

## My final code

``` bash
connection limit:           10
unicorn processes:           3
web dynos:                   2
worker dynos (sidekiq):      1
client pool size:            1
server pool size:            2
concurrency:                 2
```

``` ruby app/config/initializers/sidekiq.rb
require 'sidekiq'

Sidekiq.configure_client do |config|
  config.redis = { :size => 1 }
end

Sidekiq.configure_server do |config|
  config.redis = { :size => 2 }
end
```

``` yaml app/config/sidekiq.yml
:concurrency: 2
```

``` bash Procfile
web: bundle exec unicorn -p $PORT -c ./config/unicorn.rb
worker: bundle exec sidekiq -e production -C config/sidekiq.yml
```

``` ruby app/config/unicorn.rb
worker_processes 3
```
