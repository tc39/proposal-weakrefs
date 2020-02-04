[TOC]

# Specification [Work In Progress]

The specification for weak references and finalization will include
the `WeakCell` type, the `WeakRef` type, the `WeakFactory` type, some 
abstract functions, and some characteristics of the runtime garbage 
collector.

## WeakFactory Objects

A `WeakFactory` is an object that creates `WeakCells` and `WeakRefs`  for a
related group of target objects, and implements and manages the cleanup after
each of those targets have been reclaimed.

### The WeakFactory Constructor

The Ream constructor is the %WeakFactory% intrinsic object and the initial value of the *WeakFactory* property of the global object. When called as a constructor it creates and initializes a new WeakFactory object. WeakFactory is not intended to be called as a function and will throw an exception when called in that manner.

The WeakFactory constructor is designed to be subclassable. It may be used as the value in an extends clause of a class definition. Subclass constructors that intend to inherit the specified WeakFactory behaviour must include a super call to the WeakFactory constructor to create and initialize the subclass instance with the internal state necessary to support the WeakFactory.prototype built-in methods.

When WeakFactory is called with argument _cleanup_, it performs the following steps:

1. If NewTarget is *undefined*, throw a *TypeError* exception.
1. Let _O_ be ? OrdinaryCreateFromConstructor(NewTarget, "%WeakFactoryPrototype%", « [[WeakFactory]] »).
1. If _cleanup_ is *undefined*, then
    1. Set _O_.[[Cleanup]] to undefined.
1. Else
    1. Set _O_.[[Cleanup]] to _cleanup_.
1. Set _O_.[[ActiveCells]] to a new empty List.
1. Set _O_.[[HasDirty]] to false.
1. Perform ! RegisterWeakFactory(_O_).
1. Return _O_

### The WeakFactory.prototype Object

The initial value of WeakMap.prototype is the intrinsic object
`%WeakFactoryPrototype%`. The `%WeakFactoryPrototype%` object is an
ordinary object and its `[[Prototype]]` internal slot is the
`%ObjectPrototype%` intrinsic object. In addition,
`WeakFactory.prototype` has the following properties:

### WeakFactory.prototype.makeRef(target, holdings = undefined)

The operation `WeakFactory.prototype.makeRef` with arguments `target`
and `holdings`  is used to create `WeakRef` objects.

It performs the following steps:

1. Let _O_ be the this value.
1. If Type(_O_) is not Object, throw a TypeError exception.
1. If _O_ does not have all of the internal slots of a WeakFactory Instance, throw a TypeError exception.
1. If Type(target) is not Object, throw a TypeError exception
1. If SameValue(target, holdings), throw a TypeError exception
1. If SameRealm(target, **this**), then
    1. Let _weakRef_ be ObjectCreate(%WeakRefPrototype%, ([[Factory]], [[WeakRefTarget]], [[Holdings]])).
    1. Set _weakRef_.[[Factory]] to _O_.
    1. Set _weakRef_.[[WeakRefTarget]] to _target_.
    1. Set _weakRef_.[[Holdings]] to _holdings_.
    1. Set all weakRef's garbage collection internal operations.
    1. Perform ! KeepDuringJob(_target_).
1. Else
    1. Let _weakRef_ be ObjectCreate(%WeakRefPrototype%, ([[Factory]], [[WeakRefTarget]], [[Holdings]], [[Strong]])).
    1. Set _weakRef_.[[Factory]] to _O_.
    1. Set _weakRef_.[[WeakRefTarget]] to _target_.
    1. Set _weakRef_.[[Holdings]] to _holdings_.
    1. Set _weakRef_.[[Strong]] to _target_.
1. Add _weakref_ to _O_.[[ActiveCells]].
1. Return weakRef.

### WeakFactory.prototype.shutdown()

Disable all cleanup for `WeakCell`s created by this `WeakFactory`. This is used to
enable  a subsystem using this `WeakFactory` to shutdown completely without
triggering unnecessary finalization.

1. Let _O_ be the this value.
2. If Type(_O_) is not Object, throw a TypeError exception.
3. If _O_ does not have all of the internal slots of a WeakFactory
    Instance, throw a TypeError exception.
4. Set _O_.[[Cleanup]] to undefined.
4. Set _O_.[[ActiveCells]] to undefined.
4. Set _O_.[[HasDirty]] to false.
7. Return undefined.

### WeakFactory.prototype [ @@toStringTag ]

The initial value of the @@toStringTag property is the string value
"WeakFactory".

### Properties of WeakFactory Instances

WeakFactory instances are ordinary objects that inherit properties
from the WeakFactory.prototype intrinsic object. WeakFactory
instances are initially created with the internal slots listed in
Table K.

*Table K — Internal Slots of WeakFactory Instances*

| Internal Slot | Description |
| ----- | ----- |
| [[Cleanup]] | An optional reference to a function |
| [[ActiveCells]] | WeakRefs that are Active or Dirty. |
| [[HasDirty]] | True if some of ActiveCells has become Dirty. |

## WeakCell Objects

A `WeakCell` is an object that is used to refer to a target object
without preserving it from garbage collection, and to enable code to
be run to clean up after the target is garbage collected. Correct
instances can only be created via a `WeakFactory` object because the
instances must contain a pointer to such a factory.

### The WeakCell.prototype Object

All `WeakCell` objects inherit properties from the
`%WeakCellPrototype%` intrinsic object.  The `WeakCell.prototype`
object is an ordinary object and its `[[Prototype]]` internal slot is
the `%ObjectPrototype%` intrinsic object. In addition,
`WeakCell.prototype` has the following properties:

### WeakCell.prototype.clear()

WeakCell.prototype.clear() performs the following steps:

1. Let _O_ be the **this** value.
1. If Type(_O_) is not Object, throw a TypeError exception.
1. If _O_ does not have all of the internal slots of a WeakCell
   Instance, throw a TypeError exception.
1. Let _factory be _O_.[[Factory]].
1. If _factory_ is not **undefined**.
    1. Remove _O_ from _factory_.[[ActiveCells]].
    1. Set _O_.[[WeakRefTarget]] to undefined.
    1. Set _O_.[[Factory]] to undefined.
    1. Set _O_.[[Holdings]] to undefined.
1. Return **undefined**.

### get WeakCell.prototype.holdings

**WeakCell.prototype.holdings** is an accessor property whose set
accessor function is **undefined**. Its get accessor function performs
the following steps:

1. Let _O_ be the **this** value.
1. If Type(_O_) is not Object, throw a TypeError exception.
1. If _O_ does not have all of the internal slots of a WeakCell Instance, throw a TypeError exception.
1. Return _O_.[[Holdings]].

### WeakCell.prototype [ @@toStringTag ]

The initial value of the @@toStringTag property is the string value
"WeakCell".

### Properties of WeakCell Instances

WeakCell instances are ordinary objects that inherit properties from
the WeakCell.prototype intrinsic object. WeakCell instances are
initially created with the internal slots listed in Table L.

*Table L — Internal Slots of WeakCell Instances*

| Internal Slot | Description |
| ----- | ----- |
| [[WeakRefTarget]] | The reference to the target object |
| [[Factory]] | AThe factory that created the object |
| [[Holdings]] | Any value, passed as a parameter to the cleanup function in the factory |

## WeakRef Objects

A `WeakRef` is a kind of WeakCell that can also be dereferenced. Correct
instances can only be created via a WeakFactory object because the instances
must contain a pointer to such a factory.

### The WeakRef.prototype Object

All `WeakRef` objects inherit properties from the
`%WeakRefPrototype%` intrinsic object.  The `%WeakRefPrototype%`
object is an ordinary object and its `[[Prototype]]` internal slot is
the `%WeakCellPrototype%` intrinsic object. In addition,
`WeakRef.prototype` has the following properties:

### WeakRef.prototype.deref()

WeakRef.prototype.deref() performs the following steps:

1. Let _O_ be the this value.
2. If Type(_O_) is not Object, throw a TypeError exception.
3. If _O_ does not have all of the internal slots of a WeakRef Instance, throw a TypeError exception.
6. Let _a_ be the value of the [[WeakRefTarget]] internal slot of _O_.
4. If _a_ is not **undefined**, perform ! KeepDuringJob(_a_).
7. Return _a_.

### WeakRef.prototype [ @@toStringTag ]

The initial value of the @@toStringTag property is the string value
"WeakRef".

### Properties of WeakRef Instances

All properties on WeakRef instances are inherited from the WeakCell.prototype "superclass".

## Job and Agent Extensions
 
This section describes extensions to existing standard elements.

### Agent Additions

Each Agent contains a new property [[WeakFactories]] which points weakly (via
records TBD) to the WeakFactory instances created in that agent.

### WeakCellJobs Queue

Add a separate "WeakCellJobs" JobQueue for all cleanup operations in the agent. (Section 8.4)

## Abstract Operations

This section decribes additional abstract operations to support WeakCells and cleanup.

### SameRealm Abstract Operation

When the abstract operation `SameRealm` is called with two references, _a_ and
_b_, it returns true if they were created in the same Realm.

### KeepDuringJob Abstract Operation

When the abstract operation `KeepDuringJob` is called with a target
object  reference, it adds the target to an identity Set that will
point strongly at the target until the end of the current Job. This
may be abstractly implemented as a Set in the current Job that becomes
unreachable when the Job ends because the Job becomes unreachable, or
as a Set in the current Agent that gets cleared at the end of each
Job.

### WeakTargetReclaimed

The garbage collector calls this abstract operation when it has made a target
object unreachable and set all weakRef [[WeakRefTarget]] fields that point to it to
**undefined**.

When the abstract operation `WeakTargetReclaimed` is called with a
_weakcell_ argument (by the garbage collector), it performs the following
steps:

1. Assert: Type(_weakcell_) is Object.
1. Assert: _weakcell_ has all of the internal slots of a WeakCell Instance.
1. Let _factory_ be _weakcell_.[[Factory]].
1. If _factory_ is not **undefined**, then Set _factory_.[[HasDirty]] to **true**.
1. Return **undefined**.

### RegisterWeakFactory

Add a `WeakFactory` to the [[WeakFactories]] list of the current agent.

### DoAgentFinalization

When the abstract operation `DoAgentFinalization` is called with an argument
_agent_, it performs the following steps:

1. Assert: Type(_agent_) is Object.
1. Assert: _agent_ has all of the internal slots of an Agent Instance.
1. For each _factory_ in _agent_.[[WeakFactories]]
    1. If _factory_.[[HasDirty]], then 
        1. Perform EnqueueJob("WeakCellJobs", WeakFactoryCleanupJob, « _factory_ »).

### WeakFactoryCleanupJob(_factory_)

The job `WeakFactoryCleanupJob` with parameter _factory_, and then performs
the following steps:

1. Assert: Type(_factory_) is Object.
1. Assert: _factory_ has all of the internal slots of a WeakFactory Instance.
1. Let _iterator_ be ObjectCreate(%ReclaimedIteratorPrototype%, « [[Factory]] »).
1. Set _iterator_.[[WeakFactory]] = _factory_.
1. Let _next_ be a new built-in function defined in `WeakFactoryCleanupIterator` **next**.
1. Perform CreateMethodProperty(_iterator_, **next**, _next_).
1. Call _factory_.[[Cleanup]]\(_iterator_).
1. Return **undefined**

### WeakFactoryCleanupIterator.next()

The WeakFactoryCleanupIterator.next method is a standard built-in function object that performs the following steps:

TBD: search the _factory_.[[ActiveCells]] for a dirty WeakCell
TDB: how is done set correctly?
