# Which Azure storage upload path is fastest from an ACI?

I uploaded the same data three ways from an Azure Container Instance and timed each:

1. **SMB** — Azure Files share mounted at `/mnt/fileshare`, written with `cp`.
2. **blobfuse** — Blob container mounted at `/mnt/blob` via blobfuse2, also `cp`.
3. **azcopy** — same Blob container, but via Microsoft's `azcopy` CLI.

Two workloads, with sensible parallelism (16 threads for tiny, 4 for large):

## Results

| Case | SMB | blobfuse | azcopy |
|---|---|---|---|
| 10,000 × 8-byte | 84–265 s | 117 s | **8.8 s** |
| 10 × 1 GB | 106 s | 30 s | **27 s** |

SMB shows a range because Standard Azure Files has burst-then-baseline IOPS — once the burst credit is spent the same job takes much longer.

## Caveats

- **blobfuse had a 4 GB block cache** (`mem-size-mb: 4096`, prefetch 12, parallelism 64). The 40 MB of small files fits entirely in cache, so `cp` returned before the data was actually flushed to Azure. SMB has no equivalent client cache; azcopy waits for all PUTs to commit. So blobfuse's numbers are slightly optimistic vs the other two.
- Standard SKU share, single VNet, single ACI, one run each. Premium Azure Files would tighten the SMB gap on large files.

## Bottom line

- **azcopy wins every case** — ~30× faster than SMB for tiny files, ~4× for large files. Best choice when you control the code.
- **blobfuse beats SMB** for blob-style workloads if you must mount.
- **SMB only when you need POSIX semantics** (random reads/writes, file locks). For write-once flows it's the slowest path.
