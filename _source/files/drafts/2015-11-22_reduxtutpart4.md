---
title: Redux tutorial part 4 - action creators
author: David
tags: [redux]
date: 2015-11-22
excerpt: "In the fourth part of our concise Redux tutorial we look at action creators"
type: post
---

The code in this post can be played with in [this JSBin](http://jsbin.com/gutuno/edit?js,console).

### Introducing action creators

Up until now we've updated the store data by feeding action objects to `store.dispatch`:

```javascript
store.dispatch({type:"ADDMESSAGE",msg:"Hello there!"});
store.dispatch({type:"INCREASECOUNTER"});
```

Having to scatter such code around your app would feel rather crude. The action `type`s are really a store implementation detail that should be kept from the rest of the app.

In order to remedy this we let the rest of the app use something called **action creators**, which are simply functions that return the corresponding action object. For our demo app they could look like this:

```javascript
var actions = {
  increaseStep: function(){
    return {type:"INCREASESTEP"};
  },
  increaseCounter: function(){
    return {type:"INCREASECOUNTER"};
  },
  addMessage: function(txt){
    return {type:"ADDMESSAGE",msg:txt};
  }
};
```