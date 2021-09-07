# WeakRefs TC39 proposal

## Status

* WeakRef and FinalizationRegistry are now Stage 4, since the July 2020 TC39 meeting
* V8 -- shipping [Chrome 84](https://v8.dev/features/weak-references)
* Spidermonkey -- shipping [Firefox 79](https://developer.mozilla.org/en-US/docs/Mozilla/Firefox/Releases/79)
* JavaScriptCore -- shipping [Safari 14.1](https://webkit.org/blog/11648/new-webkit-features-in-safari-14-1)
* engine262 -- [Initial patch](https://github.com/engine262/engine262/commit/0516660577b050b17e619af4c3f053c7506c670d), now all landed
* XS -- [Shipping in Moddable XS 9.0.1](https://github.com/Moddable-OpenSource/moddable-xst/releases/tag/v9.0.1)

## Introduction

The WeakRef proposal encompasses two major new pieces of functionality:

1. creating _weak references_ to objects with the `WeakRef` class
2. running user-defined _finalizers_ after objects are garbage-collected, with the `FinalizationRegistry` class

These interfaces can be used independently or together, depending on the use case.

For developer reference documentation, see [`reference.md`](reference.md).

## A note of caution

This proposal contains two advanced features, `WeakRef` and `FinalizationRegistry`. Their correct use takes careful thought, and they are best avoided if possible.

[Garbage collectors](https://en.wikipedia.org/wiki/Garbage_collection_(computer_science)) are complicated. If an application or library depends on GC cleaning up a WeakRef or calling a finalizer in a timely, predictable manner, it's likely to be disappointed: the cleanup may happen much later than expected, or not at all. Sources of variability include:
- One object might be garbage-collected much sooner than another object, even if they become unreachable at the same time, e.g., due to generational collection.
- Garbage collection work can be split up over time using incremental and concurrent techniques.
- Various runtime heuristics can be used to balance memory usage, responsiveness.
- The JavaScript engine may hold references to things which look like they are unreachable (e.g., in closures, or inline caches).
- Different JavaScript engines may do these things differently, or the same engine may change its algorithms across versions.
- Complex factors may lead to objects being held alive for unexpected amounts of time, such as use with certain APIs.

Important logic should not be placed in the code path of a finalizer. Doing so could create user-facing issues triggered by memory management bugs, or even differences between JavaScript garbage collector implementations. For example, if data is saved persistently solely from a finalizer, then a bug which accidentally keeps an additional reference around could lead to data loss.

For this reason, the [W3C TAG Design Principles](https://w3ctag.github.io/design-principles/#js-gc) recommend against creating APIs that expose garbage collection. It's best if `WeakRef` objects and `FinalizationRegistry` objects are used as a way to avoid excess memory usage, or as a backstop against certain bugs, rather than as a normal way to clean up external resources or observe what's allocated.

## Weak references

A weak reference to an object is not enough to keep the object alive: when the only remaining references to a _referent_ (i.e. an object which is referred to by a weak reference) are weak references, garbage collection is free to destroy the referent and reuse its memory for something else. However, until the object is actually destroyed, the weak reference may return the object even if there are no strong references to it.

A primary use for weak references is to implement caches or mappings holding large objects, where it’s desired that a large object is not kept alive solely because it appears in a cache or mapping.

For example, if you have a number of large binary image objects (e.g. represented as `ArrayBuffer`s), you may wish to associate a name with each image. Existing data structures just don't do what's needed here:
- If you used a `Map` to map names to images, or images to names, the image objects would remain alive just because they appeared as values or keys in the map.
- `WeakMap`s are not suitable for this purpose either: they are weak over their *keys*, but in this case, we need a structure which is weak over its *values*.

Instead, we can use a `Map` whose values are `WeakRef` objects, which point to the `ArrayBuffer`. This way, we avoid holding these `ArrayBuffer` objects in memory longer than they would be otherwise: it's a way to find the image object if it's still around, but if it gets garbage collected, we'll regenerate it. This way, less memory is used in some situations.

```js
// This technique is incomplete; see below.
function makeWeakCached(f) {
  const cache = new Map();
  return key => {
    const ref = cache.get(key);
    if (ref) {
      const cached = ref.deref();
      if (cached !== undefined) return cached;
    }

    const fresh = f(key);
    cache.set(key, new WeakRef(fresh));
    return fresh;
  };
}

var getImageCached = makeWeakCached(getImage);
```

This technique can help avoid spending a lot of memory on `ArrayBuffer`s that nobody is looking at anymore, but it still has the problem that, over time, the `Map` will fill up with strings which point to a `WeakRef` whose referent has already been collected. One way to address this is to periodically scavenge the cache and clear out dead entries. Another way is with finalizers, which we’ll come back to at the end of the article.

A few elements of the API are visible in this example:

- The `WeakRef` constructor takes an argument, which has to be an object, and returns a weak reference to it.
- `WeakRef` instances have a `deref` method that returns one of two values:
  - The object passed into the constructor, if it’s still available.
  - `undefined`, if nothing else was pointing to the object and it was already garbage-collected.

## Finalizers

_Finalization_ is the execution of code to clean up after an object that has become unreachable to program execution. User-defined _finalizers_ enable several new use cases, and can help prevent memory leaks when managing resources that the garbage collector doesn't know about.

### Another note of caution

Finalizers are tricky business and it is best to avoid them.  They can be invoked at unexpected times, or not at all---for example, they are not invoked when closing a browser tab or on process exit.  They don’t help the garbage collector do its job; rather, they are a hindrance.  Furthermore, they perturb the garbage collector’s internal accounting.  The GC decides to scan the heap when it thinks that it is necessary, after some amount of allocation.  Finalizable objects almost always represent an amount of allocation that is invisible to the garbage collector.  The effect can be that the actual resource usage of a system with finalizable objects is higher than what the GC thinks it should be.

The proposed specification allows conforming implementations to skip calling finalization callbacks for any reason or no reason. Some reasons why many JS environments and implementations may omit finalization callbacks:
- If the program shuts down (e.g., process exit, closing a tab, navigating away from a page), finalization callbacks typically don't run on the way out. (Discussion: [#125](https://github.com/tc39/proposal-weakrefs/issues/125))
- If the FinalizationRegistry becomes "dead" (approximately, unreachable), then finalization callbacks registered against it might not run. (Discussion: [#66](https://github.com/tc39/proposal-weakrefs/issues/66))

All that said, sometimes finalizers are the right answer to a problem.  The following examples show a few important problems that would be difficult to solve without finalizers.

### Locating and responding to external resource leaks

Finalizers can locate external resource leaks. For example, if an open file is garbage collected, the underlying operating system resource could be leaked. Although the OS will likely free the resources when the process exits, this sort of leak could make long-running processes eventually exhaust the number of file handles available. To catch these bugs, a `FinalizationRegistry` can be used to log the existence of file objects which are garbage collected before being closed.

The `FinalizationRegistry` class represents a group of objects registered with a common finalizer callback. This construct can be used to inform the developer about the never-closed files.

```js
class FileStream {
  static #cleanUp(heldValue) {
    console.error(`File leaked: ${file}!`);
  }

  static #finalizationGroup = new FinalizationRegistry(FileStream.#cleanUp);

  #file;

  constructor(fileName) {
    this.#file = new File(fileName);
    FileStream.#finalizationGroup.register(this, this.#file, this);
    // eagerly trigger async read of file contents into this.data
  }

  close() {
    FileStream.#finalizationGroup.unregister(this);
    File.close(this.#file);
    // other cleanup
  }

  async *[Symbol.iterator]() {
    // read data from this.#file
  }
}

const fs = new FileStream('path/to/some/file');

for await (const data of fs) {
  // do something
}
fs.close();
```

Note, it's not a good idea to close files automatically through a finalizer, as this technique is unreliable and may lead to resource exhaustion. Instead, explicit release of resources (e.g., though `try`/`finally`) is recommended. For this reason, this example logs errors rather than transparently closing the file.

This example shows usage of the whole `FinalizationRegistry` API:
- An object can have a finalizer referenced by calling the `register` method of `FinalizationRegistry`. In this case, three arguments are passed to the `register` method:
  - The object whose lifetime we're concerned with. Here, that's `this`, the `FileStream` object.
  - A held value, which is used to represent that object when cleaning it up in the finalizer. Here, the held value is the underlying `File` object. (Note: the held value should not have a reference to the weak target, as that would prevent the target from being collected.)
  - An unregistration token, which is passed to the `unregister` method when the finalizer is no longer needed. Here we use `this`, the `FileStream` object itself, since `FinalizationRegistry` doesn't hold a strong reference to the unregister token.
- The `FinalizationRegistry` constructor is called with a callback as an argument. This callback is called with a held value.

The finalizer callback is called *after* the object is garbage collected, a pattern which is sometimes called "post-mortem". For this reason, the `FinalizerRegistry` callback is called with a separate held value, rather than the original object--the object's already gone, so it can't be used.

In the above code sample, the `fs` object will be unregistered as part of the `close` method, which will mean that the finalizer will not be called, and there will be no error log statement. Unregistration can be useful to avoid other sorts of "double free" scenarios.

### Exposing WebAssembly memory to JavaScript

Whenever you have a JavaScript object that is backed by something in WebAssembly, you might want to run custom cleanup code (in WebAssembly or JavaScript) when the object goes away. A [previous proposal](https://github.com/lars-t-hansen/moz-sandbox/blob/master/refmap/ReferenceMap.md) exposed a collection of weak references, with the idea that finalization actions could be taken by periodically checking if they are still alive. This proposal includes a first-class concept of finalizers in order to give developers a way to avoid that repeated scanning.

For example, imagine if you have a big `WebAssembly.Memory` object, and you want to create an allocator to give fixed-size portions of it to JavaScript. In some cases, it may be practical to explicitly free this memory, but typically, JavaScript code passes around references freely, without thinking about ownership. So it's helpful to be able to rely on the garbage collector to release this memory. A `FinalizationRegistry` can be used to free the memory.

```js
function makeAllocator(size, length) {
  const freeList = Array.from({length}, (v, i) => size * i);
  const memory = new ArrayBuffer(size * length);
  const finalizationGroup = new FinalizationRegistry(
    held => freeList.unshift(held));
  return { memory, size, freeList, finalizationGroup };
}

function allocate(allocator) {
  const { memory, size, freeList, finalizationGroup } = allocator;
  if (freeList.length === 0) throw new RangeError('out of memory');
  const index = freeList.shift();
  const buffer = new Uint8Array(memory, index * size, size);
  finalizationGroup.register(buffer, index);
  return buffer;
}
```

This code uses a few features of the `FinalizationRegistry` API:
- An object can have a finalizer referenced by calling the `register` method of `FinalizationRegistry`. In this case, two arguments are passed to the `register` method:
  - The object whose lifetime we're concerned with. Here, that's the `Uint8Array`
  - A held value, which is used to represent that object when cleaning it up in the finalizer. In this case, the held value is an integer corresponding to the offset within the `WebAssembly.Memory` object.
- The `FinalizationRegistry` constructor is called with a callback as an argument. This callback is called with a held value.

The `FinalizationRegistry` callback is called potentially multiple times, once for each registered object that becomes dead, with a relevant held value. The callback is not called during execution of other JavaScript code, but rather "in between turns". The engine is free to batch calls, and a batch of calls only runs after all of the Promises have been processed. How the engine batches callbacks is implementation-dependent, and how those callbacks intersperse with Promise work should not be depended upon.

### Avoid memory leaks for cross-worker proxies

In a browser with [web workers](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Using_web_workers), a programmer can create a system with multiple JavaScript processes, and thus multiple isolated heaps and multiple garbage collectors.  Developers often want to be able to address a "remote" object from some other process, for example to be able to manipulate the DOM from a worker.  A common solution to this problem is to implement a proxy library; two examples are [Comlink](https://github.com/GoogleChromeLabs/comlink) and [via.js](https://github.com/AshleyScirra/via.js).

In a system with proxies and processes, remote proxies need to keep local objects alive, and vice versa.  Usually this is implemented by having each process keep a table mapping remote descriptors to each local object that has been proxied.  However, these entries should be removed from the table when there are no more remote proxies.  With the finalization functionality in the WeakRef proposal, libraries like via.js can [send a message when a proxy becomes collectable](https://github.com/AshleyScirra/via.js#memory-management), to inform the object's process that the object is no longer referenced remotely.  Without finalization, via.js and other remote-proxy systems have to fall back to leaking memory, or to manual resource management.

Note: This kind of setup cannot collect cycles across workers. If in each worker the local object holds a reference to a proxy for the remote object, then the remote descriptor for the local object prevents the collection of the proxy for the remote object. None of the objects can be collected automatically when code outside the proxy library no longer references them. To avoid leaking, cycles across isolated heaps must be explicitly broken.

## Using `WeakRef` objects and `FinalizationRegistry` objects together

It sometimes makes sense to use `WeakRef` and `FinalizationRegistry` together. There are several kinds of data structures that want to weakly point to a value, and do some kind of cleanup when that value goes away.  Note however that weak refs are cleared when their object is collected, but their associated `FinalizationRegistry` cleanup handler only runs in a later task; programming idioms that use weak refs and finalizers on the same object need to mind the gap.

### Weak caches

In the initial example from this README, `makeWeakCached` used a `Map` whose values were wrapped in `WeakRef` instances.  This allowed the cached values to be collected, but leaked memory in the form of the entries in the map.  A more complete version of `makeWeakCached` uses finalizers to fix this memory leak.

```js
// Fixed version that doesn't leak memory.
function makeWeakCached(f) {
  const cache = new Map();
  const cleanup = new FinalizationRegistry(key => {
    // See note below on concurrency considerations.
    const ref = cache.get(key);
    if (ref && !ref.deref()) cache.delete(key);
  });

  return key => {
    const ref = cache.get(key);
    if (ref) {
      const cached = ref.deref();
      // See note below on concurrency considerations.
      if (cached !== undefined) return cached;
    }

    const fresh = f(key);
    cache.set(key, new WeakRef(fresh));
    cleanup.register(fresh, key);
    return fresh;
  };
}

var getImageCached = makeWeakCached(getImage);
```

This example illustrates two important considerations about finalizers:

 1. Finalizers introduce concurrency between the "main" program and the cleanup callbacks.  The weak cache cleanup function has to check if the "main" program re-added an entry to the map between the time that a cached value was collected and the time the cleanup function runs, to avoid deleting live entries.  Likewise when looking up a key in the ref map, it's possible that the value has been collected but the cleanup callback hasn't run yet.
 2. Given that finalizers can behave in surprising ways, they are best deployed behind careful abstractions that prevent misuse, like `makeWeakCached` above.  A profusion of `FinalizationRegistry` uses spread throughout a code-base is a code smell.

### Iterable WeakMaps

In certain advanced cases, `WeakRef` objects and `FinalizationRegistry` objects can be very effective complements. For example, WeakMaps have the limitation that they cannot be iterated over or cleared. The WeakRefs proposal enables creating an “iterable + clearable WeakMap”:

Such “iterable WeakMaps” are already used in existing DOM APIs such as `document.getElementsByClassName` or `document.getElementsByTagName`, which return live `HTMLCollection`s. As such, the `WeakRef` proposal [adds missing functionality that helps explain existing web platform features](https://extensiblewebmanifesto.org/). [Issue #17 describes a similar use case.](https://github.com/tc39/proposal-weakrefs/issues/17)

```js
class IterableWeakMap {
  #weakMap = new WeakMap();
  #refSet = new Set();
  #finalizationGroup = new FinalizationRegistry(IterableWeakMap.#cleanup);

  static #cleanup({ set, ref }) {
    set.delete(ref);
  }

  constructor(iterable) {
    for (const [key, value] of iterable) {
      this.set(key, value);
    }
  }

  set(key, value) {
    const ref = new WeakRef(key);

    this.#weakMap.set(key, { value, ref });
    this.#refSet.add(ref);
    this.#finalizationGroup.register(key, {
      set: this.#refSet,
      ref
    }, ref);
  }

  get(key) {
    const entry = this.#weakMap.get(key);
    return entry && entry.value;
  }

  delete(key) {
    const entry = this.#weakMap.get(key);
    if (!entry) {
      return false;
    }

    this.#weakMap.delete(key);
    this.#refSet.delete(entry.ref);
    this.#finalizationGroup.unregister(entry.ref);
    return true;
  }

  *[Symbol.iterator]() {
    for (const ref of this.#refSet) {
      const key = ref.deref();
      if (!key) continue;
      const { value } = this.#weakMap.get(key);
      yield [key, value];
    }
  }

  entries() {
    return this[Symbol.iterator]();
  }

  *keys() {
    for (const [key, value] of this) {
      yield key;
    }
  }

  *values() {
    for (const [key, value] of this) {
      yield value;
    }
  }
}


const key1 = { a: 1 };
const key2 = { b: 2 };
const keyValuePairs = [[key1, 'foo'], [key2, 'bar']];
const map = new IterableWeakMap(keyValuePairs);

for (const [key, value] of map) {
  console.log(`key: ${JSON.stringify(key)}, value: ${value}`);
}
// key: {"a":1}, value: foo
// key: {"b":2}, value: bar

for (const key of map.keys()) {
  console.log(`key: ${JSON.stringify(key)}`);
}
// key: {"a":1}
// key: {"b":2}

for (const value of map.values()) {
  console.log(`value: ${value}`);
}
// value: foo
// value: bar

map.get(key1);
// → foo

map.delete(key1);
// → true

for (const key of map.keys()) {
  console.log(`key: ${JSON.stringify(key)}`);
}
// key: {"b":2}
```

Remember to be cautious with use of powerful constructs like this iterable WeakMap. Web APIs designed with semantics analogous to these are widely considered to be legacy mistakes. It’s best to avoid exposing garbage collection timing in your applications, and to use weak references and finalizers only where a problem cannot be reasonably solved in other ways.

#### WeakMaps remain fundamental

It is not possible to re-create a `WeakMap` simply by using a `Map` with `WeakRef` objects as keys: if the value in such a map references its key, the entry cannot be collected. A real `WeakMap` implementation uses [ephemerons](http://www.jucs.org/jucs_14_21/eliminating_cycles_in_weak/jucs_14_21_3481_3497_barros.pdf) to allow the garbage collector to handle such cycles.\
This is the reason the [`IterableWeakMap` example](#iterable-weakmaps) keeps the value in a `WeakMap` and only puts the `WeakRef` in a `Set` for iterations. If the value had instead been added to a `Map` such as `this.#refMap.set(ref, value)`, then the following would have leaked:
```js
let key = { foo: 'bar' };
const map = new IterableWeakMap(key, { data: 123, key });
```

### Scheduling of finalizers and consistency of multiple `.deref()` calls

There are several conditions where implementations may call finalization callbacks later or not at all. The WeakRefs proposal works with host environments (e.g., HTML, Node.js) to define exactly how the `FinalizationRegistry` callback is scheduled. The intention is to coarsen the granularity of observability of garbage collection, making it less likely that programs will depend too closely on the details of any particular implementation.

In [the definition for HTML](https://github.com/whatwg/html/pull/4571), the callback is scheduled in [task queued in the event loop](https://html.spec.whatwg.org/multipage/webappapis.html#event-loops). What this means is that, on the web, finalizers will never interrupt synchronous JavaScript, and that they also won't be interspersed to Promise reactions. Instead, they are run only after JavaScript yields to the event loop.

The WeakRefs proposal guarantees that multiple calls to `WeakRef.prototype.deref()` return the same result within a certain timespan: either all should return `undefined`, or all should return the object. In HTML, this timespan runs until a [microtask checkpoint](https://html.spec.whatwg.org/multipage/webappapis.html#perform-a-microtask-checkpoint), where HTML performs a microtask checkpoint when the JavaScript execution stack becomes empty, after all Promise reactions have run.

## Historical documents

- [OLD Explanation](https://github.com/tc39/proposal-weakrefs/blob/master/history/weakrefs.md) of a previous version of the proposal
- [WeakRefGroups](https://github.com/tc39/proposal-weakrefs/wiki/WeakRefGroups): Previously proposed interface
- [Previous Spec-text](https://github.com/tc39/proposal-weakrefs/blob/master/history/spec.md) for an earlier draft of the proposal
- [Slides](https://github.com/tc39/proposal-weakrefs/blob/master/history/Weak%20References%20for%20EcmaScript.pdf): Some design considerations that went into this proposal

## Champions

* Dean Tribble
* Mark Miller
* Till Schneidereit
* Sathya Gunasekaran
* Daniel Ehrenberg

## Status

* WeakRefs are now Stage 4
* Chrome 84
* Firefox 79
* Safari 14.1
* [Available in Moddable XS](https://github.com/Moddable-OpenSource/moddable-xst/releases/tag/v9.0.1)
