---
layout: post
title: "Sidekiq on Heroku with RedisToGo Nano"
date: 2012-11-13 15:42
comments: true
categories: []
published: true
---

[{% img right /images/posts/yeoman-logo.png 200 200 Yeoman %}](/blog/2012/06/22/integrate-travis-ci-into-grunt/) As a followup of my previous post I want to explain how to get Sidekiq to work on Heroku with a RedisToGo Nano instance.

Because the Nano instance has some connection limitation you have to make some config changes so you won't get `Error fetching message: ERR max number of clients reached` error messages.

<!-- more -->

## Why this post

Today I've been struggeling a lot with getting the Sidekiq to work probably with Redis to Go Nano on Heroku. The main problem was I was having difficulties with the amount of connection's being created to the Redis server. Because the Nano variant is free but large enough for handling normal sized queue's of work, we have to face the limitation of 10 connection's.

## Why ERR max number of clients reached?

The error is quite clear isn't it? The actual question is how can we reduce the connections created to match the 10 connection limit given by the Nano instance.

After some research I found a few factors that you have to tweak in order to reach the magic number of 10.

1. Sidekiq Client connection size
2. Sidekiq Server connection size
3. Sidekiq Server reserved connections (reserved for the Fetcher and Retrier)
4. Sidekiq concurrency size
5. Unicorn worker_process size
6. Heroku Web Dyno count
7. Heroku Worker count

The answer for all our problems could be described in the following sum:

max connections = (Heroku worker count * (concurrency + 2 reserved connections)) + (web dyno count * (client connection size * unicorn worker_process size))

### Dynos * (unicorns * client size)

The client size should be calculated with the number

{% pullquote %}
One mistake I made was forgetting about the number of dynos and Unicorn worker_process size. In my `app/config/unicorn.rb` I had `worker_processes 3` defined which actually means {" 3 workers * dyno count = redis_connections "} taken by the Client.
{% endpullquote %}

### Worker * (concurrency + 2)

When calculating the server we need to modify the number of currencies a Sidekiq Worker initiates on launch. This number represents the number of threads created by the Sidekiq worker which will perform queued tasks. Each concurrency/thread takes up 1 redis connection.

When tweaking this number I found out the Sidekiq server took 2 additional connections upon the concurrency number. This seemed to be default behaviour becase Sidekiq server uses these two for the Fetcher and Retrier commands.

### Sidekiq concurrency

The second part we need to tweak are the number of connections and concurrencies created by the Sidekiq server. By default Sidekiq creates 25 concurrencies and each concurrency takes 5 redis connections from the pool.

default 25 concurrency


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
