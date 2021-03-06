---
description: Setting up the API to talk to your WebAssembly.
---

# Exports and imports

## Exports

Exports work very much like in TypeScript, with the notable difference that exports from an entry file also become the exports of the resulting WebAssembly module.

### Functions

{% tabs %}
{% tab title="index.ts" %}
```typescript
export function add(a: i32, b: i32): i32 {
  return a + b
}
```
{% endtab %}
{% endtabs %}

### Globals

{% tabs %}
{% tab title="index.ts" %}
```typescript
export const foo = 1
export var bar = 2
```
{% endtab %}
{% endtabs %}

Exporting or importing mutable globals requires that the engine supports the post-MVP and now standardized [Import/Export mutable globals proposal](https://github.com/WebAssembly/mutable-global), which is enabled by default.

### Classes

If an entire class \(possibly part of a namespace\) is exported from an entry file, its visible members will become distinct module exports using a JSDoc-like naming scheme that [the loader](loader.md) understands to make a nice object structure of. For example

{% tabs %}
{% tab title="index.ts" %}
```typescript
export namespace foo {
  export class Bar {
    a: i32 = 1
    getA(): i32 { return this.a }
  }
}
```
{% endtab %}
{% endtabs %}

will yield the raw function exports

* foo.Bar\#**constructor**
* foo.Bar\#**get:a**
* foo.Bar\#**set:a**
* foo.Bar\#**getA**

which the loader combines back to a `foo.Bar` object:

{% tabs %}
{% tab title="index.js" %}
```javascript
var bar = new myModule.foo.Bar()
bar.a = 2
console.log(bar.a)
```
{% endtab %}
{% endtabs %}

Under the hood and without the loader, it'd look like this:

```javascript
var thisBar = myModule["foo.Bar#constructor"](0)
myModule["foo.Bar#set:a"](thisBar, 2)
console.log(myModule["foo.Bar#get:a"](thisBar))
```

As one can see the `this` argument must be provided as an additional first argument to the raw functions. No argument or `0` as the first argument to a constructor indicates that the constructor is expected to allocate on its own \(not extending a class with existing memory\). Of course one usually doesn't have to deal with these internals since the loader will already take care of it.

## Imports

With [WebAssembly ES Module Integration](https://github.com/WebAssembly/esm-integration) still in the pipeline, imports utilize the ambient context currently. For example

{% tabs %}
{% tab title="env.ts" %}
```typescript
export declare function doSomething(foo: i32): void
```
{% endtab %}
{% endtabs %}

creates an import of a function named `doSomething` from the `env` module, because that's the name of the file it lives is. It is also possible to use namespaces:

{% tabs %}
{% tab title="foo.ts" %}
```typescript
export declare namespace console {
  export function logi(i: i32): void
  export function logf(f: f64): void
}
```
{% endtab %}
{% endtabs %}

This will import the functions `console.logi` and `console.logf` from the `foo` module. Bonus: Don't forget `export`ing namespace members if you'd like to call them from outside the namespace.

A typical pattern is to `declare` the interface of each external module in its own file, using the name of the module as its file name, so the external module can be imported as if it was just another source file:

```typescript
import { doSomething } from "./env";
import { console } from "./foo";
```

### Custom naming

Where automatic naming is not sufficient, the `@external` decorator can be used to give an element another external name:

{% tabs %}
{% tab title="bar.ts" %}
```typescript
@external("doSomethingElse")
export declare function doSomething(foo: i32): void
// imports doSomethingElse from bar as doSomething

@external("foo", "baz")
export declare function doSomething(foo: i32): void
// imports baz from foo as doSomething
```
{% endtab %}
{% endtabs %}

## On values crossing the boundary

More complex values than the basic integer and floating point types, like class instances or function references, are represented by an index or pointer within the WebAssembly module. The compiler knows how to work with these because it also knows the concrete type associated with the value. On the outside, however, for example in JS-code interacting with a WebAssembly module, all that's seen is the basic value.

* An **instance of a class** is a pointer to the structure in WebAssembly memory.
* A **function reference** is the index of the function in the WebAssembly table.

This is true in both directions, hence also applies when providing a value to a WebAssembly import. The most common structures like `String`, `ArrayBuffer` and the typed arrays are documented in [memory internals](../details/memory.md#internals), and custom classes adhere to [class layout](../details/interoperability.md#class-layout).

