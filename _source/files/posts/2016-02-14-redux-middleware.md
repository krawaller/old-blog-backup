---
type: post
title: "Exploring Redux middleware"
date: 2016-02-15
tags: [redux]
author: David
excerpt: A series of simple stand-alone experiments to understand Redux Middleware
draft: true
---

### The premise

This blog post is made up by a series of tiny experiments aiming to **better our understanding of Redux middlewares**. You are assumed to be familiar with the basic functionality of Redux.

Throughout we'll use a ridiculously simple counter reducer. It knows only of a single action, `INCREMENT`, and when that is encountered it increases state with the `by` payload (thus the state is not an object but a simple number):

```javascript
var reducer = function(state,action){
    state = state || 0;
    switch(action.type){
        case 'INCREMENT': return state + (action.by || 1);
        default: return state;
    }
}
```

Just to try this out we can create a store with no middlewares...

```javascript
var store = Redux.createStore(reducer);
```

...and, assuming an html document with Redux and a `div` with id `app`, we can hook this up like this:


```javascript
// make render run on every change to store, and run an initial render
store.subscribe(render);
render();

// increase counter anytime page is clicked
document.addEventListener('click', function(e){
    store.dispatch({ type: 'INCREMENT', by: 1 });
});
```

Try it out in the iframe below (or stand-alone [here](../../applets/reduxmiddleware/experiments/nomiddleware.html))

<iframe src="../../applets/reduxmiddleware/experiments/nomiddleware.html" height="100px" width="100%"></iframe>

The following experiments will all be slight variations of this little app. Each will have a standalone link to an html document containing the full code, apart from Redux itself. There will be no other dependencies at all.

I'll also be avoiding ES6 syntax to make it more clear what's going on, as arrow functions and implicit returns tend to muddy the waters a bit.

### The simplest possible middleware

A middleware sits on top `store.dispatch`. Each applied middleware is given a dispatched action object, doing their thing before passing it on to the next middleware in line, until it finally reaches the original `store.dispatch` which will call the reducer.

To get a sense for what the middleware looks like, check out this simplest possible middleware which does nothing but pass the action along:

```javascript
var noop = function(middlewareAPI){
    return function(next){
        return function(action){
            return next(action);
        }
    }
}
```

The `middlewareAPI` object we're being passed up top contains `dispatch` and `getState`, letting us do all kinds of things should we want to. Here we're only doing what we're supposed to, namely passing the `action` on to the `next` in line, which in this case will be the actual `store.dispatch` since there's no other middleware.

Just as a sanity check, let's create a store using this useless middleware...

```javascript
var middlewares = Redux.applyMiddleware(noop);
var store = Redux.createStore(reducer,middlewares);
```

...and check that it is still functioning exactly like before (standalone [here](../../applets/reduxmiddleware/experiments/noop.html)):

<iframe src="../../applets/reduxmiddleware/experiments/noop.html" height="100px" width="100%"></iframe>

Things appear to still be working as they should.


### A breadcrumb trail

In order to decipher what's going on, let's add a logging mechanism to our app! We add a `<div id="log"/>` to the page, and write to it using this `output` function:

```javascript
// put stuff into the log
var output = function(txt){
    var newparagraph = document.createElement("div");
    newparagraph.innerHTML = txt;
    document.getElementById("log").appendChild(newparagraph);
}
```

Now we create a `logger` middleware which uses `output` to spy on what's happening. Actually, let's make a `loggerFactory(name)`, so we can have more than one `logger` with different names and track when the various parts of the middlewares are being called:

```javascript
var loggerFactory = function(name){
    return function(middlewareAPI){
        output(name+": created");
        return function(next){
            output(name+": middle callback called");
            return function(action){
                output(name+": inner callback called with action "+JSON.stringify(action));
                var ret = next(action);
                output(name+": State after calling next: "+middlewareAPI.getState());
                return ret;
            }
        }
    }
}
```

Note how we use `middlewareAPI.getState` to query the updated state after the `next` call.

We create a store with two of these loggers:

```
var log1 = loggerFactory("log1"),
    log2 = loggerFactory("log2"),
    middlewares = Redux.applyMiddleware(log1,log2),
    store = Redux.createStore(reducer,middlewares);
```

It's a bit spammy, but should give a good sense of what's going on (standalone [here](../../applets/reduxmiddleware/experiments/logger.html)):

<iframe src="../../applets/reduxmiddleware/experiments/logger.html" height="400px" width="100%"></iframe>

Matching the spam to the various parts of the middleware, here's what we can deduce:

```javascript
function(middlewareAPI){ // called at start, left to right
    return function(next){ // called at start, right to left
        return function(action){ // called per action
            // code before `next` call runs left to right
            var ret = next(action);
            // code after next call runs right to left
            return ret;
        }
    }
}
```

### Having some fun

Just to internalize our findings so far, let's make some silly middlewares that messes about with the `next` call! First we have a `deaf` middleware which will only hear what you say a third of the time:

```javascript
var deaf = function(middlewareAPI){
    var i = 0;
    return function(next){
        return function(action){
            if (!(i++%3)) {
                next(action);
            }
        }
    }
}
```

Next we have `nervous` who will execute every action twice just to be sure:

```javascript
var nervous = function(middlewareAPI){
    return function(next){
        return function(action){
            next(action);
            next(action);
        }
    }
}
```

Finally there's `impatient` who will set `.by` to 5 instead of the default 1:

```javascript
var impatient = function(middlewareAPI){
    return function(next){
        return function(action){
            // it is good form not to mutate action so we make a copy
            next(Object.assign({},action,{by:5}));
        }
    }
}
```

We whip up a store using all three, in the mentioned order:

```javascript
var middlewares = Redux.applyMiddleware(deaf,nervous,impatient),
    store = Redux.createStore(reducer,middlewares);
```

The net result should be that only every third click registers, but that click will increase the counter by 2*5 (standalone [here](../../applets/reduxmiddleware/experiments/havingsomefun.html)):

<iframe src="../../applets/reduxmiddleware/experiments/havingsomefun.html" height="100px" width="100%"></iframe>


### Snooping at final `next`

Since the action eventually ends up with the regular `store.dispatch`, that implies that `next` inside the final middleware **is** the vanilla `store.dispatch`. Let's see if that holds true! We make a middleware that makes the comparison: 

```javascript
var vanilladispatch = Redux.createStore(reducer).dispatch;
var snoopForVanilla = function(middlewareAPI){
    return function(next){
        return function(action){
            output("next === vanilla dispatch? "+(next.toString() === vanilladispatch.toString()));
            return next(action);
        }
    }
}
``` 

Note how we cannot compare references directly since they obviously won't be the same (as we picked `dispatch` from a non-middlewared throwaway store), so we turn them to strings before we check equality.

Now, let's put it to the test (standalone [here](../../applets/reduxmiddleware/experiments/snoopvanillanaive.html))!

<iframe src="../../applets/reduxmiddleware/experiments/snoopvanillanaive.html" height="120px" width="100%"></iframe>

It seems our assumption was correct!

### Snooping at following middleware

This raises a question - what is `next` from the perspective of a middleware who **isn't** last in the chain? Let's revive or old friend the `noop` middleware:

```
var noop = function(middlewareAPI){
    return function(next){
        return function(action){
            return next(action);
        }
    }
}
```

If we put it at the end and a snooper before it, what would that snooper actually see? An easy way to test that would be to simply log out `next`, so let's create a snooper doing that:

```javascript
var snooper = function(middlewareAPI){
    return function(next){
        return function(action){
            output("snooping at `next`: "+next);
            return next(action);
        }
    }
}
```

We put both in a store, taking care to put `snooper` before `noop`:

```javascript
var middlewares = Redux.applyMiddleware(snooper,noop),
    store = Redux.createStore(reducer,middlewares);
```

...and voilà (standalone [here](../../applets/reduxmiddleware/experiments/snoopatfriend.html)):

<iframe src="../../applets/reduxmiddleware/experiments/snoopatfriend.html" height="150px" width="100%"></iframe>

It seems inner part of the middleware becomes the `next`. Which makes sense as the inner part is what takes the action as an argument, and it is the action we're feeding to `next`!

Curious what would happen if we put snooper last in the chain? You'd get a facefull of [Redux source code](https://github.com/reactjs/redux/blob/e7295c33776be6199b826817934dadad5d0f9bb1/src/createStore.js#L148-L180) is what. Hack the standalone version and try it!

### Dispatch return value

Ok, so the middleware passes the action along by calling the `next` it is given. But why does it return the result of that `next` call? Since the action chain is done through the `next` call, what purpose does that return value serve? 

A clue might be that the [documentation says](http://redux.js.org/docs/api/Store.html#dispatch) that `store.dispatch(action)` returns the action you passed in. The docs also notes, however, that middlewares can mess with the return value.

First off, let's just hack the click handler to also output the return from `dispatch` to the log:

```
document.addEventListener('click', function(e){
    var ret = store.dispatch({ type: 'INCREMENT', by: 1 });
    output("return: "+JSON.stringify(ret));
});
```

We run this with a no-middleware store (standalone [here](../../applets/reduxmiddleware/experiments/outputreturn.html)):

<iframe src="../../applets/reduxmiddleware/experiments/outputreturn.html" height="150px" width="100%"></iframe>

Unsurprisingly it seems the docs told the truth - we simply get the action object back.

And, also unsurprisingly, if we insert a stupid middleware which returns some bogus crap:

```javascript
var sillynoop = function(middlewareAPI){
    return function(next){
        return function(action){
            next(action);
            return "WOO!";
        }
    }
}
```

...then bogus crap is what comes out at the end of the chain, even if we put a regular noop before and after:

```
// setting up the store
var middlewares = Redux.applyMiddleware(noop,sillynoop,noop),
    store = Redux.createStore(reducer,middlewares);
```


See for yourself (standalone [here](../../applets/reduxmiddleware/experiments/outputreturnsilly.html)):

<iframe src="../../applets/reduxmiddleware/experiments/outputreturnsilly.html" height="150px" width="100%"></iframe>
