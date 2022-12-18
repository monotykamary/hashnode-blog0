# State management with Async Generators

*This post is inspired by Vitalii Akimov's post on* [***Async Generators as an alternative to State Management***](https://www.freecodecamp.org/news/async-generators-as-an-alternative-to-state-management/)*. This post extends on how interfacing with async generators can be more natural through certain design patterns.*

## Introduction

Although there are a lot of state management libraries for JavaScript and its reactive libraries and frameworks, there isn't much public discussion on using native data types to DIY and compose robust state management. Surprisingly, we have all the tools as a developer to create state management trivially, that is library and framework agnostic, and with very few lines of code.

## What are generators?

First of all, what are generators? Generators are a *special type of closure* that allows you to **stop and resume its execution at will while preserving its context**. You can `yield` primitive values in a generator to later consume through `.next()`, or through iterating with a `for..of` loop.

```javascript
/* generator */
function* fooGen() {
  yield 'a';
  yield 'b';
  yield 'c';
}

/* log each value manually */
console.log(fooGen.next().value); // a
console.log(fooGen.next().value); // b
console.log(fooGen.next().value); // c

/* or log each value through a loop */
for (const item of fooGen()) {
    console.log(item) // a, b, c
}
```

Likewise, async generators are similar, but you can also `yield` Promise values through `async/await`. To consume these values, you would have to `await` every incoming input from an async generator.

```javascript
/* async generator */
async function* fooAsyncGen() {
  yield await Promise.resolve('a');
  yield await Promise.resolve('b');
  yield await Promise.resolve('c');
}

/* log each value manually */
(async () => {
    console.log((await fooAsyncGen.next()).value); // a
    console.log((await fooAsyncGen.next()).value); // b
    console.log((await fooAsyncGen.next()).value); // c
})()

/* or log each value through a loop */
for await (const item of fooAsyncGen()) {
    console.log(item) // a, b, c
}
```

For those familiar with functional programming, the interface for generators is equivalent to the [*Do notation*](https://en.wikibooks.org/wiki/Haskell/do_notation) in Haskell. If you are interested in learning more about generators Valentino Gagliardi has a great [overview of generators](https://www.valentinog.com/blog/generator/).

## Why use async generators for state management?

Async generators help **maintain asynchronous states**, making them implicit state machines that transition through states in Promises and primitive values. This means we don't need to worry too much about composing a lifecycle of how we get state and update state.

![A SYNCHRONOUS TIME CAN BE ASYNCHRONOUS; BUT ASYNCHRONOUS CANNOT BE A  SYNCHRONOUS TIME meme - Piñata Farms - The best meme generator and meme  maker for video & image memes](https://media.pinatafarm.com/protected/B183D0EF-49B8-47BF-A523-E72FD0CFFAAC/Advice-Yoda.3.meme-pfarm-thumbnail.png align="left")

## State management with async generators

Using async generators to manage state isn't technically new:

*   [https://www.freecodecamp.org/news/async-generators-as-an-alternative-to-state-management/](https://www.freecodecamp.org/news/async-generators-as-an-alternative-to-state-management/)
    
*   [https://medium.com/dailyjs/decoupling-business-logic-using-async-generators-cc257f80ab33](https://medium.com/dailyjs/decoupling-business-logic-using-async-generators-cc257f80ab33)
    
*   [https://codepen.io/rgdelato/pen/vGNRqP](https://codepen.io/rgdelato/pen/vGNRqP)
    

However, available public implementations couple transition logic inside the generator itself, making it much different ergonomically from other state management libraries.

### Constraints

To make this accessible to the average engineer, we will give ourselves the following constraints:

*   Should be relatively portable, such that you could write this up in a few minutes.
    
*   Should allow for a reducer-like composition and decoupling, such that you could move your code to use `useReducer`, Redux, or similar libraries by just changing which store you use.
    
    *   By extension, we should **NOT** couple any business logic inside a generator.
        

### Preparing our data types

Preparing our data types beforehand will save us a lot of time composing code. If you are familiar with the [general relativity of reactivity](https://github.com/kriskowal/gtor), there are hints that certain data types that function differently spatially and temporally affect how many lines of code are needed to achieve a particular design pattern.

For our case, we need a data type that composes an async generator but allows us to push data to it like an array, or something similar to the actor class **Subject** from the Observer pattern. Inspired from [Tung Vu](https://github.com/tungv/async-generator/tree/master/packages/subject) and [Vitalii Akimov](https://gist.github.com/awto/75c1f59047e248f2f461b5d0462e23dd), here is a simple implementation of a unicast Subject:

```javascript
// subject.js
const END = Symbol("END");

/* our main data type to compose reactive async generators */
function createUnicastSubject() {
  let pushQueue = [];
  let pullSignal = defer();
  let aborted = false;

  async function* generate() {
    while (!aborted) {
      const signal = await pullSignal.promise;
      if (signal === END) {
        return;
      }
      while (pushQueue.length) {
        const value = pushQueue.shift();
        yield value;
      }
    }
  }

  function push(value) {
    pushQueue.push(value);
    pullSignal.resolve();
    pullSignal = defer();
  }

  function stop() {
    aborted = true;
    pullSignal.resolve(END);
  }

  return [generate(), push, stop];
}

/* used to flatten the Promise callback */
function defer() {
  let resolve, reject;
  const promise = new Promise((_1, _2) => {
    resolve = _1;
    reject = _2;
  });
  return {
    promise,
    resolve,
    reject
  };
}

module.exports = {
  multicast,
  createUnicastSubject,
  defer
};
```

There is a reason why we are calling it a unicast Subject, but I'll get to that later on. With this data type, we can now easily compose our reducer store:

### Preparing our reducer store

Similar to Redux, we should have a function that consumes a reducer/state machine and an initial state. The function will output us a way to *subscribe* to state changes (iterating our async generator) and a way to dispatch action events:

```javascript
// reducer.js
const { createUnicastSubject } = require("./subject");

const createReducerStore = (reducer, initialState) => {
  const [stateIter, statePush, stateStop] = createUnicastSubject();
  const [actionIter, actionPush, actionStop] = createUnicastSubject();

  let state = initialState;

  (async () => {
    for await (const action of actionIter) {
      state = reducer(state, action);
      statePush(state);
    }
  })();

  function currentState() {
    return state;
  }

  function stop() {
    stateStop();
    actionStop();
  }

  /* we also output a way to iterate over action events here */
  return [
    currentState,
    actionPush,
    stateIter,
    actionIter,
    stop
  ];
};

module.exports = {
  createReducerStore
};
```

### Running it

Here, we will make a simple counter that responds will respond to click events to dispatch actions to our generator. We *subscribe* to changes in state by iterating over our counter async generator and updating the DOM inside the loop function:

```javascript
// index.js
import { createReducerStore } from "./reducer";
import "./styles.css";

/* our reducer, decoupled from the generator as it should be */
const counterReducer = (state, action) => {
  switch (action.type) {
    case "INCREMENT":
      return { type: "VALUE", value: state.value + 1 };
    case "DECREMENT":
      return { type: "VALUE", value: state.value - 1 };
    default:
      return state;
  }
};

/* the instantiation of our reducer store */
const [
  counterState,
  counterDispatch,
  counterStateIter,
  counterActionIter
] = createReducerStore(counterReducer, { type: "VALUE", value: 0 });

const valueEl = document.getElementById("value");

/* consumes the iterator to update the DOM */
(async () => {
  valueEl.innerHTML = counterState().value.toString();
  for await (const item of counterStateIter) {
    if (item.type === "VALUE") {
      valueEl.innerHTML = item.value.toString();
      console.log(item.value);
    }
  }
})();

document.getElementById("increment")
  .addEventListener("click", function () {
    counterDispatch({ type: "INCREMENT" });
  });
document.getElementById("decrement")
   .addEventListener("click", function () {
    counterDispatch({ type: "DECREMENT" });
  });
document
  .getElementById("incrementIfOdd")
  .addEventListener("click", function () {
    if (counterState().value % 2 !== 0) {
      counterDispatch({ type: "INCREMENT" });
    }
  });
document
  .getElementById("incrementAsync")
  .addEventListener("click", function () {
    setTimeout(function () {
      counterDispatch({ type: "INCREMENT" });
    }, 1000);
  });
```

#### CodeSandbox:

<iframe src="https://codesandbox.io/embed/compassionate-farrell-7pb7uw?fontsize=14&hidenavigation=1&theme=dark" style="width:100%;height:500px;border:0;border-radius:4px;overflow:hidden" sandbox="allow-forms allow-modals allow-popups allow-presentation allow-same-origin allow-scripts"></iframe>

### Pitfalls

Remember the part where we named our data type a **unicast** Subject. Since async generators and async iterables in general are a type of pull data type, once a consumer pulls, other consumers cannot receive that value. This means **you cannot have multiple consumers receive all values from a single async generator**. By default, values from an async generator will be consumed in a round-robin style to each consumer.

We can check this by adding more consumers to our `index.js` file:

```javascript
// index.js
(async () => {
  for await (const item of counterIter) {
    if (item.type === "VALUE") {
      console.log(item.value, 2);
    }
  }
})();

(async () => {
  for await (const item of counterActionIter) {
    console.log(item.type);
  }
})();

...
```

You will then notice delays in updating the DOM, despite the state being changed slowly in the background for each click.

#### CodeSandbox:

<iframe src="https://codesandbox.io/embed/fancy-snowflake-ms0shf?fontsize=14&hidenavigation=1&theme=dark" style="width:100%;height:500px;border:0;border-radius:4px;overflow:hidden" sandbox="allow-forms allow-modals allow-popups allow-presentation allow-same-origin allow-scripts"></iframe>

### Multicasting

To solve this issue, we need to be able to multicast our Subjects. "Teeing" async generators/iterables is [not trivial at first glance](https://stackoverflow.com/questions/63543455/how-to-multicast-an-async-iterable). However, with our Subject data type, we can solve this problem a lot easier and more ergonomically.

Our case simply requires a factory function that creates a unicast Subject that **shares the** consumption of the original async generator. We can keep track and sequentially push referentially to the consumers through an array:

```javascript
// subject.js
function multicast(asyncGenerator) {
    const consumers = [];
    (async () => {
	    try {
            // share & consume the generator + push out to consumers
		    for await (const item of asyncGenerator) {
	            for (const [_, consumerPush] of consumers) {
	                consumerPush(item);
	            }
	        }
	    }
        finally {
            // "unsubscribe" consumers when the for..await loop resolves
	        for (const [_, __, consumerStop] of consumers) {
				consumerStop();
			}
        }
    })();

    return function () {
        const [iter, push, stop] = createUnicastSubject();
        consumers.push([iter, push, stop]);
        return [iter, push, stop];
    };
}

...
```

Then we refactor our reducer store to output multicasted async generators:

```javascript
// reducer.js
const { createUnicastSubject, multicast } = require("./subject");

const createReducerStore = (reducer, initialState) => {
  const [stateIter, statePush, stateStop] = createUnicastSubject();
  const [actionIter, actionPush, actionStop] = createUnicastSubject();

  const multicastedStateIter = multicast(stateIter);
  const multicastedActionIter = multicast(actionIter);

  let state = initialState;

  (async () => {
    const [iter] = multicastedActionIter();
    for await (const action of iter) {
      state = reducer(state, action);
      statePush(state);
    }
  })();

  function currentState() {
    return state;
  }

  function stop() {
    stateStop();
    actionStop();
  }

  return [
    currentState,
    actionPush,
    multicastedStateIter,
    multicastedActionIter,
    stop
  ];
};

...
```

And update our `index.js` to call the factory function to create the Subject as a derivative consumer to the original async generator:

```javascript
// index.js
const [
  counterState,
  counterDispatch,
  counterIter,
  counterActionIter
] = createReducerStore(counterReducer, { type: "VALUE", value: 0 });

const valueEl = document.getElementById("value");

(async () => {
  valueEl.innerHTML = counterState().value.toString();
  const [iter] = counterIter();
  for await (const item of iter) {
    if (item.type === "VALUE") {
      valueEl.innerHTML = item.value.toString();
      console.log(item.value);
    }
  }
})();

(async () => {
  const [iter] = counterIter();
  for await (const item of iter) {
    if (item.type === "VALUE") {
      console.log(item.value, 2);
    }
  }
})();

(async () => {
  const [iter] = counterActionIter();
  for await (const item of iter) {
    console.log(item.type);
  }
})();

...
```

You will then be able to see that our iteration no longer has any delays and we can consume all values that originated from our async generator reducer.

#### CodeSandbox:

<iframe src="https://codesandbox.io/embed/silly-architecture-vr1b7d?fontsize=14&hidenavigation=1&theme=dark" style="width:100%;height:500px;border:0;border-radius:4px;overflow:hidden" sandbox="allow-forms allow-modals allow-popups allow-presentation allow-same-origin allow-scripts"></iframe>

There may be non-obvious performance impacts when using async generators decoupled this way, but for our case it is manageable. Luckily, since async generators are like a dependency graph that propagates in reverse when consumed, it makes it easy for us to plug in a scheduler to optimize performance later on.

## Conclusion

Originally, this story was created as a random thought experiment to create an ergonomic and functional improvement of technique from Vitalii Akimov's post on [**Async Generators as an alternative to State Management**](https://www.freecodecamp.org/news/async-generators-as-an-alternative-to-state-management/). Generators in general are rarely used, but with the right design pattern, they can compose solutions to common problems very elegantly.