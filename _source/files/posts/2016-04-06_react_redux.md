---
type: post
title: "A look at the React-Redux lib"
date: 2016-04-06
tags: [react, redux]
author: David
excerpt: Exploring the official bindings between React and Redux
draft: false
---

### The premise

Let's take a closer look at [React-Redux](https://github.com/reactjs/react-redux), Dan Abramov's official helper library for **combining React and Redux**. What does this marriage actually entail?

In order to find out, we're going to build a super simple app that uses both React and Redux. We'll make one version where we glue the parts together ourselves, and then one where we let React-Redux act as bridge. 

Here's the super-complicated app we're going to build:

<iframe src="../../applets/fromcourse/demos/usinglibiframe.html" height="200px" width="100%"></iframe>


### The Redux parts

We begin by creating the Redux plumbing needed to support this app! We'll follow the same pattern as discussed in the [Redux example walkthrough post](../a-redux-app-tutorial/).


*	Our **app state** will be an **array of strings**.

*	Let's make our **initial state** prepopulated with a quote to set the tone:

	```javascript
	var initial = ['Carpe diem'];
	```

*	The single **action** that can happen, **adding a quote**, could look like this:

	```javascript
	{
		type: 'ADD',
		text: 'Do unto others etc etc'
	}
	```

*	Here's the **reducer** to handle this setup:

	```javascript
	var reducer = function(state,action){
		switch(action.type){
			case 'ADD': return state.concat(action.text);
			default: return state;
		}
	};
	```

*	We instantiate the **store** as per usual:

	```javascript
	var store = Redux.createStore(reducer,initial);
	```

*	We only need one **action creator** to interact with our store, but let's put it in an object anyway:

	```javascript
	var actionCreators = {
		addQuote: function(text){
			return {type:'ADD',text:text};
		}
	};
	```

Now all the **Redux parts** needed to support our functionality are in place!


### React components

The app can be broken down into **two components**:

*   `QuoteList` to **display the quotes**
*   `QuoteForm` to hold the **form for new quotes**

We farmed out the creation of these components to some interns. Because they didn't know what data handling solution the app would use, they've made the components **portable**. This of course is **good practice** in any scenario, as it makes the code **easier to test** and **less coupled**.

Let's check out what they gave us!

*	First we have `QuoteList`. It **expects to receive an array of quotes** in the `quotes` property:

	```javascript
	var QuoteList = function(props){
		var list = props.quotes.map(function(q,n){
			return <li key={n}>{q}</li>;
		});
		return (
			<div className="quoteslist">
				<h3>Words of Wisdom</h3>
				<ul>{list}</ul>
			</div>
		);
	};
	```

*	And then we have `QuoteForm`, which **renders the form** where the user enters a new quote. This components **expects to receive a callback** in the `callback` property which will be invoked with the text of the new quote when the user clicks the button.

	```javascript
	var QuoteForm = React.createClass({
		submit: function(e){
			var field = this.refs.field;
			this.props.callback(field.value);
			field.value = '';
			e.preventDefault();
		},
		render: function(){
			return (
				<form onSubmit={this.submit}>
					<input ref='field' />
					<button type='submit'>Add</button>
				</form>
			);
		}
	});
	```

	Note that they had to use the **createClass** syntax for the `QuoteForm` component. Otherwise they couldn't use `ref`s, which they need in order to **collect the value from the input tag**.


### Our mission

Obviously, we are far from done. The Redux code is complete and the React components are in place, but they're **not connected to each other**.

Both React components have a clear interface; they expect to be given certain props. A common pattern is to wrap them in **container components** which are hooked into the data layer and supplies these props. Then our **final app** could be:

```javascript
var App = function(props){
	return (
		<div>
			<QuoteListContainer />
			<QuoteFormContainer />
		</div>
	);
};
```

Creating these container components is exactly what our mission now entails!



### The vanilla version

As promised, before we look at `React-Redux`, we'll have a go at implementing these containers ourselves, vanilla style.


*	First we have `QuoteList`, which expects to be given an **array of quotes** as a prop. Since we must also **update when the data changes**, we need to create a component that **has Redux data as state**. Said and done!

	```javascript
	var QuoteListContainer = React.createClass({
		getInitialState: function(){
			return {quotes:[]};
		},
		componentDidMount: function(){
			store.subscribe(function(){
				this.setState({quotes: store.getState()});
			}.bind(this));
		},
		render: function(){
			return <QuoteList quotes={this.state.quotes} />;
		}
	});
	```

	In the `componentDidMount` lifecycle hook we **initiate a store subscription**, which will **update the component state** on every change.

*	Moving on to `QuoteForm` - that component needs to be given a **callback** which it **invokes with new quotes**. This should of course be **passed along to the store**.

	First we create **bound action creators** for our component to use:

	```javascript
	var boundActionCreators = Redux.bindActionCreators(
		actionCreators,
		store.dispatch
	);
	```

	This will ensure that whatever is returned from the action creators will be passed along to `store.dispatch`.

	We then make a container that simply passes in the bound action creator:

	```javascript
	var QuoteFormContainer = function(props){
		return <QuoteForm callback={boundActionCreators.addQuote}/>;
	};
	```

We can now initialise our app as expected!

```javascript
ReactDOM.render(
	<App/>,
	document.getElementById("container")
);
store.dispatch({type:'BOGUSEVENT'}); // triggers initial render
```

Aaaand, we're done!

This vanilla solution **works just fine**. The only downside is that we had to **access our store to create our containers**;

*    `QuoteListContainer` needed `store.subscribe` and `store.getState`
*    `QuoteFormContainer` needed `store.dispatch`.

A stand-alone vanilla version with full source displayed can be found [here](../../applets/fromcourse/demos/vanilla.html).


### The React-Redux API

Before we create containers with `React-Redux`, let's explore the API!

The primary workhorse of `React-Redux` is the `.connect` method which **generates container components**. Sounds like exactly what we need! It has the following syntax:

```javascript
ReactRedux.connect(
	mapStateToProps,     // connecting to store state
	mapDispatchToProps,  // connecting to store dispatch
	mergeProps           // baking all props together
)(ComponentToWrap)       // the component to be wrapped
```

Notice the slightly weird **double invocation** syntax - the call to connect returns a function which is then immediately called with a component. It could just as well have been created as one single method with 4 arguments, so don't lose sleep over this.

We'll now **look at the 3 parameters to `connect`** one at a time:

*	`mapStateToProps` is a **function**. It will be **invoked with the store state**, and what you **return** from the method will become **additional props on the component**.

	```javascript
	function mapStateToProps(appstate){
		return {
			numberOfPosts: appstate.posts.length
		};
	}
	```

*	`mapDispatchToProps` can be an object or a function. If it is an **object** it is assumed to **contain action creators**. It will be made **available as props on the component**, only now all methods **automatically pipe what they're returning to the store dispatch** much like after a `Redux.bindActionCreators` call.

	If `mapDispatchToProps` is a **function** then it'll be **invoked with the store dispatch**, and you must manually return props and **add dispatch calls yourself**.

	```javascript
	function mapDispatchToProps(dispatch){
		return {
			addPost: function(text){
				var action = actionCreators.addPost(text);
				dispatch(action);
			}
		}
	}
	```

*	Finally the `mergeProps` function handles the fact that a connected component **receives props from 3 sources** which must somehow be **baked together**:

	![bake props](../../img/rr-propsources.png)

	If `mergeProps` isn't supplied then `ReactRedux` will do this **by default**:

	```javascript
	props = Object.assign({}, parentProps, stateProps, dispatchProps)
	```

	For the majority of cases this is fine. But if you want control, **provide `mergeProps`** and do your own baking. It is **called with all three sources** like this:

	```javascript
	function mergeProps(fromState,fromDispatch,fromParent){
		// do your own baking and return it
		return myBakedProps;
	}
	```

Now we know what we need to know in order to put `ReactRedux.connect` to work! There are several other nuances to the API that we didn't look at here, so check the [official homepage](https://github.com/reactjs/react-redux) for those if you're interested.




### React-Redux integration

Ok, let's now remake the integration with the `React-Redux` lib instead! Here's how the two containers are created:

*	We start with making the **`QuoteListContainer`**. Remember, `QuoteList` expected the **array of quotes as a props**, which must also be **kept live** as the data updates. Thus we'll need to use the `mapStateToProps` function.

	We need to **map** the **entire app state**, which is the array of quotes, **to the `quotes` property**:

	```javascript
	var mapStateToQuoteListProps = function(appstate){
		return { quotes: appstate };
	};
	```

	Now we **create the container** using the `.connect` method:

	```javascript
	var QuoteListContainer = ReactRedux.connect(
		mapStateToQuoteListProps
	)(QuoteList);
	```

	This `QuoteListContainer` will **act exactly like the vanilla version**, including keeping the data updated.


*	Now for `QuoteFormContainer`. It doesn't need to map state, but it **expects the `callback` prop** to be a method whose **return value should be `dispatch`ed**.

	Thus it suffices to simply **rename the `addQuote` action creator** to "callback":

	```javascript
	var QuoteFormContainer = ReactRedux.connect(
		null, // don't need appstate
		{callback: actionCreators.addQuote}
	)(QuoteForm);
	```

	If we had been **allowed to edit** the code of `QuoteForm` to **use `props.addQuote` instead** of `props.callback`, then we could have **passed in the actionCreators object** directly:

	```javascript
	var QuoteFormContainer = ReactRedux.connect(
		null,
		actionCreators
	)(QuoteForm);
	```

There, now we have both container components in place!

But, hang on - there's **no single reference to the store** anywhere in our code. How does `.connect` **actually connect** our components to the store?

Here's the **final piece** of the pussle: we must **wrap our entire app in a `Provider`** component which is **given a store reference**:

```javascript
var Provider = ReactRedux.Provider;

ReactDOM.render(
	<Provider store={store}>
		<App/>
	</Provider>,
	document.getElementById("container")
);
```

The `Provider` uses the [React context API](https://facebook.github.io/react/docs/context.html) to supply a store reference to the generated container components.

This version of our app is published [here](../../applets/fromcourse/demos/usinglib.html), again with the full source appended.


### Controversy

Since React Context is "an advanced an experimental feature" where "the API is likely to change", there's been [some concern](http://stackoverflow.com/questions/36428355/react-with-redux-what-about-the-context-issue) around the fact that React-Redux (and others) uses it. What if context goes away and React-Redux ceases to function, what will happen to my app which uses both?

As this post has hopefully shown -  **absolutely nothing**. React-Redux is a very small helper library that saves us from having to pass a store reference around. But that isn't really a big thing, as isolating store usage to hand-made containers is pretty decopupled too.

### Wrapping up

Whether or not you decide to use React-Redux in your React and Redux app or not, I hope this post has helped you to better understand the bridging problem and the various solutions available!

<div class="smallnote">
PS: Shameless plug time again! This post is an (adapted) excerpt from a 2-day React+Redux training course that I provide through my employer Edument. So if you're in Scandinavia and hungry for more, [check it out](http://edument.se/en/training/webdevelopment/react-js)!
</div>

