---
layout: post
title: "Mollie Bank"
date: 2012-02-29 21:29
comments: true
categories: 
published: true
---

The past week I've been working with the iDeal API from Mollie. [Mollie](http://www.mollie.nl/) is a company that give's us developers an easy way to make iDeal payments through there API, for a small fee.

After a long search I wasn't able to find a stable, well tested gem, I could use in our Rails project, so I developed the [ideal-mollie gem](https://rubygems.org/gems/ideal-mollie). After I finishing this gem, there was only one problem. 

When you want to test a iDeal transaction with the Mollie test bank, you **MUST** do some routing, so that Mollie can send you a request if the payment paid or not, to your local machine from the internet. So I made a alternative Mollie test bank you can use for what ever programming language you use for making Mollie iDeal payments without the routing problems.

<!-- more -->

## Mollie Bank

I've came op with a gem that runs a small [Sinatra](http://www.sinatrarb.com/) application that does exactly the same as the Mollie test bank, only it runs on your local machine, which means you can redirect to `localhost` and more important, it can perform the payment check.

Here you can find the [mollie-bank gem](https://rubygems.org/gems/mollie-bank), and the [source code](https://github.com/manuelvanrijn/mollie-bank)

{% img /images/posts/mollie-bank.png %}

## Install en run!

To use mollie-bank just install the gem by running

```
gem install mollie-bank
```

and then run it

```
mollie-bank
```

At this point, you can go to [http://localhost:4567/](http://localhost:4567/) and you'll see an intro page.

## Setup you application

Alright so now you only have to make sure that when you're developing on your local machine, you don't make the requests to Mollie but to your locally running Mollie Bank.

To do this you just have to change `https://secure.mollie.nl` into `http://localhost:4567` in you code.

## Todo

At this point I've only wrote a sample implementation for [Ruby on Rails](http://rubyonrails.org/ "Ruby on Rails") on the [Implement into existing modules Wiki](https://github.com/manuelvanrijn/mollie-bank/wiki/Implement-into-existing-modules) page, but soon there will be more examples.
