# Glossary

This is a glossary of terms that will be commonly used in discussion. This list
will be expanded in the future. **Please avoid changing existing headings as
users may be deep-linking to the generated anchors.**

## Table of Contents

* [Error Symposium](#error-symposium)
  * [Operational Error](#operational-error)
  * [Programmer Error](#programmer-error)
* [Post Mortem Analysis](#post-mortem-analysis)
  * [Abort](#abort)
  * [Core / Core dump / Core file](#core--core-dump--core-file)
  * [Stack Frame](#stack-frame)
  * [Unwinding the Stack / Top of Stack](#unwinding-the-stack--top-of-stack)
* [Promise](#promise)
  * [Executor / Resolver](#executor--resolver)
  * [Handler](#handler)
  * [Settling](#settling)
    * [Rejection / reject](#rejection--reject)
    * [Resolving / resolve](#resolving--resolve)
  * [Synchronous Rejection](#synchronous-rejection)
  * [Unhandled Rejection / Rejection Handled](#unhandled-rejection--rejection-handled)
  * [Microtask Queue](#microtask-queue)
* [Links](#links)

## Error Symposium

An ongoing series of facilitated discussions led by **@groundwater**, originally
conceived at the August 2015 Node Collaborator Summit at Mozilla SF. Their goal
is to produce recommendations on "best practices" for error handling in Node.

Notably, the conversation has been informed by the [foundational work][joyent-errors]
written by **@davepacheco**. This document was paraphrased by **@chrisdickinson** as
the part of the Node.js [error documentation][node-errors].

[The results of their latest meeting are here][symposium-findings].

### Operational Error

An error that represents a known state of the program. To paraphrase the
[joyent error guide][joyent-errors]:

> Operational errors represent run-time problems experienced by correctly-written
> programs. These are not bugs in the program. In fact, these are usually
> problems with something else: the system itself (e.g., out of memory or too
> many open files), the system's configuration (e.g., no route to a remote host),
> the network (e.g., socket hang-up), or a remote service (e.g., a 500 error,
> failure to connect, or the like).

### Programmer Error

An error that represents a mistake on the programmer's behalf. To paraphrase the
[joyent error guide][joyent-errors]:

> Programmer errors are bugs in the program. These are things that can always
> be avoided by changing the code. They can never be handled properly (since by
> definition the code in question is broken).

#### A Note on Programmer/Operational Error Propagation

These definitions do not describe the specific propagation method for these
classes of errors. Either may appear as an error parameter to a nodeback
function, or thrown by the API. The Error Symposium recommends only throwing
on programmer error, while using the nodeback system to propagate operational
errors.

## Post Mortem Analysis

The practice of debugging a program after it has already crashed by inspecting
the serialized state of the program at the time of error. This usually leverages
tools like `lldb`, `gdb`, or `mdb` to inspect a "core" (defined below.)

### Abort

The [`abort()`][abort-syscall] system call. Accessible to Node via
`process.abort()`. Called by V8 when `--abort-on-uncaught-exception` is enabled
and an exception is thrown without a top-level `catch` handler.

Halts the program and serializes the state of the program (the contents of the
stack and heap segments, as well as the state of open file descriptors) to a
"core file." Causes the program to return a non-zero exit status.

### Core / Core dump / Core file

The serialized state of a program, containing the stack and heap segments of
memory in their entirety. Analyzable via `mdb`, `gdb`, or `lldb`. V8-specific
plugins are available in the form of `mdb_v8`, `llnode`, and `lldb-v8`, and
allow users to inspect JavaScript stack frames as well as the frozen object
graph in memory (including objects pending garbage collection.)

### Stack Frame

A data structure held in the stack segment of memory representing a function
call. This typically saves the arguments with which the function was called
as well as a return address to jump to when the call completes.

### Unwinding the Stack / Top of Stack

Stack frames are collected into a stack that grows and shrinks as functions
are called and functions return. Exceptions work by "unwinding" the stack
to the nearest exception handler, if any — that is, they remove stack frames
giving control back to the nearest frame with an exception handler. The
stack is grown and shrunk in-place, so any subsequent function calls (even
ones implicitly made by the JS VM) will clobber the original stack leading
to the error. `Error` stacktraces _record_ some frame information leading to
the error, but throwing an error is generally a lossy operation.

If no exception handler is registered in a stack, instead of unwinding the
stack, V8 may abort the program or call a user-defined `OnUncaughtException`
handler.

## Promise

The 30-second version:

A [well-specified][promises-aplus] pattern that provides a container type
designed to describe the dependence graph of asynchronous operations within a
program. As a container type, any value `T` may be cast to an asynchronous
value `Promise<T>`. Any value `Promise<T>` may be unwrapped to `T` by passing a
handler (defined below) to `.then`. Unwrapping via `.then` creates a new
`Promise` that will resolve to the return value of the handler. One promise may
have zero-to-many child promises.

Errors will propagate from a source promise to all child promises. If a
rejected promise has no children, it is considered an unhandled rejection. It
may be handled in a subsequent operation. If a promise has an error handler,
propagation from source promises will halt at that promise. If a promise has
many children, and only one handles errors, the other children will be
unhandled rejections.

Unwrapping `Promise<Promise<T>>` will automatically unwrap to `T`. That is:
`.then(() => Promise.resolve(3)).then(val => console.log(val))` will
output `3`.

Internally, a promise is represented by a state machine that may be in one of
three states — pending, fulfilled, or rejected. The fulfilled and rejected
states are collectively referred to as "settled." Once a promise is settled, it
cannot transition into any other state.

### Executor / Resolver

The function passed to the `new Promise` constructor. Run **synchronously**.
Not in common direct use by most promise-users — used mainly to wrap sources of
asynchrony. As such, it rarely resolves or rejects on the same tick. This API
is provided for the express purpose of wrapping callback APIs.

Catches all errors thrown. If an error is caught, the returned promise will be
**rejected**. Because it starts life as a rejected promise with no children, it
will immediately hit the `OnUnhandledRejection` callback.

```js
new Promise(function Excecutor (resolve, reject) {
  // run immediately.
})
```

### Handler

A function passed to `Promise#then` or `Promise#catch`. If the function is
passed as the first argument to `Promise#then`, it is a **fulfillment**
handler. Otherwise, if it is passed to `.catch` or as the second argument to
`.then`, it is a **rejection** handler. Fulfillment handlers are called with
the resolved value of the parent promise. Rejection handlers are called with
the error value of the parent promise. Calling `.then` or `.catch` on a Promise
`A` returns a new `Promise`, which is a child of `A`. A handler which returns a
value `T` or `Promise<T>` will resolve the new promise with `T`. A handler
which throws an error `R`, or returns a `Promise` which itself eventually
rejects (or has already rejected) with `R`, will reject the new promise with
`R`.

```js

const p = new Promise(SomeOmittedExecutor)

const p2 = p.then(function SuccessHandler(value) {
  // fired if p fulfills,
}, function ErrorHandler(err) {
  // fired if p rejects.
})

const p3 = p2.catch(function ErrorHandler(value) {
  // fired if p2 rejects.
})

const p4 = p3.then(function SuccessHandler (value) { 
  // resolved with p2's value if p2 resolves,
  // resolved with p3's value if p2 rejects and p3 resolves
})
```

### Resolution / Settling

The transition of a Promise from pending to a settled state, either "rejected"
or "resolved." Once settled, a promise may never return to "pending" state.

#### Rejection / reject

The act of settling a promise with an error. Can be performed by the Executor
or Handler `H` by throwing while `H` is on stack, or by resolving with a
Promise that rejects at some point in its lifetime. In the case of Executors,
rejection may additionally be achieved by calling the `reject` callback
provided as an argument.

Rejection is analogous to a synchronous `throw`.

#### Fulfillment / resolve

The act of settling a promise with a value. Can be performed by the Executor or
a Handler. Executors may only resolve by calling the `resolve` callback
provided to them as an argument, their return value is ignored. In the case of
Handlers, fulfillment is achieved by returning a value from the Handler
function.

Fulfilling a Promise `A` using a Promise `B` will cause `A` to resolve or
reject to the same value or error as `B`. The resolution of `A` is dependent on
the resolution of `B`: if `B` never settles, `A` will never settle. The process
of fulfilling a promise with another promise is known as "assimilation".

### Synchronous Rejection

Sychronous rejection refers to the rare case where an executor is immediately
rejected using `reject(<rejection>)` or `throw <rejection>`. In the case of the
proposed Node Promise API, only invalid API use will trigger a synchronous
rejection.

May also be triggered by the helper method, `Promise.reject(<error>)`.

### Unhandled Rejection / Rejection Handled

Reported by the VM using `OnUnhandledRejection` and `OnRejectionHandled`. Called
when a promise with no children is rejected. If the rejected promise is handled
by a later operation, the `OnRejectionHandled` callback is called. A promise
may synchronously transition from unhandled to handled:

```js
const p1 = new Promise(() => { throw new Error('xyz')})
const p2 = p1.catch(err => { /* err === [Error: xyz] */ })
// calls executor
//   rejects
//   calls unhandled rejection with p1 and error
// calls p2.catch
//   calls rejection handled with p1
```

Node currently [keeps track of extant rejections][rejection-pr], and fires
`process.on('unhandledRejection')` for any rejections that have not been
handled at the end of the microtask queue.

### Microtask Queue

The Microtask Queue is a [specification-mandated queue][job-queue] of functions
to be run at the exhaustion of a JavaScript stack. Promises feed into this
queue: settled promises will enqueue a task to run handlers as they are added,
pending promises with handlers will enqueue a task to run handlers when the
promise settles. Other language-level features, like `Object.observe`, also
feed into this queue.

Currently this queue is opaque to Node. Node is not notified when new
microtasks are queued, and Node may only tell V8 to run all of the microtasks
queued to completion — it can't run them one at a time.

## Links

* The public Microtask Queue API is defined [here][v8-public-microtask-api].
* V8's promise implementation is defined [here][v8-promise-implementation].
  * `%EnqueueMicrotask` is defined [here][v8-intrinsic], and runs [this code][v8-private-microtask-api].
* V8's unhandled rejection API is defined [here][v8-unhandled-rejection-api].
* V8's abort on uncaught behavior is defined [here][v8-abort].

[v8-public-microtask-api]: https://github.com/v8/v8/blob/a26d0ac676ffc96a886e294728e69f5e7ba11b25/src/api.cc#L7399-L7414
[v8-promise-implementation]: https://github.com/v8/v8/blob/a26d0ac676ffc96a886e294728e69f5e7ba11b25/src/js/promise.js
[v8-intrinsic]: https://github.com/v8/v8/blob/a26d0ac676ffc96a886e294728e69f5e7ba11b25/src/runtime/runtime-observe.cc#L39
[v8-private-microtask-api]: https://github.com/v8/v8/blob/a26d0ac676ffc96a886e294728e69f5e7ba11b25/src/isolate.cc#L2671-L2732
[v8-unhandled-rejection-api]: https://github.com/v8/v8/blob/a26d0ac676ffc96a886e294728e69f5e7ba11b25/src/isolate.cc#L2652-L2668
[v8-abort]: https://github.com/v8/v8/blob/a26d0ac676ffc96a886e294728e69f5e7ba11b25/src/isolate.cc#L1037-L1050
[promises-aplus]: https://promisesaplus.com/
[abort-syscall]: http://man7.org/linux/man-pages/man3/abort.3.html
[joyent-errors]: https://www.joyent.com/developers/node/design/errors
[node-errors]: http://nodejs.org/api/errors.html
[symposium-findings]: http://github.com/groundwater/nodejs-symposiums/
[rejection-pr]: https://github.com/nodejs/node/pull/758
[job-queue]: http://www.ecma-international.org/ecma-262/6.0/index.html#sec-jobs-and-job-queues
