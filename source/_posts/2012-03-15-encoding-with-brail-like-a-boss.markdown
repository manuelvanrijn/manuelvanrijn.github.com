---
layout: post
title: "Encoding with Brail like a boss"
date: 2012-03-15 21:29
comments: true
categories: [brail, .net, encoding, code]
published: true
---

Today was a one of those days you search hours for a solution, which finally results in a a quite dissapointing answer. However on the end of this journal you don't care about that last part anymore.

<!-- more -->

## How it started

Yesterday a customer reported that there were some weird questionmark signs when generating a PDF. Obviously this is probably caused by some html encoding or something. So me and my [college](http://blog.dltr.nl) desided to fix this minor bug after we solved some major rocket science bug. We thought "let's do something easy to cool down a bit"

## Encoding

 changed the meta Content-Type charset to utf-8, ISO-8859-1
 
## When does it stop!

Ok so at some point all our options/ideas leaded to nothing!

Ofcourse we read [The Absolute Minimum Every Software Developer Absolutely, Positively Must Know About Unicode and Character Sets (No Excuses!)](http://www.joelonsoftware.com/articles/Unicode.html) by Joel Spolsky, but at the header "IT'S NOG THAT HARD" we did wonder why that article was that LONG!



## The solution

The [solution](http://stw.castleproject.org/MonoRail.Brail.ashx#HTML_Encoding_15) .... a `!` instead of a `$` would do the trick....

