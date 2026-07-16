# Rendering a custom HTML tile from report data with a `:=` expression

**Question:** My reporting tool lets a dashboard tile return raw HTML from a small JavaScript expression. I have a dataset of rows and I want to draw one styled line per row (an icon, a bold title, and a "3 tenants, 10 clients, 55 jobs" summary). How does a tile like that actually work, and what are the traps?

**Answer:** The tile is just a tiny function: take the rows the platform hands you, tidy them into a real array, sort them, then build up one big HTML string and return it. The platform paints whatever string you return. The traps are small but real — the rows often arrive as an *index-keyed object* rather than an array, you need an empty-state, and you need a singular/plural helper so you don't print "1 jobs".

## The `:=` marker

Many report platforms let a tile's content be either **static HTML** or a **live expression**. A leading `:=` means "don't treat this as fixed text — run it as JavaScript and use whatever it returns as the HTML." Inside that expression you get the tile's data (here called `rows`). That's the only reason the tile can react to filters and time ranges: it recomputes every time the data changes.

## The one gotcha that bites everyone: `rows` is not an array

You'd expect `rows` to be a normal array you can `.sort()` or loop with `.length`. It usually isn't — it's an **index-keyed object** that looks like this:

```js
rows = { "0": {...}, "1": {...}, "2": {...} };
```

`rows.length` is `undefined` and `rows.sort` doesn't exist. So the first job is to copy it into a real array:

```js
var rs = [];
if (rows) {
  for (var k in rows) {
    if (Object.prototype.hasOwnProperty.call(rows, k)) { rs.push(rows[k]); }
  }
}
```

`for...in` walks the keys (`"0"`, `"1"`, …); `hasOwnProperty` makes sure we only take the object's own entries and skip anything inherited. Now `rs` is a normal array.

## A runnable example

This is the whole idea in a form you can paste into a file and run with `node tile.js`:

```js
// The platform would pass this in. Note: an index-keyed object, not an array.
var rows = {
  "0": { Vendor: "AWS",   Tenants: 3, Clients: 10, Jobs: 55 },
  "1": { Vendor: "Azure", Tenants: 1, Clients: 2,  Jobs: 8  }
};

function render(rows) {
  // 1. object -> real array
  var rs = [];
  if (rows) {
    for (var k in rows) {
      if (Object.prototype.hasOwnProperty.call(rows, k)) { rs.push(rows[k]); }
    }
  }

  // 2. busiest first (|| 0 guards against a missing number -> no NaN)
  rs.sort(function (a, b) { return (b.Jobs || 0) - (a.Jobs || 0); });

  // 3. singular/plural helper: n(1,'job','jobs') -> "1 job"
  function n(v, one, many) { v = v || 0; return v + ' ' + (v === 1 ? one : many); }

  // 4. empty state
  if (!rs.length) { return '<div>No data in this time range</div>'; }

  // 5. build one line per row
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

Output:

```html
<div><div><b>AWS</b> - 3 tenants, 10 clients, 55 jobs</div><div><b>Azure</b> - 1 tenant, 2 clients, 8 jobs</div></div>
```

AWS lands first because 55 > 8, and Azure reads "1 tenant" (singular) because the helper checks `v === 1`.

## The parts, one at a time

- **Sort, busiest first.** A comparator that returns `b - a` sorts high to low. Writing `(b.Jobs || 0)` treats a missing value as `0` so the subtraction never becomes `NaN` and scrambles the order.
- **Singular vs plural.** `n(v, one, many)` picks the right word. Without it you get "1 clients", which looks broken.
- **Empty state.** `if (!rs.length) return ...` shows a friendly message instead of an empty box. Always handle the no-data case — dashboards are viewed with filters that legitimately return nothing.
- **Build a string, then return it.** `h` starts as an opening `<div>`, each loop `+=`'s one more row, and you close it at the end. The platform injects that string as the tile's markup.

In the real tile each row also carries an inline SVG icon and fl: `flex:0 0 auto` on the icon so it never shrinks, and `flex:1 1 auto; min-width:0` on the text block so it takes the rest of the width and can truncate instead of overflowing.

## Why the stored version looks unreadable

If you ever open the saved tile you'll see `&lt;`, `&gt;`, and `\"` everywhere. That's because the HTML string is stored *inside another document* (XML/JSON) that reserves `<`, `>`, and `"`. They get written in their escaped spelling so the outer file stays valid, and they turn back into real `<`, `>`, `"` at runtime. It reads as noise; unescape it before trying to understand it.

## Takeaways

- `:=` means "run this as code and render what it returns" — that's what makes the tile data-driven.
- Treat the incoming `rows` as an **index-keyed object**: copy it into an array before you `sort`/loop. `rows.length` and `.slice()` will not work.
- Guard numbers with `|| 0`, add a singular/plural helper, and always render an empty state.
- Build HTML by concatenating a string and returning it; keep an icon from shrinking with `flex:0 0 auto` and let text truncate with `min-width:0`.
- Escaped `&lt;`/`\"` in the stored file is just encoding — unescape first, then read.
