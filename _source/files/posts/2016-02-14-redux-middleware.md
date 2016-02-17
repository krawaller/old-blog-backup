---
type: post
title: "Exploring Redux middleware"
date: 2016-02-17
tags: [redux]
author: David
excerpt: Nine simple stand-alone experiments to understand Redux Middlewares
---

### 0. The premise

This blog post is made up by 9 tiny self-contained experiments aiming to **better our understanding of Redux middlewares**. You are assumed to be familiar with the basic functionality of Redux. If you're not then [go remedy that immediately](http://redux.js.org/), and let it be known that I'm severely jealous of you for still having that experience in front of you!

Throughout we'll use a ridiculously simple **counter reducer**. It knows only of a single action, `INCREMENT`, and when that is encountered it increases state with the `by` payload (thus the state is not an object but a simple number):

```javascript
var reducer = function(state,action){
    switch(action.type){
        case 'INCREMENT': return (state || 0) + (action.by || 1);
        default: return state;
    }
}
```

Just to try this out we can create a store with no middlewares:

```javascript
var store = Redux.createStore(reducer);
```

Assuming a `div` with id `app` we whip up a render function:

```javascript
var render = function(){
    var newhtml = "<h2>Clicked "+store.getState()+" times.</h2>";
    document.getElementById("app").innerHTML = newhtml;
}
```

We hook `render` up to run when the store updates, and we also run an initial render:

```javascript
store.subscribe(render);
render();
```

Finally we set a click event which dispatches an action to the store:

```
document.addEventListener('click', function(e){
    store.dispatch({ type: 'INCREMENT', by: 1 });
});
```

Try it out in the iframe below (or standalone [here](../../applets/reduxmiddleware/experiments/nomiddleware.html))

<iframe src="../../applets/reduxmiddleware/experiments/nomiddleware.html" height="100px" width="100%"></iframe>

The following experiments will all be slight variations of this little app. Each will have a standalone link to an html document containing the full code, apart from Redux itself. There will be no other dependencies at all.

I'll also be avoiding ES6 syntax to make it more clear what's going on, as arrow functions and implicit returns tend to muddy the waters a bit.

### 1. The simplest possible middleware

A middleware sits on top `store.dispatch`. Each middleware is given a dispatched action object, doing their thing before passing it on to the next middleware in line, until it finally reaches the original `store.dispatch` which will call the reducer.

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


### 2. A breadcrumb trail

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

### 3. Having some fun

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


### 4. Snooping at final `next`

Since the action eventually ends up with the regular `store.dispatch`, that implies that `next` inside the final middleware **is** the vanilla `store.dispatch`. Let's see if that holds true! We make a middleware that makes the comparison: 

```javascript
var vanilladispatch = Redux.createStore(reducer).dispatch;
var snoopForVanilla = function(middlewareAPI){
    return function(next){
        return function(action){
            var comparison = (next.toString() === vanilladispatch.toString());
            output("next === vanilla dispatch? "+comparison);
            return next(action);
        }
    }
}
``` 

Note how we cannot compare references directly since they obviously won't be the same (as we picked `dispatch` from a non-middlewared throwaway store), so we turn them to strings before we check equality.

Now, let's put it to the test (standalone [here](../../applets/reduxmiddleware/experiments/snoopvanillanaive.html))!

<iframe src="../../applets/reduxmiddleware/experiments/snoopvanillanaive.html" height="120px" width="100%"></iframe>

It seems our assumption was correct!

### 5. Snooping at following middleware

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

Evidently the inner part of the middleware becomes the `next`. Which makes sense as the inner part is what takes the action as an argument, and it is the action we're feeding to `next`!

Curious what would happen if we put snooper last in the chain? You'd get a facefull of [Redux source code](https://github.com/reactjs/redux/blob/e7295c33776be6199b826817934dadad5d0f9bb1/src/createStore.js#L148-L180) is what, spelling out the entrails of the `dispatch` method. Hack the standalone version and try it!

### 6. Dispatch return value

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

```javascript
// setting up the store
var middlewares = Redux.applyMiddleware(noop,sillynoop,noop),
    store = Redux.createStore(reducer,middlewares);
```


See for yourself (standalone [here](../../applets/reduxmiddleware/experiments/outputreturnsilly.html)):

<iframe src="../../applets/reduxmiddleware/experiments/outputreturnsilly.html" height="150px" width="100%"></iframe>


### 7. Localstorage getsate

We've utlized `middlewareAPI.getState` in the `logger` middleware. Here's another quick example of a more practical use; a `persist` middleware which uses `middlewareAPI.getState` to store the whole store state to `localStorage` for every change.

```javascript
var persist = function(middlewareAPI){
    return function(next){
        return function function(action){
            var ret = next(action),
                str = JSON.stringify({data:middlewareAPI.getState()});
            window.localStorage.setItem("SAVESTATE",str);
            return ret;
        }
    }
}
```

We put this in a store, and seed it with data from localStorage as initial state:

```javascript
var savedstr = window.localStorage.getItem("SAVESTATE")||"{}",
    initialstate = JSON.parse(savedstr).data,
    middlewares = Redux.applyMiddleware(persist),
    store = Redux.createStore(reducer,initialstate,middlewares);
```

Give it a few clicks below, then reload the page to watch the counter remember its position (and not at *all* to give me extra page views).

<iframe src="../../applets/reduxmiddleware/experiments/persistence.html" height="100px" width="100%"></iframe>

Standalone [here](../../applets/reduxmiddleware/experiments/persistence.html).


### 8. The injected `dispatch`

We've utlized `middlewareAPI.getState` in the `logger` middleware. The use of it is pretty obvious. But what about the other method on the `middlewareAPI`, namely `dispatch`? At first glance it doesn't really make sense. To continue the action chain we merely call `next`, as we've seen several times over. Why would we want to start a new chain?

Just to test that this is really what will happen (because who knows, maybe `middlewareAPI.dispatch` is a wormhole to the final `dispatch` in the chain?), let's make a silly `echo` middleware:

```javascript
var echo = function(middlewareAPI){
    return function(next){
        return function(action){
            if (action.type !== 'ECHO'){
                setTimeout(function(){
                    middlewareAPI.dispatch({
                        type: 'ECHO',
                        volume: 3
                    });
                },500);
                return next(action);
            } else if (action.volume){
                setTimeout(function(){
                    middlewareAPI.dispatch({
                        type: 'ECHO',
                        volume: action.volume - 1
                    });
                },500);
            }
        }
    }
}
```

If the action it is given isn't an `ECHO`, it will start a new chain with an `ECHO` action with `volume` 3. If it was an echo it will repeat it with one less volume unit, until the echo fades. Note that we're swallowing all `ECHO` actions, not passing them along to `next`.

In order to see this in action let's also revive our `loggerFactory`, although slightly less spammy this time:

```javascript
var loggerFactory = function(name){
    return function(middlewareAPI){
        return function(next){
            return function(action){
                var ret = next(action),
                    newstate = middlewareAPI.getState();
                output(name+": called with "+JSON.stringify(action)+", state now "+newstate);
                return ret;
            }
        }
    }
}
```

We set up a store with the `echo` middleware sandwiched between two `logger`s:

```javascript
var log1 = loggerFactory("log1"),
    log2 = loggerFactory("log2"),
    middlewares = Redux.applyMiddleware(log1,echo,log2),
    store = Redux.createStore(reducer,middlewares);
```

Try it out (standalone [here](../../applets/reduxmiddleware/experiments/testingdispatch.html))!

<iframe src="../../applets/reduxmiddleware/experiments/testingdispatch.html" height="400px" width="100%"></iframe>


### 9. Redux-thunk

So the previous experiment merely proved that `middlewareAPI.dispatch` is a way to start a new chain, something which on the surface seems pretty useless. Here's a very good example for when it *is* useful: [Redux-thunk](https://github.com/gaearon/redux-thunk). It is a middleware where the source code looks like this:

```javascript
var thunk = function(middlewareAPI){
    return function(next){
        return function(action){
            if (typeof action === 'function'){
                return action(middlewareAPI.dispatch,middlewareAPI.getState);
            } else {
                return next(action);
            }
        }
    }
}
```

So what's going on here? It checks if `action` is a function (what?!), and if so it invokes that function with `dispatch` and `getState` from the `middlewareAPI`. If `action` is not a function, it simply passes it on to `next` as per usual.

The point of this is to enable developers to have action creators that are async. Let's take a look again at how the counter click handler is set in our experiments:

```
document.addEventListener('click', function(e){
    store.dispatch({ type: 'INCREMENT', by: 1 });
});
```

In normal Redux usage, like for example in a React app using the [React-Redux](https://github.com/reactjs/react-redux) bridge, you won't pass around a reference to the store itself. Instead you'll pass around action creators, and have some central system feed whatever they return to `store.dispatch`. Translating our code above, that might look something like this:

```
// these action creators are what "views" in your app will use
var actionCreators = {
    increase: function(inc){
         // we return an action that'll go to the `dispatch` chain
        return {type: 'INCREMENT', by: inc};
    }
}

// The views will use the actionCreators through a bridge to the dispatch method:
var bridge = function(dispatch){
    return {
        increase: function(inc){ dispatch(actions.increase(inc)); }
    }
};
```

The point of the bridge is that we don't pass the store or `dispatch` around, we just provide it at hookup time by passing it to the bridge function.

This model as seen so far doesn't really allow for defining async actions in a smooth way. Sure, we could do some async stuff in our `bridge` function above, but ideally we'd like to have that logic in the `actionCreators` definition. This is what `Redux-thunk` allows us to do. Remember how it checked if an action was a function, and if so invoked that? That allows us to do stuff like this (this is an extract from our [Redux Firebase example](a-react-redux-firebase-app-with-authentication/)): 


```javascript
var actionCreators = {
    startQuoteEdit: function(qid){
        return {type:C.START_QUOTE_EDIT,qid};
    },
    cancelQuoteEdit: function(qid){
        return {type:C.FINISH_QUOTE_EDIT,qid};
    },
    deleteQuote: function(qid){
        return function(dispatch,getState){
            dispatch({type:C.SUBMIT_QUOTE_EDIT,qid});
            quotesRef.child(qid).remove(function(error){
                dispatch({type:C.FINISH_QUOTE_EDIT,qid});
                if (error){
                    dispatch({type:C.DISPLAY_ERROR,error:"Deletion failed! "+error});
                } else {
                    dispatch({type:C.DISPLAY_MESSAGE,message:"Quote successfully deleted!"});
                }
            });
        };
    },
    // ...
}
```

As you can see, `startQuoteEdit` and `cancelQuoteEdit` are normal sync actions using regular plain action objects, while `deleteQuote` contains lots of logic doing both sync and async stuff. 

When a `deleteQuote` action (which is a function) enters the `dispatch` chain, the `thunk` middleware will catch it, run it, and the result with be further calls to `dispatch` with (most often) regular action objects.

As you can imagine it is important to put `thunk` at the beginning of the middleware chain, since no other middleware will be expecting actions which are functions!

To further wrap our brains around this, let's make a contrived experiment of our own using `thunk`. Let's pretend-play that we're a real app and define our action like this:

```
var actionCreators = {
    increase: function(inc){
        return function(dispatch,getState){
            dispatch({type: 'INCREMENTINCOMING'});
            setTimeout(function(){
                dispatch({type: 'INCREMENT', by:inc});
            },1000);
        }
    }
}
```

We'll change the click hookup to look like this (although a "real" app would use a bridge of some kind and not a direct store reference):

```
document.addEventListener('click', function(e){
    store.dispatch( actionCreators.increase(1) );
});
```

To see this in action we also make use of a simple `logger`:

```
var logger = function(middlewareAPI){
    return function(next){
        return function(action){
            var ret = next(action),
                newstate = middlewareAPI.getState();
            output("action "+JSON.stringify(action)+", state now "+newstate);
            return ret;
        }
    }
}
```

We take care to put that *after* `thunk` in the middleware chain:

```
var middlewares = Redux.applyMiddleware(thunk,logger),
    store = Redux.createStore(reducer,middlewares);
```

Check it out (standalone [here](../../applets/reduxmiddleware/experiments/thunk.html))!

<iframe src="../../applets/reduxmiddleware/experiments/thunk.html" height="150px" width="100%"></iframe>

This use was rather contrived, but again, remember - the point of thunks as actions is merely to allow us to write asynchronous action creators without being dependent on a reference to the store.

### Wrapping up

I hope this little series of laboratory concoction was useful. Mind you, a good way to understand Redux is to read the source - it is tiny! In fact the full version with warnings and comments just has an ever so slightly larger character count than this blog post...
 