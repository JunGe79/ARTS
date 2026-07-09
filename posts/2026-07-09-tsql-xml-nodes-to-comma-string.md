# Reading XML in T-SQL: `.nodes()`, `CROSS APPLY`, and Turning Rows Into One String

**Question:** I have an XML column. The regions I need are nested at
different depths, and one cloud vendor stores the region as an attribute
instead of an element. I want one row per record, with all regions in a
single comma-separated cell. What does this query actually do?

**Answer:** `.nodes()` shreds the XML into rows. `.value()` pulls a scalar
out of each row. `UNION` merges the two different XML shapes and removes
duplicates. Then `FOR XML PATH('')` is abused as a string aggregator to
collapse the rows back into one string, and `STUFF` trims the separator
it leaves at the front.

Here is the query we will take apart:

```sql
ISNULL(STUFF((
    select distinct N', ' + q.rn
    from (
        select r.n.value('@name','nvarchar(128)') as rn
        from   T.policyXml.nodes('//regionDetails') r(n)
        union
        select a.n.value('@region','nvarchar(128)')
        from   T.policyXml.nodes('//regionSpecificInfoList[@region]') a(n)
    ) q
    where q.rn is not null and q.rn <> N''
    for xml path(''), type).value('.','nvarchar(max)'), 1, 2, N''), N'') as [Regions]
```

## Setup: run this first

Every sample below runs on its own. Start with this table. Note the three
vendors nest the region differently — that is the whole reason the query
looks the way it does.

```sql
SET QUOTED_IDENTIFIER ON;   -- required for any XML method

DECLARE @CloudTarget TABLE (id int, name varchar(30), policyXml xml);

INSERT INTO @CloudTarget VALUES
(1, 'aws-target', '<CloudTargetConfig>
   <awsDetails>
     <regionSpecificDetails><regionDetails name="us-east-2"/></regionSpecificDetails>
     <regionSpecificDetails><regionDetails name="us-east-1"/></regionSpecificDetails>
     <regionSpecificDetails><regionDetails name="us-east-2"/></regionSpecificDetails>
   </awsDetails>
 </CloudTargetConfig>'),
(2, 'gcp-target', '<CloudTargetConfig>
   <gcpDetails>
     <regionSpecificInfoList><regionDetails name="asia-south1"/></regionSpecificInfoList>
   </gcpDetails>
 </CloudTargetConfig>'),
(3, 'azure-target', '<CloudTargetConfig>
   <azureDetails>
     <regionSpecificInfoList region="eastus2"/>
   </azureDetails>
 </CloudTargetConfig>'),
(4, 'no-region', '<CloudTargetConfig><awsDetails/></CloudTargetConfig>');

SELECT * FROM @CloudTarget;
```

AWS and GCP use a `<regionDetails name="...">` **element**. Azure uses a
`region="..."` **attribute**. Target 4 has no region at all.

## 1. `.nodes()` turns XML into rows

`.nodes(xpath)` shreds an XML value into a rowset: **one row per matching
node**. Zero matches gives zero rows, not a NULL row.

```sql
SET QUOTED_IDENTIFIER ON;
DECLARE @x xml = '<a>
  <regionDetails name="us-east-2"/>
  <regionDetails name="us-east-1"/>
  <regionDetails name="us-east-2"/>
</a>';

SELECT r.n.value('@name','nvarchar(128)') AS rn
FROM   @x.nodes('//regionDetails') r(n);
-- us-east-2
-- us-east-1
-- us-east-2
```

`.value(xquery, sqltype)` runs the XQuery **relative to the current node**
and returns one SQL scalar. Here `@name` reads that element's `name`
attribute. A missing attribute gives `NULL`, not an error.

`.value()` needs a **singleton** — an expression that can only yield one
item. `@name` qualifies. A path like `/a/b` does not, which is why you
often see `(/a/b)[1]`.

## 2. `r(n)` is just aliases — not `CROSS APPLY`

`r(n)` means: table alias `r`, column alias `n`. Nothing more.

`.nodes()` returns a one-column rowset and that column has no name, so you
**must** supply both aliases. Leave them out and it is a syntax error.

```sql
SET QUOTED_IDENTIFIER ON;
DECLARE @x xml = '<a><regionDetails name="us-east-1"/></a>';

-- r(n) present, but nothing is correlated: no CROSS APPLY anywhere
SELECT r.n.value('@name','nvarchar(128)')
FROM   @x.nodes('//regionDetails') r(n);
```

The receiver here is a **variable**, so there is no correlation at all.

## 3. When the receiver is a column, it becomes an implicit `CROSS APPLY`

`CROSS APPLY` is a join whose right side runs **once per left row** and may
reference that row's columns. A plain `JOIN` cannot do that.

So when the receiver is `T.policyXml` — a column of the left table — SQL
Server treats it as `CROSS APPLY`. These two are the same:

```sql
FROM T CROSS APPLY T.policyXml.nodes('//regionDetails') r(n)
FROM T,            T.policyXml.nodes('//regionDetails') r(n)   -- implicit
```

**This matters.** `CROSS APPLY` drops left rows that match nothing, exactly
like an `INNER JOIN`:

```sql
SET QUOTED_IDENTIFIER ON;
DECLARE @CloudTarget TABLE (id int, name varchar(30), policyXml xml);
INSERT INTO @CloudTarget VALUES
(1,'aws-target','<c><regionDetails name="us-east-1"/></c>'),
(4,'no-region', '<c/>');

-- target 4 silently DISAPPEARS
SELECT t.name, r.n.value('@name','nvarchar(128)') AS rn
FROM   @CloudTarget t
CROSS APPLY t.policyXml.nodes('//regionDetails') r(n);
```

That is why the real query keeps the region logic inside a **correlated
subquery in the SELECT list**. No rows means the subquery returns `NULL`,
and the record still shows up with a blank cell. `OUTER APPLY` would also
keep the row, but then you would need a `GROUP BY` to squash the
multi-region records back into one row each.

## 4. `n` is a node reference, not an XML value

The column `n` is typed `xml`, but it is a **pointer into the original
document**, not a standalone value. You cannot select it.

```sql
SET QUOTED_IDENTIFIER ON;
DECLARE @x xml = '<a><regionDetails name="us-east-1"/></a>';

-- Msg 493: the column 'n' ... cannot be used directly
SELECT r.n FROM @x.nodes('//regionDetails') r(n);

-- legal: only exist(), nodes(), query(), value(), and IS [NOT] NULL
SELECT r.n.query('.') AS node_xml            -- <regionDetails name="us-east-1"/>
FROM   @x.nodes('//regionDetails') r(n);
```

Use `.query('.')` when you actually want the subtree as XML.

## 5. `//` searches at any depth

`//` is the descendant-or-self axis. It finds the element **anywhere in the
document**, at any nesting level.

```sql
SET QUOTED_IDENTIFIER ON;
DECLARE @x xml = '<c><aws><deep><regionDetails name="us-east-1"/></deep></aws></c>';

SELECT r.n.value('@name','nvarchar(128)')
FROM   @x.nodes('//regionDetails') r(n);   -- us-east-1, depth does not matter
```

We need this because AWS puts `regionDetails` under `regionSpecificDetails`
and GCP puts it under `regionSpecificInfoList`. One fixed path would only
catch one vendor.

The trade-off: `//` searches the whole document, so it would also match an
unrelated `regionDetails` elsewhere, and it cannot use an XML index path
lookup as efficiently.

## 6. `UNION` merges the two shapes, and deduplicates

Azure has no `regionDetails` element at all, so `//regionDetails` returns
nothing for it. A second `.nodes()` call picks up the attribute form. The
`[@region]` predicate keeps it from matching GCP's `regionSpecificInfoList`,
which has no `region` attribute.

`UNION` (not `UNION ALL`) also removes duplicates — target 1 lists
`us-east-2` twice.

```sql
SET QUOTED_IDENTIFIER ON;
DECLARE @x xml = '<c>
  <regionSpecificDetails><regionDetails name="us-east-2"/></regionSpecificDetails>
  <regionSpecificDetails><regionDetails name="us-east-1"/></regionSpecificDetails>
  <regionSpecificDetails><regionDetails name="us-east-2"/></regionSpecificDetails>
  <regionSpecificInfoList region="eastus2"/>
</c>';

SELECT rn FROM (
    SELECT r.n.value('@name','nvarchar(128)') AS rn
    FROM   @x.nodes('//regionDetails') r(n)
    UNION
    SELECT a.n.value('@region','nvarchar(128)')
    FROM   @x.nodes('//regionSpecificInfoList[@region]') a(n)
) q;
-- eastus2 / us-east-1 / us-east-2   (three rows, us-east-2 deduped)
```

After `.value()`, every row in `q` is a plain `nvarchar(128)`. **The XML is
already gone.** That is why `where q.rn is not null and q.rn <> N''` is a
normal string test.

## 7. `FOR XML PATH(''), TYPE` is a string aggregator

`q` has N rows. The SELECT list needs **one scalar**. Older SQL Server has
no "concatenate rows" aggregate, and `FOR XML` is the only thing that
collapses a rowset into a single value. So people abuse it:

* `PATH('')` — empty element name, so no wrapping tags.
* no column alias — so no column tags either.

What is left is just the text of every row, glued together.

```sql
SET QUOTED_IDENTIFIER ON;
DECLARE @q TABLE (rn nvarchar(128));
INSERT INTO @q VALUES (N'us-east-1'), (N'us-east-2');

SELECT (SELECT N', ' + rn FROM @q FOR XML PATH(''), TYPE).value('.','nvarchar(max)');
-- ", us-east-1, us-east-2"
```

No XML is being read here. XML serialization is only the vehicle.

## 8. `.value('.','nvarchar(max)')` converts back — and un-escapes

`, TYPE` makes `FOR XML` return the `xml` type. `STUFF` needs a string, so
you convert back. `'.'` is the context node, and `.value()` returns its text.

The real reason is **escaping**. `FOR XML` encodes `&`, `<`, `>`:

```sql
SELECT ', a&b' FOR XML PATH('');
-- ', a&amp;b'                                        <-- corrupted

SELECT (SELECT ', a&b' FOR XML PATH(''), TYPE).value('.','nvarchar(max)');
-- ', a&b'                                            <-- correct
```

Without `TYPE` you receive already-escaped text. With `TYPE`, `.value('.')`
decodes it on the way out. Region names have no special characters, so this
is defensive — but it is why this is the canonical form.

## 9. `STUFF` trims the leading separator

Each element is built as `N', ' + rn`, so the separator is prefixed to
**every** row, including the first. The result starts with a stray `", "`.

`STUFF(string, start, length, replaceWith)` deletes `length` characters
starting at 1-based `start`, then inserts `replaceWith`.

```sql
SELECT STUFF('abcdef', 2, 3, 'XY');                   -- aXYef   (delete + insert)
SELECT STUFF('abcdef', 2, 3, '');                     -- aef     (pure delete)
SELECT STUFF(N', us-east-1, us-east-2', 1, 2, N'');   -- us-east-1, us-east-2
```

Why prefix at all? Inside `FOR XML` there is no row number, so you cannot
ask "am I first?". You unconditionally prefix, then chop once at the end.

The `2` is `LEN(N', ')`. Change the separator and you must change the `2`.
That mismatch is the classic bug in this idiom.

## 10. `ISNULL` handles the no-rows case

`FOR XML` over an empty rowset returns `NULL`, not `''`. Without the outer
`ISNULL`, a record with no regions renders as NULL instead of blank.

```sql
SET QUOTED_IDENTIFIER ON;
DECLARE @empty TABLE (rn nvarchar(128));   -- no rows

SELECT (SELECT N', ' + rn FROM @empty FOR XML PATH(''), TYPE).value('.','nvarchar(max)');
-- NULL

SELECT ISNULL((SELECT N', ' + rn FROM @empty FOR XML PATH(''), TYPE)
              .value('.','nvarchar(max)'), N'');
-- '' (empty string)
```

## Putting it together

```sql
SET QUOTED_IDENTIFIER ON;
DECLARE @CloudTarget TABLE (id int, name varchar(30), policyXml xml);
INSERT INTO @CloudTarget VALUES
(1,'aws-target','<c><regionSpecificDetails><regionDetails name="us-east-2"/></regionSpecificDetails>
                    <regionSpecificDetails><regionDetails name="us-east-1"/></regionSpecificDetails>
                    <regionSpecificDetails><regionDetails name="us-east-2"/></regionSpecificDetails></c>'),
(2,'gcp-target','<c><regionSpecificInfoList><regionDetails name="asia-south1"/></regionSpecificInfoList></c>'),
(3,'azure-target','<c><regionSpecificInfoList region="eastus2"/></c>'),
(4,'no-region','<c/>');

SELECT T.name,
  ISNULL(STUFF((
      SELECT DISTINCT N', ' + q.rn
      FROM (
          SELECT r.n.value('@name','nvarchar(128)') AS rn
          FROM   T.policyXml.nodes('//regionDetails') r(n)
          UNION
          SELECT a.n.value('@region','nvarchar(128)')
          FROM   T.policyXml.nodes('//regionSpecificInfoList[@region]') a(n)
      ) q
      WHERE q.rn IS NOT NULL AND q.rn <> N''
      FOR XML PATH(''), TYPE).value('.','nvarchar(max)'), 1, 2, N''), N'') AS Regions
FROM @CloudTarget T;
```

| name | Regions |
|---|---|
| aws-target | us-east-1, us-east-2 |
| gcp-target | asia-south1 |
| azure-target | eastus2 |
| no-region | *(blank)* |

Target 4 survives with a blank cell. If you had used `CROSS APPLY` at the
top level it would have vanished.

## On SQL Server 2017+, most of this disappears

`STRING_AGG` puts the separator **between** elements, so there is nothing to
trim and no XML round-trip:

```sql
SELECT STRING_AGG(q.rn, N', ') FROM ( ... ) q;
```

That deletes the `FOR XML`, the `TYPE`, the `.value('.')`, and the `STUFF`.
Use it if you can guarantee the version. The old form exists because a
query shipped inside a product may run on whatever server the customer has.

## Takeaways

* `.nodes()` shreds XML into rows; `.value()` pulls one scalar per row.
* `r(n)` is only aliases. `CROSS APPLY` comes from the receiver being a
  **column**, not from `r(n)`.
* `CROSS APPLY` silently drops rows that match nothing. Aggregate in a
  correlated subquery when you need to keep them.
* The column from `.nodes()` is a node **reference**. Only `exist()`,
  `nodes()`, `query()`, `value()`, and `IS NULL` work on it.
* `//` matches at any depth — handy when vendors nest the same element
  differently.
* After `.value()`, you are working with strings, not XML.
* `FOR XML PATH(''), TYPE` is a string aggregator, not XML processing.
  `TYPE` + `.value('.')` is what protects `&` and `<` from being escaped.
* `STUFF(s,1,2,'')` removes the leading `", "`. The `2` must match the
  separator length.
* `FOR XML` over zero rows gives `NULL` — wrap it in `ISNULL`.
* Any XML method needs `SET QUOTED_IDENTIFIER ON`, or you get `Msg 1934`.
