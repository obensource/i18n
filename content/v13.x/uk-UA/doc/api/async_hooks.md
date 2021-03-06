# Асинхронні хуки

<!--introduced_in=v8.1.0-->

> Stability: 1 - Experimental

The `async_hooks` module provides an API to track asynchronous resources. It can be accessed using:

```js
const async_hooks = require('async_hooks');
```

## Terminology

An asynchronous resource represents an object with an associated callback. This callback may be called multiple times, for example, the `'connection'` event in `net.createServer()`, or just a single time like in `fs.open()`. A resource can also be closed before the callback is called. `AsyncHook` does not explicitly distinguish between these different cases but will represent them as the abstract concept that is a resource.

If [`Worker`][]s are used, each thread has an independent `async_hooks` interface, and each thread will use a new set of async IDs.

## Public API

### Overview

Following is a simple overview of the public API.

```js
const async_hooks = require('async_hooks');

// Return the ID of the current execution context.
const eid = async_hooks.executionAsyncId();

// Return the ID of the handle responsible for triggering the callback of the
// current execution scope to call.
const tid = async_hooks.triggerAsyncId();

// Create a new AsyncHook instance. All of these callbacks are optional.
const asyncHook =
    async_hooks.createHook({ init, before, after, destroy, promiseResolve });

// Allow callbacks of this AsyncHook instance to call. This is not an implicit
// action after running the constructor, and must be explicitly run to begin
// executing callbacks.
asyncHook.enable();

// Disable listening for new asynchronous events.
asyncHook.disable();

//
// The following are the callbacks that can be passed to createHook().
//

// init is called during object construction. The resource may not have
// completed construction when this callback runs, therefore all fields of the
// resource referenced by "asyncId" may not have been populated.
function init(asyncId, type, triggerAsyncId, resource) { }

// Before is called just before the resource's callback is called. It can be
// called 0-N times for handles (e.g. TCPWrap), and will be called exactly 1
// time for requests (e.g. FSReqCallback).
function before(asyncId) { }

// After is called just after the resource's callback has finished.
function after(asyncId) { }

// Destroy is called when an AsyncWrap instance is destroyed.
function destroy(asyncId) { }

// promiseResolve is called only for promise resources, when the
// `resolve` function passed to the `Promise` constructor is invoked
// (either directly or through other means of resolving a promise).
function promiseResolve(asyncId) { }
```

#### `async_hooks.createHook(callbacks)`

<!-- YAML
added: v8.1.0
-->

* `callbacks` {Object} The [Hook Callbacks](#async_hooks_hook_callbacks) to register
  * `init` {Function} The [`init` callback][].
  * `before` {Function} The [`before` callback][].
  * `after` {Function} The [`after` callback][].
  * `destroy` {Function} The [`destroy` callback][].
  * `promiseResolve` {Function} The [`promiseResolve` callback][].
* Returns: {AsyncHook} Instance used for disabling and enabling hooks

Registers functions to be called for different lifetime events of each async operation.

The callbacks `init()`/`before()`/`after()`/`destroy()` are called for the respective asynchronous event during a resource's lifetime.

All callbacks are optional. For example, if only resource cleanup needs to be tracked, then only the `destroy` callback needs to be passed. The specifics of all functions that can be passed to `callbacks` is in the [Hook Callbacks](#async_hooks_hook_callbacks) section.

```js
const async_hooks = require('async_hooks');

const asyncHook = async_hooks.createHook({
  init(asyncId, type, triggerAsyncId, resource) { },
  destroy(asyncId) { }
});
```

The callbacks will be inherited via the prototype chain:

```js
class MyAsyncCallbacks {
  init(asyncId, type, triggerAsyncId, resource) { }
  destroy(asyncId) {}
}

class MyAddedCallbacks extends MyAsyncCallbacks {
  before(asyncId) { }
  after(asyncId) { }
}

const asyncHook = async_hooks.createHook(new MyAddedCallbacks());
```

##### Error Handling

If any `AsyncHook` callbacks throw, the application will print the stack trace and exit. The exit path does follow that of an uncaught exception, but all `'uncaughtException'` listeners are removed, thus forcing the process to exit. The `'exit'` callbacks will still be called unless the application is run with `--abort-on-uncaught-exception`, in which case a stack trace will be printed and the application exits, leaving a core file.

The reason for this error handling behavior is that these callbacks are running at potentially volatile points in an object's lifetime, for example during class construction and destruction. Because of this, it is deemed necessary to bring down the process quickly in order to prevent an unintentional abort in the future. This is subject to change in the future if a comprehensive analysis is performed to ensure an exception can follow the normal control flow without unintentional side effects.

##### Printing in AsyncHooks callbacks

Because printing to the console is an asynchronous operation, `console.log()` will cause the AsyncHooks callbacks to be called. Using `console.log()` or similar asynchronous operations inside an AsyncHooks callback function will thus cause an infinite recursion. An easy solution to this when debugging is to use a synchronous logging operation such as `fs.writeFileSync(file, msg, flag)`. This will print to the file and will not invoke AsyncHooks recursively because it is synchronous.

```js
const fs = require('fs');
const util = require('util');

function debug(...args) {
  // Use a function like this one when debugging inside an AsyncHooks callback
  fs.writeFileSync('log.out', `${util.format(...args)}\n`, { flag: 'a' });
}
```

If an asynchronous operation is needed for logging, it is possible to keep track of what caused the asynchronous operation using the information provided by AsyncHooks itself. The logging should then be skipped when it was the logging itself that caused AsyncHooks callback to call. By doing this the otherwise infinite recursion is broken.

### Class: `AsyncHook`

The class `AsyncHook` exposes an interface for tracking lifetime events of asynchronous operations.

#### `asyncHook.enable()`

* Returns: {AsyncHook} A reference to `asyncHook`.

Enable the callbacks for a given `AsyncHook` instance. If no callbacks are provided enabling is a noop.

The `AsyncHook` instance is disabled by default. If the `AsyncHook` instance should be enabled immediately after creation, the following pattern can be used.

```js
const async_hooks = require('async_hooks');

const hook = async_hooks.createHook(callbacks).enable();
```

#### `asyncHook.disable()`

* Returns: {AsyncHook} A reference to `asyncHook`.

Disable the callbacks for a given `AsyncHook` instance from the global pool of `AsyncHook` callbacks to be executed. Once a hook has been disabled it will not be called again until enabled.

For API consistency `disable()` also returns the `AsyncHook` instance.

#### Hook Callbacks

Key events in the lifetime of asynchronous events have been categorized into four areas: instantiation, before/after the callback is called, and when the instance is destroyed.

##### `init(asyncId, type, triggerAsyncId, resource)`

* `asyncId` {number} A unique ID for the async resource.
* `type` {string} The type of the async resource.
* `triggerAsyncId` {number} The unique ID of the async resource in whose execution context this async resource was created.
* `resource` {Object} Reference to the resource representing the async operation, needs to be released during _destroy_.

Called when a class is constructed that has the _possibility_ to emit an asynchronous event. This _does not_ mean the instance must call `before`/`after` before `destroy` is called, only that the possibility exists.

This behavior can be observed by doing something like opening a resource then closing it before the resource can be used. The following snippet demonstrates this.

```js
require('net').createServer().listen(function() { this.close(); });
// OR
clearTimeout(setTimeout(() => {}, 10));
```

Every new resource is assigned an ID that is unique within the scope of the current Node.js instance.

###### `type`

The `type` is a string identifying the type of resource that caused `init` to be called. Generally, it will correspond to the name of the resource's constructor.

```text
FSEVENTWRAP, FSREQCALLBACK, GETADDRINFOREQWRAP, GETNAMEINFOREQWRAP, HTTPINCOMINGMESSAGE,
HTTPCLIENTREQUEST, JSSTREAM, PIPECONNECTWRAP, PIPEWRAP, PROCESSWRAP, QUERYWRAP,
SHUTDOWNWRAP, SIGNALWRAP, STATWATCHER, TCPCONNECTWRAP, TCPSERVERWRAP, TCPWRAP,
TTYWRAP, UDPSENDWRAP, UDPWRAP, WRITEWRAP, ZLIB, SSLCONNECTION, PBKDF2REQUEST,
RANDOMBYTESREQUEST, TLSWRAP, Microtask, Timeout, Immediate, TickObject
```

There is also the `PROMISE` resource type, which is used to track `Promise` instances and asynchronous work scheduled by them.

Users are able to define their own `type` when using the public embedder API.

It is possible to have type name collisions. Embedders are encouraged to use unique prefixes, such as the npm package name, to prevent collisions when listening to the hooks.

###### `triggerAsyncId`

`triggerAsyncId` is the `asyncId` of the resource that caused (or "triggered") the new resource to initialize and that caused `init` to call. This is different from `async_hooks.executionAsyncId()` that only shows *when* a resource was created, while `triggerAsyncId` shows *why* a resource was created.

The following is a simple demonstration of `triggerAsyncId`:

```js
async_hooks.createHook({
  init(asyncId, type, triggerAsyncId) {
    const eid = async_hooks.executionAsyncId();
    fs.writeSync(
      1, `${type}(${asyncId}): trigger: ${triggerAsyncId} execution: ${eid}\n`);
  }
}).enable();

require('net').createServer((conn) => {}).listen(8080);
```

Output when hitting the server with `nc localhost 8080`:

```console
TCPSERVERWRAP(5): trigger: 1 execution: 1
TCPWRAP(7): trigger: 5 execution: 0
```

The `TCPSERVERWRAP` is the server which receives the connections.

The `TCPWRAP` is the new connection from the client. When a new connection is made, the `TCPWrap` instance is immediately constructed. This happens outside of any JavaScript stack. (An `executionAsyncId()` of `0` means that it is being executed from C++ with no JavaScript stack above it.) With only that information, it would be impossible to link resources together in terms of what caused them to be created, so `triggerAsyncId` is given the task of propagating what resource is responsible for the new resource's existence.

###### `resource`

`resource` is an object that represents the actual async resource that has been initialized. This can contain useful information that can vary based on the value of `type`. For instance, for the `GETADDRINFOREQWRAP` resource type, `resource` provides the host name used when looking up the IP address for the host in `net.Server.listen()`. The API for accessing this information is currently not considered public, but using the Embedder API, users can provide and document their own resource objects. For example, such a resource object could contain the SQL query being executed.

In the case of Promises, the `resource` object will have an `isChainedPromise` property, set to `true` if the promise has a parent promise, and `false` otherwise. For example, in the case of `b = a.then(handler)`, `a` is considered a parent `Promise` of `b`. Here, `b` is considered a chained promise.

In some cases the resource object is reused for performance reasons, it is thus not safe to use it as a key in a `WeakMap` or add properties to it.

###### Asynchronous context example

The following is an example with additional information about the calls to `init` between the `before` and `after` calls, specifically what the callback to `listen()` will look like. The output formatting is slightly more elaborate to make calling context easier to see.

```js
let indent = 0;
async_hooks.createHook({
  init(asyncId, type, triggerAsyncId) {
    const eid = async_hooks.executionAsyncId();
    const indentStr = ' '.repeat(indent);
    fs.writeSync(
      1,
      `${indentStr}${type}(${asyncId}):` +
      ` trigger: ${triggerAsyncId} execution: ${eid}\n`);
  },
  before(asyncId) {
    const indentStr = ' '.repeat(indent);
    fs.writeFileSync('log.out',
                     `${indentStr}before:  ${asyncId}\n`, { flag: 'a' });
    indent += 2;
  },
  after(asyncId) {
    indent -= 2;
    const indentStr = ' '.repeat(indent);
    fs.writeFileSync('log.out',
                     `${indentStr}after:  ${asyncId}\n`, { flag: 'a' });
  },
  destroy(asyncId) {
    const indentStr = ' '.repeat(indent);
    fs.writeFileSync('log.out',
                     `${indentStr}destroy:  ${asyncId}\n`, { flag: 'a' });
  },
}).enable();

require('net').createServer(() => {}).listen(8080, () => {
  // Let's wait 10ms before logging the server started.
  setTimeout(() => {
    console.log('>>>', async_hooks.executionAsyncId());
  }, 10);
});
```

Output from only starting the server:

```console
TCPSERVERWRAP(5): trigger: 1 execution: 1
TickObject(6): trigger: 5 execution: 1
before:  6
  Timeout(7): trigger: 6 execution: 6
after:   6
destroy: 6
before:  7
>>> 7
  TickObject(8): trigger: 7 execution: 7
after:   7
before:  8
after:   8
```

As illustrated in the example, `executionAsyncId()` and `execution` each specify the value of the current execution context; which is delineated by calls to `before` and `after`.

Only using `execution` to graph resource allocation results in the following:

```console
Timeout(7) -> TickObject(6) -> root(1)
```

The `TCPSERVERWRAP` is not part of this graph, even though it was the reason for `console.log()` being called. This is because binding to a port without a host name is a *synchronous* operation, but to maintain a completely asynchronous API the user's callback is placed in a `process.nextTick()`.

The graph only shows *when* a resource was created, not *why*, so to track the *why* use `triggerAsyncId`.

##### `before(asyncId)`

* `asyncId` {number}

When an asynchronous operation is initiated (such as a TCP server receiving a new connection) or completes (such as writing data to disk) a callback is called to notify the user. The `before` callback is called just before said callback is executed. `asyncId` is the unique identifier assigned to the resource about to execute the callback.

The `before` callback will be called 0 to N times. The `before` callback will typically be called 0 times if the asynchronous operation was cancelled or, for example, if no connections are received by a TCP server. Persistent asynchronous resources like a TCP server will typically call the `before` callback multiple times, while other operations like `fs.open()` will call it only once.

##### `after(asyncId)`

* `asyncId` {number}

Called immediately after the callback specified in `before` is completed.

If an uncaught exception occurs during execution of the callback, then `after` will run *after* the `'uncaughtException'` event is emitted or a `domain`'s handler runs.

##### `destroy(asyncId)`

* `asyncId` {number}

Called after the resource corresponding to `asyncId` is destroyed. It is also called asynchronously from the embedder API `emitDestroy()`.

Some resources depend on garbage collection for cleanup, so if a reference is made to the `resource` object passed to `init` it is possible that `destroy` will never be called, causing a memory leak in the application. If the resource does not depend on garbage collection, then this will not be an issue.

##### `promiseResolve(asyncId)`

<!-- YAML
added: v8.6.0
-->

* `asyncId` {number}

Called when the `resolve` function passed to the `Promise` constructor is invoked (either directly or through other means of resolving a promise).

`resolve()` does not do any observable synchronous work.

The `Promise` is not necessarily fulfilled or rejected at this point if the `Promise` was resolved by assuming the state of another `Promise`.

```js
new Promise((resolve) => resolve(true)).then((a) => {});
```

calls the following callbacks:

```text
init for PROMISE with id 5, trigger id: 1
  promise resolve 5      # corresponds to resolve(true)
init for PROMISE with id 6, trigger id: 5  # the Promise returned by then()
  before 6               # the then() callback is entered
  promise resolve 6      # the then() callback resolves the promise by returning
  after 6
```

#### `async_hooks.executionAsyncResource()`

<!-- YAML
added: v13.9.0
-->

* Returns: {Object} The resource representing the current execution. Useful to store data within the resource.

Resource objects returned by `executionAsyncResource()` are most often internal Node.js handle objects with undocumented APIs. Using any functions or properties on the object is likely to crash your application and should be avoided.

Using `executionAsyncResource()` in the top-level execution context will return an empty object as there is no handle or request object to use, but having an object representing the top-level can be helpful.

```js
const { open } = require('fs');
const { executionAsyncId, executionAsyncResource } = require('async_hooks');

console.log(executionAsyncId(), executionAsyncResource());  // 1 {}
open(__filename, 'r', (err, fd) => {
  console.log(executionAsyncId(), executionAsyncResource());  // 7 FSReqWrap
});
```

This can be used to implement continuation local storage without the use of a tracking `Map` to store the metadata:

```js
const { createServer } = require('http');
const {
  executionAsyncId,
  executionAsyncResource,
  createHook
} = require('async_hooks');
const sym = Symbol('state'); // Private symbol to avoid pollution

createHook({
  init(asyncId, type, triggerAsyncId, resource) {
    const cr = executionAsyncResource();
    if (cr) {
      resource[sym] = cr[sym];
    }
  }
}).enable();

const server = createServer(function(req, res) {
  executionAsyncResource()[sym] = { state: req.url };
  setTimeout(function() {
    res.end(JSON.stringify(executionAsyncResource()[sym]));
  }, 100);
}).listen(3000);
```

#### `async_hooks.executionAsyncId()`

<!-- YAML
added: v8.1.0
changes:
  - version: v8.2.0
    pr-url: https://github.com/nodejs/node/pull/13490
    description: Renamed from `currentId`
-->

* Returns: {number} The `asyncId` of the current execution context. Useful to track when something calls.

```js
const async_hooks = require('async_hooks');

console.log(async_hooks.executionAsyncId());  // 1 - bootstrap
fs.open(path, 'r', (err, fd) => {
  console.log(async_hooks.executionAsyncId());  // 6 - open()
});
```

The ID returned from `executionAsyncId()` is related to execution timing, not causality (which is covered by `triggerAsyncId()`):

```js
const server = net.createServer((conn) => {
  // Returns the ID of the server, not of the new connection, because the
  // callback runs in the execution scope of the server's MakeCallback().
  async_hooks.executionAsyncId();

}).listen(port, () => {
  // Returns the ID of a TickObject (i.e. process.nextTick()) because all
  // callbacks passed to .listen() are wrapped in a nextTick().
  async_hooks.executionAsyncId();
});
```

Promise contexts may not get precise `executionAsyncIds` by default. See the section on [promise execution tracking](#async_hooks_promise_execution_tracking).

#### `async_hooks.triggerAsyncId()`

* Returns: {number} The ID of the resource responsible for calling the callback that is currently being executed.

```js
const server = net.createServer((conn) => {
  // The resource that caused (or triggered) this callback to be called
  // was that of the new connection. Thus the return value of triggerAsyncId()
  // is the asyncId of "conn".
  async_hooks.triggerAsyncId();

}).listen(port, () => {
  // Even though all callbacks passed to .listen() are wrapped in a nextTick()
  // the callback itself exists because the call to the server's .listen()
  // was made. So the return value would be the ID of the server.
  async_hooks.triggerAsyncId();
});
```

Promise contexts may not get valid `triggerAsyncId`s by default. See the section on [promise execution tracking](#async_hooks_promise_execution_tracking).

## Promise execution tracking

By default, promise executions are not assigned `asyncId`s due to the relatively expensive nature of the [promise introspection API](https://docs.google.com/document/d/1rda3yKGHimKIhg5YeoAmCOtyURgsbTH_qaYR79FELlk/edit) provided by V8. This means that programs using promises or `async`/`await` will not get correct execution and trigger ids for promise callback contexts by default.

```js
const ah = require('async_hooks');
Promise.resolve(1729).then(() => {
  console.log(`eid ${ah.executionAsyncId()} tid ${ah.triggerAsyncId()}`);
});
// produces:
// eid 1 tid 0
```

Observe that the `then()` callback claims to have executed in the context of the outer scope even though there was an asynchronous hop involved. Also, the `triggerAsyncId` value is `0`, which means that we are missing context about the resource that caused (triggered) the `then()` callback to be executed.

Installing async hooks via `async_hooks.createHook` enables promise execution tracking:

```js
const ah = require('async_hooks');
ah.createHook({ init() {} }).enable(); // forces PromiseHooks to be enabled.
Promise.resolve(1729).then(() => {
  console.log(`eid ${ah.executionAsyncId()} tid ${ah.triggerAsyncId()}`);
});
// produces:
// eid 7 tid 6
```

In this example, adding any actual hook function enabled the tracking of promises. There are two promises in the example above; the promise created by `Promise.resolve()` and the promise returned by the call to `then()`. In the example above, the first promise got the `asyncId` `6` and the latter got `asyncId` `7`. During the execution of the `then()` callback, we are executing in the context of promise with `asyncId` `7`. This promise was triggered by async resource `6`.

Another subtlety with promises is that `before` and `after` callbacks are run only on chained promises. That means promises not created by `then()`/`catch()` will not have the `before` and `after` callbacks fired on them. For more details see the details of the V8 [PromiseHooks](https://docs.google.com/document/d/1rda3yKGHimKIhg5YeoAmCOtyURgsbTH_qaYR79FELlk/edit) API.

## JavaScript Embedder API

Library developers that handle their own asynchronous resources performing tasks like I/O, connection pooling, or managing callback queues may use the `AsyncWrap` JavaScript API so that all the appropriate callbacks are called.

### Class: `AsyncResource`

The class `AsyncResource` is designed to be extended by the embedder's async resources. Using this, users can easily trigger the lifetime events of their own resources.

The `init` hook will trigger when an `AsyncResource` is instantiated.

The following is an overview of the `AsyncResource` API.

```js
const { AsyncResource, executionAsyncId } = require('async_hooks');

// AsyncResource() is meant to be extended. Instantiating a
// new AsyncResource() also triggers init. If triggerAsyncId is omitted then
// async_hook.executionAsyncId() is used.
const asyncResource = new AsyncResource(
  type, { triggerAsyncId: executionAsyncId(), requireManualDestroy: false }
);

// Run a function in the execution context of the resource. This will
// * establish the context of the resource
// * trigger the AsyncHooks before callbacks
// * call the provided function `fn` with the supplied arguments
// * trigger the AsyncHooks after callbacks
// * restore the original execution context
asyncResource.runInAsyncScope(fn, thisArg, ...args);

// Call AsyncHooks destroy callbacks.
asyncResource.emitDestroy();

// Return the unique ID assigned to the AsyncResource instance.
asyncResource.asyncId();

// Return the trigger ID for the AsyncResource instance.
asyncResource.triggerAsyncId();
```

#### `new AsyncResource(type[, options])`

* `type` {string} The type of async event.
* `options` {Object}
  * `triggerAsyncId` {number} The ID of the execution context that created this async event. **Default:** `executionAsyncId()`.
  * `requireManualDestroy` {boolean} Disables automatic `emitDestroy` when the object is garbage collected. This usually does not need to be set (even if `emitDestroy` is called manually), unless the resource's `asyncId` is retrieved and the sensitive API's `emitDestroy` is called with it. **Default:** `false`.

Example usage:

```js
class DBQuery extends AsyncResource {
  constructor(db) {
    super('DBQuery');
    this.db = db;
  }

  getInfo(query, callback) {
    this.db.get(query, (err, data) => {
      this.runInAsyncScope(callback, null, err, data);
    });
  }

  close() {
    this.db = null;
    this.emitDestroy();
  }
}
```

#### `asyncResource.runInAsyncScope(fn[, thisArg, ...args])`
<!-- YAML
added: v9.6.0
-->

* `fn` {Function} The function to call in the execution context of this async resource.
* `thisArg` {any} The receiver to be used for the function call.
* `...args` {any} Optional arguments to pass to the function.

Call the provided function with the provided arguments in the execution context of the async resource. This will establish the context, trigger the AsyncHooks before callbacks, call the function, trigger the AsyncHooks after callbacks, and then restore the original execution context.

#### `asyncResource.emitDestroy()`

* Returns: {AsyncResource} A reference to `asyncResource`.

Call all `destroy` hooks. This should only ever be called once. An error will be thrown if it is called more than once. This **must** be manually called. If the resource is left to be collected by the GC then the `destroy` hooks will never be called.

#### `asyncResource.asyncId()`

* Returns: {number} The unique `asyncId` assigned to the resource.

#### `asyncResource.triggerAsyncId()`

* Returns: {number} The same `triggerAsyncId` that is passed to the `AsyncResource` constructor.

<a id="async-resource-worker-pool"></a>

### Using `AsyncResource` for a `Worker` thread pool

The following example shows how to use the `AsyncResource` class to properly provide async tracking for a [`Worker`][] pool. Other resource pools, such as database connection pools, can follow a similar model.

Assuming that the task is adding two numbers, using a file named `task_processor.js` with the following content:

```js
const { parentPort } = require('worker_threads');
parentPort.on('message', (task) => {
  parentPort.postMessage(task.a + task.b);
});
```

a Worker pool around it could use the following structure:

```js
const { AsyncResource } = require('async_hooks');
const { EventEmitter } = require('events');
const path = require('path');
const { Worker } = require('worker_threads');

const kTaskInfo = Symbol('kTaskInfo');
const kWorkerFreedEvent = Symbol('kWorkerFreedEvent');

class WorkerPoolTaskInfo extends AsyncResource {
  constructor(callback) {
    super('WorkerPoolTaskInfo');
    this.callback = callback;
  }

  done(err, result) {
    this.runInAsyncScope(this.callback, null, err, result);
    this.emitDestroy();  // `TaskInfo`s are used only once.
  }
}

class WorkerPool extends EventEmitter {
  constructor(numThreads) {
    super();
    this.numThreads = numThreads;
    this.workers = [];
    this.freeWorkers = [];

    for (let i = 0; i < numThreads; i++)
      this.addNewWorker();
  }

  addNewWorker() {
    const worker = new Worker(path.resolve(__dirname, 'task_processor.js'));
    worker.on('message', (result) => {
      // In case of success: Call the callback that was passed to `runTask`,
      // remove the `TaskInfo` associated with the Worker, and mark it as free
      // again.
      worker[kTaskInfo].done(null, result);
      worker[kTaskInfo] = null;
      this.freeWorkers.push(worker);
      this.emit(kWorkerFreedEvent);
    });
    worker.on('error', (err) => {
      // In case of an uncaught exception: Call the callback that was passed to
      // `runTask` with the error.
      if (worker[kTaskInfo])
        worker[kTaskInfo].done(err, null);
      else
        this.emit('error', err);
      // Remove the worker from the list and start a new Worker to replace the
      // current one.
      this.workers.splice(this.workers.indexOf(worker), 1);
      this.addNewWorker();
    });
    this.workers.push(worker);
    this.freeWorkers.push(worker);
  }

  runTask(task, callback) {
    if (this.freeWorkers.length === 0) {
      // No free threads, wait until a worker thread becomes free.
      this.once(kWorkerFreedEvent, () => this.runTask(task, callback));
      return;
    }

    const worker = this.freeWorkers.pop();
    worker[kTaskInfo] = new WorkerPoolTaskInfo(callback);
    worker.postMessage(task);
  }

  close() {
    for (const worker of this.workers) worker.terminate();
  }
}

module.exports = WorkerPool;
```

Without the explicit tracking added by the `WorkerPoolTaskInfo` objects, it would appear that the callbacks are associated with the individual `Worker` objects. However, the creation of the `Worker`s is not associated with the creation of the tasks and does not provide information about when tasks were scheduled.

This pool could be used as follows:

```js
const WorkerPool = require('./worker_pool.js');
const os = require('os');

const pool = new WorkerPool(os.cpus().length);

let finished = 0;
for (let i = 0; i < 10; i++) {
  pool.runTask({ a: 42, b: 100 }, (err, result) => {
    console.log(i, err, result);
    if (++finished === 10)
      pool.close();
  });
}
```

## Class: `AsyncLocalStorage`
<!-- YAML
added: v13.10.0
-->

This class is used to create asynchronous state within callbacks and promise chains. It allows storing data throughout the lifetime of a web request or any other asynchronous duration. It is similar to thread-local storage in other languages.

The following example builds a logger that will always know the current HTTP request and uses it to display enhanced logs without needing to explicitly provide the current HTTP request to it.

```js
const { AsyncLocalStorage } = require('async_hooks');
const http = require('http');

const kReq = 'CURRENT_REQUEST';
const asyncLocalStorage = new AsyncLocalStorage();

function log(...args) {
  const store = asyncLocalStorage.getStore();
  // Make sure the store exists and it contains a request.
  if (store && store.has(kReq)) {
    const req = store.get(kReq);
    // Prints `GET /items ERR could not do something
    console.log(req.method, req.url, ...args);
  } else {
    console.log(...args);
  }
}

http.createServer((request, response) => {
  asyncLocalStorage.run(new Map(), () => {
    const store = asyncLocalStorage.getStore();
    store.set(kReq, request);
    someAsyncOperation((err, result) => {
      if (err) {
        log('ERR', err.message);
      }
    });
  });
})
.listen(8080);
```

When having multiple instances of `AsyncLocalStorage`, they are independent from each other. It is safe to instantiate this class multiple times.

### `new AsyncLocalStorage()`
<!-- YAML
added: v13.10.0
-->

Creates a new instance of `AsyncLocalStorage`. Store is only provided within a `run` or a `runSyncAndReturn` method call.

### `asyncLocalStorage.disable()`
<!-- YAML
added: v13.10.0
-->

This method disables the instance of `AsyncLocalStorage`. All subsequent calls to `asyncLocalStorage.getStore()` will return `undefined` until `asyncLocalStorage.run()` or `asyncLocalStorage.runSyncAndReturn()` is called again.

When calling `asyncLocalStorage.disable()`, all current contexts linked to the instance will be exited.

Calling `asyncLocalStorage.disable()` is required before the `asyncLocalStorage` can be garbage collected. This does not apply to stores provided by the `asyncLocalStorage`, as those objects are garbage collected along with the corresponding async resources.

This method is to be used when the `asyncLocalStorage` is not in use anymore in the current process.

### `asyncLocalStorage.getStore()`
<!-- YAML
added: v13.10.0
-->

* Returns: {any}

This method returns the current store. If this method is called outside of an asynchronous context initialized by calling `asyncLocalStorage.run` or `asyncLocalStorage.runAndReturn`, it will return `undefined`.

### `asyncLocalStorage.enterWith(store)`
<!-- YAML
added: v13.11.0
-->

* `store` {any}

Calling `asyncLocalStorage.enterWith(store)` will transition into the context for the remainder of the current synchronous execution and will persist through any following asynchronous calls.

Example:

```js
const store = { id: 1 };
asyncLocalStorage.enterWith(store);
asyncLocalStorage.getStore(); // Returns the store object
someAsyncOperation(() => {
  asyncLocalStorage.getStore(); // Returns the same object
});
```

This transition will continue for the _entire_ synchronous execution. This means that if, for example, the context is entered within an event handler subsequent event handlers will also run within that context unless specifically bound to another context with an `AsyncResource`.

```js
const store = { id: 1 };

emitter.on('my-event', () => {
  asyncLocalStorage.enterWith(store);
});
emitter.on('my-event', () => {
  asyncLocalStorage.getStore(); // Returns the same object
});

asyncLocalStorage.getStore(); // Returns undefined
emitter.emit('my-event');
asyncLocalStorage.getStore(); // Returns the same object
```

### `asyncLocalStorage.run(store, callback[, ...args])`
<!-- YAML
added: v13.10.0
-->

* `store` {any}
* `callback` {Function}
* `...args` {any}

Calling `asyncLocalStorage.run(callback)` will create a new asynchronous context. Within the callback function and the asynchronous operations from the callback, `asyncLocalStorage.getStore()` will return the object or the primitive value passed into the `store` argument (known as "the store"). This store will be persistent through the following asynchronous calls.

The callback will be ran asynchronously. Optionally, arguments can be passed to the function. They will be passed to the callback function.

If an error is thrown by the callback function, it will not be caught by a `try/catch` block as the callback is ran in a new asynchronous resource. Also, the stacktrace will be impacted by the asynchronous call.

Example:

```js
const store = { id: 1 };
asyncLocalStorage.run(store, () => {
  asyncLocalStorage.getStore(); // Returns the store object
  someAsyncOperation(() => {
    asyncLocalStorage.getStore(); // Returns the same object
  });
});
asyncLocalStorage.getStore(); // Returns undefined
```

### `asyncLocalStorage.exit(callback[, ...args])`
<!-- YAML
added: v13.10.0
-->

* `callback` {Function}
* `...args` {any}

Calling `asyncLocalStorage.exit(callback)` will create a new asynchronous context. Within the callback function and the asynchronous operations from the callback, `asyncLocalStorage.getStore()` will return `undefined`.

The callback will be ran asynchronously. Optionally, arguments can be passed to the function. They will be passed to the callback function.

If an error is thrown by the callback function, it will not be caught by a `try/catch` block as the callback is ran in a new asynchronous resource. Also, the stacktrace will be impacted by the asynchronous call.

Example:

```js
asyncLocalStorage.run('store value', () => {
  asyncLocalStorage.getStore(); // Returns 'store value'
  asyncLocalStorage.exit(() => {
    asyncLocalStorage.getStore(); // Returns undefined
  });
  asyncLocalStorage.getStore(); // Returns 'store value'
});
```

### `asyncLocalStorage.runSyncAndReturn(store, callback[, ...args])`
<!-- YAML
added: v13.10.0
-->

* `store` {any}
* `callback` {Function}
* `...args` {any}

This methods runs a function synchronously within a context and return its return value. The store is not accessible outside of the callback function or the asynchronous operations created within the callback.

Optionally, arguments can be passed to the function. They will be passed to the callback function.

If the callback function throws an error, it will be thrown by `runSyncAndReturn` too. The stacktrace will not be impacted by this call and the context will be exited.

Example:

```js
const store = { id: 2 };
try {
  asyncLocalStorage.runSyncAndReturn(store, () => {
    asyncLocalStorage.getStore(); // Returns the store object
    throw new Error();
  });
} catch (e) {
  asyncLocalStorage.getStore(); // Returns undefined
  // The error will be caught here
}
```

### `asyncLocalStorage.exitSyncAndReturn(callback[, ...args])`
<!-- YAML
added: v13.10.0
-->

* `callback` {Function}
* `...args` {any}

This methods runs a function synchronously outside of a context and return its return value. The store is not accessible within the callback function or the asynchronous operations created within the callback.

Optionally, arguments can be passed to the function. They will be passed to the callback function.

If the callback function throws an error, it will be thrown by `exitSyncAndReturn` too. The stacktrace will not be impacted by this call and the context will be re-entered.

Example:

```js
// Within a call to run or runSyncAndReturn
try {
  asyncLocalStorage.getStore(); // Returns the store object or value
  asyncLocalStorage.exitSyncAndReturn(() => {
    asyncLocalStorage.getStore(); // Returns undefined
    throw new Error();
  });
} catch (e) {
  asyncLocalStorage.getStore(); // Returns the same object or value
  // The error will be caught here
}
```

### Choosing between `run` and `runSyncAndReturn`

#### When to choose `run`

`run` is asynchronous. It is called with a callback function that runs within a new asynchronous call. This is the most explicit behavior as everything that is executed within the callback of `run` (including further asynchronous operations) will have access to the store.

If an instance of `AsyncLocalStorage` is used for error management (for instance, with `process.setUncaughtExceptionCaptureCallback`), only exceptions thrown in the scope of the callback function will be associated with the context.

This method is the safest as it provides strong scoping and consistent behavior.

It cannot be promisified using `util.promisify`. If needed, the `Promise` constructor can be used:

```js
const store = new Map(); // initialize the store
new Promise((resolve, reject) => {
  asyncLocalStorage.run(store, () => {
    someFunction((err, result) => {
      if (err) {
        return reject(err);
      }
      return resolve(result);
    });
  });
});
```

#### When to choose `runSyncAndReturn`

`runSyncAndReturn` is synchronous. The callback function will be executed synchronously and its return value will be returned by `runSyncAndReturn`. The store will only be accessible from within the callback function and the asynchronous operations created within this scope. If the callback throws an error, `runSyncAndReturn` will throw it and it will not be associated with the context.

This method provides good scoping while being synchronous.

#### Usage with `async/await`

If, within an async function, only one `await` call is to run within a context, the following pattern should be used:

```js
async function fn() {
  await asyncLocalStorage.runSyncAndReturn(new Map(), () => {
    asyncLocalStorage.getStore().set('key', value);
    return foo(); // The return value of foo will be awaited
  });
}
```

In this example, the store is only available in the callback function and the functions called by `foo`. Outside of `runSyncAndReturn`, calling `getStore` will return `undefined`.
