# Local Build & Test

How to build, test, and check `mycelium-chat-clipper` on your machine. CI runs the same commands
on Linux and Windows on Node 22 and 24 LTS; Chromium (Chrome/Edge) >= 120 for the extension; reproducing them locally avoids a red round-trip.

## Prerequisites

- **TypeScript 5 (strict)** toolchain.
- **Build system:** WXT (Vite) for the extension; tsc --build for the core, adapters and tools packages.
- **Package manager:** pnpm (workspaces, locked).
- **Formatter / linter:** Prettier, ESLint (typescript-eslint type-aware, eslint-plugin-boundaries for the layer matrix) + tsc --noEmit --strict + commitlint.
- **Docs:** TypeDoc (for the API docs build).

## Commands

```bash
# Build
pnpm build

# Test
pnpm test:unit && pnpm test:contract

# Format check
pnpm prettier --check .

# Lint
pnpm eslint . --max-warnings 0 && pnpm tsc --noEmit

# Benchmark
pnpm vitest bench

# Cross-artifact congruence (run before drafting any PR)
python tools/consistency_lint.py
```

## Before you open a PR

1. `pnpm prettier --check .` and `pnpm eslint . --max-warnings 0 && pnpm tsc --noEmit` are clean.
2. `pnpm test:unit && pnpm test:contract` passes; new/changed behavior is covered (≥ 80% line).
3. tsc --strict (type soundness); eslint --max-warnings 0; no-network-check (static scan of the production bundle, content script included); size-limit (content script <= 300 kB); pnpm audit; CodeQL are green where applicable.
4. `python tools/consistency_lint.py` passes.
5. The relevant docs (README, ROADMAP, ADRs, patterns, changelog) are updated in the same
   PR — see [`../workflow/documentation.md`](../workflow/documentation.md).
