# Merge Intervals: Sort Then Sweep

**Question:** Given a list of intervals like `[[1,3],[2,6],[8,10],[15,18]]`,
merge every pair that overlaps so the result is `[[1,6],[8,10],[15,18]]`.
My first working version did it with a "keep re-checking the pile" loop and
it passed all the tests — so why is sorting first the answer everyone reaches
for?

**Answer:** Once the intervals are sorted by start, overlap can only happen
between an interval and the one you most recently kept. That turns an
all-pairs problem into a single left-to-right pass: walk the sorted list, and
for each interval either stretch the last kept interval or start a new one.
It's `O(n log n)` for the sort plus `O(n)` for the sweep, and the whole merge
loop is six lines.

## The problem

Two intervals overlap when they share any point. Touching counts here too:
`[1,4]` and `[4,5]` merge into `[1,5]`. The output is the smallest set of
disjoint intervals covering the same ground.

## The overlap test

Two closed intervals `a` and `b` overlap exactly when:

```python
def is_overlap(a, b):
    return a[0] <= b[1] and b[0] <= a[1]
```

The easy way to trust this is to flip it. They *don't* overlap only if one
sits entirely left of the other: `a[1] < b[0]` **or** `b[1] < a[0]`. Negate
that (De Morgan) and you get `a[1] >= b[0] and b[0] <= ... ` — i.e. `b[0] <=
a[1] and a[0] <= b[1]`, which is the line above. Using `<=` (not `<`) is what
makes touching intervals count as overlapping.

## My first version: correct but quadratic

Before sorting occurred to me, I wrote a worklist loop: pop an interval, scan
everything I'd already kept, and if it overlaps one, remove that one, merge
them, and throw the merged interval back on the pile to be re-checked.

```python
def merge(intervals):
    if not intervals:
        return []
    result = []
    while intervals:
        item = intervals.pop()
        placed = True
        for target in result:
            if is_overlap(item, target):
                result.remove(target)
                intervals.append(merge_items(item, target))
                placed = False
                break
        if placed:
            result.append(item)
    return sorted(result)
```

This is correct — it passes every test, including the unsorted input. But
each popped interval scans the whole `result` list, and a merge pushes work
back onto the queue, so it's `O(n^2)` in the worst case. It also mutates the
caller's input (`intervals.pop()`), which is a rude surprise if the caller
still wants their list. It works; it's just doing far more than it needs to.

## The canonical version: sort, then sweep

```python
def merge(intervals):
    if not intervals:
        return []
    s = sorted(intervals)              # sort by start
    out = [s[0][:]]
    for a, b in s[1:]:
        if a <= out[-1][1]:            # overlaps/touches current
            out[-1][1] = max(out[-1][1], b)
        else:
            out.append([a, b])
    return out
```

Five moving parts:

- `s = sorted(intervals)` orders them by start (Python compares lists
  element by element, so `[1,2]` sorts before `[1,3]`).
- `out = [s[0][:]]` seeds the output with a **copy** of the first interval.
- `for a, b in s[1:]` walks the rest, unpacking each `[start, end]` into
  `a` and `b`.
- `if a <= out[-1][1]` asks "does this start at or before the end of the last
  one I kept?"
- If yes, extend: `out[-1][1] = max(out[-1][1], b)`. If no, it's a clean gap,
  so append `[a, b]` as a new interval.

## Why sorting collapses the problem

In the worklist version a new interval could overlap *anything* already kept,
so I had to scan them all. After sorting by start, that can't happen. Every
later interval starts at or after the current one, so the only interval it
could possibly touch is the last one I kept — everything before that ended
even earlier. I only ever have to look at `out[-1]`. That single guarantee is
the entire reason the one-pass sweep is correct, and it's why `out[-1][1]`
(the running end) is the only state the loop carries.

Note I compare the new start against the running **end**, not the new end,
and I extend with `max` — a fully contained interval like `[2,3]` inside
`[1,4]` must not shrink the end back to 3.

## Two small gotchas

**`s[0][:]` is a copy, on purpose.** `[:]` makes a shallow copy of the first
interval. Without it, `out[-1][1] = ...` would write straight into the
caller's original sublist. The slice gives the output its own object to
mutate so the input is left untouched — the exact bug the worklist version
has.

**`sorted` doesn't mutate.** `sorted(intervals)` returns a new list, unlike
`intervals.sort()`. So this version reads its input and hands back a fresh
result, with no side effects. A good default.

## Takeaways

- Sorting by start is the unlock: overlap can then only be with the last kept
  interval, so an all-pairs check becomes one linear sweep.
- Compare the next **start** against the running **end**; extend with `max` so
  a contained interval can't shrink the result.
- `<=` (not `<`) in the overlap/extend test is what merges touching intervals.
- Seed the output with a **copy** (`s[0][:]`) and prefer `sorted` over
  `.sort()` so you never mutate the caller's data.
- A version that passes every test can still be the wrong one — mine was
  `O(n^2)` and mutated its input. Correct is the floor, not the ceiling.
