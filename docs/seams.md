# Interchangeable seams

This is the trait map for InterchangeDB. Each trait is a *seam* — a point
where one implementation can be swapped for another that satisfies the same
contract. This file is the living index of what is interchangeable, and of
what each seam has behind it.

Legend: `(built)` exists in code · `(planned)` designed, not yet built ·
`(stretch)` aspirational or research.

Validation status at a glance:

- **Validated (2+ impls):** `DiskManager`, `EvictionPolicy`,
  `StorageEngine`, `PlannerStrategy`, `ExecutionModel`, `StatsProvider`,
  `IsolationPolicy`, `CostModel` (one production impl, three test models).
- **Hypothesis (1 impl, needs a second):** `DataLayout` (`RowLayout` only).
- **Not yet a trait (extract it):** `CommitProtocol`, full
  `ConcurrencyControl` (beyond isolation), `QueryEngine`.

---

## 1. Disk I/O — `DiskManager`

Raw page read, write, allocate, and sync. The lowest seam, and the one
that makes deterministic simulation testing possible.

- **FileDiskManager** (built) — single file, real `fsync` per write.
- **MemoryDiskManager** (built) — in-RAM page array, for tests and
  simulation.
- **FaultInjectionDiskManager** (built, in `testkit/src/faults.rs`) — wraps
  another `DiskManager` and injects read/write/allocate errors and torn
  node-page writes. It drives the crash-recovery torture tests.
- **IoUringDiskManager** (planned) — Linux `io_uring`, real async with
  batched submission, kept behind this seam.
- **DirectIoDiskManager** (stretch) — `O_DIRECT`, bypass the OS page cache
  for predictable I/O accounting.

## 2. Buffer pool — `EvictionPolicy`

Chooses which frame to evict. The marquee *runtime*-swappable seam, with
warm state transfer on swap.

- **FIFO** (built) — evict the oldest-loaded frame. The baseline.
- **CLOCK** (built) — second-chance approximation of LRU.
- **LRU** (built) — least recently used.
- **LRU-K** (built) — evict by backward K-distance. Scan-resistant.
- **2Q** (built) — A1in/A1out/Am queues. Scan-resistant.
- **ARC** (built) — adaptive balance of recency and frequency.
- **Random / MRU** (stretch) — degenerate baselines for comparison work.
- **LIRS** (stretch) — low inter-reference recency set.

## 3. Storage engine — `StorageEngine`

Key/value get, put, delete, and scan. Compile-time swap (generic
`E: StorageEngine`).

- **BTreeEngine** (built) — B+Tree over the buffer pool. Read-optimized.
- **LsmEngine** (built) — LSM-tree. Write-optimized, bypasses the buffer
  pool.
- **InMemoryEngine** (planned) — skiplist or hashmap, no pages, no buffer
  pool. The first rung of the in-memory speed ladder.
- **FractalTreeEngine / Bε-tree** (stretch) — buffered, write-optimized
  B-tree.
- **ProllyTreeEngine** (stretch) — content-addressed, history-independent
  B-tree. Sequenced after the Fractal Tree.

## 4. Secondary indexing — `IndexBackend`

Non-primary-key access paths. Not a trait: an enum in `idb-core`
(`common/ids.rs`) that names which engine backs an index.

- **BTree** (built) — ordered secondary index.
- **Lsm** (built) — LSM-backed secondary index.
- **Hash** (planned) — point-lookup only, no range support.

Secondary indexes are unversioned. Reads recheck MVCC visibility. The
versioned-index design is [`plan-versioned-indexes.md`](plan-versioned-indexes.md).

## 5. Isolation — `IsolationPolicy`

How a transaction's snapshot admits or blocks anomalies. Extracted as a
trait in `idb-txn` (`txn/isolation/`). The Hermitage anomaly scenarios run
per level through testkit's `for_each_isolation!` matrix.

- **SnapshotIsolation** (built, default) — write skew permitted by design.
- **ReadCommitted** (built) — the comparison level. Admits
  non-repeatable-read and lost-update where SI blocks them.
- **SSI** (planned) — rw-antidependency cycle detection. Closes the gap to
  full serializability.

A broader `ConcurrencyControl` seam (replace MVCC itself with OCC,
lock-only 2PL, or H-Store-style partitioned serial execution) is **not yet
a trait** — MVCC is hard-wired.

## 6. Commit / durability — `CommitProtocol`

How durability is achieved at commit time. **Not yet a trait** — the
policy is fused into the WAL. Extract it.

- **Group commit** (built, inside the WAL) — batched fsync across
  overlapping commits.
- **EpochCommit** (planned) — Silo-style. Persist one epoch at a time and
  amortize fsync off the critical path.
- **AsyncCommit** (stretch) — acknowledge before durability, with a
  bounded loss window. For benchmarking the durability/throughput trade.

## 7. Query planner — `PlannerStrategy`

Logical → physical plan selection. The trait exists. Dispatch is via the
`Planner` enum on the `Session` (runtime-swappable), not `dyn` — an open
set can move to `dyn` later if needed.

- **RuleBased** (built, default) — heuristic rewrites, no cost.
- **Selinger** (built) — System-R dynamic-programming join ordering,
  cost-driven on `ANALYZE` statistics.
- **VolcanoMemo** (built) — top-down memo search.

All three produce identical answers on the proven query corpus. Only plan
shape and speed vary.

## 8. Cost model — `CostModel`

Estimates plan cost for the cost-based planners. A trait, consumed
generically (`SelingerPlanner<C>`, `VolcanoPlanner<C>`).

- **DefaultCostModel** (built) — cardinality and I/O cost formulas. The
  production impl.
- **CountingCostModel / HashHostileModel / MergeFriendlyModel** (built,
  test) — instrumented and adversarial models that prove the seam and pin
  planner behavior.
- **Calibrated cost** (planned) — coefficients tied to measured per-engine
  profiles.
- **Learned cost** (stretch) — trained on the workload log.

## 9. Statistics — `StatsProvider`

Feeds the cost model.

- **CatalogStatsProvider** (built) — reads per-table and per-column
  statistics from the catalog. The production impl.
- **MockStatsProvider** (built, test) — fixed guesses. The test double
  that proves the seam.
- **HistogramStats** (planned) — richer per-column histograms.
- **SamplingStats** (stretch) — runtime sampling and sketches.

## 10. Execution model — `ExecutionModel`

How operators produce rows. A trait, selected at run time via the
`ExecModel` enum on the `Session`.

- **Volcano** (built, default) — row-at-a-time pull (`next()`).
- **Push** (built) — data-driven. Native push sinks, including grouped
  HashAggregate.
- **Vectorized** (planned) — batch-at-a-time. The OLAP path.
- **Compiled** (stretch) — codegen per query plan.

## 11. Row layout — `DataLayout`

How tuples are encoded on a page. A trait in `idb-sql` (`layout/`), with
one implementation — a hypothesis until a second rides it.

- **RowLayout** (built) — row-major encoding. The production impl.
- **ColumnLayout** (planned) — column-major. The seam's validation target.

## 12. Whole query engine — `QueryEngine`

The coarsest seam: SQL string → result set. **Not yet a trait.**

- **NativeEngine** (built) — `parse → bind → plan → execute`.
- **DataFusionEngine** (stretch) — mount IDB storage via `TableProvider`.
  An external industrial-strength oracle and columnar baseline.

---

## Next extractions (dependency-aware)

1. `CommitProtocol` out of the WAL (unblocks `EpochCommit`).
2. A second `DataLayout` impl (`ColumnLayout`) — validates the one
   unproven seam.
3. `SSI` behind `IsolationPolicy` (the matrix already has its slot).
4. Full `ConcurrencyControl` extraction (unblocks OCC and partitioned
   serial).
5. `InMemoryEngine` behind `StorageEngine`.
6. `QueryEngine` extraction, then DataFusion.

An extraction is only "done" when a second implementation rides the same
trait and a differential harness proves they agree.
