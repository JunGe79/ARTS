# How to refresh one dashboard tile without reloading the page

**Question:** I change a filter, and one tile on my dashboard does not update. Only a full page refresh fixes it. How do I refresh just that tile, without reloading the page?

**Answer:** Run a small timer. Every second it reads what is already on the screen and repaints that tile if anything changed. You need this because a tile only re-runs when its **own** number changes - so anything it draws from another tile stays stale until a reload.

## First: what `:=` means

A tile's content is either **fixed text** or a **`:=` snippet**. The leading `:=` tells the tool: *"this is not fixed text - run it as a program and show whatever it hands back."* Inside, you get the tile's data in a variable called `rows`.

## The whole tile

Here it is. Names are made generic, but the shape is real. Do not worry about it yet - we take it apart below.

```js
:=
var value = 0;
if (rows && rows[0] && rows[0]['Failed']) { value = rows[0]['Failed']; }

(function () {
  if (window.__iconInit) { return; }      // only ever set this up once
  window.__iconInit = true;

  var W = '<svg width="18" height="18" viewBox="0 0 50 50"><polygon fill="#FFC219" points="25 0 0 50 50 50"/></svg>';  // amber
  var K = '<svg width="18" height="18" viewBox="0 0 24 24"><circle cx="12" cy="12" r="10" fill="#4caf50"/></svg>';      // green

  function state() {
    var idle = 0, failed = 0, ns = document.querySelectorAll('.dash-status-number');
    for (var i = 0; i < ns.length; i++) {
      var l = ns[i].parentElement.querySelector('.dash-event-status');
      if (!l) { continue; }
      var t = l.textContent.trim().toUpperCase();
      var v = parseInt((ns[i].textContent || '').trim(), 10) || 0;
      if (t === 'IDLE') { idle = v; } else if (t === 'FAILED') { failed = v; }
    }
    return (idle > 0 || failed > 0) ? 'a' : 'g';
  }

  function title() {
    var all = document.querySelectorAll('*'), best = null;
    for (var i = 0; i < all.length; i++) {
      var n = all[i];
      if (n.textContent.trim().toLowerCase() !== 'needs attention') { continue; }
      if (best === null || n.getElementsByTagName('*').length < best.getElementsByTagName('*').length) { best = n; }
    }
    return best;
  }

  function upd() {
    var t = title(); if (!t) { return; }
    var st = state();
    var ex = t.querySelector('.status-icon');
    if (!ex) {
      ex = document.createElement('span');
      ex.className = 'status-icon';
      ex.style.marginRight = '8px';
      t.insertBefore(ex, t.firstChild);
      ex.__st = '';
    }
    if (ex.__st !== st) { ex.innerHTML = (st === 'a') ? W : K; ex.__st = st; }
  }

  setTimeout(upd, 200);
  setInterval(upd, 1000);
})();

return '<div style="display:flex;align-items:flex-start;padding:6px 4px;">'
     + '<div style="margin-right:10px;"><svg width="34" height="34"><rect width="34" height="34" fill="#f44336"/></svg></div>'
     + '<div>'
     +   '<div class="dash-status-number" style="font-size:22px;font-weight:600;">' + value + '</div>'
     +   '<div><span class="dash-event-status">FAILED</span></div>'
     + '</div></div>';
```

It does **two jobs at once**, which is the thing to notice:

1. It draws **its own tile** - a red icon, a big number, and the label `FAILED` (that is the `return` at the bottom).
2. It also keeps an icon next to a **different** heading ("Needs attention") in sync - amber if anything is wrong, green if not.

Job 2 is the interesting one.

## The problem this solves

The obvious way to write job 2 is: read the two numbers, decide the colour, paint the icon. Once. That is what I did first, and it was subtly broken.

A tile's snippet is re-run by the tool **only when that tile's own value changes**. My tile's own value is the "Failed" count, which is nearly always `0`. So:

- Page loads with 9 idle -> snippet runs -> icon painted **amber**. Correct.
- I change a filter; idle drops to 0, failed stays 0.
- My tile's own value did not change (still 0), so **the snippet never re-runs**.
- The icon stays **amber**. Wrong - and it only corrects on a full page reload, because a reload re-runs everything.

So the icon cannot be a one-shot calculation inside a tile that does not reliably re-run.

## The fix, in one idea

**Do not compute the icon from the data. Read the numbers that are already painted on the screen, on a timer.**

That decouples the icon from whether any particular tile re-ran. When a filter changes, the numbers on screen change, and the watcher notices within a second and repaints.

## Line by line

```js
var value = 0;
if (rows && rows[0] && rows[0]['Failed']) { value = rows[0]['Failed']; }
```
Get this tile's own number. `rows[0]` is the first row of data; `['Failed']` is the column. The `&&` chain means *"only if all of these exist"* - it avoids an error when there is no data. If anything is missing, `value` stays `0`.

```js
(function () { ... })();
```
This is an **immediately-run function**. Wrapping code in `(function(){ ... })()` runs it right now, once, and keeps its variables private. Nothing else on the page can trip over names like `W` or `state`.

```js
  if (window.__iconInit) { return; }
  window.__iconInit = true;
```
The **run-once guard**. `window` is the page's global scratchpad. The first time through, `window.__iconInit` is undefined, so we skip the `return`, set the flag, and carry on. Every later run sees the flag and **exits immediately**. Without this, every re-run would start *another* timer and you would end up with a pile of them all fighting.

```js
  var W = '<svg ...amber triangle.../>';
  var K = '<svg ...green circle.../>';
```
The two icons, stored as plain text. `W` = warning, `K` = OK.

```js
  function state() {
    var idle = 0, failed = 0, ns = document.querySelectorAll('.dash-status-number');
```
`document.querySelectorAll('.dash-status-number')` = *"find every element on the page with this CSS class"* - these are the big numbers on the dashboard. `ns` is the collection of them.

```js
    for (var i = 0; i < ns.length; i++) {
      var l = ns[i].parentElement.querySelector('.dash-event-status');
      if (!l) { continue; }
```
For each number, look **next to it** for its label. `parentElement` is the box holding both. `continue` means *"skip this one and move to the next"* - a number with no label is not one of ours.

```js
      var t = l.textContent.trim().toUpperCase();
      var v = parseInt((ns[i].textContent || '').trim(), 10) || 0;
```
`textContent` = the visible text. `.trim()` strips spaces. `.toUpperCase()` makes the label comparison case-proof. `parseInt(x, 10)` turns the text `"9"` into the number `9` (the `10` just means "normal base-10 counting"). The `|| 0` at the end means *"if that failed, use 0"*.

```js
      if (t === 'IDLE') { idle = v; } else if (t === 'FAILED') { failed = v; }
    }
    return (idle > 0 || failed > 0) ? 'a' : 'g';
  }
```
Match the label, keep the number. Then: *if either is above zero, return `'a'` (amber), otherwise `'g'` (green).* `||` means **or**.

```js
  function title() {
    var all = document.querySelectorAll('*'), best = null;
    for (var i = 0; i < all.length; i++) {
      var n = all[i];
      if (n.textContent.trim().toLowerCase() !== 'needs attention') { continue; }
      if (best === null || n.getElementsByTagName('*').length < best.getElementsByTagName('*').length) { best = n; }
    }
    return best;
  }
```
Find the heading to attach the icon to. `'*'` means **every element on the page**. We keep only those whose text is exactly "needs attention" - but a heading's whole ancestry technically "contains" that text too, so several elements match. `getElementsByTagName('*').length` counts how many elements are **inside** each candidate, and we keep the one with the **fewest**. That is the innermost, most specific element - the heading itself, not the panel wrapped around it.

```js
  function upd() {
    var t = title(); if (!t) { return; }
    var st = state();
```
The update step: find the heading, work out the colour. If the heading is not on screen (different page), quietly do nothing.

```js
    var ex = t.querySelector('.status-icon');
    if (!ex) {
      ex = document.createElement('span');
      ex.className = 'status-icon';
      ex.style.marginRight = '8px';
      t.insertBefore(ex, t.firstChild);
      ex.__st = '';
    }
```
Is our icon holder already there? If not, **build one**: `createElement('span')` makes a new empty element, we tag it with a class and a little right margin, and `insertBefore(ex, t.firstChild)` puts it **before the heading's first child** - i.e. in front of the words. `ex.__st = ''` starts a note-to-self about the current colour.

```js
    if (ex.__st !== st) { ex.innerHTML = (st === 'a') ? W : K; ex.__st = st; }
  }
```
**Only touch the page if the colour actually changed.** `ex.__st` remembers what we drew last time. Without this check we would rewrite the icon every single second for no reason.

```js
  setTimeout(upd, 200);
  setInterval(upd, 1000);
```
`setTimeout(upd, 200)` runs `upd` **once**, 200ms from now - a quick first paint. `setInterval(upd, 1000)` runs it **again and again, every second** - that is what keeps it in sync forever after.

```js
return '<div ...>' + value + '<span class="dash-event-status">FAILED</span>...</div>';
```
Finally, job 1: hand back this tile's own HTML - the red icon, the big `value`, and the label. The tool paints whatever string you return.

## Three traps worth knowing

**1. The icon must not add any text.** `title()` finds the heading by matching its text *exactly*. The icons are SVG shapes, which contribute no text, so the heading's text stays "needs attention" and keeps matching on the next tick. If you inject a text character instead (say `▲`), the heading's text becomes "▲Needs attention", `title()` stops matching, and the icon freezes after the first paint.

**2. Guard the timer, or they stack.** Without `window.__iconInit`, each re-run adds another `setInterval`. Ten re-runs = ten timers doing the same work.

**3. Only write when something changed.** Repainting identical HTML every second is wasteful and can make the browser fight you (flicker, lost focus). The `ex.__st !== st` check keeps writes rare.

## Try it yourself

Save this as `demo.html` and open it in a browser. Click the buttons and watch the icon next to the heading flip **without any page reload** - that is the whole point.

```html
<!doctype html>
<meta charset="utf-8">
<h3 id="heading">Needs attention</h3>

<div><span class="dash-status-number">0</span> <span class="dash-event-status">IDLE</span></div>
<div><span class="dash-status-number">0</span> <span class="dash-event-status">FAILED</span></div>

<button onclick="setNum(0, 9)">IDLE = 9</button>
<button onclick="setNum(0, 0)">IDLE = 0</button>
<button onclick="setNum(1, 3)">FAILED = 3</button>
<button onclick="setNum(1, 0)">FAILED = 0</button>

<script>
function setNum(i, v) {
  document.querySelectorAll('.dash-status-number')[i].textContent = v;
}

(function () {
  if (window.__iconInit) { return; }
  window.__iconInit = true;

  var W = '<svg width="18" height="18" viewBox="0 0 50 50"><polygon fill="#FFC219" points="25 0 0 50 50 50"/></svg>';
  var K = '<svg width="18" height="18" viewBox="0 0 24 24"><circle cx="12" cy="12" r="10" fill="#4caf50"/></svg>';

  function state() {
    var idle = 0, failed = 0, ns = document.querySelectorAll('.dash-status-number');
    for (var i = 0; i < ns.length; i++) {
      var l = ns[i].parentElement.querySelector('.dash-event-status');
      if (!l) { continue; }
      var t = l.textContent.trim().toUpperCase();
      var v = parseInt((ns[i].textContent || '').trim(), 10) || 0;
      if (t === 'IDLE') { idle = v; } else if (t === 'FAILED') { failed = v; }
    }
    return (idle > 0 || failed > 0) ? 'a' : 'g';
  }

  function title() {
    var all = document.querySelectorAll('*'), best = null;
    for (var i = 0; i < all.length; i++) {
      var n = all[i];
      if (n.textContent.trim().toLowerCase() !== 'needs attention') { continue; }
      if (best === null || n.getElementsByTagName('*').length < best.getElementsByTagName('*').length) { best = n; }
    }
    return best;
  }

  function upd() {
    var t = title(); if (!t) { return; }
    var st = state();
    var ex = t.querySelector('.status-icon');
    if (!ex) {
      ex = document.createElement('span');
      ex.className = 'status-icon';
      ex.style.marginRight = '8px';
      t.insertBefore(ex, t.firstChild);
      ex.__st = '';
    }
    if (ex.__st !== st) { ex.innerHTML = (st === 'a') ? W : K; ex.__st = st; }
  }

  setTimeout(upd, 200);
  setInterval(upd, 1000);
})();
</script>
```

## What you should see

| You click | Numbers on screen | Icon becomes |
|---|---|---|
| *(nothing - page load)* | IDLE 0, FAILED 0 | green, within a second |
| `IDLE = 9` | IDLE 9, FAILED 0 | **amber** |
| `IDLE = 0` | IDLE 0, FAILED 0 | back to **green** |
| `FAILED = 3` | IDLE 0, FAILED 3 | **amber** (either one counts) |

Note the second row is exactly the case that was broken before: only *idle* moved, yet the icon still updates - because the watcher reads the screen, not one tile's data.

## Is polling every second not wasteful?

It is one pass over the page's elements per second, comparing a couple of numbers, and it writes to the page only when the colour genuinely changes. On a dashboard that is nothing. It is also far simpler and more robust than trying to guess exactly when the tool decides to re-run each tile - which was the thing that broke in the first place.

If you want to be tidier, the same logic works inside a `MutationObserver` (which fires only when the numbers actually change), but the timer version is easier to read and hard to get wrong.

## Takeaways

- A tile's `:=` snippet re-runs when **that tile's own value** changes - not whenever the page changes. Anything you derive from *another* tile will go stale.
- The escape hatch: **read the rendered screen on a timer** instead of computing once from the data. It does not care which tile re-ran.
- Always guard one-time setup with a flag like `window.__iconInit`, or your timers pile up.
- Cache the last state (`ex.__st`) and only write to the page when it actually changes.
- If you inject an icon into an element you find **by its text**, use an SVG so it adds no text - otherwise your own icon breaks the search next time round.
- `x || 0`, `parseInt(x, 10) || 0`, and `a && b && c` are all just "use a sensible default instead of crashing".
