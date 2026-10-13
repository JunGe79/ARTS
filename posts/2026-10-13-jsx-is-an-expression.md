# JSX Is an Expression That Compiles to a createElement Call

**Question:** JSX looks like HTML sitting inside my JavaScript. What is it really? Is it a special kind of function, or HTML, or something else?

**Answer:** JSX is neither HTML nor a function. It is a plain **JavaScript expression**. A build tool rewrites it into a normal function call (`React.createElement(...)`, or `_jsx(...)` on modern React) that returns a React element object. Because it is an expression, you can use it anywhere a value is allowed — not only inside a function.

## What "expression" means

An expression is anything that evaluates to a value: `2 + 2`, `"hello"`, `user.name`. You can assign it, return it, put it in an array, or pass it as an argument.

JSX is exactly that. This line:

```jsx
<h1 className="title">Hello</h1>
```

is not markup the browser reads. It compiles to:

```js
React.createElement("h1", { className: "title" }, "Hello")
```

That call returns a value — a React element, which is just a plain object describing the UI:

```js
{ type: "h1", props: { className: "title", children: "Hello" } }
```

So "JSX" is sugar. Before it runs, it is already a function call that produces an object.

## Non-function sample (JSX used directly as a value)

You do **not** need a function to use JSX. Here it is used as a bare expression — assigned to variables and stored in an array:

```jsx
// JSX assigned to a variable
const heading = <h1>Dashboard</h1>;

// JSX inside an array
const items = [
  <li key="a">Apple</li>,
  <li key="b">Banana</li>,
];

// JSX chosen by a ternary expression
const badge = isNew ? <span>New</span> : null;
```

None of this is inside a component. Each JSX snippet is just a value. The first one compiles to:

```js
const heading = React.createElement("h1", null, "Dashboard");
```

`heading` now holds an element object, the same way `const n = 2 + 2` holds a number.

## Function sample (a component returns JSX)

The thing that *is* a function is a **component** — a function that returns JSX:

```jsx
function Greeting({ name }) {
  return <h1>Hello, {name}</h1>;
}
```

Compiled:

```js
function Greeting({ name }) {
  return React.createElement("h1", null, "Hello, ", name);
}
```

The function doesn't *contain magic* — it just returns the expression that JSX compiled into. The return value is an element object, which React later renders into real DOM.

## Side by side

| | Non-function sample | Function sample |
|---|---|---|
| Shape | `const x = <h1>Hi</h1>;` | `function C() { return <h1>Hi</h1>; }` |
| What JSX is | an expression used as a value | an expression returned by a function |
| Compiles to | `const x = React.createElement("h1", null, "Hi")` | `return React.createElement("h1", null, "Hi")` |
| Result | an element object | an element object |

Same compilation either way. The only difference is *where* you put the expression.

## A note on modern React

Since React 17, the default build uses the automatic JSX runtime, so the compiled call looks slightly different:

```js
import { jsx as _jsx } from "react/jsx-runtime";
const heading = _jsx("h1", { children: "Dashboard" });
```

Same idea — JSX becomes a function call that builds an element. This is also why newer code doesn't need `import React` just to write JSX.

## Takeaways

- JSX is a **JavaScript expression**, not HTML and not a function.
- It **compiles to a function call** — `React.createElement(...)` (classic) or `_jsx(...)` (modern) — that returns a React element object.
- Because it's an expression, you can use it **outside any function**: assign it, array it, ternary it.
- A **component** is a function; it simply **returns** that JSX expression.
- The browser never sees JSX — a build tool turns it into plain JS first.
