This proposal introduces new metaprogramming concepts on private fields and methods. This document describes the invariants, guarantees and scope of this privacy as a whole, in light of this feature.

## Private fields and methods are *hidden*

Previous documentation used the term "hard private" to explain the guarantees, but that term may lead some to false conclusions. Here, we use the term *hidden*, defined as follows:

> A class element is *hidden* with the syntax `#name`, which makes it inaccessible (for reads, writes and observing its presence) except to those granted explicit access to it.

In this proposal, the following mechanisms grant access to hidden elements:
- Being lexically contained inside the place where the hidden element is defined, e.g., `class { #x; method() { this.#x } }`, the body of `method()` can see `#x`.
- Decorating an element, e.g., in `class { @dec #x; #y; }`, `@dec` can see `#x` but not `#y`.
- Decorating a class containing the element, e.g., in `@dec class { #x; #y; }`, `@dec` can see `#x` and `#y`.

Future proposals may also grant access to hidden elements, but this will always be through a mechanism which is syntactically apparent; implicit access will never be granted. For example, a `protected` contextual keyword could be added to classes which grants access to subclasses, and this access grant would be similar to that which decorators do.

## Object capability analysis

Private fields and methods are analogous to a WeakMap. Each hidden class element has an associated Private Name, which can be seen as in 1-1 correspondence with a WeakMap. Just as WeakMaps may be passed around, so can Private Names. Private Names are passed by default to element decorators and class decorators. With this understanding, private class elements can be used to achieve object capability-based security.

In practice, the ability to decorate hidden class elements is hoped to strengthen, not weaken, encapsulation by making it applicable in more cases, where a strict lexical-scoping-only regime would reduce applicability and force greater use of non-private mechanisms.

## Invoking decorators without trust

To invoke a decorator which should not be trusted with private names, it is possible to wrap the decorator so that it will not see Private Names as follows. This technique, of using JavaScript code to achieve isolation goals in conjunction with new JavaScript features by design, is nothing new; for example, Proxy and WeakMap were designed to be combined to build a membrane abstraction. Membranes are a small, trusted JavaScript library which enables interaction with untrusted code, much like this function.

```js
function restrict(decorator) {
  assert(typeof decorator === "function");
  return descriptor => {
    assert(descriptor[Symbol.toStringTag] === "Descriptor");
    switch (descriptor.kind) {
      case "class":
        let elements, privateElements;
        for (let i = 0; i < descriptor.elements.length; i++) {
          let element = descriptor.elements[i];
          let arr = typeof element.key === "object" ? privateElements : elements;
          arr[arr.length] = element;
        }
        let result = decorator({...descriptor, elements});
        for (let i = 0; i < privateElements.length; i++) {
          result.elements[result.elements.length] = privateElements[i];
        }
        return result;
      case "method":
      case "field":
        assert(typeof descriptor.key !== "object");
        return decorator(descriptor);
      default:
        assert(false);
    }
  }
}

// Example usage

@restrict(defineElement("my-element"))
class MyElement {
  #x;  // will not show up in defineElement
  y;   // will show up in defineElement
};
```