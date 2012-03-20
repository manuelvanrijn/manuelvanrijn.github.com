---
layout: post
title: Failed to build gem native extension
date: 2012-03-20 13:14
comments: true
categories: [code, gem, ruby, extension, capybara-webkit]
published: true
---

{% img left /images/posts/failed-gem.jpg 140 140 Go fix! %}
Today I had to reinstall my Linux installation because my harddisk died a quick dead. While configuring all the software I use to develop I got some weird error after I'd installed [rbenv](https://github.com/sstephenson/rbenv) and tried to execute `bundle install` on one of our projects.<br /><br />

<!-- more -->

## The error

The following error was raised when I tried to install capybara-webkit

```
Installing capybara-webkit (0.10.1) with native extensions
Gem::Installer::ExtensionBuildError: ERROR: Failed to build gem native extension.

/home/manuel/.rbenv/versions/1.9.3-p125/bin/ruby extconf.rb
sh: qmake: not found

Gem files will remain installed in /home/manuel/.rbenv/versions/1.9.3-p125/lib/ruby/gems/1.9.1/gems/capybara-webkit-0.10.1 for inspection.
Results logged to /home/manuel/.rbenv/versions/1.9.3-p125/lib/ruby/gems/1.9.1/gems/capybara-webkit-0.10.1/./gem_make.out
An error occured while installing capybara-webkit (0.10.1), and Bundler cannot continue.
Make sure that `gem install capybara-webkit -v '0.10.1'` succeeds before bundling.
```

## The solution

I first thought there were some problems with `make` or `cmake` because that's what the error says. `qmake: not found`, but that didn't seem to be te real problem.

After some googlefoo I found out that I was missing the `libqtwebkit-dev` package. So the problem in this case was easily resolved by running:

```
sudo apt-get install libqtwebkit-dev
```
