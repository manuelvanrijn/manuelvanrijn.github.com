---
layout: post
title: "Rake, pull me that DB!"
date: 2012-01-19 15:15
comments: true
categories:
published: false
---

After writing my last article how to [Convert PostgreSQL to SQLite](http://manuel.manuelles.nl/blog/2012/01/18/convert-postgresql-to-sqlite/ "How to: Convert a PostgreSQL database to a SQLite database"), I was asked why this couldn't be automated?

After some thoughts and scripting I've managed to create a [Rake](http://rake.rubyforge.org/ "Rake -- Ruby Make") task that will do all the tasks I described in just a few seconds!

## What will it do?

1. remove old dump and ssh the new dump to tmp/dump.sql
2. converts the file to readable stuff
3. remember the version number of the migration of the live db
4. backup and create development untill migration version live
5. import data
6. tell you all went well

## Configuration

To be able to use the Rake task, you have to add 2 additional fields (`ssh_user` and `ssh_host`) to your `database.yml` file. These fields are used to create a ssh connection for retrieving the database.

Here's example of a modified `config/database.yml`:

{% gist 1640429 database.yml %}

## Get the Rake file

Put the following code into `libs/tasks/database.rake`

{% gist 1640429 database.rake %}
