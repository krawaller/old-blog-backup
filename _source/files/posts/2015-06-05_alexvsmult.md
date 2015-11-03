---
title: Tool complexity (or how 4 days can be longer than 4 years)
author: David
tags: [react, algebra explorer, education]
date: 2015-06-05
excerpt: "Discussing the necessity of ease of use (and, of course, how great React is)"
type: post
---


###Algebra Explorer

Last year, after 4 years of hard labour, I finally released [Algebra Explorer](../algebra-explorer-a-symbolic-calculator-web-app/), a web app designed to help you learn Algebra. Since then I've tried to find a place for it in my own teaching, as well as spreading the gospel to teachers and students.

<p style='text-align:center;'>
![substeps](../../img/alexscreen.png)
</p>

But in spite of making it a comletely free web app, I've had a really hard time "selling" it. Two big reasons are presumably that...

*    I really don't enjoy marketing and thus haven't given it enough time
*    the UI isn't intuitive enough

However, I believe an even bigger reason to be the old truth about tool complexity.

###The Multiplication Table app

Recently, on a whim, I made [simple web app](http://blog.krawaller.se/multhelp/) to help my students practice the multiplication table.

![Multiplication table practice](../../img/mult.png)

Click a button with a multiplication, and it will rise to show you the answer. All other buttons with the same answer will rise too. Click a side button to raise all in that row/column. By using the top-left button you can also practice multiplications within a given range.

Super simple and rather limited. Yet this trivial app, built during commuting over 4 days, has already generated more attention and praise than Algebra Explorer has accumulated since its release!


###Tool complexity

The biggest reason for this disparity in reception is likely that the multiplication table practice app is so much easier to pick up. It is instantly obious what it is designed to do, and almost as obvious how it works. Algebra Explorer, on the other hand, is very opaque. 

We see the same pattern in programming world. Part of React's success is definitely due to how approachable it is, and how easy it is to use, when compared to Ember and Angular. This in spite of the undisputable fact that the latter two are far more powerful.

Speaking of React, it really was an excellent fit for the multiplication app! The code can be found [here](https://github.com/krawaller/multhelp) (although I should warn you that it is pretty crude). Even when tacking on the gamification practice part the code amounts to little more than button callbacks changing state in the parent component.

###The hard truth

So, does this mean that Algebra Explorer was a mistake? Should I have spent those 4 years making lots and lots of smaller web apps in the vein of the multiplication pracice app?

Damned if I know. Maybe?


