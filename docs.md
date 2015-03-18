Explore React
====

Ever try finding your way around a huge codebase? It doesn't take very long before the vertigo kicks in.  When looking at something like React, what we see is the cumulative result of years of effort from hundreds of very bright people.  So how do we mere mortals even begin to understand all of the careful decisions that have gone into it?  Today might be the day you take the first couple of steps.

It's helpful to understand the basic concepts behind React, and much has already been written about [the concepts behind React's architecture](#further-reading). I encourage you to read up if you're interested, but it's not essential for our exploration today.

Hello World
----

This being my first journey in to the React codebase, we'll start with the basics and see what we find. Let's begin with the most simple of examples, the Hello World. If you've done any tutorials then something like this should look familiar:

```js
React.render(
  React.createElement('h1', null, 'Hello, world!'),
  document.getElementById('example')
);
```

Not a lot of action on the surface here. We create an `h1` element and render it to the DOM node with ID of `example`.

Now, if our only goal was to get our message onto the screen we could just set some innerHTML and we'd be done. With React, there's a little more going on because it's not really trying to solve the problem of putting content on a screen. Rendering _changes_ to content is where React's strength shines. We'll be taking a look under the hood to see some of the ways React tries to meet this challenge.


React.render
====

Let's take a step into `React.render` (you can use the _"source"_ link below the code snippet if you'd like to see it in context):

```embed
src/browser/ui/ReactMount.js:432
```

If you recall our [Hello World example](#explore-react:hello-world), `nextElement` is the `h1` we created, `container` is the `#example` DOM node, and we have omitted the `callback` parameter (this is optional).

At a high level we can describe this function in three steps:

1. [Make sure we're rendering a valid element](#react-render:make-sure-we-39-re-rendering-a-valid-element)
2. [Work out whether to replace or reuse the previous element](#react-render:replace-it-or-reuse-it-)
3. [Finally render a new component](#react-render:render-a-new-component)

Before we go on it's interesting to note the use of the argument name `nextElement`. We should ask the question: why is it called "next element" and not just "element"? As mentioned earlier, React is not merely trying to render content to the page. Its main concern is in efficiently rendering _changing content_.  In that light the name `nextElement` makes more sense. When the content changes, rather than updating the DOM manually ourselves we simply create the "next element" and pass it to React to work out the best way to apply those changes.

We'll also see further on in [[ . ][ Replace it or reuse it? ]] that `nextElement` has a counterpart of `prevElement`, which adds useful context.


Make sure we're rendering a valid element
----

The first thing the `render` function does is make sure we have a valid element. Placing this check at the function's entry-point follows the same idea as a booth to check your passport at a border crossing. Once we have verified that `nextElement` is a valid React element we can freely make assumptions on what to expect from it in all of the following function calls, without having to re-check each time.

React enforces the "valid elements only" rule by putting `React.isValidElement(nextElement)` inside a call to the `invariant` function. This function simply checks for a truthy first argument and throws an error otherwise:

```embed
src/browser/ui/ReactMount.js:433,449
```

The `invariant` function is a simple concept but there are still some interesting implementation details which you can read more about in [[ The invariant function ]].

In our example we're using a "native element" (representing an `<h1>`), so a lot of the complexity of dealing with elements won't apply here. You can read some more about elements in the [[ Elements ]] section.


Replace it or reuse it?
----

As mentioned earlier, React's strength is in rendering changes. As we'll see further on, part of this strength comes from the fact that React makes the decision of whether to replace or reuse, freeing the developer from needing to handle both cases.

Looking further down in the `React.render` function we see a comparison of two elements to see whether we'll be updating a previously rendered element or rendering a new one:

```embed
src/browser/ui/ReactMount.js:454,455
```

`prevComponent` is a component that was previously rendered to the `container` element. React looks up this information by storing component IDs. You can read more about that in the [[ React IDs ]] section.

For now, let's take a look inside the `shouldUpdateReactComponent` function:

```embed
src/core/shouldUpdateReactComponent.js:28
```

First of all, if either `prevElement` or `nextElement` don't exist we can return straight away with the answer _"Nope, don't update the component. Make a new one"_:

```embed
src/core/shouldUpdateReactComponent.js:29
```

In the next couple of lines we see how the function handles scalar values. This is for the case where `prevElement` or `nextElement` is actually a string or a number rather than an element object. In this situation we can get our answer very quickly:

- Are `prevElement` and `nextElement` _both_ scalars? Update.
- `prevElement` is a scalar but `nextElement` is an element object? Replace.

```embed
src/core/shouldUpdateReactComponent.js:30,34
```

Now, if _neither_ are scalars we need to check some more things to answer the question. We will only update the previous component if all of the following are true:

- both elements must have the same `type` attribute (Note: this is different to what we get from `typeof`. In our example the type is `'h1'`, but it could be any custom element type you have defined).
- both elements must have the same `key`. (You can read more about [why keys are important in the React docs](https://facebook.github.io/react/docs/multiple-components.html#dynamic-children))
- both elements must have the same owner.

```embed
src/core/shouldUpdateReactComponent.js:35,38
```

This brings up an interesting concept. All of these checks are more or less saying _"If the 2 elements are the same, update the content. Otherwise replace the old element with the new one"_. Sounds pretty obvious when you say it like that. So why did we need to go to all of this trouble just to tell if two things are the same?  The clever thing about this approach is that React doesn't actually care whether the two things are the actual same element. It just needs to determine whether they are _close enough_ to be able to efficiently transition from one to the next.

It's an important underlying concept to grasp when using React. You'll notice that when you write your own custom elements, the `render` function returns new elements every time it is executed. If you're used to other paradigms this can feel strange because normally we associate a significant cost with "creating something". The instinct then is to locate the existing element ourselves and update it, to avoid the waste of creating a new one. React turns this idea upside down. Creating a React element is cheap, so we just create every time and leave it to React to decide how to render it.  Not only will the end result be more efficient than if we updated DOM elements manually, our code is also much cleaner, going from two execution paths (ie. "update? create?") to one.

In the case of our "Hello World" example, there won't be any `prevComponent`.  But if there was we would either update that component:

```embed
src/browser/ui/ReactMount.js:455,461
```

or remove the existing one before proceeding to render the new element:

```embed
src/browser/ui/ReactMount.js:462,464
```


Render a new component
----

To recap, we've determined that there is no existing component, and we want to render a new one. We call the `_renderNewRootComponent` function:

```embed
src/browser/ui/ReactMount.js:492,496
```

Let's take a look at where that function is defined:

```embed
src/browser/ui/ReactMount.js:377,381
```

See that `shouldReuseMarkup` argument? We already decided we weren't going to reuse an existing component in the previous step, but this is something else. In the case where we have rendered our React elements on the server, we still won't have a `prevComponent` but we _will_ have some React markup inside `container`. As long as it looks valid, React will attempt to pick up the existing markup and run with it.

The next thing we see in this function is a warning against nested updates, by making sure the `ReactCurrentOwner.current` is `null`. The "current owner" is whichever component is running its `render` function. If we're rendering the root component (like we are here) we expect there to be no owner:

```embed
src/browser/ui/ReactMount.js:385,391
```

Next we instantiate the component and register it. Registering does a couple of things like:

- starting up a monitor to keep track of the scroll value (so that React can provide "virtual DOM scroll events" to its "virtual DOM elements"):

```embed
src/browser/ui/ReactMount.js:363
```

- indexes the container and the root component instance against the root ID (so that we always have an easy way to find the root node of the tree):

```embed
src/browser/ui/ReactMount.js:365,366
```

Finally the update is batched so that React can perform several DOM updates together and [avoid unnecessary reflow](http://stackoverflow.com/questions/510213/when-does-reflow-happen-in-a-dom-environment):

```embed
src/browser/ui/ReactMount.js:403,409
```

There's a lot of interesting things happening inside `ReactUpdates.batchedUpdates`, and it probably deserves a separate post all to itself. We've only just scratched the surface, but my hope is that what we've looked at here will be enough to get you started on your own exploration. Adventure awaits!


Further Reading
====

- [Official React docs](http://facebook.github.io/react/docs/)
- [Re-implementing some React concepts from scratch](https://gcanti.github.io/2014/10/29/understanding-react-and-reimplementing-it-from-scratch-part-1.html)
- [Some nicely annotated examples](http://binarymuse.github.io/react-primer/build/)


Elements
====

React keeps the definition of an "element" very simple. Rather than trying to abstract the general behaviour of every possible kind of element, a `ReactElement` is just a javascript object which follows a certain structure.


What makes a valid element?
----

How does React know if it is dealing with an element or some other type of object? Often when faced with this question you might reach for `instanceof` but a decision was made here to use the more flexible [Duck Typing](https://en.wikipedia.org/wiki/Duck_typing) approach. There's a reference to this in the comments:

```embed
src/browser/ui/ReactMount.js:443
```

Stepping into the `isValidElement` function we check for the element's "quack" by looking for the presence of a special flag, `_isReactElement`:

```embed
src/classic/element/ReactElement.js:295
```

The flag is set on the `ReactElement` prototype defined earlier in that same file:

```embed
src/classic/element/ReactElement.js:140,143
```


React IDs
====

React assigns every element a unique ID (you'll see it in the custom `data-reactid` attribute if you inspect the DOM). These IDs usually look like numbers joined by dots, eg. `.0.0.1`, and each dot represents a new level in the element tree.  Doing it this way means it's possible to do a lot of navigating just by manipulating that string.  For example we can find the ID of the parent:

```embed
src/core/ReactInstanceHandles.js:85,87
```

By the way, that `SEPARATOR` constant is defined earlier in the same file:

```embed
src/core/ReactInstanceHandles.js:19
```

There are also a lot of other useful functions in here for doing more complicated traversals, like finding the first common ancestor of two elements:

```embed
src/core/ReactInstanceHandles.js:138
```

It's also useful to note that IDs are treated a little differently when rendering in a browser or on a server. In the browser IDs are sequential, starting at `0` and incrementing each time you ask for a new one:

```embed
src/browser/ClientReactRootIndex.js:15,21
```

However when rendering on the server they are chosen randomly:

```embed
src/browser/server/ServerReactRootIndex.js:21,27
```

There are some interesting comments on the reasons in this [thread about data-reactid](https://groups.google.com/forum/#!topic/reactjs/ewTN-WOP1w8).


The invariant function
====

```embed
src/vendor/core/invariant.js:14,25
```

You might wonder why the invariant function uses those `a, b, c, d, e, f` arguments. These are used to create the error message. `format` is a string template, and `a` to `f` are substituted any place `format` contains the string `%s`. So `format("I don't always %s but when I do I %s", thing, otherThing)` could be the basis for your exciting new meme generation app. Feel free to steal that one.

But the question remains, couldn't we have written this using the `arguments` object instead of `a..f`? Yes we could, but it turns out there's a significant impact on speed. Since React uses `invariant` quite often, and in some performance-critical code, even a little gain will be worthwhile. A quick google showed a [jsperf experiment on this topic](http://jsperf.com/invariant-with-and-without-explicit-parameters) if you're interested in some numbers.

Another common React pattern we see in this function is making "development-only" blocks:

```embed
src/vendor/core/invariant.js:26
```

This is used in several places to give extra warnings and feedback when running in a development environment, in a way that is easy to strip out for a production build.

The interesting thing here is that if you inspect code running in a browser, you won't see `__DEV__` anywhere. In its place you'll see:

```js
if ("production" !== "development") {
```

You can take a look at how this is done (with the help of the [recast](https://npmjs.com/package/recast) module) here:

```embed
vendor/constants.js:42,45
```
