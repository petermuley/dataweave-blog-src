---
title: "Getting Started with Data-Weave"
date: "2020-05-24"
path: "/getting-started-with-dw"
author: "Michael"
excerpt: "Need to get working with data-weave as fast as possible? Let's talk about how you can hit the ground running, and where to go from there."
coverImage: "../images/getting-started-with-dw.png"
tags: ["dataweave", "mule 4"]
---

## Table of Contents
1. [Environment setup](#environment-setup)
   1. [Anypoint Studio](#anypoint-studio)
   2. [Data-weave playground](#data-weave-playground)
2. Syntax Basics
   1. [What the heck is the `---`?](#what-the-heck-is-the----)
   2. [Anonymous functions, lambdas, and their syntax sugar - `$`, `$$`..](#anonymous-functions-lambdas-and-their-syntax-sugar----)
   3. `({})` .. what magic is this?!?

---

# Environment Setup

### **Anypoint Studio**

Your go to IDE should always be Anypoint Studio - this is the only **official** way to work with Mule (outside of design center at least). No other environment is going to give you access to all of the features of data-weave, and no other environment is going to guarntee that your data-weave behaves as expected when moved into production.

The two most common arguments against studio that I hear are that it is either too slow / resource intensive, or that the experience isn't as good as the web based playground. To that end, I've laid out what I think are some 'best practices' for using studio.

**Use a separate workspace or project**. If you're looking for a place to quickly work on data-weave, and you find studio to be too slow / non-reponse, consider using a separate workspace just for your data-weave. When you have a multiple projects in studio, the background toolingk is constantly running in the background trying to validate / resolve metadata. The less you have open, the faster it's going to run! So if you're working on a complicated data-weave for a project, separating it out into a separate project and closing your main project can be beneficial.

Think the three panel experience the web playground gives you is better? Guess what, studio can do it too! Just throw a transform message component in your dw-testing project, double click the component properties tab, and turn on preview!

![](../images/studio-dw-editor.gif)

And there you go! Full screen code editing, you can set your sample data on the left (if you haven't set it yet, it will prompt you for a data type first), and all functionality of dataweave is supported, such as including libraries, custom flat file formats, etc.

### **Data-weave playground**

***The playground is NOT an official tool and is not supported by Mulesoft. The playground is a hobby project written by someone with a passion for data-weave, but you shouldn't be using it for anything other than quick snippets***

So I just got done telling you to do all of your data-weave development in studio, and now I'm talking about the web-based playground. What gives? Well, one good reason to use the playground is for testing in specific snapshots of data-weave - I personally keep the latest DW2.X version and the latest DW1.X version. I typically go to the playground when quickly writing up a snippet for someone on stackoverflow or to answer a question on slack - you just can't beat how quick it is to bring up and play with. What I'm not doing is writing data-weave intended for a project; I can't setup proper folder structures, unit testing, logging, etc. For a quick snippet it makes a lot of sense.

If you don't know what docker is, [start here](https://docker-curriculum.com/).

You can find the list of data-weave playground tags [here](https://hub.docker.com/r/machaval/dw-playground/tags).

---
To quickly launch the 2.3.1 playground for one-time usage:

`docker run --publish 9999:8080 machaval/dw-playground:2.3.1-SNAPSHOT`

And now you can access the playground at http://localhost:9999

---

To launch 2.3.1 as a named, detached playground which will start with your system:

`docker run --detach --publish 8080:9999 --name dw-playground-2.3.1 --restart unless-stopped machaval/dw-playground:2.3.1-SNAPSHOT`

Pick whichever snapshot you want to use from the tag list and get going with it. I'm not going to go any further into the playground as it is pretty straightforward - and pretty limited. If you run into issues with the playground, I suggest moving your wokr to the studio as there is no support for the playground.

# Syntax Basics

### **What the heck is the ---?**

```data-weave{4,7}
%dw 2.0
output application/json
var numbers = 1 to 30
---
numbers map do {
  var prevNumber = numbers[$$-1]
  ---
  $ + prevNumber
}
```

So what exactly do the `---` we see above mean? For a scoped expression, they represent the separation of a header, and body. The body of an expression ***must*** yield a value. That makes it rather difficult to do things like setting a variable... which is where the header comes in. If you need to set a variable, define a function, etc, you get it done in the header. The `do { }` create a new scope in which we can put another head/body, allowing us to set variables and make use of them in our expressions. Here is a real world example (with all the messy bits removed) showing why this is useful:

```data-weave{7,11}
%dw 2.0
output application/json

var doCalc(item, previousItem) ->
  //add some logic here where we do a loyalty adjustment using the previous item
  item
---
//our payload is a set of loyalty items
payload orderBy $.transactionDate map do {
  var calc = doCalc(item, payload[max([0, $$-1])])
  ---
  ($ - "customAttributes") ++ "customAttributes": {
    "adjustedQualifyingFills": calc.adjustedQualifyingFills,
    "adjustedEarnLevel": calc.adjustedEarnLevel
  }
}
```

As you can see, we are now able to create a scope in our main body, with its own head/body, allowing us to call a function to get a result and then make use of that in our map. Neat.

### **Anonymous functions, lambdas, and their syntax sugar - `$`, `$$`..**

In data-weave examples, you probably see things like this A LOT right?

```data-weave
%dw 2.0
output application/json

var items = 1 to 10
---
items map ($$): $
```

Which results in:

```JSON
[
 { "0": 1 }, { "1": 2 }, { "2": 3 }, { "3": 4 }, { "4": 5 },
 { "5": 6 }, { "6": 7 }, { "7": 8 }, { "8": 9 }, { "9": 10 }
]
```
Yeah.. ok.. but where exactly did the `$` and `$$` come from?

First, lets talk about infix notation vs prefix notation. We know that `map` is a function which takes the parameters of `Array<T>` and `Function`. You can just take a look at the documentation [here](https://docs.mulesoft.com/mule-runtime/4.3/dw-core-functions-map#map1) to see that. If you're coming from another language, you'd probaby expect to call it doing something like this:

```data-weave
%dw 2.0
output application/json

var items = 1 to 10
var myMapFunction = (item, index) ->
  (index): item
---
map(items, myMapFunction)
```

Well, this is valid!

This is known as prefix notation - our function followed by our inputs. If you've been doing software development for a while, prefix notation makes sense - its how we do things! For most people though, infix notation, which looks like `X + Y`, where our function/operator is between our inputs, makes a lot more sense. We say `X plus Y`, not `plus X, Y`. With dataweave, we can use infix notation for any function that takes exactly two parameters. Take [`orderBy`](https://docs.mulesoft.com/mule-runtime/4.3/dw-core-functions-orderby#orderby2) for example - it takes an `Array` and a `Function` as input; exactly two. This also allows us to chain our functions:

```data-weave
payload orderBy $.transactionId map $
```

What we're saying here is `payload ordered by transactionId, and then mapped to $`. With function chaining, we take the result of each function and use it for the input for the next function. There are some precedence rules you should keep in mind, which you can find [here](https://docs.mulesoft.com/mule-runtime/4.3/dataweave-flow-control-precedence).

Ok - so we've covered infix notation that explains a good bit about whats going on.. that still doesn't tell us what `$` and `$$` stand for though! These are what are known as [syntax sugar](https://en.wikipedia.org/wiki/Syntactic_sugar). They exist to make our code more concise and easier to understand (at least once you understand what they're doing) by allowing us to only define the body of our anonymouse functions. Lets take our example before, where we had

```data-weave
var myMapFunction = (item, index) ->
  (index): item
```

Instead of defining this function in the header, we could actually do it inline, thus creating an anonymouse function:

```data-weave
payload orderBy $.transactionId map ((item, index) ->
  (index): item
)
```

By why not make it a little more concise? We find ourselves making anonymous functions so often, it would be really nice to just supply the body of the function, rather than the entire lambda. Well, data-weave allows us to do this.. and says, assign `$` for the first paramter, `$$` for the second, `$$$` for the third.. noticing the pattern here? All the `$` represents is a way to access the parameter, where the number of `$` signs represents the position of the paramter.. so since `map` takes in a function with `(item, index)` as its parameters, `$` becomes our item and `$$` becomes our index.

What about nested functions? Well, how will you be able to tell the `$` from the outer function and the `$$` from the inner function apart? You can't, which means you won't be able to reference the outer function's parameters. When working with nested functions, even if you don't need the outer function's parameters, I would recommend using a lambda with named parameters to keep things clear.