---
layout: post
title: "SELECT total_rows FROM database"
date: 2012-07-26 14:34
comments: true
categories: [heroku, postgres, database]
published: true
---

[{% img left /images/posts/heroku-postgres.png 200 200 Heroku Postgres %}](/blog/2012/07/26/select-total-rows-from-database/) On the first of August this year, [Heroku](http://www.heroku.com) will add 2 new plans to their database add-on known as **Dev** and **Basic**. Both plans have a row limit restriction and that made me thinking how I could fetch that number with one simple query.

<!-- more -->

<div class="clearfix"></div>
## TL;DR

Of course you already know the stuff described below so go to [the final query](#final-query).

## Getting the list with tables

For retrieving almost any information about your database you can query the `pg_class` see [Postgres documentation](http://www.postgresql.org/docs/9.1/static/catalog-pg-class.html).

* The `relkind` is set to `r` to query only ordinary table information.
* We are joining the `pg_namespace` table to filter out the `pg_catalog` and `information_schema` tables

As result we'll have on column called `tableName` with all tables from the database

```sql
SELECT
  pgClass.relname AS tableName
FROM
  pg_class pgClass
LEFT JOIN
  pg_namespace pgNamespace ON (pgNamespace.oid = pgClass.relnamespace)
WHERE
  pgNamespace.nspname NOT IN ('pg_catalog', 'information_schema') AND
  pgClass.relkind='r'
```

### Adding the row count

Adding the total number of rows was quite simple after reading the [Postgres documentation](http://www.postgresql.org/docs/9.1/static/catalog-pg-class.html). It seems that the `pg_class` contains a column called `reltuples` that holds the number of rows in the table.

So adding this column to the select seems to do the trick:

```sql
SELECT
  pgClass.relname   AS tableName,
  pgClass.reltuples AS rowCount
FROM
  pg_class pgClass
LEFT JOIN
  pg_namespace pgNamespace ON (pgNamespace.oid = pgClass.relnamespace)
WHERE
  pgNamespace.nspname NOT IN ('pg_catalog', 'information_schema') AND
  pgClass.relkind='r'
```

<a id="final-query"></a>
## Selecting the total count

With this query it was easy to transform it into a SUM for all rowCount's we retrieve. Here's the final query to retrieve the total number of rows from all tables of a Postgres database

```sql
SELECT
  SUM(pgClass.reltuples) AS totalRowCount
FROM
  pg_class pgClass
LEFT JOIN
  pg_namespace pgNamespace ON (pgNamespace.oid = pgClass.relnamespace)
WHERE
  pgNamespace.nspname NOT IN ('pg_catalog', 'information_schema') AND
  pgClass.relkind='r'
```
