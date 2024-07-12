# `ArrayBuffer.prototype.transfer` and friends

Stage: 4 ([included in ES2024](https://github.com/tc39/ecma262/pull/3175)). **This repository is no longer active.**

Author: Shu-yu Guo (@syg)

Champion: Shu-yu Guo (@syg), Jordan Harband (@ljharb), Yagiz Nizipli (@anonrig)

## Introduction

`ArrayBuffer`s may be transferred and detached by HTML's serialization algorithms, but there lacks a programmatic JS API for the same expressivity. A programmatic API is useful for programming patterns such as transferring ownership of `ArrayBuffer`s, optimized reallocations (i.e. `realloc` semantics), and fixing resizable `ArrayBuffer`s into fixed-length ones. This proposal fills out this expressivity by adding new methods to `ArrayBuffer.prototype`.

This proposal is spun out of the [resizable buffers proposal](https://github.com/tc39/proposal-resizablearraybuffer/issues/113). At the time of spinning out, resizable buffers was Stage 3, and this proposal was demoted to Stage 2.


## API

```javascript
class ArrayBuffer {
  // ... existing stuff

  // Returns a new ArrayBuffer with the same byte content
  // as this buffer for [0, min(this.byteLength, newByteLength)],
  // then detaches this buffer.
  //
  // The maximum byte length and thus the resizability of this buffer
  // is preserved in the new ArrayBuffer.
  //
  // Any new memory is zeroed.
  //
  // If newByteLength is undefined, it is set to this.bytelength.
  //
  // Designed to be implementable as a copy-free move or a realloc.
  //
  // Throws a RangeError unless all of the following are satisfied:
  // - 0 <= newByteLength
  // - If this buffer is resizable, newByteLength <= this.maxByteLength
  transfer(newByteLength);

  // Like transfer, except always returns a non-resizable ArrayBuffer.
  transferToFixedLength(newByteLength);

  // Returns whether this ArrayBuffer is detached.
  get detached();
}
```

## Motivation and use cases

### Ownership

A "move and detach original `ArrayBuffer`" method can be used to implement ownership semantics when working with `ArrayBuffer`s. This is useful in many situations, such as disallowing other users from modifying a buffer when writing into it.

For example, consider the following example from @domenic from the original transfer proposal:

```javascript
function validateAndWrite(arrayBuffer) {
  // Do some asynchronous validation.
  await validate(arrayBuffer);

  // Assuming we've got here, it's valid; write it to disk.
  await fs.writeFile("data.bin", arrayBuffer);
}

const data = new Uint8Array([0x01, 0x02, 0x03]);
validateAndWrite(data.buffer);
setTimeout(() => {
  data[0] = data[1] = data[2] = 0x00;
}, 50);
```

Depending on the time taken for `await validate(arrayBuffer)`, the validation result may be stale due to the callback passed to `setTimeout`. A defensive approach would copy the input first, but this is markedly less performant:

```javascript
function validateAndWriteSafeButSlow(arrayBuffer) {
  // Copy first!
  const copy = arrayBuffer.slice();

  await validate(copy);
  await fs.writeFile("data.bin", copy);
}
```

With `transfer`, the ownership transfer can be succinctly expressed:

```javascript
function validateAndWriteSafeAndFast(arrayBuffer) {
  // Transfer to take ownership, which implementations can choose to
  // implement as a zero-copy move.
  const owned = arrayBuffer.transfer();

  // arrayBuffer is detached after this point.
  assert(arrayBuffer.detached);

  await validate(owned);
  await fs.writeFile("data.bin", owned);
}
```

### Realloc

The same `transfer` API, when passing a `newByteLength` argument, can double to have the same expressivity as [`realloc`](https://en.cppreference.com/w/c/memory/realloc). Operating systems often implement `realloc` more efficiently than a copy.

### Fixing resizable buffers to be fixed-length

The `transferToFixedLength` method is a variant of `transfer` that always returns a fixed-length `ArrayBuffer`. This is useful in cases when the new buffer no longers needs resizability, allowing implementations to free up virtual memory if resizable buffers were implemented in-place (i.e. address space is reserved up front).

### Checking detachedness

Owing to the [messy history](https://esdiscuss.org/topic/arraybuffer-neutering) of TypedArray and `ArrayBuffer` standardization, and preservation of web compatibility, TypedArray views on detached buffers throw for some operations (e.g. prototype methods), and return sentinel values (`0` or `undefined`) for others (e.g. indexed access and length).

The `detached` getter is added to authoritatively determine whether an `ArrayBuffer` is detached.

Currently, there isn't any performant way of detecting whether an `ArrayBuffer` is detached. The following implementation is an example of how the detachedness can be detected, but has some flaws in V8: functions with try catch blocks are not inlined in V8. See also [this Node internal comment](https://github.com/nodejs/node/blob/main/lib/querystring.js#L472).

```javascript
const assert = require('node:assert')

function isBufferDetached(buffer) {
  if (buffer.byteLength === 0) {
    try {
      new Uint8Array(buffer);
    } catch (error) {
      assert(error.name === 'TypeError');
      return true;
    }
  }
  return false
}
```

## FAQ and design rationale tradeoffs

### Why do both `transfer` and `transferToFixedLength` exist instead of a single, more flexible method?

Most folks seem to have the intuition that the move semantics, being the primary use case, ought to preserve resizability. Transferring `ArrayBuffer`s in HTML serialization preserves resizability, and symmetry with that is good for intuition.

A flexible `transfer` also complicates the API design for a more minority use case, thus the separate `transferToFixedLength` method.

### Why can't I pass a new `maxByteLength` to `transfer`?

One of the goals of `transfer`, in addition to detach semantics, is to be more efficiently implementable than a copy in user code. It is not clear to the author that for resizable buffers implemented in-place, reallocation of the virtual memory pages is possible and efficient on all popular operating systems.

And besides it adds complexity in service of a more minority use case. Resizable buffers ought to be allocated with sufficient maximum size from the start.

### If performance is the goal, why add new methods instead of implementing copy-on-write (CoW) as a transparent optimization?

In a word, security.

`ArrayBuffer`s are a very popular attack vector for exploiting JavaScript engines. An important security mitigation engines employ is to ensure the `ArrayBuffer`'s data pointer is constant and does not move. For this same reason, resizable buffers are specified to allow in-place implementation.

CoW `ArrayBuffer`s may be implemented by moving the data pointer. When the CoW `ArrayBuffer` is modified, new memory is allocated and the backing store is updated. However, this conflicts with the security mitigation.

It is possible to both implement copy-on-write `ArrayBuffer`s and keep the "fixed data pointer" security mitigation only with additional help from the underlying operating system: by mapping new virtual memory that is marked as CoW and initially point to the same physical pages as the source buffer. This technique is, however, not portable.

At this time, Google Chrome deems this mitigation important enough for security to not implement CoW `ArrayBuffer`s.

### How is `get detached()` used in browsers and in runtimes?

- Node.js [discussed adding a public API](https://github.com/nodejs/node/pull/45512) for `get detached()`
- WebKit has [`isDetached`](https://github.com/WebKit/WebKit/blob/6545977030f491dd87b3ae9fd666f6b949ae8a74/Source/JavaScriptCore/runtime/ArrayBuffer.h#L308) in internal `ArrayBuffer` class
- V8 added [`v8::ArrayBuffer::WasDetached`](https://github.com/v8/v8/commit/9df5ef70ff18977b157028fc55ced5af4bcee535) which was later [backported to Node.js](https://github.com/nodejs/node/pull/45568) and used in Node webstreams.

### How is `transfer` used in browsers and in runtimes?

Most browsers use `detach()` as the name for detaching `ArrayBuffer`s.

- V8 has `Detach()` as [`v8::ArrayBuffer::Detach()`](https://v8docs.nodesource.com/node-18.2/d5/d6e/classv8_1_1_array_buffer.html#abb7a2b60240651d16e17d02eb6f636cf)
 - WebKit has [`void detach()`](https://github.com/WebKit/WebKit/blob/6545977030f491dd87b3ae9fd666f6b949ae8a74/Source/JavaScriptCore/runtime/ArrayBuffer.h#L307)
 - Node.js [has an internal implementation](https://github.com/nodejs/node/blob/22c645d411b7cba9e7b0d578a3e7108147a5d89e/lib/internal/webstreams/util.js#L131) to support detaching internal buffers called `transferArrayBuffer`, and used in Node webstream.

## Open questions

### Do we really need `transferToFixedLength`?

Feels nice to round out the expressivity, but granted the use case here isn't as compelling as `transfer`.

## History and acknowledgment

Thanks to:
  - @domenic for https://github.com/domenic/proposal-arraybuffer-transfer/tree/HEAD~1
