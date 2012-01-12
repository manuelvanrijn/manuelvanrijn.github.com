---
layout: post
title: "'Disable Rails before/after callback'"
date: 2012-01-12 17:18
comments: true
categories: [rails, code, optimize]
published: false
---

```ruby
Filter.skip_callback(:save, :before, :propagate_inactivity)
```
