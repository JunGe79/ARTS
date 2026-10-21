# CTE vs Temp Table in SQL Server

**Question:** A `WITH` (CTE) and a `#temp` table can hold "the same"
intermediate result. If the rows come out identical, what's the actual
difference — and when does it matter which one I use?

**Answer:** A CTE is an inline definition the optimizer expands into your
query each place it's referenced — no storage, re-evaluated per reference,
alive for one statement only. A `#temp` table is physically written to
tempdb once, has statistics, can be indexed, and lives for the whole
batch. The rows are usually identical; the difference is *how* and *how
many times* the work runs, and whether the optimizer can see across the
boundary.

## The one-line mental model

- **CTE** = a named inline subquery. Think "logical macro / one-shot
  view." The optimizer folds it into a single plan.
- **Temp table** = a real table in tempdb. Think "materialize now, read
  later." It's an optimization **fence**.

## Side by side

| | `WITH` (CTE) | `#temp` table |
|---|---|---|
| Storage | None — expanded inline | Materialized in tempdb |
| Referenced N times | Re-evaluated **N times** | Computed **once**, read N times |
| Scope | The **one** statement that follows | Whole **batch / session** (+ nested procs) |
| Statistics / indexes | None (uses base-table stats) | Has row stats; can be indexed |
| Optimizer boundary | Transparent — predicates push through | A fence — fresh plan reads it |
| Capture `INSERT … EXEC proc` | **No** | **Yes** |

## "Re-evaluated N times" is the big one

A CTE is not a cached result. If you reference it three times, its
definition is inlined three times, so the underlying work runs three
times.

```sql
;WITH Big AS (
    SELECT key, SUM(amount) AS total
    FROM huge_table           -- expensive scan + aggregate
    GROUP BY key
)
SELECT * FROM Big a JOIN Big b ON a.key = b.key + 1;
-- 'Big' is referenced twice -> the huge scan/aggregate can run TWICE
```

Swap to a temp table and the expensive part runs once:

```sql
SELECT key, SUM(amount) AS total
INTO #Big
FROM huge_table
GROUP BY key;                 -- runs ONCE, lands in tempdb

SELECT * FROM #Big a JOIN #Big b ON a.key = b.key + 1;  -- reads cheaply
```

SQL Server has no `MATERIALIZE` hint for this (unlike Postgres/Oracle).
If you need "compute once, reuse many times," that's a temp table.

## The optimizer fence cuts both ways

Because a CTE is transparent, the optimizer can push a filter from the
outer query *into* the CTE and reorder joins across it — often a win.

Because a temp table is a fence, the optimizer plans the fill and the
read separately. That's a loss when it breaks a good all-in-one plan —
but a **win** on big or skewed data, because the temp table carries real
statistics (and you can index it), so the second query gets an accurate
plan instead of a guess inlined from base-table stats.

Rough rule: small/medium and referenced once → CTE. Large, skewed, or
referenced several times → temp table.

## The case only a temp table can do

You cannot put `EXEC <proc>` inside a CTE or a subquery. The only way to
consume a stored procedure's result set in a later set-based query is to
land it in a table first:

```sql
CREATE TABLE #rows (id BIGINT, label NVARCHAR(40));
INSERT INTO #rows (id, label) EXEC dbo.ClassifyRows;  -- only a table can capture this

;WITH scoped AS (            -- now a CTE can refine it
    SELECT r.id, r.label, p.ownerId
    FROM #rows r
    JOIN #perms p ON p.id = r.id   -- INNER join also filters out rows the caller can't see
)
SELECT ... FROM scoped ...;
```

This pattern is common: a **temp table** captures the proc output (the
expensive, must-run-once part), and a **CTE** on top joins it to a
permissions/lookup table and keeps a tidy name for the final query. The
temp table isn't optional there — `INSERT … EXEC` needs a real table.

## A word on table variables

`@tablevar` is a third option. Closer to a temp table (it is materialized
and can capture `INSERT … EXEC`), but historically it had **no useful
statistics** (the optimizer assumed 1 row), which wrecked plans on larger
sets. Modern SQL Server (2019+, deferred compilation) improved this, but
for anything non-trivial a `#temp` table is still the safer default.

## Takeaways

- Same rows ≠ same execution. CTE = inline, re-run per reference,
  one-statement scope, no stats. Temp table = materialized once, reusable,
  batch scope, has stats, indexable.
- Referenced multiple times and expensive? Temp table — a CTE re-runs the
  work each reference; there's no `MATERIALIZE` hint in SQL Server.
- Capturing `INSERT … EXEC proc`? Must be a table (temp table or table
  variable), never a CTE.
- CTE lets the optimizer see through it (predicate pushdown); temp table
  is a fence that, in exchange, gives accurate stats on big/skewed data.
- Default: CTE for readability on small/once-used sets; temp table when
  size, reuse, stats, or an `EXEC` capture demand it.
