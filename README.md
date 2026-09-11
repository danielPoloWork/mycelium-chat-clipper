# mycelium-chat-clipper

> Local-first MV3 browser extension that exports the open ChatGPT, Claude, or Gemini conversation as a self-contained, write-once JSON bundle

![Status](https://img.shields.io/badge/Status-v0.0.0-blue)

A
app written in **TypeScript 5 (strict)**, built and governed to an enterprise quality
bar: full CI matrix, static analysis, sanitizers, documented design decisions, and SemVer
releases.

## What it is

Knowledge workers hold long, valuable conversations with ChatGPT, Claude and Gemini that stay
locked in each provider's web UI: no complete local copy, no provider-independent format, and
no safe hand-off to the owner's own tooling. Mycelium Chat Clipper is a Chromium Manifest V3
extension (Chrome/Edge >= 120) that captures the conversation open in the active tab from the
page DOM — all of it, including virtualized history and collapsed content — normalizes it into
a provider-independent canonical JSON, and writes a self-contained, write-once export bundle
(raw.json, capture.json, manifest.json) into an inbox folder that an external ingestion module
(the owner's agentic OS) consumes. It is local-first: zero network, zero telemetry, zero private
provider endpoints, zero budget (OSS toolchain, GitHub Actions, GitHub Releases).

The extension is an exporter, not an archive (spec Rev 3.0, ADR-021): it does not keep the
archive, reconcile captures, index, chunk, search, or produce Markdown. Those live in the
ingestion module. Provider knowledge (selectors, roles, blocks) lives only in the extension;
the ingestion module consumes only the canonical contract. The export contract (spec §6) is
the product: any change to fields, file names, layout, or completion semantics requires an
ADR and a schema_version bump.

The frozen specification is in
[`docs/specs/01_spec_chatclipper.md`](docs/specs/01_spec_chatclipper.md).

## Build, test, run

```bash
pnpm build
pnpm test:unit && pnpm test:contract
```

- **Toolchain:** WXT (Vite) for the extension; tsc --build for the core, adapters and tools packages, Vitest (unit, golden snapshots), fast-check (property-based), happy-dom (adapter DOM contract), Ajv (JSON Schema contract), Playwright (e2e against a synthetic provider) with axe-core (accessibility scan; approved by the PO on 2026-09-11 as an addition to spec §14.4), Prettier, ESLint (typescript-eslint type-aware, eslint-plugin-boundaries for the layer matrix) + tsc --noEmit --strict + commitlint.
- **Supported platforms:** Linux and Windows on Node 22 and 24 LTS; Chromium (Chrome/Edge) >= 120 for the extension.
- Consumers import the public surface via: `import { CanonicalCapture, normalize, validateInvariants } from '@mycelium-labs/chatclipper-core';`.

See [`docs/development/local-build.md`](docs/development/local-build.md) for the full local
setup.

## How this project is run

| Document | Purpose |
|---|---|
| [`AGENTS.md`](AGENTS.md) | How AI agents (and humans) work in this repo — the contract. |
| [`ROADMAP.md`](ROADMAP.md) | The numbered plan and what is done. |
| [`docs/adr/`](docs/adr/) | Why it is built the way it is (Architecture Decision Records). |
| [`docs/patterns/`](docs/patterns/) | Design patterns adopted, rejected, or considered. |
| [`docs/workflow/`](docs/workflow/) | Git, documentation, release, and maintenance conventions. |
| [`CHANGELOG.md`](CHANGELOG.md) | User-visible changes per release. |
| [`SECURITY.md`](SECURITY.md) | How to report a vulnerability. |

## Milestones

| # | Title | Status |
|---|---|---|
| 1 | Project bootstrap & CI | ⏳ in progress |
| 2 | Core and export contract (WP-01) | ⏳ planned |
| 3 | Platform spikes: FSA, downloads and the Port channel (WP-02, checkpoint CP1) | ⏳ planned |
| 4 | ChatGPT adapter and CaptureEngine: first real capture (WP-03) | ⏳ planned |
| 5 | Bundle writer: first real bundle in the inbox (WP-04, checkpoint CP2) | ⏳ planned |
| 6 | Hardening (WP-05) | ⏳ planned |
| 7 | Claude adapter (WP-06) | ⏳ planned |
| 8 | Gemini adapter (WP-07) | ⏳ planned |
| 9 | Production hardening: privacy, security, accessibility, release pipeline (WP-08) | ⏳ planned |
| 10 | Tail export with anchor (WP-09, Phase 3) | ⏳ planned |
| 11 | Re-normalization CLI, Native Messaging sink, Firefox (WP-10, Phase 3 — each behind an ADR) | ⏳ planned |
| 12 | Release v1.0.0 — all milestones complete, all open issues fixed | ⏳ planned |


## License

Apache-2.0 © 2026 Daniel Polo. See [`LICENSE`](LICENSE).
