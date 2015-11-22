---
title: Redux tutorial part 2 - combining reducers
author: David
tags: [redux]
date: 2015-11-21
excerpt: "In the second part of our concise Redux tutorial we look at how to split up our reducer"
type: post
---

The code in this post can be played with in [this JSBin](http://jsbin.com/hewova/1/edit?js,console).

### Where we're at

In the [first post]() we defined a simple Redux store. It had this initial state:

```javascript
var initialState = {
  counter: {value:0, stepsize:1},
  feedback: ["Mini redux demo"],
};
```

...and computed the new state after an action using this reducer:

```javascript
var reducer = function(currentstate,action){
    // sloppily copying `currentstate` since we mustn't mutate it
  var newstate = JSON.parse(JSON.stringify( currentstate ));
  switch(action.type){
    case "INCREASECOUNTER":
      newstate.counter.value += newstate.counter.stepsize;
      newstate.feedback.push("Counter updated");
      return newstate;
    case "INCREASESTEP":
      newstate.counter.stepsize += 1;
      newstate.feedback.push("Step size updated");
      return newstate;
    case "ADDMESSAGE":
      newstate.feedback.push(action.msg);
      return newstate;
    default: return currentstate || initialState;
  }
};
```

### Introducing reducer composition

Handling all actions in one single reducer will quickly become unwieldy. Redux therefore allows you to compose your reducer out of several sub reducers, each one operating on a key in your app state. In our case that means one reducer for `state.counter` and one for `state.feedback`.

Such a reducer will not get the full app state but only the part for which it is responsible, and it is only expected to return an update for that part, and not an entire app state. But every reducer will still be called with every single action that happens. 

Here's a reducer dealing only with `counter`. This reducer will only get and return an object looking like `{value:<number>,stepsize:<number>}`:

```javascript
var counterReducer = function(counterstate,action){
  var newstate = Object.assign({},counterstate);
  switch(action.type){
    case "INCREASESTEP":
      newstate.stepsize = newstate.stepsize + 1;
      return newstate;
    case "INCREASECOUNTER": 
      newstate.current = newstate.value + newstate.stepsize;
      return newstate;
    default: return counterstate || initialState.counter;
  }
};
```

For `feedback` we can use this reducer, which will only operate on the array of messages:

```javascript
feedbackReducer = function(feedbackstate,action){
  var newstate = [].concat(feedbackstate);
  switch(action.type){
    case "ADDMESSAGE":
      return newstate.concat(action.msg);
    case "INCREASESTEP":
      return newstate.concat("Step size updated");
    case "INCREASECOUNTER":
      return newstate.concat("Counter updated");
    default: return feedbackstate || initialState.feedback;
  }
};
```

Together, these two reducers will behave exactly like the big single reducer from the last post.

### Composing a store

We now merge the reducers using `Redux.combineReducers`:

```javascript
var rootReducer = Redux.combineReducers({
  counter: counterReducer,
  feedback: feedbackReducer
});
```

Redux uses the properties of the object we pass in to understand what part of the app state each reducer should operate on.

We can now instantiate the store just like before:

```javascript
var store = Redux.createStore(reducer,initialState);
```

This store is completely identical to the one from last time, as we'll see if we run the same dispatches as before:

```javascript
store.dispatch({type:"INCREASECOUNTER"}); // value will be 0+1 = 1
store.dispatch({type:"INCREASESTEP"});
store.dispatch({type:"INCREASECOUNTER"}); // value will be 1+2 = 3
store.dispatch({type:"ADDMESSAGE",msg:"Works exactly like before!"});

console.log(store.getState());
```
This will, as expected, output: 

```javascript
{
  {
    stepsize: 2,
    value: 3
  },
  feedback: [
    "Mini redux demo",
    "Counter updated",
    "Step size updated",
    "Counter updated", 
    "Works exactly like before!"
  ]
}
```

### Next steps

In the [next post]() we'll look at how to subscribe to the store so we're automatically notified whenever an action occurs.