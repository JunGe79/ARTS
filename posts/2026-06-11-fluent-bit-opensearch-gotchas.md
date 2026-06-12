# Five Gotchas in the Fluent Bit → OpenSearch Stack

Real-world traps I hit running a log pipeline of Fluent Bit shipping
into OpenSearch + OpenSearch Dashboards. Each one cost me hours
before I found the cause, so I'm writing them down.

---

## 1. `**` in a Fluent Bit Path is NOT recursive (v4.x)

In Fluent Bit 4.x, `**` expands to **exactly one** directory segment.
It does not recurse, and it does not match zero segments.

`Path /var/log/**/*.log` matches `/var/log/a/foo.log` but NOT
`/var/log/foo.log` or `/var/log/a/b/foo.log`.

**Fix:** list every depth explicitly:
```ini
Path /var/log/*.log, /var/log/*/*.log, /var/log/*/*/*.log, ...
```

**Verify:** run with `log_level debug` and grep `inotify_fs_add` —
one line per file actually being watched.

---

## 2. OpenSearch Dashboards caches the field list per index pattern

Each index pattern in `.kibana_<N>` stores a JSON-serialized field
cache. It is **never auto-updated** when the underlying mapping
changes. Every dropdown (Discover filter editor, Visualize field
picker, etc.) reads the cache, not the live `_field_caps`.

**Fix:** Stack Management → Index Patterns → click the refresh-field-list
icon. Or via API:
```bash
POST /api/index_patterns/index_pattern/<saved-id>/_refresh_fields
```

**Trap:** snapshots and `export.ndjson` carry the cached `fields`
forward. Patch the cache string surgically rather than re-exporting,
or fresh installs land with stale dropdowns.

---

## 3. Filter autocomplete silently misses rare values

The autocomplete endpoint runs a `terms` aggregation with
`terminate_after: 100000` — only the **first 100k docs per shard** are
scanned. A value that lives past that window is invisible to the
dropdown even though it's in the index.

**Fix** in `opensearch_dashboards.yml`:
```yaml
opensearchDashboards.autocompleteTerminateAfter: 10000000
```

---

## 4. Fluent Bit's filesystem buffer can eat your disk

When OpenSearch can't keep up with the rate of incoming logs, Fluent
Bit saves the overflow to disk at `/tmp/fluentbit-storage` so nothing
is lost. The default has no size limit. If the backlog persists, that
directory keeps growing until it fills the container's disk. Once the
disk is almost full, OpenSearch trips a safety switch and turns every
index read-only — including `.kibana_1`.

**Fix:**
```ini
[INPUT]
    storage.type                      filesystem
    storage.pause_on_chunks_overlimit on
    Mem_Buf_Limit                     64MB

[OUTPUT]
    storage.total_limit_size 4G
    Workers                  1
```

Trade-off: `storage.total_limit_size` drops oldest chunks once
exceeded. Data loss only kicks in if OpenSearch is unhealthy long
enough to queue >4 GB.


---

## 5. Fluent Bit caps out at 1024 open file descriptors

Default `RLIMIT_NOFILE = 1024`. Fluent Bit tail keeps one fd per
matched file. Once your glob matches ~950 files, new inotify-adds get
`EMFILE` and the file is silently never tailed.

**Two fixes:**

- **Raise the limit** — `ulimit -n 65536` in entrypoint, or
  `LimitNOFILE=` in the systemd unit. Treats the symptom. **Doesn't
  work on Azure Container Instances** — ACI ignores entrypoint
  `ulimit` and there's no systemd, so you're locked into option 2.
- **Bound the working set** — stage files in a directory Fluent Bit
  doesn't watch, then `os.rename` each batch (atomic on same FS) into
  the watched dir. Fluent Bit only ever sees one batch worth of files.

Layout for option 2:
```
/staging/   <-- not watched; copy + pre-process here
/target/    <-- Fluent Bit Path glob points here; one batch at a time
```

Side benefits: failed pre-processing stays in staging, no half-written
JSON ever reaches the tail input, and `Read_from_Head true` does
exactly what it says.

---

## What ties these together

In every one of these cases the error message was either absent or
pointed somewhere unrelated. Each component was doing exactly what its
config said, and the config silently did the wrong thing.

When something in this stack "just doesn't work" and nothing logs an
error, **bypass the UI and hit the API directly**. `_field_caps`,
`_search?aggs=...`, `inotify_fs_add` — a few curls beat hours of
guessing at the GUI.
