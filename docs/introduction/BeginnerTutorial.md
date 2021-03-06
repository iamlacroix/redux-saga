# Beginner Tutorial

## Objectives of this tutorial

This tutorial attempts to introduce redux-saga in a (hopefully) accessible way.

For our getting started tutorial, we are going to use the trivial Counter demo from the Redux repo.
The application is quite simple but is a good fit to illustrate the basic concepts of redux-saga
without being lost in excessive details.

### The initials setup

Before we start, you have to clone the repository at

https://github.com/yelouafi/redux-saga-beginner-tutorial

>The final code of this tutorial is located in the sagas branch

Then in the command line, type

```
cd redux-saga-beginner-tutorial
npm install
```

To run the application type

```
npm start
```

We are starting with the simplest use case: 2 buttons to `Increment` and `Decrement` a counter.
Later, we will introduce asynchronous calls.

If things go well, you should see 2 buttons `Increment` and `Decrement` along with a message
below showing `Counter : 0`.

>In case you encountered an issue with running the application. Feel free to create an issue
on the Tutorial repo

>https://github.com/yelouafi/redux-saga-beginner-tutorial/issues


## Hello Sagas!

We are going to create our first Saga. Following the tradition, we will write our 'Hello, world'
version for Sagas.

Create a file `sagas.js` then add the following snippet

```javascript
export function* helloSaga() {
  console.log('Hello Sagas!');
}
```

So nothing scary, just a normal function (Ok except for the `*`). All it does
is printing a greeting message into the console.

In order to run our Saga, we need to

- create a Saga middleware with a list of Sagas to run (so far we have only one `helloSaga`)
- connect the Saga middleware to the Redux store


We will makes the changes to `main.js`

```javascript
// ...
import { createStore, applyMiddleware } from 'redux'
import createSagaMiddleware from 'redux-saga'

//...
import { helloSaga } from './sagas'

const store = createStore(
  reducer,
  applyMiddleware(createSagaMiddleware(helloSaga))
)

// rest unchanged
```

First we import our Saga from the `./sagas` module. Then we create a middleware using the factory function
`createSagaMiddleware` exported by the `redux-saga` library. `createSagaMiddleware` accepts a list of Sagas
which will be started immediately by the middleware.


So far, our Saga does nothing special, just log a message then exits.


## Making Asynchronous calls

Now let's add soemthing closer to the original Counter demo. To illustrate asynchronous calls, we will add another button
to increment the counter 1 second after the click.

First things first, we'll provide an additional callback `onIncrementAsync` to the UI component.

```javascript
export class Counter extends React.Component {

  render() {
    const { ..., onIncrementAsync, ... } = this.props

    return (
      <div>
        ...
        {' '}
        <button onClick={onIncrementAsync}>Increment after 1 second</button>
        <hr />
        <div>Counter : {counter}</div>
      </div>
    )
  }
}
```

Next we should connect the `onIncrementAsync` of the Component to a Store action.

We will modify the `main.js` module as follows

```javascript
function render() {
  ReactDOM.render(
    <Counter
      ...
      onIncrementAsync={() => store.dispatch({ type: 'INCREMENT_ASYNC' })}
    />,
    rootEl
  )
}
```

Note that unlike in redux-thunk, our component dispatches a plain object action.

Now we will introduce another Saga to perform the asynchronous call. Our use case is as follows

> On each `INCREMENT_ASYNC` action, we want to start a task that will do the following

>- Wait 1 second then increment the counter


Add the following code to the `sagas.js` module

```javascript
import { takeEvery } from 'redux-saga'
import { put } from 'redux-saga/effects'

// an utility function: return a Promise that will resolve after 1 second
const delay = ms => new Promise(resolve => setTimeout(resolve, ms))

// Our worker Saga: will perform the async increment task
function* incrementAsync() {
  yield delay(1000)
  yield put({ type: 'INCREMENT' })
}

// Our watcher Saga: spawn a new incrementAsync task on each INCREMENT_ASYNC
export function* watchIncrementAsync() {
  yield* takeEvery('INCREMENT_ASYNC', incrementAsync)
}
```

Ok time for some explanations. First we create an utility function `delay` which returns a Promise
that will resolve after 1 second. We'll use this function to *block* the Generator.

Sagas, which are implemented as Generator functions, yield objects to the
redux-saga middleware. The yielded objects are a kind of instructions to be interpreted by
the middleware. When the middleware retrieves a yielded Promise, it'll suspend the Saga until
the Promise completes. In the above example, the `incrementAsync` Saga will be suspended until
the Promise returned by `delay` resolves, which will happen after 1 second.

Once the Promise is resolved, the middleware will resume the Saga to execute the next statement
(more accurately to execute all the following statement until the next yield). In our case, the
next statement is another yielded object: which is the result of calling `put({type: 'INCREMENT'})`.
It means the Saga instructs the middleware to dispatch an `INCREMENT` action.

`put` is one example of what we call an *Effect*. Effects are simple JavaScript Objects which
contains some instructions to be fulfilled by the middleware. When a middleware retreives an Effect
yielded by a Saga, it pauses the Saga until the Effect is fullfilled then the Saga is resumed
again.

So to recapitulate the `incrementAsync` Saga sleeps for 1 second via the call to `delay(1000)` then
dispatch an `INCREMENT` action.

Next, we created another Saga `watchIncrementAsync`. The Saga will watch the dispatched `INCREMENT_ASYNC`
actions and spawn a new `incrementAsync` task on each action. For this purpose, we use a helper function
provided by the library `takeEvery`. Which will perform the process above.

Before we start the application, we need to connect the `watchIncrementAsync` Saga to the Store

```javascript

//...
import { helloSaga, watchIncrementAsync } from './sagas'

const store = createStore(
  reducer,
  applyMiddleware(createSagaMiddleware(helloSaga, watchIncrementAsync))
)

//...
```

Note we don't need to connect the `incrementAsync` Saga, because it'll be started dynamically
by `watchIncrementAsync` on each `INCREMENT_ASYNC` action.


## Making our code testable

We want to test our `incrementAsync` Saga to make sure it performs the desired task.

Create another file `sagas.spec.js`

```javascript
import test from 'tape';

import { incrementAsync } from './sagas'

test('incrementAsync Saga test', (assert) => {
  const gen = incrementAsync()

  // now what ?
});
```

Since `incrementAsync` is a Generator function, when we run it outside the middleware,
we get a Generator object. A Generator object is an object which has a `next` method.
Each time you invoke `next` on the generator, you get an object of the following shape

```javascript
gen.next() // => { done: boolean, value: any }
```

The `value` field contains the yielded expression, i.e. the result of the expression after
the `yield`. The `done` field indicates if the generator has terminated or if there are still
more 'yield' expressions.

In the case of `incrementAsync`, the generator yield 2 values consecutively

1. `yield delay(1000)`  
2. `yield put({type: 'INCREMENT'})`  


So if we invoke the next method of the generator 3 times consecutively we get the following
results

```javascript
gen.next() // => { done: false, value: <result of calling delay(1000)> }
gen.next() // => { done: false, value: <result of calling put({type: 'INCREMENT'})> }
gen.next() // => { done: true, value: undefined }
```

The first 2 invocations return the results of the yield expressions. On the 3rd invocation
since there is no more yield the `done` field is set to true. And since the `incrementAsync`
Generator doesn't return anything (no `return` statement), the `value` field is set to
`undefined`.

So now, in order to test the logic inside `incrementAsync`, we'll simply have to iterate
over the returned Generator and check the values yielded by the generator.

```javascript
import test from 'tape';

import { incrementAsync } from '../src/sagas'

test('incrementAsync Saga test', (assert) => {
  const gen = incrementAsync()

  assert.deepEqual(
    gen.next().value,
    { done: false, value: ??? },
    'incrementAsync should return a Promise that will resolve after 1 second'
  )
});
```

The issue is how do we test return value of `delay`. We can't simply do a simple equality test
on Promises. If `delay` returned a *normal* value, things would've been be easier to test.

Well, `redux-saga` provides a way which makes the above statement possible. Instead of calling
`delay(1000)` directly inside `incrementAsync`, we'll call it *indirectly*


```javascript
//...
import { put, call } from 'redux-saga/effects'

const delay = ms => new Promise(resolve => setTimeout(resolve, ms))

function* incrementAsync() {
  // use the call Effect
  yield call(delay, 1000)
  yield put({ type: 'INCREMENT' })
}
```

Instead of doing `yield delay(1000)`, we're now doing `yield call(delay, 1000)` so what's the difference ?

In the first case, the yield expression `delay(1000)` is evaluated before it gets passed to the caller of `next`
(the caller could be the middleware when running our code. It could also be our test code which runs the Generator
function and iterates over the returned Generator). So what the caller gets is a Promise, like in the test code
above.

In the second case, the yield expression `call(delay, 1000)` is what gets passed to the caller of `next`. `call`
just like `put`, returns an Effect which instructs the middleware to call a given function with the given arguments.
In fact, neither `put` nor `call` performs any dispatch or asynchronous call by themselves, they simply return
plain JavaScript objects.

```javascript
put({type: 'INCREMENT'}) // => { type: PUT, action: {type: 'INCREMENT'} }
call(delay, 1000)        // => { type: CALL, function: delay, args: [1000]}
```

What happens is that the middleware examines the type of each yielded Effect then decide how
to fulfill that Effect. If the Effect type is a `PUT` then it'll dispatch an action to the Store.
If the Effect is a `CALL` then it'll call the given function.

This separation between Effect creation and Effect execution makes possible to test our Generator
in a surprisingly easy way

```javascript
import test from 'tape';

import { put, call } from 'redux-saga/effects'
import { incrementAsync, delay } from './sagas'

test('incrementAsync Saga test', (assert) => {
  const gen = incrementAsync()

  assert.deepEqual(
    gen.next().value,
    call(delay, 1000),
    'incrementAsync Saga must call delay(1000)'
  )

  assert.deepEqual(
    gen.next().value,
    put({type: 'INCREMENT'}),
    'incrementAsync Saga must dispatch an INCREMENT action'
  )

  t.deepEqual(
    generator.next(),
    { done: true, value: undefined },
    'incrementAsync Saga must be done'
  )

  t.end()
});
```

Since `put` and `call` return plain objects, we can reuse the same functions in our test
code. And to test the logic of `incrementAsync`, we simply iterate over the generator
and doing `deepEqual` tests on its values.

In order to run the above test, type
```
npm test
```

which should report the results on the console
