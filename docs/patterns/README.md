# Design Patterns Catalogue

Living index of every design pattern **adopted**, **planned**, **considered and rejected**,
or **under evaluation** for `mycelium-chat-clipper`. Mandatory reading whenever a PR introduces
or removes a pattern, and updated in the same PR.

- **Rules** — [`AGENTS.md`](../../AGENTS.md) §8.
- **Canonical taxonomy** — [`design-patterns.md`](design-patterns.md). All pattern names
  used here, in ADRs, and in commit messages must match its spelling and categorisation.

## Architecture style

**Committed style:** Hexagonal (Ports & Adapters) — from [`design-patterns.md`](design-patterns.md) §5.
**Pattern discipline:** `enforced` — `advisory` means the agent advises and the human
decides; `enforced` makes conformance to the committed style + adopted patterns a review expectation.


## How to use this catalogue

- **Adding a pattern** — when a PR lands one, add a row to *Implemented / Planned* as
  `Implemented`, with the ADR link and the code location (a real path under
  `src/main/typescript/...`); a pattern decided in an ADR but not yet in code is added as `Planned`.
- **Refining** — update the row and link the new ADR.
- **Rejecting** — add it to *Rejected* with the reason; do not silently drop it.
- **Removing** — move the row to *Superseded*, link the superseding ADR, keep the history.

Status vocabulary: `Planned` (decided in an ADR, not yet landed) · `Implemented` (present
in `src/main/...`, ADR `Accepted`) · `Considered` · `Rejected` · `Superseded`.

## Implemented / Planned

_Patterns named in the spec at intake are seeded below as **Planned**; each becomes
**Implemented** with its ADR and a real code location in the PR that introduces it._

| # | Pattern | Status | Problem it addresses | Code location | ADR / PR |
|---|---------|--------|----------------------|---------------|----------|
| — | Ports & Adapters (Hexagonal) | Planned | Domain package testable in Node; sinks, HTML parser, clock, id generator and hasher are swappable behind ports | _TBD_ | _spec (intake)_ |
| — | Provider Adapter (by composition) | Planned | One CaptureEngine consumes a ProviderAdapter per platform, so a provider DOM change touches exactly one module | _TBD_ | _spec (intake)_ |
| — | Ordered parser chain (pure functions) | Planned | Heterogeneous DOM nodes are recognized by the first competent ContentBlock parser; unrecognized residue becomes UnknownBlock with a reason | _TBD_ | _spec (intake)_ |
| — | Repository | Planned | LocalCacheRepository and PendingBundleRepository sit behind interfaces with in-memory implementations for tests | _TBD_ | _spec (intake)_ |
| — | Write sequence with completion marker (a Unit of Work form) | Planned | writeBundle() never marks a bundle complete if a step fails; retry resumes from the failed step with identical bytes | _TBD_ | _spec (intake)_ |
| — | Finite State Machine | Planned | ExportSession transitions live in a declared table; an illegal transition raises IllegalStateTransitionError, no per-state classes | _TBD_ | _spec (intake)_ |
| — | Result type | Planned | Result<T, DomainError> makes failures explicit in the type; throw only for invariant bugs and at browser boundaries | _TBD_ | _spec (intake)_ |
| — | Manual Dependency Injection | Planned | One composition root per MV3 context gives testability without singletons or a container | _TBD_ | _spec (intake)_ |


## Rejected

_No rejections recorded yet._

| # | Pattern | Considered for | Rejected because | ADR / PR |
|---|---------|----------------|------------------|----------|
| — | —       | —              | —                | —        |

## Superseded

_No superseded patterns yet._

| # | Pattern | Superseded by | When | ADR / PR |
|---|---------|---------------|------|----------|
| — | —       | —             | —    | —        |

## Candidate patterns to consider

The taxonomy in [`design-patterns.md`](design-patterns.md) lists every pattern in scope. As
the architecture takes shape, narrow that universe to the patterns plausibly applicable to
*this* artifact and list them here by category, each with a one-line "possible application".
A candidate remains a candidate until adopted (own ADR) or explicitly rejected.

## Out-of-scope categories

Record here any taxonomy category pre-classified as not applicable to this artifact (with a
one-line reason), so the policy of explicit rejection is honoured without filling the
*Rejected* table with N/A noise.
