---
layout: post
title: "“Rake, pull me that DB!”"
date: 2012-01-19 15:15
comments: true
categories: [rails, code, sqlite, postgresql, rake, ruby]
published: true
---

After writing my last article how to [Convert PostgreSQL to SQLite](http://manuel.manuelles.nl/blog/2012/01/18/convert-postgresql-to-sqlite/ "How to: Convert a PostgreSQL database to a SQLite database"), I was asked why this couldn't be automated?

So I started coding and managed to create a [Rake](http://rake.rubyforge.org/ "Rake -- Ruby Make") task that will do all of the steps I described in the article in just a few seconds!

<!-- more -->

## Update

Today a found a project that does exactly the samething as the below rake task, but with support for more db providers. [Heroku](http://www.herko.com) uses this project for retrieving and pushing you database to the heroku database server.

The project is called Taps and can be found here: [http://adam.heroku.com/past/2009/2/11/taps_for_easy_database_transfers](http://adam.heroku.com/past/2009/2/11/taps_for_easy_database_transfers/)

## What will it do?

1. remove old dump and ssh the new dump to `tmp/dump.sql`
2. converts the dumped SQL in to a valid format for SQLite
3. remembers the version number of the migration on the production db
4. backup, create and migrates the development SQLite to the version remembered
5. import the SQL
6. tell you all went well

## Configuration

To use the Rake task, you have to add 2 additional fields (`ssh_user` and `ssh_host`) to your `database.yml` file. These fields are used to create an ssh connection for retrieving the PostgreSQL dump.

Here's example of a modified `config/database.yml`:

{% gist 1640429 database.yml %}

## Get the Rake file

Put the following code into `libs/tasks/database.rake`

{% gist 1640429 database.rake %}

## Usage

```
bundle exec rake db:pull
```

### No SSH?

If you want to skip the step where the rake tasks get's the dump using ssh, you have to copy the dump.sql into the tmp folder by yourself (note that the name __must__ be `dump.sql`)

After you copied it, you should execute the following commands, to generate the development database with the dumped sql.

```
bundle exec rake db:optimze_pg_dump_for_sqlite
bundle exec rake db:recreate_with_dump
```

### Want to skip entering the ssh password every time?

You could generate a public/private key pair with RSA and append that key to the production server so you don't have to enter the password over and over again to connect with ssh.

#### 1. Generate a public/private key pair with RSA

First check if you haven't already generated a `id_rsa` file in your `$HOME/.shh` folder. If you already have a `id_rsa` file continue with step 2.

Run the following command on you local machine and accept the defaults.

```
ssh-keygen -t rsa
```

This command creates an RSA public/private key pair in your `$HOME/.ssh` directory. The private key is `~/.ssh/id_rsa` and the public key is `~/.ssh/id_rsa.pub`

#### 2. Install public key on remote machine

Now you can copy the public key to the remote machine by executing the following command:

```
ssh-copy-id -i root@productionserver.com
```

This command will ask you to enter the ssh password for the ssh user "root" for the hostname "productionserver.com".

After you've enter the password (for the last time) you can create a ssh connection without entering the password by executing:

```
ssh root@productionserver.com
```
