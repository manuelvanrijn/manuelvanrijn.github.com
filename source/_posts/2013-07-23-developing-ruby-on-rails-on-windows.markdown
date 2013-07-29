---
layout: post
title: "Developing Ruby on Rails on Windows"
date: 2013-07-23 23:06
comments: true
categories: [rails, windows, vagrant, chef, recipes, ruby, development, setup, howto, box, virtual machine, ubuntu, cookbooks, virtualbox, rbenv, ruby-build, git, sqlite, mysql, postgresql, node.js]
published: true
---

[{% img right /images/posts/developing-ruby-on-rails-on-windows.png 250 250 Develop Ruby on Rails on Windows %}](/blog/2013/03/04/rails-belongs-to-to-has-many/) I do a lot of C# web development which means I have to work with Visual Studio on Windows. I usually grab the Mac to do my Ruby on Rails development, but it would be easier not to switch every time I have to do C# and Ruby development.

A while ago I tried several things getting a fast Rails experience on Windows, but it failed. It was just too slow ... until now! In this post I'd like to share my current setup which I plan to use as long as it pleases me.

<!-- more -->

## What are we going to do?

We are going to install [Vagrant](http://www.vagrantup.com/) which uses [VirtualBox](https://www.virtualbox.org/) to build a complete development environment. Also we will create and use some [recipes for Chef](http://community.opscode.com/) to automatically install and configure the development environment. Think of installing Ruby, PostgreSQL, and MySQL etc. automatically.

## Installing Vagrant and VirtualBox

First let's setup our Windows machine with the needed software.

1. Install [VirtualBox](https://www.virtualbox.org/wiki/Downloads)
2. Install [Vagrant](http://downloads.vagrantup.com/)
3. After reboot add the VirtualBox folder to your PATH variable<br />(something like: `c:\Program Files\Oracle\VirtualBox`)
4. Install [a base box](http://www.vagrantbox.es/) (I used Ubuntu precise 64 VirtualBox) with the following command
`vagrant box add precise64 http://files.vagrantup.com/precise64.box`

## Prepare your Rails project

Next we have to configure our project to create a box from the base box and make it install all our goodies we need to run our Rails application.

### Setup Chef (Librarian)

I'm using a gem called [librarian-chef](https://github.com/applicationsonline/librarian-chef) which actually is a bundler for your Chef recipes.

To install run the following from the root of you project:

``` bash
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

I know we are trying to create a non-Ruby Windows solution but we do need this gem to work on Windows to complete our setup with Chef. If you really don't want Ruby being installed on Windows, you should do the above steps on UNIX or OSX and also have to commit the cookbooks. This means we're back at 1900 where everybody stuffed there vendor folder with code they found... If you do want to perform this on Windows and haven't already installed Ruby, you should do this now (I suggest [RubyInstaller](http://rubyinstaller.org/downloads/)).

### Configure Chef

The command `librarian-chef init` created the file `chef/Cheffile` where we can add our cookbooks. Mine contains the following:

``` ruby <strong>chef/Cheffile</strong>
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

As you can see I've setup Git, Ruby with rbenv, SQLite, MySQL, PostgreSQL and node.js for the asset pipeline. The `apt` cookbook is for running `apt-get dist-upgrade` and the cookbook `database` will be used later to create users to access the databases.

You could skip one or two of the database servers if you like, but it can't harm if you do install them all and don't use them.

#### Downloading the cookbooks

Now that we have defined our cookbooks we need to run our final command so that Chef downloads them. Just as you would do with bundler we now run the following command from our `chef/` folder

``` bash
librarian-chef install
```

### Adding our custom recipes

I had some difficulties with running `bundle install` because the Vagrant user didn't have permissions on the rbenv gem folder, so I had to add a custom recipe for this. Also I've added a recipe to modify the `pg_hba.conf` file of PostgreSQL so that we can connect with the virtual machine. Finally I've created a file to add our database users with full permissions. Here are the 3 files I added:

``` ruby <strong>chef/site-cookbooks/database/recipes/default.rb</strong>
# Setup database servers
#
# 1. Setup MySQL and PostgreSQL
# 2. Create a rails user with a blank password with full control

# MySQL
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

# PostgreSQL
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

``` ruby <strong>chef/site-cookbooks/database/templates/default/pg_hba.conf.erb</strong>
local   all             all                                     trust
host    all             all             127.0.0.1/32            trust
host    all             all             ::1/128                 trust
```

``` ruby <strong>chef/site-cookbooks/postinstall/recipes/default.rb</strong>
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

### Adding our role

Next we have to create a role that defines what recipes should be executed in what order. Create the file `chef/roles/rails-dev.rb` containing the following content:

``` ruby <strong>chef/roles/rails-dev.rb</strong>
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

If you removed some cookbooks from the `chef/Cheffile` you should also remove them here.

### Setup Vagrant

Next we need to create a `Vagrantfile` that contains the configurations for creating a box from the base box. Create the config file by running:

``` bash
vagrant init precise64
```

Note: `precise64` is the name we used with the `vagrant add` command from step 4 above.

#### Modify the Vagrantfile

Here's the stripped version of how my file looks like:

``` ruby Vagrantfile
Vagrant.configure("2") do |config|
  config.vm.box = "precise64"

  config.vm.box_url = "http://files.vagrantup.com/precise64.box"

  config.vm.network :forwarded_port, guest: 3000, host: 3000  # forward the default rails port
  config.vm.network :forwarded_port, guest: 3306, host: 3306  # forward the MySQL port
  config.vm.network :forwarded_port, guest: 5432, host: 5432  # forward the PostgreSQL port

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

I had some troubles installing the (currently) latest version `11.4.4` of Chef because of a bug with Ruby v1.8.7, which is the default on a the base box. [This problem](https://github.com/opscode/chef/pull/734) should be solved soon but until then I've added the `gem install chef --version 11.4.2` line to install a working (but a bit older) version.

## Ready to rock!

Alright we're ready. We now can boot up Vagrant which will take some time the first time because it will create a new box from the base box and install all the cookbooks. From then you're able to connect with the box and use it just like you would do on UNIX or OSX.

``` bash
# starts and setups the box
vagrant up

# ... now we wait ...

# connect to the box with ssh
vagrant ssh

# install gems and run the server on the box
cd /vagrant
bundle install
bundle exec rails s
```

### What we've created

A VirtualBox with Ubuntu precise 64 installed and containing the following software/configurations:

* Software:
  1. rbenv with ruby-build
  2. ruby-2.0.0-p247
  3. Git
  4. SQLite
  5. MySQL with user: `rails` with a empty password
  6. PostgreSQL with user: `rails` with a empty password
  7. node.js
* Ports:
  1. `localhost:3000` goes to `vagrant-box:3000`
  2. `localhost:3306` goes to `vagrant-box:3306`
  3. `localhost:5432` goes to `vagrant-box:5432`

## Develop workflow

If you have to run specific Ruby (on Rails) tasks like `rspec` or `bundle exec rails g migration` you should do this within the ssh session of your box.

Editing files can be done on your Windows machine, because the root of you project is directly binded to the `/vagrant/` folder on the box.
