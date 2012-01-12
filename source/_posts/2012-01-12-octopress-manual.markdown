---
layout: post
title: "octopress manual"
date: 2012-01-12 16:13
comments: false
categories: [CSS3, Sass, Media Queries]
published: false
---

## Git Checkout

```
git clone ***** octopress
cd octopress
git checkout source
git clone ***** _deploy
```

## To create/preview and deploy

```
# create post
rake "new_post['post name here']"

# preview (http://localhost:4000)
rake preview

# deploy
rake generate
rake deploy
```

# Header 1

## Header 2

### Header 3

#### Header 4

##### Header 5

## Format text

this is *bold* text and this is _italic_

## links

### Inline links

An [example](http://example.com/ "The Example")

### Refering links

link [www.toeter.com] [0] or [www.toeter.com]

  [0]: http://www.toeter.com/ "Toeter"
  [www.toeter.com]: http://www.toeter.com/ "Toeter"

## Images

### Octopress style

`{% img [class names (left, right)] /path/to/image [width] [height] [title text [alt text]] %}`

{% img http://placekitten.com/300/500 150 250 'Place Kitten #4' 'An image of a very cute kitten' %}


## Lists

### Ordered

  1.  Foo
  2.  Bar

### Dash/Astriks

  - point 1
  * point 2

### Nested/combined

  * Abacus
    * answer
  * Bubbles
    1. bunk
    2. bupkis
    3. burper

## BlockQuote

### Markdown

> Email-style angle brackets
> are used for blockquotes.

>> And, they can be nested.

### Octopress style

{% blockquote Seth Godin http://sethgodin.typepad.com/seths_blog/2009/07/welcome-to-island-marketing.html Welcome to Island Marketing %}
Every interaction is both precious and an opportunity to delight.
{% endblockquote %}

### PullQuote

{% pullquote %}
Surround your paragraph with the pull quote tags. Then when you come to
the text you want to pull, {" surround it like this "} and that's all there is to it.
{% endpullquote %}


## Underline

---

## Code samples

### full monty

``` ruby Discover if a number is prime http://www.noulakaz.net/weblog/2007/03/18/a-regular-expression-to-check-for-prime-numbers/ Source Article
class Fixnum
  def prime?
    ('1' * self) !~ /^1?$|^(11+?)\1+$/
  end
end
```

### single line

```
$ sudo make me a sandwich
```

### include gist

{% gist 921430 %}

