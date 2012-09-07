---
layout: post
title: "Integrate Travis CI into Grunt"
date: 2012-06-22 15:04
comments: true
categories: [code, grunt, travis-ci, qunit, javascript, jquery, testing]
published: true
---

[{% img right /images/posts/travis-grunt.jpg 200 200 Code Fail %}](/blog/2012/06/22/integrate-travis-ci-into-grunt/) Today I've done some research how to get my latest jQuery plugin ([jquery-numeric_input](http://manuel.manuelles.nl/jquery-numeric_input/)) to be tested automatically with the help of [Travis CI](http://travis-ci.org/). Travis CI is a hosted continuous integration service for open source projects.

Because I've just finished the latest functionality of the plugin I wanted to add the project to Travis CI, just like I do with all my other projects (the ones with unit tests).

<!-- more -->

## Grunt

For the development of the plugin I used [Grunt](http://gruntjs.com/) which is a task-based command line build tool for JavaScript projects. Starting a new project is as easy as running `grunt init:jquery` and you have a entire generated jQuery plugin project.

By default this command configures a jQuery plugin, QUnit test suite, sample plugin, sample tests and some grunt tasks you can use. Running the `grunt` command in your project folder will execute the default task meaning:

1. lint - Analysis your code with [JSLint](https://github.com/douglascrockford/JSLint/)
2. qunit - Runs your QUnit tests using [PhantomJS](http://phantomjs.org/) (so it supports the exit code for success or failure)
3. concat - Concatenate your project files
4. min - Minify your project files using [UglifyJS](https://github.com/mishoo/UglifyJS/)

Really easy to get started coding instead of setting up your environment :)

## Integration with Travis CI

Ok so at this point I want to add my project to Travis CI so that it tests my suite when pushing new branches/commits to Github. To do this we have to make a few changes.

### TL;DR

Here's the [travis ci integration commit](https://github.com/manuelvanrijn/jquery-numeric_input/commit/188e1fa53ca565b2901f0ac4b6de102b6349577b) from my plugin with all the steps described below.

### Registering the travis task

I'd like to keep the Grunt tasks organized so I register a new task called `travis` below the `default` task.

```javascript grunt.js
// Default task.
grunt.registerTask('default', 'lint qunit concat min');

// Travis CI task.
grunt.registerTask('travis', 'lint qunit');
```

As you can see I only added `lint` and `qunit` because we don't have to concatenate or minify a new build of our plugin. At this point you are able to run `grunt travis` from the command line.

### Adding the dependency

Now we have to add Grunt as a dependency to our `package.json` file so that npm running on Travis CI knows it has to install Grunt. Mine dependencies block looks like this:

```json package.json
"dependencies": {
  "jQuery": "~1.7.3"
},
"devDependencies": {
  "grunt": "~0.3.14"
},
```

### Adding the npm task

Travis CI runs `npm test` after it fetched your project and installed the dependencies, so we need to add this task to the `package.json` file.

```json package.json
"scripts": {
  "test": "grunt travis --verbose"
}
```

I've added the `--verbose` option, so we'll see more output of what is going on.

### Adding the .travis.yml

Every Travis CI project needs to have a `.travis.yml` file in the root of the project folder, so it know what platform and version it should use to build/test your project. Here's the one I used:

```yaml .travis.yml
language: node_js
node_js:
  - 0.6
  - 0.8
  - 0.9
```

## Ready to go!

Alright after these changes your project is ready to be continuously builded with Travis CI. But don't forget to setup the Service Hook on Github!

## Update

Thanks to **Ryan Seddon** and **Ariya Hidayat** for noticing me that the `before_script` section for running PhantomJS on the headless server on Travis CI isn't needed. See [this link](http://about.travis-ci.org/docs/user/gui-and-headless-browsers/)
