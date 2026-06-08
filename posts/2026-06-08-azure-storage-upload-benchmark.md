# Which Azure storage upload path is actually fastest?

I needed to know which way of writing to Azure Storage from an Azure Container Instance (ACI) is fastest, so I ran a small benchmark on a live container.

## What I tested

Inside the ACI, the same data was uploaded three ways:

1. **SMB** — an Azure Files share mounted as `/mnt/fileshare`, written with plain `cp`.
2. **blobfuse** — an Azure Blob container mounted as `/mnt/blob` via blobfuse2, also written with `cp`.
3. **azcopy** — the same Blob container, but uploaded with Microsoft's `azcopy` CLI instead of a mount.

Two workloads that stress different bottlenecks:

- **10,000 tiny files** (8 bytes each) — stresses per-file overhead.
- **10 large files** (1 GB each) — stresses raw throughput.

Each run used reasonable parallelism: 16 worker threads for the tiny files, 4 for the large ones.

### Setup the blobfuse numbers depend on

The blobfuse2 mount on this ACI was tuned aggressively for write throughput:

```yaml
block_cache:
  mem-size-mb: 4096       # 4 GB in-memory write cache
  prefetch: 12
  parallelism: 64         # 64 concurrent upload workers
```

That matters a lot for how to read the results — see *Caveats* below.

## Results

| Case | SMB | blobfuse | azcopy |
|---|---|---|---|
| 10,000 × 8-byte | 84–265 s | 117 s | **8.8 s** |
| 10 × 1 GB | 106 s | 30 s | **27 s** |

The SMB row shows a range because Standard Azure Files shares have an IOPS burst credit. While the credit is fresh the share is faster; once the credit is spent you drop to baseline and the same job takes much longer.

## What the numbers say

**azcopy wins every test.** Tiny files: ~30× faster than SMB, ~13× faster than blobfuse. Large files: ~4× faster than SMB, slightly faster than blobfuse.

**SMB is consistently the slowest.** Two reasons:

- Every file write is a synchronous round-trip (create → write → close). That kills small-file performance.
- Standard Azure Files SMB tops out around 100 MB/s for sequential throughput. That caps large-file performance.

**blobfuse is solid for large files, decent for tiny ones.** Parallel block uploads and the in-memory cache hide a lot of latency.

## Caveats (so you can decide how much to trust this)

- **blobfuse is slightly favored** by the way I measured. With a 4 GB block cache, the small-file workload (40 MB total) fits entirely in memory — so `cp` returned as soon as bytes were in the cache, while the cache flushed asynchronously to Azure in the background. The large-file workload (10 GB) is too big for the cache, but the 64-way upload parallelism and 12-block prefetch still hide latency well. SMB has no equivalent client-side cache by default, and azcopy waits for all PUTs to commit before exiting — so for those two, the time you see is closer to "end-to-end". A stricter comparison would do an explicit `sync` after each `cp` on the blobfuse mount.
- **Single run, single VNet, single ACI.** Real numbers will shift with share tier, blob SKU, region, and whether the Azure Files burst credit was full or empty when the run started.
- The SMB share was Standard (not Premium). Premium has much higher per-share IOPS / throughput limits and would tighten the gap on the large-file case.

## Practical take-aways

- If you control the upload code, **call `azcopy` directly** instead of writing to a mount. It uses concurrent block uploads, reuses HTTP connections, and skips the kernel CIFS/FUSE layers — and the time you see is the time the data actually landed.
- If you must mount, **blobfuse beats SMB** for blob-style workloads, especially when you can give it a generous block cache.
- Reach for Azure Files SMB only when you genuinely need POSIX semantics (shared random reads/writes, file locks). For write-once / read-once flows it's the slowest path of the three.
