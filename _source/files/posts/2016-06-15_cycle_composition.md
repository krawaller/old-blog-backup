---
type: post
title: "Exploring composition in CycleJS"
date: 2016-06-15
tags: [cyclejs]
author: David
excerpt: Experimenting with a helper function for component composition in CycleJS
draft: false
---

### The premise

This post plays around with a **helper method for combining components in CycleJS**, which can be somewhat painful when a child and parent each depend on streams from the other. You are assumed to be familiar with [CycleJS](http://cycle.js.org/) basics.

We'll use [most](https://github.com/cujojs/most) for our streams in the code examples, but the concept works for any stream implementation.


### Our demo app

To test the concepts we'll be using the below application. Please kick it around for a bit to see how it works!

<iframe class="nb" src="../../applets/cyclecomp/demos/nameform_02.html" height="160px" width="100%"></iframe>

There are three components in use here; **Nameform**, **Textentry** and **Confirm**. You can see the border between them as dashed lines in the live app above.

Here's a diagram of how they are nested:

![components](../../diagrams/cycle-components.svg)

We'll now walk through them one at a time, starting from the inside and working our way out!

### The Confirm component

The innermost component is the button. It has **three modes**:

*    Showing Submit but disabled
*    Showing Submit
*    Showing Confirm/Cancel

Here's the API:

![confirm api](../../diagrams/cycle-confirm-api.svg)

It **expects the parent to give it a `disabled` stream** as a source to let it know when to enter the disabled mode. In turn the component **returns a `submit` stream** for the parent to consume, containing an event per time the user clicked the confirm button.

Since it is a leaf component the implementation isn't relevant to the subject of this post, but if you're interested you can check out the source code [here](../../applets/cyclecomp/components/confirm.js).


### The Textentry component

This component is responsible for... 
*    keeping track of the contents of a field
*    notifying the parent of submitted values.

Here's the API:

![textentry api](../../diagrams/cycle-textentry-api.svg)

`Textentry` uses the `Confirm` component, but here things get hairy:

*    in order to provide the `disabled` stream to `Confirm` we need to know our state
*    to know our state we need our action stream
*    but to know our action stream we need the `submit` stream from `Confirm`!

This gives us a **circular dependency**, which we must **solve using a proxy**:

```javascript
function Textentry(sources){

  const disabledproxy = mostSubject.holdSubject(1)
  const childsources = {DOM:sources.DOM,disabled$:disabledproxy}
  const confirm = Confirm(childsources)

  const intents = intent(sources, confirm.submit$)
  const state$ = model(intents.action$)
  const vtree$ = view(state$, confirm.DOM)

  const disabled$ = state$.map(i => !i).startWith(true)
  disabled$.subscribe(disabledproxy)

  return {
    DOM: vtree$,
    submit$: intents.submit$.map(a => a.data)
  }
});
```

If you want to see the full source including `intent`, `model` and `view`, check [here](../../applets/cyclecomp/components/textentry_02.js).



### The Nameform component

This is the outermost component, which **holds the currently submitted name**. It has the following API:

![nameform api](../../diagrams/cycle-nameform-api.svg)

The `store` streams in sources and sinks are handled by a `localStorage` driver, persisting the name between sessions.

Here's the source code for this rather skeletal component:

```javascript
import Textentry from './textentry'

function Nameform(sources){
  const child = Textentry(sources)

  const state$ = child.submit$
    .startWith('John')
    .merge(sources.store)

  const vtree$ = most.combine((state,childvtree)=>{
    return div([
      childvtree,
      h4('Hello, '+state)
    ]);
  },state$,child.DOM)

  return {
    DOM: vtree$,
    store: state$
  }
}
```

We didn't need a proxy here since `Textentry` didn't need any input from `Nameform`.


### Introspective

Here's an overview of how our components fit together:

![nameform api](../../diagrams/cycle-composition.svg)

The red arrow shows the circular dependency we had to solve between `Textentry` and `Confirm`.

As you get used to CycleJS, doing the proxy dance becomes second nature and doesn't really feel that complicated. But it is still rather cumbersome code to write! Even when there isn't a circular dependency, like when we use `Texentry` inside `Nameform`, the code does feel somewhat boilerplaty.

This spurred me to try to make a **composition helper method**. After several iterations I arrived at an API where I:

*    expose the child sinks among the parent sources
*    provide (and proxy) needed parent sinks into child sources

![helper pattern](../../diagrams/cycle-helper.svg)

Let's see some details!


### The `withComponent` method

The `withComponent` helper has the following signature:

```javascript
withComponent(main, child, ...dependencies)
```

where...

*    `main` is a component main function
*    `child` is a main function for the child component
*    `dependencies` are names of props in the parent sink that the child wants as sources

The helper then **returns a wrapped `main`** where `sources` will include a `childsinks` prop. You must make sure that the `main` function you are wrapping actually includes the expected streams in the sinks.

Here is `Textentry` reimplemented using `withComponent`:

```javascript

function main(sources) {
  const intents = intent(sources, sources.childsinks.submit$)
  const state$ = model(intents.action$)
  const vtree$ = view(state$, sources.childsinks.DOM)
  return {
    DOM: vtree$,
    submit$: intents.submit$.map(a => a.data),
    disabled$: state$.map(i => !i).startWith(true)
  }
}

const Textentry = withComponent(main,Confirm,"disabled$")
```

Comparing to the previous implementation we see that **all proxy dancing went away**. And we didn't need to touch `intent`, `model` or `view` at all (although if we did we could do some more cleanup).

![code anim](../../img/codeanim.gif)



We can also **reimplement `Nameform`**:

```javascript
function main(sources){

  const state$ = sources.childsinks.child.submit$
    .startWith('John')
    .merge(sources.store)

  const vtree$ = most.combine((state,childvtree)=>{
    return div([
      childvtree,
      h4('Hello, '+state)
    ]);
  },state$,sources.childsinks.DOM)

  return {
    DOM: vtree$,
    store: state$
  }
}

const Nameform = withComponent(main,Confirm)
```

Complexitywise it didn't make much difference since there was no circular dependency, but there is still something neat about abstracting away the entire composition to a helper call.

Here is the **source code for `withComponent`**. The `proxy` comes from [cycle-circular](https://github.com/whitecolor/cycle-circular/) by [https://twitter.com/_whitecolor](Whitecolor).

```javascript
const withComponent = (main, constructor, ...dependencies) => sources => {
  let proxies = dependencies.reduce((proxies,dep)=>({
    ...proxies,
    [dep]: proxy()
  }),{})
  const childsinks = constructor({...sources,...proxies})
  const sinks = main({...sources,[config.childrensinks||'childsinks']:childsinks})
  Object.keys(proxies).forEach(proxy => proxies[proxy].proxy(sinks[proxy]))
  return sinks
}
```

### The `withComponents` helper

Note the plural s at the end! This is for integrating many child components into a parent. Here's the signature:

```javascript
withComponents(main,children)
```

The `children` parameter is an object where each key is a reference to a child, and each value an array containing `[constructor, ...dependencies]` for that child.

It doesn't make too much sense since it only uses one child, but here is `Textentry` using `withComponents` instead:

```javascript

function main(sources) {
  const intents = intent(sources, sources.childrensinks.confirm.submit$)
  const state$ = model(intents.action$)
  const vtree$ = view(state$, sources.childrensinks.confirm.DOM)
  return {
    DOM: vtree$,
    submit$: intents.submit$.map(a => a.data),
    disabled$: state$.map(i => !i).startWith(true)
  }
}

const Textentry = withComponents(main,{
  confirm: [Confirm,"disabled$"]
})
```

Note how the sinks from a child are now available at `sources.childrensinks[childref]`.

Here's the source code for `withComponents`:

```javascript
const withComponents = (main,children) => sources => {
  let proxies = {}
  const childrensinks = Object.keys(children).reduce((childrensinks,child)=>{
    const [constructor,...dependencies] = children[child]
    const myproxies = dependencies.reduce((myproxies,dep)=>({
      ...myproxies,
      [dep]: proxies[dep] || (proxies[dep] = proxy())
    }),{})
    return {
      ...childrensinks,
      [child]: constructor({
        ...sources,
        ...myproxies
      })
    }
  },{})
  const sinks = main({...sources,[config.childrensinks||'childrensinks']:childrensinks})
  Object.keys(proxies).forEach(proxy => proxies[proxy].proxy(sinks[proxy]))
  return sinks
}
```


### Wrapping up

I *think* I have found a good abstraction, but I need to do more battletesting to be sure. If it holds up, and the community doesn't make me aware of something stupid I've overlooked (which is entirely possible), I'll publish it as an npm module.

If you made it this far you are very likely already enamored with CycleJS, but in the improbable case that you're not, definitely give it a go! Whether you end up actually using it in sharp situations or not, simply trying it out will expand the way you think about application structure.



