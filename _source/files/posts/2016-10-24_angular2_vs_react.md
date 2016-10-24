---
type: post
title: "Angular2 versus React, comparing components"
date: 2016-10-24
tags: [react,angular2]
author: David
excerpt: Some musings around the comparison, and my part in an upcoming live debate 
---

### Solid straw man

If you're a regular reader of this blog, you'll be surprised to know that my picture is to the *left* on the flyer below:

![](../../img/angular2_vs_react.jpg)

In other words, in this debate, I will be arguing on behalf of Angular2 against React! Have I been swayed by the newest kid on the block?

Naaah, not really. My colleague and opponent simply called dibs on React before I had the chance. And fair enough, as I have spent the last few months building an [Angular2 course](https://edument.se/education/categories/webdevelopment/angular-2) and could be expected to come up with a few retorts. 

But even though I still prefer React and will secretly wear a React t-shirt under the Angular2 t-shirt during the debate, I do recognise that there is an argument to be made for Angular2! In this post we'll take a quick look at two of them.


### View and model split in Angular2

Most of what I like about Angular2 is centered around the component model. First off, I like the fact that it *has* a component model - and a composable one to boot - instead of the really weird MVC bastardisation Angular1 was pushing.

I enjoy how Angular2 naturally splits the component definition between view and model:

```typescript
@Component({
  selector: 'clicker',
  template: `
    <p>{{count}} bottles of beer on the wall</p>
    <button (click)="more()">Buy more</button>
  `
})
export class Clicker {
  count = 3
  more() {
    this.count++
  }
}
```

The view-related definitions are in the decorator, while the class encapsulates all model-related stuff. Elegant!

### View and model split in React

Compare this to the React code for the same component:

```typescript
let Clicker = React.createClass({
  getInitialState: () => ({count: 3}),
  more () {
    this.setState({count: this.state.count + 1})
  },
  render () {
    return <div>
      <p>{this.state.count} bottles of beer on the wall</p>
      <button onClick={this.more}>Buy more</button>
    </div>
  }
})
```

You could argue that the view stuff goes into the `render` method and the model is the rest, but that doesn't always hold true. 

If we rephrase it using an ES6 class instead...

```typescript
class Clicker extends React.Component {
  constructor () {
    super()
    this.state = {count: 3}
    this.more = () => this.setState({count: this.state.count + 1})
  }
  render () {
    return <div>
      <p>{this.state.count} bottles of beer on the wall</p>
      <button onClick={this.more}>Buy more</button>
    </div>
  }
}
```

...you could argue that the model is defined in the `constructor`, but like before that isn't necessarily always true. Also it seems a bit unintuitive.


### Component interface in Angular2

The above is a minor point, but here is one I feel more strongly for; Angular2 components have a really clean interface.

What do I mean? I'd argue there are three main ways in which we interface with a component:

* we **input** data
* we provide **dependencies**
* we catch **output**

In Angular2, these are immediately identifiable:

```typescript
@Component({ ... })
export class SceneComponent {
  constructor(private someDependency){ ... }
  @Input() someInput
  @Output() someOutput = new EventEmitter<SomeType>()
  // ...
}
```
Dependencies are constructor arguments, and inputs and outputs are literally marked as such.


It is equally clear when we use a component - inputs are in brackets and outputs in parens:

```html
<MyComponent [someInput]="someVar" (someOutput)="handler($event)">
```

And the dependencies are handled by providers defined in the `NgModule` container.


### Component interface in React

Contrast this to React, where we simply throw everything into props. Like an *animal*!

```
let {someInput, someOutput, someDependency} = this.props;
```

It is equally opaque when we use a component:

```html
<MyComponent someInput={someVar} someOutput={handler} someDependency={someDep}>
```

Things become somewhat cleaner if we use `context` to handle dependencies instead of passing them in from the parent, something I only recently began experimenting with, but that's not something you see in common usage.


### Further comparisons

For more small scoped comparisons looking at single aspects of frameworks, take a sneak peek at our WIP [JavaScript framework comparison site](http://blog.krawaller.se/jscomp). Sort of TodoMVC, only on a much smaller and hopefully more approachable scale.

### Wrapping up

Anyway, as stated initially, I still much prefer React over Angular2. Writing pseudo-JavaScript in strings? Like an *animal*? No thank you.

However, learning Angular2 has definitely helped widen my perspective on JavaScript frameworks in general, and there are enough neat ideas in there to warrant exploration. If you haven't tried it, I recommend you do so!

And, of course, if you happen to be in Malmö, Sweden on november 8th - likely attending the [Øredev developer conference](http://oredev.org/) which begins the next day - do come along to [our debate event](https://edument.se/news/ng2-vs-react), say hello and root for your favourite! 

