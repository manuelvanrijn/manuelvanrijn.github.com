---
layout: post
title: "Installing Android SKD for Eclipse"
date: 2012-04-04 09:53
comments: true
categories: [code, android, eclipse, java, mobile, linux]
published: true
---

{% img right /images/posts/android.jpg 140 140 Android! %} Today I was trying to install the Android SDK for Eclipse for a new project I'll work on. After downloading the necessary packages I got some strange error I couldn't resolve quickly, so I like to share the solution with you guys.

The error Eclipse returned was

```
Cannot complete the install because one or more required items could not be found.
Software being installed: Android Development Tools **** (com.android.ide.eclipse.adt.feature.group ****)
Missing requirement: Android Development Tools **** (com.android.ide.eclipse.adt.feature.group ****)
requires 'org.eclipse.wst.sse.core 0.0.0' but it could not be found
```

<!-- more -->

## What I did to produce the error

I installed Eclipse and the Android SDK tested the Android SDK by creating an AVD.

After running Eclipse I tried to add the `https://dl-ssl.google.com/android/eclipse/` url to my Available Software Sites to install the Eclipse package. After selecting all the Developer Tools in the filtered list for this Software Site and hitting Next I received the error as I added above

## The solution

After some googling I found a tip that I needed to add another Software Site to Eclipse.

After adding `http://download.eclipse.org/releases/indigo` and retrying to install the Android Developer Tools all went well.
