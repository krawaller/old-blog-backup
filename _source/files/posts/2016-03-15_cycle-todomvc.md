---
type: post
title: "A Cycle.js TodoMVC codebase walkthrough"
date: 2016-03-15
tags: [cycle,case study,todomvc]
author: David
excerpt: Picking apart the TodoMVC Cycle.js demo app
draft: true
---

### The premise

In this post we'll pick apart the CycleJS TodoMVC project.

These are the involved files:

<img src="../../img/cycle-mvc-project.png" style="width:200px;" />

The `js` folder just contains generated files.

The file dependency tree is a bit weird in that `components/TodoItem.js` acts as it was inside the `components/Todos` folder with the others.

![dependencies](../../img/cycle-mvc-filedep.png)