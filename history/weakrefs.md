# NOT CURRENT. PLEASE SEE PRESENTATION

While much of this document is still relevant, the spec and background
information _documents_ are still being revised. For information on
the current proposal, please see the [current
slides](https://github.com/tc39/proposal-weakrefs/blob/master/specs/Weak%20References%20for%20EcmaScript.pdf).
They are up to date.

[Initial Spec-text](https://github.com/tc39/proposal-weakrefs/blob/master/specs/spec.md)

[Slides](https://github.com/tc39/proposal-weakrefs/blob/master/specs/Weak%20References%20for%20EcmaScript.pdf)

# Background

In a simple garbage-collected language, memory is a graph of
references between objects, anchored in the roots such as global data
structures and the program stack. The garbage collector may reclaim
any object that is not reachable through that graph from the
roots. Some patterns of computation that are important to JavaScript
community are difficult to implement in this simple model because
their straightforward implementation causes no-longer used objects to
still be reachable. This results in storage leaks or requires
difficult manual cycle breaking. Example include:

  * MVC and data binding frameworks
  * reactive-style libraries and languages that compile to JS
  * caches that require cleanup after keys are no longer referenced
  * proxies/stubs for comm or object persistence frameworks
  * objects implemented using external or manually-managed resources

Two related enhancements to the language runtime will enable
programmers to implement these patterns of computation: weak
references and finalization. A strong reference is a reference that
causes an object to be retained; in the simple model all references
are strong. A *weak reference* is a reference that allows access to an
object that has not yet been garbage collected, but does not prevent
that object from being garbage collected. *Finalization* is the
execution of code to clean up after an object that has become
unreachable to program execution.

For example, MVC frameworks often use an observer pattern: the view
points at the model, and also registers as an observer of the model.
If the view is no longer referenced from the view hierarchy, it should
be reclaimable. However in the observer pattern, the model points at
its observers, so the model retains the view (even though the view is
no longer displayed). By having the model point at its observers using
a weak reference, the view can just be garbage collected normally with
no complicated reference management code.

Similarly, a graphics widget might have a reference to a primitive
external bitmap resource that requires manual cleanup (e.g., it must
be returned to a buffer pool when done). With finalization, code can
cleanup the bitmap resource when the graphics widget is reclaimed,
avoiding the need for pervasive manual disposal discipline across the
widget library.

# Intended Audience

The garbage collection challenges addressed here largely arise in the
implementation of libraries and frameworks. The features proposed here
are advanced features (e.g., like proxies) that are primarily intended
for use by library and framework creators, not their clients. Thus,
the priority is enabling library implementors to correctly,
efficiently, and securely manage object lifetimes and finalization.

# Solution Approach

This proposal is based on Jackson et.al.'s **Weak References and
Post-mortem Finalization**. Post-mortem finalization was a major
innovation in garbage collection pioneered for Smalltalk and since
applied in numerous garbage-collected systems. It completely and
simply addresses two major challenges for finalization: resurrection
and (what I will call) layered-collection.

An object that the garbage collector has decided to reclaim is called
a condemned object. Resurrection happens if a condemned object becomes
reachable again as the result of the finalization; for example, if the
finalization code stores the object into a data structure.

Layered collection happens when a multi-object data structure such as
a tree becomes unreachable by its client. For such a tree, if the
finalization for each node requires the children, then the first GC
can only condemn and reclaim the root node; the children can be
reclaimed at the second GC; the grandchildren at the third GC, etc.

The post-mortem finalization approach eliminates resurrection entirely
and avoids layered collection naturally. The central insight is that
finalization never touches the condemned object. Once an object is
condemned, it stays condemned (the transition is monotonic).

To support finalization, an object called an *executor* can be
associated with any target object. When the target object is
condemned, the executor will execute to clean up after the target. The
executor needs to use some state to clean up after target; the
condemned target is no longer available. That state importantly might
*not* be state previously "owned" by the target. For example, cached
XML nodes for an XML parser would not have been owned by the file they
are parsed from. Therefore the proposal uses the general terms
*holdings* for this state (rather than the attractive but misleading
term "estate").

By never allowing a reference to the condemned object, resurrection is
simply precluded. For a layered data structure, the entire data
structure can be reclaimed in one GC, and then the executors for the
individual nodes run.

Resurrection and related finalization issues are especially subtle
when composing libraries and abstractions that use finalization
internally. For further background, Chris Brumme discusses some of
C#'s [pre-mortem finalization challenges](http://blogs.msdn.com/b/cbrumme/archive/2004/02/20/77460.aspx).

# Proposed Solution

We specify the proposed API in terms of a new function:
`makeWeakRef`. It takes a `target` object, an optional `executor`,
and an optional `holdings` object, and returns a fresh weak reference
that internally has a weak pointer to the target. The target can be
retrieved from the resulting weak reference unless the target has been
condemned, in which case `null` is returned. Once target has been
condemned, there are no references to it in the system. If the target
is condemned and the weak reference is not, then the weak reference's
optional executor will eventually be invoked in its own job (aka
turn), with the optional holdings passed as parameter.

Note that objects may be unreachable but not yet condemned, e.g.,
because they are in a different generation. The "condemned" objects
are the subset of unreachable objects that the garbage noticed it can
reclaim. Because condemned objects can never become reachable again,
it is not visible to the program whether they were actually reclaimed
or not at any particular time.

## Usage Overview

[Pre-weakref](pre-structure.svg) The first diagram shows the objects
in an example scenario before weak references are used.

  * A client gets the target (e.g., a remote reference) from the 
    service object (e.g., the remote connection)
  * The target allocates/uses Holdings to accomplish it's function
    (e.g., a remote reference index)
  * The Service Object wants to clean up using the holdings after the
    target is "done"

The problem is that the Service Object will retain the Target, thus
preventing it from getting collected.

[With-weakref](weakref-structure.svg) The second diagram inserts the
weak reference in the reference path from the Service Object to the
Target. It adds the Executor, which will be invoked to clean up after
the Target, using the Holdings (e.g., drop the remote reference).

## Basic Usage

The root of the API is the `makeWeakRef` operation to construct a weak
reference.

```js
// Make a new weak reference.
// The target is a strong pointer to the object that will be pointed
// at weakly by the result.
// The executor is an optional argument that will be invoked after the
// target becomes unreachable.
// The holdings is an optional argument that will be provided to the
// executor when it is invoked for target.
makeWeakRef(target, executor, holdings);
```

This brief example shows the usage and core behavior of the API.
```js
let buf = pool.getBuf();
let original = someObject(buf);
let executor = buffer => buffer.release();
let wr = makeWeakRef(original, executor, buf);
// full GC and finalization happens here
assert(wr.get() === original);
original = undefined;
// full GC and finalization happens here
assert(wr.get() === null);
assert(buf.isReleased);
```

## Expanded Example

This example shows a typical pattern of finalization for a simple
remote reference implementation. `RemoteConnection` manages the client
side references to objects on a server. Each remote reference is
represented on the wire with an index (called `remoteId`, below). The
`makeResultRemoteRef` operation creates the appropriate local object
and arranges so that when that object is condemned, finalization will
invoke `dropRef` in order to remove the associated bookkeeping and
notify the server that the client is no longer referencing that
object.

```js

class RemoteConnection {

  constructor(transport) {
    this.transport = transport; // the low-level communication channel
    this.executor = remoteId => this.dropRef(remoteId);
    this.remotes = new Map();  // map from id to weakRef
    ??? // more construction
  }

  // Create a remote reference for a remote return result. Setup  
  // finalization to clean up after it if it is garbge collected.
  makeResultRemoteRef(remoteId) {
    let remoteRef = ???; // remoteRef construction elided
    this.remotes.set(remoteId, makeWeakRef(remoteRef, this.executor, remoteId));
    return remoteRef;
  }

  // The remote reference object has been garbage collected. Discard  
  // the associated bookeeping and notify the server that it is no 
  // longer referenced.
  dropRef(remoteId) {
    this.transport.send("DROP", remoteId);
    this.remotes.delete(remoteId);
  }
}
```

It is useful to note that in this scenario, the holdings for each
remote reference is simply the remoteId itself. Additionally, if the 
entire remote connection is garbage-collected, then there's no need
to perform finalization for any of these remote references. Thus 
finalization of the remote references created for the connection is
scoped to the connection.

# Additional Requirements and Discussion

## Finalization between turns

The finalization code is user code that may modify application,
library, and system state. It's crucial that it not run in the midst
of other user code because invariants may be in transition. ECMAScript
can address this ideally by running executors only between turns;
i.e., when the application stack is empty.

Because there may be a large number of finalizers and they are user
code that could run for unbounded periods, it crucial for system
responsiveness that the finalizers can run interleaved with multiple
program jobs. Therefore the proposal specifies that finalizers are
scheduled conceptually on a separate job queue (or queues) for
finalization. This allows an implementation to progress finalization
while preserving application program responsiveness.

## Multiple Independent Executors

Finalization is sometimes about an object's internal implementation
details and sometimes about the client usage of an object. An example
of internal usage is file handle cleanup -- when a File object is
collected, the implementation calls a system primitive to close the
corresponding file handle. An example of client usage could be a cache
of parsed content from a file -- when the file is collected, the cache
wants to clear the corresponding cache information. These scenarios
can both be implemented with the WeakRef and finalization approach,
using holdings of a file handle and a cache key, respectively. A
single file object might participate independently in both these uses.

## Unregistering from Finalization

The finalization needs for a given target can change depending on
circumstances. For example, if a file is manually closed (e.g.,
because it was in a stream and the stream read all the contents) then
finalization would be unnecessary and potentially a source of errors
or exceptions. To avoid this source of complexity and bugs, this
proposal specifies that clearing a WeakRef before the executor runs
atomically prevents the executor for being invoked for the original
target. In practice, this tends to eliminate the need for  conditional
finalization code.

## Portability and Observable Garbage Collection

Revealing the non-deterministic behavior of the garbage collector
creates a potential for portability bugs. Different host environments
may collect a weakly-held object at different times, which a
`WeakRef` exposes to the program. More generally, the
`makeWeakRef` function is not safe for general access since it
grants access to the non-determinism inherent in observing garbage
collection. The resulting side channel reveals information that may
violate the [confidentiality assumptions](https://web.archive.org/web/20160318124045/http://wiki.ecmascript.org/doku.php?id=strawman:gc_semantics)
of other programs. Therefore we add `makeWeakRef` to the `System`
object, just like other authority-bearing objects, e.g.,
[the default loader](https://github.com/whatwg/loader/issues/34).

WeakRef instances may often be referenced by code that should not have
the same authority. Therefore the constructor for WeakRefs must not be
available via WeakRef instances.

## Reference stability during turns

To further eliminate unnecessary non-determinism, we tie the
observable collection of weak references to the event loop semantics.
The informal invariant is:

*A program cannot observe a weak reference be automatically deleted
within a turn of the event loop.*

Specifically, this means that, unless the weak reference is explicitly
modified during the turn (e.g., via `clear`):

  * When a weak reference is created, subsequent calls to its
    `get` method within the same turn must return the object with
    which it was created.
  * If a weak reference's `get` method is called and produces an
    object, subsequent calls to its `get` method within the same
    turn must return the same object.

A naive approach to this is to simply restrict garbage collection to
between turns. However, some programs have significant, non-retained
allocation during turns.  For example, if a large tree is made
unreachable during a turn, and replaced with a new tree (e.g., virtual
DOM creation in React), this strategy would prevent the original,
now-unreachable tree from being collected until the next GC between
turns. Such a restriction would lead to unexpected OOM for otherwise
correct, non-leaking programs. Therefore this approach is not
sufficient or acceptable.

The example pseudocode below illustrates a simple approach that
straightforwardly and efficiently achieves the reference stability
requirements. The example is a builtin that has access to some (all
upper case) magic that is not available to normal JavaScript. This
pseudocode is more magical than a self-hosted builtin, because we
assume the following code blocks are atomic wrt gc.

The MARK method explains how the normal mark phase of gc marks through
a weak reference. It treats the `target` pointer as strong *unless* it
has not been observed in the current turn.

```js
function makeWeakRef(target, executor = void 0, holdings = void 0) {
  if (target !== Object(executor)) {
    throw new TypeError('Object expected');
  }
  let observedTurn = CURRENT_TURN;
  WEAK_LET weakPtr = target;

  const weakReference = {
    get() {
      observedTurn = CURRENT_TURN;
      return weakPtr;
    }
    clear() {
      weakPtr = void 0;
      executor = null;
      holdings = null;
    }
    // MARK is only called by gc, as part of its normal mark phase.
    // Obviously, not visible to JS code.
    MARK() {
      MARK(executor);
      MARK(holdings);
      if (observedTurn === CURRENT_TURN) {
        MARK(weakPtr);
      }
    }
    // FINALIZE is called in its own turn and only if the target was
    // condemned. Obviously, not visible to JS code.
    FINALIZE() {
      if (typeof executor === 'function') {
        let exec = executor;
        let hold = holdings;
        weakReference.clear();
        exec(hold);
      }
    }
  };
  return weakReference;
}
```
NOTE A more complete version of this code is at the end of the
document, which avoids allocations and covers more cases and
subtleties.

## Allocation During GC and Finalization

It is essential that the garbage collection itself not be forced to
allocate; that can substantially complicate the garbage collector
implementation and lead to storage allocation thrashing. The design in
this proposal allows all space required for bookkeeping to be
preallocated so that the GC is never required to allocate. Since
finalization is not executed directly by the garbage collector, user
finalization code is allowed to allocate normally.

## Exceptions during Finalization

Each finalization action occurs in its own turn. Therefore exceptions
thrown at the top level of finalization can use normal exception
handling behavior.

## Unintended Retention

Post-mortem finalization prevents resurrection bugs. However, some
patterns of code using finalization may inadvertently retain objects
that would otherwise have been garbage. This proposal is designed to
minimize these leaks, because bugs of this nature tend to only
manifest when there's enough load that a small leak matters. The most
basic cause is if the executor is setup to use the target itself to
clean up after itself (i.e., someone tries to emulate destructors).
Providing the target itself as the executor or holdings prevents the
entire purpose of WeakRefs because the WeakRef points strongly at
those. In practice this is almost always an error and so should be
detected and signaled at WeakRef creation time.

Another cause of unexpected retention is functions whose stored
context includes the target (this may be because the functions
mentions the target explicitly, or because the runtime used a shared
context for multiple functions). A pattern of allocating new executor
functions for each new weak reference is more susceptible to this
issue.

An example with file stream construction that arranges the underlying
file to be closed (with an optional warning). The last line of the
constructor is deliberately unrelated code that uses a function.
```js
const openfiles = new Map();

// closeFile is the shared executor for all files.
const closeFile(file) {
  file.close();
  openFiles.delete(file);
  console.info("Filestream dropped before close: ", file.name)
}

class FileStream {
  constructor(filename) {
    this.file = new File(filename, "r");
    openFiles.set(file, makeWeakRef(this, () => closeFile(this.file)));
    // now eagerly load the contents
    this.loading = file.readAsync().then(data => this.setData(data));    
  }, ...
}
```
The constructor code straightforward, but unfortunately closes over
the file stream target `this`, and therefore prevents garbage 
collection of it. A more careful variant is:
```js
class FileStream {
  constructor(filename) {
    let file = new File(filename, "r");
    this.file = file;
    openFiles.set(file, makeWeakRef(this, () => closeFile(file)));
    // now eagerly load the contents
    this.loading = file.readAsync().then(data => this.setData(data));    
  }, ...
```
With sufficient care, the finalization avoids retaining `this`.
However, as mentioned above, a wide variety of function implementation
approaches use shared records for multiple functions in the same
scope. The mere existence of the `data => this.setData(data)` function
in the same scope could cause the executor function
`() => closeFile(file)` to additionally point at `this`. The pattern
using `holdings` avoids this:
```js
class FileStream {
  constructor(filename) {
    let file = new File(filename, "r");
    this.file = file;
    openFiles.set(file, makeWeakRef(this, closefile, file));
    // now eagerly load the contents
    this.loading = file.readAsync().then(data => this.setData(data));    
  }, ...
```

## WeakRef collection

Execution of the executor associated with a WeakRef is only required
if the WeakRef itself is still retained. Allowing unreachable WeakRefs
to be condemned without handling their executor prevents resurrection
issues with holdings and executor functions, and allows efficient
collection when the application discards entire subsystems that
internally use finalization for resources that are also discarded
(e.g., an entire cache is dropped, not just the keys).

## Cross-realm references

Weak references enable observation of the behavior of the garbage
collector, which can provide a channel of communication between
isolated subgraphs that share only transitively immutable objects, and
therefore should not be able to communicate. Security-sensitive code
would most likely need to virtualize or censor access to the
`makeWeakRef`  function from untrusted code. However, such
restrictions do not enable one realm to police other realms. To plug
this leak, a weak reference created within realm A should only point
weakly within realm A. When set to point at an object from another
realm, this can be straightforwardly enforced in the WeakRef
construction: if the Target is from another realm, then the Target is
also stored in the strong holdings pointer. Because the WeakRef
thereby retains the Target, Target will only become unreachable if the
WeakRef is unreachable, and therefore will never require finalization.
Security demands only that such inter-realm references not point
weakly.

See [https://mail.mozilla.org/pipermail/es-discuss/2013-January/028542.html](https://mail.mozilla.org/pipermail/es-discuss/2013-January/028542.html)
for a nice refinement for pointing weakly at objects from a set of
realms.

## Information leak via Executor

Particularly in a library context, the same executor might be used for
multiple otherwise-unrelated objects. A misbehaving executor could
pass secrets from one object to another. In order to facilitate
pre-existing, shared executors without compromising encapsulation, the
protocol is designed to not require mutation within the
executor. Thus, a deeply immutable function can safely be used as the
executor without inadvertently creating a communications channel.

## Process Termination

Finalization is for the purpose of application code, and not to
protect system resources. The runtime for resources must ensure that
resources they use are appropriately reclaimed upon process
shutdown. Process/worker shutdown must not require garbage collection
or execution of finalizers.

## WeakRefs vs. WeakArrays

Most early post-mortem finalization systems used a WeakArray
abstraction rather than individual WeakRefs. With WeakArrays, all
array slots point weakly at their target, and the executor for the
array is provided the index that is no longer reachable. Among other
things, this was intended to amortize the overhead of weak reference
management while providing convenient bulk finalization support.
Modern generational garbage collectors change these trade-offs: for
many use cases, the Weak Array gets promoted to an older generation
and ends up pointing at objects in newer generations. The resulting
"remember set" overhead and impact on GC outweighs that potential
advantage. Therefore this proposal specifies the simpler WeakRef
approach here.

## The API

This shows pseudo typing and behavioral description for the proposed
weak references API.

```js
makeWeakRef : function(
                target   : object,
                executor : undefined | function(holdings : any) -> undefined,
                holdings : any) -> WeakRef
WeakRef     : object {
                get   : function() -> object | void 0
                clear : function() -> void 0
              }
```

  * `makeWeakRef(target, executor = void 0, holdings = void 0)` -
  returns a new weak reference to `target`. Throws `TypeError` if
  `target` is not an object or if it is the same as `holdings` or
  `executor`.

  * `get` - returns the weakly held target object, or `undefined`
   if the object has been collected.

  * `clear` - sets the internal weak reference to `undefined` and
    prevents the executor from getting invoked.

If an executor is provided, then it may be invoked on the holdings
(with **this** bound to **undefined**) in it's own turn if:

* the `target` is condemned
* the `WeakRef` is not condemned
* the `WeakRef` has not been `clear()`ed.

Constructing a new `WeakRef` or retrieving the `target` from an
existing `WeakRef` (using `get()`) causes the `WeakRef` to act as a
normal (strong) reference to the target for the remainder of the
current turn. Thus for example, if a GC is performed during that turn
but after such a `get()` call the `WeakRef` is processed as a normal
object.

## Specification [Work In Progress]

The specification for Weak references and finalization will include
the `WeakRef` type and some characteristics of the runtime garbage
collector.

### WeakRef Objects

A `WeakRef` is an object that is used to refer to a target object
without preserving it from garbage collection, and to enable code to
be run to clean up after the target is garbage collected. There is not
a named constructor for `WeakRef` objects. Instead, `WeakRef` objects
are created by calling a privileged system function [TBD].

#### MakeWeakRef Abstract Operation

The abstract operation `makeWeakRef` with arguments `target`,
`executor`, and `holdings` is used to create such `WeakRef` objects.
It performs the following steps:

1. If Type(target) is not Object, throw a TypeError exception
1. If Type(executor) is neither Undefined nor Function, throw a TypeError exception
2. If SameValue(target, holdings), throw a TypeError exception
3. If SameValue(target, executor), throw a TypeError exception
4. Let _currentTurn_ be ! GetCurrentJobReference().
5. Let _targetRealm_ be ! GetFunctionRealm(target).
6. Let _thisRealm_ be ! GetFunctionRealm(**this**).
7. If SameValue(targetRealm, thisRealm), then
  1. Let _weakRef_ be ObjectCreate(%WeakRefPrototype%, ([[Target]], [[ObservedTurn]], [[Executor]], [[Holdings]])).
  2. Set weakRef's [[Target]] internal slot to _target_.
  3. Set weakRef's [[ObservedTurn]] internal slot to _currentTurn_.
  4. Set weakRef's [[Executor]] internal slot to _executor_.
  5. Set weakRef's [[Holdings]] internal slot to _holdings_.
  6. Set all weakRef's garbage collection internal operations.
8. Else
  1. Let _weakRef_ be ObjectCreate(%WeakRefPrototype%, ([[Target]], [[ObservedTurn]], [[Executor]], [[Holdings]])).
  2. Set weakRef's [[Target]] internal slot to _target_.
  3. Set weakRef's [[ObservedTurn]] internal slot to _currentTurn_.
  4. Set weakRef's [[Executor]] internal slot to **undefined**.
  5. Set weakRef's [[Holdings]] internal slot to **undefined**.
9. Return weakRef.

#### The %WeakRefPrototype% Object

All `WeakRef` objects inherit properties from the
`%WeakRefPrototype%` intrinsic object.  The `%WeakRefPrototype%`
object is an ordinary object and its `[[Prototype]]` internal slot is
the `%ObjectPrototype%` intrinsic object. In addition,
`%WeakRefPrototype%` has the following properties:

##### %WeakRefPrototype%.get( )

1. Let _O_ be the this value.
2. If Type(_O_) is not Object, throw a TypeError exception.
3. If _O_ does not have all of the internal slots of a WeakRef Instance, throw a TypeError exception.
4. Let _currentTurn_ be ! GetCurrentJobReference().
5. Set the value of the [[ObservedTurn]] internal slot of _O_ to _currentTurn_.
6. Let _a_ be the value of the [[Target]] internal slot of _O_.
7. Return a.

##### %WeakRefPrototype%.clear( )
1. Let _O_ be the this value.
2. If Type(_O_) is not Object, throw a TypeError exception.
3. If _O_ does not have all of the internal slots of a WeakRef
    Instance, throw a TypeError exception.
4. Set the value of the [[Target]] internal slot of O to undefined.
5. Set the value of the [[Executor]] internal slot of O to null.
6. Set the value of the [[Holdings]] internal slot of O to null.
7. Return undefined.

##### %WeakRefPrototype% [ @@toStringTag ]

The initial value of the @@toStringTag property is the string value
"WeakRef".

#### Properties of WeakRef Instances

WeakRef instances are ordinary objects that inherit properties from
the %WeakRefPrototype% intrinsic  object. WeakRef instances are
initially created with the internal slots listed in Table K.

*Table K — Internal Slots of WeakRef Instances*

| Internal Slot | Description |
| ----- | ----- |
| [[Target]] | The reference to the target object |
| [[ObservedTurn]] | A unique reference associated with the the turn |
| [[Executor]] | An optional reference to a function |
| [[Holdings]] | Any value, passed as a parameter to [[Executor]] |

#### GetCurrentJobReference

`WeakRef` operations rely on runtime state. This operation returns
reference that is uniquely associated with the current turn.

# GC Implementation Approach

First we present pseudo-code for the garbage collector phases, and
then a more complete sample of the weak reference implementation.

The approach is illustrated in the context of a mark-and-sweep
collector. It uses two additional elements over simple gc:

  * ENCOUNTERED set -- the WeakRefs encountered during marking.
  * Finalization QUEUE -- the WeakRefs whose targets have been
    condemned.

WeakRefs whose targets are collected will have their contents set to
undefined.

### The algorithm

#### Setup

  * Clear the finalization QUEUE. It will be rebuilt by the current GC
    cycle and we don't want to retains things because of it.

#### Marking

  * Mark the reachable graph from roots, as normal
  * Remember any encountered weak refs in the ENCOUNTERED set
  * WeakRefs mark their targets only if they were accessed during the
    current turn

#### Schedule finalization

  * Atomically revisit encountered WeakRefs
    * If the target is not marked or already undefined
      * Set the target pointer to undefined
      * Schedule the WeakRef for finalization

This phase sets all references still weakly pointing at condemned
objects to be undefined. This phase must be atomic so that no code can
dereference a WeakRef to a condemned target. Multiple WeakRefs to the
same target must all get set to undefined atomically.

By checking for `undefined` in this phase, GC and scheduling
finalization is idempotent. If GC started over, a still-reachable weak
ref with an executor and a collected target will be rescheduled for
finalization.

#### Sweep

  * Sweep as normal.

Because everything condemned is truly unreachable, the sweep does not
need to coordinate with other jobs, finalization, etc.

#### Finalization

For each weak ref that was selected for finalization, if they haven't
been cleared, execute their executor in it's own turn. The executor
reference must be cleared first since it's presence indicates the
finalization is still required. This ensures that each executor will
only execute once per WeakRef with a condemned target.

### The code

```js
function makeWeakRef(target, executor = void 0, holdings = void 0) {
  if (target !== Object(target)) {
    throw new TypeError('Object expected');
  }
  if (target === holdings) {
    throw new TypeError('Target cannot be used to clean up itself');
  }
  if (target === executor) {
    throw new TypeError('Target cannot be used to clean up itself');
  }
  if (typeof executor !== 'function' || executor !== null) {
    throw new TypeError('Executor must be callable, if provided');
  }
  if (REALM_OF(target) === REALM_OF(makeWeakRef)) {
    return {
      __proto__: %WeakRefPrototype%,
      [[Target]]: target,  // WEAK
      [[ObservedTurn]]: [[CURRENT_TURN]],
      [[Executor]]: executor,
      [[Holdings]]: holdings,
    };
  } else {
    // For cross-realm targets, the target is also copied into 
    // holdings so that it is pointed to strongly by the WeakRef.
    // Since a cross-realm weakRef is just strong, the target will be 
    // retained at least as long as the weakRef. Therefore 
    // finalization can ever be triggered and the supplied executor 
    // and holdings can never be used. So this just uses the holdings
    // internal slot to strongly point at the target.
    return {
      __proto__: %WeakRefPrototype%,
      [[Target]]: target,
      [[ObservedTurn]]: [[CURRENT_TURN]],
      // the Target is strongly held by the WeakRef so finalization
      // for Target will never happen. Therefore we don't need these
      [[Executor]]: void 0,
      [[Holdings]]: target,
    };    
  }
}

%WeakRefPrototype% = {
  get() {
    this.[[ObservedTurn]] = [[CURRENT_TURN]];
    return this.[[Target]];
  },
  clear() {
    // Undefined is used to indicate the collected or cleared
    // state so that it's analogous to being now uninitialized.
    this.[[Target]] = void 0;
    this.[[Executor]] = null;
    this.[[Holdings]] = null;
  }
};

// The following internal Mark and finalization methods are on
// each WeakRef instance.

// Called by gc as part of its normal mark phase.
function [[WeakRefMark]](weakref) {
  MARK(weakref.[[Executor]]);
  MARK(weakref.[[Holdings]]);
  MARK(weakref.[[ObservedTurn]]);
  if (weakref.[[ObservedTurn]] === [[CURRENT_TURN]]) {
    MARK(weakref.[[Target]]);
    return;
  }
  if (typeof weakref.[[Executor]] === 'function'
        && ![[IS_MARKED]](weakref.[[Target]])) {
    // weakref will only potentially need finalization if it has
    // an executor.
    [[ENCOUNTER]](weakref);
  }
};

// Called for each weak ref after all marking on all weak refs
// encountered. Checks whether the WeakRef needs finalization.
function [[WeakRefCheckFinalization]](weakref){
  // This step is only run if there was an executor, so this weak
  // ref needs to be finalized if the target is not strongly
  // reachable. The target will be undefined if the target became
  // unreachable during a previous collection and we are
  // rebuilding the finalization queue.
  let target = weakref.[[Target]]);
  if (! [[IS_MARKED]](target) || target === void 0) {
    weakref.[[Target]] = void 0;
    [[QUEUE_FINALIZATION]](weakref);
  }
};

// Called by gc in its own turn, if and only if this weak reference is
// not already finalized. It is a no-op if the executor is null in
// order to coordinate with user code that clears the weak ref.
function [[WeakRefFinalize]](weakref){
  let exec = weakref.[[Executor]];
  if (typeof exec === 'function') {
    let hold = weakref.[[Holdings]];
    weakref.clear();
    exec(hold, weakRef);
  }
};

```

# Open questions

* Should get() on a weak ref whose target is collected return null or
  undefined?

The proposal is that it returns undefined. Collection effectively
returns a reference to the uninitialized state.

* Should the holdings default to null or undefined?

This answer is user-visible to the invocation of the executor
function. The proposed code currently defaults holdings to undefined.

* Should a weak ref be obligated to preserve the holdings or
  executor until the target is collected?

The proposal is that it is not required to retain any state it no
longer needs. It will certainly drop them after finalization, so it
doesn't have a long-term obligation. Following the design principle of
"allow implementations to collect as much as possible", the
implementation not be obligated to retain them. The implementation of
cross-realm references was modified to reflect this.

* Should weakRefs follow the class pattern?

Yes, except that it must preserve the security property that
construction of WeakRefs is restricted; the power to make a new weak
ref must not be available from instances.

* Should the weakRef be provided as an additional argument to the executor?

NOTE: The current proposal provides only the holdings to the executor
invocation. Adding the argument is upwards compatible, so this does
not need to be resolved immediately.

Additional usage examples are needed to determine the best pattern
here. In examples so far, there is already a mapping from holdings to
weakRef, so there's no need for the additional argument, but the
presence of the weakRef may simplify some other usage patterns, and
it's trivially available to provide.

# References

 * [Automatic storage-reclamation postmortem finalization process](http://www.google.com/patents/US5274804)
 * [pre-mortem finalization challenges](http://blogs.msdn.com/b/cbrumme/archive/2004/02/20/77460.aspx)
 * [Python Weak References](https://docs.python.org/2/library/weakref.html)
 * [Java WeakReference](https://docs.oracle.com/javase/7/docs/api/java/lang/ref/WeakReference.html)
 * [C# Weak References](https://msdn.microsoft.com/en-us/library/ms404247.aspx)
 * [Racket Ephmerons](https://docs.racket-lang.org/reference/ephemerons.html)
 * [Ephemerons](https://pdfs.semanticscholar.org/1803/a320584a35b797ca6089e8240393505ad410.pdf)
 * [Smalltalk VisualWorks WeakArrays](http://esug.org/data/Old/vw-tutorials/Adv_VWNotPrintable/Weak_References.pdf)
 * [WeakRefs for wasm-gc](https://github.com/WebAssembly/gc/blob/master/proposals/gc/Overview.md#possible-extension-weak-references-and-finalisation)
 * [Mozilla feature request issue](https://bugzilla.mozilla.org/show_bug.cgi?id=1367476)
 * [Bradley Meck's "weakref" branch of v8](https://github.com/bmeck/v8/commits/weakref) starting at commit named "[api] Prototype WeakRef implementation"


[ECMAScript language value]:      https://people.mozilla.org/~jorendorff/es6-draft.html#sec-ecmascript-language-types
[property key]:                   https://people.mozilla.org/~jorendorff/es6-draft.html#sec-object-type
[Completion Record]:              https://people.mozilla.org/~jorendorff/es6-draft.html#sec-completion-record-specification-type
