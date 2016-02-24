---
type: post
title: "Using d3.js with animations in React"
date: 2016-02-22
tags: [react,d3]
author: David
excerpt: Detailing a solution to use d3.js code and animations in React
draft: true
---

[Mike Bostock](https://bost.ocks.org/mike/), creator of [D3](https://d3js.org/), made a beautiful [Stacked-to-group Bar chart](https://bl.ocks.org/mbostock/3943967).


D3 in React? There has been many attempts. None sat right with me until I read [Oliver Caldwell's post](http://oli.me.uk/2015/09/09/d3-within-react-the-right-way/) on the matter. There he presented [React-faux-dom](https://github.com/Olical/react-faux-dom), a library that works like a light-weight [jsdom](https://github.com/tmpvar/jsdom) but with a `.toReact` method which turns the node into a virtual DOM node.

The idea behind this is rather clever:

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

4. Use the `.toReact` method to inject the fake node into jsx output

```
var Chart = React.createClass({
	// before this we've stored the faux node onto the component instance
    render: function(){
        return (<div>
        	<h2>Chart</h2>
        	{this.fauxnode.toReact()}
        </div>)
    }
});
```

This is a very powerful technique as it allows us to keep working with our existing node-mutating tools, while still being able to easily consume those in a React app.