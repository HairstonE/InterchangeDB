# TPC-H query timings

Informational only, not a gate. Captured 2026-09-20 (post-H6) with
`cargo run --release --bin tpch -- --time` on an idle reference dev
machine (2015 2-core MacBook, macOS), seed 19920101, scale factor 0.01,
config **rule-based / volcano**. These are correctness-driven timings on a
small machine, not a performance benchmark. Read the shape, not the third
digit.

| Query |     ms | rows | note |
|-------|-------:|-----:|------|
| Q1    |  164.2 |    4 | |
| Q2    |  172.1 |    8 | |
| Q3    |  282.9 |   10 | |
| Q4    |   92.7 |    5 | was ~100 s pre-H6 |
| Q5    | 1745.4 |    5 | slower than pre-H6 (see below) |
| Q6    |  134.7 |    1 | |
| Q7    |  449.6 |    4 | |
| Q8    |   45.8 |    2 | |
| Q9    | 5399.6 |  175 | largest join fan-out |
| Q10   | 1529.3 |   20 | slower than pre-H6 (see below) |
| Q11   |   40.0 |  160 | |
| Q12   | 1119.1 |    2 | slower than pre-H6 (see below) |
| Q13   |   81.2 |   24 | |
| Q14   |  143.7 |    1 | |
| Q15   |  263.0 |    1 | |
| Q16   |   17.7 |  268 | |
| Q17   |  164.0 |    1 | was ~9 s pre-H6 |
| Q18   | 1531.7 |    1 | slower than pre-H6 (see below) |
| Q19   |  168.6 |    1 | OR-factored (see queries/tpch/q19.sql) |
| Q20   |   97.9 |    1 | was ~27 s pre-H6 |
| Q21   | 9223.2 |    1 | did not complete pre-H6 (120 s guard) |
| Q22   |   47.6 |    7 | was ~7.6 s pre-H6 |

Two observations:

1. **H6 collapsed the correlated queries.** Q4, Q17, Q20, Q21, and Q22 —
   pre-H6 all dominated by a full inner-table scan per outer row — now run
   through index seeks on the correlation keys. Q21 completes on every
   config.
2. **Open observation, not yet investigated:** Q5, Q10, Q12, and Q18 are
   4–7× slower than the pre-H6 capture on this config, on an idle machine
   with matching baselines. The suspected cause: the H6 secondary indexes
   change join lowering — this config is cost-blind, and its join
   heuristic prefers INLJ when the inner side has an index on the key, so
   joins on `l_orderkey` / `l_partkey` / `o_custkey` may now take per-row
   seeks where a hash join was better. Confirm with EXPLAIN before and
   after index creation, and compare against the cost-based configs.

The cost-based planners beat rule-based where cost matters, and Push beats
Volcano on the correlated queries (~2.4× on Q4 in the H6 sweep). This
table is the rule-based / volcano cell only — regenerate any cell with
`--time` and the `--configs` filter.
