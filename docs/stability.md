# Stability & testing strategy

How InterchangeDB is verified, and why the suite is shaped the way it is.
This file describes strategy and mechanisms. Discrete open items live in
[`../ISSUES.md`](../ISSUES.md) (the quality tracker, `Q-NN` numbers), and
the conformance-matrix mechanics live in
[`../testkit/README.md`](../testkit/README.md).

The thesis: stability for an ACID engine decomposes into three concerns.

1. **Automated regression catching** — a regression is caught without a
   person remembering to look.
2. **Adversarial bug-finding** — the suite finds bugs that no example test
   was written for.
3. **Provable ACID claims** — the isolation and durability guarantees are
   *demonstrated* under adversarial conditions, not asserted.

Each concern below states its mechanism, where it lives, and what it has
found.

---

## 1. Regression catching

**CI** (`.github/workflows/ci.yml`, toolchain pinned by
`rust-toolchain.toml`): fmt check, clippy with denied warnings, tests in
debug and release, and doc build, on every push and PR. The release run
matters — races and overflow bugs surface there first.

**The conformance matrix** (`testkit`): each swappable trait's contract is
written once and runs across every implementation via the `for_each_*!`
registries. Equivalence differentials assert the interchange thesis
itself: the same workload produces identical results across all six
eviction policies, both disk managers, both engines, and all six
planner × execution-model configs. A new implementation inherits the
whole suite from one registry line.

**Goldens**: EXPLAIN plan shapes are pinned as goldens, and the
sqllogictest corpus (`tests/slt/`) pins SQL surface behavior.

## 2. Adversarial bug-finding

**Fuzzing**: the decode surfaces (key encoding, tuples, WAL records,
SSTable and manifest readers, parse → bind) run under a CI-resident
proptest suite and a coverage-guided `cargo-fuzz` scaffold (`fuzz/`,
nightly and manual). This pairing found three real bugs: a
`tuple::decode_column` cursor overrun, an `SSTableReader::open` short-file
assert, and a SQL-reachable `DECIMAL`-scale panic. All are fixed and
pinned.

**Mutation testing** (`cargo-mutants`): the meter for test quality — it
injects bugs and measures whether the suite kills them. The full campaign
closed 2026-07-17 with every surviving mutant either test-killed or
proven unkillable. Lesson recorded: scoped mutant runs create false
survivors. Run whole modules.

**Property tests**: workloads are data (`testkit::workload`), so the same
seeded op streams drive proptest differentials across the config matrix.

## 3. Provable ACID claims

**Deterministic concurrency (`shuttle`)**: a perf-preserving shim
(`idb-core`'s `sync` module) keeps `parking_lot` in production and swaps
in shuttle's instrumented primitives under `--features shuttle`. The
buffer-pool models deterministically reproduced two eviction races that
had defeated inspection (Q-30, Q-35), drove the fixes, and now stand as
permanent regression guards (`tests/bpm_swap_shuttle.rs` runs a 3-writer
storm across all six policies).

**Deterministic simulation of crashes (DST)**: the WAL format is
self-describing (length prefix plus CRC per record), so a crash at LSN k
is a WAL truncated at a record boundary — no fault injector needed. The
recovery sweep (`tests/dst_recovery_test.rs`) crashes at *every* LSN of a
seeded workload, recovers, and checks an oracle at each boundary.
Invariants asserted after every recovery:

- Every committed write is present. No aborted write is visible.
- No torn page passes checksum validation undetected.
- Recovery is idempotent — recover twice, get identical state.

Data-page faults ride `FaultInjectionDiskManager` (`testkit/src/faults.rs`):
scheduled I/O errors and torn node-page writes, asserted to surface as
detected corruption, never as silent acceptance. The durability sweep is
parametric over both engines — LSM recovery (manifest and SSTables) is a
genuinely different implementation of the same crash contract.

**Isolation proof**: MVCC isolation sits behind the `IsolationPolicy`
trait (`SnapshotIsolation` default, `ReadCommitted` second). The
Hermitage anomaly scenarios run as a conformance matrix per level
(`for_each_isolation!`): each level must block its required anomalies and
admit its permitted ones (both block dirty write. SI blocks
non-repeatable read and lost update where RC admits them. Both admit
write skew). A new level is one registry line.

---

## Open extensions

Tracked in `ISSUES.md` when picked up:

- `shuttle` over the transaction/lock cone. The buffer pool is
  instrumented. The lock manager and transaction engine are not yet. The
  gnarliest remaining concurrency lives there.
- An SI history checker (Elle-style): record a concurrent history, then
  decide whether it is admissible under the claimed level. This
  generalizes Hermitage from six named anomalies to all histories, and it
  validates a future SI → SSI transition.
- SSI itself, plus the fuller Hermitage set (G1a/b/c, OTV, PMP).
- DST fault extensions: a structurally-valid single-byte corruption to
  exercise the CRC check specifically, reordered and dropped flushes, and
  multi-segment WAL truncation.

## When to add more

New testing *modalities* stopped finding new bugs — fuzzing found three,
shuttle found the eviction races, and the later DST cuts corroborated
already-fixed behavior. The suite is in the regression-prevention regime.
The rule: extend the suite when a new subsystem lands (give it the same
three-concern treatment), and let mutation kill-rate — not intuition —
decide whether more test code has value.
