# JavaScript & React Basics for Backend Developers

**Question:** You can code, but JS/React isn't your daily language. What do `useMemo`, `async`/`await`, `===`, `Promise.all().flat()`, `styled.div`, and "controlled vs uncontrolled" actually mean?

**Answer:** A quick plain-English cheat sheet — the basics that trip up people coming from other languages.

## styled.div
`styled-components`. `styled.div\`css\`` makes a React component that renders a `<div>` with that CSS attached. No logic, just style.
```js
import styled from "styled-components";

const Panel = styled.div`
  display: block;
  width: 100%;
  padding: 8px 0;
`;

function Card() {
  return <Panel>Hello</Panel>;   // renders a <div> with that CSS + "Hello"
}
```
Use `<Panel>` instead of `<div className="panel">`.

## useMemo
Caches a computed value; recomputes only when its dependency list changes.
```js
const v = useMemo(() => build(a, b), [a, b]);
```
Why it also matters (not just speed): every render, `[1, 2]` makes a **brand-new array** — same contents, but a different object. React often checks "did this change?" by asking "is it the same object?", not "are the contents equal?" So a fresh array each render looks "new" and can make a child component re-render or reset. `useMemo` hands back the **same** array until the inputs change.

The `[a, b]` list should include every value the function uses. Forget one and you get an old (stale) result.

## async / await
A slow call returns a **Promise** (an IOU). `await` pauses **this function** until it resolves (page doesn't freeze); only allowed inside `async`. An `async` function always returns a Promise.

**Does the next line wait?** Inside the same function after `await` — yes. In the *caller* — no; calling an async function returns a Promise immediately and moves on. Only whoever `await`s/`​.then()`s it sees the value.

## Promise.all().flat()
```js
const all = (await Promise.all(tasks)).flat();
```
Run promises in parallel, wait for all (returns array-of-arrays), `.flat()` merges into one list.

## .map / .filter / .map
Each returns a new array, so you chain them (no mutation):
```js
(list || []).map(x => x.entity).filter(x => keep(x)).map(x => ({id: x.id}));
```
`(x) => ({...})` needs the parens or JS reads `{` as a code block.

## Dedupe with a Set
```js
const seen = new Set();
items.filter(x => x.id && !seen.has(x.id) && seen.add(x.id));
```
Keeps the first of each `id`, drops repeats.

## === vs ==
Use `===` — checks value **and** type, no conversion. `==` converts first and is full of surprises (`"" == 0` is `true`). Exception: `x == null` matches both `null` and `undefined`.

## Number.isFinite(x)
"Is this a real number?" — rejects `NaN`, `Infinity`, and non-numbers. Handy after `Number(input)`, which yields `NaN` for garbage.

## Ternary + template literals
```js
const fq = ids ? [`clientId:in:${ids}`] : [];
```
`cond ? A : B` = inline if/else that returns a value. Backticks + `${x}` embed a variable in a string.

## Controlled vs uncontrolled (a real bug)
React warns *"changing an uncontrolled input to be controlled … value changing from undefined to a defined value."* If an input's value goes `undefined → defined` mid-life, it can **reset the component**. In my case a tree's `defaultSelected` flipped `undefined → []` when the selection was written back into its own prop — so checkboxes ticked, then un-ticked ~0.5s later. Fix: freeze the initial value so it never flips:
```js
const [initial] = useState(() => prop || []);
<Tree defaultSelected={initial} />
```
Lesson: avoid feedback loops where a component's output resets its own input.

## Takeaways
- `useMemo` = don't redo work + keep stable references.
- `await` pauses its own function, not the whole program.
- Prefer `===` and `Number.isFinite`; process lists with map/filter (no mutation).
- Watch for **uncontrolled → controlled** flips and prop feedback loops in React.
