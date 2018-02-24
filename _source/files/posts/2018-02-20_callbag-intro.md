---
type: post
title: "Callbags introduction"
date: 2018-02-20
tags: [callbags]
author: David
excerpt: Wrapping our brains around the Callbags specification for streams represented by functions
---

### The premise

In this post we'll explore **Callbags** - a new spec by André Staltz for streams. You're assumed to be familiar with streams in general. If you're not, check out André's excellent [guide to reactive programming](https://gist.github.com/staltz/868e7e9bc2a7b8c1f754).

After familiarising ourselves with callbags in this post, we'll use our newfound knowledge to [explore a Callbags TodoMVC example](../dissecting-a-callbag-todomvc-implementation) in the next post.


### Callbags

As mentioned above, **Callbags** are a spec for streams. It is brilliant in it's simplicity, and allows for streams to be represented by just a simple function!

A function is a callbag if it has the signature `(t: 0 | 1 | 2, d: any) => void` and behaves thusly when called:

```javascript
callbag(0, talkback); // If this is the initiating call: Hey, I am a listener
                      // and I want data from you! Please call the talkback (which is
                      // a callbag representing me) with an acknowledgement and any
                      // subsequent communication.

callbag(0, talkback); // If it this is a reply: Hello, I am a source, I have
                      // registered your subscription! I will now start pushing data
                      // to you if I am a listenable source, or I will await requests 
                      // from you if I am a pullable source. Talk to the talkback
                      // if you want to terminate our relationship or request pullable
                      // data.

callbag(1);           // I assume you are a pullable source! Please send data!

callbag(1, stuff);    // Here you go listener, here is some data!

callbag(2);           // I don't want to be friends with you anymore! Please don't
                      // talk to me again! Also I won't talk to you again ever!

callbag(2, stuff);    // I know you wanted data but something wen't wrong! I won't
                      // send you more data ever!
```

In other words, the first `t` parameter has the following meanings:

* `0`: creating a relationship.
* `1`: requesting information (if `d` is undefined) or sending information
* `2`: terminating the relationship

The initial exchanging of `(0, talkback)` messages is called a *handshake*. We'll never send `1` or `2` before having performed a successful handshake.

In other words, we'll send `0` to a callbag, but `1` and `2` are only sent to a talkback that was provided via a handshake (but technically the talkback is a callbag too).

Another observation: a *source* is simply a callbag who will shake your hand back if you call it with `(0, talkback)`. A *sink* is a callbag who takes the initiative to a handshake.


### Pushing and pulling

In the callbag world there are two different kinds of sources:

* A source is *listenable* if it *pushes* data to the source. The sink just needs to subscribe, and after that the data is sent down along the wire whenever the source deems appropriate.
* It is *pullable* if the source has to *pull* each piece of data from it. Data is only ever sent as a response to a date request.

Being able to represent both push and pull with the same spec is one of the main strengths of callbags. Most stream libraries can only do one or the other - for instance, [RxJS](https://github.com/reactivex/rxjs) is all push ("reactive programming"), while the evil twin [IxJS](https://github.com/ReactiveX/IxJS) is all pull ("iterative programming").

How did André manage this? Through the realisation that **a pull is simply two pushes**:

1. the sink pushes a request message to the pullable (`pullableTalkback(1)`)
2. the pullable pushes the response back (`sinkTalkback(1, data)`)

In other words, you'd never call a listenable source with `(1)` - there's no need, it will send data to you whenever it sees fit.

### Sources

As a first look at a callbag example, here's the source code for Staltz' [callbag-interval](https://github.com/staltz/callbag-interval) which creates a source that emits at an interval:

```javascript
const interval = period => (start, sink) => {
  if (start !== 0) return;
  let i = 0;
  const id = setInterval(() => {
    sink(1, i++);
  }, period);
  sink(0, t => { // <--- here we send the talkback to the sink
    if (t === 2) clearInterval(id);
  });
};
```

That was an example of a listenable source, since it pushes messages to the listener without it having to request it.

Why was the signature of the callbag `(start, sink)` and not `(t, d)` in the code? To signify that it only ever expects to be called with `0`. Subsequent calls will be made to the returned talkback.

Why is the signature of the talkback `(t)` and not `(t,d)`? Because it only ever expects to be called with `(2)`. Also this is a listenable source so there's no need to send `(1)` data requests, so the parameter could really have been named `end` instead of `t`.

An example of the opposite, a pullable source, would be Staltz' [callbag-from-iter](https://github.com/staltz/callbag-from-iter) It creates a source from an array (or other iterable) and then emits the next item in the array whenever asked to. Here's an abbreviated version of the code:

```javascript
const fromIter = iter => (start, sink) => {
  if (start !== 0) return;
  /* prepare iterable here */
  sink(0, t => {
    if (t === 1) {
      /* send next item in iterable (unless terminated or pending request) */
    }
  });
};
```

Note how it only sends data when the sink calls the talkback with `(1)`.

### Sinks

Let's also look at Staltz' [callbag-for-each](https://github.com/staltz/callbag-for-each) for an example of a basic *sink*:

```javascript
const forEach = operation => source => {
  let talkback;
  source(0, (t, d) => {
    if (t === 0) talkback = d;
    if (t === 1) operation(d);
    if (t === 1 || t === 0) talkback(1);
  });
};
```

The `forEach` sink performs the given operation on every emission on the connected source. We could use it on the interval source like so:

```javascript
forEach(v => console.log(v))(interval(100)); // 0
                                             // 1
                                             // 2
```

To more clearly see the flow of stream juggling it is common to use Staltz' [callbag-pipe](https://github.com/staltz/callbag-pipe) utility function. Using that we can rephrase the above to this:

```javascript
pipe(
  interval(100),
  forEach(v => console.log(v)) // 0
);                             // 1
                               // 2
```

The `forEach` sink is clever in that it works with both `listenable` and `pullable` sources. Notice this line in the sink talkback in the code:

```
if (t === 1 || t === 0) talkback(1);
```

This calls the source talkback with a data request after initiation and after each received data. If the source is pullable, this means it will send the next data. If it isn't the request is simply ignored.

### Callbag operators

An operator is simply a function that takes an argument (or more), returns a function that takes a callbag source which in turn returns a transformed source.

```typescript
(...args: Array<any>) => (source: Callbag) => Callbag;
```

As an example, here's Staltz' [callbag-map](https://github.com/staltz/callbag-map):

```javascript
const map = f => source => (start, sink) => {
  if (start !== 0) return;
  source(0, (t, d) => {
    sink(t, t === 1 ? f(d) : d)
  });
};
``` 

This returns a new source that passes each emitted data through the mapping function `f`.

We could use that on the previously shown `callbag-interval` like so:

```javascript
pipe(
  interval(100),
  map(v => 2*v),
  forEach(v => console.log(v)) // 0
);                             // 2
                               // 4
```

In other words Callbag code (much like any stream code) is often a chain of sources passing through operators with a sink at the end:

![](../../diagrams/callbag-chain.svg)

### Wrapping up

Here's the real kicker regarding callbags - there's *no core library*. Working with callbags means working with a bunch of different functions that all follow the spec.

In my mind the spec that Staltz has created is a remarkable feat. Much like Crockford likes to say he didn't *invent* JSON as much as *discover* it, I feel Staltz can say the exact same thing about callbags. 

There is already a [significant number of callbag packages](https://npms.io/search?q=keywords%3Acallbag) out there. And if something you need isn't there, it is super easy to roll your own.

For more in-depth detail about callbags check out the following links:

* Staltz' [blog post introducing callbags](https://staltz.com/why-we-need-callbags.html)
* The [callbag spec](https://github.com/callbag/callbag)
* The [getting started guide](https://github.com/callbag/callbag/blob/master/getting-started.md) hidden in the spec

Also, for a more practical exercise, check out the [next post](../dissecting-a-callbag-todomvc-implementation) where we dissect a TodoMVC app implemented with callbags!
