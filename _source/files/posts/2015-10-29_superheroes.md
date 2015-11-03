---
title: A React-Redux example app 
author: David
tags: [react, redux]
date: 2015-10-29
excerpt: "Dissecting a small React-Redux example application"
type: post
---

###The Superhero battler app

This post presents a simple React-Redux example app. The aim was to have a small example to complement the official Redux ones, that is also more portable. It uses React, Redux, React-Router, React-Redux and browserify. And (almost) no ES6 syntax.

Instead of the venerable ToDo, the app is a tiny "game" where superheroes battle it out until there's only one left standing:

<iframe src="http://blog.krawaller.se/riastart2015/" style="height:500px;width:100%"></iframe>

The url for the app is [http://blog.krawaller.se/riastart2015/](http://blog.krawaller.se/riastart2015/), and the source code is at [https://github.com/krawaller/riastart2015](https://github.com/krawaller/riastart2015).

Kick the tires for a bit so you get a feel for what's going on, then read on to see how it is all put together!

###Under the hood

These are the involved files (along with `index.html` and the generated `bundle.js`):

![components](../../img/superherofiles.png)

Before we look at the components, let's examine the main plumbing first. The entry point is `index.js`:

```javascript
var React = require('react'),
	ReactDOM = require('react-dom'),
	Router = require('react-router').Router,
	Provider = require('react-redux').Provider,
	store = require('./store'),
	routes = require('./routes');

ReactDOM.render(
	// The top-level Provider is what allows us to `connect` components to the store
	// using ReactRedux.connect (see components Home and Hero)
	<Provider store={store}>
		<Router routes={routes}/>
	</Provider>,
	document.getElementById("root")
);
```

In here we load `routes.js`, hook up `store.js`, and feed it all to `ReactDOM.render`!

### Data model

Now we enter Redux land! I won't go into too much detail on how Redux works, as [the official docs](http://redux.js.org/) are just marvelous at doing that. Instead we'll focus on how this app uses Redux.

Here's what our store (in Redux there's only one) looks like:

```javascript
var Redux = require("redux"),
	heroReducer = require("./reducers/heroes"),
	battlefieldReducer = require("./reducers/battlefield"),
	initialState = require("./initialstate"),
	thunk = require('redux-thunk'); // allows us to use asynchronous actions

var rootReducer = Redux.combineReducers({
	heroes: heroReducer,   // this means heroReducer will operate on appState.heroes
	battlefield: battlefieldReducer // battlefieldReducer will operate on appState.battlefield,
});

module.exports = Redux.applyMiddleware(thunk)(Redux.createStore)(rootReducer,initialState());
```

There are some weird parts to this, but what's relevant for our app is `initialstate` and the two reducers, `heroReducer` and `battlefieldReducer`.

Let's peek at `initialstate` first, as that will also serve as a picture of what the app data model looks like:

```javascript
var C = require("./constants");

module.exports = function(){
	return {
		heroes: {
			batman: {
				quote: "I'm batman.",
				kills: 0
			},
			superman: {
				quote: "I can fly!",
				kills: 0
			},
			spiderman: {
				quote: "Why don't you love me, Lois?",
				kills: 0
			},
			"he-man": {
				quote: "By the power of Grayskull!",
				kills: 0
			}
		},
		battlefield: {
			doing: {batman:C.WAITING,superman:C.WAITING,spiderman:C.WAITING,"he-man":C.WAITING},
			standing: 4,
			log: ["Ready.... fight!"]
		}
	};
};
```

As you can see there are two main parts to the data; `heroes` and `battlefield`. These parts correspond to the two reducers you saw earlier in the store file.

The constants file is merely just that, a list of constants:

```
module.exports = {
	// ACTION TYPES
	AIM_AT: "AIM_AT",
	KILL_HERO: "KILL_HERO",
	RESET: "RESET",
	DUCK_DOWN: "DUCK_DOWN",
	STAND_UP: "STAND_UP",

	// HERO STATUSES
	AIMING: "AIMING",
	DUCKING: "DUCKING",
	WAITING: "WAITING",
	DEAD: "DEAD"
};
```

###Mutating the data

Now we've seen what the data looks like; how is it updated? Through `actions`, that's how! Here's what the action creators for our app looks like:

```javascript
var constants = require("./constants");

module.exports = {
	reset: function(){
		// A normal action creator, returns a simple object describing the action.
		return {type:constants.RESET};
	},
	duckDown: function(who){
		// here we take advantage of Redux-thunk; instead of returning an object describing an action,
		// we return a function that takes dispatch and getState as arguments. This function can then
		// invoke dispatch, now or later using setTimeout or similar.
		return function(dispatch,getState){
			dispatch({type:constants.DUCK_DOWN,coward:who});
			setTimeout(function(){
				dispatch({type:constants.STAND_UP,coward:who});
			},2000);
		}
	},
	aimAt: function(killer,victim){
		// Another async action using the Redux-thunk syntax
		return function(dispatch,getState){
			dispatch({type:constants.AIM_AT,killer:killer,victim:victim});
			setTimeout(function(){
				dispatch({type:constants.KILL_HERO,killer:killer,victim:victim});
			},2000);
		};
	}
};
```

These functions will be used inside components in response to user events. But before we look at those, let's see what happens when they are invoked - that magic is all in the reducers!

First off the `heroReducer`, as that is by far the simplest. The only thing it does is increase the kill count whenever a hero kills another.

```javascript
var C = require("../constants"),
	initialState = require("../initialstate");

module.exports = function(state,action){
	var newstate = Object.assign({},state); // sloppily copying the old state here, so we never mutate it
	switch(action.type){
		case C.KILL_HERO:
			newstate[action.killer].kills += 1;
			return newstate;
		default: return state || initialState().heroes;
	}
};
```

The `battlefieldReducer`, however, has far more responsibilities! 

```javascript
var C = require("../constants"),
	initialState = require("../initialstate");

module.exports = function(state,action){
	var newstate = Object.assign({},state); // sloppily copying the old state here, so we never mutate it
	switch(action.type){
		case C.DUCK_DOWN:
			newstate.doing[action.coward] = C.DUCKING;
			newstate.log.push(action.coward+" ducks down like a coward.");
			return newstate;
		case C.STAND_UP:
			newstate.doing[action.coward] = C.WAITING;
			newstate.log.push(action.coward+" stands back up.");
			return newstate;
		case C.AIM_AT:
			newstate.doing[action.killer] = C.AIMING;
			newstate.log.push(action.killer+" takes aim at "+action.victim+"!");
			return newstate;
		case C.KILL_HERO:
			// the shooter has died before he got the shot off
			if (state.doing[action.killer] === C.DEAD){
				newstate.log.push("The trigger finger twitches on "+action.killer+"'s corpse");
			} else {
				newstate.doing[action.killer] = C.WAITING;
				// the target is ducking
				if (state.doing[action.victim] === C.DUCKING) {
					newstate.log.push(action.victim+" dodges a blast from "+action.killer+"!");
				// the target has already been killed
				} else if (state.doing[action.victim] === C.DEAD) {
					newstate.log.push(action.killer+" blasts "+action.victim+"'s corpse.");
				// we kill the target!
				} else {
					if (state.doing[action.victim]===C.AIMING){
						newstate.log.push(action.killer+" killed "+action.victim+" before he got his shot off!");
					} else {
						newstate.log.push(action.killer+" killed "+action.victim+"!");
					}
					newstate.doing[action.victim] = C.DEAD;
					newstate.standing = newstate.standing - 1;
					if (newstate.standing === 1){
						newstate.log.push(action.killer+" WINS!!");
					}
				}
			}
			return newstate;
		case C.RESET:
			return initialState().battlefield;
		default: return state || initialState().battlefield;
	}
};
```

Please take a moment to look at the various cases, and connect it to how the app behaved when you tested it earlier!

###Components

The app are made up by these components:

![components](../../img/superherocomponents.png)

The hierarchy isn't strictly true as `Hero` and `Home` are served through `routes`:


```javascript
var React = require('react'),
    ReactRouter = require('react-router'),
    Route = ReactRouter.Route,
    IndexRoute = ReactRouter.IndexRoute,
    Wrapper = require('./components/wrapper'),
    Home = require('./components/home'),
    Hero = require('./components/hero');

module.exports = (
    <Route path="/" component={Wrapper}>
        <IndexRoute component={Home} />
        <Route path="/hero/:name" component={Hero} />
    </Route>
);
```

The app only has two routes, either we're in the main battle arena or we are in a closeup of one of the heroes.

`Wrapper` doesn't do anything exciting, it simply renders out the children:

```javascript
var React = require('react');

var Wrapper = React.createClass({
    render: function() {
        return (
            <div className="wrapper">
                <h2>Superhero battle arena 2000!</h2>
                {this.props.children}
            </div>
        );
    }
});

module.exports = Wrapper;
```

For slightly higher excitement let's take a look at `Hero`, our first Redux-aware component:

```javascript
var React = require("react"),
	ptypes = React.PropTypes,
	Link = require("react-router").Link,
	ReactRedux = require("react-redux");

var Hero = React.createClass({
	propTypes: {
		params: ptypes.shape({name:ptypes.string.isRequired}).isRequired, // will be provided by react-router
		heroes: ptypes.object.isRequired // will be provided by react-redux
	},
	render: function(){
		var name = this.props.params.name,
			data = this.props.heroes[name];
		return <div>
			<p><Link to="/">Back to arena</Link></p>
			<p>Here's some info on {this.props.params.name}:</p>
			<p><strong>Quote:</strong> {data.quote} </p>
			<p><strong>Kills:</strong> {data.kills} </p>
		</div>;
	}
});

// connect to Redux store

var mapStateToProps = function(state){
	// This component will have access to `appstate.heroes` through `this.props.heroes`
	return {heroes:state.heroes};
};

module.exports = ReactRedux.connect(mapStateToProps)(Hero);

```

Through `ReactRedux` we give the component access to the `battlefield` part of the app state. The component can the use that just as any other property data served down from a higher-level component.

Using `ReactRouter` we learn the name of the current hero we've navigated to (there should probably be some error checking in there though), and then we can dig out the correct data.

###Getting dirty

On to the next Redux-aware component - `Home`, which is the main battle arena!

```javascript
var React = require("react"),
	ptypes = React.PropTypes,
	ReactRedux = require("react-redux"),
	Log = require("./log"),
	Battlers = require("./battlers"),
	actions = require("../actions");

var Home = React.createClass({
	propTypes: {
		// redux store state, imported below
		battle: ptypes.shape({ 
			doing: ptypes.object.isRequired,
			log: ptypes.arrayOf(ptypes.string)
		}).isRequired,
		// redux action hookups, set up below
		kill: ptypes.func.isRequired,
		duck: ptypes.func.isRequired,
		reset: ptypes.func.isRequired
	},
	render: function(){
		var battleprops = this.props.battle;
		return (
			<div>
				<Battlers doing={battleprops.doing} kill={this.props.kill} duck={this.props.duck} />
				<Log log={battleprops.log}/>
				{ battleprops.standing === 1 && <button onClick={this.props.reset}>Reset</button> }
			</div>
		);
	}
});

// now we connect the component to the Redux store:

var mapStateToProps = function(state){
	// This component will have access to `state.battlefield` through `this.props.battle`
	return {battle:state.battlefield};
};

var mapDispatchToProps = function(dispatch){
	return {
		kill: function(killer,victim){ dispatch(actions.aimAt(killer,victim)); },
		duck: function(coward){ dispatch(actions.duckDown(coward)); },
		reset: function(){ dispatch(actions.reset()); }
	}
};

module.exports = ReactRedux.connect(mapStateToProps,mapDispatchToProps)(Home);
```

Like before we connect it to the Redux Store, but unlike before we also import the action creators we saw earlier to use as callbacks!

Of these, `reset` is used in this very component while `kill` and `duck` are passed down to the `Battlers` component:

```javascript
var React = require("react"),
	ptypes = React.PropTypes,
	Battler = require("./battler"),
	_ = require("lodash");

var Battlers = React.createClass({
	propTypes: {
		kill: ptypes.func.isRequired,
		duck: ptypes.func.isRequired,
		doing: ptypes.object.isRequired
	},
	render: function(){
		var p = this.props, boxes = _.map(p.doing,function(doing,name){
			var kill = p.kill.bind(this,name), // prefill the kill method so that killer is always `name`
				duck = p.duck.bind(this,name); // make sure battler can only duck himself
			return <Battler key={name} name={name} doing={p.doing} kill={kill} duck={duck} />;
		},this);
		return <div className="battlers">{boxes}</div>;
	}
});

module.exports = Battlers;
```

This components renders a `Battler` component for each super hero, giving them their name, personalized versions of `kill` and `duck`, as well as the full `doing` object containing the status of everyone. Here's the code for `Battler`:

```javascript
var React = require("react"),
	ptypes = React.PropTypes,
	_ = require("lodash"),
	Link = require("react-router").Link,
	C = require("../constants");

var Battler = React.createClass({
	propTypes: {
		name: ptypes.string.isRequired,
		kill: ptypes.func.isRequired,
		duck: ptypes.func.isRequired
	},
	render: function(){
		var p = this.props,
			name = p.name,
			doing = p.doing,
			// list all enemies that aren't dead yet
			killable = _.reduce(doing,function(list,status,opp){
				return status !== C.DEAD && opp!==name ? list.concat(opp) : list;
			},[]),
			// make buttons for all killable enemies
			buttons = killable.map(function(opp){
				return <button key={opp} onClick={p.kill.bind(this,opp)}>{"Kill "+opp}</button>;
			},this).concat(<button key="duck" onClick={p.duck}>duck</button>);
		//controls depend on what we're doing
		var controls = { // using ES6 syntax for dynamic object properties
			[C.WAITING]: buttons.length > 1 ? buttons : "Winner!!",
			[C.DUCKING]: "ducking",
			[C.DEAD]: "...dead...",
			[C.AIMING]: "aiming!"
		}[p.doing[name]];
		return <div className="battler">
			<Link to={"/hero/"+name}>{name}</Link>
			<div>{controls}</div>
		</div>;
	}
});

module.exports = Battler;
```

Here we finally hook up `kill` and `duck` to user interactions.

For completion's sake let's also list the code for `Log`, although there's nothing much going on there:

```javascript
var React = require("react"),
	ptypes = React.PropTypes;

var Log = React.createClass({
	propTypes: {
		log: ptypes.arrayOf(ptypes.string).isRequired
	},
	render: function(){
		return <ul>{this.props.log.map(function(txt,n){
			return <li key={n}>{txt}</li>;
		})}</ul>;
	}
});

module.exports = Log;
```

###Wrapping up

As warned I didn't explain much about how Redux works, but again, the [official docs](http://redux.js.org/) are really brilliant. But I hope this hands-on example helped you learn what I have; that Redux, although tiny, is one beast of a data flow library!