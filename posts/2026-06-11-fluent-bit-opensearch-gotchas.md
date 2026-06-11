# Five Gotchas in the Fluent Bit → OpenSearch Stack

I run a log pipeline where Fluent Bit tails files on disk and ships them
into OpenSearch (with OpenSearch Dashboards on top). The whole thing
*works* — until it doesn't. Each of the five issues below cost me half
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

## 4. Fluent Bit's filesystem buffer can eat your disk and break OpenSearch

The pipeline ran fine for weeks, then one day a saved-objects import
into OpenSearch Dashboards silently dropped half its records. Some of
my dashboards came back empty. There was no error in the OSD logs.

What had happened, in order:

1. A burst of incoming files generated more events than the OpenSearch
   `[OUTPUT]` could index.
2. Fluent Bit started spilling chunks to its filesystem buffer at
   `/tmp/fluentbit-storage`. There was no cap, so it kept spilling.
3. The buffer reached **34 GB on a 50 GB container overlay**, tripping
   the OpenSearch flood-stage watermark (default 95% full).
4. OpenSearch flipped `.kibana_1` to read-only as a self-protection
   measure.
5. The saved-objects import was running at exactly the wrong moment.
   The writes silently failed, the importer reported success anyway,
   and I was left with a half-populated dashboard set.

Three knobs together fix this:

```ini
[INPUT]
    Name tail
    ...
    # Pause the tail input when in-memory chunks fill, instead of
    # spilling to disk without limit.
    storage.type                      filesystem
    storage.pause_on_chunks_overlimit on
    # Bound per-input memory; reduces downstream index pressure.
    Mem_Buf_Limit                     64MB

[OUTPUT]
    Name es
    ...
    # Hard cap on the filesystem-buffered queue for THIS output.
    # When the cap is hit, oldest chunks are dropped rather than
    # the disk filling up.
    storage.total_limit_size 4G
    # One worker is gentler on smaller OpenSearch clusters --
    # otherwise you'll see HTTP 429s pile up under load.
    Workers 1
```

The first two stop unbounded spill at the source. The third puts a hard
ceiling on what the output queue can hold. Together they trade "no
data loss ever" for "the system stays alive under backpressure," which
is the right trade for an observability pipeline — a dashboard with a
gap in it is much better than no dashboard at all.

**Confirm it's working:** under sustained load, check
`/tmp/fluentbit-storage`. If the directory hovers below 4 GB and Fluent
Bit's log shows occasional `pause` / `resume` messages on the input,
backpressure is being applied at the right layer. If the directory is
growing without bound, one of the knobs is missing or misplaced.

(Note: `storage.pause_on_chunks_overlimit` is a *per-input* setting.
Putting it in `[SERVICE]` does nothing — Fluent Bit silently ignores
it. Easy mistake.)

---

## 5. Fluent Bit will quietly cap out at 1024 open file descriptors

By default a Linux process gets `RLIMIT_NOFILE = 1024`. Fluent Bit's
`tail` input keeps one open file descriptor per matched file, plus a
handful for its own SQLite DB, sockets, and internal buffers. So once
your `Path` glob matches around 950 files, the next inotify-add tries
to open a new fd, fails with `EMFILE`, and the file is silently never
tailed. The `fluent-bit.log` shows zero errors at the default
verbosity.

Two ways to fix this. We picked the second.

**Option A: raise the limit.** Add `ulimit -n 65536` to the
container's entrypoint (or `LimitNOFILE=65536` in the systemd unit
file). Works, but you're treating the symptom — Fluent Bit will tail
65535 files now, which is *more* memory + CPU pressure per inotify
event, not less.

**Option B: bound the working set.** Don't put all your files in the
tailed directory at once. Stage them somewhere fluent-bit doesn't
watch, batch them, and atomically move each batch into the watched
directory just before it's ready to read. Fluent Bit only ever sees a
bounded number of files — in our case, one batch of 200 at a time —
which sits comfortably under 1024.

Concretely, the on-disk layout looks like:

```
/opensearch_staging/       <-- fluent-bit does NOT watch this dir
    a/                     <-- copy raw logs here
    b/                     <-- pre-process JSON-encodes them in place
    c/

/opensearch_target/        <-- fluent-bit's Path glob points here
    a/                     <-- os.rename'd in from staging
    b/                     <-- atomic on same FS = no half-written files
    c/
```

The processor copies → pre-processes → `os.rename`s each file into
target. Same-filesystem rename is atomic, so Fluent Bit's first
inotify event for a file is for a fully-finished file — no half-parsed
JSON. Once a batch is drained downstream into OpenSearch, the target
dir is wiped before the next batch lands.

Side benefits this happens to give you:

- Pre-processing failures stay in staging and never confuse the tail
  input.
- A crashed pre-processor is recoverable: just wipe staging and
  restart from the source. Nothing leaked through to the index.
- The `Read_from_Head true` semantics make sense again — every file
  in the target dir is fresh, so reading from the head is exactly
  what you want.

If you're tempted to skip this and just raise the ulimit: do whichever
fits your environment, but remember that 1024 was the symptom, not
the disease. The disease is "one file = one open fd, and you have
unbounded files." Capping the working set fixes both.

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
