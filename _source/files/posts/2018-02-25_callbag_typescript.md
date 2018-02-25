---
type: post
title: "Explaining callbags via TypeScript definitions"
date: 2018-02-25
tags: [callbags, typescript]
author: David
excerpt: Understanding callbags by splitting them into types with individual TypeScript definitions
---

### Premise

I've already written an [introduction to callbags](../callbags-introduction/), but I wasn't completely happy - it still feels like the initial threshold to grokk what's going on is rather high. Callbags are a rather simple concept, but communicating the nuances is surprisingly difficult.

In this post I make a second attempt, this time via splitting up the lifecycle and definining precise TypeScript interfaces for the various parts. You are assumed to have read the introductory post and/or be familiar with the [callbags spec](https://github.com/callbag/callbag). 

### The official callbag typing

The spec defines a callbag as...

> a function of signature (TypeScript syntax:)<br/> `(type: 0 | 1 | 2, payload?: any) => void`

This is further reflected in the [official `types.d.ts` file](https://github.com/callbag/callbag/blob/master/types.d.ts) which looks like this (slightly abbreviated):

```
export type START = 0;
export type DATA = 1;
export type END = 2;

export type Callbag = (type: START | DATA | END, payload?: any) => void;
export type Factory = (...args: Array<any>) => Callbag;
export type Operator = (...args: Array<any>) => (source: Callbag) => Callbag;
```

The trouble is that `START`, `DATA` and `END` are *not always* valid as the first parameter to a callbag. Exactly what we can send depends on where we are in the life cycle.

Or, phrased differently: it depends on *from where we got the callbag we're calling*. This can be surmised from the written text of the spec, but it isn't reflected in the typings.

### The holy trinity of callbags

Things become more clear when you realise that there are actually **three different kinds of callbags**:

* **Source**: The initial callbag function that a sink can subscribe to
* **Sink talkback** The callbag sent to the source by the sink when subscribing
* **Source talkback** The callbag sent in response to the sink talkback by the source talkback

These three kinds of callbags expect to be called in different ways. In other words, they have different signatures! Here's a lowdown in table form on how they can be called:

<style>
.YEP { color: green; font-weight: bold; }
.NOPE { color: red; font-weight: bold; }
.sup {vertical-align: baseline; position: relative; font-size: 70%; top: -0.6em;}
.note {font-size: 0.8em;}
</style>

| call | source | source talkback | sink talkback |
| --- | --- | --- | --- |
| `(0, callbag)` | <span class="YEP">yes</span> | <span class="NOPE">no</span> | <span class="YEP">yes</span> |
| `(1)` | <span class="NOPE">no</span> | <span class="YEP">yes</span><span class="sup">1</span> | <span class="NOPE">no</span> |
| `(1, data)` | <span class="NOPE">no</span> | <span class="NOPE">no</span> | <span class="YEP">yes</span> |
| `(2)` | <span class="NOPE">no</span> | <span class="YEP">yes</span> | <span class="YEP">yes</span> |
| `(2, error)` | <span class="NOPE">no</span> | <span class="YEP">yes</span><span class="sup">2</span> | <span class="YEP">yes</span> |

<span class="note">
<span class="sup">1</span> If the source is pullable <br/>
<span class="sup">2</span> A source likely doesn't care why the sink terminates, so you could argue for a "no" here
</span>

And, applying the notion of "three different kinds of callbags" consistently, we should split the first row in the table...

| call | source | source talkback | sink talkback |
| --- | --- | --- | --- |
| `(0, callbag)` | <span class="YEP">yes</span> | <span class="NOPE">no</span> | <span class="YEP">yes</span> |

...to this:

| call | source | source talkback | sink talkback |
| --- | --- | --- | --- |
| `(0, sinkTalkback)` | <span class="YEP">yes</span> | <span class="NOPE">no</span> | <span class="NOPE">no</span> |
| `(0, sourceTalkback)` | <span class="NOPE">no</span> | <span class="NOPE">no</span> | <span class="YEP">yes</span> |

Let's take our three different types one by one and see if our observations match some real-world code. And, if they do, create individual type definitions for them!

### Sources

According to our table, a *source* callbag only ever expects to be called with `(0, sinkTalkback)`. This is reflected in the code for many source implementations in that the first parameter is called `start`, not `t`.

For example, look at Staltz' [callbag-interval](https://github.com/staltz/callbag-interval) source:

```javascript
const interval = period => (start, sink) => {
  if (start !== 0) return;
  let i = 0;
  const id = setInterval(() => {
    sink(1, i++);
  }, period);
  sink(0, t => {
    if (t === 2) clearInterval(id);
  });
};
```

The signature in the code is called `(start, sink)`, and if `start` doesn't equal `0` we bail because nothing else makes sense at this point. This function will never receive data requests `(1)` or terminations `(2)` - those will be sent to a source talkback.

In other words, a TypeScript definition for a `Source` could be:

```typescript
type Source = (start: START, sinkTalkback: SinkTalkback) => void;
```

### Source talkbacks

In our table we noted `(1)` and `(2)` (and maybe `(2, error)`) as valid calls to the *source talkback*. Peeking back at the code for `interval`, the source talkback is implemented as:

```javascript
t => {
  if (t === 2) clearInterval(id);
}
```

It only ever handles terminations `(2)`. This makes sense since `interval` isn't pullable, so it doesn't care about data requests `(1)`.

To take a pullable example, here's Staltz' [callbag-from-pull-stream](https://github.com/staltz/callbag-from-pull-stream):

```javascript
const fromPullStream = read => (start, sink) => {
  if (start !== 0) return;
  sink(0, (t, d) => { // <--- source talkback
    if (t === 1) read(null, (end, data) => {
      if (end === true) sink(2);
      else if (end) sink(2, end);
      else sink(1, data);
    });
    if (t === 2) read(d || true, () => {});
  });
};
```

Here we can see the source talkback definition dealing with both `(1)` and `(2)`.

Let's now tie up our observations about the source talkback into a TypeScript definition:

```javascript
type SourceTalkback = (type: DATA | END, payload: any) => void;
```

If we accept the notion that a source is never interested in why a sink terminates, we could skip the second parameter since it is only used for `(2, error)`:

```javascript
type SourceTalkback = (type: DATA | END) => void;
```

We could also potentially split this into separate interfaces for pullable and listenable sources:

```javascript
type PullableSourceTalkback = (type: DATA | END) => void;
type ListenableSourceTalkback = (type: END) => void;
```

And then define a `SourceTalkback` as being one or the other:

```javascript
type SourceTalkback = PullableSourceTalkback | ListenableSourceTalkback;
```

### Sink talkbacks

Now for the third and final type - sink talkbacks! In our table we had them listed as accepting all three types `START`, `DATA` and `END`. This makes sense since they...

* must be greeted back by the source with a source talkback, which means `START`
* will receive data from the source, which means `DATA`
* can be terminated by the source, which means `END`

Here is Staltz' [callbag-for-each](https://github.com/staltz/callbag-for-each):

```javascript
const forEach = operation => source => {
  let talkback;
  source(0, (t, d) => { // <--- sink talkback
    if (t === 0) talkback = d;
    if (t === 1) operation(d);
    if (t === 1 || t === 0) talkback(1);
  });
};
```

It doesn't handle `(2)`, but it doesn't need to - if there's no more data coming in, nothing more will happen. There's nothing going on downstream from `forEach` and nothing to clean up, so we don't need to act.

To see termination handling we must look at something that needs to clean up. Or something that isn't at the end of the chain - in other words, a sink used by an operator to make a new source! As an example, here is my [callbag-latest](https://github.com/krawaller/callbag-latest) which turns a listenable source into a pullable one:

```javascript
const latest = listenable => (start, sink) =>  {
  if (start !== 0) return;
  let ltalkback;
  let latestValue;
  let hasLatestValue = false;
  listenable(0, (lt, ld) => { // <--- sink talkback
    if (lt === 0) {
      ltalkback = ld;
      sink(0, (st, sd) => {
        if (st === 1 && hasLatestValue) sink(1, latestValue);
        if (st === 2) ltalkback(2, sd);
      });
    }
    if (lt === 1) {
      latestValue = ld;
      hasLatestValue = true;
    }
    if (lt === 2) sink(2, ld);
  });
};
```

Here we can clearly see the sink talkback handling all three cases including terminations `(2)` - although, as is often the case, the only thing it does with the termination is to pass it down the chain.

Since sink talkbacks actually handle all three cases, they are the first kind of callbag that matches the official callbag typing!

```typescript
type SinkTalkback = (type: START | DATA | END, payload?: any) => void;
```

We can clarify the second parameter by splitting this into three different types and joining them via an [intersection type](https://www.typescriptlang.org/docs/handbook/advanced-types.html) using `&`:

```typescript
type SinkTalkback = 
  & ((start: START, sourceTalkback: SourceTalkback) => void)
  & ((deliver: DATA, data: any) => void)
  & ((terminate: END, error?: any) => void);
```

The extra parenthesises around the three function types are needed for TypeScript not to throw a fit.

This gives us rather nice intellisense help in a TypeScript setting:

![](../../gifs/callbag-typing.gif)

In fact, that's so nice that we should split up `SourceTalkback` in the same way so that it too gets separate parameter names per call type:

```typescript
type SourceTalkback = 
  & ((request: DATA) => void)
  & ((terminate: END) => void);
```

### Source VS source talkback

Our tour of the three different kinds of callbags - source, source talkback and sink talkback - is now concluded!

But let's zoom out and discuss the difference between *source* and *source talkback* for a bit. Our TypeScript definition for `Source` was:

```typescript
type Source = (start: START, sinkTalkback: SinkTalkback) => void;
```

...and `Source talkback`:

```typescript
type SourceTalkback = 
  & ((request: DATA) => void)
  & ((terminate: END) => void);
```

But what exactly does the spec say about **source**? Well:

> a callbag which is expected to deliver data

Hmm. Who is it that delivers data - is it the `Source` or the `SourceTalkback`? Well, that actually depends!

In a *pullable source* it is the *source talkback*. Check again at the source for `fromPullStream`:

```javascript
const fromPullStream = read => (start, sink) => {
  if (start !== 0) return;
  sink(0, (t, d) => { // <--- source talkback
    if (t === 1) read(null, (end, data) => {
      if (end === true) sink(2);
      else if (end) sink(2, end);
      else sink(1, data); // <--- delivery call
    });
    if (t === 2) read(d || true, () => {});
  });
};
```

The `sink(1, data)` call takes place inside the source talkback. But if we look at a listenable source like `interval`...

```javascript
const interval = period => (start, sink) => {
  if (start !== 0) return;
  let i = 0;
  const id = setInterval(() => {
    sink(1, i++); // <---- delivery call
  }, period);
  sink(0, t => { // <---- source talkback
    if (t === 2) clearInterval(id);
  });
};
```

...you'll note the `sink(1, data)` call is taking place inside the *source*.

This means that the word **source** as defined in the spec encompasses *both* `Source` and `SourceTalkback`. The concept includes both typings. Or you could argue that `SourceTalkback` is an implementation detail of `Source`.

Either way, using the name `Source` for a type seems like needless confusion potential. To mitigate that, let's rename what we've defined as `Source` and call it `SourceInitiator` instead:

```typescript
type SourceInitiator = (start: START, sinkTalkback: SinkTalkback) => void;
```

Now the words make more sense - the definition of *source* in the spec includes both concepts of `SourceInitiator` and `SourceTalkback`.


### Sink VS sink talkback

The obvious next question - what, then, is the difference between a sink and a sink talkback? What even *is* a sink?

The spec simply says...

> a callbag which is expected to be delivered data

...which clearly fits the sink talkback. That will always be what receives the data. However, that doesn't answer our question of what a sink actually is.

But if a `SourceInitiator` is what provides a `SourceTalkback`, an interesting thing might be to look at what provides the `SinkTalkback`! Let's look again at `forEach`:

```javascript
const forEach = operation => source => {
  let talkback;
  source(0, (t, d) => { // <--- sink talkback
    if (t === 0) talkback = d;
    if (t === 1) operation(d);
    if (t === 1 || t === 0) talkback(1);
  });
};
```

The `sourceInitiator(0, sinkTalkback)` call takes place inside a function that takes a source parameter and returns nothing. Is perhaps that a sink? Or, by the previous logic, a sink initiator?

```typescript
type SinkInitiator = (source: SourceInitiator) => void;
```

I'd say yes, that is a sink (initiator)!

Except this time the typescript definition isn't as helpful, since it doesn't encode the fact that to be a `SinkInitiator` you must also *call the passed source* in a specific way. It is kind of implied through the type of the passed in `SourceInitiator`, but, still.

Perhaps we can make the implication stronger still by calling it `SinkConnector` instead of `SinkInitiator`? Yeah, let's do that!

```javascript
type SinkConnector = (source: SourceInitiator) => void;
```

However, we can note that every single usage of the word "sink" in the spec fits what we've labeled as a `SinkTalkback`. In other words, we could have called that a `Sink`. But it seems nice to be constistent in suffixing the type for a callbag that is passed in to another callbag with `-Talkback`.

### Operators and sinks

In the official typings, *operators* are defined as:

```typescript
type Operator = (...args: Array<any>) => (source: Callbag) => Callbag;
```

We can replace `Callbag` with what we've called `SourceInitiator`:

```typescript
type Operator = (...args: Array<any>) => (source: SourceInitiator) => SourceInitiator;
```

So what's returned from calling the operators with some arguments is *almost* a sink connector, except it returns `SourceInitiator` instead of `void`!

```typescript
type SinkConnector = (source: SourceInitiator) => void;
type ReturnedFromOp = (source: SourceInitiator) => SourceInitiator;
```

Do we want to be able to say that an operator returns a sink connector? Or should we invent a new concept for this?

I'm voting for the former - it makes sense to say that an operator returns a sink connector. We can do that by merging the two types above thusly:

```typescript
type SinkConnector = (source: SourceInitiator) => SourceInitiator | void;
```

This has four benefits:

* no need for a separate type to describe what is returned by an operator
* we can type operators who *don't* return anything (in other words, an operator for a sink)
* we can type sinks who returns a source initiator, allowing for the chain to continue
* the chainability is now visible in the typings!

So our revised type for `Operator` is now:

```
type Operator = (...args: Array<any>) => SinkConnector
```

### Factories

The official typings also had a typing for `Factory`:

```typescript
type Factory = (...args: Array<any>) => Callbag;
```

We can now clarify that into:

```
type SourceFactory = (...args: Array<any>) => SourceInitiator;
```

What about a `SinkFactory`? It turns out we don't need one - a `SinkFactory` is simply an `Operator` that returns a `SinkConnector` that *doesn't* return a `SourceInitiator`!


### The new typings

Here are all our new typings collected:

```typescript

// Source
type SourceFactory = (...args: Array<any>) => SourceInitiator;

type SourceInitiator = (start: START, sinkTalkback: SinkTalkback) => void;

type SourceTalkback = 
  & ((request: DATA) => void)
  & ((terminate: END) => void);

// Sink
type SinkConnector = (source: SourceInitiator) => SourceInitiator | void;

type SinkTalkback = 
  & ((start: START, sourceTalkback: SourceTalkback) => void)
  & ((deliver: DATA, data: any) => void)
  & ((terminate: END, error?: any) => void);

// Other
type Operator = (...args: Array<any>) => SinkConnector;
```

However, if we want to avoid using types that haven't been defined yet, we can look at the dependency graph of the types...

![](../../diagrams/callbag-typedefs.svg)

...and reorder them backwards according to that:

```typescript
type SourceTalkback = 
  & ((request: DATA) => void)
  & ((terminate: END) => void);

type SinkTalkback = 
  & ((start: START, sourceTalkback: SourceTalkback) => void)
  & ((deliver: DATA, data: any) => void)
  & ((terminate: END, error?: any) => void);

type SourceInitiator = (type: START, payload: SinkTalkback) => void;

type SinkConnector = (source: SourceInitiator) => SourceInitiator | void;

type SourceFactory = (...args: Array<any>) => SourceInitiator;

type Operator = (...args: Array<any>) => SinkConnector;
```

### Reaping the rewards

With these typings down we can use them as a very precise vocabulary for explaining the process of using callbags!

Here's the setup and handshake:

1. We call a `SinkConnector` passing in a `SourceInitiator`: `sinkConnector(sourceInitiator)`
1. The `SourceInitiator` is called with a `SinkTalkback`: `sourceInitiator(0,sinkTalkback)`
1. The `SinkTalkback` is called with a `SourceTalkback`: `sinkTalkback(0,sourceTalkback)`

And data passing:

1. The `SourceTalkback` is called with a request for data (if pullable): `sourceTalkback(1)`
1. The `SinkTalkback` is called with data delivery: `sinkTalkback(1,data)`

And termination:

* The `SourceTalkback` can be called to terminate the relationship: `sourceTalkback(2)`
* The `SinkTalkback` can be called to terminate the relationship: `sinkTalkback(2)`
* The `SinkTalkback` can be called to terminate the relationship due to an error: `sinkTalkback(2,error)`

### Wrapping up

Creating the types definitely helped me cement my own understanding of callbags, but I'm not sure if they have a pedagogic power in general. I can only hope you found this instructive, otherwise you just made a very long journey for nothing. :)
