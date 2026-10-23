# Right Side View: Last Node of Each Level

**Question:** Stand to the right of a binary tree. Which nodes can you see,
top to bottom? For `[1,2,3,null,5,null,4]` the answer is `[1,3,4]`. The obvious
idea is "walk the right children all the way down" — why is that wrong, and
what's the clean way?

**Answer:** The visible node at each depth is the *rightmost* one on that
level — which is not always a right child. Do a level-order pass and take the
last node of each level. That's it. Writing the level walk by hand, though, has
two small traps that both produce a silent wrong answer: an inverted loop
condition, and forgetting to drop the empty (`None`) children before the next
level.

## The problem

Looking from the right, you see exactly one node per depth: the one furthest
right. Return those values top to bottom. `O(n)` time (you touch every node
once), `O(w)` space where `w` is the widest level.

## The trap: "just walk right children down"

The tempting one-liner is to start at the root and keep following `.right`.
It works until the right subtree is shorter than the left:

```
        1
       / \
      2   3
     /
    4
```

That's `[1,2,3,4]`. Following right children gives `1 -> 3`, then `3.right` is
empty, so you stop with `[1,3]`. But the real answer is `[1,3,4]` — at depth 2
the right subtree already ended, so the rightmost (and only) node you can see
is `4`, which lives in the **left** subtree. The visible node is "rightmost on
its level," not "a right child."

## The fix: level-order, take the last

Walk the tree one level at a time. The last node of each level is the visible
one. You don't even need a queue — a plain list of "this level's nodes" works:

```python
def right_side_view(root):
    if not root:
        return []

    res = []
    lvl = [root]
    while lvl:
        res.append(lvl[-1].val)        # rightmost node of this level

        children = []
        for item in lvl:
            if item.left:
                children.append(item.left)
            if item.right:
                children.append(item.right)

        lvl = children                 # move down one level

    return res
```

Each pass records `lvl[-1]` (the rightmost, because the list is built
left-to-right), then builds the next level from every node's children. When a
level has no children, `lvl` becomes empty and the loop ends.

Trace `[1,2,3,4]`:

- `lvl = [1]` -> record `1`, children `[2,3]`
- `lvl = [2,3]` -> record `3`, children `[4]` (that's 2's *left* child; 3 has none)
- `lvl = [4]` -> record `4`, children `[]`
- loop ends -> `[1,3,4]` ✓

## Two traps that cost me

I got the shape right the first time but shipped two bugs that each returned a
wrong answer with no error.

**1. The loop condition, inverted.** I wrote `while not lvl:`. `lvl` starts as
`[root]`, which is truthy, so `not lvl` is `False` and the loop body never
runs — the function returns `[]` for every non-empty tree.

```python
while not lvl:     # wrong — exits immediately, returns []
while lvl:         # right — run while this level has nodes
```

`while lvl:` reads as "while there is a level to process." The negation is easy
to type on autopilot and the empty-list result looks almost plausible, which is
what makes it sneaky.

**2. Pushing `None` children into the next level.** My first children loop was
unconditional:

```python
for item in lvl:
    children.append(item.left)
    children.append(item.right)      # appends None for a missing child
```

Now `lvl` can hold `None` entries. Two things break: `lvl[-1].val` can
dereference a `None` when the rightmost slot is empty, and the next pass does
`item.left` on a `None` and raises `AttributeError: 'NoneType' object has no
attribute 'left'`. Guard each append:

```python
for item in lvl:
    if item.left:
        children.append(item.left)
    if item.right:
        children.append(item.right)
```

With the filter, `lvl` only ever holds real nodes, so `lvl[-1]` is genuinely
the rightmost visible node and the next pass is safe.

## Why the list version is fine without a queue

The textbook solution uses a `deque` and pops `width` nodes per level. The
list-of-levels version above does the same work: it just materializes each
level as a list and takes `[-1]`. Both are `O(n)` time and `O(w)` space. The
list form reads a little more directly for this problem because "rightmost of
the level" is literally `lvl[-1]` — no per-level counter needed.

## Takeaways

- The visible node at a depth is the **rightmost on its level**, not a right
  child — so level-order + take-the-last, never "follow `.right` down."
- `while lvl:` means "while there's a level left." `while not lvl:` is the
  inverse and silently returns `[]` — read loop conditions out loud.
- Filter `None` children before they enter the next level, or you get a wrong
  rightmost value and then an `AttributeError` one pass later.
- A bug that returns an empty list (or the wrong list) with no exception is the
  dangerous kind — run it against a case where the answer dips into the *left*
  subtree, like `[1,2,3,4] -> [1,3,4]`.
