# fr(e)actal

`freactal` is a composable state management library for React.

The library grew from the idea that state should be just as flexible as your React code; the state containers you build with `freactal` are just components, and you can compose them however you'd like.  In this way, it attempts to address the often exponential relationship between application size and complexity as projects grow.

Like [`redux`](http://redux.js.org/) and React in general, `freactal` builds on the principle of unidirectional flow of data.  However, it does so in a way that feels idiomatic to ES2015+ and doesn't get in your way.

When building an application, it can replace [`redux`](redux.js.org), [`reselect`](https://github.com/reactjs/reselect), [`redux-loop`](https://github.com/redux-loop/redux-loop), [`redux-thunk`](https://github.com/gaearon/redux-thunk), [`redux-saga`](https://github.com/redux-saga/redux-saga), `[fill-in-the-blank sub-app composition technique]`, and potentially [`recompose`](https://github.com/acdlite/recompose), depending on how you're using it.

It's design philosophy aligns closely with the [Zen of Python](https://www.python.org/dev/peps/pep-0020/):

```
Beautiful is better than ugly.
  Explicit is better than implicit.
    Simple is better than complex.
  Complex is better than complicated.
    Flat is better than nested.
  Sparse is better than dense.
Readability counts.
```

<a href="#table-of-contents"><p align="center" style="margin-top: 400px"><img src="https://cloud.githubusercontent.com/assets/5016978/24835268/f983b58e-1cb1-11e7-8885-6c029cbbd224.png" height="60" width="60" /></p></a>


## Table of Contents

- [Guide](#guide)
  - [Containing state](#containing-state)
  - [Accessing state from a child component](#accessing-state-from-a-child-component)
  - [Transforming state](#transforming-state)
  - [Transforming state (cont.)](#transforming-state-cont)
  - [Intermediate state](#intermediate-state)
  - [Effect arguments](#effect-arguments)
  - [Transforming state from a child](#transforming-state-from-a-child)
  - [Computed state values](#computed-state-values)
  - [Composing multiple state containers](#composing-multiple-state-containers)
- [Architecture](#architecture)
- [API Documentation](#api-documentation)
  - [`provideState`](#providestate)
    - [`initialState`](#initialstate)
    - [`effects`](#effects)
      - [`initialize`](#initialize)
    - [`computed`](#computed)
    - [`middleware`](#middleware)
  - [`injectState`](#injectstate)
  - [`hydrate`](#hydrate)
- [Helper functions](#helper-functions)
  - [`hardUpdate`](#hardupdate)
  - [`softUpdate`](#softupdate)
  - [`spread`](#spread)
- [Server-side Rendering](#server-side-rendering)
  - [with `React#renderToString`](#with-reactrendertostring)
  - [with Rapscallion](#with-rapscallion)
- [FAQ](#faq)


<a href="#table-of-contents"><p align="center" style="margin-top: 400px"><img src="https://cloud.githubusercontent.com/assets/5016978/24835268/f983b58e-1cb1-11e7-8885-6c029cbbd224.png" height="60" width="60" /></p></a>


## Guide

This guide is intended to get you familiar with the `freactal` way of doing things.  If you're looking for something specific, take a look at the [API Documentation](#api-documentation).  If you're just starting out with `freactal`, read on!


### Containing state

Most state management solutions for React put all state in one place.  `freactal` doesn't suffer from that constraint, but it is a good place to start.  So let's see what that might look like.

```javascript
import { provideState } from "freactal";

const wrapComponentWithState = provideState({
  initialState: () => ({ counter: 0 })
});
```

In the above example, we define a new state container type using `provideState`, and provide it an argument.  You can think about the arguments passed to `provideState` as the schema for your state container; we'll get more familiar with the other possible arguments later in the guide.

The `initialState` function is invoked whenever the component that it is wrapping is instantiated.  But so far, our state container is not wrapping anything, so let's expand our example a bit.

```javascript
import React, { Component } from "react";
import { render } from "react-dom";

const Parent = ({ state }) => (
  <div>
    { `Our counter is at: ${state.counter}` }
  </div>
);

render(<Parent />, document.getElementById("root"));
```

That's a _very_ basic React app.  Let's see what it looks like to add some very basic state to that application.

```javascript
import React, { Component } from "react";
import { render } from "react-dom";
import { provideState } from "freactal";

const wrapComponentWithState = provideState({
  initialState: () => ({ counter: 0 })
});

const Parent = wrapComponentWithState(({ state }) => (
  <div>
    { `Our counter is at: ${state.counter}` }
  </div>
));

render(<Parent />, document.getElementById("root"));
```

Alright, we're getting close.  But we're missing one important piece: `injectState`.

Like `provideState`, `injectState` is a component wrapper.  It links your application component with the state that it has access to.

It may not be readily apparent to you why `injectState` is necessary, so let's make it clear what each `freactal` function is doing.  With `provideState`, you define a state template that can then be applied to any component.  Once applied, that component will act as a "headquarters" for a piece of state and the effects that transform it (more on that later).  If you had a reason, the same template could be applied to multiple components, and they'd each have their own state based on the template you defined.

But that only _tracks_ state.  It doesn't make that state accessible to the developer.  That's what `injectState` is for.

In early versions of `freactal`, the state was directly accessible to the component that `provideState` wrapped.  However, that meant that whenever a state change occurred, the entire tree would need to re-render.  `injectState` intelligently tracks which pieces of state that you actually access, and a re-render only occurs when _those_ pieces of state undergo a change.

Alright, so let's finalize our example with all the pieces in play.

```javascript
import React, { Component } from "react";
import { render } from "react-dom";
import { provideState, injectState } from "freactal";

const wrapComponentWithState = provideState({
  initialState: () => ({ counter: 0 })
});

const Parent = wrapComponentWithState(injectState(({ state }) => (
  <div>
    { `Our counter is at: ${state.counter}` }
  </div>
)));

render(<Parent />, document.getElementById("root"));
```

That'll work just fine!


### Accessing state from a child component

As was mentioned above, the `provideState`-wrapped component isn't really the one that provides access to state.  That's `injectState`'s job.  So what would stop you from injecting state into a child component, one that isn't containing state itself?  The answer is nothing!

Let's modify the example so that we're injecting state into a child component.

```javascript
import React, { Component } from "react";
import { render } from "react-dom";
import { provideState, injectState } from "freactal";


const Child = injectState(({ state }) => (
  <div>
    { `Our counter is at: ${state.counter}` }
  </div>
));

const wrapComponentWithState = provideState({
  initialState: () => ({ counter: 0 })
});

const Parent = wrapComponentWithState(({ state }) => (
  <Child />
));


render(<Parent />, document.getElementById("root"));
```

Let's review what's going on here.

1. Using `provideState`, we define a state-container template intended to store a single piece of state: the `counter`.
2. That template is applied to the `Parent` component.
3. When the `Parent` is rendered, we see that it references a `Child` component.
4. That `Child` component is wrapped with `injectState`.
5. Because `Child` is contained within the subtree where `Parent` is the root node, it has access to the `Parent` component's state.

We could insert another component at the end, and `injectState` into the `GrandChild` component, and it would work the same.


### Transforming state

Alright, so we know how to setup state containers, give them an initial state, and consume that state from child components.  But all of this is not very useful if state is never updated.  That's where effects come in.

Effects are the one and only way to change `freactal` state in your application.  These effects are defined as part of your state container template when calling `provideState`.  And they can be invoked from anywhere that state has been injected (with `injectState`).

Let's take a look at that first part.

```javascript
const wrapComponentWithState = provideState({
  initialState: () => ({ counter: 0 }),
  effects: {
    addOne: () => state => Object.assign({}, state, { counter: state.counter + 1 })
  }
});
```

You might be wondering why we have that extra `() =>` right before `state =>` in the `addOne` definition.  That'll be explained in the next section - for now, let's look at all the other pieces.

In the above example, we've defined an effect that, when invoked, will update the `counter` in our state container by adding `1`.

Since updating an element of state based on previous state (and potentially new information) is something you'll be doing often, `freactal` [provides a shorthand](#softupdate) to make this a bit more readable:

```javascript
const wrapComponentWithState = provideState({
  initialState: () => ({ counter: 0 }),
  effects: {
    addOne: softUpdate(state => ({ counter: state.counter + 1 }))
  }
});
```

Now let's look at how you might trigger this effect:

```javascript
const Child = injectState(({ state, effects }) => (
  <div>
    { `Our counter is at: ${state.counter}` }
    <button onClick={effects.addOne}>Add one</button>
  </div>
));
```

Wherever your `<Child />` is in your application, the state and effects it references will be accessible, so long as the state container is somewhere further up in the tree.

### Transforming state (cont.)

If you've used Redux, effects are roughly comparable to an action-reducer pair, with a couple of important differences.

The first of those differences relates to asychronicity.  Under the hood, `freactal` relies heavily on `Promise`s to schedule state updates.  In fact, the following effects are all functionally equivalent:

```javascript
addOne: () => state => Object.assign({}, state, { counter: state.counter + 1 })
/* vs */
addOne: () => state => Promise.resolve(Object.assign({}, state, { counter: state.counter + 1 }))
/* vs */
addOne: () => state => new Promise(resolve => resolve(Object.assign({}, state, { counter: state.counter + 1 })))
```

To put it explicitly, the value you provide for each key in your `effects` object is:

1. A function that takes in some arguments (we'll cover those shortly) and returns...
2. A promise that resolves to...
3. A function that takes in state and returns...
4. The updated state.

Step 2 can optionally be omitted, since `freactal` wraps these values in `Promise.resolve`.

For most developers, this pattern is probably the least familiar of those that `freactal` relies upon.  But it allows for some powerful and expressive state transitions with basically no boilerplate.

For example, any number of things can occur between the time that an effect is invoked and the time that the state is updated.  These "things" might include doing calculations, or talking to an API, or integrating with some other JS library.

So, you might define the following effect:

```javascript
updatePosts: () => fetch("/api/posts")
  .then(result => result.json())
  .then(({ posts }) => state => Object.assign({}, state, { posts }))
```

In other words, any action that your application might take, that ultimately _could_ result in a state change can be simply expressed as an effect.  Not only that, but this pattern also allows for effects and UI components to be tested with clean separation.

And, perhaps most importantly, this pattern allows for intermediate state.


### Intermediate state

**TODO**


### Effect arguments

**TODO**


### Transforming state from a child

**TODO**


### Computed state values

**TODO**


### Composing multiple state containers

**TODO**


<a href="#table-of-contents"><p align="center" style="margin-top: 400px"><img src="https://cloud.githubusercontent.com/assets/5016978/24835268/f983b58e-1cb1-11e7-8885-6c029cbbd224.png" height="60" width="60" /></p></a>


## Architecture

**TODO**


<a href="#table-of-contents"><p align="center" style="margin-top: 400px"><img src="https://cloud.githubusercontent.com/assets/5016978/24835268/f983b58e-1cb1-11e7-8885-6c029cbbd224.png" height="60" width="60" /></p></a>


## API Documentation

### `provideState`

**TODO**

#### `initialState`

**TODO**

#### `effects`

**TODO**

##### `initialize`

**TODO**

#### `computed`

**TODO**

#### `middleware`

**TODO**

### `injectState`

**TODO**

### `hydrate`

**TODO**


<a href="#table-of-contents"><p align="center" style="margin-top: 400px"><img src="https://cloud.githubusercontent.com/assets/5016978/24835268/f983b58e-1cb1-11e7-8885-6c029cbbd224.png" height="60" width="60" /></p></a>


## Helper functions

**TODO**

### `hardUpdate`

This handy helper provides better ergonomics when defining an effect that updates state, regardless of the previous state.

It can be consumed like so:

```javascript
import { provideState, hardUpdate } from "freactal";
const wrapComponentWithState = provideState({
  // ...
  effects: {
    myEffect: hardUpdate({ setThisKey: "to this value..." })
  }
});
```

Which is equivalent to the following:

```javascript
import { provideState } from "freactal";
const wrapComponentWithState = provideState({
  // ...
  effects: {
    myEffect: () => state => Object.assign({}, state, { setThisKey: "to this value..." })
  }
});
```


### `softUpdate`

`softUpdate` is provides a shorthand for updating an element of state that _is_ dependant on the previous state.

It can be consumed like so:

```javascript
import { provideState, softUpdate } from "freactal";
const wrapComponentWithState = provideState({
  // ...
  effects: {
    myEffect: softUpdate(state => ({ counter: state.counter + 1 }))
  }
});
```

Which is equivalent to the following:

```javascript
import { provideState, softUpdate } from "freactal";
const wrapComponentWithState = provideState({
  // ...
  effects: {
    myEffect: () => state => Object.assign({}, state, { counter: state.counter + 1 })
  }
});
```

Any arguments that are passed to the invocation of your effect will also be passed to the function you provide to `softUpdate`.

I.e.

```javascript
effects: {
  updateCounterBy: (state, addVal) => Object.assign({}, state, { counter: state.counter + addVal })
}
```

is equivalent to:

```javascript
effects: {
  myEffect: softUpdate((state, addVal) => ({ counter: state.counter + addVal }))
}
```


### `spread`

**TODO**


<a href="#table-of-contents"><p align="center" style="margin-top: 400px"><img src="https://cloud.githubusercontent.com/assets/5016978/24835268/f983b58e-1cb1-11e7-8885-6c029cbbd224.png" height="60" width="60" /></p></a>


## Server-side Rendering

**TODO**

### with `React#renderToString`

**TODO**

### with Rapscallion

**TODO**


<a href="#table-of-contents"><p align="center" style="margin-top: 400px"><img src="https://cloud.githubusercontent.com/assets/5016978/24835268/f983b58e-1cb1-11e7-8885-6c029cbbd224.png" height="60" width="60" /></p></a>


## FAQ

**Do you support time-traveling?**

TODO

**What middleware is available?**

- [freactal-devtools](https://github.com/FormidableLabs/freactal-devtools)
- [freactal-logger](https://github.com/FormidableLabs/freactal-logger)

**TODO**
