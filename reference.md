# Introduction

The WeakRefs proposal adds WeakRefs and FinalizationRegistries to the standard library.

* [WeakRef](#weakref) - weak references
* [FinalizationRegistry](#finalizationregistry) - registries providing cleanup callbacks

This page provides developer reference information for them.

----

# Warning - Avoid Where Possible

Please read [the WeakRef proposal's explainer](./README.md) for several words of caution regarding weak references and cleanup callbacks. Their correct use takes careful thought, and they are best avoided if possible. It's also important to avoid relying on any specific behaviors not guaranteed by the specification. When, how, and whether garbage collection occurs is down to the implementation of any given JavaScript engine. Any behavior you observe in one engine may be different in another engine, in another version of the same engine, or even in a slightly different situation with the same version of the same engine. Garbage collection is a hard problem that JavaScript engine implementers are constantly refining and improving their solutions to.

----

# `WeakRef`

A WeakRef instance contains a weak reference to an object, which is called its *target* or *referent*. A *weak reference* to an object is a reference to it that does not prevent it being reclaimed by the garbage collector. In contrast, a normal (or *strong*) reference keeps an object in memory. When an object no longer has any strong references to it, the JavaScript engine's garbage collector may destroy the object and reclaim its memory. If that happens, you can't get the object from a weak reference anymore.

**WeakRef Contents:**

* [Overview](#weakref-overview)
* [WeakRef API](#weakref-api)
    * [The `WeakRef` Constructor](#the-weakref-constructor)
    * [`WeakRef.prototype.deref`](#weakrefprototypederef)
* [Notes](#notes-on-weakrefs)

## WeakRef Overview

To create a WeakRef, you use the `WeakRef` constructor, passing in the target object (the object you want a weak reference to):

```js
let ref = new WeakRef(someObject);
```

Then at some point you stop using `someObject` (it goes out of scope, etc.) while keeping the WeakRef instance (`ref`). At that point, to get the object from a WeakRef instance, use its `deref` method:

```js
let obj = ref.deref();
if (obj) {
    // ...use `obj`...
}
```

`deref` returns the object, or `undefined` if the object is no longer available.

## WeakRef API

### The `WeakRef` Constructor

The `WeakRef` constructor creates a WeakRef instance for the given `target` object:

```js
ref = new WeakRef(target)
```

**Parameters**

* `target` - The target object for the WeakRef instance.

**Returns**

* The WeakRef instance.

**Notes**

* `WeakRef` is a constructor function, so it throws if called as a function rather than as a constructor.
* `WeakRef` is designed to be subclassed, and so can appear in the `extends` of a `class` definition and similar.

### `WeakRef.prototype.deref`

The `deref` method returns the WeakRef instance's target object, or `undefined` if the target object has been reclaimed:

```js
targetOrUndefined = weakRef.deref();
```

**Parameters**

* *(none)*

**Returns**

* The target object of the WeakRef, or `undefined`.

**Errors**

* *(none)*

## Notes on WeakRefs

Some notes on WeakRefs:

* If your code has just created a WeakRef for a target object, or has gotten a target object from a WeakRef's `deref` method, that target object will not be reclaimed until the end of the current JavaScript [job](https://tc39.es/ecma262/#job) (including any promise reaction jobs that run at the end of a script job). That is, you can only "see" an object get reclaimed between turns of the event loop. This is primarily to avoid making the behavior of any given JavaScript engine's garbage collector apparent in code&nbsp;&mdash; because if it were, people would write code relying on that behavior, which would break when the garbage collector's behavior changed. (Garbage collection is a hard problem; JavaScript engine implementers are constantly refining and improving how it works.)
* If multiple WeakRefs have the same target, they're consistent with one another. The result of calling `deref` on one of them will match the result of calling `deref` on another of them (in the same job), you won't get the target object from one of them but `undefined` from another.
* If the target of a WeakRef is also in a [`FinalizationRegistry`](#finalizationregistry), the WeakRef's target is cleared at the same time or before any cleanup callback associated with the registry is called; if your cleanup callback calls `deref` on a WeakRef for the object, it will receive `undefined`.
* You cannot change the target of a WeakRef, it will always only ever be the original target object or `undefined` when that target has been reclaimed.
* A WeakRef might never return `undefined` from `deref`, even if nothing strongly holds the target, because the garbage collector may never decide to reclaim the object.

# `FinalizationRegistry`

A finalization registry provides a way to request that a *cleanup callback* get called at some point after an object registered with the registry has been *reclaimed* (garbage collected). (Cleanup callbacks are sometimes called *finalizers*.)

**NOTE:** Cleanup callbacks should not be used for essential program logic. See [*Notes on Cleanup Callbacks*](#notes-on-cleanup-callbacks) for more.

**FinalizationRegistry Contents**:

* [Overview](#finalizationregistry-overview)
* [Notes on Cleanup Callbacks](#notes-on-cleanup-callbacks)
* [FinalizationRegistry API](#finalizationregistry-api)
    * [The `FinalizationRegistry` Constructor](#the-finalizationregistry-constructor)
    * [`FinalizationRegistry.prototype.register`](#finalizationregistryprototyperegister)
    * [`FinalizationRegistry.prototype.unregister`](#finalizationregistryprototypeunregister)

## FinalizationRegistry Overview

A `FinalizationRegistry` instance (a "registry") lets you get *cleanup callbacks* after objects registered with the registry are reclaimed. You create the registry passing in the callback:

```js
const registry = new FinalizationRegistry(heldValue => {
    // ....
});
```

Then you register any objects you want a cleanup callback for by calling the `register` method, passing in the object and a *held value* for it:

```js
registry.register(theObject, "some value");
```

The registry does not keep a strong reference to the object, as that would defeat the purpose (if the registry held it strongly, the object would never be reclaimed).

If `theObject` is reclaimed, your cleanup callback may be called at some point in the future with the *held value* you provided for it (`"some value"` in the above). The held value can be any value you like: a primitive or an object, even `undefined`. If the held value is an object, the registry keeps a *strong* reference to it (so it can pass it to your cleanup callback later).

If you might want to unregister an object later, you pass a third value, which is the *unregistration token* you'll use later when calling the registry's `unregister` function to unregister the object. The registry only keeps a weak reference to the unregister token.

It's common to use the object itself as the unregister token, which is just fine:

```js
registry.register(theObject, "some value", theObject);
// ...some time later, if you don't care about `theObject` anymore...
registry.unregister(theObject);
```

It doesn't have to be the same object, though; it can be a different one:

```js
registry.register(theObject, "some value", tokenObject);
// ...some time later, if you don't care about `theObject` anymore...
registry.unregister(tokenObject);
```

## Notes on Cleanup Callbacks

Developers shouldn't rely on cleanup callbacks for essential program logic. Cleanup callbacks may be useful for reducing memory usage across the course of a program, but are unlikely to be useful otherwise.

A conforming JavaScript implementation, even one that does garbage collection, is not required to call cleanup callbacks. When and whether it does so is entirely down to the implementation of the JavaScript engine. When a registered object is reclaimed, the cleanup callbacks associated with any registries it's registered with may be called some time later, or not at all.

It's likely that major implementations will call cleanup callbacks at some point during execution, but those calls may be substantially after the related object was reclaimed.

There are also situations where even implementations that normally call cleanup callbacks are unlikely to call them:

* When the JavaScript program shuts down entirely (for instance, closing a tab in a browser).
* When the `FinalizationRegistry` instance itself is no longer reachable by JavaScript code.

## FinalizationRegistry API

### The `FinalizationRegistry` Constructor

Creates a finalization registry with an associated cleanup callback:

```js
registry = new FinalizationRegistry(cleanupCallback)
```

**Parameters**

* `cleanupCallback` - The callback to call after an object in the registry has been reclaimed. This is required and must be callable.

**Returns**

* The FinalizationRegistry instance.

**Notes**

* `FinalizationRegistry` is designed to be subclassed, and so can appear in the `extends` of a `class` definition and similar.

**Example**

```js
const registry = new FinalizationRegistry(heldValue => {
    // ...use `heldValue`...
});
```

### `FinalizationRegistry.prototype.register`

Registers an object with the registry:

```js
registry.register(target, heldValue[, unregisterToken])
```

**Parameters**

* `target` - The target object to register.
* `heldValue` - The value to pass to the finalizer for this object. This cannot be the `target` object.
* `unregisterToken` - (Optional) The token to pass to the `unregister` method to unregister the target object. If provided (and not `undefined`), this must be an object. If not provided, the target cannot be unregistered.

**Returns**

* *(none)*

**Examples**

The following registers the target object referenced by `target`, passing in the held value `"some value"` and passing the target object itself as the unregistration token:

```js
registry.register(target, "some value", target);
```

The following registers the target object referenced by `target`, passing in another object as the held value, and not passing in any unregistration token (which means `target` can't be unregistered):

```js
registry.register(target, {"useful": "info about target"});
```

### `FinalizationRegistry.prototype.unregister`

Unregisters an object from the registry:

```js
registry.unregister(unregisterToken)
```

**Parameters**

* `unregisterToken` - The token that was used as the `unregisterToken` argument when calling `regiter` to register the target object.

**Returns**

* *(none)*

**Notes**

When a target object has been reclaimed and its cleanup callback has been called, it is no longer registered in the registry. There is no need to all `unregister` in your cleanup callback. Only call `unregister` if you haven't received a cleanup callback and no longer need to receive one.

**Example**

This example shows registering a target object using that same object as the unregister token, then later unregistering it via `unregister`:

```js
class Thingy {
    #cleanup = label => {
    //         ^^^^^−−−−− held value
        console.error(
            `The \`release\` method was never called for the object with the label "${label}"`
        );
    };
    #registry = new FinalizationRegistry(this.#cleanup);

    /**
     * Constructs a `Thingy` instance. Be sure to call `release` when you're done with it.
     *
     * @param   label       A label for the `Thingy`.
     */
    constructor(label) {
        //                            vvvvv−−−−− held value
        this.#registry.register(this, label, this);
        //          target −−−−−^^^^         ^^^^−−−−− unregister token
    }

    /**
     * Releases resources held by this `Thingy` instance.
     */
    release() {
        this.#registry.unregister(this);
        //                        ^^^^−−−−− unregister token
    }
}
```

This example shows registering a target object using a different object as its unregister token:

```js
class Thingy {
    #file;
    #cleanup = file => {
    //         ^^^^−−−−− held value
        console.error(
            `The \`release\` method was never called for the \`Thingy\` for the file "${file.name}"`
        );
    };
    #registry = new FinalizationRegistry(this.#cleanup);

    /**
     * Constructs a `Thingy` instance for the given file. Be sure to call `release` when you're done with it.
     *
     * @param   filename    The name of the file.
     */
    constructor(filename) {
        this.#file = File.open(filename);
        //                            vvvvv−−−−− held value
        this.#registry.register(this, label, this.#file);
        //          target −−−−−^^^^         ^^^^^^^^^^−−−−− unregister token
    }

    /**
     * Releases resources held by this `Thingy` instance.
     */
    release() {
        if (this.#file) {
            this.#registry.unregister(this.#file);
            //                        ^^^^^^^^^^−−−−− unregister token
            File.close(this.#file);
            this.#file = null;
        }
    }
}
```

<!-- for scrolling to links near the end -->
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
