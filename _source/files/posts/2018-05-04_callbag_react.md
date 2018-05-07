---
type: post
title: "Building TodoMVC with Callbags and React"
date: 2018-05-07
tags: [callbags,react,redux,case study, todomvc]
author: David
excerpt: Making a TodoMVC implementation using Callbags and React, introducing the needed tools as we go!
---

### Premise

A while back I wrote about a [TodoMVC implementation using Callbags](/dissecting-a-callbag-todomvc-implementation/). It used [SnabbDOM](https://github.com/snabbdom/snabbdom) for rendering, and thus didn't really need to address the problem of how to marry Callbags with an actual framework.

In this post we take on that challenge through rewriting the TodoMVC solution to use React instead! The code for the final result is available at [https://github.com/krawaller/callbag-todomvc-react](https://github.com/krawaller/callbag-todomvc-react), but we'll walk through the highlights below.

### The old app

Let's begin by reviewing what we already have. The [previous implementation](https://github.com/krawaller/callbag-todomvc) was built like this...

![](../../diagrams/callbag-mvi.svg)

...which translates to [this bootstrapping code](https://github.com/krawaller/callbag-todomvc/blob/master/src/index.js):

```javascript
const actions = makeActions(window, window.document.getElementById('app'));
const state = makeStateStream(actions);
const view = makeViewStream(state);

makeSideEffects(window, actions, view);
```

Here's a brief rundown of the involved functions:

* In [`makeActions`](https://github.com/krawaller/callbag-todomvc/blob/master/src/actions.js) we create a bunch of action streams by using [`callbag-from-delegated-event`](https://github.com/krawaller/callbag-from-delegated-event). Here's an example:

  ```javascript
  const newTodoActions = pipe(
    fromDelegatedEvent(root, '.new-todo', 'keyup'),
    filter(e => e.key === 'Enter'),
    mapTo({type: 'NEWTODO'}),
  );
  ```

  The function returns all of the action streams in an object, and throw in an `allActions` stream for good measure.

* The [`makeState`]() function turns the actions into a stream of app state:

  ```javascript
  function makeStateStream(actions){
    return pipe(
      actions.allActions,
      scan(augmenter(reducer), initialState),
    );
  }
  ```

  The [`callbag-scan`](https://github.com/staltz/callbag-scan) operator reduces the stream, which means the end result of this file is a miniature Redux implementation.

* The [`makeView`](https://github.com/krawaller/callbag-todomvc/blob/master/src/view.js) function simply maps the state emissions to JSX:

  ```javascript
  function makeViewStream(state){
    return pipe(
      state,
      map(s => (
        <div>
          ...
        </div>
      ))
    );
  }
  ```

  This really is just a templating layer - we're not dealing with event handlers or anything of the sort, we just express what the UI should look like given the state. In other words we could very easily replace SnabbDOM with something like Handlebars.

* Inside [`makeSideEffects`](https://github.com/krawaller/callbag-todomvc/blob/master/src/sideeffects.js) we do three things:

  * update the DOM for all `view` emissions (using SnabbDOM), getting the DOM reference from `window`:

    ```
    window.document.getElementById('renderoutput')
    ```

  * focus the edit Todo input on emissions from `actions.editActions`:

    ```javascript
    pipe(
      actions.editActions,
      forEach(() => window.document.querySelector(".editing .edit").focus())
    );
    ```

  * focus the new Todo input on emissions from a bunch of actions in a similar manner as above


### The three problems

As you've seen, we're reading and writing to the DOM throughout our solution. Which of course rhymes rather badly with a framework, which wants to own the DOM and expects you to express your intentions through the framework instead!

All things considered, we have **3 problems to solve** in order to be able to use React efficiently:

1. We need to let React capture our DOM events
1. We need to make React rerender when there's new state
1. We need to make React do the other focusing side effects

We'll now walk through these one at a time!


### Problem 1 - catching DOM events

Let's start with the events! As you saw above, the action streams are seeded with a source created from `fromDelegatedEvent`, which sets a listener directly on the DOM.

But, in React we are expected to reference our handlers in the render output!

```html
<button onClick={this.launchMissile}>Launch</button>
```

In a situation where we use Redux, we would probably call an action creator in that handler:

```javascript
launchMissile(e){
  boundActionCreators.triggerLaunch();
}
```

We want to do something very similar now, but, what does that translate to when we have Callbags instead of Redux?


### Introducing `callbag-from-function`

There are two facts we cannot escape:

* React connects a DOM event with a function
* The action streams must start with a source

In other words - we need the missing piece in this chain:

![](../../diagrams/callbag-from-function-chain.svg)

To solve this I made `callbag-from-function` - it is a factory function that takes a function (or defaults to the identity function), and returns an emitter and a source:

![](../../diagrams/callbag-from-function-api.svg)

The emitter behaves exactly like the passed-in function, except it will also make the source emit all returned values.

In the React version of the app I've added an [`events.js`](https://github.com/krawaller/callbag-todomvc-react/blob/master/src/data/events.js) file looking like this:

```javascript
import fromFunction from 'callbag-from-function';

export default {
  clearCompleted: fromFunction(),
  // ... lots more ...
}
```

In the React app we'd then use the emitter as handler in the JSX output from `render`...

```javascript
<button onClick={clearCompleted.emitter}>Clear completed</button>
```

...and in the new version of `makeActions` we'd seed the relevant stream thusly:

```javascript
const clearCompletedActions = pipe(
  clearCompleted.source, // <--- instead of using `fromDelegatedEvent`
  mapTo({type: 'CLEARCOMPLETED'})
);
```

And so we have solved problem number 1 - DOM events are now handled by React, yet still piped into the Callbag layer!

### Problem 2 - rerendering on new state

In the earlier solution, rerendering is one of the side effects dealt with in `makeSideEffects`:

```javascript
pipe(
  withPrevious(view),
  forEach(([cur,prev,isFirst]) => {
    patch(isFirst ? window.document.getElementById('renderoutput') : prev, cur)
  })
);
```

There's some SnabbDOM shenanigans going on here, but the essence is that whenever the `view` stream outputs a new VirtualDOM emission, we mutate the DOM.

In React we instead want our `App` component to rerender, presumably by passing in new props. How do we accomplish that?

### Introducing `callbag-connect-react`

The situation is very similar to using Redux with React - if we have Redux as a data layer, then we want to rerender our React app whenever the store has new data. Commonly we'd use [`ReactRedux`](https://github.com/reactjs/react-redux) to automatically pass that data into the top of our component pyramid as props.

I've created a very similar decorator tool for Callbags, published in the npm package [`callbag-connect-react`](https://github.com/krawaller/callbag-connect-react). We feed the decorator an object where...

* each **key** is a **prop name**
* each **value** is a **callbag** whose emissions will populate the prop named by the key.

```javascript
@connect({appState: stateStream})
class App extends React.Component {
  // ...implementation using `this.props.appState`
}
```

With the above code, the `appState` prop will always update to whatever `stateStream` emits.

And so we have solved the rendering problem - our React component will now rerender for every new state emission!

### Problem 3 - other side effects

That took care of one of the side effects, but we have two left; focusing input fields at various points in the component life cycle. They're both very similar, so I'll just talk through one of them here. In the final app they both benefit from the same solution.

The old code in `makeSideEffects` looks like this:

```javascript
pipe(
  actions.editActions,
  forEach(() => window.document.querySelector(".editing .edit").focus())
);
```

Here's the kicker - there's **no analogue for this in Redux**. Redux is built around the notion that we have a single source that is fed into the app, causing a rerender each time. But what we have here is a *different* source, where we want something *other than rendering* to happen.

We can move the actual focusing call to a `.focusEditField` method in the component easily enough:

```javascript
class App extends React.Component {
  constructor(props){
    super(props);
    this.editField = React.createRef();
  }
  focusEditField(){
    this.editField.current.focus();
  }
  render(){
    return (
      // JSX using `this.editField` as ref for the relevant input element
    );
  }
}
```

But how do we make it so that `.focusEditField` is called when `editActions` emit?

### Introducing signals

To handle this I've made `connect` also take a second parameter; an array of what I call **signals**.

Each signal is an **array of two elements**:

* the first is a **callbag source**
* the second is a **callback function** to be invoked when that source emits. That callback will be invoked with...
  * a reference to the **component instance**
  * the **value that was emitted** from the source (not used below)

```javascript
@connect(
  {appState: stateStream},
  [ [actions.editActions, comp => comp.focusEdit()] ]
)
class App extends React.Component {
  // ...
}
```

Now, whenever `editActions` emit, the `focusEdit` method will be invoked!

Note that the signals are nothing like `mapDispatchToProps` from `ReactRedux` - that just lets us dispatch without a (direct) reference to `store.dispatch`. The signals here solve a completely different problem - acting on a stream beyond the first - which, again, is beyond the scope of Redux (without use of exotic middleware).


### The final result

The **old app** had this flow...

![](../../diagrams/callbag-mvi.svg)

...which meant [this bootstrapping](https://github.com/krawaller/callbag-todomvc/blob/master/src/index.js):

```javascript
const actions = makeActions(window, window.document.getElementById('app'));
const state = makeStateStream(actions);
const view = makeViewStream(state);

makeSideEffects(window, actions, view);
```

The **new, React-based app** - with the repo [https://github.com/krawaller/callbag-todomvc-react](https://github.com/krawaller/callbag-todomvc-react) - ended upp looking like this:

![](../../diagrams/callbag-mvi-2.svg)

Here's the [new bootstrapping code](https://github.com/krawaller/callbag-todomvc-react/blob/master/src/index.js):

```javascript
const events = makeEvents();
const actions = makeActions(events, window);
const state = makeStateStream(actions);
const App = makeApp(state, actions, events);

ReactDOM.render(
  <App/>,
  document.getElementById('app')
);
```

Let's tour the involved factory functions and compare them to their older counterparts!

* The [`makeEvents`](https://github.com/krawaller/callbag-todomvc-react/blob/master/src/data/events.js) function was a new addition, using `callbag-from-function` to create bridges between normal JavaScript functions and callbag sources:

  ```javascript
  function makeEvents(){
    return {
      clearCompleted: fromFunction(),
      toggleTodo: fromFunction(),
      // ...
    };
  }
  ```

  This is (somewhat) comparable to (parts of) the action creator layer in Redux.

* The [`makeActions`](https://github.com/krawaller/callbag-todomvc-react/blob/master/src/data/actions.js) function is almost exactly the same as before, except we use sources from `makeEvents` to seed the streams instead of setting event handlers ourselves:

  ```javascript
  const clearCompletedActions = pipe(
    clearCompleted.source, // <--- instead of using `fromDelegatedEvent`
    mapTo({type: 'CLEARCOMPLETED'})
  );
  ```

* The new [`makeState`](https://github.com/krawaller/callbag-todomvc-react/blob/master/src/data/state.js) function is completely identical to the old one!

* The [`makeApp`](https://github.com/krawaller/callbag-todomvc-react/blob/master/src/app/index.js) function replaces the old `makeView` and `makeSidEffects`. This makes sense, since...

  * the `.render` method plays the role of `makeView`, turning app state into JSX
  * the `.connect`ed state source triggers the rerender, replacing the first side effect in `makeSideEffects`
  * the signals take care of the other side effects

  The `makeApp` function is implemented as a factory that returns the component definition:

  ```javascript
  function makeApp(state, actions, events){
    const shouldFocusNew = merge(
      actions.initActions, actions.deleteActions, actions.confirmEditActions,
      actions.newTodoActions, actions.cancelEditActions
    );
    
    @connect(
      {appState: stateStream},
      [
        [shouldFocusNew, comp => comp.focusNew()],
        [actions.editActions, comp => comp.focusEdit()]
      ]
    )
    class App extends React.Component { ... }

    return App;
  }
  ```

### Wrapping up

Now we have successfully used callbags in conjunction with a framework! Was this actually a good idea?

I feel like there are two separate comparisons to be made:

* **SnabbDOM VS React**: In other words, did we gain anything when switching out a templating solution for a framework (ok, I know not all agree that React can be called a framework...)? Well, maybe we didn't gain much in this small app, but the newer version definitely scales better, and we're touching the DOM way less.

* **Callbag VS Redux**: Now this is more relevant - are we better of using Callbags for our data layer as opposed to Redux? This will always be a very subjective discussion, but my main takeaway is that I'm very happy about how easy it is to express *other sideeffects* with callbags.

What I definitely do know is that I enjoy juggling callbags very much, and look forward to an opportunity to try it out on a larger scale project. And now that we have a working setup for marrying callbags with React, the threshold for such an adventure is way lower!

