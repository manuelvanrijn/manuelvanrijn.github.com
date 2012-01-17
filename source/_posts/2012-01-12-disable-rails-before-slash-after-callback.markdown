---
layout: post
title: "Disable Rails callback"
date: 2012-01-12 21:30
comments: true
categories: [rails, code, optimize, performance]
published: true
---

{% img left /images/posts/skip-step.png 240 240 Skip Step %}

Today I've had some difficulties with a Rails migration, that took ages to complete. After some debugging I figured out that the issues was because of the `before_save` and `after_save` callbacks.

Of course we could remove the before and after save callbacks to speed up the process, but I don't like the idea of releasing a model that misses these behavior. I mean they're not there for no reason right?

After some googling I found a quick and easy way to disable the callbacks without modifying the model. Here's an simple example of a model class and a migration the disable the callbacks only within the migration.

## Code solution

``` ruby app/models/some_model.rb
class SomeModel < ActiveRecord::Base
  before_save :before_action
  after_save :after_action
  
  private
    def before_action
      # heavy calculation/queries
      puts "you shouldn't see this"
    end
  
    def after_action
      # another heavy calculation/queries
      puts "you shouldn't see this"
    end
end
```

``` ruby db/migrations/20120112092136_update_some_model.rb
class UpdateSomeModel < ActiveRecord::Migration
  def up
    # disable the before_save
    SomeModel.skip_callback(:save, :before, :before_action)
    # disable the after_save
    SomeModel.skip_callback(:save, :after, :after_action)
    
    SomeModel.all.each do | obj |
      # .. modify obj ...

      obj.save
      # you shouldn't see the puts defined in the SomeModel before and after actions
    end
  end

  def down
  end
end
```

## Supported callbacks

In the above example I show only two rails callbacks. Here's a list of the supported callbacks you also can disable with it.

  - `:after_initialize`
  - `:after_find`
  - `:after_touch`
  - `:before_validation`
  - `:after_validation`
  - `:before_save`
  - `:around_save`
  - `:after_save`
  - `:before_create`
  - `:around_create`
  - `:after_create`
  - `:before_update`
  - `:around_update`
  - `:after_update`
  - `:before_destroy`
  - `:around_destroy`
  - `:after_destroy`
  - `:after_commit`
  - `:after_rollback`

Hope these could help you out some day =)