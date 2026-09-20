# TPC-H capability — results and open levers

**Status: COMPLETE (2026-07-22).** All 22 TPC-H queries run and produce
oracle-matching answers in all six planner × execution-model configs. The
`--validate` sweep passes 132/132 cells against DuckDB v1.3.0 at SF 0.01.
Zero answer mismatches were ever observed against the oracle. The build
history (phases H1–H6, with per-phase execution notes) lives in the git
history of this file.

**Goal (met):** execute all 22 TPC-H queries correctly on generated data at
small scale factors, through every planner and both execution models.
**Non-goals:** competitive analytical performance, subquery decorrelation,
sort-based aggregation, and PK-less tables.

## The capability ladder (all landed)

| Phase | Added |
|-------|-------|
| H1 | Grouped aggregation: GROUP BY and HAVING. |
| H2 | Expressions in aggregate arguments (H2a) and computed projections (H2b). |
| H3 | The scalar and predicate surface: true Decimal division, DATE + INTERVAL + EXTRACT(YEAR), BETWEEN / IN / LIKE / CASE / IS NULL, and LEFT OUTER JOIN (H3b). |
| H4 | Derived tables (H4a), uncorrelated subqueries (H4b), correlated subqueries as per-outer-row apply (H4c), SUBSTRING and subqueries inside derived tables (H4d). |
| H5 | The harness: seeded generator, committed query assets, DuckDB oracle, the `--validate` sweep. |
| H6 | Index seeks for correlated inners: secondary indexes on the correlation keys, plus conjunctive index lowering. This made Q21 feasible. |

## The harness (`bin/tpch`)

- A seeded, deterministic generator (SplitMix64, spec-shaped cardinalities,
  no new dependencies). `--csv-out` feeds the oracle.
- The 22 queries are committed assets in `queries/tpch/*.sql`. Some files
  carry commented, identity-preserving accommodations: FROM-order rewrites
  for textual lowering, Q19 OR-conjunct factoring, ORDER BY-alias rewrites,
  and Q15's view as a derived table.
- A DuckDB oracle script writes the committed `expected/*.csv` answers. The
  oracle is a one-time offline tool, never a dependency.
- `--validate` sweeps 22 queries × (rule-based | selinger | memo) ×
  (volcano | push), with per-cell progress lines, a 120 s feasibility guard,
  `--queries` / `--configs` / `--timeout-secs` filters, and TPC-H §2 decimal
  tolerance.
- After load, the harness creates secondary indexes on the correlation keys
  (`lineitem(l_orderkey)`, `lineitem(l_partkey)`, `orders(o_custkey)`) and
  opens the index engines in memory. The file-backed opener stalled the
  sweep in uninterruptible I/O wait.

## Results 
H6's measurement overturned its own spec. Per-row plan cost was 0.03–0.04%
of the correlated-apply cost — the bottleneck was the full inner-table scan
per outer row. Index seeks removed it: Q21 ∞ → 9.8–18.6 s, Q4 ~100 s →
0.1 s, Q20 27.4 s → 0.1 s, Q17 9 s → 0.5 s, Q22 7.6 s → 0.2 s.

With the scan cost gone, the planner and execution-model signal is visible:
cost-based planners beat rule-based on Q21 (~10 s vs 18.6 s) and Q5 (0.2 s
vs 2.2 s). Push beats Volcano on the correlated queries (~2.4× on Q4).

Open quirk: memo's Q12 runs in ≈6.9 s vs Selinger's 0.2 s. 

## Deviations from industry practice 
Three conscious deviations, each with a named trigger. Revisit when the
trigger fires, not before:

1. **Query-block logical IR** (flat `Select`, the Postgres/SQLite school)
   instead of an algebra tree (the Calcite/DuckDB/Cascades school).
   Trigger: decorrelation work that fights the flat block.
2. **PK required, no synthesized hidden key.** Industry synthesizes one
   (InnoDB `GEN_CLUST_INDEX`, CockroachDB `rowid`, SQLite rowid). This is a
   real dialect gap, not a design position. Trigger: SQLite-corpus
   adoption — it is the corpus's last blocker.
3. **Max-groups assert, no spill.** The production standard is to degrade
   (spill, or a sort-based fallback), never to crash. Trigger: the bound
   fires on a legitimate workload. The standard middle step is sort-based
   grouping. True spill is warehouse-scale scope.

## Open levers, recorded

Planner and optimizer:

- Decorrelation (semi-join rewrite) — better asymptotics than index-NL
  apply at large outer cardinality.
- Semi/outer join edges in the Selinger DP and the memo (today both bail
  to textual position or rule-based shape on non-inner joins).
- Q21 inner-plan caching — only if Q21 ever needs less than ~8.5 s
  (74,776 per-row plans cost 1.66 s of its 10 s).
- Sort-based grouped aggregation (exploits the 17-B order properties).
  GROUP BY pushdown.
- Expression GROUP BY keys (`EXTRACT` grouping) — unlocks flattened
  rewrites only. No verbatim query needs it.
- Pushdown and reordering into derived-table subplans. Derived-table
  cardinality estimation (today the un-ANALYZEd default).
- IN-subquery selectivity (a documented constant). IS NULL true
  null-fraction selectivity.
- The memo Q12 cost artifact (above).

SQL surface:

- CREATE VIEW (Q15 runs as a derived table).
- ORDER BY output aliases (Q13 and Q21 use equivalent forms).
- Index paths on the right leaf of an outer join.
- Multi-source runtime subqueries across the derived-table boundary.
- Hidden rowid / PK-less tables (SQLite-corpus concern only).

Engine and harness:

- Byte-aware B-tree leaf split — entry-count calibration overflows 4 KB
  pages on wide rows. The harness works around it with `with_sizes`.
- Preemptable sweep workers — a guard-tripped cell burns a core until the
  sweep ends (the cause of three false Q4 timeouts, since exonerated).
- LIKE per-row `Vec<char>` allocation (an ASCII fast path).
- TPC-H performance work proper, which waits on a Linux/NVMe environment.
