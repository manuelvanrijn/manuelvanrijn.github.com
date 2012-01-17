---
layout: post
title: "automate postgres production to local sqlite"
date: 2012-01-18 17:51
comments: true
categories:
published: false
---



```ruby
require 'fileutils'

namespace :db do
  desc 'pull the production postgresql database into the development sqlite'
  task :pull_production do
    Rake::Task['db:download_pg_dump'].invoke
    Rake::Task['db:optimze_pg_dump_for_sqlite'].invoke
    Rake::Task['db:recreate_with_dump'].invoke
  end

  desc 'download the pg_dump content into tmp/dump.sql'
  task :download_pg_dump do
    config = Rails.application.config.database_configuration

    abort "Missing live db config" if config['production'].blank?

    dev  = config['development']
    prod = config['production']

    abort "Development db is not sqlite3" unless dev['adapter'] =~ /sqlite3/
    abort "Production db is not postgresql" unless prod['adapter'] =~ /postgresql/
    abort "Missing ssh host" if prod['ssh_host'].blank?
    abort "Missing database name" if prod['database'].blank?

    # remove the old one
    if File.exists?(pg_dump_file_path)
      File.delete(pg_dump_file_path)
    end

    cmd  = "ssh -C "
    cmd << "#{prod['ssh_user']}@" if prod['ssh_user'].present?
    cmd << "#{prod['ssh_host']} "
    cmd << "pg_dump --data-only --inserts #{prod['database']} > "
    cmd << pg_dump_file_path

    `#{cmd}`
  end

  desc 'remove unused statements and optimze sql for sqlite'
  task :optimze_pg_dump_for_sqlite do
    result = []
    lines = File.readlines(pg_dump_file_path)
    @version = 0
    lines.each do | line |
      next if line =~ /SELECT pg_catalog.setval/  # sequence value's
      next if line =~ /SET /                      # postgres specific config
      next if line =~ /--/                        # comment

      if line =~ /INSERT INTO schema_migrations/
        @version = line.match(/INSERT INTO schema_migrations VALUES \('([\d]*)/)[1]
      end

      # replace true and false for 't' and 'f'
      line.gsub!("true","'t'")
      line.gsub!("false","'f'")
      result << line
    end

    File.open(pg_dump_file_path, "w") do |f|
      # Add BEGIN and END so we add it to 1 transaction. Increase speed!
      f.puts("BEGIN;")
      result.each{|line| f.puts(line) unless line.blank?}
      f.puts("END;")
    end
  end

  task :toeter do
    Rake::Task['db:optimze_pg_dump_for_sqlite'].invoke
    puts @version
  end

  desc 'backup development.sqlite3 and create a new one with the dumped data'
  task :recreate_with_dump do
    # sqlite so backup
    database = Rails.configuration.database_configuration['development']['database']
    database_path = File.expand_path("#{Rails.root}/#{database}")
    # remove old backup
    if File.exists?(database_path + '.backup')
      File.delete(database_path + '.backup')
    end
    # copy current for backup
    FileUtils.cp database_path, database_path + '.backup' if File.exists?(database_path)

    # dropping and re-creating db
    ENV['VERSION'] = @version
    Rake::Task['db:drop'].invoke
    Rake::Task["db:migrate"].invoke

    puts "migrated to version: #{@version}"
    puts "importing..."
    # remove migration info
    system `sqlite3 #{database_path} "delete from schema_migrations;"`
    # import dump.sql
    system `sqlite3 #{database_path} ".read #{pg_dump_file_path}"`

    puts "done!"
  end

  def pg_dump_file_path
    File.expand_path("#{Rails.root}/tmp/dump.sql")
  end
end
```
