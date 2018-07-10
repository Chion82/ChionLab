---
title: Using Gulp to Simplify Front end Development Workflow
date: 2016-01-20 01:20:18
tags:
- front-end
categories:
- 学习笔记
- 前端
---

# What is Gulp
Gulp is a streaming build system which is usually used to simplify front-end development workflow, such as automatically minifying JavaScript or compiling LESS. In this tutorial, you'll learn the basic usage of Gulp.

# Preparation
To make sure you'll have fun following my instructions, I assume you:

* have node.js & npm installed
* have basic knowledge of node.js scripting.

# Installing Gulp
Let's start with the installation of Gulp.  
Run the following command in your shell terminal. The installation requires sudo or root previlege and you may be required to enter the password.  
``` bash
$ sudo npm install --global gulp
```

# Project Setup
`cd` into the root directory of your project and run the following command, which will save the Gulp dependencies in your project directory.
``` bash
$ npm install --save-dev gulp
```

# Create gulpfile.js
At the root of your project, create a `gulpfile.js` containing the following code.

``` javascript
var gulp = require('gulp');

gulp.task('mytask', function() {  
 	//All task code places here  
	console.log('Hello World!');  
});
```

The code above defines a gulp task named "mytask" with the detailed commands defined in the callback function as the second parameter passed to the `gulp.task()` method. When running this task, `console.log('Hello World!');` will be executed.

# Testing
Now you should be able to run `mytask` using the following command:  
``` bash
$ gulp mytask
```
Assume the root directory of your project is `~/project`, the output should be like:
``` bash
➜  project  gulp mytask
[21:14:25] Using gulpfile ~/project/gulpfile.js
[21:14:25] Starting 'mytask'...
Hello World!
[21:14:25] Finished 'mytask' after 62 μs
```
You can always run specific tasks by executing `gulp <task> <other_task>`

# Basic File Streaming
In this section we'll use Gulp's streaming system which is its primary function.  
We will use `Gulp.src()`, `Gulp.dest()`, `readable.pipe()` to implement a basic JavaScript source file copying program using Gulp.  
For detailed API doc of the methods above, please refer to [Gulp API doc](https://github.com/gulpjs/gulp/blob/master/docs/API.md) and [Node.js:Stream](https://nodejs.org/api/stream.html#stream_readable_pipe_destination_options).  

* Create a directory named `js` at the root of your project (Assume you created this directory: `~/project/js`) and place some JavaScript files in it.  
* Add the following code at the end of `gulpfile.js`  
``` javascript
gulp.task('copyjs',function(){
        gulp.src('js/*.js')
           .pipe(gulp.dest('dest'));
});
```
* At the root of the project which is `~/project`, run:
``` bash
$ gulp copyjs
```
* Check out `~/project/dest` to which you'll find all js files in `~/project/js` are copied.

# Using Gulp to Minify JS
Next we'll use Gulp to do some amazing tasks which bring great convenience for front-end development.  
Let's start with JavaScript minifying.   
To make Gulp powerful enouth to do this job, we must install some plugins of Gulp. Here we'll use **gulp-uglify**.(For more amazing gulp plugins, check out [Gulp Plugins](http://gulpjs.com/plugins/))  

* Back to `~/project`, run the following command to install **gulp-uglify**.
``` bash
$ npm install --save-dev gulp-uglify
```
* Replace `gulpfile.js` with the following code.
``` javascript
var gulp = require('gulp');
var uglify = require('gulp-uglify');

gulp.task('minifyjs', function(){
        gulp.src('js/*.js')	//Get the stream of the source file
                .pipe(uglify())//Pass the stream to the uglify module to minify all JS files.
                .pipe(gulp.dest('build'));//Pass the stream to the destination directory which is ~/project/build
});
```
* Exucute the task by running:
``` bash
$ gulp minifyjs
```
* Check out `~/project/build`. All minified JavaScript source files are placed here!

# Using Gulp.watch()
Sometimes we want the JS files to be automatically minified everytime we modify them and `Gulp.watch()` will do the trick.  
`Gulp.watch()` allows us to implement a daemon to monitor file modifications and automatically execute specific tasks every time the modifications are made.  

* Add the following code at the end of `gulpfile.js`.
``` javascript
gulp.task('watchjs', function(){
        gulp.watch('js/*.js',['minifyjs']);	//Watch all *.js files under ~/project/js directory and run task "minifyjs" when files are modified
});
```
* Execute the daemon task by running:
``` bash
$ gulp watchjs
```
* Now, JS files will be automatically minified every time you modify them.
