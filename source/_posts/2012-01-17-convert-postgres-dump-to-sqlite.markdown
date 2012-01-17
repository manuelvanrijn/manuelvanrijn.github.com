---
layout: post
title: "Convert PostgreSQL dump to SQLite"
date: 2012-01-17 12:04
comments: true
categories: [rails, code, sqlite, postgresql]
published: false
---

Today I'd like to share the steps I take when I want to convert the production [PostgreSQL](http://www.postgresql.org/ "PostgreSQL") database into my development [SQLite](http://www.sqlite.org/ "SQLite") database for a Rails project.

## Steps

### 1. Generate a SQL dump

First we have to create a sql dump on the production server. I use the following command that results in a `dump.sql` file in the current folder:

```
pg_dump --data-only --inserts YOUR_DB_NAME > dump.sql
```

I use the `--data-only` option, so it doesn't generate the schema. Converting the schema to a valid SQLite schema gave

backup je development.sqlite3 en creeer een nieuw leeg schema

`$ rake db:migrate`

in de dump.sql moet het volgende nog even recht gezet worden:

* verwijder de bovenste `SET` statments
* verwijder de `SELECT pg_catalog.setval****` regels
* vervang true door 't'
* vervang false door 'f'

**belangrijk**
voeg aan het begin van dump.sql `BEGIN;` toe en aan het einde van het bestand `END;`. Dit scheelt zo'n 12 minuten van de 12 minuten.

voeg de data toe in je nieuwe lege development.sqlite3 db

```bash
$ sqlite3 development.sqlite3
$ sqlite> delete from schema_migrations;
$ sqlite> .read dump.sql
```
