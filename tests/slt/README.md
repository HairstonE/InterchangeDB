# sqllogictest corpus

SQL correctness tests as data. Each `.slt` file runs through
`tests/it/slt.rs` against a full `Session` on the default tier (memory
engine + WAL `NoSync`); one `#[test]` per file. Adding or editing a SQL
test here recompiles nothing.

The corpus has two kinds of file. Per-feature files pin one SQL surface
each (for example `group_by.slt`, `date.slt`, `outer_join.slt`,
`correlated.slt`, `create_index.slt`). The `tpch_q*.slt` files pin
TPC-H query shapes on hand-computed micro-datasets — exact answers a
person can verify, complementing the oracle validation in `bin/tpch`.

Format: sqllogictest (`statement ok` / `statement error <regex>` /
`query <types>` + `----` + expected rows). Conventions from the harness:
`NULL` for SQL NULL, `(empty)` for empty strings, decimals with an
explicit point at their stored scale, EXPLAIN output one trimmed line per
row (pins operator trees exactly).

Provenance: converted from `sql_order_by_test.rs`, `sql_aggregate_test.rs`,
`sql_join_test.rs` (unblocked by SQL CREATE INDEX — `join.slt` pins exact
INLJ plan trees over a SQL-created, backfilled index), and the
single-session half of `sql_e2e_test.rs` — assertion-for-assertion,
several strengthened (full-row expectations, exact EXPLAIN trees).
`create_index.slt` covers the CREATE INDEX feature itself, including the
backfill path.

Deliberately still Rust:
- `sql_e2e_test.rs` (slimmed): snapshot isolation and write-conflict need
  two sessions; workload-log capture asserts on the filesystem.

Later: the same corpus can run against every engine config (implement the
harness over `Database<E>` generically).

SQLite's cross-verified corpus (probed 2026-07 via the gregrahn mirror):
its very first statement — CREATE TABLE without PRIMARY KEY — misses our
dialect, which poisons every later statement on that table. The probe
originally also blamed column-list INSERT and the `INTEGER` alias, but
both were already supported. `insert.slt` proves it with select1.test's
own INSERTs verbatim (correction 2026-07). The TPC-H dialect work landed
CASE and GROUP BY, so the sole remaining blocker is PK-less tables (see
the PK deviation in `docs/plan-tpch.md`). Adopt the corpus when that
lands. Partial adoption earlier would mostly measure the dialect gap.
