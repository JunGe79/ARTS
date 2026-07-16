# Reading a report tile's JavaScript, line by line

**Question:** My reporting tool lets a dashboard tile return raw HTML from a small JavaScript snippet. I am not a JavaScript person. Someone handed me a 20-line tile that draws "AWS - 3 tenants, 10 clients, 55 jobs" and I cannot read it. What does each line actually do?

**Answer:** The whole thing is a tiny recipe: **take the data, turn it into a list, sort it, then glue together a long piece of HTML text and hand it back.** The tool paints whatever text you hand back. Below is every line in plain words, plus samples you can run.

## First: what `:=` means

A tile's content can be one of two things:

- **Fixed text** - always shows the same thing.
- **A `:=` snippet** - the leading `:=` tells the tool: *"this is not fixed text. Run it as a program and show whatever it gives back."*

That is the only reason the tile can react to filters and time ranges. It re-runs and rebuilds its HTML each time the data changes. Inside the snippet you are handed the tile's data in a variable called `rows`.

## The whole tile

Here is the complete thing we are about to read. It is 20-odd lines. Do not worry about it yet - the rest of the post takes it apart one line at a time.

```js
:=
var rs = [];
if (rows) {
  for (var k in rows) {
    if (Object.prototype.hasOwnProperty.call(rows, k)) { rs.push(rows[k]); }
  }
}
rs.sort(function (a, b) { return (b.Jobs || 0) - (a.Jobs || 0); });

function n(v, one, many) { v = v || 0; return v + ' ' + (v === 1 ? one : many); }

if (!rs.length) {
  return '<div style="padding:10px;color:#888;">No Auto Scale jobs in this time range</div>';
}

var h = '<div style="padding:4px 2px;">';
for (var i = 0; i < rs.length; i++) {
  var r = rs[i];
  h += '<div style="display:flex;align-items:center;gap:10px;padding:7px 4px;border-bottom:1px solid #eee;">'
     + '<svg style="fill:#0078d4;width:20px;height:20px;flex:0 0 auto;" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24"><path d="M19.35 10.04A7.49 7.49 0 0 0 12 4C9.11 4 6.6 5.64 5.35 8.04A5.994 5.994 0 0 0 0 14a6 6 0 0 0 6 6h13a5 5 0 0 0 5-5c0-2.64-2.05-4.78-4.65-4.96z"/></svg>'
     + '<div style="flex:1 1 auto;min-width:0;">'
     + '<div style="font-weight:600;font-size:13px;">' + r.Vendor + '</div>'
     + '<div style="font-size:12px;color:#666;">'
     + n(r.Tenants, 'tenant', 'tenants') + ', '
     + n(r.Clients, 'client', 'clients') + ', '
     + n(r.Jobs, 'job', 'jobs')
     + '</div></div></div>';
}
return h + '</div>';
```

What it draws, one row per vendor:

```
[cloud icon]  AWS
              3 tenants, 10 clients, 55 jobs
[cloud icon]  Azure
              1 tenant, 2 clients, 8 jobs
```

Read it as five moves: **make a list -> sort it -> define a grammar helper -> bail out if empty -> glue one HTML row per item and hand it back.** Everything else is styling noise.

## The big gotcha: the data is not a list

You would expect `rows` to be a normal list. It is not. It arrives like a **numbered filing cabinet**:

```js
rows = { "0": {...}, "1": {...}, "2": {...} };
```

It has drawers labelled `"0"`, `"1"`, `"2"` - but it is not a list. That means:

- `rows.length` gives **nothing** (undefined)
- `rows.sort(...)` **does not exist** and will error

So the first job is always: open each drawer and copy the contents into a real list.

## Line by line

```js
var rs = [];
```
Make an empty **list** called `rs`. `[]` means "empty list".

```js
if (rows) {
```
"If we actually got data." A safety check, in case `rows` is missing.

```js
  for (var k in rows) {
```
Go through each **key** in the cabinet. `k` holds one key at a time: first `"0"`, then `"1"`, and so on.

```js
    if (Object.prototype.hasOwnProperty.call(rows, k)) { rs.push(rows[k]); }
```
"Is this drawer really one of `rows`' own?" (skips inherited junk). If yes: `rows[k]` is what is inside that drawer, and `.push(...)` **adds it to the end** of our list.

After this, `rs` is a normal list and we can sort and loop it.

```js
rs.sort(function (a, b) { return (b.Jobs || 0) - (a.Jobs || 0); });
```
**Sort** the list. The little function compares two items, `a` and `b`. Returning `b - a` means **biggest first**. `b.Jobs || 0` reads as *"use `b.Jobs`, but if it is missing use 0"* - without that, a missing number breaks the maths and the order goes random.

```js
function n(v, one, many) { v = v || 0; return v + ' ' + (v === 1 ? one : many); }
```
A small **grammar helper**. It takes a number and two words:

- `v = v || 0` - if the number is missing, treat it as 0.
- `v === 1 ? one : many` - this is a shorthand for "if / else". It reads: *if `v` is exactly 1, use the singular word, otherwise use the plural.*

So `n(1,'job','jobs')` gives `"1 job"`, and `n(5,'job','jobs')` gives `"5 jobs"`. Without it you print "1 jobs", which looks broken.

```js
if (!rs.length) {
  return '<div style="padding:10px;color:#888;">No Auto Scale jobs in this time range</div>';
}
```
`.length` is how many items are in the list. `!` means **not**. So: *"if the list is empty"* - hand back a grey message and **stop right here**. `return` exits immediately; nothing below runs.

```js
var h = '<div style="padding:4px 2px;">';
```
Start a piece of **text** called `h` that will hold our HTML. This is the opening box.

```js
for (var i = 0; i < rs.length; i++) {
```
Repeat **once per item**. `i` counts `0, 1, 2...`; `i++` adds 1 each round; it stops when `i` reaches the size of the list.

```js
  var r = rs[i];
```
`r` is the **current item** - one vendor's data, e.g. `{Vendor:"AWS", Tenants:3, Clients:10, Jobs:55}`.

```js
  h += '<div style="display:flex; ...">'
```
`+=` means *"stick this onto the end of `h`"*. And `+` between two pieces of text **glues them together**. This opens one row.

```js
     + '<svg ...><path d="M19.35 10.04A7.49 ..."/></svg>'
```
The **cloud icon**. That long `d="..."` is just drawing instructions for the shape - there is nothing to read there.

```js
     + '<div style="font-weight:600;...">' + r.Vendor + '</div>'
```
The **vendor name** in bold, e.g. `AWS`.

```js
     + n(r.Tenants, 'tenant', 'tenants') + ', '
     + n(r.Clients, 'client', 'clients') + ', '
     + n(r.Jobs,    'job',    'jobs')
```
Builds the smaller grey line: `"3 tenants, 10 clients, 55 jobs"`.

```js
     + '</div></div></div>';
}
return h + '</div>';
```
**Close** the boxes we opened, end the loop, then close the outer box and **hand the finished HTML back**.

## The style bits, one line each

- `display:flex; align-items:center; gap:10px` - put the icon and text side by side, lined up in the middle, 10px apart.
- `flex:0 0 auto` on the icon - **never shrink** it.
- `flex:1 1 auto; min-width:0` on the text - **take the remaining width**, and allow it to shrink so long text truncates instead of overflowing.
- `border-bottom:1px solid #eee` - a thin divider line under each row.

## Try it yourself

Save this as `tile.js` and run `node tile.js`. It is the same recipe, with the styling stripped so you can see the logic:

```js
// The tool hands this in. Note: a numbered cabinet, not a list.
var rows = {
  "0": { Vendor: "AWS",   Tenants: 3, Clients: 10, Jobs: 55 },
  "1": { Vendor: "Azure", Tenants: 1, Clients: 2,  Jobs: 8  }
};

function render(rows) {
  // 1. cabinet -> real list
  var rs = [];
  if (rows) {
    for (var k in rows) {
      if (Object.prototype.hasOwnProperty.call(rows, k)) { rs.push(rows[k]); }
    }
  }

  // 2. biggest first
  rs.sort(function (a, b) { return (b.Jobs || 0) - (a.Jobs || 0); });

  // 3. grammar helper
  function n(v, one, many) { v = v || 0; return v + ' ' + (v === 1 ? one : many); }

  // 4. nothing to show?
  if (!rs.length) { return '<div>No data in this time range</div>'; }

  // 5. one line per item
  var h = '<div>';
  for (var i = 0; i < rs.length; i++) {
    var r = rs[i];
    h += '<div>'
       + '<b>' + r.Vendor + '</b> - '
       + n(r.Tenants, 'tenant', 'tenants') + ', '
       + n(r.Clients, 'client', 'clients') + ', '
       + n(r.Jobs,    'job',    'jobs')
       + '</div>';
  }
  return h + '</div>';
}

console.log(render(rows));
```

## Three samples

**Sample 1 - normal data.** Using the `rows` above:

```html
<div><div><b>AWS</b> - 3 tenants, 10 clients, 55 jobs</div><div><b>Azure</b> - 1 tenant, 2 clients, 8 jobs</div></div>
```

Two things to notice: **AWS is first** even though it was not first in the data (55 jobs beats 8, and we sorted biggest first). And Azure reads **"1 tenant"**, not "1 tenants" - that is the grammar helper doing its job.

**Sample 2 - a missing number.** Change one row to:

```js
"1": { Vendor: "Azure", Tenants: 1, Clients: 2 }   // Jobs is missing
```

Output:

```html
<div><div><b>AWS</b> - 3 tenants, 10 clients, 55 jobs</div><div><b>Azure</b> - 1 tenant, 2 clients, 0 jobs</div></div>
```

It prints `0 jobs` instead of blowing up or printing "undefined jobs". That is what `v = v || 0` and `(b.Jobs || 0)` are protecting you from.

**Sample 3 - no data at all.** Call `render({})`:

```html
<div>No data in this time range</div>
```

A friendly message instead of an empty box. Always handle this - people view dashboards with filters that legitimately match nothing.

## Why the saved version looks like gibberish

If you open the stored tile you will see things like `&lt;div&gt;` and `\"` everywhere. Nothing clever is going on: the HTML text is stored **inside another file** (XML or JSON) that already uses `<`, `>` and `"` for its own purposes. So those characters get written in a "safe" spelling to keep the outer file valid, and they turn back into real `<`, `>`, `"` when the tile runs.

Do not try to read it in that form. Unescape it first, then it reads like normal code.

## Takeaways

- `:=` means "run this as a program and show what it returns". That is what makes a tile react to filters.
- The data you are handed is a **numbered cabinet, not a list**. Copy it into a list before you sort or loop. `rows.length` and `rows.sort()` will not work.
- `x || 0` means "use x, or 0 if it is missing". Sprinkle it on any number you do maths with.
- `v === 1 ? one : many` is just if/else in shorthand - use it so you never print "1 jobs".
- Always return something for the **empty** case.
- The tile is only building a long piece of text with `+` and `+=`, then handing it back. Nothing more.
- Escaped `&lt;` and `\"` in the saved file is just encoding. Unescape first, then read.
