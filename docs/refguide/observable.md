---
title: Creating observables
sidebar_label: Creating observables
hide_title: true
---

<script async type="text/javascript" src="//cdn.carbonads.com/carbon.js?serve=CEBD4KQ7&placement=mobxjsorg" id="_carbonads_js"></script>

# Creating observables

Properties, entire objects, arrays, Maps and Sets can all be made observable.
The basics of making objects observable is by specifying an annotation per property by using `makeObservable`.
The most important annotations are:

-   `observable` define a trackable field that stores state.
-   `action` mark a method as action that will modify state.
-   `computed` mark a getters that will derive new facts from the state and cache its output.

Collections such as arrays, Maps and Sets are made observable automatically.

## makeObservable

Usage:

-   `makeObservable(target, annotations?, options?)`

MakeObservable can be used to trap _existing_ object properties and make them observable. Any JavaScript object (including class instances) can be passed into `target`.
Typically `makeObservable` is used in the constructor of a class, and its first argument is `this`.
The `annotations` argument then maps property names to [annotations](#available-annotations) to apply (see below for the list of available annotations).

Methods that derive information and take arguments (for example `findUsersOlderThan(age: number): User[]`) don't need any annotation.
Their read operations will still be tracked when they are called from a reaction, but their output won't be memoized to avoid memory leaks (see also [mobx-utils:computedFn](https://github.com/mobxjs/mobx-utils#computedfn)).

<div class="detail">

MakeObservable can only annotate properties declared by its own class definition. If a sub- or superclass introduces observable fields, it will have to call `makeObservable` for those properties itself.

Decorators note: When using [decorators](../best/decorators), the `annotations` argument can be omitted.

TypeScript note: When decorating private properties in TypeScript, you can pass the private property names as generic argument to `makeObservable` to suppress the compile error about the field not existing like this: `makeObservable<"myPrivateField" | "myOtherPrivateField>(this, { myPrivateField: observable })`

</div>

<!--DOCUSAURUS_CODE_TABS-->
<!--makeObservable-->

```javascript
import { makeObservable, observable, computed, action } from "mobx"

class Doubler {
    value

    constructor(value) {
        makeObservable(this, {
            value: observable,
            double: computed,
            increment: action
        })
        this.value = value
    }

    get double() {
        return this.value * 2
    }

    increment() {
        this.value++
    }
}
```

<!--makeAutoObservable-->

```javascript
import { makeAutoObservable } from "mobx"

class Doubler {
    value

    constructor(value) {
        makeAutoObservable(this)
        this.value = value
    }

    get double() {
        return this.value * 2
    }

    increment() {
        this.value++
    }
}
```

<!--observable-->

```javascript
import { observable } from "mobx"

function createDoubler(value) {
    return observable({
        value,
        get double() {
            return this.value * 2
        },
        increment() {
            this.value++
        }
    })
}
```

<!--END_DOCUSAURUS_CODE_TABS-->

## `makeAutoObservable`

Usage:

-   `makeAutoObservable(target, overrides?, options?)`

`makeAutoObservable` is like `makeObservable` on steroids, as it infers all properties by default. You can still use `overrides` to override the default behavior with specific annotations.
In particular `false` can be used to exclude a property or method from being processed entirely.
See the code tabs above for an example.
The `makeAutoObservable` function can be more compact and easier to maintain than using `makeObservable`, since new members don't have to be mentioned explicitly.
However, `makeAutoObservable` cannot be used on classes that have super- or are subclassed.

Inference rules:

-   Any (inherited) member that contains a `function` value will be annotated with `autoAction`.
-   Any `get`ter will be annotated with `computed`.
-   Any other _own_ field will be marked with `observable`.
-   Members marked with `false` in the `overrides` argument will not be annotated. Use this for for example read only fields such as identifiers.

<div class="detail">

When you call `makeObservable` or `makeAutoObservable` all properties you want to be observable
_must_ exist on the instance already. Either by [declaring](https://github.com/tc39/proposal-class-fields) them (recommended, as done above) or otherwise by assigning them _before_ calling `makeAutoObservable`

Providing annotations must be done unconditionally, as this makes it possible to cache the inference results.

</div>

## `observable`

Usage:

-   `observable(source, overrides?, options?)`

The `observable` annotation can also be called as function to make an entire object observable at once.
The `source` object will be cloned and all members will be made observable, similar to how it would be done by `makeAutoObservable`.
Likewise, an `overrides` map can be provided to specify the annotations of specific members.
See the above code block for an example.

The object returned by `observable` will be a Proxy, which means that properties that are added later to the object will be picked up and made observable as well (except when [proxy usage](../refguide/configure.md#proxy-support) is disabled).

The `observable` method can also be called with collections types like [arrays](../refguide/api.md#observablearray), [Maps](../refguide/api.md#observablemap) and [Sets](../refguide/api.md#observableset). Those will be cloned as well and converted into their observable counterpart.

<div class="detail">

Primitive values cannot be made observable by MobX since they are immutable in JavaScript (but they can be [boxed](../refguide/api.md#observablebox)).
Class instances will never be made observable automatically by passing them to `observable` or assigning them to an `observable` property.
Making class members observable is considered the responsibility of the class constructor.

</div>

## Available annotations

| Annotation                         | Description                                                                                                                                                                                                                |
| ---------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `observable`<br/>`observable.deep` | Defines a trackable field that stores state. Any value assigned to an `observable` field will be made recursively observable as well, if possible. That is, if and only if the value is a plain object, array, Map or Set. |
| `observable.ref`                   | Like `observable`, but only reassignments will be tracked. The assigned values themselves won't be made observable automatically. Use this if you intend to store for example immutable data in an observable field.       |
| `observable.shallow`               | Like `observable.ref` but for collections; any collection assigned will be made observable, but the contents of the collection itself won't become observable.                                                             |
| `observable.struct`                | Like `observable`, except that any assigned value that is structurally equal to the current value will be ignored.                                                                                                         |
| `action`                           | Mark a method as action that will modify state. See [action](../refguide/action.md) for details.                                                                                                                           |
| `action.bound`                     | Like action, but will also bind the action to the instance so that `this` will always be set.                                                                                                                              |
| `computed`                         | Can be used on a [getter](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/get) to declare it as a derived value that can be cached. See [computed](../refguide/computed.md) for details.       |
| `computed.struct`                  | Like `computed`, except that if after recomputing the result is structurally equal to the previous result, no observers will be notified                                                                                   |
| `true`                             | Infer the best annotation. See [makeAutoObservable](#makeautoobservable).                                                                                                                                                  |
| `false`                            | Explicitly do not annotate this property.                                                                                                                                                                                  |
| `autoAction`                       | Should not be used explicitly, but is used under the hood by `makeAutoObservable` to mark methods that can act as action or derivation, based on their calling context                                                     |

## The `options` argument

The above APIs take an optional `options` argument which is an object that supports the following options:

-   `autoBind: true`. Automatically binds all created actions to the instance.
-   `deep: false`. Use `observable.ref` by default, rather than `observable` to create new observable members
-   `name: <string>`. Gives the object a debug name that is printed in error messages and reflection APIs.
