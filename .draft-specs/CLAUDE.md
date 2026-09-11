# CLAUDE.md — LLM Conversation Exporter (`lce`)

Operating instructions for Claude Code. Specification: `docs/SPEC-LCA-001.md` **Rev 3.0** (Italian prose, English code).
Precedence on conflict: this file → spec §14 (standards) → spec §6 (export contract) → the rest.
Unresolvable conflict: **stop and ask the Product Owner**.

## What this project is — and is not

A Manifest V3 extension (Chrome/Edge) that captures the open LLM conversation (ChatGPT, Claude, Gemini) from the page DOM,
normalizes it into a provider-independent canonical JSON, and writes a **self-contained, write-once export bundle**
(`raw.json` → `capture.json` → `manifest.json`) into an inbox folder consumed by an external ingestion module (the PO's
agentic OS). Local-first, zero network, zero telemetry, zero budget (OSS only).

It is **not** an archive. The extension does not keep the archive, does not reconcile captures (added/revised/absent),
does not index, chunk, search, or produce Markdown files. Those live in the ingestion module (spec Appendix D).
If you find yourself implementing any of them, stop: you are re-introducing the scope removed in Rev 3.0.

## Golden rules (non-negotiable)

1. **No network, permanently.** Never add `fetch`, `XMLHttpRequest`, `WebSocket`, `sendBeacon`, `EventSource` — content script
   included. Features that need network are out of scope until an ADR rewrites spec §11. Refuse and say why.
2. **No private provider endpoints.** DOM extraction only. Never inject into the page MAIN world.
3. **Domain is pure.** `packages/core` never imports `chrome`, `browser`, `window`, `document`, or WXT. DOM parsing goes through
   the `HtmlParser` port (browser: `DOMParser`; Node: `linkedom`). `@lce/core` must run in Node — the ingestion module may reuse it.
4. **Browser APIs live only in** `packages/extension/infrastructure/**` and `packages/extension/entrypoints/**`
   (`eslint-plugin-boundaries`, spec §5.1). Never disable a lint rule to get around it.
5. **The extension is stateless with respect to the archive.** IndexedDB holds only settings, the FSA handle, a UX cache
   (`exports`) and `pending_bundles` for retry. The cache never decides what to export. Losing it must be harmless.
6. **The export contract (spec §6) is the product.** Any change to fields, file names, folder layout, or completion semantics
   requires an ADR, a `schema_version` bump, regenerated JSON Schema, updated `docs/CONTRACT.md`, and green `validate-bundle`.
   Within a major: additive only (new nullable/optional fields, documented enum values).
7. **Bundles are write-once.** Never overwrite (except byte-identical retry), never delete, never read under `inbox/`.
   `manifest.json` is written **last**; its presence means "complete". Never write it if an earlier file failed or (FSA) failed
   read-back verification.
8. **Never log message content or titles.** Only ids, counts, hashes, error codes (`LogSafeValue`).
9. **Never fabricate data.** Missing timestamp/model/id → `null` + coded warning. Relative times are not timestamps.
10. **Never invent DOM selectors.** Develop adapters against `fixtures/`. Missing fixture → use spec Appendix C hints, tag tests
    `@fixture-pending`, add an entry to `docs/TODO-HUMAN.md`.
11. **Validate at trust boundaries with Zod**: every Port message, every settings read, every file the extension reads
    (`export-root.json`, and in Phase 2 `state/*.anchor.json`, size-capped and treated as hostile).
12. **Errors are values.** `Result<T, DomainError>` in domain/application; `throw` only for invariant bugs and at browser boundaries.
13. **Patterns: exactly the 8 mandatory ones in spec §5.7**, each with `@pattern <Name> — <problem>` JSDoc. Do **not** introduce
    Visitor, Command Bus, Event Bus, Mediator, extra Facades, Decorator hierarchies, Builders for schema objects, Specification
    objects, Memento classes, Template Method inheritance, or single-implementation Strategy hierarchies without an ADR.
    Content blocks are data: use exhaustive `switch`.
14. **No new dependencies** outside spec §14.4 without asking. `dompurify` is not needed (nothing renders HTML).

## Architecture in one screen

```
content script  ── CaptureEngine(adapter) → RawMessage[] ── @lce/core normalize/hash/validate ──┐
      ▲ capture.start / capture.cancel                                                          │ ONE Port
      │ capture.progress / capture.chunk / capture.complete                                     ▼ (tabs.connect)
side panel ── ExportUseCase.execute(input, { onProgress, signal }) ── writeBundle():
              raw.json → (verify) → capture.json → (verify) → manifest.json  ── ExportSink: FSA | downloads | memory
              LocalCacheRepository (IndexedDB, non-authoritative) · PendingBundleRepository (retry)
service worker ── lifecycle only (open panel, on-demand inject, badge). No domain logic.
export root ── inbox/<platform>/<conversation_key>/<capture_id>/{raw,capture,manifest}.json · schema/ · state/ (written by the OS)
```

Fixed by spec §5.2, ADR-005/007/021/022/023. Normalization never moves to the service worker (no `DOMParser`).

## Correctness rules that are easy to get wrong

- **Expand collapsed content inside every scroll window, before collecting that window** (§7.3). Expanding once at the end
  misses messages that virtualized lists have unmounted.
- **Merge windows by overlap; latest observation wins; `expanded` replaces `collapsed_unresolved`** (§7.4). Gaps → `incomplete`.
- **Write order and verification** (§9.2): raw → capture → manifest. With FSA, read back and compare hashes before proceeding.
  With downloads, wait for `downloads.onChanged` → `complete` before the next file; set `write_verified: false`.
- **Deterministic serialization** (schema key order, indent 0, trailing `\n`, UTF-8 no BOM) so a retry writes identical bytes.
- **Three hashes, three questions**: `content_sha256` (semantic, excludes ids/timestamps/model/citations),
  `manifest.files[].sha256` (file bytes), `match_hints.dom_text_sha256` (DOM fragment, normalizer-independent). Never mix them.
- **Sanitize URLs** for provider media hosts (strip query/fragment) in both `capture.json` and `raw.json` (§8.5).
  External citation URLs stay untouched.
- **ULIDs identify things within a capture.** Archive identity is the ingestion module's job. Do not build identity maps.
- **`scan_scope: 'tail'` exists only with a valid anchor** (Phase 2). In v1 every export is a full scan.

## Repository layout

```
packages/core        @lce/core        schema (capture, raw, manifest, anchor), canon/hash, normalizer, url, path, serialize, invariants
packages/adapters    @lce/adapters    engine/capture-engine.ts, provider-adapter.ts, chatgpt/ claude/ gemini/ (selectors.ts, adapter.ts)
packages/extension   @lce/extension   WXT app: entrypoints/ app/(use-cases, ports) infrastructure/(sinks, indexeddb, page-gateway, settings, logger) ui/ public/_locales/
packages/tools       validate-bundle (Node, no extension deps), no-network-check, schema-gen, fixture-anonymizer, perf-fixture-gen
fixtures/            anonymized DOM snapshots per platform/selectors_version (+ *.fixture.json)
e2e/                 Playwright + synthetic-provider/ (lazy loading, virtualization, collapsed content, branches)
docs/                SPEC, CONTRACT.md, ARCHITECTURE.md, adr/, spikes/, runbooks/, TEST-STRATEGY.md, TODO-HUMAN.md
```

## Commands

```bash
pnpm install
pnpm dev              # wxt dev (Chrome) with HMR
pnpm build            # production build → packages/extension/.output/chrome-mv3
pnpm check            # lint + format:check + typecheck + test:unit + test:contract + no-network-check
pnpm test:unit        # vitest (gates: core ≥ 90%; canon/hash/slug/path/sanitizeUrl 100%; adapters ≥ 80%)
pnpm test:e2e         # playwright against the synthetic provider (after pnpm build)
pnpm schema:gen       # JSON Schema from Zod → packages/core/schema/
pnpm validate-bundle <dir>   # validates a bundle folder (schemas + hashes); ships with releases
pnpm fixture:record   # dev-only, see docs/TEST-STRATEGY.md
```

Run `pnpm check` before every commit. Never commit with a red gate.

## Work protocol

- One Work Package at a time, in the order of spec §15 (vertical slice: real capture at WP-03, real bundle validated at WP-04).
  Start each WP by restating in ≤ 10 lines how you will meet its acceptance criteria and which ADRs you will write.
- TDD where practical; test names carry the requirement id: `should write manifest last [FR-009]`.
- Small Conventional Commits (`feat(core): …`, `fix(adapters/chatgpt): …`). Every commit passes `pnpm check`.
- Keep `CHANGELOG.md` (Unreleased), `docs/ARCHITECTURE.md`, `docs/CONTRACT.md`, and `docs/adr/` current.
- Definition of Done = spec §14.3. Do not start the next WP before it is met.
- **Checkpoints**: stop after WP-02 (spike results, ADR-005 final) and after WP-04 (first real bundle) with a PO report:
  done / missing / decisions / open questions / files produced.

## When to stop and ask

- The spec is silent or contradictory on the export contract, folder layout, permissions, or completion semantics.
- A capability appears impossible in MV3 (write findings in `docs/spikes/` first).
- A dependency outside §14.4 seems necessary.
- Any change to `permissions`, `host_permissions`, or CSP.
- A test can only pass by weakening a gate (coverage, lint, CSP, invariants, contract).

## Coding standards (short form — full list in spec §14.2)

TypeScript strict (+ `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`), ESM, no `any`, no `!`, readonly domain types,
branded ids, kebab-case files, one concept per file, complexity ≤ 10, JSDoc on public API, UI strings only in `_locales/{en,it}`,
`Clock`/`IdGenerator`/`Hasher` injected (no `Date.now()`/`crypto.randomUUID()` in domain code).

## Things that look reasonable but are wrong here

- Building an archive index, a reconciliation step, a Markdown file writer, or a "sync" button.
- Deciding what to export from the local cache.
- Writing `manifest.json` before the other files are complete (and, with FSA, verified).
- Overwriting or reading anything under `inbox/`.
- Extracting or expanding once after scrolling to the top.
- Hashing timestamps/models/ids into `content_sha256`.
- Storing a signed media URL anywhere in the bundle.
- Rendering RAW `outer_html` in the extension UI.
- Using provider CSS utility classes as essential selectors.
- Adding a bus, a Visitor, or a Facade "for structure".
