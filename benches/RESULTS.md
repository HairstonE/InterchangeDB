# Benchmarks

The bench suite, what each target measures, and how to run it. Numbers are
machine-relative — criterion writes its reports to `target/criterion/`,
and the TPC harnesses print their own tables. Historical hand-recorded
numbers live in this file's git history.

Run everything with `cargo bench`, or one target with
`cargo bench --bench <name>`.

## Criterion harnesses

| Target | Measures |
|--------|----------|
| `engine_bench` | Storage-engine microbenchmarks: B+Tree insert/lookup/mixed (`btree_bench.rs`) and LSM equivalents (`lsm_bench.rs`). |
| `sql_bench` | The SQL layer: the config-matrix head-to-head (`config_matrix.rs` — same testkit workload across every registry config), push vs volcano execution (`push_vs_volcano.rs`), and transaction throughput (`txn_bench.rs`). |
| `eviction_policies` | The six eviction policies across workload patterns. `cargo bench -- eviction` for throughput, `cargo bench -- summary` for hit rates. |
| `bpm_bench` | Buffer-pool page-access throughput under eviction pressure. |

## Custom-main experiments (not criterion)

The seven `engine_*` targets are one-shot experiments with their own
`main`: `engine_crossover`, `engine_steady_state`, `engine_range_scan`,
`engine_amplification`, `engine_space_amplification`,
`engine_zipfian_updates`, `engine_ycsb`. Each prints its own report.
They compare the B-tree and LSM engines head-to-head on a specific
workload shape (crossover points, steady state, range scans, write and
space amplification, Zipfian updates, YCSB mixes).

## Macro-benchmarks (bins, not `cargo bench`)

- **TPC-C** — `cargo run --release --bin tpcc` (throughput, tpmC).
- **TPC-H** — `cargo run --release --bin tpch -- --time` (query timings;
  see [`docs/tpch-timings.md`](../docs/tpch-timings.md)).

## Notes

- File-backed benches pay a real fsync per write. 
- The config-matrix bench consumes the same `testkit` registries as the
  conformance tests — a new engine or policy gets a bench slot from the
  same registry line (see [`testkit/README.md`](../testkit/README.md)).
