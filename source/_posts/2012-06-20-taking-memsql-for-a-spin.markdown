---
layout: post
title: "Taking MemSQL for a spin"
date: 2012-06-20 10:18
comments: true
categories: [code, memsql, rails, ruby, mysql, postgresql]
published: true
---

[{% img left /images/posts/memsql.png 200 200 MemSQL %}](/blog/2012/06/20/taking-memsql-for-a-spin/) This week I read an article on [HackerNews](http://news.ycombinator.com/item?id=4126007) about Ex-facebook employs releasing a new database server called MemSQL.

After viewing there [product overview video](http://developers.memsql.com/) I got excited and wanted to take this product for a spin.

<!-- more -->

## What to test?

First I needed a project for testing the performance on. At [my work](http://www.auxilium.nl) we developed a Rails application that needs to validate a permit request with rules, questions, filters, answers depending on a region etc. etc. This process of validating a whole permit request takes some time because it has to perform **6850 queries**.

## Performance PostgreSQL

At this moment the application runs on a PostgreSQL database so lets see what the performance is at this point:

```
run       user     system      total        real
#01  10.690000   0.540000  11.230000 ( 14.828312)
#02   9.450000   0.490000   9.940000 ( 13.895988)
#03   9.550000   0.470000  10.020000 ( 14.215028)
```

{% pullquote %}
The average result is that it takes {" 14.312ms for 6850 queries "} on PostgreSQL.
{% endpullquote %}

## Configuring the Rails app to use MemSQL

After installing MemSQL and starting it running on port 3307, I created a new database added the existing data and changed some connection strings for the Rails app.

### mysql2 gem

Because MemSQL is built on top of MySQL you can use all the MySQL client tools to perform operations on MemSQL.

The MemSQL documentation recommended using the memsql2 gem but to do this I had to install MySQL first on my machine.

### Failed to connect?

After installing MySQL and configured the Rails application I got some strange error when starting the application.

```
Failed to connect to database:
  Sequel::AdapterNotFound -> LoadError: dlopen(.rbenv/versions/1.9.3-p194/lib/ruby/gems/1.9.1/gems/mysql2-0.3.11/lib/mysql2/mysql2.bundle, 9): Library not loaded: /usr/local/mysql/lib/libmysqlclient.16.dylib
  Referenced from: /Users/mvanrijn/.rbenv/versions/1.9.3-p194/lib/ruby/gems/1.9.1/gems/mysql2-0.3.11/lib/mysql2/mysql2.bundle
  Reason: image not found - /Users/mvanrijn/.rbenv/versions/1.9.3-p194/lib/ruby/gems/1.9.1/gems/mysql2-0.3.11/lib/mysql2/mysql2.bundle
```

After some googling I found out that on OSX you might need to symlink the dylib manually. **NOTE** Because I've installed a newer version of MySQL I have a `libmysqlclient.18.dylib` instead of the required `libmysqlclient.16.dylib`. Symlinking this file from libmysqlclient.18.dylib to libmysqlclient.16.dylib seemed to work for me.

```
sudo ln -s /usr/local/mysql/lib/libmysqlclient.18.dylib /usr/local/mysql/lib/libmysqlclient.16.dylib
```

### Explain not supported for joins

At this point I am able to connect to the database and perform the benchmark, but I got the next error:

```
ActiveRecord::StatementInvalid: Mysql2::Error: Feature 'EXPLAIN for join queries' is not supported by MemSQL.
```

Apparently I've EXPLAIN queries on in my config and this isn't supported for JOIN queries as MemSQL tells us.

To resolve this you just have to modify/add this line in your `config/environment/development.rb`

```
config.active_record.auto_explain_threshold_in_seconds = nil
```

## MemSQL's first spin!

{% pullquote %}
So after resolving all these issues I was able to run the benchmark, but the {" first run took 3 min and 05 seconds! "} First I thought that this can not be right, but I forgot the fact that it generate's c code from the queries.
{% endpullquote %}

On the MemSQL server I got the following output:

```sql
2908919519 2012-06-20 09:56:16 INFO: Query appdb.'SELECT  `permit_requests`.* FROM `permit_requests`  WHERE `permit_requests`.`id` = @ LIMIT @' compiled in 7306 milliseconds
2908983637 2012-06-20 09:56:16 WARNING: WARN DISABLED LOCKDOWN: BEGIN TRANSACTION
2916240227 2012-06-20 09:56:23 INFO: Query appdb.'SELECT  `categories`.* FROM `categories`  WHERE `categories`.`id` = @ ORDER BY name LIMIT @' compiled in 7203 milliseconds
2924143142 2012-06-20 09:56:31 INFO: Query appdb.'SELECT `questions`.* FROM `questions`  WHERE `questions`.`category_id` = @ ORDER BY position' compiled in 7722 milliseconds

... etc ...
```

As you can see, generating the compiled version for every query toke about 7 seconds.

## Performance MemSQL

So bellow again the same table with results. The first line is the time with generating all the compiled queries:

```
run       user     system      total        real
#01   5.260000   0.360000   5.620000 (183.511099)
#02   3.830000   0.260000   4.090000 (  6.657682)
#03   3.790000   0.260000   4.050000 (  6.613542)
```

{% pullquote %}
The performance boost is quite huge! The average result is {" 6.635ms for 6850 queries "} if we skipping the first result. This means it's like 7.677ms faster!
{% endpullquote %}

### Cold boot of MemSQL

I've also tried the performance when I restart the database server. I just wanted to know if it keeps the data in memory and would take longer to run after a restart.

The result:

```
run       user     system      total        real
#01   5.110000   0.330000   5.440000 (  8.030930)
#02   3.770000   0.260000   4.030000 (  6.580312)
#03   3.780000   0.250000   4.030000 (  6.524884)
```

{% pullquote %}
The first time it toke just a bit longer, but with an average of {" 7.044ms for 6850 queries "} this is still faster than running it on PostgreSQL!
{% endpullquote %}

## Pros and Cons

Here's a list with some pros and cons I noticed while experimenting with MemSQL:

### Pros

- Blazing fast!
- You can use all the MySQL client tools out there. Also because it uses MySQL, troubleshooting might not be an issue because it isn't a new product.

### Cons

- Huge cache folder! After restoring the database (15Mb) MemSQL creates a folder in the `plancache` folder which is **5.5GB**. So performance comes with a price I think.
- Unable to create users. When creating the database, I tried to add a new user but this isn't supported. Maybe this is because I'm using the Developer Edition.
