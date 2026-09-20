# Documentation map

This file tells you what to read, and when. Start with the two orientation
files. Then read a reference doc only when its topic is your task.

## Start here

- [`../CONTEXT.md`](../CONTEXT.md) — **shared vocabulary.** The engines,
  policies, seams, planners, executors, and benchmark terms. Read this first.
- [`../README.md`](../README.md) — the project entry point: dev-loop
  commands, test tiers, and pre-commit gates.

## Active reference (read when…)

| Doc | Read when… |
|-----|------------|
| [`plan-tpch.md`](plan-tpch.md) | You touch TPC-H: the harness (`bin/tpch`) reference, validated results, open levers. The build history is in this file's git history. |
| [`stability.md`](stability.md) | You reason about the testing and verification strategy (deterministic concurrency, fault injection). Complements `../ISSUES.md`. |
| [`seams.md`](seams.md) | You touch a swappable boundary (engine, policy, disk) and decide between static and `dyn`. |
| [`plan-versioned-indexes.md`](plan-versioned-indexes.md) | You pick up the (unscheduled) versioned-secondary-index design. |
| [`tpch-timings.md`](tpch-timings.md) | You want query-shape timings (re-timed post-H6, 2026-09-20). Carries one open plan-shape observation. |
| [`../ISSUES.md`](../ISSUES.md) | You check or add an open quality item. This is the live tracker. |

## Component docs (co-located with what they describe)

- [`../testkit/README.md`](../testkit/README.md) — the conformance-matrix
  crate. **Read this before you write tests that touch a swappable axis.**
- [`../tests/slt/README.md`](../tests/slt/README.md) — the sqllogictest
  corpus.
- [`../fuzz/README.md`](../fuzz/README.md) — the cargo-fuzz targets.
- [`../benches/RESULTS.md`](../benches/RESULTS.md) — the bench-suite map:
  what each target measures, and how to run it.
- [`architecture.svg`](architecture.svg) — the layer diagram.
