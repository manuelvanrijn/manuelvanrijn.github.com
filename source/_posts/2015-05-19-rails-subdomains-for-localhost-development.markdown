---
layout: post
title: "Rails subdomains for localhost/development"
date: 2015-05-19 23:00
comments: true
categories: [rails, subdomains, development, ruby, domains, redirect, rack, handler, filters, localhost]
published: true
---

[{% img right /images/posts/local-subdomains.jpg 300 150 Rails subdomains for localhost development %}](/blog/2015/05/19/rails-subdomains-for-localhost-development/) Today I've just released version 1.0.0 of my latest gem [local-subdomain](https://github.com/manuelvanrijn/local-subdomain) offering subdomain support for your development environment, out of the box.

In this post I'd like to describe my findings and motivations.

<!-- more -->

## What I'd used to do

To support subdomains during development is frustrating. By default you can't have some subdomain.localhost to just bind to localhost.
So you have to edit your `/etc/hosts` file to add some fake subdomain and when you have to test another subdomain, you'd had to edit the file again and again and... well you get my point.

### But then!

I've stubbled on the magic domain [http://lvh.me](http://lvh.me) which redirects all request (including subdomains) to `127.0.0.1`. This means [http://some-subdomain.lvh.me:3000](http://some-subdomain.lvh.me:3000) redirects to `127.0.0.1` on port `3000` and let `request.subdomain` to return `some-subdomain` on my Rails controller.

```bash $ dig lvh.me
; <<>> DiG 9.8.3-P1 <<>> lvh.me
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 48494
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 0

;; QUESTION SECTION:
;lvh.me.        IN  A

;; ANSWER SECTION:
lvh.me.     3600  IN  A 127.0.0.1

;; AUTHORITY SECTION:
lvh.me.     3600  IN  NS  ns36.domaincontrol.com.
lvh.me.     3600  IN  NS  ns35.domaincontrol.com.

;; Query time: 167 msec
;; SERVER: 90.145.32.32#53(90.145.32.32)
;; WHEN: Tue May 19 23:05:01 2015
;; MSG SIZE  rcvd: 95
```

## Why this gem?

This week I've started on a side project that includes subdomain support. I remembered that I could use [http://lvh.me](http://lvh.me) but when I browsed to the url it just said **ERR_CONNECTION_REFUSED**

After some time I remembered Rails binds the `development` environment by default to `localhost` which is != `127.0.0.1` where [http://lvh.me](http://lvh.me) is redirecting to..

So after fixing this, I remembered to have some code, that checks if I was browsing to [http://localhost:3000](http://localhost:3000) and if so redirecting me to [http://lvh.me:3000](http://lvh.me:3000). Just to help me remember and enforce request for subdomains to go through [http://lvh.me](http://lvh.me) for subdomain support.

## So a gem it is!

Well because as many developers, I don't like to repeat myself, and decided to make the above to steps possible within a gem.

### 1. The before filter

The easy part was creating a module that adds a `before_filter` to the controller where the modules is included. The action would check if the request is from `lvh.me` and if not, redirect to it. Nothing special.

```ruby
module LocalSubdomain
  extend ActiveSupport::Concern

  included do
    before_filter :redirect_to_lvh_me
  end

  def redirect_to_lvh_me
    return unless Rails.env.development?
    served_by_lvh_me = !request.env['SERVER_NAME'][/lvh.me$/].nil?
    return if served_by_lvh_me

    http = request.env['rack.url_scheme']
    port = request.env['SERVER_PORT']
    path = request.env['ORIGINAL_FULLPATH']
    redirect_to "#{http}://lvh.me:#{port}#{path}"
  end
end
```

### 2. The Rack::Handler

This part was kind of tricky because we have to inject our code before the `boot` of rails is triggered, because we want to enforce the binding address `localhost` to be changed to `0.0.0.0`.

After some reading about the [rails initialization process](http://guides.rubyonrails.org/initialization.html) and experimenting I noticed that the gems are required (and if so) executed before the `Rack::Handler` is being called, which will trigger a Ruby server to be executed (WEBrick by default).

While looking at the [rack/handler.rb](https://github.com/rack/rack/blob/master/lib/rack/handler.rb) I noticed that the method `self.default` returns a Ruby server handler. Based on what Ruby server is being used, this will return a different handler.

After inspecting [WEBrick's](https://github.com/rack/rack/blob/master/lib/rack/handler/webrick.rb) and [Puma's](https://github.com/puma/puma/blob/master/lib/rack/handler/puma.rb) rack handlers, we'll see they both have the `self.run` method with the `options` argument, containing the `Host` which indicates the binding address.

Here's a code example, showing how to intercept the used Ruby server (Puma, WEBrick, ...) handler and extend that handler, with a custom `run` method to be able to modify the `options`

```ruby
module Rack
  module Handler
    # first override the self.default, to be able to intercept the handler
    # used by Puma, WEBrick, Thin, ... other?
    class << self
      alias_method :orig_default, :default
    end

    def self.default(options = {})
      # call the original method to return the handler and extend it's self.run method
      orig_default.instance_eval do
        class << self
          alias_method :orig_run, :run
        end

        def self.run(app, options = {})
          env = (options[:environment] || Rails.env)
          # only modify the options[:Host] if environment is development
          # and options[:Host] equals 'localhost'
          if options[:Host] == 'localhost' && env == 'development'
            options.merge!(Host: '0.0.0.0')
          end

          # after modifications, trigger original run method with the new options
          orig_run(app, options)
        end
      end

      # don't forget to return the extended handler
      orig_default
    end
  end
end
```

## Conclusion

After finishing these two pieces, I've bundled them into a Gem called [local-subdomain](https://github.com/manuelvanrijn/local-subdomain), so I don't have to add the `before_filter` and add `-b 0.0.0.0` to my `rails s` command.
