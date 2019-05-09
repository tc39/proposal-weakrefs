# WeakReferences TC39 proposal

## Introduction

The WeakRef proposal encompasses two major new pieces of functionality:

1. creating _weak references_ to objects with the `WeakRef` class
2. running user-defined _finalizers_ after objects are garbage-collected, with the `FinalizationGroup` class

These interfaces can be used independently or together, depending on the use case.

## A note of caution

[Garbage collectors](https://en.wikipedia.org/wiki/Garbage_collection_(computer_science)) are complicated. If an application or library depends on GC cleaning up a WeakRef or calling a finalizer in a timely, predictable manner, it's likely to be disappointed: the cleanup may happen much later than expected, or not at all. Sources of variability include:
- One object might be garbage-collected much sooner than another object, even if they become unreachable at the same time, e.g., due to generational collection.
- Garbage collection work can be split up over time using incremental and concurrent techniques.
- Various runtime heuristics can be used to balance memory usage, responsiveness.
- The JavaScript engine may hold references to things which look like they are unreachable (e.g., in closures, or inline caches).
- Different JavaScript engines may do these things differently, or the same engine may change its algorithms across versions.

For this reason, the [W3C TAG Design Principles](https://w3ctag.github.io/design-principles/#js-gc) recommend against creating APIs that expose garbage collection. It's best if `WeakRef`s and `FinalizationGroup`s are used as a way to avoid excess memory usage, or as a backstop against certain bugs, rather than as a normal way to clean up external resources or observe what's allocated.

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

### Weak references and the event loop

When you call `deref()` on a weak reference and you get back a value---i.e., not `undefined`---that value is protected from garbage collection for the rest of the turn ([Job](https://tc39.github.io/ecma262/#sec-jobs-and-job-queues)).  Therefore this function will always return true:

```js
function test(x) {
  const ref = new WeakRef(x)
  const y = ref.deref();
  const z = ref.deref();
  return y === z;
}
```

However this function may return true or false:

```js
async function test(x) {
  const ref = new WeakRef(x)
  const y = ref.deref();
  await new Promise(resolve => setTimeout(resolve));
  const z = ref.deref();
  return y === z;
}
```

This is because `y` and `z` are computed in separate turns.

## Finalizers

_Finalization_ is the execution of code to clean up after an object that has become unreachable to program execution. User-defined _finalizers_ enable several new use cases, and can help prevent memory leaks when managing resources that the garbage collector doesn't know about.

### Another note of caution

Finalizers are tricky business and it is best to avoid them.  They can be invoked at unexpected times, or not at all---for example, they are not invoked when closing a browser tab or on process exit.  They don’t help the garbage collector do its job; rather, they are a hindrance.  Furthermore, they perturb the garbage collector’s internal accounting.  The GC decides to scan the heap when it thinks that it is necessary, after some amount of allocation.  Finalizable objects almost always represent an amount of allocation that is invisible to the garbage collector.  The effect can be that the actual resource usage of a system with finalizable objects is higher than what the GC thinks it should be.

All that said, sometimes finalizers are the right answer to a problem.  The following examples show a few important problems that would be difficult to solve without finalizers.

### Exposing WebAssembly memory to JavaScript

For example, whenever you have a JavaScript object that is backed by something in WebAssembly, you might want to run custom cleanup code (in WebAssembly or JavaScript) when the object goes away. A [previous proposal](https://github.com/lars-t-hansen/moz-sandbox/blob/master/refmap/ReferenceMap.md) exposed a collection of weak references, with the idea that finalization actions could be taken by periodically checking if they are still alive. This proposal includes a first-class concept of finalizers in order to give developers a way to avoid that repeated scanning.

For example, imagine if you have a big `WebAssembly.Memory` object, and you want to create an allocator to give fixed-size portions of it to JavaScript. In some cases, it may be practical to explicitly free this memory, but typically, JavaScript code passes around references freely, without thinking about ownership. So it's helpful to be able to rely on the garbage collector to release this memory.

```js
function makeAllocator(size, length) {
  const freeList = Array.from({length}, (v, i) => size * i);
  const memory = new ArrayBuffer(size * length);
  const finalizationGroup = new FinalizationGroup(
    iterator => freeList.unshift(...iterator));
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

This code uses a few features of the `FinalizationGroup` API:
- An object can have a finalizer referenced by calling the `register` method of `FinalizationGroup`. In this case, two arguments are passed to the `register` method:
  - The object whose lifetime we're concerned with. Here, that's the `Uint8Array`
  - A “holdings” value, which is used to represent that object when cleaning it up in the finalizer. In this case, the holdings are an integer corresponding to the offset within the `WebAssembly.Memory` object.
- The `FinalizationGroup` constructor is called with a callback as an argument. This callback is called with an iterator of the holdings values.

The finalizer callback is called *after* the object is garbage collected, a pattern which is sometimes called "post-mortem". For this reason, a separate "holdings" value is put in the iterator, rather than the original object--the object's already gone, so it can't be used.

The `FinalizationGroup` callback is passed an iterator of holdings to give that callback control over how much work it wants to process. The callback may pull in only part of the iterator, and in this case, the rest of the work would be "saved for later". The callback is not called during execution of other JavaScript code, but rather "in between turns"; it is currently proposed to be restricted to run after all of of the `Promise`-related work is done, right before turning control over to the event loop.

### Releasing external resources as a backstop

Another use case is running custom clean-up code after Finalization use case: e.g. file handle: run some code once a file is no longer in use, in case nothing points to it anymore. Note, it’s still encouraged to use an explicit method call to clean up the resource; this is just a backup.

```js
class FileStream {
  static #cleanUp(holdings) {
    for (const file of holdinds) {
      file.close(); // cleans up the file descriptor
    }
  }
  
  static #finalizationGroup = new FinalizationGroup(this.#cleanUp);

  #token = {};
  #file;

  constructor(fileName) {
    this.#file = new File(fileName);
    FileStream.#finalizationGroup.register(this, this.#file, this.#token);
    // eagerly trigger async read of file contents into this.data
  }

  close() {
    FileStream.#finalizationGroup.unregister(this.#token);
    File.close(this.#file);
    // other cleanup
  }

  *[Symbol.iterator]() {
    // read data from this.#data or this.#file
  }
}

const fs = new FileStream('path/to/some/file');

for (const data of fs) {
  // do something
}
fs.close();
```

Note, it’s not a good idea to *depend* on the `FinalizationGroup` ever calling the cleanup action, as this is inherently unreliable, as described at the beginning of this README. However, it can be a reasonable fallback to handle the case where a resource was dropped on the floor due to a bug. If nothing references the file anymore, no one is going to close it! Just try not to rely on it.

This code uses an additional feature of finalizers: unregistration. It works like this:

- When registering the `FileStream` instance with the finalization group, a third argument is passed: the unregistration token.
- The `FinalizationGroup.prototype.unregister` method is passed the unregistration token.
- Afterwards, the cleanup callback is never called with the holdings of this `FileStream`.

Unregistration is useful here to avoid holding a reference to the file past when it was already closed, to avoid closing it multiple times. Imagine--once the file's been closed, the same file descriptor may have been reused for something unrelated, and closing it a second time could mess up unrelated code!

### Avoid memory leaks for cross-worker proxies

In a browser with [web workers](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Using_web_workers), a programmer can create a system with multiple JavaScript processes, and thus multiple isolated heaps and multiple garbage collectors.  Developers often want to be able to address a "remote" object from some other process, for example to be able to manipulate the DOM from a worker.  A common solution to this problem is to implement a proxy library; two examples are [Comlink](https://github.com/GoogleChromeLabs/comlink) and [via.js](https://github.com/AshleyScirra/via.js).

In a system with proxies and processes, remote proxies need to keep local objects alive, and vice versa.  Usually this is implemented by having each process keep a table mapping remote descriptors to each local object that has been proxied.  However, these entries should be removed from the table when there are no more remote proxies.  With the finalization functionality in the WeakRef proposal, libraries like via.js can [send a message when a proxy becomes collectable](https://github.com/AshleyScirra/via.js#memory-management), to inform the object's process that the object is no longer referenced remotely.  Without finalization, via.js and other remote-proxy systems have to fall back to leaking memory, or to manual resource management.

## Using `WeakRef`s and `FinalizationGroup`s together

It sometimes makes sense to use `WeakRef` and `FinalizationGroup` together. There are several kinds of data structures that want to weakly point to a value, and do some kind of cleanup when that value goes away.  Note however that weak refs are cleared when their object is collected, but their associated `FinalizationGroup` cleanup handler only runs in a later microtask; programming idioms that use weak refs and finalizers on the same object need to mind the gap.

### Weak caches

In the initial example from this README, `makeWeakCached` used a `Map` whose values were wrapped in `WeakRef` instances.  This allowed the cached values to be collected, but leaked memory in the form of the entries in the map.  A more complete version of `makeWeakCached` uses finalizers to fix this memory leak.

```js
// Fixed version that doesn't leak memory.
function makeWeakCached(f) {
  const cache = new Map();
  const cleanup = new FinalizationGroup(iterator => {
    for (const key of iterator) {
      // See note below on concurrency considerations.
      const ref = cache.get(key);
      if (ref && !ref.deref()) cache.delete(key);
    }
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
    cleanup.register(fresh, key, key);
    return fresh;
  };
}

var getImageCached = makeWeakCached(getImage);
```

This example illustrates two important considerations about finalizers:

 1. Finalizers introduce concurrency between the "main" program and the cleanup callbacks.  The weak cache cleanup function has to check if the "main" program re-added an entry to the map between the time that a cached value was collected and the time the cleanup function runs, to avoid deleting live entries.  Likewise when looking up a key in the ref map, it's possible that the value has been collected but the cleanup callback hasn't run yet.
 2. Given that finalizers can behave in surprising ways, they are best deployed behind careful abstractions that prevent misuse, like `makeWeakCached` above.  A profusion of `FinalizationGroup` uses spread throughout a code-base is a code smell.

### Iterable WeakMaps

In certain advanced cases, `WeakRef`s and `FinalizationGroup`s can be very effective complements. For example, WeakMaps have the limitation that they cannot be iterated over or cleared. The WeakRefs proposal enables creating an “iterable + clearable WeakMap”:

Such “iterable WeakMaps” are already used in existing DOM APIs such as `document.getElementsByClassName` or `document.getElementsByTagName`, which return live `HTMLCollection`s. As such, the `WeakRef` proposal [adds missing functionality that helps explain existing web platform features](https://extensiblewebmanifesto.org/). [Issue #17 describes a similar use case.](https://github.com/tc39/proposal-weakrefs/issues/17)

```js
class IterableWeakMap {
  #weakMap = new WeakMap();
  #refMap = new Map();
  #finalizationGroup = new FinalizationGroup(IterableWeakMap.#cleanup);

  static #cleanup(iterator) {
    for (const { map, ref } of iterator) {
      map.delete(ref);
    }
  }

  constructor(iterable) {
    for (const [key, value] of iterable) {
      this.set(key, value);
    }
  }

  set(key, value) {
    const ref = new WeakRef(key);

    this.#weakMap.set(key, { value, ref });
    this.#refMap.set(ref, value);
    this.#finalizationGroup.register(key, {
      map: this.#refMap,
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
    this.#refMap.delete(entry.ref);
    this.#finalizationGroup.unregister(entry.ref);
    return true;
  }

  *[Symbol.iterator]() {
    for (const [ref, value] of this.#refMap) {
      const key = ref.deref();
      if (key) yield [key, value];
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

### Finalizers, visibility, and turns: final notes of caution

As mentioned above, finalizers run after an object is collected, in a separate microtask.  An object can be collected at any point where it is no longer needed to compute the result of a computation.  JavaScript implementations have a wide latitude to determine what this means in practice.  For example, consider this async function:

```js
async function process(x) {
  const p = x.getPtr();
  await new Promise(resolve => setTimeout(resolve));
  return handle(p);
}
```

If `x` has a finalizer that invalidates the value returned by `getPtr()`, an implementation is free to collect `x` at any point after the `getPtr()` call, and to finalize it during the `await`, making the `handle(p)` call operate on an invalid pointer.  From a language perspective, the fact that `x` was an argument to a function does not prevent it from being collected.  Additionally the `return handle(p)` call is a tail call, which an engine may implement as throwing away any stack frame with the `x` binding, which is another opportunity for `x` to become collectible.

In practice, the fact that finalizers are delayed until a future microtask will prevent most early-finalization bugs.  However, developers writing libraries that use `FinalizationGroup` should be wary of the interactions of async functions with finalizers, and avoid exposing invalidatable internals of finalizable objects.

## Historical documents

- [OLD Explanation](https://github.com/tc39/proposal-weakrefs/blob/master/history/weakrefs.md) of a previous version of the proposal
- [WeakRefGroups](https://github.com/tc39/proposal-weakrefs/wiki/WeakRefGroups): Previously proposed interface
- [Support for long wasm jobs](https://github.com/tc39/proposal-weakrefs/wiki/Support-for-long-wasm-jobs): Background on the motivation for the `cleanupSome` method
- [Previous Spec-text](https://github.com/tc39/proposal-weakrefs/blob/master/history/spec.md) for an earlier draft of the proposal
- [Slides](https://github.com/tc39/proposal-weakrefs/blob/master/history/Weak%20References%20for%20EcmaScript.pdf): Some design considerations that went into this proposal

## Champions

* Dean Tribble
* Mark Miller
* Till Schneidereit

## Status

* WeakReferences are now Stage 2
* Till has a prototype of the new API in the SpiderMonkey console
* Available behind the --harmony-weak-refs flag in V8, by Marja Hölttä
