---
layout: post
title: "Develop Ruby on Rails on Windows"
date: 2013-07-23 23:06
comments: true
categories: [rails, windows, vagrant, chef, recipes, ruby, development, setup, howto]
published: true
---

[{% img left /images/posts/habtm-migration.jpg 250 250 Migration what?! %}](/blog/2013/03/04/rails-belongs-to-to-has-many/) At my work we do alot of C# web development which means I have to work with Visual Studio on Windows. I usually grab the Mac to do my Ruby on Rails development, but it would be easier not to switch everytime I have to do C# and Ruby development.

A while ago I tried several things getting a fast Rails experience on Windows, but it failed. It was just to slow ... untill now! In this post I'd like to share my current setup which I plan to use as long as it pleases me.

<!-- more -->

## What are we going to do

We are going to install [Vagrant](http://www.vagrantup.com/) which uses [VirtualBox](https://www.virtualbox.org/) to build a complete development enviroment. Also we will create and use some recipes with [Chef]() to automatically install and configure the development enviroment. Think of installing Ruby, Postgresql, MySQL etc. automatically.

## Installing Vagrant and VirtualBox

First let's setup our Windows machine with the needed software.

1. Install [VirtualBox](https://www.virtualbox.org/wiki/Downloads)
2. Install [Vagrant](http://downloads.vagrantup.com/)
3. After reboot add the VirtualBox folder to your PATH variable<br />(something like: `c:\Program Files\Oracle\VirtualBox`)
4. Install [a box](http://www.vagrantbox.es/) (I used Ubuntu precise 64 VirtualBox) with the command
`vagrant add precise64 http://files.vagrantup.com/precise64.box`

## Prepare your Rails project

Next we have to configure our project to create a box from the installed box and make it install all our goodies we need to run our Rails application.

### Setup Chef (Librarian)

I'm using a gem called [librarian-chef](https://github.com/applicationsonline/librarian-chef) which actually is a bundler for your Chef recipes.

To install run the following from the root of you project:

```shell
# Install librarian-chef
gem install librarian-chef

# Add its work directories to your gitignore
echo chef/cookbooks >> .gitignore
echo chef/tmp >> .gitignore
# OPTIONAL: you can add the Cheffile.lock which holds the currently installed versions
#           just like bundler does
echo chef/Cheffile.lock >> .gitignore

# Create the Librarian config file and needed folders
mkdir -p chef
cd chef
mkdir -p roles
mkdir -p site-cookbooks
librarian-chef init
```

#### WUT?! gem install librarian-chef?

I know we are trying to create a non-Ruby Windows solution but we do need this gem to work on Windows to complete our setup with Chef. If you realy don't want Ruby being installed on Windows, you should do the above steps on Unix or OSX and also have to commit the cookbooks. This means we're back at 1900 where everybody stuffed there vendor folder with code they found... If you do want to perform this on Windows and haven't already installed Ruby, you should do this now (I suggest [RubyInstaller](http://rubyinstaller.org/downloads/)).

### Configure Chef

At this point we have the file `chef/Cheffile` where we can add our cookbooks. Mine contains the following:

```ruby
#!/usr/bin/env ruby
#^syntax detection

site 'http://community.opscode.com/api/v1'

cookbook 'apt'
cookbook 'git'
cookbook 'sqlite'
cookbook 'mysql'
cookbook 'postgresql'
cookbook 'database'
cookbook 'nodejs'
cookbook 'build-essential'
cookbook 'ruby_build'
cookbook 'rbenv', :git => 'git://github.com/fnichol/chef-rbenv.git', :ref => 'v0.7.2'
```

As you can see I've setup git, ruby with rbenv, sqlite, mysql, postgresql and nodejs for the asset pipeline. The `apt` cookbook is for running `apt-get dist-upgrade` and the cookbook `database` will be used later to create users to access the databases.

You could skip one or two of the database servers if you like, but it can't harm if you do install them all and don't use them.

### Add our custom recipes

I had some difficulties with running `bundle install` because the vagrant user didn't have permissions on the rbenv folder, so I had to added a custom recipe for this. Also I've added a recipe to modify the `pg_hba.conf` file of postgresql so that we can connect with the virtual machine. Here's the result:

##### chef/site-cookbooks/database/recipes/default.rb

``` ruby Discover if a number is prime http://www.noulakaz.net/ Source Article
# Setup database servers
#
# 1. setup mysql and postresql
# 2. create a rails user with a blank password with full control

# mysql
mysql_connection_info = {
  :host     => "localhost",
  :username => 'root',
  :password => node['mysql']['server_root_password']
}
mysql_database_user 'rails' do
  connection mysql_connection_info
  password ''
  action :grant
end

# postgresql
template "#{node[:postgresql][:dir]}/pg_hba.conf" do
  source "pg_hba.conf.erb"
  notifies :restart, "service[postgresql]", :immediately
end

postgresql_connection_info = {
  :host     => "localhost",
  :password => node['postgresql']['password']['postgres']
}

postgresql_database_user 'rails' do
  connection postgresql_connection_info
  privileges [:Superuser]
  password ""
  action :create
end
```

##### chef/site-cookbooks/database/templates/default/pg_hba.conf.erb

```
local   all             all                                     trust
host    all             all             127.0.0.1/32            trust
host    all             all             ::1/128                 trust
```

##### chef/site-cookbooks/postinstall/recipes/default.rb

```
group "rbenv" do
  action :create
  members "vagrant"
  gid 1100
  append true
end

bash "chgrp and chmod" do
  user "root"
  cwd "/usr/local"
  code <<-EOH
    chgrp -R rbenv rbenv
    chmod -R g+rwxX rbenv
  EOH
end
```

### Add our role

Next we have to create a role that defines what recipes should be executed in what order. Create the file `chef/roles/rails-dev.rb` containing the following content:

```
name "rails-dev"
description "setup for ruby on rails core development"
run_list(
  "recipe[apt]",
  "recipe[git]",
  "recipe[sqlite]",
  "recipe[mysql::client]",
  "recipe[mysql::ruby]",
  "recipe[mysql::server]",
  "recipe[postgresql::ruby]",
  "recipe[postgresql::server]",
  "recipe[nodejs::install_from_binary]",
  "recipe[ruby_build]",
  "recipe[rbenv::system]",
  "recipe[rbenv::vagrant]",
  "recipe[database]",
  "recipe[postinstall]"
)
```

If you removed some cookbooks from the `Cheffile` you should also remove these from here.

### Prepare the Chef recipes

Now that we have defined our cookbooks we need to run our final command so that Chef downloads these. Just as you would do with bundler we now run the following command from our `chef/` folder

```
librarian-chef install
```

### Setup Vagrant

Next we need to create a `Vagrantfile` where we setup the box vagrant needs to create for our project. First create the config file by running:

```
vagrant init precise64
```

Note: `precise64` is the name we used with the `vagrant add` command from step 4 above.

#### Modify the Vagrantfile

Here's the stripped version of how my file looks like:

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "precise64"

  config.vm.box_url = "http://files.vagrantup.com/precise64.box"

  config.vm.network :forwarded_port, guest: 3000, host: 3000  # default rails port
  config.vm.network :forwarded_port, guest: 3306, host: 3306  # mysql
  config.vm.network :forwarded_port, guest: 5432, host: 5432  # postgresql

  config.vm.provision :shell, :inline => "gem install chef --version 11.4.2"

  config.vm.provision :chef_solo do |chef|
    chef.cookbooks_path = ["chef/cookbooks", "chef/site-cookbooks"]
    chef.roles_path     = [[:host, "chef/roles"]]

    chef.add_role "rails-dev"
    chef.json = {
      "mysql" => {
        "server_root_password"   => "",
        "server_debian_password" => "",
        "server_repl_password"   => ""
      },
      "postgresql" => {
        "password" => {
          "postgres" => ""
        }
      },
      "rbenv" => {
        "global" => "2.0.0-p247",
        "rubies" => ["2.0.0-p247"],
        "gems" => {
          "2.0.0-p247" => [
            { "name" => "bundler" }
          ]
        }
      }
    }
  end
end
```

#### One catch

I had some troubles installing the current latest version 11.4.4 of chef because of a bug with ruby 1.8.7 which is default on a the clean box. [This problem](https://github.com/opscode/chef/pull/734) should be solved soon but untill then I've added the following line to install a working (but a bit older) version:

```
config.vm.provision :shell, :inline => "gem install chef --version 11.4.2"
```

## Ready to rock!

Alright we're ready. We now can bootup our vagrant which will take some time the first time because it will install all our cookbooks. From then you can connect with box and use it just like you would do on unix/osx

```
# starts and setups the box
vagrant up
# connect to the box with ssh
vagrant ssh

# install gems and run the server
cd /vagrant
bundle install
bundle exec rails s
```

### What you've created

Software:
* rbenv withruby-build
* ruby-2.0.0-p247
* git
* sqlite
* mysql with user: 'rails' password: ''
* postgresql with user: 'rails' password: ''
* nodejs

Ports:
* localhost:3000 goes to vagrant-box:3000
* localhost:3306 goes to vagrant-box:3306
* localhost:5432 goes to vagrant-box:5432

### Develop workflow

If you have to run specific rails tasks like `rspec` or `bundle exec rails g migration` you should do this within the ssh session of your box.

Editing files can be done on your Windows machine, because the root of you project is directly binded to the `/vagrant/` folder on the box.
