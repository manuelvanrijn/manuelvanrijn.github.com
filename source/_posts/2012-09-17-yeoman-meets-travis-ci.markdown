---
layout: post
title: "Yeoman meets Travis CI"
date: 2012-09-17 14:45
comments: true
categories: [code, yeoman, travis-ci, automation, javascript, testing, frontend]
published: true
---

[{% img right /images/posts/yeoman-logo.png 200 200 Yeoman %}](/blog/2012/06/22/integrate-travis-ci-into-grunt/) Today I'd like to show how you can integrate Travis CI within your Yeoman project.

[Travis CI](http://travis-ci.org/) is a hosted continuous integration service for open source projects and [Yeoman](http://yeoman.io) is a robust and opinionated set of tools, libraries, and a workflow that can help developers quickly build beautiful, compelling web apps.

<!-- more -->

## Adding Travis CI support

When Yeoman was released I've tried to add Travis CI support to a test project. After setting everything up there seemed to be a problem with the update check being fired when running tests. This was annoying because Travis CI was waiting for input from the user.

So I've fixed the issue and created a [pull request](https://github.com/yeoman/yeoman/pull/369) that was merged very quick thanks to [Addy Osmani](http://www.addyosmani.com).

Anyway today I saw that there was a new version released which includes my fix and I like to share how you can make your Yeoman project be tested within Travis CI.

### TL;DR;

See [this commit](https://github.com/manuelvanrijn/yeoman-travis-ci/commit/32203d84ca2dbba2faa671e55afede8a1e6666df) with all the changes you have to make to your existing Yeoman project in order to get Travis CI integration.

Or just browse my test project here: [yeoman-travis-ci](https://github.com/manuelvanrijn/yeoman-travis-ci)

### package.json

#### Adding the yeoman dependency

First we have to add the yeoman package as a dependency to our `package.json` file because we can't install yeoman globally on Travis CI.

```yml
"devDependencies": {
  "yeoman": "~0.9.1"
}
```

After adding the dependency and running `npm install` you'll see that yeoman is installed in the `node_modules` folder of your project.

#### Setup npm test script

Travis CI runs the `npm test` command after fetching the project and installing the dependencies. To make npm test run the Yeoman test task, we need to add the following line into our `package.json` file.

```yml
"scripts": {
  "test": "node_modules/.bin/yeoman test --verbose"
}
```

I've added the `--verbose` argument because I'd like to see detailed information when my tests fail, but this is an optional argument.

At this point you should be able to run `npm test` and see Yeoman running your tests.

### Add the .travis.yml file

Finally we have to add a `.travis.yml` file to let Travis CI know what language our project is and on which version(s) of NodeJS it should use to test our project against (note: Yeoman requires >= 0.8).

```yml
language: node_js
node_js:
  - 0.8
  - 0.9
```

## Hook & Push!

The final step is to add the Travis CI hook to you project in GitHub and push your project.

Happy coding!
