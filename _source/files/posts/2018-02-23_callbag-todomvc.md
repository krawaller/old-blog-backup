---
type: post
title: "Dissecting a Callbag TodoMVC implementation"
date: 2018-02-23
tags: [callbags,case study,snabbdom,redux,todomvc]
author: David
excerpt: Peeking under the hood of a TodoMVC implementation using only Callbags and Snabbdom
---

### The premise

In the previous post we [familiarised ourselves with Callbags](../callbags-introduction/). Now we'll look at a more practical example by digging through the codebase of a callbags-based TodoMVC app!


### Chopper view of the app

You can find the repo for the app at [https://github.com/krawaller/callbag-todomvc](https://github.com/krawaller/callbag-todomvc), and the live app at [callbag-todomvc.netlify.com](callbag-todomvc.netlify.com) (although that isn't very sexy as it works exactly like all TodoMVC apps ever (which of course is the point)).

The code is split into 4 parts; `makeActions`, `makeState`, `makeView` and `makeSideEffects`. They are chained together like so:

![](../../diagrams/callbag-mvi.svg)

If you've ever played with the [CycleJS framework](https://cycle.js.org/) you might recognise this as inspired by the [Model View Intent pattern](https://cycle.js.org/model-view-intent.html).

The chain from the diagram is also visible in the bootstrapping code in the [`index.js`](https://github.com/krawaller/callbag-todomvc/blob/master/src/index.js) file of the app:

```javascript
const actions = makeActions(window, window.document.getElementById('app'));
const state = makeStateStream(actions);
const view = makeViewStream(state);

makeSideEffects(window, actions, view);
```

We'll now walk through the four main parts in order, making some random callbag-related observations along the way!

### Actions

The first part of the flow is to hook up event streams and bend those into action streams, which we do in the [actions.js](https://github.com/krawaller/callbag-todomvc/blob/master/src/actions.js) file.

Staltz had already made a [callbag-from-event](https://github.com/staltz/callbag-from-event) to create DOM event stream sources, but that didn't really work for me - I'm creating the DOM via a Virtual DOM view stream, so there's nothing to initially listen to!

To mitigate that I made [callbag-from-delegated-event](https://github.com/krawaller/callbag-from-delegated-event), which listens to a root node but filters the events by selector. Much like jQuery's `.on(evt, selector, handler)` syntax.

As an example of what it looks like, here's the definition of the `deleteActions` stream:

```javascript
const deleteActions = pipe(
  fromDelegatedEvent(root, '.destroy', 'click'),
  map(e => ({type: 'DELETETODO', idx: nOfLi(e.target)})),
  share
);
```

The `nOfLi` method is a helper used in many action streams. It uses the target node reference to calculate the index of todo that was clicked:

```javascript
const nOfLi = node => {
  const li = node.closest('li');
  return Array.from(li.parentElement.children).indexOf(li);
};
```

Also notice the use of Staltz' [`share`](https://github.com/staltz/callbag-share) operator at the end of `deleteActions`- in normal stream vocabulary, that turns the stream from *cold* to *hot*. Refer to [Ben Lesh' blog post](https://medium.com/@benlesh/hot-vs-cold-observables-f8094ed53339) for more on that subject, but in essence: If the stream is `cold` (which is the default), every listener will get its own instance of the stream. Here that would mean different results for `noOfLi` if you listen after the state maker. We must therefore use `share` to make it hot!

The `makeActions` function will define a whole bunch of action streams in a similar manner. There are nothing else of interest to note, they all look pretty much like `deleteActions` (but a twist to that story is coming later).

After all action streams are created, `makeActions` returns an object containing all of the action strams. It also throws in an `allActions` stream for good measure, using Staltz' [callbag-merge](https://github.com/staltz/callbag-merge):

```javascript
const allActions = merge(initActions, newTodoActions, ... );
```

### State

The `allActions` stream is a good clue to what will come in [state.js](https://github.com/krawaller/callbag-todomvc/blob/master/src/state.js); we'll calculate the state by using a [Redux](https://redux.js.org/)-like pattern!

![](../../diagrams/callbag-redux.svg)

This means that our app state is one single object. Using TypeScript definitions it looks like this:

```javascript
type TodoAppState = {
  todos: Todo[]
  newName: string // content of new todo input
  editText: string // content of edit todo input
  editing: number | null // index of todo that we're currently editing
  filter: 'all' | 'completed' | 'active'
}
```

Here's a single `Todo`:

```javascript
type Todo = {
  text: string
  done: boolean
}
```

There's a quote that has been made for every single stream library out there:

> Redux is 1 line of [insert stream library name here]

This is true for callbags too, thanks to Staltz' [callbag-scan](https://github.com/staltz/callbag-scan) operator. Here's how the `makeStateStream` function is implemented:

```javascript
function makeStateStream(actions){
  return pipe(
    actions.allActions,
    scan(augmenter(reducer), initialState),
  );
}
``` 

Here `reducer` looks *exactly* like a normal Redux reducer, made up by a big switch statement over `action.type`.

```javascript
function reducer(currentState, action){
  switch (action.type) {
    case 'EDITTODO': return {...currentState, editing: action.idx };
    // ....
  }
}
```

The `augmenter` is how I chose to handle computed properties - it adds in the `remaining` count to the state returned by the reducer:

```javascript
const augmenter = reducer => (state, action) => {
  let newState = reducer(state, action);
  return {
    ...newState,
    remaining: newState.todos.filter(t => !t.done).length
  };
};
```

The end result is that `makeStateStream` returns a stream emitting state objects everytime there is a new action.

### View

The code in [view.js](https://github.com/krawaller/callbag-todomvc/blob/master/src/view.js) is rather boring - we simply `map` the passed-in state stream to a JSX stream!

```javascript
function makeViewStream(state){
  return pipe(
    state,
    map(s => (
      <div>
        { /* full app UI made here */ }
      </div>
    ))
  );
}
```

I'm using [Snabbdom-pragma](https://github.com/Swizz/snabbdom-pragma) to handle the JSX, which means that all JSX elements are transformed into calls to `Snabbdom.createElement`.

### Side effects

Finally, [sideeffects.js](https://github.com/krawaller/callbag-todomvc/blob/master/src/sideeffects.js)!

The primary duty here is to use the view stream to make stuff happen on the screen. For this I'm using the Snabbdom `patch` command which has the following syntax:

```javascript
patch(rootNode, JSX); // first call
patch(previousJSX, newJSX); // subsequent calls
```

In other words I'll need access to the new *and previous* JSX at the same time! For this I built the [callbag-with-previous](https://github.com/krawaller/callbag-with-previous) operator:

```javascript
pipe(
  withPrevious(view),
  forEach(([cur,prev,isFirst]) => {
    patch(isFirst ? window.document.getElementById('renderoutput') : prev, cur)
  })
);
```

At this point fox-eyed readers might wonder why we passed in the `actions` too to `makeSideEffects`:

```
makeSideEffects(window, actions, view);
```

That's because we also want to `.focus()` the various inputs on certain actions. For example, when we edit a todo we want it to gain focus:

```javascript
pipe(
  actions.editActions,
  forEach(() => window.document.querySelector(".editing .edit").focus())
);
```

And, this pretty much concludes our tour through the code base!

### Losing control

I was actually a bit disappointed with how easy it was to define the needed streams. In particular I had hoped for the action streams to provide more of a challenge. Therefore I made a separate version of the app where the contents of the input fields are not part of the app state. Phrased in React lingo; an *[uncontrolled](https://reactjs.org/docs/uncontrolled-components.html)* approach instead of *[controlled](https://reactjs.org/docs/forms.html#controlled-components)* one!

As suspected/hoped, not having the inputs in the state meant having to put more information into the actions, which in turn meant more interesting stream crossing problems. You'll find that version in the [`uncontrolled` branch on Github](https://github.com/krawaller/callbag-todomvc/tree/uncontrolled), but I'll walk through a couple of highlights here!

First example - in the `newTodoActions` we must now include the current content of the new todo input field. We can do that by transforming the stream of `enter` key presses into the latest output from the input typing stream, using Staltz' [callbag-sample](https://github.com/staltz/callbag-sample). However since we want to map to the *latest* value of that stream I had to make a new operator, [callbag-latest](https://github.com/krawaller/callbag-latest), that in essence turns a listenable source into a pullable one. With these operators we can define `newTodoActions` like so:

```javascript
const newTodoActions = pipe(
  fromDelegatedEvent(root, '.new-todo', 'keyup'),
  filter(e => e.key === 'Enter'),
  sample(latest(newNameTypeStream)),
  filter(v => v.length),
  map(v => ({type: 'NEWTODO', value: v})),
  share
);
```

However, when the input value is not in the state, I need to emit an extra empty string on the `newNameTypeStream` after each event on `newTodoActions`. Otherwise you could hit enter again to add the same todo.

But this means that there is a circular dependency between `newNameTypeStream` and `newTodoActions` as both depend on the other! To sort that I had to make [callbag-proxy](https://github.com/krawaller/callbag-proxy) for use in `newNameTypeStream`:

```javascript
const newTodoActions_proxy = proxy();
const newNameTypeStream = pipe(
  fromDelegatedEvent(root, '.new-todo', 'change'),
  map(e => e.target.value),
  mergeWith(pipe(
    newTodoActions_proxy,
    mapTo('')
  ))
);
```

Further down, after `newTodoActions` is defined, we can connect it to the proxy:

```javascript
newTodoActions_proxy.connect(newTodoActions);
```

Another stream shenanigan example - There's an `editSubmissions` typing stream that we use to make `confirmEditActions` (when the edit field has a value) and `deleteActions` (when the edit field is empty). This stream must now contain *both* the event target (for calculating which todo it is) *and* the value (to know the content). For that I had to make the [callbag-sample-combine](https://github.com/krawaller/callbag-sample-combine) operator, so that we have access to both:

```javascript
const editSubmissions = pipe(
  fromDelegatedEvent(root, '.edit', 'keyup'),
  filter(e => e.key === 'Enter'),
  sampleCombine(latest(editTypeStream)),
  map(([e, v]) => ({value: v, target: e.target}))
);
```

The uncontrolled version of the app also needs more work in the side effect parts, since we have to manually set the contents of the inputs.

### Wrapping up

I've done my fair share of stream programming using RxJS, xstream, most, etc. But I've never felt as in control as just now with callbags. The fact that there is no core library, that every sink factory and operator is just a function with a predictable signature, makes for a remarkably pure development experience. It becomes more about the streams, instead of the implementation of the streams.

Of course, having to implement the operators and factories you need yourself is a high price to pay. But since everything is so *simple*, my counter-argument would be that it didn't take me much more energy to do that than it would have for me to wade through the gazillion operators on RxJS to find the one that best suited my needs.

So where does this all leave us? Will coding with callbags be the way forward for reactive programmers, or will it merely be a pattern that'll help with under-the-hood code refactorings for stream libraries?  I'm honestly not sure.

But what I do know is that I really enjoyed putting the TodoMVC app together with callbags, and I'll definitely try to get some more playtime in with them.
