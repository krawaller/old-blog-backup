---
type: post
title: "A library webworker wrapper"
date: 2017-03-07
tags: [webworker]
author: David
excerpt: Presenting a tool to create an asynchronous webworker version of a library
---


### The premise

Imagine you have a library, consisting of an **object with a bunch of methods**. Some or all of those are heavy, and will lock up the thread for a while.

```javascript
var myHeavyLib = {
  aHeavyMethod: function(arg1,arg2){
    // extremely heavy computing left out here
    return arg1 + arg2;
  },
  anotherHeavyMethod: function(arg1,arg2){
    // extremely heavy computing left out here
    return arg1 + arg2;
  },
  // etc
};
```

If we're using this library in a web app, this won't be a nice user experience. We should fix that by **delegating the heavy lifting to a Web worker**.

This situation happened to me very recently, and instead of just webworkerifying my library, I generalised prematurely and made a **tool to webworkerify any library**!


### Meet WorkerWrapper

I made a command line tool I call WorkerWrapper. Feed it a library...

![](../../diagrams/workerwrapper.svg)

...and it will **generate two files**. First off an **async version** of the library. This is a very small file that gives you an object containing the same method names as the library, but they now return promises that resolves when a webworker gets the result from the real method.

```javascript
var asyncVersion = {
  aHeavyMethod: function(arg1,arg2){
    // wrapping magic left out here
    return promise;
  },
  anotherHeavyMethod: function(arg1,arg2){
    // wrapping magic left out here
    return promise;
  },
  // etc
};
```

The async version then spawns web workers from the **worker version**, which is really just the library with a web worker facade on top.

### Usage

Say our heavy lib is at `lib/heavylib.js`.

We would then **include WorkerWrapper** as a dependency in `package.json`, and add a **script using the `workerwrap` command** that WorkerWrapper exposes:

```
{
  "dependencies": {
    "workerwrapper": "github:krawaller/workerwrapper"
  },
  "scripts": {
    "wrap": "workerwrap lib/heavylib.js"
  }
}
```

The command just takes one parameter, namely the relative path to the library to be wrapped.

Now `heavylib_worker.js` and `heavylib_async.js` will be created next to `heavylib.js`. I consume `heavylib_async.js` in my app.

However! In order to allow setup with the worker file, `heavylib_async.js` doesn't export the wrapped library directly, but a function!

```
module.exports = function(pathToWorkerFile, numberOfWorkers){

    // ...wrapping magic here...

    return wrappedLib;
}
```

You call this function with a relative path to where you've placed the worker file, as well as the number of parallel webworkers you want spun up.


### See it in action

In the live app below I've wrapped this silly library:

```
var heavyLib = {
  aHeavyMethod: function(arg1,arg2){
    console.log("aHeavyMethod called in lib");
    for(var a=0; a<70000; a++){
      for(var b=0; b<70000; b++){
      }
    }
    return arg1 + arg2;
  }
}
```

On my machine the method takes around 2 seconds. Experiment with hitting the single-worker and multi-worker versions multiple times in quick succession, and you'll see the benefits of parallel workers! 

<iframe src="http://blog.krawaller.se/workerdemo/" style="height:500px;width:100%"></iframe>

You can run the demo in a standalone tab [here](http://blog.krawaller.se/workerdemo/), and the demo source code is [here](https://github.com/krawaller/workerdemo).


### Under the hood of the wrapper

You can peruse the full source code for the tool [here](https://github.com/krawaller/workerwrapper), but in short, the script doing the wrapping works with **two template files**.

Here's the one for the async version:

```
module.exports = function(pathToLib, nbrOfWorkers){

  nbrOfWorkers = nbrOfWorkers || 1;

  var workerListeners = {};
  var freeWorkers = [];
  var busyWorkers = [];
  var nextCallId = 0;

  function freeUpWorker(worker){
    busyWorkers = busyWorkers.filter(function(w){ return w !== worker; });
    freeWorkers.push(worker);
  }

  function requestWorker(){
    var worker = (freeWorkers.length ? freeWorkers : busyWorkers).shift();
    busyWorkers.push(worker);
    return worker;
  }

  function workerMessageHandler(e){
    var resultid = e.data[0];
    var result = e.data[1];
    workerListeners[resultid](result);
    delete workerListeners[resultid];
    freeUpWorker(this);
  }

  function libMethod(method){
    var args = Array.prototype.slice.call(arguments).slice(1);
    return new Promise(function(resolve,reject){
      var callid = ++nextCallId;
      var worker = requestWorker();
      worker.postMessage([method,callid,args]);
      workerListeners[callid] = resolve;
    });
  }

  for(var i=0; i<nbrOfWorkers; i++){
    var worker = new Worker(pathToLib);
    worker.onmessage = workerMessageHandler;
    freeUpWorker(worker);
  }

  return LIB_METHODS.reduce(function(mem,method){
    mem[method] = libMethod.bind(null,method);
    return mem;
  },{});

};
```

Notice the `LIB_METHODS` at the bottom, which the build script replaces with an array of the methods in the library to be wrapped:

```
fs.writeFileSync(
  pathToLibDir + '/'+libName+'_async.js',
  wrapperTemplate.replace('LIB_METHODS', JSON.stringify(Object.keys(lib)))
);
```

The template to the worker file is much simpler:

```
var lib = require('PATH_TO_LIB');

onmessage = function(e){
  var method = e.data[0];
  var callid = e.data[1];
  var args = e.data[2];
  var result = lib[method].apply(lib,args);
  postMessage([callid,result]);
}
```

The script uses webpack to bundle the library into the template.

```
fs.writeFileSync(pathToTempFile, workerTemplate.replace('PATH_TO_LIB', pathToLib));

var compiler = webpack({
  entry: pathToTempFile,
  output: {
    path: pathToLibDir,
    filename: libName + '_worker.js'
  },
  resolve: {
    extensions: [".js"]
  }
});

compiler.run(function(err,stats){
  var message = stats.toString("errors-only") || 'Webworker file created at '+pathToLibDir+'/'+libName + '_worker.js';
  console.log(message);
  fs.unlinkSync(pathToTempFile);
});
```

Refer to the [source code](https://github.com/krawaller/workerwrapper) for the full truth.

### Wrapping up

Pun very much intented.

Is a stand-alone tool to do this really needed? Likely not, but I had fun, and it felt good to be able to remove all the wrapping code from the repo of the library I was working on.

Which is also the library I should immediately go back to work on, if I want to keep my job.
