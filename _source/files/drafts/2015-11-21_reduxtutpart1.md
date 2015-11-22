---
title: Redux tutorial part 1 - the basics
author: David
tags: [redux]
date: 2015-11-21
excerpt: "The first post in a concise guide to Redux. Here we get to now the basics"
type: post
---

The code in this post can be played with in [this JSBin](http://jsbin.com/raneye/1/edit?js,console).

### Starting off

This is the first post in a short series that aim to explain [Redux](redux.js.org) in a short, concice way. My students have rightfully complained that [my previous example app](http://blog.krawaller.se/posts/a-react-redux-example-app/) didn't really make it easy to decipher Redux because of all the noise, a fault it has in common with many other learning resources. The main Redux documentation is excellent but also rather verbose, so here in this series I'll strive to get straight to the punch and kill my darlings.

### Designing the shape of the state

When using Redux the entire state of our application is stored as an object in one single store. Our first step is therefore to design the shape of that object. Here's what it will look like for this demonstration:

```javascript
var initialState = {
  counter: {value:0, stepsize:1},
  feedback: ["Mini redux demo"],
};
```

As you see the state is made up by two parts:

1.   `counter` which contains a current counter value and the size of the steps to be taken when we increase the counter.
2.   `feedback` which is an array of messages to the user.

### Figuring out the actions

The next step is to define the various actions that can happen that will change the state of this app. An action is merely an object describing what happened. It has a `type`, and sometimes other props as well.

Here are the action types I'm having in mind for this demo:

*    `INCREASECOUNTER`: add current stepsize to counter value
*    `INCREASESTEP`: add 1 to current stepsize
*    `ADDMESSAGE`: add a feedback message to the user

Of these three actions only the last would need an additional prop, let's call it `msg`, containing the message to be added.

### Computing new state with a reducer

The actual store is defined by a Reducer function. This function is called every time something happens in the app in order to calculate the new state. It receives the current state and the action object describing what happened, and returns the updated state. A reducer is traditionally made up by a switch statement with a case for each action.

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

It *must* include a `default` case which just returns the current state. You might think you've covered all possible action types, but Redux will start off by dispatching a special init action.

### Instantiating a store

A Redux store is created by passing the `reducer` and `initialState` to `Redux.createStore`:

```javasript
var store = Redux.createStore(reducer,initialState);
```

### Updating the store and querying the state

Now we can update the store data by passing an action object to `store.dispatch`:

```javascript
store.dispatch({type:"INCREASECOUNTER"}); // value will be 0+1 = 1
store.dispatch({type:"INCREASESTEP"});
store.dispatch({type:"INCREASECOUNTER"}); // value will be 1+2 = 3
store.dispatch({type:"ADDMESSAGE",msg:"Cool, innit?"});
```

To check the current store state, use `store.getState`:

```javascript
console.log(store.getState());
```

Given the dispatched actions this would log out the following:

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
    "Cool, innit?"
  ]
}
```

You can imagine that when you call `store.dispatch(action)`, the store will do `this.value = this.reducer(this.value,action)`. Then when you call `getState()`, the store simply returns `this.value`. The actual code is somewhat more convoluted, but not by much at all.  

### Next step

In the [next post]() we'll take a look at reducer composition, a concept which saves us from ending up with one monolith reducer.