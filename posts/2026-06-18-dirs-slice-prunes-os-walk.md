# Why `dirs[:]` Prunes an `os.walk`, but `dirs =` Doesn't

**Question:** I want `os.walk` to skip some subdirectories. The recipe
is `dirs[:] = [d for d in dirs if keep(d)]`. Why the `[:]`? Why not
just `dirs = ...`?

**Answer:** `dirs[:] = ...` mutates the list in place; `dirs = ...`
rebinds the local name to a new list and leaves the original alone.
`os.walk` holds a reference to the original list, so only the in-place
mutation reaches it.

## The two operations are not the same

```python
a = [1, 2, 3]
b = a            # b points to the SAME list as a

a[:] = [9, 9]    # in-place mutation; same object, new contents
print(b)         # [9, 9]   <- b sees the change

a = [7, 7]       # rebind a; new object
print(b)         # [9, 9]   <- b still sees the old object
```

## Why `os.walk` cares

`os.walk` yields the `dirs` list to you, then re-reads the SAME list
after your loop body to decide which subdirectories to descend into.

- `dirs = [...]` makes a new list and points your local name at it.
  `os.walk` is still holding the original — nothing changed for it. It
  descends into every original entry, skip list and all.
- `dirs[:] = [...]` clears the original list and refills it. `os.walk`
  re-reads its reference and sees only the kept entries. Pruned
  subtrees are never opened.

## Other handy forms

```python
copy = dirs[:]                # shallow copy (RHS form)
dirs[:] = []                  # empty in place
dirs[:] = sorted(dirs)        # sort + replace, same as dirs.sort()
```

Same rule applies anywhere a function holds a reference to your list
and re-reads it: in-place mutation reaches them, rebinding does not.
