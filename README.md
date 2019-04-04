# WeakReferences TC39 proposal

[Initial Spec-text](https://github.com/tc39/proposal-weakrefs/blob/master/specs/spec.md)

[Slides](https://github.com/tc39/proposal-weakrefs/blob/master/specs/Weak%20References%20for%20EcmaScript.pdf)

## Introduction

The WeakRef proposal encompasses two major new pieces of functionality:

1. creating _weak references_ to objects;
2. running user-defined _finalizers_ after such objects get garbage-collected.

### Weak references

A weak reference to an object is not enough to keep the object alive: when the only remaining references to a _referent_ (i.e. an object which is referred to by a weak reference) are weak references, garbage collection is free to destroy the referent and reuse its memory for something else. However, until the object is actually destroyed, the weak reference may return the object even if there are no strong references to it.

A primary use for weak references is to implement caches or mappings holding large objects, where it’s desired that a large object is not kept alive solely because it appears in a cache or mapping.

For example, if you have a number of large binary image objects (e.g. represented as `ArrayBuffer`s), you may wish to associate a name with each image.

If you used a `Map` to map names to images, or images to names, the image objects would remain alive just because they appeared as values or keys in the map.

If you used a `WeakMap` (where values can be garbage-collected once no references to the key object exist anymore) to map images to names, then each image object can be garbage-collected if there isn’t any other reference to it outside of the WeakMap. But, WeakMaps are not iterable by definition, and they cannot be cleared either. The WeakRefs proposal enables creating an “iterable + clearable WeakMap”:

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
      map: this.#weakMap,
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
    for (const [ref] of this.#refMap) {
      const key = ref.deref();
      if (key) yield key;
    }
  }

  values() {
    return this.#refMap.values();
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

Such “iterable WeakMaps” are already used in existing DOM APIs such as `document.getElementsByClassName` or `document.getElementsByTagName`, which return live `HTMLCollection`s. As such, the `WeakRef` proposal [adds missing functionality that helps explain existing web platform features](https://extensiblewebmanifesto.org/). [Issue #17 describes a similar use case.](https://github.com/tc39/proposal-weakrefs/issues/17)

The same concept can be used to implement custom caches for a limited number of items.

### Finalizers

_Finalization_ is the execution of code to clean up after an object that has become unreachable to program execution. User-defined _finalizers_ enable several new use cases, and can help prevent memory leaks.

For example, whenever you have a JavaScript object that is backed by something in WebAssembly, you might want to [run custom cleanup code (in WebAssembly)](https://github.com/lars-t-hansen/moz-sandbox/blob/master/refmap/ReferenceMap.md) when the object goes away.

Another use case is running custom clean-up code after Finalization use case: e.g. file handle: run some code once the file handle is closed

```js
class FileStream {
  static #cleanUp(iterable) {
    for (const file of iterable) {
      FileStream.#close(file.fd);
    }
  }
  
  static #close(fd) {
    fs.close(fd, (err) => /* handle error */);
  }

  static #finalizationGroup = new FinalizationGroup(this.#cleanUp);
  
  token = Symbol();
  data, file;
 
  constructor(fileName) {
    this.file = new File(fileName);
    FileStream.#finalizationGroup.register(this, this.file, token);
    // eagerly trigger async read of file contents into this.data
  }

  kill() {
    FileStream.#finalizationGroup.unregister(token);
    // other cleanup
  }

  *[Symbol.iterator]() { 
    // read data from this.data or this.file
  }
}

const fs = new FileStream('path/to/some/file');

for (const data of fs) {
  // do something
}
```

Finalizers can help prevent memory leaks. [Comlink](https://github.com/GoogleChromeLabs/comlink) wraps a value into a proxy so that any changes to the object are mirrored across (service) workers or iframes. However, there's no way to tell when such an object becomes unreachable and gets garbage-collected in one of the workers/iframes in order to then clean up the corresponding object on the other side, resulting in [memory leaks](https://github.com/GoogleChromeLabs/comlink/issues/63). With the finalization functionality in the WeakRef proposal, objects could send a message to the other side signalling that the object is being cleaned up.

## Deprecated docs that are still useful

[OLD Explanation](https://github.com/tc39/proposal-weakrefs/blob/master/specs/weakrefs.md)

[WeakRefGroups](https://github.com/tc39/proposal-weakrefs/wiki/WeakRefGroups): Likely refactoring of API surface

[Support for long wasm jobs](https://github.com/tc39/proposal-weakrefs/wiki/Support-for-long-wasm-jobs): Possible additional APIs for non-JS top levels that don't yield.

## Champions

* Dean Tribble
* Mark Miller
* Till Schneidereit 

## Status

* WeakReferences are now Stage 2
* Till Schneidereit has joined as a co-champion
* Till has a prototype of the new API in the SpiderMonkey console


## Todo
