---
title: Redux tutorial part 3 - subscribing to store updates
author: David
tags: [redux]
date: 2015-11-21
excerpt: "In the third part of our concise Redux tutorial we look at how to be notified on store data updates"
type: post
---

The code in this post can be played with in [this JSBin](http://jsbin.com/sumihu/3/edit?js,console).

### Where we're at

In the [last post]() we improved on the store definition from the [first post]() by defining separate reducers for each app state property. This post will continue using the exact same store, which was created like this:

```javascript
var initialState = {
  counter: {value:0, stepsize:1},
  feedback: ["Mini redux demo"],
};

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

var rootReducer = Redux.combineReducers({
  counter: counterReducer,
  feedback: feedbackReducer
});

var store = Redux.createStore(reducer,initialState);
```

### Subscribing to store change

Up until now we've tested the stores by dispatching actions and then manually checking `getState`. But wouldn't it be nice if we could be automatically notified when the store has new data? That's what `store.subscribe` lets us do!

Here's a listener function that we want to run whenever the store updates:

```javascript
var listener = function(){
  console.log("New data!",store.getState());
};
```

To have it executed after an action has happened, simply pass it to `store.subscribe`:

```javascript
store.subscribe(listener);
```

Now whenever we dispatch an action, our listener function (and any other subscribing function) is called. So doing this:

```javascript
store.dispatch({type:"INCREASESTEP"});
store.dispatch({type:"INCREASECOUNTER"});
```

...will log this...

```javascript
"New data!"
{
  {
    stepsize: 2,
    value: 0
  },
  feedback: [
    "Mini redux demo",
    "Step size updated"
  ]
}
```

...and this...

```javascript
"New data!"
{
  {
    stepsize: 2,
    value: 2
  },
  feedback: [
    "Mini redux demo",
    "Step size updated",
    "Counter updated"
  ]
}
```

...in quick succession. 

Note how we're not getting the new data as an argument to the listener, but have to access it through `store.getState`.