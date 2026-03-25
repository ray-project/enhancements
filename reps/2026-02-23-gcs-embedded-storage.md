# REP: Embedded Storage Backend for GCS Fault Tolerance

## Summary

### General Motivation

Ray's Global Control Store (GCS) is a single process on the head node that stores all cluster metadata — node membership, actor state, placement groups, job history, and resource availability. Today, GCS fault tolerance requires an **external Redis instance** as a persistence backend. While this works, it introduces operational complexity and a fragile dependency:

- Redis must be provisioned, monitored, and maintained separately
- The Redis client implementation has multiple P1 bugs: SIGSEGV on connection reset ([#53475](https://github.com/ray-project/ray/issues/53475)), worker pods restarting unexpectedly during recovery ([#52480](https://github.com/ray-project/ray/issues/52480)), silent crashes on timeouts ([#47419](https://github.com/ray-project/ray/issues/47419)), and lost job history after recovery ([#44218](https://github.com/ray-project/ray/issues/44218))
- Redis Sentinel/Cluster topology changes cause GCS crashes ([#46982](https://github.com/ray-project/ray/issues/46982))
- No support for Redis IAM authentication ([#49001](https://github.com/ray-project/ray/issues/49001)) or SSL certificate validation ([#41161](https://github.com/ray-project/ray/issues/41161))

This enhancement proposes an **embedded storage backend** (RocksDB) as an alternative to Redis for GCS fault tolerance. Instead of writing state over the network to an external Redis instance, GCS writes to a local embedded database on a persistent volume. On head node restart, GCS reads state back from the local database and resumes — the same recovery model as Redis-based FT, but without the external dependency.

**Production context:** At LinkedIn, we run distributed PyTorch training jobs on Ray via Flyte on Kubernetes. These are long-running jobs (hours to days) where head node restarts — due to node maintenance, spot preemption, or transient failures — are inevitable. Today, head node failure is catastrophic: all training progress since the last application-level checkpoint is lost, and the entire job must be restarted. GCS fault tolerance with embedded storage, combined with training checkpointing, would allow these jobs to survive head node restarts and resume from where they left off — minimizing repeated computation and eliminating manual intervention.

### Should this change be within `ray` or outside?

This change affects both `ray` (new `StoreClient` implementation in C++) and `kuberay` (PVC provisioning and configuration for the embedded storage path).

## Stewardship

- **Required Reviewers**: @jjyao (GCS maintainer), @kevin85421 (KubeRay / GCS FT integration)
- **Shepherd of the Proposal**: TBD — requesting @jjyao or another GCS-experienced committer

## Design and Architecture

### Current Architecture (Redis based)

```mermaid
graph LR
       GCS1[GCS Server<br/>on Head Node] -->|network writes| Redis[(External Redis)]
       W1[Worker 1] --> GCS1
       W2[Worker 2] --> GCS1
       W3[Worker 3] --> GCS1
```

### Proposed Architecture (RocksDB based)

```mermaid
graph LR
       GCS2[GCS Server<br/>on Head Node] -->|local writes| RDB[(RocksDB on PVC)]
       W4[Worker 1] --> GCS2
       W5[Worker 2] --> GCS2
       W6[Worker 3] --> GCS2
```

GCS fault tolerance is built on a clean `StoreClient` abstraction (`src/ray/gcs/store_client/store_client.h`) with two implementations:

1. **`InMemoryStoreClient`** — hash maps in GCS process memory. Used when FT is disabled. State is lost on crash.
2. **`RedisStoreClient`** — persists state to an external Redis instance. Used when `RAY_REDIS_ADDRESS` is set.

The `StoreClient` interface exposes ~9 async methods operating on a table-oriented key-value model:

```cpp
// Simplified interface
class StoreClient {
  virtual Status AsyncPut(table, key, data, overwrite, callback);
  virtual Status AsyncGet(table, key, callback);
  virtual Status AsyncGetAll(table, callback);
  virtual Status AsyncMultiGet(table, keys, callback);
  virtual Status AsyncDelete(table, key, callback);
  virtual Status AsyncBatchDelete(table, keys, callback);
  virtual Status AsyncGetNextJobID(callback);
  virtual Status AsyncGetKeys(table, prefix, callback);
  virtual Status AsyncExists(table, key, callback);
};
```

GCS recovery flow today (with Redis):

```mermaid
flowchart TD
    A[Head pod crashes] --> B[KubeRay detects pod<br/>failure]
    B --> C[KubeRay restarts<br/>head pod]
    C --> D[GCS process starts,<br/>connects to Redis]
    D --> E[GCS reads all state<br/>from Redis]
    E --> F["Workers reconnect<br/>(within gcs_rpc_server_<br/>reconnect_timeout_s)"]
    F --> G[Cluster resumes operation]
```

### Proposed Change: RocksDB StoreClient

We propose adding a third `StoreClient` implementation: **`RocksDbStoreClient`**.

#### Data Model Mapping

The `StoreClient` interface is table-oriented (table_name, key, value). RocksDB supports this naturally using **column families** (one per table) or **key prefixes** (`{table_name}/{key}`). We propose column families for cleaner isolation:

```cpp
// Each GCS table maps to a RocksDB column family
// Tables: ACTOR, NODE, PLACEMENT_GROUP, JOB, WORKER, etc.
//
// Put("ACTOR", "actor_123", data)  →  CF["ACTOR"].Put("actor_123", data)
// GetAll("ACTOR")                  →  CF["ACTOR"].NewIterator()
```

#### AsyncGetNextJobID

The `RedisStoreClient` uses Redis `INCR` for atomic job ID generation. RocksDB provides `Merge` operators that can implement atomic increment:

```cpp
// Option 1: Read-modify-write under a mutex (simple, GCS is single-process)
// Option 2: RocksDB Merge operator with uint64 addition
//
// Since GCS is a single process, a simple mutex-guarded read-modify-write
// on a reserved key is sufficient. No distributed coordination needed.
```

#### Storage Path and Configuration

```bash
# Enable embedded storage FT
export RAY_GCS_STORAGE=rocksdb
export RAY_GCS_STORAGE_PATH=/data/gcs-state    # must be on a persistent volume

# Existing Redis-based FT remains supported
export RAY_GCS_STORAGE=redis                    # or: export RAY_REDIS_ADDRESS=...
```

#### RocksDB Configuration

GCS metadata is small (10–100 MB across ~10 column families: ACTOR, NODE, JOB, PLACEMENT_GROUP, WORKER, etc.) with moderate write throughput and read-heavy recovery. The configuration priorities are **crash safety** and **fast recovery**, not write throughput.

##### Default Configuration (Hardcoded)

The following are compiled into `RocksDbStoreClient` with conservative values sized for this workload:

| Category | Setting | Default | Rationale |
|---|---|---|---|
| **Durability** | `WriteOptions::sync` | `true` (fsync every write) | GCS acknowledges state changes to callers. Without fsync, a crash can lose acknowledged writes — violating the FT durability contract. On SSD, fsync adds ~0.1–0.5ms per write, acceptable for GCS's moderate write rate. |
| **Durability** | `manual_wal_flush` | `false` | WAL writes flush to OS page cache immediately. Combined with sync writes, ensures durability. |
| **Durability** | `bytes_per_sync` | 1 MB | Periodic sync of SST files during compaction reduces data loss window. |
| **Memory** | Write buffer (per CF) | 16 MB × 2 buffers | Default 64 MB is oversized for CFs holding 1–10 MB each. With ~10 CFs, worst-case memory: `10 × 16 MB × 2 + 32 MB cache ≈ 352 MB`. |
| **Memory** | Block cache (shared) | 32 MB LRU | Covers a large fraction of total data. Hot metadata (active actors, live nodes) stays cached during normal operation. |
| **Compaction** | Style | Level compaction with dynamic level bytes | Correct for mixed read/write with recovery scans. Dynamic level bytes auto-adjusts for small data sizes. |
| **Compaction** | `max_background_jobs` | 2 | One flush thread, one compaction thread. With 10–100 MB data, compaction is rare and fast. |
| **Compaction** | `target_file_size_base` | 4 MB | Sized for small metadata. Smaller files = more granular compaction, faster recovery reads. |
| **Compaction** | `max_bytes_for_level_base` | 32 MB | Most data stays in L0/L1. Default 256 MB is far too large for this workload. |
| **Compression** | Per-level | None (L0–L1), LZ4 (L2+) | No compression on hot levels for fastest recovery reads. LZ4 on cold levels for modest space savings. |
| **Read perf** | Bloom filter | 10 bits/key | ~1% false positive rate. Essential for point lookups (`AsyncGet`, `AsyncExists`). |
| **Read perf** | `max_open_files` | -1 (unlimited) | With tiny data (tens of SST files), keep all files open to avoid table cache misses during recovery. |
| **I/O** | Direct I/O | Disabled | Does not apply to WAL (no durability benefit). OS page cache helps with repeated hot-path reads. Can break on NFS/FUSE-backed PVs in Kubernetes. |
| **Logging** | `info_log_level` | WARN | INFO-level RocksDB logs are extremely verbose. WARN captures actionable events (compaction stalls, corruption warnings). |

##### User-Configurable Options

Operators can tune these settings via environment variables for their deployment:

| Env Var | Default | Description |
|---|---|---|
| `RAY_GCS_STORAGE_PATH` | *(required)* | Path to persistent volume for RocksDB data. |
| `RAY_GCS_ROCKSDB_SYNC_WRITES` | `1` | fsync on every write. Set `0` for dev/test when crash safety is not needed (~5–10× write throughput improvement). |
| `RAY_GCS_ROCKSDB_BLOCK_CACHE_MB` | `32` | Block cache size in MB. Increase for clusters with very large metadata (thousands of actors/jobs). Range: 8–256. |
| `RAY_GCS_ROCKSDB_WRITE_BUFFER_SIZE_MB` | `16` | Per-column-family write buffer size in MB. Increase for high metadata churn. Range: 4–64. |
| `RAY_GCS_ROCKSDB_MAX_BACKGROUND_JOBS` | `2` | Max compaction/flush threads. Increase to 4 on head nodes with spare CPU if compaction falls behind. Range: 1–8. |
| `RAY_GCS_ROCKSDB_COMPRESSION` | `1` | LZ4 compression on L2+ levels. Set `0` to disable — saves CPU at cost of slightly more disk. |
| `RAY_GCS_ROCKSDB_WAL_DIR` | *(same as storage path)* | Separate directory for WAL files. Useful if main storage is slow but a faster disk is available for WAL. |

##### What Is NOT Configurable (and Why)

The following are intentionally hardcoded because incorrect values can break correctness or crash safety:

- **`create_if_missing` / `create_missing_column_families`** — Must be `true`. GCS creates the DB on first start, and new Ray versions may add column families. Exposing these invites misconfiguration that prevents startup or breaks upgrades.
- **Compaction style and level sizing** — Level compaction with dynamic level bytes is the only correct choice for this workload. Universal compaction increases space amplification; FIFO is for TTL workloads. `target_file_size_base` and `max_bytes_for_level_base` are coupled — changing one without the other causes pathological compaction.
- **Bloom filter bits and block size** — 10 bits/key and 16 KB blocks are tuned for point lookups on metadata. These have subtle interactions with compression ratios and cache efficiency.
- **`prefix_extractor`** — RocksDB warns this cannot be changed after creation. GCS uses full-key lookups, not prefix scans on key substrings. Incorrect settings are unrecoverable.
- **`merge_operator`** — Custom operator for `AsyncGetNextJobID` atomic increment. Changing it corrupts the job ID sequence.
- **Direct I/O** — No durability benefit (doesn't apply to WAL) and can break on Kubernetes PV implementations.

#### Recovery Flow

```mermaid
flowchart TD
    A[Head pod crashes] --> B[KubeRay detects pod<br/>failure, restarts<br/>head pod]
    B --> C[PVC with RocksDB<br/>data is re-mounted<br/>on new head pod]
    C --> D["GCS process starts,<br/>opens RocksDB at<br/>RAY_GCS_STORAGE_PATH"]
    D --> E["GCS reads all<br/>state from local<br/>RocksDB<br/>(fast — no network)"]
    E --> F[Workers reconnect]
    F --> G[Cluster resumes<br/>operation]
```

#### Performance Characteristics (Expected)

| Operation | Redis (network) | RocksDB (local SSD) |
|-----------|----------------|---------------------|
| Write latency | 0.5–2ms (network RTT) | 0.01–0.1ms (local I/O) |
| Read latency | 0.5–2ms | 0.01–0.05ms |
| Recovery read (full state) | Bound by network bandwidth | Bound by disk bandwidth (faster) |
| Failure mode | Network partition, Redis crash, topology change | Disk failure (rare with cloud PVs) |

#### What This Does NOT Provide

This proposal provides **persistence**, not **high availability**. The distinction:

- **Persistence**: GCS state survives head pod restart. There is a recovery window (seconds to minutes) during which the cluster is unavailable.
- **High Availability**: Zero-downtime failover with a standby GCS. This requires active/standby or Raft replication — a separate, larger effort.

The recovery model is identical to today's Redis-based FT: GCS goes down, restarts, reads state, workers reconnect. The difference is purely in where state is persisted — local disk instead of a remote Redis instance.

### Ray Core Changes

All changes are in `src/ray/gcs/store_client/`:

1. **New file: `rocksdb_store_client.h/.cc`**
   - Implements `StoreClient` interface
   - Opens RocksDB at configured path with one column family per GCS table
   - Synchronous RocksDB calls wrapped in async callbacks (RocksDB local ops are fast enough that blocking is acceptable)

2. **Modified: `store_client_factory` (or equivalent initialization code)**
   - Read `RAY_GCS_STORAGE` env var
   - Create `RocksDbStoreClient` when value is `rocksdb`
   - Preserve existing behavior for `redis` and default (in-memory)

3. **Build system: `BUILD.bazel`**
   - Add RocksDB as a new third-party dependency. RocksDB is available in the [Bazel Central Registry](https://registry.bazel.build/modules/rocksdb) (`bazel_dep(name = "rocksdb", version = "9.11.2")`) and is widely used in C++ infrastructure projects (CockroachDB, TiKV, Kafka Streams).
   - **Note:** Ray does not currently depend on RocksDB. This is a new dependency.

### KubeRay Changes

1. **RayCluster CRD extension**

   ```yaml
   spec:
     gcsFaultTolerance:
       # Existing: Redis-based
       # redisAddress: "redis:6379"

       # New: Embedded storage
       backend: rocksdb
       storage:
         size: 10Gi
         storageClassName: ssd   # optional, uses default SC if omitted
   ```

2. **Operator reconciliation**
   - When `backend: rocksdb` is set, create a PVC and mount it on the head pod at the configured path
   - Set `RAY_GCS_STORAGE=rocksdb` and `RAY_GCS_STORAGE_PATH` env vars on the head container
   - Ensure PVC lifecycle is tied to the RayCluster (deleted when cluster is deleted, persists across head pod restarts)

3. **Head pod restart behavior**
   - KubeRay already restarts the head pod on failure
   - The PVC survives pod deletion/restart by design
   - No special handling needed beyond ensuring the PVC is mounted

## Compatibility, Deprecation, and Migration Plan

- **Fully backward compatible.** Redis-based FT remains the default when `RAY_REDIS_ADDRESS` is set. No existing behavior changes.
- **New opt-in feature.** Users explicitly choose embedded storage via `RAY_GCS_STORAGE=rocksdb`.
- **No migration path needed.** This is a new alternative, not a replacement. Users can switch between Redis and RocksDB by changing configuration.

## Test Plan and Acceptance Criteria

### Unit Tests

- `RocksDbStoreClient` passes all existing `StoreClient` test cases (the interface is well-tested via `InMemoryStoreClient` and `RedisStoreClient` tests)
- Column family creation, put/get/delete, batch operations, key prefix scanning, atomic job ID increment

### Integration Tests

- GCS starts with RocksDB backend, cluster forms normally
- GCS crash + restart recovers full state from RocksDB
- Workers reconnect successfully after head restart
- Actor state, placement groups, and job history survive recovery
- Concurrent writes during normal operation don't corrupt state

### KubeRay E2E Tests

- RayCluster with `gcsFaultTolerance.backend: rocksdb` creates PVC and mounts it
- Head pod deletion + restart recovers cluster state
- RayJob completes successfully after head pod restart mid-job
- PVC is cleaned up when RayCluster is deleted

### Performance Tests

- Write throughput comparison: RocksDB vs Redis vs InMemory
- Recovery time comparison: RocksDB vs Redis (expected: RocksDB faster due to local I/O)
- Steady-state overhead: memory and CPU impact of RocksDB on head node

### Acceptance Criteria

- All existing GCS FT tests pass when run against RocksDB backend
- Recovery time is equal to or better than Redis-based FT
- No measurable regression in GCS write latency during normal operation
- Documentation and examples for both bare-metal and KubeRay deployment

## Embedded Storage Backend Selection

Ray does not currently depend on any embedded database. Adding one is a new dependency, so we evaluated candidates systematically against the `StoreClient` requirements: table-oriented key-value model, prefix scans, atomic increment, crash recovery, and single-process access on Linux with persistent volumes.

### Candidates Evaluated

| Criteria | RocksDB | SQLite | LMDB | LevelDB | BerkeleyDB | UnQLite | Speedb |
|---|---|---|---|---|---|---|---|
| **License** | Apache 2.0 | Public Domain | OpenLDAP (BSD-like) | BSD-3 | **AGPL v3** | BSD-2 | Apache 2.0 |
| **Compatible with Ray?** | Yes | Yes | Yes | Yes | **No** | Yes | Yes |
| **Language** | C++ | C | C | C++ | C | C | C++ |
| **Write perf (local SSD)** | Excellent (100-400K ops/s) | Good (50-100K ops/s) | Moderate | Good | N/A | Adequate | Good |
| **Read perf** | Good | Excellent | Excellent (zero-copy) | Good | N/A | Good | Good |
| **Crash safety** | Strong (WAL) | Strong (WAL) | Excellent (copy-on-write) | Adequate | N/A | Questionable | Strong |
| **Memory (10-100MB data)** | 20-50MB | 5-20MB | ~data size | 10-30MB | N/A | Small | 20-50MB |
| **Active maintenance?** | Yes (v10.10.1, Feb 2026) | Yes (daily commits) | Yes | **No** (maintenance-only since 2021) | No (last release 2020) | Minimal | **Dead** (website down, no release since Jan 2024) |
| **Bazel integration** | BCR native | Trivial (1 file) | Custom BUILD | Custom BUILD | N/A | Trivial | N/A |
| **Column family / table support** | Native column families | Tables via SQL | Named databases | No (prefix only) | N/A | No | Native |
| **Atomic increment** | Merge operators | SQL UPDATE | Manual txn | No | N/A | No | Merge operators |
| **Major production users** | Meta, CockroachDB, TiKV, LinkedIn | Everywhere | OpenLDAP, Monero | Chrome (legacy) | Legacy only | Niche | N/A |

### Disqualified Candidates

- **BerkeleyDB** — AGPL v3 license is incompatible with Ray's Apache 2.0 license.
- **LevelDB** — Maintenance-only since 2021. Google's README explicitly states it is "receiving very limited maintenance." RocksDB (forked from LevelDB) is strictly superior in every dimension.
- **Speedb** — RocksDB fork by a startup that appears defunct. Website unreachable, no releases in 14+ months. High abandonment risk.
- **UnQLite** — History of data corruption bugs. 157 total commits, sparse releases. Too risky for infrastructure.
- **DuckDB** — Columnar OLAP engine, architecturally wrong for point-lookup KV workloads. Not evaluated further.

### Top Three Candidates

#### RocksDB — Recommended

RocksDB is the best fit for the `StoreClient` interface:

- **Column families** map 1:1 to GCS tables (ACTOR, NODE, JOB, etc.), providing clean isolation without key-prefix hacks.
- **Merge operators** implement atomic `GetNextJobID` (equivalent to Redis `INCR`).
- **WriteBatch** maps to `AsyncBatchDelete`.
- **Prefix iterators** map to `AsyncGetKeys(table, prefix)`.
- **Bazel Central Registry** availability (`bazel_dep(name = "rocksdb")`) makes build integration straightforward.
- **Production proven** at the exact scale and use case: Meta (social graph), CockroachDB/TiKV (distributed KV), Kafka Streams (state stores).
- **Apache 2.0 license** — no friction with Ray.

Downsides: ~20-50MB memory overhead (acceptable for head node), background compaction threads consume some CPU, many tuning knobs (sensible defaults exist). Requires C++20 / GCC >= 11.

#### SQLite — Runner-up

SQLite is the simplest option with the smallest dependency footprint (single C file, public domain). It has the strongest crash-safety track record of any database and the smallest memory footprint (~5-20MB).

However, the SQL layer adds unnecessary overhead for pure KV operations. Each `StoreClient` call would go through SQL parsing and query planning. There are no native column families — tables must be created via DDL. The relational model is overkill for what is fundamentally a KV store. If minimal dependency surface were the primary concern, SQLite with a thin KV wrapper would be viable.

#### LMDB — Honorable Mention

LMDB's copy-on-write B-tree provides the strongest crash safety guarantee: the on-disk structure is always valid, no WAL needed. Zero-copy reads are the fastest of all candidates. Named databases map to GCS tables.

Downsides: requires setting `mapsize` upfront, smaller ecosystem than RocksDB/SQLite, no Bazel BCR presence, architecture-dependent format. Single-writer model is fine for GCS but limits future flexibility.

### Decision

We recommend **RocksDB** based on:

1. Best API fit — every `StoreClient` method has a natural, efficient RocksDB counterpart
2. Bazel-native integration via BCR
3. Largest production ecosystem and active maintenance
4. Apache 2.0 license compatibility

## Follow-on Work

1. **[potential] Active/Standby GCS** — A hot standby GCS process that takes over on failure, using RocksDB on a shared ReadWriteMany volume or replicated via WAL shipping. This would provide true HA (zero-downtime failover). The embedded storage backend proposed here is a prerequisite.

2. **[potential] SQLite backend** — SQLite as a lighter alternative for smaller clusters where minimal dependency footprint is preferred over KV-optimized performance (see alternatives analysis above).

3. **[potential] State compaction and TTL** — Automatic cleanup of stale entries (completed jobs, dead actors) to bound RocksDB size.

4. **[potential] Pluggable external backends** — The `StoreClient` interface can support additional external backends (etcd, DynamoDB, etc.) using the same configuration pattern. This REP establishes the multi-backend configuration surface that future backends would use.

## References

- [#53115](https://github.com/ray-project/ray/issues/53115) — Pluggable KV store for GCS backup
- [#45824](https://github.com/ray-project/ray/issues/45824) — GCS FT without external dependencies
- [#20498](https://github.com/ray-project/ray/issues/20498) — GCS High Availability RFC
- [#52480](https://github.com/ray-project/ray/issues/52480) — Worker pods restart unexpectedly with GCS FT
- [#39820](https://github.com/ray-project/ray/issues/39820) — MySQL/Couchbase GCS store (LinkedIn)
- [KubeRay #1033](https://github.com/ray-project/kuberay/issues/1033) — GCS fault tolerance umbrella
- [KubeRay #4025](https://github.com/ray-project/kuberay/issues/4025) — GCS FT for operator-managed clusters
- `src/ray/gcs/store_client/store_client.h` — StoreClient interface
- `src/ray/gcs/store_client/redis_store_client.h` — Current Redis implementation
- `src/ray/gcs/store_client/in_memory_store_client.h` — Current in-memory implementation
