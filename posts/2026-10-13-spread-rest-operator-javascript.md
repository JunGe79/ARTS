# The `...` operator in JavaScript: spread vs rest

**Question:** JavaScript keeps using three dots — `...` — in object literals, arrays, and function signatures. It is the same three characters every time, so what does it actually mean, and why does it sometimes "unpack" a value and other times "collect" one?

**Answer:** `...` is one piece of syntax with two mirror-image jobs. When it appears where a value is *produced* — inside `{ }`, `[ ]`, or a function call — it is **spread**: it expands a collection into its individual pieces. When it appears where a value is *received* — the last parameter of a function, or the left side of a destructuring assignment — it is **rest**: it gathers the leftovers into a single array or object. Which one you are looking at depends entirely on the side of the `=` (or the call) it sits on.

## Spread: unpacking a value

Spread takes the elements or properties *out* of something and drops them in one by one somewhere else.

In an object literal, it copies all the keys of one object into another:

```js
const base = { x: 1, y: 2 };
const next = { ...base, z: 3 };   // → { x: 1, y: 2, z: 3 }
```

This is the most common real-world use: build a new object from an existing one, then add or override a few fields. A typical case is updating a settings object without mutating the original:

```js
const settings = {};   // nothing set yet

const updated = {
  ...settings,          // copy whatever is already there (here: nothing)
  useSourceCpu: true,   // force these two on
  useSourceMemory: true
};
// → { useSourceCpu: true, useSourceMemory: true }
```

If a later key duplicates an earlier one, **the last one wins** — so the two explicit lines above override anything the spread brought in with the same name.

Spread works on arrays too:

```js
const nums = [1, 2, 3];
const more = [0, ...nums, 4];   // → [0, 1, 2, 3, 4]
```

And in a function call, it turns an array back into individual arguments:

```js
const args = [1, 2, 3];
Math.max(...args);   // same as Math.max(1, 2, 3) → 3
```

## Rest: gathering values

Rest is the mirror image. When `...` is on the receiving side, it collects whatever is left into a single value.

As a function parameter, it gathers all remaining arguments into an array:

```js
function sum(...nums) {
  return nums.reduce((a, b) => a + b, 0);
}
sum(1, 2, 3);   // nums = [1, 2, 3] → 6
```

In a destructuring assignment, it collects the remaining properties into a new object:

```js
const { x, ...others } = { x: 1, y: 2, z: 3 };
// x = 1, others = { y: 2, z: 3 }
```

## How to tell which one you are reading

A quick rule that always works:

- `...` where a value is **expected or produced** — inside `{ }`, `[ ]`, or a function call → **spread** (unpacking).
- `...` where a value is **received** — the last function parameter, or the left side of a destructure → **rest** (gathering).

## The one gotcha: spread is a shallow copy

`{ ...obj }` copies the *top-level* keys only. Any nested object or array is copied by reference, not duplicated — so both the original and the copy point at the same inner object:

```js
const a = { nested: { k: 1 } };
const b = { ...a };
b.nested.k = 99;   // this also changes a.nested.k — same inner object!
```

If your object holds only flat values (strings, numbers, booleans), this never bites. The moment it holds other objects or arrays, remember that spread does not deep-clone them. When you genuinely need an independent copy of a nested structure, reach for `structuredClone(obj)` or copy the nested parts explicitly.

## Takeaways

- `...` is one operator with two roles: **spread** (unpack) and **rest** (gather).
- Spread lives where a value is produced — `{ ...obj }`, `[ ...arr ]`, `fn(...args)`.
- Rest lives where a value is received — `function f(...args)`, `const { x, ...rest } = obj`.
- The "copy then override" object pattern — `{ ...old, key: newValue }` — is the everyday workhorse; the last duplicate key wins.
- Spread is shallow. Nested objects are shared by reference; use `structuredClone` when you need a deep copy.
