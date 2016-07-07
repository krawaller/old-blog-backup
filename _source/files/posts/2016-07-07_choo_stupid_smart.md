---
type: post
title: "Stupidly smart components in Choo"
date: 2016-07-07
tags: [choo,react,redux]
author: David
excerpt: Philosophizing about why smart components in Choo are stupid and why that is a good thing
---

### The premise

In the aftermath to the [component composition comparison](../composition-in-cyclejs-choo-react-and-angular2/) we did between [React](https://facebook.github.io/react/), [Angular2](https://angular.io), [Choo](https://github.com/yoshuawuyts/choo) and [Cycle](http://cycle.js.org/), I realised something that made my already high esteem for Choo go higher still - *smart components in Choo are stupid*! This post meanders its way to that point.

It started when a [discussion on an unrelated Choo issue](https://github.com/yoshuawuyts/choo/issues/131) led to the age-old argument between having smart isolated components versus all state in a central store. Let's begin by reviewing the stances!

### The merits of smart components

 Take a traditional React app (or any app, really). Most components are (hopefully) dumb, meaning they have no state of their own - they are just a pure render function acting on props fed by the parent. Easy to use, easy to understand and reason about.

 But some components might have state that only they care about, and then it is convenient to let them manage that state. In the framework comparison comparison we created the app running in the iframe below:

<iframe src="../../applets/comparison/index.html" height="100px" width="100%"></iframe>

The button is a separate component, meant to be self-contained and portable. It therefore needs to track the current state (waiting or confirming), and handle the events that toggle this. If you want the button in your app, all you need to do is replace your static button with the component and listen to the confirm callbacks. Convenient!

### The merits of a centralized store

However, if we allow these scattered state machines all around our app, debugging becomes harder and cool stuff like time travel and hot reloading impossible. Solving this is what made [Redux](http://redux.js.org/) popular - by having a *single central place to store your data*, you also get *time travel and ridiculously easy debugging for free*.

What you lose, however, is an easy way to encapsulate component concerns. If I was to let Redux manage the state of the button component, the relevant code would be scattered across a view file, a few cases in a reducer somewhere and some action creators somewhere else. *Portability is lost*.

Some therefore argue that only *app state belongs in Redux*, while *trivial UI state should be kept in components*. I tend to disagree and side with the centralisers, arguing that the advantages of an omnipotent centralised data store is worth the lost portability.

For example in my [React-Redux-Firebase example](../a-react-redux-firebase-app-with-authentication/), even the are-we-editing-or-not-flag for every single row is kept in Redux, even though that is really just a UI detail.


### Choo introduction intermission

It turns out, however, that *in Choo you don't have to choose*! Before we get into why, please humor me through this super-quick introduction to Choo.

Continuing the comparison to Redux, the equivalent to Redux' reducers and action creators in Choo is a *centralised model definition*. Here's a silly example:

```
app.model({
  state: {
    msg: 'hello'
  },
  reducers: {
    saysth: (data, state)=> ({ msg: data.msg }),
  },
  effects: {
    echo: (data, state, send)=> {
      send('saysth',data)
      setTimeout(()=> { send('saysth',data) },1000)
      setTimeout(()=> { send('saysth',data) },2000)
    }
  },
  subscriptions: [otherComplexStuff]
})
```

The `state` prop holds the initial state, `reducers` are the same as in Redux, and `effects` handle side effects like Redux' thunked action creators.

For a full-featured example, check out the [model definition](https://github.com/shuhei/todomvc-choo/blob/master/model.js) of the [Choo TodoMVC implementation](http://shuheikagawa.com/todomvc-choo/).

However, Choo lets you *split the model definition into namespaced pieces*. Here's the code for the Choo version of my confirmation button component:

```
const Confirm = (app,confirmevent)=> {
  app.model({
    namespace: 'confButt',
    state: { button: 'waiting' },
    reducers: {
      maybeSubmit: (action, state) => ({button: 'confirm'}),
      cancelSubmit: (action, state) => ({button: 'waiting'})
    },
    effects: {
      confirmSubmit: (action, state, send)=> {
        send('confButt:cancelSubmit')
        send(confirmevent)
      }
    }
  })
  return (state, disabled, send) => state.confButt.button !== 'confirm' 
    ? choo.view`
      <button onclick=${e => send('confButt:maybeSubmit')} disabled=${disabled}>Submit</button>
    `
    : choo.view`
      <span>
        <button onclick=${e => send('confButt:cancelSubmit')}>Cancel</button>
        <button onclick=${e => send('confButt:confirmSubmit')}>Confirm</button>
      </span>
    `
}
``` 

Side note: there are [discussions on better patterns](https://github.com/yoshuawuyts/choo/issues/131#issuecomment-230811754) than passing the `app` instance around, such as attaching the model as a prop to the renderer instead. More to follow!

But in essence, no matter the exact method of encapsulation, *a self-contained component is made up by two things* in Choo:

*    a namespaced model definition
*    a pure render function

### The merits of Choo

Now for the kicker; even though I split my model definition into parts, the *data is still stored centrally*. In an app using the confirm button, if I logged out the app state, I'd still see the confirm button state in there:

```
{
  foo: 'bar',
  confButt: {
    button: 'waiting'
  },
  otherStuff: { ... },
  ...
}
```

Which means I can enjoy time travel, hot reloading and powerful debugging without paying the price of lost component portability!

What I do lose is the advantage of having the full definition in one single place, which isn't trivial. Still, I feel that it is worth more to be able to piggyback on the community by dropping in 3rd party complex components. My confirm button is of course a contrived example, but imagine an autocomplete field or some other non-trivial piece of functionality.


### Wrapping up

Being able to do this is something I haven't seen in any other framework, which has enamored me with Choo even more. It truly is a cool piece of software, so try it out if you haven't already!

And yes I realise I've ended other posts saying the exact same thing about CycleJS. Heck, go try both! :)


