# Azure Table Storage vs Cosmos DB Table API

**Question:** Azure gives me two things both called a "Table." One lives inside a storage account (the same account that holds my blobs), the other is a Cosmos DB account. The SDK is identical — I open a `TableClient`, I write rows with a `PartitionKey` and a `RowKey`, I do point lookups. If the code is the same, does it actually matter which one is behind it?

**Answer:** Yes, a lot. They speak the same wire protocol and the same client library can point at either, so the code looks interchangeable. Underneath they are two different products with different billing, latency guarantees, indexing, scale limits, and global-distribution stories. Use **Table Storage** when you want cheap, simple, best-effort key-value storage sitting next to your blobs. Use **Cosmos DB Table API** when you need single-digit-millisecond reads, guaranteed throughput, rich queries, or multi-region replication — and you're willing to pay for it.

I ran into this because one service reads the same logical record from two places: a storage-account table (the authoritative source, co-located with other account data) with a Cosmos table as a fallback. Same `TableClient` code, two very different backends — so it's worth knowing exactly what you trade.

## Same API, different engine

Both services implement the **Azure Table protocol** (OData over HTTP). That's why one SDK works for both:

```python
from azure.data.tables import TableClient

# Table Storage — the table service of a storage account
client = TableClient.from_connection_string(
    "DefaultEndpointsProtocol=https;AccountName=myacct;"
    "AccountKey=...;EndpointSuffix=core.windows.net",
    table_name="records",
)

# Cosmos DB Table API — a Cosmos account exposed as tables
client = TableClient.from_connection_string(
    "DefaultEndpointsProtocol=https;AccountName=mycosmos;"
    "AccountKey=...;TableEndpoint=https://mycosmos.table.cosmos.azure.com:443/;",
    table_name="records",
)

# identical from here on
entity = client.get_entity(partition_key="pk-1", row_key="rk-1")
```

The **connection string** is the tell: Table Storage ends in `core.windows.net`; Cosmos has a `TableEndpoint=...table.cosmos.azure.com`. Same `get_entity`, same `query_entities`, same `upsert_entity`.

## Where the data physically lives

- **Table Storage** is one of the four services of a **general-purpose storage account** (blob, file, queue, table). The table shares the account's billing, keys, firewall, and region. If you already have a storage account for blobs, a table there is "free real estate."
- **Cosmos Table API** is a **separate Cosmos DB account**. It's a database product that happens to wear a Table-shaped API so you can lift-and-shift Table Storage code onto it.

That co-location matters operationally: with Table Storage, one account key and one firewall rule cover both your blobs and your tables. Cosmos is a different account with its own keys and its own networking.

## The differences that actually bite

| | Table Storage | Cosmos DB Table API |
|---|---|---|
| **Cost model** | Per-GB stored + per-transaction. Very cheap. | Provisioned RU/s **or** serverless RU. You pay for throughput whether or not you use it (provisioned). |
| **Latency SLA** | None on latency; best-effort. | SLA-backed single-digit-ms reads/writes at p99. |
| **Indexing** | Only `PartitionKey` + `RowKey` are indexed. Any other filter is a **scan**. | **Every property auto-indexed.** Filtering on a non-key column is still fast. |
| **Global distribution** | Single region (+ optional read replica via RA-GRS). | Turnkey multi-region, multi-master writes. |
| **Throughput** | Scales up to account limits automatically; no reservation. | You provision (or serverless-meter) RU/s; throttles with `429` past the limit. |
| **Max item / partition** | 1 MB entity, 1000-entity batch, 20k ops/s per partition (soft). | 2 MB item, higher per-partition throughput. |
| **Consistency** | Strong within the region. | Five tunable levels (strong → eventual). |

The two that surprise people most:

**1. Non-key queries.** On Table Storage, a query that filters on anything other than `PartitionKey`/`RowKey` scans the whole partition (or table). It works in a demo, then quietly degrades as the table grows. Cosmos indexes every property, so the same query stays fast — that's a chunk of what the RU price buys you.

**2. You pay for throughput, not storage, on Cosmos.** A nearly-empty Cosmos table with provisioned 400 RU/s still bills 400 RU/s around the clock. An almost-empty Table Storage table costs essentially nothing. For a small, rarely-read lookup table, Table Storage is dramatically cheaper; for a hot path, Cosmos's guaranteed latency is worth it.

## Design the keys the same way — point lookups win on both

Whichever backend you use, the cheapest and fastest access is a **point lookup**: supply both `PartitionKey` and `RowKey` and you get an O(1) `get_entity`. That's one indexed read on Table Storage and one RU on Cosmos.

```python
# cheap on BOTH backends — exact point read
e = client.get_entity(partition_key=tenant_id, row_key=record_id)

# a filter without PartitionKey:
#   Table Storage -> table scan (gets slower as data grows)
#   Cosmos        -> indexed, fast, but costs RUs per matched item
items = client.query_entities("Status eq 'active'")
```

So the key-design rule is identical — make your common access pattern a `(PartitionKey, RowKey)` point lookup — but the **penalty for breaking it differs**: Table Storage punishes you with latency that worsens over time; Cosmos punishes you with RU cost (and `429` throttling) but keeps the latency.

## How to tell which one you're actually talking to

When you inherit code with a bare `TableClient`, check the endpoint, not the SDK call:

```python
url = client.url                       # or parse the connection string
is_cosmos = "cosmos.azure.com" in url  # vs "core.windows.net"
```

Two more giveaways:
- In the portal, Table Storage tables live under a **Storage account → Tables**; Cosmos tables under a **Cosmos DB account → Data Explorer**.
- A `429 Too Many Requests` with a `x-ms-retry-after-ms` header on a *read-mostly* workload is a Cosmos RU throttle — Table Storage throttles differently and far less often for the same shape of load.

## When to pick which

- **Table Storage** — a lookup/config table next to your blobs, write-rarely/read-sometimes, cost-sensitive, single region, and you only ever query by key. This is the "it's basically free and good enough" choice.
- **Cosmos Table API** — latency-sensitive hot path, queries on non-key fields, multi-region reads/writes, or a hard throughput SLA. You're buying guaranteed performance and rich indexing, and paying provisioned/serverless RU for it.
