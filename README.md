# ember-test-waiters

![CI Build](https://github.com/rwjblue/ember-test-waiters/workflows/CI%20Build/badge.svg)
[![npm version](https://badge.fury.io/js/ember-test-waiters.svg)](https://badge.fury.io/js/ember-test-waiters)
[![Monthly Downloads from NPM](https://img.shields.io/npm/dm/ember-test-waiters.svg?style=flat-square)](https://www.npmjs.com/package/ember-test-waiters)
[![Code Style: prettier](https://img.shields.io/badge/code_style-prettier-ff69b4.svg?style=flat-square)](#badge)

This addon provides APIs to allow [@ember/test-helpers](https://github.com/emberjs/ember-test-helpers/) to play nicely
with other asynchronous operations, such as an application that is waiting for a CSS transition or an IndexDB transaction.
The async helpers inside `@ember/test-helpers` return promises (i.e. `click`, `andThen`, `visit`, etc). Waiters run periodically
after each helper has executed until a predetermined condition is met. After the waiters finish, the next async helper
is executed and the process repeats.

This allows the test suite to pause at deterministic intervals, and helps thread together the async nature of tests.

Test waiters can be added to application code to provide the necessary integration with the test suite. The waiters will
function as intended in development mode, and we strip the vast majority of the functionality in production mode so as to
minimize negative performance impact.

This addon implements the design specified in [RFC 581](https://github.com/emberjs/rfcs/blob/master/text/0581-new-test-waiters.md).

## Table of Contents

- [Compatibility](#compatibility)
- [Installation](#installation)
- [Quickstart](#quickstart)
  - [buildWaiter function](#buildwaiter-function)
  - [waitForPromise function](#waitforpromise-function)
  - [Waiting on all waiters](#waiting-on-all-waiters)
- [General Design](#general-design)
  - [Comparison of old waiters system to new](#comparison-of-old-waiters-system-to-new)
  - [New Test Waiters Design](#new-test-waiters-design)
  - [`Waiter`](#waiter)
  - [`TestWaiter`](#testwaiter)
  - [Using the `TestWaiter` class](#using-the-testwaiter-class)
  - [`waitForPromise`](#waitforpromise)
- [Contributing](#contributing)
- [License](#license)

## Compatibility

- Ember.js v2.18 or above
- Ember CLI v2.13 or above

## Installation

```
ember install ember-test-waiters
```

## Quickstart

`ember-test-waiters` uses a minimal API to provide waiting functionality. This minimal API can be composed to accommodate various complex scenarios.

### buildWaiter function

The `buildWaiter` function is, in most cases, all you will need to wait for async operations to complete before continuing tests. It returns a waiter instance
that provides a number of methods. The key methods that allow you to control async behavior are `beginAsync` and `endAsync`, which are expected to be called as
a pair to _begin_ waiting and _end_ waiting respectively. The `beginAsync` method returns a `token`, which uniquely identifies that async operation. To mark the
async operation as complete, call `endAsync`, passing in the `token` that was returned from the prior `beginAsync` call.

```js
import Component from '@ember/component';
import { buildWaiter } from 'ember-test-waiters';

let waiter = buildWaiter('friend-waiter');

export default class Friendz extends Component {
  didInsertElement() {
    let token = waiter.beginAsync();

    someAsyncWork()
      .then(() => {
        //... some work
      })
      .finally(() => {
        waiter.endAsync(token);
      });
  }
}
```

### waitForPromise function

This addon also provides a `waitForPromise` function, which can be used to wrap a promise to register it with the test waiter system. _Note_: the
`waitForPromise` function will ensure that `endAsync` is called correctly in the `finally` call of your promise.

```js
import Component from '@ember/component';
import { waitForPromise } from 'ember-test-waiters';

export default class MoreFriendz extends Component {
  didInsertElement() {
    waitForPromise(someAsyncWork).then(() => {
      doOtherThings();
    });
  }
}
```

### Waiting on all waiters

The `ember-test-waiters` addon provides a `waiter-manager` to register, unregister, iterate and invoke waiters to determine if we should wait for conditions to be met or continue test execution. This functionality is encapsulated in the `hasPendingWaiters` function, which evaluates each registered waiter to determine its current state.

```js
import { hasPendingWaiters } from 'ember-test-waiters';

// ...

// true if waiters are pending, allowing us to still wait for async to complete
let hasPendingWaiters = hasPendingWaiters();

// ...
```

## General Design

Ember’s test framework has an internal concept of settledness, that is used by all of its internal helpers. Settledness can be defined as **_all known active async operations have completed, and there’s no outstanding work to be done_**. This is codified in the [settled](https://github.com/emberjs/ember-test-helpers/blob/master/API.md#settled) helper.

The settled helper, as noted, wires itself up to known asynchronous behaviors. Those include whether there

- is an active runloop (more on the runloop)
- are any pending timers within the runloop (run.later, run.debounce, run.throttle)
- are any pending test waiters (more on waiters later!)
- are any pending `jQuery.ajax` requests
- are any pending route transitions.

The settled check returns a Promise that is fulfilled when all of the above behaviors return `false`, indicating all async for each behavior is completed. These cover a vast number of async behaviors that are typical in our applications.

An enhancement was added to [`ember-qunit`](https://github.com/emberjs/ember-qunit) that allowed for detecting a lack of settledness at the end of a test. This enhancement, [test isolation validation](https://github.com/emberjs/ember-qunit/blob/master/docs/TEST_ISOLATION_VALIDATION.md), evaluated the settled state once a test was considered done and reported to the user whether there were active async operations pending. This helped developers quickly identify and fix known asynchronous leaks in their tests, allowing for a more deterministic test suite.

During the development of the test isolation validation feature, we discovered that most asynchronous operations used in the settled check provided good debug information that could be provided to the end user, with the exception of the existing test waiters. Those waiters only provided rudimentary information that could be exposed, specifically whether there were any active test waiters pending, but nothing more.

To address this, a new addon was written to experiment on a new test waiter system that would provide a number of things (as noted above):

1. A new API that's _explicit_ and _straightforward_
1. A more robust way to gather debugging information for the test waiter
1. Default test waiters with the ability to author your own, more complex test waiters

This allows developers to utilize `ember-test-waiters` to annotate their asynchronous operations that are not tracked by an `await settled()` check, and for those annotations to provide useful debugging information in the event their async extended past the expected duration of the test.

## Comparison of old waiters system to new

In the old test waiters system, you would do the following:

```js
import { registerWaiter } from '@ember/test';

registerWaiter(function() {
  return myPendingTransactions() === 0;
});
```

While reading the above is straightforward, when writing a test waiter using the old system it's easy to forget what the expected return value is: `true` or `false`. Additionally, it's a bit more cognitive overhead to derive what the intended result of the particular boolean return value is: does returning `true` result in the test waiter waiting or not?

As mentioned before, there's no additional information provided via `registerWaiter`, and capturing stack traces at the call site is currently not implemented. Unmanaged async that 'hangs' can cause your tests to stall and ultimately timeout. Not having stack traces is particularly problematic when trying to identify which of many test waiters has caused this timeout, as it's like looking for a needle in a haystack.

The new system captures an error object when the waiter's `beginAsync` method is called (more on `beginAsync` later), but evaluates the `stack` property lazily, when this value is processed by `@ember/test-helpers`' `getSettledState`. This allows for identifying the offending code more easily.

The new test waiters system looks like this:

```js
import Component from '@ember/component';
import { buildWaiter } from 'ember-test-waiters';

let waiter = buildWaiter('friend-waiter');

export default class Friendz extends Component {
  didInsertElement() {
    let token = waiter.beginAsync();

    someAsyncWork()
      .then(() => {
        //... some work
      })
      .finally(() => {
        waiter.endAsync(token);
      });
  }
}
```

In the above example, a new test waiter is built that is identified via a `name` string passed into the `buildWaiter` function. This allows the waiter to be identifiable, and that name is ultimately used with test isolation validation to help developers narrow down problems in their tests.

## New Test Waiters Design

The new test waiters addon is built using low-level primitives that are complimented with some convenience utilities.

### [`Waiter`](addon/types/index.ts#L24)

At its core, the addon uses an `Waiter` interface defined as follows:

```ts
export type WaiterName = string;
export type Token = unknown;

export interface Waiter {
  name: WaiterName;
  waitUntil(): boolean;
  debugInfo(): TestWaiterDebugInfo[];
}
```

- `name`: The name of the test waiter, which is used to help identify it in test isolation validation output.
- `waitUntil`: Used to determine if the waiter system should still wait for async
  operations to complete. The `waitUntil` method will return `true` to signal completion.
- `debugInfo`: Returns the `debugInfo` for each item tracking async operations in a waiter. The `debugInfo` for each waiter item is ultimately used in `@ember/test-helpers`' `getSettledState` function, which is used for test isolation validation output.

This allows for maximum flexibility when creating your own waiter implementations.

### [`TestWaiter`](addon/types/index.ts#L57)

The `Waiter` interface is built upon to create a more specific interface for a test waiter, `TestWaiter`:

```ts
export interface TestWaiter<T extends object | Primitive | unknown = Token> extends Waiter {
  beginAsync(token?: T, label?: string): T;
  endAsync(token: T): void;
  reset(): void;
}
```

- `beginAsync`: Should be used to signal the beginning of an async operation that
  is to be waited for. Invocation of this method should be paired with a subsequent
  `endAsync` call to indicate to the waiter system that the async operation is completed.
- `endAsync`: Should be used to signal the end of an async operation. Invocation of this
  method should be paired with a preceding `beginAsync` call, which would indicate the
  beginning of an async operation.
- `reset`: Resets the waiter state, clearing items tracking async operations in this waiter.

This interface is used for the concrete `TestWaiter` type. This type forms the basis for the addon, and will likely satisfy the majority of use cases.

The most common practice is to import and invoke the `buildWaiter` function to create a new test waiter. The recommendation is to do so at the module level, which allows a single waiter to be created per type (this should likely be enforced via a lint rule added to `eslint-plugin-ember`). A single waiter is then usable across multiple instances.

```ts
function buildWaiter(name: string): TestWaiter;
```

In anything but a production build, this function will return a `TestWaiter` instance. When in production mode, we make this instance _inert_ and essentially no cost to invoke. Since test waiters are intended to be called from application or addon code, but are only required to be _active_ when in tests, this process of making the instance _inert_ is important. Even though code is still invoked, this has a negligible impact on performance.

### Using the Test Waiter

After building a test waiter, most users interact with a limited set of methods within this class, namely `beingAsync` and `endAsync`.

The API used to signal whether an asynchronous operation has begun and ultimately ended is through the **_paired_** calls of `beginAsync` and `endAsync`: begin to denote the start of the asynchronous operation, and end to denote the end. Unique instances of async operations are identified using a `token` returned from `beginAsync`, which is subsequently provided to the `endAsync` call.

To annotate the example provided above:

```js
import Component from '@ember/component';
import { buildWaiter } from 'ember-test-waiters';

// Creates a test waiter with the name 'friend-waiter' that
// is usable by all instances of the `Friendz` component.
let waiter = buildWaiter('friend-waiter');

export default class Friendz extends Component {
  didInsertElement() {
    // Alerts the test waiter system that an async operation has started,
    // storing the resulting unique token to be used to notify the test
    // waiter system that the operation has ended.
    let token = waiter.beginAsync();

    someAsyncWork()
      .then(() => {
        //... some work
      })
      .finally(() => {
        // Notifies the test waiter system that
        // this unique async operation has ended.
        waiter.endAsync(token);
      });
  }
}
```

### `waitForPromise`

The `waitForPromise` utility provides a convenience wrapper around the `TestWaiter` class for use with promises. It ensures the `endAsync` call is invoked in the `finally` of the configured promise.

```js
import Component from '@ember/component';
import { waitForPromise } from 'ember-test-waiters';

export default class MoreFriendz extends Component {
  didInsertElement() {
    waitForPromise(someAsyncWork).then(() => {
      doOtherThings();
    });
  }
}
```

This new test waiters system has been through multiple iterations of refinement, and is in use and integrated with the test isolation validation system.

## Contributing

See the [Contributing](CONTRIBUTING.md) guide for details.

## License

This project is licensed under the [MIT License](LICENSE.md).
