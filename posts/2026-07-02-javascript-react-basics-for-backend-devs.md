# JavaScript & React Basics for Backend Developers

**Question:** You can code, but JavaScript and React aren't your daily language. Reading a React pull request, you keep hitting small things — `useMemo`, `async`/`await`, `===`, `Promise.all().flat()`, `styled.div`, "controlled vs uncontrolled input". What do they actually mean, in plain English?

**Answer:** Here's a plain-language tour of the JavaScript and React basics that trip up people coming from other languages. Each one is short, with a tiny example. No prior JS/React expertise assumed — this is the stuff I asked about while reading real code.

## 1. `styled.div` — CSS baked into a component

```js
export const Panel = styled.div`
  display: block;
  width: 100%;
  padding: 8px 0;
`;
```

This is the **styled-components** library. `styled.div\`...\`` creates a real React component that renders a `<div>` with that CSS already attached. So instead of `<div className="panel">` you write `<Panel>...</Panel>`. It holds no logic — it's a styled box. (The backtick string is a *tagged template literal*; you don't need to understand the tag to use it.)

## 2. `useMemo` — remember a value so you don't redo the work

Think of it as a **sticky note that caches an answer**.

A React component re-runs its function *constantly* (on every keystroke, click, anything). If you compute something inside it, that runs every time — wasteful. `useMemo` says "only recompute this when its inputs change":

```js
const result = useMemo(() => expensiveThing(a, b), [a, b]);
```

- First render: runs `expensiveThing`, saves the result.
- Later renders: if `a` and `b` are unchanged → returns the **saved** result.
- Only when `a` or `b` change → recomputes.

There's a sneakier reason too. In JavaScript, `[1, 2]` creates a **brand-new array** every time, even if the contents look identical — like two photocopies that are still two separate pieces of paper. React child components often check "did this input change?" by asking "is it the *same* piece of paper?", not "do the contents match?". So handing a child a fresh array every render makes it think the input changed and re-do work (or reset itself). `useMemo` hands back the **same** array until the inputs actually change.

## 3. The dependency array `[a, b]`

That second argument to `useMemo` (and `useEffect`) is the **list of things to watch**:

```js
const value = useMemo(() => build(x, y), [x, y]);
//                                        ^^^^^^ dependencies
```

React compares these between renders. Unchanged → reuse the cached value. Changed → recompute. The rule: **list everything the function reads from outside itself.** Miss one and you get a *stale* value (the #1 hooks bug); add unstable ones and it recomputes constantly.

- `[]` (empty) = "compute once, never again."
- No array at all = "recompute every render" (usually pointless).

## 4. `async` / `await`

Some operations are slow — mainly talking to a server. JavaScript can't freeze and wait, or the whole page would lock up. So a slow call returns a **Promise** — a "claim ticket" that says *"I'll give you the result when it's ready."*

```js
const resp = await fetchData(url);   // pause HERE until the result arrives
process(resp);                        // runs only after resp is ready
```

`await` pauses **this function** until the Promise resolves, without freezing the page. And `await` is only allowed inside a function marked `async`. One catch: an `async` function itself **always returns a Promise** — so its caller has to `await` it (or use `.then()`) to get the real value.

### Does the next line run immediately? (the part everyone gets wrong)

It depends *which* next line:

- **Inside the same function**, after `await` — **no**, it waits:
  ```js
  const r = await getThing();   // pauses
  use(r);                       // runs only after getThing resolves
  ```
- **In the caller**, after calling an async function — **yes**, it keeps going:
  ```js
  const p = getThing();   // returns a Promise instantly, does NOT wait
  console.log(p);         // runs NOW; p is a pending Promise, not the value
  ```

So `await` makes *its own function* wait; the rest of the program does not. Only code that explicitly `await`s or `.then()`s the Promise sees the final value.

## 5. `Promise.all(...)` and `.flat()`

```js
const results = (await Promise.all(tasks)).flat();
```

- `tasks` is an **array of Promises** (several requests in flight).
- `Promise.all(tasks)` runs them **in parallel** and resolves once **all** finish — total wait = the slowest one, not the sum. It returns an **array of arrays** (each task's result).
- `.flat()` merges that array-of-arrays into one flat list.

So: "fire all requests at once, wait for all of them, then combine their results into a single list." The parentheses matter — `await` must finish *first*, then `.flat()` runs on the real array.

## 6. Array chaining: `.map().filter().map()`

Each method returns a **new array**, so you can chain them like an assembly line:

```js
return (resp?.items || [])
  .map((x) => x.entity)                          // reshape each item
  .filter((x) => x?.name && !HIDDEN.has(x.name)) // keep only some
  .map((x) => ({ id: x.id, name: x.name }));     // trim to what you need
```

- `.map(fn)` — transform every item into something else.
- `.filter(fn)` — keep only items where `fn` returns true.
- `(resp?.items || [])` — use the list, or an empty list if it's missing, so the chain never crashes.
- `(x) => ({ ... })` — the parentheses around `{}` are required; without them JS reads `{` as a function body, not an object.

None of these change the original data — they produce new arrays. That matters in React, where mutating data in place causes bugs.

## 7. Removing duplicates with a `Set`

A `Set` holds **unique** values. Combined with `.filter`, it's the standard "dedupe by a key" trick:

```js
const seen = new Set();
return items.filter((x) => {
  if (!x?.id || seen.has(x.id)) return false;  // drop: no id, or already seen
  seen.add(x.id);                              // remember this id
  return true;                                 // keep the first occurrence
});
```

The filter isn't just testing — it **remembers as it goes**. The first item with each `id` is kept; every later copy is dropped because its `id` is now in `seen`.

## 8. `===` vs `==`

Both check equality; the difference is strictness.

| | Behaviour | Example |
|---|---|---|
| `===` | value **and** type must match, no conversion | `5 === "5"` → `false` |
| `==` | converts types first (surprising) | `5 == "5"` → `true`, `"" == 0` → `true` |

`==` has weird, inconsistent rules (`"" == 0` is true, but `"" == "0"` is false). **Always use `===`** (and `!==`). The one common exception is `x == null`, which conveniently matches both `null` and `undefined`.

## 9. `Number.isFinite(x)`

Asks: *"is this a real, usable number?"* — true only for an actual number that isn't `NaN` or `Infinity`.

```js
const n = Number(input);              // may produce NaN if input is garbage
if (Number.isFinite(n) && n > 0) { …} // only proceed on a genuine positive number
```

`Number(undefined)` is `NaN`, `Number("abc")` is `NaN` — and `NaN` slips past many checks. `Number.isFinite` rejects `NaN`, `Infinity`, and non-numbers (unlike the older global `isFinite`, which converts first and is looser). Same "no surprises" spirit as `===`.

## 10. Ternary + template literals

```js
const fq = ids
  ? [`filter=clientId:in:${ids}`]
  : [];
```

- **Ternary** `cond ? A : B` is a compact if/else that *produces a value*. Here: if `ids` has a value → the array with the filter string; otherwise an empty array.
- **Template literal** (backticks) lets you embed a variable with `${...}`: `` `clientId:in:${ids}` `` becomes `"clientId:in:42,55"` when `ids` is `"42,55"`.

## 11. Controlled vs uncontrolled inputs (a real bug I found)

This one caused an actual defect. React warns:

> *Warning: A component is changing an uncontrolled input to be controlled. This is likely caused by the value changing from undefined to a defined value.*

Plain English: an input (like a checkbox or a tree's selection) can be **uncontrolled** (it manages its own state) or **controlled** (a prop drives its value). If you feed it `undefined` at first and then a real value later, it flips from uncontrolled to controlled — and that flip can **reset the component**, wiping what the user just did.

In my case a tree component received `defaultSelected={someProp?.length ? … : undefined}`. When the selection was written *back* into `someProp`, `someProp.length` went `0 → 2`, so `defaultSelected` flipped `undefined → []` — which re-rendered the tree and **cleared every checkbox about half a second after the click**. The user saw a box tick, then untick itself.

The fix was to stop feeding the live selection back into that prop — freeze it to a stable initial value so it never flips:

```js
// capture the value once, on mount; it never changes afterward
const [initial] = useState(() => someProp || []);
// ...
<Tree defaultSelected={initial} />
```

Lesson: **don't create a feedback loop** where a component's output flows back into a prop that resets it. And an input's `value`/`defaultValue` should not silently go from `undefined` to defined mid-life.

## Takeaways

- `useMemo` and dependency arrays are about **not redoing work** and **keeping the same object reference** so children don't needlessly re-render or reset.
- `await` pauses *its own function*, not the whole program; an `async` function always returns a Promise.
- `Promise.all().flat()` = run in parallel, wait for all, merge results.
- `.map`/`.filter`/`.map` chains and `Set`-based dedup are the everyday way to process lists without mutating them.
- Use `===`, not `==`. Use `Number.isFinite` to reject `NaN`/garbage.
- In React, watch for **uncontrolled → controlled** flips and **feedback loops** — they quietly reset your UI.
