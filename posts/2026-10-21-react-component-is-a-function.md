# A React Component Is Just a Function

**Question:** React tutorials throw around the word "component" as if it
were some special framework construct you have to learn from scratch. But
what *is* a component, really — and how do I actually use one?

**Answer:** A component is a plain JavaScript function. It takes one
argument (an object called `props`) and returns a description of what the
UI should look like (JSX). That's the whole idea. You "use" a component by
letting React call that function for you — you write `<Greeting />` and
React runs `Greeting(props)` and puts the returned markup on screen.

## A component is a function that returns markup

Here is a complete component. Notice there is nothing magic about it — it
is a function:

```jsx
function Greeting() {
  return <h1>Hello</h1>;
}
```

`Greeting` takes no arguments and returns `<h1>Hello</h1>`. The returned
thing looks like HTML but it is JSX — a bit of syntax that compiles down to
a normal JavaScript object describing the element. So the function really
returns a value, the same way any function does. React's job is to take
that returned value and turn it into real DOM.

An arrow function works exactly the same way; the style is a preference,
not a different thing:

```jsx
const Greeting = () => <h1>Hello</h1>;
```

## Props are the function's one argument

A function that always returns the same thing is not very useful. To make a
component configurable, React passes it a single argument: an object
holding everything the parent gave it. By convention it is called `props`.

```jsx
function Greeting(props) {
  return <h1>Hello, {props.name}</h1>;
}
```

The curly braces `{props.name}` drop back into JavaScript inside the JSX, so
whatever `props.name` holds gets printed. Because `props` is just an object,
most people destructure it right in the parameter list:

```jsx
function Greeting({ name }) {
  return <h1>Hello, {name}</h1>;
}
```

Same function, same single argument — `{ name }` just pulls the `name` field
out of the object as it arrives.

## Using a component = calling it (with angle brackets)

Here is the part that trips people up. You do **not** call a component the
way you call a normal function. You never write `Greeting({ name: "Ada" })`
yourself. Instead you write it like a tag:

```jsx
<Greeting name="Ada" />
```

That line means exactly "call `Greeting` with `props = { name: "Ada" }`."
The attributes on the tag become the fields of the `props` object. React
does the calling — that indirection is the entire point of the framework.
It decides *when* to call your function and *when* to call it again, so you
only describe what the UI should look like for a given set of props.

So these two are the same call, written two ways:

| You write | React runs |
|---|---|
| `<Greeting name="Ada" />` | `Greeting({ name: "Ada" })` |
| `<Greeting />` | `Greeting({})` |

## Why `<Greeting />` and not `Greeting()`

If a component is a function, why not just call it directly? Because React
needs to track each usage as a node in its tree so it can re-run the right
functions when data changes, keep each one's state separate, and update the
DOM efficiently. When you write `<Greeting />`, React records "there is a
Greeting here" and manages it. If you call `Greeting()` yourself, you get a
one-time return value and React loses all of that bookkeeping. Write the
tag; let React do the call.

This is also why component names must start with a **capital letter**. JSX
treats a lowercase tag like `<greeting />` as a built-in HTML element (a
literal string `"greeting"`), and a capitalized tag like `<Greeting />` as a
reference to your function. The capital letter is how JSX tells "your
component" apart from "an HTML tag."

## Composing: functions calling functions

Because components are functions that return markup, and you use them by
putting their tags inside other markup, bigger UIs are just functions used
inside other functions' return values:

```jsx
function App() {
  return (
    <div>
      <Greeting name="Ada" />
      <Greeting name="Linus" />
    </div>
  );
}
```

`App` is a function that returns markup containing two `<Greeting>` tags, so
rendering `App` leads React to call `Greeting` twice, with different props.
That is the whole composition model — no special wiring, just functions
returning markup that mentions other functions.

## The twist: a function that React re-runs

A plain function runs once and forgets everything. A React component is
special in one way: React may call it **again and again**, and you often
want it to remember something between calls (a counter, what the user typed,
whether a menu is open). You cannot keep that in an ordinary local variable,
because every call starts fresh. That is what `useState` is for:

```jsx
function Counter() {
  const [count, setCount] = useState(0);
  return (
    <button onClick={() => setCount(count + 1)}>
      Clicked {count} times
    </button>
  );
}
```

`useState(0)` says "give me a value that survives re-runs, starting at 0,"
and hands back the current value plus a setter. When you call `setCount`,
React re-runs `Counter` — the function executes top to bottom again, but
`useState` now hands back the new value instead of 0, so the button shows
the updated count. The component is still a function; React just calls it
repeatedly and feeds it its own remembered state through `useState`.

Functions like `useState` are called **hooks**, and they only work inside a
component (or another hook), called unconditionally at the top level. That
rule exists precisely because React matches each hook to its stored value by
call order across re-runs — another consequence of a component being a
function React runs many times.

## Takeaways

- A component is a JavaScript function: one argument (`props`), returns JSX.
- You "use" it by writing a tag (`<Greeting name="Ada" />`); React, not you,
  calls the function, and the tag's attributes become `props`.
- Write `<Greeting />`, never `Greeting()` — the tag lets React own the call
  so it can manage state and re-rendering.
- Capitalize component names so JSX treats them as your function, not an
  HTML tag.
- Compose UIs by putting component tags inside other components' returned
  markup — functions used inside functions.
- The one twist over a plain function: React re-runs it, and `useState` (a
  hook) is how the function remembers things between runs.
