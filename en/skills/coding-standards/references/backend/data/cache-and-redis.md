# Cache and Redis

This document covers server-side caching. For distributed locks, see [consistency.md](../reliability/consistency.md). For general performance optimization methods, see [performance.md](../reliability/performance.md).

## Whether to use a cache

### Confirm the cache pays off first [Recommended]

Caches suit read-heavy, write-light scenarios that tolerate briefly stale data. When most requests miss, the cost of checking and writing the cache can exceed the benefit.

**Source:** Azure: Cache-Aside Pattern

### A cache is not the primary data store [Required]

The database is the source of truth. A cache entry can expire or be evicted at any time, so business decisions must not depend on cached values.

**Source:** Azure: Caching Guidance

### Be careful when caching sensitive data [Required]

Read permission checks and security-related data directly from the primary store. When caching is truly needed, isolate keys per user and control who can access the cache instance.

**Source:** Azure: Cache-Aside Pattern

### Fall back to the backing store when an optional cache fails [Recommended]

The project must state whether the cache is a required dependency. When an optional cache is unavailable, read from the backing store, and put a timeout on waiting for the cache. High-traffic systems must assess the load this puts on the database. For techniques, see [resilience.md](../reliability/resilience.md).

**Source:** Azure: Caching Guidance

## Cache patterns

### Use Cache-Aside by default [Recommended]

On read, check the cache first. On a miss, read the database and write the cache. On write, update the database first, then delete the cache entry.

**Why:** Deleting is safer than updating the cache. When two concurrent writes update the cache, the last value written to the cache may be the stale one.

**Example:**

Avoid: deleting the cache entry first, then updating the database. A request that reads in that gap writes the old value back to the cache.

Prefer:

```text
Read: GET order:1001; on a miss, query the database, then SET order:1001 with a TTL
Write: UPDATE orders, then UNLINK order:1001 after commit
```

**Source:** Azure: Cache-Aside Pattern

### Keep read-after-write paths off the stale cache [Recommended]

Cache-Aside does not guarantee that the cache and the database agree. Between a write and the next read, stale values can briefly appear. Paths that must read the new value right after a write should read the database directly or use write-through.

**Source:** Azure: Cache-Aside Pattern

### Use TTL as the consistency fallback [Required]

Set a TTL on every cache entry. The TTL decides the longest that stale data can survive a failed delete or a concurrent race, so set it to what the business can accept. A TTL that is too short causes frequent backing-store reads. One that is too long magnifies the impact of stale data.

**Exceptions:** Static data held entirely in memory can be warmed at startup with no expiry.

**Source:** Azure: Cache-Aside Pattern

### Use shorter TTLs for local caches [Recommended]

Each instance has its own in-process cache, and the copies quickly drift apart. Store only data that rarely changes, or use a shorter TTL.

**Source:** Azure: Cache-Aside Pattern

## Penetration, breakdown, and avalanche

### Guard against the three backing-store load risks [Recommended]

| Problem | Cause | Countermeasure |
|---|---|---|
| Cache penetration | Queries for nonexistent data reach the backing store every time | Cache empty values with a shorter TTL, or use a Bloom filter |
| Cache breakdown | A hot key expires and many requests hit the backing store at once | Use mutual exclusion on the backing-store read, or refresh before expiry |
| Cache avalanche | Many keys expire at once, or the whole cache is unavailable | Add random jitter to TTLs, and rate-limit backing-store reads |

**Example:**

Avoid: using the same TTL for everything during bulk warming.

```text
SET product:1001 ... EX 3600
SET product:1002 ... EX 3600
```

Prefer: adding random jitter to a base TTL.

```text
SET product:1001 ... EX 3487
SET product:1002 ... EX 3712
```

## Key conventions

### Use a uniform segmented key format [Recommended]

Redis has no namespaces, so key structure is what prevents collisions. Separate segments with colons. The first segment is the business prefix, in the format `object_type:ID`.

**Example:**

Avoid:

```text
u1000flw
1001
```

Prefer:

```text
user:1000:followers
order:1001
```

**Source:** Redis Docs

### Keep keys short [Recommended]

Long keys take up memory and are slower to compare during lookups. When a key must be built from long content, hash it first.

**Source:** Redis Docs

### Do not use KEYS in business code [Required]

**Applies to:** production service

`KEYS` severely hurts performance on large datasets. Use it only for debugging. To iterate, use `SCAN`, or maintain the keys you need to enumerate in a set.

**Source:** Redis Docs

### Use hash tags for multi-key operations in a cluster [Recommended]

In Redis Cluster, multi-key operations require all keys to be in the same slot. Use `{...}` to mark the part that is hashed, for example `{user:1000}:following` and `{user:1000}:followers`. Tag values must be spread widely enough, or data concentrates on a few nodes.

**Source:** Redis Docs

## Big keys and hot keys

### Avoid big keys [Recommended]

Cap the value size or element count of a single key. The project sets the threshold. Split collection data into multiple keys by ID or time, and read only the part you need.

**Why:** Redis executes commands on a single thread. Operating on a big key blocks other requests and causes network and memory spikes.

**Tooling:** Find them with `redis-cli --bigkeys`, `--memkeys`, or `--keystats`.

**Source:** Redis Docs

### Delete big keys with UNLINK [Recommended]

`UNLINK` reclaims memory on a background thread and does not block. `DEL` may reclaim synchronously and block other commands.

**Source:** Redis Docs

### Spread hot keys [Recommended]

When traffic to one key exceeds what a single node can handle, use an in-process cache with a short TTL, or copy the value to several suffixed keys and pick one at random on read.

**Tooling:** `redis-cli --hotkeys` finds hot keys. It requires `maxmemory-policy` to be an LFU policy.

**Source:** Redis Docs

## Memory and eviction

### Set a memory limit and eviction policy [Required]

**Applies to:** production service

Set `maxmemory` and an eviction policy on cache instances. 64-bit systems have no memory limit by default. With replicas or persistence, reserve memory for buffers, which do not count toward `maxmemory`.

| Scenario | Policy |
|---|---|
| Pure cache, some data accessed more often | `allkeys-lru`, a reasonable default when unsure |
| All keys accessed at similar rates | `allkeys-random` |
| Data loss not allowed | `noeviction`; write commands return errors once memory is full |

`volatile-*` policies have no effect on keys without an expiry, so the result is the same as `noeviction`.

**Source:** Redis Docs

### Keep cache and persistent data on separate instances [Recommended]

Cache data can be evicted and persistent data cannot, so their eviction policies conflict.

**Source:** Redis Docs

### Monitor the hit rate [Recommended]

Calculate the hit rate from `keyspace_hits` and `keyspace_misses` in `INFO stats`, and adjust TTLs and the eviction policy based on it. For metric alerts, see [metrics-and-alerts.md](../observability/metrics-and-alerts.md).

**Source:** Redis Docs
