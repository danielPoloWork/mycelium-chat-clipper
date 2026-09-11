# Benchmarks

Reproducible performance measurements for `mycelium-chat-clipper`. Any performance claim in the
spec, README, or a PR must be backed by a benchmark here and by code under
`src/bench/typescript/it/mycelium-labs/chatclipper/`. Numbers without a reproducible method
are not evidence.

## Methodology

- **Harness:** `WXT (Vite) for the extension; tsc --build for the core, adapters and tools packages` builds the bench target; run with `pnpm vitest bench`.
- **Environment:** record the machine (CPU, RAM, OS), the toolchain version, and the build
  configuration (release/optimized) with every result — a number without its environment is
  not comparable.
- **Discipline:** warm up, run multiple iterations, report a central tendency **and** spread
  (e.g. median + p99), and pin the commit SHA the run was taken at.
- **Regression gate:** the CI `benchmark` job runs the suite; a result is a regression only
  against a recorded baseline on comparable hardware (note when CI hardware is too noisy to
  gate and the run is informational).

## Results

One report per measured scenario, from [`template.md`](template.md). Keep the index newest-first.

| Date | Scenario | Version | Headline result | Report |
|------|----------|---------|-----------------|--------|
| —    | —        | —       | —               | —      |

_No benchmarks recorded yet._
