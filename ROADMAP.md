# Roadmap — mycelium-chat-clipper

The project's plan as a numbered, checkbox-driven list. When an item completes in a PR,
flip its checkbox (`- [ ]` → `- [x]`) **in the same PR**. New work goes at the bottom of
its section with a fresh `<milestone>.<task>` number; never renumber.

- **Versioning start:** pre-1.0 milestone-driven — a +0.1 minor bump per completed milestone from v0.1.0 at M1 (M2 -> v0.2.0, ... M11 -> v0.11.0); v1.0.0 only after every milestone and open issue is fixed (M12).
- **Session journal:** see [`docs/journal/`](docs/journal/). Latest checkpoint: _none yet_.

## Model & effort routing (advisory)

An item may carry an advisory **route** — `route: <tier> / <effort>` — derived from its intake
signals through the `os/routing` policy's only-raise resolution (ADR-0017: start at the floor;
matched signals only ever raise, never lower). Tiers, cheapest → most capable: fast → standard → frontier-reasoning.
Efforts: low → medium → high → extra → max. An item with no route takes the floor (fast / low). The route
*recommends*; **the human keeps final model authority** — switch with your host's own model
control, never mid-session by the agent.

Tiers map to concrete models only through the dated catalog (as of 2026-07-27;
a stale date is the review cue):

- **claude-code**: fast → Sonnet 5 · standard → Opus 5 · frontier-reasoning → Fable 5
- **codex**: fast → GPT Luna · standard → GPT Terra · frontier-reasoning → GPT Sol
- **gemini**: fast → — · standard → — · frontier-reasoning → —
- **opencode**: fast → Sonnet 5 · standard → Opus 5 · frontier-reasoning → Fable 5

Where the EADOS core is vendored (`.eados-core/`), the authoritative per-issue call once tracker
labels exist is `python .eados-core/tools/route_advice.py --issue <N>`.

---

## Milestone 1 — Project bootstrap & CI

The thinnest slice that compiles, tests, and ships under the full quality bar.

- [ ] 1.1 Lay down the build system (WXT (Vite) for the extension; tsc --build for the core, adapters and tools packages) and a buildable skeleton under
      `src/main/typescript/it/mycelium-labs/chatclipper/`.
- [ ] 1.2 Wire the test framework (Vitest (unit, golden snapshots), fast-check (property-based), happy-dom (adapter DOM contract), Ajv (JSON Schema contract), Playwright (e2e against a synthetic provider) with axe-core (accessibility scan; approved by the PO on 2026-09-11 as an addition to spec §14.4)) with one passing smoke test under
      `src/test/typescript/it/mycelium-labs/chatclipper/`.
- [ ] 1.3 Add formatter + linter configs (Prettier, ESLint (typescript-eslint type-aware, eslint-plugin-boundaries for the layer matrix) + tsc --noEmit --strict + commitlint) at the repo root.
- [ ] 1.4 Stand up the CI matrix (Linux and Windows on Node 22 and 24 LTS; Chromium (Chrome/Edge) >= 120 for the extension) with build + test + format + lint.
- [ ] 1.5 Seed the version constant (export const VERSION = 'X.Y.Z') in `version.ts`.
- [ ] 1.6 Lay down the pnpm workspace at `src/mycelium/{core,adapters,extension,tools}` (RFC-0001 D2): `pnpm-workspace.yaml` glob `src/mycelium/*`, one `package.json` + `tsconfig.json` per package, the shared strict `tsconfig.base.json` (noUncheckedIndexedAccess, exactOptionalPropertyTypes, noImplicitOverride, verbatimModuleSyntax, ES2022 ESM); move the 1.1/1.2 skeleton out of the derived `src/main` / `src/test` tree and remove that tree; add the unanchored ignore patterns node_modules/, .output/, .wxt/, coverage/, test-results/, playwright-report/ — size: L — route: standard / high (severity:high)
- [ ] 1.7 ESLint with typescript-eslint (type-aware) and eslint-plugin-boundaries enforcing the PRD §5.1 dependency matrix; Prettier; commitlint for Conventional Commits (RFC-0001) — size: M — route: standard / medium (severity:medium)
- [ ] 1.8 Vitest `vitest.config.ts` with `test.projects` = the four packages and the NFR-010 coverage thresholds; Playwright scaffold with --load-extension and `testDir: e2e/`; the `pnpm check` aggregate script (RFC-0001) — size: M — route: standard / medium (severity:medium)
- [ ] 1.9 WXT extension skeleton: MV3 manifest per PRD §11.2 (minimal permissions, three host permissions, CSP connect-src 'none', ISOLATED world), empty Side Panel, Options page, lifecycle-only service worker, `_locales/{en,it}` base with the display name Mycelium Chat Clipper (RFC-0001) — size: M — route: frontier-reasoning / extra (security)
- [ ] 1.10 Day-zero ADRs, numbered at authoring time from ADR-0003 (RFC-0001 D3): spec 001 MV3 Chromium-first; 002 TypeScript/WXT/pnpm; 003 Ports & Adapters with lint-enforced boundaries; 004 Zod single definition with generated JSON Schema; 012 zero network permanently; 013 Apache-2.0 + GitHub Releases + Edge Add-ons; 014 Preact + signals, native CSS, _locales; 018 the eight mandatory patterns; 021 exporter-not-archiver; 022 write-once bundle with final manifest; 023 archive-stateless extension; plus RFC-0001 D1 (the `chatclipper` identifier), D2 (the `src/mycelium/` tree, SUPERSEDING the generated ADR-0002; re-point the generated consistency_lint `version_file` to src/mycelium/core/src/version.ts and the rendered AGENTS.md §5) and D4 (documents and precedence); `docs/adr/README.md` gains a plain-text Spec-mapping column — size: L — route: frontier-reasoning / extra (adr, decision-heavy)
- [ ] 1.11 CI workflow in the PRD §13.4 gate order (lint → format → typecheck → unit → contract → build → e2e → size-limit → no-network-check → audit) on the manifest's matrix, Dependabot weekly, CodeQL (RFC-0001) — size: M — route: frontier-reasoning / extra (security)
- [ ] 1.12 Root documents (RFC-0001 D4): import the PRD verbatim to `docs/prd/SPEC-LCA-001.md`; README/CONTRIBUTING note on the 160-character path budget (clone root <= ~90 characters, or core.longpaths + LongPathsEnabled); PRIVACY.md and SECURITY.md stubs; docs/TODO-HUMAN.md; docs/ARCHITECTURE.md skeleton; retire `.draft-specs/` once both moves have landed — size: S — route: fast / low
- [ ] 1.13 A `tools` unit test asserting that no tracked repo-relative path exceeds 160 characters (RFC-0001 D2) — size: XS — route: fast / low
- [ ] 1.14 Tag v0.1.0 on completion of M1 — the first release of the owner's cadence (+0.1 minor per completed milestone; v1.0.0 at M12); annotated tag, drafted GitHub Release, changelog rolled (RFC-0001) — size: XS — route: fast / low


---

## Milestone 2 — Core and export contract (WP-01)

Implements RFC-0001 D0/D1 and the API-contract fold: the domain package and the export contract exist, are fully tested in Node, and any consumer can validate a bundle with the published schemas alone. Tag v0.2.0 on completion.

- [ ] 2.1 Zod schemas for CanonicalCapture, RawCapture, Manifest and Anchor 1.0.0 with branded ids and readonly domain types; `SCHEMA_VERSION` constants; `producer.name: 'chatclipper'` and the `export-root.json` producer schema (RFC-0001 D1) — size: L — route: frontier-reasoning / high (sets-pattern, severity:high)
- [ ] 2.2 `canon()` and `Hasher` (Web Crypto SHA-256): message and conversation `content_sha256`, `dom_text_sha256`, `primary_text` per block type; property tests for idempotence, determinism and field-order independence (RFC-0001) — size: M — route: standard / high (severity:high)
- [ ] 2.3 ULID `IdGenerator` and injected `Clock`; `sanitizeUrl` (media hosts and `source_url` reduced to origin + pathname, counting redactions); `sanitizePathSegment` / `buildBundlePath` with the Windows rules and the 120-character export-path budget (RFC-0001) — size: M — route: standard / medium (severity:medium)
- [ ] 2.4 Deterministic JSON serialization (schema key order, indent 0, trailing newline, UTF-8 without BOM); `validateInvariants()`; `Result` / `DomainError` types (RFC-0001) — size: M — route: standard / high (severity:high)
- [ ] 2.5 `schema:gen` (Zod → JSON Schema draft 2020-12) and the `validate-bundle` CLI with zero extension dependencies; PRD Appendix A example bundles as self-healing fixtures with `producer.name: 'chatclipper'` (Ajv accepts them, rejects negatives, flags an altered hash) — flagged until PRD Rev 3.1 updates Appendix A (RFC-0001 D1) — size: M — route: standard / medium (severity:medium)
- [ ] 2.6 `docs/CONTRACT.md` — the export-contract extract for the ingestion module; coverage gate core >= 95% at this milestone (RFC-0001) — size: S — route: fast / low
- [ ] 2.7 The M2 ADRs, numbered at authoring (RFC-0001 D3): spec 008 ULID identity within a capture; 009 the three hashes; 011 RAW as redacted DOM fragments; 016 attachments as metadata only; 024 the file-based anchor protocol — size: S — route: frontier-reasoning / extra (adr)


---

## Milestone 3 — Platform spikes: FSA, downloads and the Port channel (WP-02, checkpoint CP1)

Implements RFC-0001 D0: time-boxed (one day) evidence on the MV3 capabilities the design depends on, recorded in docs/spikes/SPIKE-001.md, closing the provisional sink decision (spec ADR-005). Tag v0.3.0 on completion.

- [ ] 3.1 FSA picker from Options and Side Panel; handle persistence in IndexedDB across browser restarts (permission state and prompt); 'always allow' on the extension origin (RFC-0001) — size: M — route: frontier-reasoning / extra (decision-heavy)
- [ ] 3.2 Write-once semantics on NTFS and SMB shares: latency and behavior when the browser is killed mid-write (RFC-0001) — size: S — route: fast / low
- [ ] 3.3 `downloads.download` with subfolders, `onChanged` → complete, and the effect of 'Ask where to save' (RFC-0001) — size: S — route: fast / low
- [ ] 3.4 `tabs.connect` / `onDisconnect` behavior under navigation and tab reload (RFC-0001) — size: S — route: fast / low
- [ ] 3.5 Checkpoint CP1: the sink ADR (spec ADR-005, FSA vs downloads as default, Reconnect UX) and confirmation of the single Port channel; PO report (done / missing / decisions / open questions) (RFC-0001 D3) — size: S — route: frontier-reasoning / extra (adr, decision-heavy)


---

## Milestone 4 — ChatGPT adapter and CaptureEngine: first real capture (WP-03)

Implements RFC-0001 D0 and the algorithm-sketch fold: a real ChatGPT conversation of at least 100 messages is captured completely, normalized and validated in memory; the synthetic e2e shows zero gaps with every collapsed section expanded. Tag v0.4.0 on completion.

- [ ] 4.1 `ProviderAdapter` interface and `runCapture()`: per-window expansion, quiescence wait, overlap merge, downward sweep, cancel via AbortSignal, in-page shadow-DOM banner (RFC-0001) — size: L — route: frontier-reasoning / high (sets-pattern, severity:high)
- [ ] 4.2 `ChatGptAdapter`: `selectors.ts` (essential flags, stripSelectors, selectorsVersion), `probe`, `detect` with `provider_project_name`, `findCollapsed` / `expand`, `mediaHostPatterns` (RFC-0001) — size: M — route: standard / medium (severity:medium)
- [ ] 4.3 Normalizer: ordered parser chain (code, tool_call, thinking, image, file, text via Turndown + GFM), coded warnings, citations, attachment metadata, title fallback (RFC-0001) — size: L — route: frontier-reasoning / high (sets-pattern)
- [ ] 4.4 Single Port channel (`name: 'chatclipper'`) with Zod validation on both sides, chunked streaming with cooperative yield; `ExportUseCase` up to in-memory bundle assembly (RFC-0001 D1) — size: M — route: standard / medium (severity:medium)
- [ ] 4.5 Side Panel: detection card with adapter health, progress phases, final report, on-screen Markdown preview (never persisted) (RFC-0001) — size: M — route: standard / medium (severity:medium)
- [ ] 4.6 Record fixture (dev) with the anonymizer; three ChatGPT fixtures recorded by the PO; adapter contract tests with happy-dom (RFC-0001) — size: S — route: fast / low
- [ ] 4.7 Synthetic-provider e2e: 300 virtualized messages with 12 'Show more' sections inside unmounted messages → all expanded, 0 gaps; manual smoke on a real conversation >= 100 messages (RFC-0001) — size: M — route: standard / high (severity:high)
- [ ] 4.8 The M4 ADRs, numbered at authoring (RFC-0001 D3): spec 006 DOM-only, no private endpoints; 007 normalization in the content script behind `HtmlParser`; 015 active branch only; 017 fixtures + synthetic provider, no live provider in CI — size: S — route: frontier-reasoning / extra (adr)


---

## Milestone 5 — Bundle writer: first real bundle in the inbox (WP-04, checkpoint CP2)

Implements RFC-0001 D0 and the Data & schema fold: a complete, read-back-verified, validate-bundle-green bundle lands in the real export folder via FSA; the ingestion module has something to ingest. Tag v0.5.0 on completion.

- [ ] 5.1 `ExportSink` port with `FsaExportSink` (read-back verification), `DownloadsExportSink` (`onChanged` ordering, `write_verified: false`, `write_not_verified` warning) and `MemoryExportSink` (RFC-0001) — size: L — route: frontier-reasoning / high (sets-pattern, severity:high)
- [ ] 5.2 Export-root initialization: `export-root.json` (with the `chatclipper` producer), `schema/`, `README.md`; refusal of a higher layout version (`ExportRootLayoutAhead`) (RFC-0001 D1) — size: S — route: standard / medium (severity:medium)
- [ ] 5.3 `writeBundle()`: raw → capture → manifest with hash verification; `withRetry` with backoff; the local `exports` cache; `PendingWrite` parking of the serialized bundle (RFC-0001) — size: M — route: frontier-reasoning / extra (adr, severity:high)
- [ ] 5.4 User annotations (project, tags, note) remembered per conversation; Options page (folder, language) (RFC-0001) — size: S — route: fast / low
- [ ] 5.5 E2E with `MemoryExportSink`: complete bundle and `validate-bundle` green; cancel leaves no manifest; manual smoke: real bundle via FSA, validated (RFC-0001) — size: M — route: standard / high (severity:high)
- [ ] 5.6 Checkpoint CP2: has the ingestion module ingested the first real bundle? Contract corrections before the tag; adapter priority; tag v0.5.0 (RFC-0001) — size: S — route: frontier-reasoning / extra (decision-heavy)


---

## Milestone 6 — Hardening (WP-05)

Implements the RFC-0001 scalability-budgets fold: performance budgets become blocking gates and every failure path has a tested recovery. Tag v0.6.0 on completion.

- [ ] 6.1 Synthetic 1,000-message performance fixture in `src/mycelium/core/bench/`; NFR-001/002/003 measured and enforced; `size-limit` on the content script (NFR-017); results under docs/benchmarks/ (RFC-0001) — size: M — route: standard / high (severity:high)
- [ ] 6.2 `PendingWrite` and Retry write producing byte-identical files; `ContentScriptDisconnected` with an `incomplete` bundle on user confirmation (RFC-0001) — size: M — route: standard / high (severity:high)
- [ ] 6.3 Full error taxonomy surfaced in the UI with remedies (NFR-014); `downloads.onChanged` interrupted handling (RFC-0001) — size: S — route: standard / medium (severity:medium)
- [ ] 6.4 E2E: retry write, disconnection, downloads-sink ordering (RFC-0001) — size: S — route: standard / medium (severity:medium)


---

## Milestone 7 — Claude adapter (WP-06)

Implements RFC-0001 D0 for the second provider: Claude conversations are captured to the same bar as ChatGPT. Tag v0.7.0 on completion.

- [ ] 7.1 `ClaudeAdapter`: `selectors.ts`, platform-specific expansion (reasoning and artifacts), `mediaHostPatterns`, project name from the header (RFC-0001) — size: M — route: standard / medium (severity:medium)
- [ ] 7.2 Three anonymized fixtures, adapter contract tests, PRD Appendix C update (RFC-0001) — size: S — route: fast / low
- [ ] 7.3 Manual smoke on a real Claude conversation (RFC-0001) — size: XS — route: fast / low


---

## Milestone 8 — Gemini adapter (WP-07)

Implements RFC-0001 D0 for the third provider: Gemini conversations are captured to the same bar as ChatGPT. Tag v0.8.0 on completion.

- [ ] 8.1 `GeminiAdapter`: Angular elements (`user-query`, `model-response`, `message-content`), longer quiescence, 'Show reasoning' expansion, `mediaHostPatterns` (RFC-0001) — size: M — route: standard / medium (severity:medium)
- [ ] 8.2 Three anonymized fixtures, adapter contract tests, PRD Appendix C update (RFC-0001) — size: S — route: fast / low
- [ ] 8.3 Manual smoke on a real Gemini conversation (RFC-0001) — size: XS — route: fast / low


---

## Milestone 9 — Production hardening: privacy, security, accessibility, release pipeline (WP-08)

Implements the RFC-0001 cross-cutting fold: the three providers ship with the privacy, security, accessibility and diagnostic surfaces complete and the release pipeline proven. Tag v0.9.0 on completion.

- [ ] 9.1 UI i18n (`_locales/{en,it}`), accessibility (keyboard, ARIA, AA contrast) with the axe-core scan in the Playwright e2e as a blocking gate, diagnostic bundle export without content or titles (FR-016) (RFC-0001) — size: M — route: standard / medium (severity:medium)
- [ ] 9.2 `no-network-check` on the production bundle including the content script; CSP and permissions review against PRD §11.2 (RFC-0001) — size: S — route: frontier-reasoning / extra (security)
- [ ] 9.3 PRIVACY.md, SECURITY.md, enterprise-account and ToS notice in README and Options; runbook 'the provider changed its DOM'; docs/ARCHITECTURE.md complete (RFC-0001) — size: M — route: frontier-reasoning / extra (security)
- [ ] 9.4 Release pipeline: tag → `chatclipper-chromium-vX.Y.Z.zip`, `SHA256SUMS`, CycloneDX SBOM, `schema/` and `validate-bundle` attached; release smoke on all three providers; tag v0.9.0 (RFC-0001 D1) — size: M — route: standard / medium (severity:medium)


---

## Milestone 10 — Tail export with anchor (WP-09, Phase 3)

Implements RFC-0001 D0 (Phase 2 contract): when the ingestion module writes anchors, an export scans only the tail; an invalid anchor degrades to a full scan. Gated on the PO's answers to Q2/Q3. Tag v0.10.0 on completion.

- [ ] 10.1 Anchor reader (FSA only): Zod, 256 kB cap, 90-day freshness, hostile-input handling with `anchor_invalid` fallback to a full scan (RFC-0001) — size: M — route: frontier-reasoning / extra (security)
- [ ] 10.2 `anchorFound()` in the CaptureEngine (>= 3 ordered matches by `provider_message_id` or `dom_text_sha256`); `TailExportUseCase`; `scan_scope: tail` and `capture.anchor` (RFC-0001) — size: M — route: standard / high (severity:high)
- [ ] 10.3 Side Panel 'in archive' state, Export tail action, 'new since last export' indicator (RFC-0001) — size: S — route: standard / medium (severity:medium)
- [ ] 10.4 E2E with synthetic anchors: a regenerated last message still anchors; an invalid anchor yields a full scan (RFC-0001) — size: S — route: standard / medium (severity:medium)


---

## Milestone 11 — Re-normalization CLI, Native Messaging sink, Firefox (WP-10, Phase 3 — each behind an ADR)

Implements RFC-0001 D0 (Phase 3 options): consumers whose stack cannot run the core package, a filesystem sink without FSA, and Firefox — each decided by its own ADR. Gated on the PO's answer to Q2. Tag v0.11.0 on completion.

- [ ] 11.1 `chatclipper renormalize <bundle>` CLI on the core package with linkedom, so the ingestion module can regenerate the canonical from raw.json after a normalizer fix (RFC-0001 D1) — size: M — route: frontier-reasoning / extra (adr)
- [ ] 11.2 `NativeMessagingExportSink` (RFC-0001) — size: L — route: frontier-reasoning / extra (adr, security)
- [ ] 11.3 Firefox build (RFC-0001) — size: M — route: frontier-reasoning / extra (adr)


---

## Milestone 12 — Release v1.0.0 — all milestones complete, all open issues fixed

The owner's release rule (2026-09-11): v1.0.0 is tagged only after every milestone M1-M11 is complete and every open issue is fixed. Implements the RFC-0001 cross-cutting fold's release step at the end of the cadence.

- [ ] 12.1 Fix or explicitly dispose every open issue and every open bug-ledger record (fixed / wontfix / rejected with rationale); no `TODO` without a reference (RFC-0001) — size: L — route: standard / high (severity:high)
- [ ] 12.2 Final release smoke on ChatGPT, Claude and Gemini with the release build, real FSA folder, `validate-bundle` green on every bundle (RFC-0001) — size: S — route: fast / low
- [ ] 12.3 Tag v1.0.0: annotated tag, drafted GitHub Release with `chatclipper-chromium-v1.0.0.zip`, `SHA256SUMS`, SBOM, `schema/` and `validate-bundle`; changelog rolled; the human publishes (RFC-0001 D1) — size: S — route: standard / medium (severity:medium)
- [ ] 12.4 Edge Add-ons submission of the v1.0.0 build (PRD ADR-013: when mature) (RFC-0001) — size: S — route: fast / low



---

## Spec Coverage Map

Tracks which spec section is fulfilled by which roadmap item(s). Every spec section has a
row with at least one fulfilling item and a status glyph. Legend: ⏳ not started · 🚧 in
progress · ✅ done · ❎ N/A.

| Spec § | Requirement | Roadmap items | Status |
|--------|-------------|---------------|--------|
| §1 | Objective & business context | 1.1 | ⏳ |
| §2 | Functional requirements | 1.1, 1.2 | ⏳ |
| §3 | Non-functional requirements | 1.3, 1.4 | ⏳ |
| §4 | Logical architecture | 1.1 | ⏳ |
| §5 | Public interface | 1.2 | ⏳ |
| §6 | Verification & test strategy | 1.2, 1.4 | ⏳ |
