# InterchangeDB

<img src="docs/architecture.svg" width="100%">

InterchangeDB is a relational database written in Rust from scratch. Each
subsystem is a swappable trait: the storage engine, the eviction policy, the
disk backend, the planner, and the execution model. The project exists to
study how these choices trade off. TPC-C and TPC-H benchmarks measure them.

**Docs:** start at [`docs/README.md`](docs/README.md) (the documentation map)
and [`CONTEXT.md`](CONTEXT.md) (the domain glossary).

## Highlights

- **Two storage engines** — B+Tree and LSM — behind one `StorageEngine`
  trait.
- **Six eviction policies** (FIFO, CLOCK, LRU, LRU-K, 2Q, ARC), swappable
  at run time with warm state transfer.
- **A SQL layer** with three planners (rule-based, Selinger, memo) and two
  execution models (pull and push). All six combinations are proven to
  produce identical answers.
- **ACID**: a write-ahead log with group commit, MVCC snapshot isolation
  behind an `IsolationPolicy` trait, strict 2PL with deadlock detection,
  and GC.
- **TPC-H**: all 22 queries validated against DuckDB as an oracle —
  132/132 planner × execution-model cells pass. **TPC-C**: a tpmC
  throughput harness.
- **A conformance matrix** ([`testkit`](testkit/README.md)): each trait's
  contract is written once and runs against every implementation. Crash
  recovery is tested at every WAL position. Concurrency races are
  model-checked with shuttle.

## Quick start

InterchangeDB is a library. Open a database, then speak SQL through a
`Session`:

```rust
use std::{path::Path, sync::Arc};
use interchangedb::{
    buffer::BufferPoolManager, catalog::Catalog, database::Database,
    engines::btree::BTreeEngine, session::{QueryResult, Session},
    storage::FileDiskManager,
};

// Set up: pages on disk → buffer pool → B+Tree engine → database + WAL.
let disk = FileDiskManager::create("data/db.pages")?;
let engine = BTreeEngine::new(BufferPoolManager::new(512, disk))?;
let database = Arc::new(Database::open(Path::new("data"), engine)?);
let catalog = Arc::new(Catalog::open(database.engine_arc().clone())?);
let mut session = Session::new(database, catalog);

// Make a table and query it.
session.execute("CREATE TABLE users (id INT PRIMARY KEY, name VARCHAR(20))")?;
session.execute("INSERT INTO users VALUES (1, 'ada'), (2, 'grace')")?;
if let QueryResult::Rows { rows, .. } = session.execute("SELECT * FROM users WHERE id = 1")? {
    println!("{rows:?}");
}
```

Run the suite and the benchmark harnesses:

```bash
cargo test --workspace                      # the full suite
cargo run --release --bin tpch -- --time    # TPC-H: 22 queries, timed
cargo run --release --bin tpcc              # TPC-C throughput
```

## Development

Inner-loop commands:

```bash
cargo check --lib      # fastest signal on src/ edits
cargo test --lib       # unit tests only, no integration binaries
cargo clippy --lib     # lint without building benches
cargo test --test <harness> <filter>   # one integration suite (harnesses: it, stress)
```

Run the pre-commit gates before you push, not in the tight loop. CI mirrors
them:

```bash
cargo test --workspace
cargo test --workspace --release
cargo clippy --workspace --all-targets -- -D warnings
```

### Test tiers

Tests run in three named tiers:

- **Default tier** — in-memory backends (`MemoryDiskManager`, WAL
  `SyncMode::NoSync`). These suites do not test the disk, so they are fast on
  all machines. New tests go here unless the disk is their subject.
- **Durability tier** — real files and real fsync (`SyncMode::Durable`). A
  suite goes here when its subject is crash durability, recovery, persistence,
  or fsync semantics. A test with reopen or crash semantics must live here.
- **Conformance matrix** — the testkit `for_each_*!` registries run every
  backend, `FileDiskManager` included, against the shared contracts. Because
  of this, the in-memory default tier loses no coverage.
