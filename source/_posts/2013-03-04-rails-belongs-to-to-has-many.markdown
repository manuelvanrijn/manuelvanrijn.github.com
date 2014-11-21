---
layout: post
title: "Rails belongs_to to has_many"
date: 2013-03-04 17:19
comments: true
categories: [rails, migrations, relations, associates, ruby, ActiveRecord]
published: true
---

[{% img left /images/posts/habtm-migration.jpg 250 250 Migration what?! %}](/blog/2013/03/04/rails-belongs-to-to-has-many/) Today I had to change a `belongs_to` associate to an `has_and_belongs_to_many` in a Rails project I'm currently working on. Not that hard you would say, but there were some catches for deploying/migrating these changes to the production environment.

<br />
In this post I'd like to explain how to achieve this without having to do multiple deployments of your model to maintain a consistent database schema and model.

<!-- more -->

## TL;DR;

Here you can find [the final migration](#final-migration). Note that this example could also be solved with plain sql instead of using ActiveRecord, but there might be cases were you depend on the ActiveRecord associates. In these cases this is a great workaround

## The problem

Let's say we have a Article that belongs to a Category. Our model would look like something like this:

```ruby
class Article < ActiveRecord::Base
  attr_accessible :title, :content
  belongs_to :category
end
```

And the executed migration would look like this:

```ruby
class CreateArticle < ActiveRecord::Migration
  def change
    create_table :articles do |t|
      t.string :title
      t.text :content
      t.references :category
    end
  end
end
```

## The initial migration

First let's create a new table that can hold the associate between multiple Articles and Categories. The migration would look like this:

```ruby
class MultipleCategoriesForArticles < ActiveRecord::Migration
  def up
    create_table :articles_categories, :id => false do |t|
      t.references :article, :category
    end
  end

  def down
    drop_table :articles_categories
  end
end
```

At this point all is fine and we can migrate the database without any problems. Only this doesn't add the current category to the new collection of categories.

### Moving the belongs_to associate

Here's were the problem actually starts, because to move the category to the categories collection for the articles, we have to define the `belongs_to` but also the `has_and_belongs_to_many` associate on the `Article` model.

I don't like this approach because we'll have to define both these associates and the `belongs_to` should be removed in the next release. So how to deal with this?

<a id="final-migration"></a>
## The fix

The fix is quite simple. We remove the `belongs_to` associate from the model and only define the new `has_and_belongs_to_many` associate and within the migration we extend the model with the "old" `belongs_to` associate so we can use this within the migration.

So our files will look like this

```ruby app/models/article.rb
class Article < ActiveRecord::Base
  attr_accessible :title, :content
  has_and_belongs_to_many :categories
end
```

```ruby db/migrate/20130305120000_multiple_categories_for_articles.rb
class MultipleCategoriesForArticles < ActiveRecord::Migration
  def up
    create_table :articles_categories, :id => false do |t|
      t.references :article, :category
    end

    # define the old belongs_to category associate
    Article.class_eval do
      belongs_to :old_category,
                 :class_name => "Category",
                 :foreign_key => "category_id"
    end

    # add the belongs_to category to the has_and_belongs_to_many categories
    Article.all.each do | article |
      unless article.old_category.nil?
        article.categories << article.old_category
        article.save
      end
    end

    # remove the old category_id column for the belongs_to associate
    remove_column :articles, :category_id
  end

  def down
    add_column :articles, :category_id, :integer

    Article.class_eval do
      belongs_to :new_category,
                 :class_name => "Category",
                 :foreign_key => "category_id"
    end

    Article.all.each do | article |
      # NOTE: we'll grabe the first category (if present), so if there are more, these will be lost!
      article.new_category = article.categories.first unless article.categories.empty?
      article.save
    end

    drop_table :articles_categories
  end
end
```

