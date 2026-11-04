# Every Game of Life bug lives in the neighbour count

**Question:** Conway's Game of Life (LeetCode medium) looks easy — apply four
rules to each cell at once. So why do so many first attempts crash or return the
wrong board?

**Answer:** The rules are not where the bugs are. Almost every mistake is in the
small block that counts a cell's 8 neighbours — a missing diagonal, an off-by-one
on the grid edge, or an `and`/`or` mix-up on the center cell. Write that block as
one 3x3 loop with two fixed habits and the problem becomes trivial.

## The rules (the easy part)

Each step, every cell looks at its 8 neighbours and all cells update
**simultaneously**:

- live cell, `< 2` live neighbours -> dies (underpopulation)
- live cell, `2 or 3` live neighbours -> survives
- live cell, `> 3` live neighbours -> dies (overpopulation)
- dead cell, exactly `3` live neighbours -> becomes alive (reproduction)

"Simultaneously" is the one real trap in the rules: you must not let a cell you
just updated change the count seen by its neighbour. The simplest way to honour
that is to write results into a **fresh board** and never touch the input while
counting. That is O(m*n) extra space and is the right first answer.

## Where the bugs actually hide

I watched the same solution get fixed three times. Each round the rules were
fine; each round the bug was in the neighbour count:

1. **Missing diagonals.** Four hand-written `if`s for up/down/left/right count
   only 4 of the 8 neighbours. The board comes out wrong, not crashed — the
   nastiest kind.
2. **Off-by-one on the edge.** A guard like `if i-1 > 0` silently skips row 0 as
   a neighbour; `if ni > rows` lets `ni == rows` through and then `board[rows]`
   throws `IndexError`.
3. **`and` vs `or` on the center.** Switching to a 3x3 delta loop is the right
   move, but `if di == 0 or jdi == 0: continue` skips every orthogonal neighbour
   and keeps only the diagonals — exactly backwards. You only want to skip the
   one center cell, which is `di == 0` **and** `dj == 0`.

Three different bugs, one location. That is the whole lesson.

## The two habits that kill all three

**Habit 1 — sweep neighbours with a delta loop, skip the center with `and`.**

```python
for di in (-1, 0, 1):
    for dj in (-1, 0, 1):
        if di == 0 and dj == 0:   # skip ONLY the cell itself
            continue
        ...
```

This visits all 8 neighbours automatically — you cannot "forget" a diagonal.

**Habit 2 — bound every index with `0 <= x < len`.**

The valid indices are `0 .. len-1`, so the upper bound is strict (`<`, never
`<=`). The reject test is therefore `x < 0 or x >= len`:

```python
if 0 <= ni < rows and 0 <= nj < cols:
    count += board[ni][nj]   # cells are 0/1, so add the value directly
```

Adding `board[ni][nj]` (a 0 or 1) is cleaner than an `if ... == 1: count += 1`.

## The full solution

```python
def is_live(board, i, j):
    rows, cols = len(board), len(board[0])
    count = 0
    for di in (-1, 0, 1):
        for dj in (-1, 0, 1):
            if di == 0 and dj == 0:
                continue
            ni, nj = i + di, j + dj
            if 0 <= ni < rows and 0 <= nj < cols:
                count += board[ni][nj]
    if board[i][j] == 1:
        return 1 if count in (2, 3) else 0   # survive on 2-3
    return 1 if count == 3 else 0            # born on exactly 3

def game_of_life(board):
    if not board:
        return None
    rows, cols = len(board), len(board[0])
    return [[is_live(board, i, j) for j in range(cols)] for i in range(rows)]
```

## Test it with a blinker, not just a block

A **block** (2x2 of live cells) is a still life — it survives unchanged. That is
a fine test but a weak one: it passes even if your "simultaneous" update is
accidentally sequential. A **blinker** (a row of three) is the real check — it
flips between horizontal and vertical every step:

```
. . .        . 1 .        . . .
1 1 1   ->   . 1 .   ->   1 1 1
. . .        . 1 .        . . .
```

If you mutated the board in place while counting, the blinker breaks immediately.
It is the cheapest test that actually proves simultaneity, so always include one.

## The harder follow-up

The version above uses an extra board. The classic follow-up asks for **O(1)
extra space** — update in place. The trick is to encode both the old and new
state in one cell (e.g. use `2` to mean "was 0, now 1" and `-1` to mean "was 1,
now 0", or pack two bits), count using only the low bit, then do a second pass to
normalise every cell to its new value. Worth trying once you have the copy
version passing — but reach for the copy version first in an interview.

## Takeaways

- The Game of Life rules are not where bugs live — the **neighbour count** is.
- Sweep the 8 neighbours with a `for di/dj in (-1, 0, 1)` loop and skip the
  center with `di == 0 **and** dj == 0`. No hand-written `if`s, no missing
  diagonal.
- Guard every index with `0 <= x < len` — strict upper bound, every time.
- Cells are 0/1, so `count += board[ni][nj]` beats an `if == 1` check.
- Write into a fresh board to get simultaneity for free; test with a **blinker**,
  not just a still life.
- Reflex: run the file before trusting it. A one-character typo (`resilt` for
  `result`) is a free catch that a single run flags instantly.
