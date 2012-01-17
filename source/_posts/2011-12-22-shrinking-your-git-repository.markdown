---
layout: post
title: "Shrinking your git repository"
date: 2011-12-22 15:01
comments: true
categories: [git, code]
published: true
---

In the past weeks I've looked into a problem that kept us from releasing to [Heroku](http://www.heroku.com/ "Heroku"), because the git repository was to large. In time the repository has grown to approximate __180 Mb__.

{% img right /images/posts/shrink-mario.jpg 140 140 Shrink! %}

In the past we've made some decisions that we probably wouldn't take again. We for example, we decided to store the gem files in the vendor folder and having the rails code included in our repository. Our goal then was, that releasing the project with [Capistrano](http://www.capify.org "Capistrano") would be easier and faster, because it didn't have to download all the gem's.

Anyway at this point we don't have the gem's stored anymore in the vendor folder. Since tools like [RVM](http://beginrescueend.com/ "Ruby version Manager") and [rbenv](https://github.com/sstephenson/rbenv/ "rbenv") we store these within the ruby/rails version of that specific project.

Still we have the problem that the repository contains files etc. we want to remove to shrink the size of it.

## Find large files from your git history

First we have to track down what files are making our repository so large. I've found a perl script that scans your repository for files of a specific size. Here's the code:

{% gist 1354502 %}

So I've placed this file within my repository and executed the below command to see all files, from all commits, that are bigger then __500 Kb__

```
perl git-large-files.pl 500k
```

This give's you an output like:

```
9794784 6 weeks ago
  609280  vendor/cache/builder-3.0.0.gem
  823808  vendor/cache/ffi-1.0.9.gem
  801792  vendor/cache/gherkin-2.4.21.gem
.... more ....
874451c 3 months ago
  609280  vendor/bundle/ruby/1.9.1/cache/builder-3.0.0.gem
  823808  vendor/bundle/ruby/1.9.1/cache/ffi-1.0.9.gem
  801792  vendor/bundle/ruby/1.9.1/cache/gherkin-2.4.21.gem
.... more ....
47ebdca 1 year, 1 month ago
  15467626  vendor/gems/webrat-0.7.1/vendor/selenium-server.jar
  6185324 vendor/rails/activerecord/examples/performance.sql
  14465482  log/development.log
```

As you can see the following folders contain data that we don't need anymore and fills our repository with unused data:

  1. vendor/cache/*
  2. vendor/bundle/*
  3. vendor/rails/*
  4. log/development.log

## Removing the files from git

To remove the folders specified above I used the commands specified by the [Remove sensitive data](http://help.github.com/remove-sensitive-data/ "GitHub Help: Remove sensitive data") post from GitHub Help

The only thing I changed was adding `-rf` to the `git rm` command to recursively force remove the files because I am dealing with multiple files/folders within the target folder.

### Rewrite history

The final command I used was:

```
git filter-branch -f --index-filter 'git rm -rf --cached --ignore-unmatch vendor/cache/* vendor/bundle/* vendor/rails/* log/development.log' --prune-empty -- --all
```

Mind the `vendor/cache/* vendor/bundle/* vendor/rails/* log/development.log` part. You can provide multiple path's

When the command is finished, the history has been rewritten, but still the size of the repository hasn't changed at this point.

### Cleanup and reclaim space

You have to execute the following commands to also remove the files from you local repository.

```
rm -rf .git/refs/original/
git reflog expire --expire=now --all
git gc --aggressive --prune=now
```

Now we can force push our repository so that others can enjoy our effort.

```
git push origin master --force
```

## The result (98,55% smaller)

After following the above steps our repository was shrinked by __98,55 %!__. First the repository was 180 Mb and now it is __2.6 Mb__

Ofcourse we had this numbers because there were alot of gem files within our repository that were updated frequently and pushed over and over again to the master branch.

I hope this post will help other's to track large files within there git repository and how they can remove them to shrink the size of there repostiories.
