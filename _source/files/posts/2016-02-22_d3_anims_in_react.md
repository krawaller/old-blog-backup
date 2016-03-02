---
type: post
title: "Using D3.js with animations in React"
date: 2016-03-01
tags: [react,D3]
author: David
excerpt: Detailing a solution to use D3.js code with animations in React
draft: false
---

### The problem

The other week I was tasked with putting some [D3](https://d3js.org/) graphics into a React app. This is not a good fit - D3 works by mutating the DOM, which React wants ownership of. How to deal with that?

It was easy to find examples of ready-made React D3 components, but they didn't sit right with me - all they tend to do was to put the D3 code in a thin React wrapper and pretend the problem doesn't exist. As long we don't do anything else in React space then things will kind of work, but don't try to put this into a regular SPA app lest you enjoy explosions.


### The solution

Then I read [Oliver Caldwell's post](http://oli.me.uk/2015/09/09/d3-within-react-the-right-way/) on the matter. A heartily recommended read if you haven't already caught it!

In the post he presents [React-faux-dom](https://github.com/Olical/react-faux-dom), a library that works like a light-weight [jsdom](https://github.com/tmpvar/jsdom). The faux nodes it creates have a `.toReact` method which turns them into virtual DOM.

Oliver's idea in using this as a solution to the DOM ownership problem is rather clever:

1. Create a faux element

  ```javascript
  var faux = react-faux-dom.createElement("div");
  ```

2. Feed that element to a library which works on a DOM node, such as d3.js

  ```javascript
  var svg = d3.select(faux).append("svg")
  ```

3. Work with the library as you normally would, mutating the fake node

  ```javascript
  svg.attr("width", width + margin.left + margin.right)
     .attr("height", height + margin.top + margin.bottom)
     .append("g")
     .attr("transform", "translate(" + margin.left + "," + margin.top + ")");
     // etc, normal d3.js code!
  ```

4. Use the `.toReact` method to inject the fake node into virtual DOM output

  ```
  var Chart = React.createClass({
    render: function(){
      // ... D3 code trunkated ...
      return (<div>
        <h2>Chart</h2>
          {fauxnode.toReact()}
        </div>)
      }
  });
  ```

This is a very powerful technique as it allows us to keep working with our existing node-mutating tools, while still being able to easily consume those in a React app.


### The solution problem

However there's a big shortcoming to this approach - **animations won't work!** This is especially a shame with regards to D3 as animations in diagrams can be especially catching. 

For me not having animations wasn't an option, as the whole point of my task was to add some whizzbang to a demo. I had fallen in love with D3 creator [Mike Bostock's](https://bost.ocks.org/mike/) beautiful [Stacked-to-group Bar chart](https://bl.ocks.org/mbostock/3943967), and now wanted to somehow make this work with Oliver's solution, while still being able to preserve as much as possible of Mike's code.


### The solution problem solution

First off - here's the final result! The chart in the iframe below (standalone [here](../../applets/react-d3-anim/)) is rendered in React, the animations are done in JSX space, and only tiny tweaks to the D3 code was needed.

<iframe src="../../applets/react-d3-anim/" height="450px" width="100%"></iframe>

You can read the source code [here](https://github.com/krawaller/d3animatedchartinreact/blob/gh-pages/index.js), but here's a walkthrough of the general idea.

I've created a tiny **`createHook(component,fauxelement,statename)`** function which takes three arguments:

1. A reference to the **React component** housing the d3 stuff
2. The **faux element** created with `react-faux-dom` that we'll feed to d3
3. Which **state propname** we want the resulting virtual DOM to end up in

The function returns a hook which you're supposed to butt to the end of every d3 `.transition` definition like this:

```javascript
// at the beginning of the d3 code, housed in `componentDidMount`
let hook = createHook(this,faux,"chart")

// further down:
rect.transition()
    .duration(500)
    .delay(function(d, i) { return i * 10; })
    .attr("y", function(d) { return y(d.y0 + d.y); })
    .attr("height", function(d) { return y(d.y0) - y(d.y0 + d.y); })
    .call(hook);
```

Note the last line where we do `.call(hook)`.

The hook will make sure that the following call is made once per 16 ms for as long as something is animating (as well as once initially to set things up):

```javascript
component.setState({[statename]:fauxelement.toReact()})
```

The net result is that we have a **virtual DOM representation of the chart inside component state**, and this representation will **update when the chart animates**.


Here's how the above component housing Mike's pretty chart works:

1. We provide an **initial state** object with `chart` set to null and the default look as `stacked`.

  ```javascript
  getInitialState: function(){
    return { chart: null, look: 'stacked' }
  },
  ```

2. The **render function** merely outputs a look toggler button and the chart virtual DOM:

  ```
  render: function(){
    return <div>
      <button onClick={this.toggleLook}>toggle layout</button>
      {this.state.chart}
    </div>
  }
  ```

3. Inside **`componentDidMount`** I've pasted Mike's code. The only changes I did were the following:

  1. Feeding D3 a `faux` element as per Oliver's approach
  2. Creating a `hook` and attaching it to each `.transition` call as detailed above
  3. Attaching his radiobutton look toggler callbacks (`transitionStacked` and `transitionGrouped`) to the component
  4. Removing the radio buttons themselves, as well as the initial automatic switch after 3 seconds

4. In the **`toggleLook`** component method I call one of the two look togglers:

  ```javascript
  toggleLook: function(){
    if (!this.isAnimating()){
      if (this.state.look === 'stacked'){
        this.setState({ look: 'grouped' })
        this.transitionGrouped();
      } else {
        this.setState({ look: 'stacked' })
        this.transitionStacked();
      }
    }
  },
  ```

  Note how toggling is wrapped in a `this.isAnimating()` check - that method is attached to the component by the `createHook` call. I couldn't (yet) get spamming the toggle button to clean up correctly, and this was a quick way around that.

And that's it, that's the entire component!

### Wrapping up

I rather like how `createHook` serves as a(n almost) standalone solution to the animation problem, and look forward to solidifying it and putting it to work. If you try this approach out, please let me know how you fare!



