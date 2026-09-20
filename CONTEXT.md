# CONTEXT — shared vocabulary

This file gives the terms that a new reader needs. It gives definitions only.
The "why" lives in code comments and in the docs listed in
[`docs/README.md`](docs/README.md). The sections follow the data path, from
the disk up to SQL.

## What this is

**InterchangeDB** is a from-scratch Rust database. Its subsystems are
swappable *traits*: `StorageEngine`, `EvictionPolicy`, `DiskManager`,
`Planner`, `ExecModel`, and `DataLayout`. The project exists to study how
these choices trade off. TPC-C and TPC-H benchmarks measure the trade-offs.

## Crates (workspace)

The root is both the `interchangedb` facade crate and the workspace root.

| Crate | Holds |
|-------|-------|
| `idb-core` | Contracts and primitives: the `StorageEngine` and `DiskManager` traits, `PageId`, `Error`, the `sync` shim (parking_lot ↔ shuttle). |
| `idb-storage` | Buffer pool, eviction policies, the two storage engines, page and disk formats. |
| `idb-wal` | Write-ahead log, segments, recovery, `SyncMode`. |
| `idb-txn` | Transactions, lock manager, `TransactionManager`. |
| `idb-sql` | Catalog, tables, the SQL pipeline (parse → bind → plan → execute), planners, executors. |
| `interchangedb` (root) | Facade that re-exports the crates above, plus `Database`, `Session`, and the `tpch`/`tpcc` bins. |
| `testkit` | Dev-dependency: the conformance-matrix registries, workloads, fault injection. |

**Thesis:** the `[dependencies]` of `idb-sql` name only `idb-core`
(contracts), never a storage implementation. Thus the crate boundaries
enforce the seams.

## Storage & buffer pool

- **Disk backends (2):** `MemoryDiskManager` (RAM) and `FileDiskManager`
  (disk). Both implement the `DiskManager` trait.
- **Buffer pool (`BufferPoolManager`):** a fixed pool of frames over a
  `DiskManager`. Pages are pinned, unpinned, and flushed when dirty.
- **Eviction policies (6):** `fifo`, `clock`, `lru`, `lru_k`, `two_q` (2Q),
  and `arc`. All are swappable at run time through the `EvictionPolicy`
  trait. The replacer mutex is the documented hot path on each page hit.
- **Storage engines (2):** `BTreeEngine` (B+Tree, variable-length keys and
  values, tombstone deletes) and `LsmEngine` (LSM-tree, memtable, leveled
  SSTables, bloom filters). Both implement `StorageEngine`.

## Durability & concurrency

- **WAL:** append the record → sync → apply. `SyncMode::Durable` (the
  default) does a real fsync. `NoSync` skips it, in tests only. Recovery
  replays records from the last checkpoint.
- **Transactions:** strict two-phase locking (2PL), with a lock manager that
  detects deadlocks.
- **MVCC:** snapshot isolation over versioned keys. A timestamp oracle issues
  snapshots. **GC** purges versions below the watermark.
- All `Database` methods take `&self`. Concurrent callers share an
  `Arc<Database>`.

## SQL layer

- **Pipeline:** `Session::execute` runs parse → **bind** (name and type
  resolution) → **plan** (logical → physical) → **execute**.
- **Planners (3), the `Planner` enum:** `RuleBased` (heuristic, left-deep,
  FROM order — the default), `Selinger` (System-R dynamic programming,
  cost-based on `ANALYZE` statistics), and `VolcanoMemo` (top-down memo). All three share the join-algorithm heuristic: INLJ
  when the inner side has an index on the key, HashJoin for an equi-key without an index, NLJ otherwise.
- **Execution models (2), the `ExecModel` enum:** `Volcano` (pull, iterator —
  the default) and `Push` (data-driven, with a native HashAggregate sink).
- **The matrix:** 3 planners × 2 executors = **6 configs**. Tests prove that
  all six produce *identical answers*. Only plan shape and speed vary.
- **Indexes:** primary key, plus secondary indexes (`CREATE INDEX`, with
  backfill). An equality on an indexed column lowers to an `IndexScan` or
  `PkLookup` seek. A Filter rechecks MVCC visibility, because secondary
  indexes are unversioned.
- **Row layout:** the `DataLayout` trait separates tuple encoding from the
  executors. `RowLayout` is its only implementation at this time.

## Testing

- **testkit conformance matrix** — write a contract or property *once*, then
  run it across every config with the registries: `for_each_policy!`,
  `for_each_disk!`, `for_each_engine!`, `for_each_isolation!`. A new
  implementation is one registry line, and it inherits the whole suite. See
  [`testkit/README.md`](testkit/README.md).
- **Three shapes:** *conformance* (each config meets a contract),
  *equivalence* (configs produce identical results), and *head-to-head
  bench*.
- **Test tiers:** Default (in-memory plus `NoSync`, most tests), Durability
  (real files plus fsync, for crash and recovery subjects), and the
  conformance matrix (all backends).
- **Gates (mirror CI):** `cargo test --workspace`,
  `cargo test --workspace --release`,
  `cargo clippy --workspace --all-targets -- -D warnings`, `cargo fmt --all`,
  and `cargo doc --workspace`.

## Benchmarks

- **TPC-H** — the capability ladder H1…H6 ([`docs/plan-tpch.md`](docs/plan-tpch.md)).
  Results are validated against DuckDB as the oracle, at SF 0.01. Run with
  `cargo run --release --bin tpch`.
- **TPC-C** — a throughput (tpmC) harness. Run with
  `cargo run --release --bin tpcc`.

## Conventions

- Priorities: **Safety → Performance → Developer-Experience**
- Names carry units and qualifiers last (`latency_ms_max`). Assertions guard
  the positive *and* the negative space. Loops and resources have explicit
  bounds.
- `ISSUES.md` is the live quality tracker. `docs/stability.md` gives the
  testing strategy.
