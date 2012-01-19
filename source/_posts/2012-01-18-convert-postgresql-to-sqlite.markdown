---
layout: post
title: "Convert PostgreSQL to SQLite"
date: 2012-01-18 00:30
comments: true
categories: [rails, code, sqlite, postgresql]
published: true
---

{% img right /images/posts/postgres-to-sqlite.png 220 220 Postgres to SQLite %}

Today I'd like to share the steps I take when I need to convert a [PostgreSQL](http://www.postgresql.org/ "PostgreSQL") database into a [SQLite](http://www.sqlite.org/ "SQLite") database.

Commonly I have to do this when a [Ruby on Rails](http://rubyonrails.org/ "Ruby on Rails") application is in production and I have to check some issues with the production data. In the production environment we usually use a PostgreSQL database and for developing I use a SQLite database, so we need some conversion.

<!-- more -->

## Short story a.k.a I know what I'm doing.

  1. Create a dump of the PostgreSQL database.
  `ssh -C username@hostname.com pg_dump --data-only --inserts YOUR_DB_NAME > dump.sql`
  2. Remove/modify the dump.
    * Remove the lines starting with `SET`
    * Remove the lines starting with `SELECT pg_catalog.setval`
    * Replace true for 't'
    * Replace false for 'f'
    * Add `BEGIN;` as first line and `END;` as last line
  3. Recreate an empty development database.
  `bundle exec rake db:migrate`
  4. Import the dump.
```bash
sqlite3 db/development.sqlite3
sqlite> delete from schema_migrations;
sqlite> .read dump.sql
```
## Longer story a.k.a please explain a little more.

So basically you can do the following 4 major steps to convert the PostgreSQL database into a SQLite database.

### 1. Generate a SQL dump

First we have to create a sql dump on the production server. I use the following command that results in a `dump.sql` file in the current folder:

```bash
pg_dump --data-only --inserts YOUR_DB_NAME > dump.sql
```

I use the `--data-only` option, so it doesn't generate the schema. Converting the pg_dump generate schema to a valid SQLite schema gave me a lot of difficulties so I chose to generate the schema with the `rake db` task (we'll discuss this in the next step).

After you created the dump, you have to download/transfer/mail/etc. that file so you have local access to it.

#### Trick: Got ssh access?

If you have ssh access, you can also run the following command, which will output the file directly on you local drive

```bash
ssh -C username@hostname.com pg_dump --data-only --inserts YOUR_DB_NAME > dump.sql
```

### 2. Modify the dump.sql

There are a few manual find/replace and delete action's you have to perform on the `dump.sql` file by hand.

#### 2.1 Remove the `SET` statements at the top

You will see some statements at the top of the file like `SET statement_timeout = 0;` and `SET client_encoding = 'SQL_ASCII';` etc. Remove all of these lines that start with `SET`, because SQLite doesn't use these.

#### 2.2 Remove the setval sequence queries

Under the `SET` queries you'll see some queries to set the correct sequence for auto incrementing the id's. SQLite doesn't keep these value's in a catalog and must be removed to prevent errors.

Remove all the line's that look like `SELECT pg_catalog.setval('MY_OBJECT_id_seq', 10, true);`

#### 2.3 Replace true => 't' and false => 'f'

The `pg_dump` generate's `true` and `false` as value's for the `INSERT INTO` statements. If we want to import these to SQLite we have to replace these to 't' and 'f'.

```sql
-- These:
INSERT INTO table_name VALUES (1, true, false);
-- Should be replace to:
INSERT INTO table_name VALUES (1, 't', 'f');
```

#### 2.4 Transactions. Make it fast!

The first time I imported the dump (that was 2 mb) it took like 12 minutes to complete! After some googling I found out that SQLite's default behavior is putting each statement into a transaction, which seems to be the time waster (after the fix it toke 12 seconds).

To prevent this behavior you can run the script within 1 transaction by specifying `BEGIN;` at the top of the `dump.sql` and `END;` at the end of the file.

So you would have:

```sql
BEGIN;
-- a lot of INSERT INTO statments
END;
```

### 3. Recreate the development database

So now we have fetched the production data from the PostgreSQL database, we need to recreate the `development.sqlite3` database.

Make a backup and run the migration task

```bash
mv db/development.sqlite3 db/development.backup.sqlite3
bundle exec rake db:migrate
```

#### Side note when migrating

You must run the migration __until__ the migrated version that is active on the production database. If not, you could have the situation where you have dropped a column and can't import the dump because the data depends on that column.

Check the `dump.sql` for the latest version number in the `schema_migrations` table and migrate to that version.

For example for the version `20121701120000` you would do:

```bash
bundle exec rake db:migrate VERSION=20121701120000
```

### 4. Import the dump

The final step is importing the dump file. To do this we have to execute the following command within a terminal:

```bash
sqlite3 db/development.sqlite3
sqlite> delete from schema_migrations;
sqlite> .read dump.sql
```

As you can see we first remove the records from the `schema_migrations` table, because these are also included in the `dump.sql`. Of course you could also remove the lines from the file, but I prefer this way.

The `.read` command just execute's all the lines within the specified file.

## Result

And that's it! You now have a stuffed `development.sqlite3` database with all the production data out of the PostgreSQL database.

