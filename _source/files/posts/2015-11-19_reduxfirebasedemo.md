---
title: A React-Redux-Firebase app with authentication
author: David
tags: [react, redux, case study, firebase]
date: 2015-11-22
excerpt: "Another Redux sample app, this time coupled with Firebase. Demonstrates authentication, validation and keeping all UI state in Redux."
type: post
---

### The quotes app

This post presents an example app demonstrating how to can use Firebase on a React-Redux foundation. The app is a simple collection of quotes. If you log in (using Github credentials) you can add a new quote and edit/delete previous quotes added by you. Try it out below!

<iframe src="http://blog.krawaller.se/reduxfirebasedemo/" style="height:600px;width:100%"></iframe>

You'll find the code repo [here](https://github.com/krawaller/reduxfirebasedemo), and the app is published [here](http://blog.krawaller.se/reduxfirebasedemo/#/?_k=98yl14).

The app was built as a demo for the [RIA development course](https://coursepress.lnu.se/kurs/ria-utveckling-med-javascript/) I'm running at [Linnaeus university](http://lnu.se).

It is meant to serve as an example of several things:

*    How to couple Redux and Firebase
*    How to let Redux manage all UI state
*    How to deal with validation in Redux
*    How to handle authentication


### App state

As with all Redux app, the entire state of the app is stored in an object in one single store. Here's what the [initial state](https://github.com/krawaller/reduxfirebasedemo/blob/gh-pages/src/store/initialstate.js) looks like:

```javascript
var C = require("../constants");

module.exports = {
	feedback: [
		{msg:"Welcome to this little demo! It is meant to demonstrate three things:",error:false},
		{msg:"1) How to use Redux + Firebase",error:false},
		{msg:"2) How to use authentication in a Redux app",error:false},
		{msg:"3) How to have all UI state in Redux and none in the components",error:false}
	],
	auth: {
		currently: C.ANONYMOUS,
		username: null,
		uid: null
	},
	quotes: {
		hasreceiveddata: false,
		submittingnew: false,
		states: {}, // this will store per quote id if we're reading, editing or awaiting DB response
		data: {} // this will contain firebase data
	}
};
```

As you can see the state can be divided into three parts;

1.    `feedback`: An array of UI messages to the user
2.    `auth`: Information about the currently logged in user
3.    `quotes`: The actual list of quotes, and also the UI state of each quote

We'll now look at the corresponding reducer and action creator for these three parts one at a time.

### Authentication

Beginning with authentication, here's what the [reducer](https://github.com/krawaller/reduxfirebasedemo/blob/gh-pages/src/store/reducers/auth.js) looks like:

```javascript
var C = require("../../constants"),
	initialState = require("../initialstate");

module.exports = function(currentstate,action){
	switch(action.type){
		case C.ATTEMPTING_LOGIN:
			return {
				currently: C.AWAITING_AUTH_RESPONSE,
				username: "guest",
				uid: null
			};
		case C.LOGOUT:
			return {
				currently: C.ANONYMOUS,
				username: "guest",
				uid: null
			};
		case C.LOGIN_USER:
			return {
				currently: C.LOGGED_IN,
				username: action.username,
				uid: action.uid
			};
		default: return currentstate || initialState.auth;
	}
};
```

You'll note there are three things that can happen;

1.    `ATTEMPTING_LOGIN` which is the user having entered his credentials and now awaiting a response
2.    `LOGIN_USER` represents a successfull login
2.    `LOGOUT` means we discard the current user credentials and reenter guestmode

So far we've seen no trace of actual Firebase code, which is to be expected. The reducers should always be instantaneous and free of side effects, and simply react synchronously to an action notification just received.

Instead, the related Firebase code can be found in the [authentication action creators](https://github.com/krawaller/reduxfirebasedemo/blob/gh-pages/src/actions/auth.js):

```javascript
var C = require("../constants"),
	Firebase = require("firebase"),
	fireRef = new Firebase(C.FIREBASE);

module.exports = {
	// called at app start
	startListeningToAuth: function(){
		return function(dispatch,getState){
			fireRef.onAuth(function(authData){
				if (authData){ 
					dispatch({
						type: C.LOGIN_USER,
						uid: authData.uid,
						username: authData.github.displayName || authData.github.username
					});
				} else {
					if (getState().auth.currently !== C.ANONYMOUS){ // log out if not already logged out
						dispatch({type:C.LOGOUT});
					}
				}
			});
		}
	},
	attemptLogin: function(){
		return function(dispatch,getState){
			dispatch({type:C.ATTEMPTING_LOGIN});
			fireRef.authWithOAuthPopup("github", function(error, authData) {
				if (error) {
					dispatch({type:C.DISPLAY_ERROR,error:"Login failed! "+error});
					dispatch({type:C.LOGOUT});
				} else {
					// no need to do anything here, startListeningToAuth have already made sure that we update on changes
				}
			});
		};
	},
	logoutUser: function(){
		return function(dispatch,getState){
			dispatch({type:C.LOGOUT}); // don't really need to do this, but nice to get immediate feedback
			fireRef.unauth();
		};
	}
};
```

We're using [Redux-Thunk](https://github.com/gaearon/redux-thunk) which allows for the action creators to return a function instead of a straight-up actions object, making asynchronous actions possible.

There are three authentication-related action creators;

1.    `startListeningToAuth` is called att app start, setting upp the real-time updates from Firebase. This means we never have to bother catching the result of subsequent auth requests made to Firebase, this listener will catch them all!
2.    `attemptLogin` is when a user submits his credentials. This synchronously fires an `ATTEMPTING_LOGIN` action, setting the spinner to notify the user that judgement is a-coming.
3.    `logoutUser` is when a user clicks log out. Note that we synchronously log out here, not totally necessary since Firebase would push the log out to us.

Here's what the [app-start kickoff](https://github.com/krawaller/reduxfirebasedemo/blob/gh-pages/src/index.js) looks like:

```javascript
var React = require('react'),
	ReactDOM = require('react-dom'),
	Router = require('react-router').Router,
	Provider = require('react-redux').Provider,
	store = require('./store'),
	routes = require('./routes'),
	actions = require('./actions')

ReactDOM.render(
	<Provider store={store}>
		<Router routes={routes}/>
	</Provider>,
	document.getElementById("root")
);

setTimeout(function(){
	store.dispatch( actions.startListeningToAuth() );
	store.dispatch( actions.startListeningToQuotes() );
});
```

Note the action calls at the very bottom.

Now you've seen all authentication-related code. This code ensures that `appstate.auth` contains the current user, which can then be queried by your other code. In a production app you'd also have security rules on the Firebase side of things to prevent easy manipulation.

### Quotes

Now let's look at the quotes-related code. To save you scrolling back, here's what the quote state looked like:

```javascript
quotes: {
	hasreceiveddata: false,
	submittingnew: false,
	states: {}, // this will store per quote id if we're reading, editing or awaiting DB response
	data: {} // this will contain firebase data
}
```

With that fresh on our retinas, here's the [reducer](https://github.com/krawaller/reduxfirebasedemo/blob/gh-pages/src/store/reducers/quotes.js):

```javascript
var C = require("../../constants"),
	initialState = require("../initialstate"),
	_ = require("lodash");

module.exports = function(currentstate,action){
	var newstate;
	switch(action.type){
		case C.RECEIVE_QUOTES_DATA:
			return Object.assign({},currentstate,{
				hasreceiveddata: true,
				data: action.data
			});
		case C.AWAIT_NEW_QUOTE_RESPONSE:
			return Object.assign({},currentstate,{
				submittingnew: true
			});
		case C.RECEIVE_NEW_QUOTE_RESPONSE:
			return Object.assign({},currentstate,{
				submittingnew: false
			});
		case C.START_QUOTE_EDIT:
			newstate = _.cloneDeep(currentstate);
			newstate.states[action.qid] = C.EDITING_QUOTE;
			return newstate;
		case C.FINISH_QUOTE_EDIT:
			newstate = _.cloneDeep(currentstate);
			delete newstate.states[action.qid];
			return newstate;
		case C.SUBMIT_QUOTE_EDIT:
			newstate = _.cloneDeep(currentstate);
			newstate.states[action.qid] = C.SUBMITTING_QUOTE;
			return newstate;
		default: return currentstate || initialState.quotes;
	}
};
```

As the switch statement shows there are 6 quote actions:

1.    `RECEIVE_QUOTES_DATA` means an update from Firebase. We grab that payload and put it in `appstate.quotes.data`.
2.    `AWAIT_NEW_QUOTE_RESPONSE` is when we have submitted a new quote but haven't yet had an answer, so we should show a spinner in the form for adding new quotes.
3.    `RECEIVE_NEW_QUOTE_RESPONSE` means we got the answer on our request to add a new quote, so the new quote form should no longer show a spinner.
4.    `START_QUOTE_EDIT` is when a user clicked "edit" on a quote he/she has written, so that qoute should now be displayed in a form.
5.    `C.FINISH_QUOTE_EDIT` means they cancelled or received submit result, so the quote should no longer be displayed in a form
6.    `SUBMIT_QUOTE_EDIT` is when they submitted an edit, so the form for that quote should now display a spinner.

Let's take a look at the render function for [a single quote](https://github.com/krawaller/reduxfirebasedemo/blob/gh-pages/src/pages/components/quote.js), showing off all the quote state we've seen so far:

```javascript
function(){
	var p = this.props,
		q = p.quote,
		button;
	if (p.state === C.EDITING_QUOTE){
		return (<form className="quote" onSubmit={this.submit}>
			<input ref="field" defaultValue={q.content}/>
			<button type="button" onClick={p.cancel}>Cancel</button>
			<button type="submit" onClick={this.submit}>Submit</button>
		</form>);
	}
	if (!p.mayedit){
		button = ""
	} else if (p.state === C.SUBMITTING_QUOTE) {
		button = <button disabled="disabled">Submitting...</button>;
	} else {
		button = <span><button onClick={p.edit}>Edit</button><button onClick={p.delete}>Delete</button></span>;
	}
	return <div className="quote"><span className="author">{q.username+" said: "}</span>{q.content} {button}</div>;
}
```

The quote is rendered in a [quotelist parent component](https://github.com/krawaller/reduxfirebasedemo/blob/gh-pages/src/pages/quoteslist.js). This parent component is connected to Redux using [React-Redux](https://github.com/rackt/react-redux):

```javascript
var mapStateToProps = function(appState){
	return {
		quotes: appState.quotes,
		auth: appState.auth
	};
};

var mapDispatchToProps = function(dispatch){
	return {
		submitNewQuote: function(content){ dispatch(actions.submitNewQuote(content)); },
		startEdit: function(qid){ dispatch(actions.startQuoteEdit(qid)); },
		cancelEdit: function(qid){ dispatch(actions.cancelQuoteEdit(qid)); },
		submitEdit: function(qid,content){ dispatch(actions.submitQuoteEdit(qid,content)); },
		deleteQuote: function(qid){ dispatch(actions.deleteQuote(qid)); }
	}
};

module.exports = ReactRedux.connect(mapStateToProps,mapDispatchToProps)(Quoteslist);
```

And it renders each quote like thus, passing down the relevant state and callbacks:

```javascript
var p = this.props, rows = _.map(p.quotes.data,function(quote,qid){
	var quotestate = p.quotes.states[qid];
	return <Quote
		key={qid}
		quote={quote}
		qid={qid}
		state={quotestate}
		edit={p.startEdit.bind(this,qid)}
		cancel={p.cancelEdit.bind(this,qid)}
		submit={p.submitEdit.bind(this,qid)}
		delete={p.deleteQuote.bind(this,qid)}
		mayedit={p.auth.uid === quote.uid}
	/>;
}).reverse();
```

Note how it checks if you're allowed to edit this quote by comparing `quote.uid` with the `uid` of the currently logged in user.

### Feedback

For completion's sake let's also look at the feedback functionality, which is much simpler and doesn't include Firebase. The feedback state is just an array of string messages to show:

```javascript
feedback: [
	{msg:"Welcome to this little demo! It is meant to demonstrate three things:",error:false},
	{msg:"1) How to use Redux + Firebase",error:false},
	{msg:"2) How to use authentication in a Redux app",error:false},
	{msg:"3) How to have all UI state in Redux and none in the components",error:false}
]
```

The [feedback reducer](https://github.com/krawaller/reduxfirebasedemo/blob/gh-pages/src/store/reducers/feedback.js) deals with 3 actions:

```javascript
module.exports = function(currentfeedback,action){
	switch(action.type){
		case C.DISMISS_FEEDBACK:
			return currentfeedback.filter((i,n)=>n!==action.num);
		case C.DISPLAY_ERROR:
			return currentfeedback.concat({msg:action.error,error:true});
		case C.DISPLAY_MESSAGE:
			return currentfeedback.concat({msg:action.message,error:false});
		default: return currentfeedback || initialState.feedback;
	}
};
```

And the related [feedback action creator](https://github.com/krawaller/reduxfirebasedemo/blob/gh-pages/src/actions/feedback.js) is only the feedback dismissal is the result of a user interaction:

```javascript
module.exports = {
	dismissFeedback: function(num){
		return {type:C.DISMISS_FEEDBACK,num:num};
	}
};
```

### Wrapping up

Perhaps this demo took on more than it can chew, but I hope it at least can serve as an example of how to pair Redux and Firebase in a sane way.