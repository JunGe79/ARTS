# Three Gotchas in the Fluent Bit → OpenSearch Stack

I run a log pipeline where Fluent Bit tails files on disk and ships them
into OpenSearch (with OpenSearch Dashboards on top). The whole thing
*works* — until it doesn't. Each of the three issues below cost me half
a day or more before I found the root cause, so I'm writing them down
in case the next engineer is googling the same symptom.

---

## 1. Fluent Bit's `**` is NOT recursive (v4.x)

I had a `tail` input with `Path /var/log/**/*.log` and most of my files
were never tailed. The Fluent Bit log was clean — no errors. Files just
silently never showed up downstream.

It turns out the `**` wildcard in Fluent Bit 4.x expands to **exactly
one** directory segment. It does not recurse, and it does not match zero
segments. So:

| File on disk | Matched by `Path /var/log/**/*.log`? |
|---|---|
| `/var/log/foo.log`         | No  (zero segments not supported) |
| `/var/log/a/foo.log`       | Yes |
| `/var/log/a/b/foo.log`     | No |
| `/var/log/a/b/c/foo.log`   | No |

**Fix:** list every depth explicitly, comma-separated:

```ini
[INPUT]
    Name tail
    Path /var/log/*.log, /var/log/*/*.log, /var/log/*/*/*.log, /var/log/*/*/*/*.log
```

A tiny bash loop builds the list if you have a deep tree:

```bash
for d in 1 2 3 4 5 6 7; do
  stars=""; for ((i=1;i<d;i++)); do stars="${stars}*/"; done
  printf '/var/log/%s*.log,' "$stars"
done
```

**Verify:** run Fluent Bit with `log_level debug` and grep for
`inotify_fs_add` in the output — one line per file actually being
watched.

---

## 2. OpenSearch Dashboards caches the field list per index pattern

I changed an index template, reindexed the data, and the new field still
didn't show up in the Discover filter dropdown. The old field, which I'd
removed, was still there. The cluster `_field_caps` API returned the
correct (new) field set. So why was the UI wrong?

OSD stores each index pattern as a saved object in `.kibana_<N>`. One of
the attributes is a JSON-serialized **cache of the field list**. That
cache is populated when the pattern is created and **never auto-updated**
when the underlying mapping changes. Every dropdown that lets you pick a
field reads from this cache, not from the live cluster.

**Inspect the cache directly** (no OSD needed):

```bash
curl -sku admin:$PW "https://localhost:9200/.kibana_1/_search?q=type:index-pattern" |
  python3 -c '
import sys, json
for h in json.load(sys.stdin)["hits"]["hits"]:
    ip = h["_source"]["index-pattern"]
    fields = json.loads(ip.get("fields","[]"))
    print(ip["title"], "cached fields:", len(fields))
'
```

**Fix:** in OSD, go to Stack Management → Index Patterns → pick the
pattern → click the refresh-field-list icon (a small circular arrow,
top-right). The cache gets rebuilt from the live mapping.

Two follow-on traps if you ship saved-object exports between
environments:

- A snapshot of `.kibana_<N>` carries the cache forward. Restoring it
  resurrects the stale field list.
- `export.ndjson` from OSD includes the cached `fields` string in the
  saved object. Re-importing it replays the cache. If you're editing the
  ndjson by hand, patch the `attributes.fields` string surgically rather
  than dropping it entirely — a fresh import with `fields: "[]"` will
  show empty dropdowns until the user clicks refresh.

---

## 3. The filter autocomplete dropdown silently misses rare values

This one had me staring at the screen for an hour. A value I knew was in
the index — I could see it with a direct `terms` aggregation — would not
appear when I typed its prefix into the filter editor. The UI just
offered "Hit ↵ to add `<prefix>` as a custom option," as if the value
didn't exist.

The autocomplete endpoint
(`/api/opensearch-dashboards/suggestions/values/...`) runs a `terms`
aggregation with `terminate_after: 100000` by default. That tells
OpenSearch to scan **only the first 100k documents per shard**. A value
that lives past that window is invisible to the dropdown — even though
it's in the index.

**Smoking gun:** empty query returns the top 10 most common values.
Prefix query for a rare value returns `[]`. Same `terms` aggregation
without `terminate_after` against the cluster directly finds it
instantly.

**Fix:** bump the setting in `opensearch_dashboards.yml` and restart
OSD:

```yaml
opensearchDashboards.autocompleteTerminateAfter: 10000000
```

Two naming traps to know about for OpenSearch Dashboards 3.x:

- The Kibana / OSD 2.x key was `data.autocomplete.valueSuggestions.terminateAfter`. OSD 3.x **rejects this on startup** ("definition for this key is missing") and refuses to boot.
- The value must be a **plain integer**. `10000000ms` is rejected with "must be a number." No unit suffix.

---

## What ties these together

In all three cases the error message was either absent or pointed
somewhere unrelated. The Fluent Bit log was happy with no files; OSD
showed a stale dropdown with no warning; the autocomplete just shrugged.
Each piece of the pipeline was doing exactly what its config said, and
the config silently did the wrong thing.

If you take one habit from this post: when something in this stack "just
doesn't work" and nothing logs an error, **bypass the UI and hit the API
directly**. A `curl` against `_field_caps` or `_search?aggs=...` will
tell you the truth in seconds. The UI layer's caching, sampling, and
glob-quirks have hidden too much from me too many times.
