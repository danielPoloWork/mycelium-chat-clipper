# Threat model — mycelium-chat-clipper

> **Owner:** the **security-auditor** role (it drafts here; findings feed the audit risk
> register). Produced and kept current by the **audit threat-modeling sub-mode**
> (`/eados security` → `/eados audit`). Method: **STRIDE**. First pass: **2026-09-11**, at the
> bootstrap audit, before any source code exists — the boundaries below are those the PRD
> (`SPEC-LCA-001` Rev 3.0 §5–§11) and RFC-0001 fix by design, plus the repository's own
> delivery surface. Update it in the same PR as any change to a trust boundary.
>
> Confidence tags follow `AGENTS.md` §12: `certain` = verified in a file or by a command in this
> audit, `likely` = an inference from named evidence (the PRD, the RFC, a template), `guessing`
> = flagged as such.

## 1. Scope & trust boundaries

| # | Boundary | Untrusted inputs crossing it | Assumptions |
|---|---|---|---|
| B1 | **Provider page → content script** (isolated world; ChatGPT, Claude, Gemini DOM) | the conversation DOM itself: message HTML, attributes, `data-*` ids, media URLs, collapsed-content controls, timestamps, model labels; page scripts that may mutate the DOM during capture | the content script runs in the ISOLATED world and never in MAIN (PRD §11.2, NFR-007); nothing but scrolling and expansion is performed on the page (§7.3); every DOM-derived value is data, never code |
| B2 | **Content script ↔ Side Panel** (the single `tabs.connect` Port) | every Port message in both directions: `detect.result`, `capture.progress`, `capture.chunk` payloads (raw + canonical messages), `capture.complete`, `capture.error` | the Port is opened by the Side Panel; every message is Zod-validated on both sides with `protocol_version: 1`, a random `session_id` and a monotonic `seq` (PRD §5.6) |
| B3 | **Extension → export folder** (`FsaExportSink` via the File System Access API; `DownloadsExportSink` under `Downloads/LLM-Exports/`) | path segments derived from provider ids and titles (`conversation_key`, `capture_id`); the bytes written; the read-back on FSA | write-once, never delete, never read under `inbox/`; `manifest.json` last; FSA refuses `..`; segment sanitization and the 120-character path budget (PRD §9.3, CON-006) |
| B4 | **Ingestion module → extension** (`state/<platform>/<conversation_key>.anchor.json`, Phase 2) | the anchor file: schema, size, freshness, `anchor_messages` content | hostile by declaration: Zod, ≤ 256 kB, ≤ 90 days, read-only, ignored with `anchor_invalid` on any failure; never affects a bundle's validity (PRD §6.6, §11.1) |
| B5 | **Extension → ingestion module** (the bundle contract itself: `raw.json`, `capture.json`, `manifest.json`, `export-root.json`, `schema/`) | none inbound; outbound, `raw.json → outer_html` is untrusted page content re-emitted verbatim (URLs redacted) | the consumer never renders or content-parses `outer_html` (Appendix D.1); hashes in `manifest.json` let the consumer verify integrity without trusting the producer (ADR-022) |
| B6 | **Extension ↔ browser storage** (IndexedDB `chatclipper`: `exports`, `pending_bundles`, `handles`, `logs`; `storage.local` settings) | settings and cache records read back from storage (same profile, other extensions cannot read them, but the data is ours to distrust after a schema change) | non-authoritative except `handles`; Zod on every settings read; losing the cache is harmless (P1, ADR-023) |
| B7 | **Extension → network** | none — by construction | CSP `connect-src 'none'`; no `fetch`, `XMLHttpRequest`, `WebSocket`, `sendBeacon`, `EventSource` anywhere, content script included; `no-network-check` scans the production bundle (NFR-006, ADR-012) |
| B8 | **Distribution → user** (GitHub Releases zip + `SHA256SUMS` + SBOM; Edge Add-ons later; "Load unpacked" in developer mode) | the artifact the user installs | integrity via `SHA256SUMS` and the release attached to a tag; the human publishes the release (`AGENTS.md` §6.1) |
| B9 | **Repository delivery surface** (GitHub Actions CI/release, Dependabot, contributors' PRs, `tools/consistency_lint.py` executing in CI) | third-party actions, dependency updates, inbound PR content, the lint's inputs (repository files) | squash-only, PR-required `main` (ruleset pending the owner's one-time setup); CODEOWNERS = the owner; actions pinned as rendered (see finding RR-02); Dependabot grouped weekly; the lint reads files, never network |

## 2. STRIDE pass

Work the six categories per boundary. Every cell carries a threat, a mitigation, or an explicit `n/a (reason)`. Status: ▢ open · ✅ mitigated by design (to be verified by the test named) · ☐ deferred to a roadmap item.

### B1 — Provider page → content script

| Category | Threat considered | Mitigation / control | Status |
|---|---|---|---|
| Spoofing | A page (or an injected script on it) impersonates a supported provider to be captured | `matches()` on origin + `probe()` on essential selectors; host permissions limit injection to the three provider origins (PRD §11.2) — `likely` | ✅ verified by adapter contract tests (4.2) |
| Tampering | Page content is crafted so that normalization emits attacker-chosen Markdown/HTML into the bundle | RAW is never rendered by the extension; the canonical is Markdown text; residual HTML is never emitted (PRD §8.2); consumers must not render `outer_html` (D.1) — `likely` | ✅ verified by normalizer tests (4.3) |
| Repudiation | A capture cannot be traced to its page state | `provenance` (adapter, selectors version, browser), `dom_text_sha256` per message, `raw.json` kept as insurance (P4) — `certain` (contract) | ✅ contract tests (2.1) |
| Information disclosure | Signed media URLs or tokens embedded in the page leak into the bundle | `sanitizeUrl` reduces provider media hosts and `source_url` to origin + pathname in both RAW and canonical (FR-019, §8.5) — `likely` | ✅ property + contract tests (2.3, Appendix B) |
| Denial of service | A huge or pathological conversation (thousands of messages, deeply nested HTML) exhausts the tab | chunking + cooperative yield (NFR-002), `maxPasses` / `maxDurationMs` guards, memory budget NFR-003, gap handling → `incomplete` (§7.3) — `likely` | ☐ perf fixture and blocking budgets at 6.1 |
| Elevation of privilege | Page script escapes into the extension context | ISOLATED world only; no MAIN-world injection; no `eval`, no remote code (CSP `script-src 'self'`) — `certain` (PRD §11.2) | ✅ manifest snapshot test (NFR-007) |

### B2 — Content script ↔ Side Panel (the Port)

| Category | Threat considered | Mitigation / control | Status |
|---|---|---|---|
| Spoofing | A compromised page forges Port messages to the Side Panel | the Port is opened by the Side Panel (`tabs.connect`), the content script only accepts on `runtime.onConnect`; random `session_id`; the page cannot reach the extension's Port in the isolated world — `likely` | ✅ e2e (4.4) |
| Tampering | Chunks reordered, duplicated or altered | monotonic `seq`, Zod on every message, chunk caps (≤ 50 messages / 512 kB) — `certain` (§5.6) | ✅ unit tests on the protocol (4.4) |
| Repudiation | A session's outcome cannot be reconstructed | `capture.complete` stats + warnings; the diagnostic bundle (FR-016); the ring-buffer log (§12) — `likely` | ☐ 9.1 diagnostics |
| Information disclosure | Message content leaks into logs | `LogSafeValue`: ids, counts, hashes, codes only (NFR-008) — `certain` (requirement) | ✅ type-level guard + unit test |
| Denial of service | A flood of chunks starves the panel | chunk caps and cooperative yield; `Cancelled` on abort — `likely` | ✅ e2e (6.4) |
| Elevation of privilege | n/a — both ends run in the same extension with the same permissions; no privilege differential to escalate across (`certain`) | — | ✅ |

### B3 — Extension → export folder (FSA / downloads)

| Category | Threat considered | Mitigation / control | Status |
|---|---|---|---|
| Spoofing | Writing into a folder the user did not choose | FSA handle obtained by a user gesture and persisted per profile; downloads confined under `Downloads/` (CON-002) — `certain` (platform) | ✅ |
| Tampering | Path traversal via a crafted title or provider id | segment sanitization (reserved characters and names, no trailing dot/space), paths relative to the handle, FSA forbids `..`, 120-character budget (§9.3) — `likely` | ✅ property tests (2.3) |
| Repudiation | A bundle appears complete but was not verified | `manifest.json` written last only after read-back verification (FSA) or `downloads.onChanged → complete`; `write_verified` recorded honestly as `false` on downloads (NFR-004) — `certain` (contract) | ✅ e2e Appendix B "Bundle completeness" (5.5) |
| Information disclosure | Bundles contain the full conversation text and land in a user-chosen folder | documented in the export root `README.md` and `PRIVACY.md`: bundles are personal data, protect them as any local file (§11.4) — `certain` (requirement) | ☐ 9.3 PRIVACY.md |
| Denial of service | Disk full, share unreachable, download interrupted | `WriteFailed` / `WriteVerificationFailed` / `DownloadInterrupted` → `PendingWrite` + byte-identical Retry write (FR-015) — `likely` | ☐ 6.2 |
| Elevation of privilege | Overwriting or deleting the consumer's files under `inbox/` | write-once (fail if exists with different bytes), never delete, never read under `inbox/` (ADR-022) — `certain` (contract) | ✅ sink unit tests (5.1) |

### B4 — Ingestion module → extension (anchor file, Phase 2)

| Category | Threat considered | Mitigation / control | Status |
|---|---|---|---|
| Spoofing | A forged anchor makes the extension skip messages (a partial export presented as complete) | a `tail` bundle is by contract an append/revision of the tail, never evidence of absence (Appendix D.3); `scan_scope: tail` + `capture.anchor` recorded so the consumer knows — `certain` (contract) | ☐ 10.1–10.4 |
| Tampering | Malformed or oversized anchor | Zod, ≤ 256 kB, ≤ 90 days; invalid → `anchor_invalid` + full scan — `certain` (§6.6) | ☐ 10.1 |
| Repudiation | n/a — the anchor is advisory input, not an action of the extension (`certain`) | — | ✅ |
| Information disclosure | The anchor reveals archive content to the extension | it carries only ids, hashes and roles of the last 3–10 messages, no text (§6.6) — `certain` | ✅ |
| Denial of service | A huge anchor file stalls the read | the 256 kB cap before parsing (§6.6) — `certain` | ☐ 10.1 |
| Elevation of privilege | n/a — read-only input with no code paths beyond matching (`certain`) | — | ✅ |

### B5 — Extension → ingestion module (the bundle contract)

| Category | Threat considered | Mitigation / control | Status |
|---|---|---|---|
| Spoofing | Another producer writes bundles the consumer trusts | `producer {name: 'chatclipper', version}` in `manifest.json` and `export-root.json` (RFC-0001 D1); integrity via `manifest.files[].sha256` — the consumer verifies before ingesting (D.1) — `certain` (contract) | ✅ `validate-bundle` (2.5) |
| Tampering | A bundle altered after writing | file hashes in `manifest.json`; `validate-bundle` red on an altered hash (Appendix A fixture) — `certain` | ✅ 2.5 |
| Repudiation | Which extension build produced a bundle | `provenance.extension_version`, `core_version`, adapter + `selectors_version` — `certain` | ✅ 2.1 |
| Information disclosure | `raw.json → outer_html` carries page HTML, possibly with scripts | declared untrusted; media URLs redacted in RAW too (§8.5); consumers must not render or content-parse it (D.1) — `certain` (contract) | ✅ documented in `docs/CONTRACT.md` (2.6) |
| Denial of service | Bundle floods the inbox (repeated exports) | `capture_id` idempotency; consumer-side quarantine rule for manifest-less bundles (D.1) — `likely` | ✅ contract |
| Elevation of privilege | n/a — files only; no execution path into the consumer (`certain`) | — | ✅ |

### B6 — Browser storage

| Category | Threat considered | Mitigation / control | Status |
|---|---|---|---|
| Spoofing | n/a — same-profile extension storage; no other principal can write it (`likely`, platform guarantee) | — | ✅ |
| Tampering | A stale or corrupted cache decides what is exported | the cache never decides what to export (FR-020, P1); Zod on settings reads — `certain` (requirement) | ✅ Appendix B "Statelessness" (5.5) |
| Repudiation | n/a — cache is UX only (`certain`) | — | ✅ |
| Information disclosure | The `exports` cache holds titles/annotations; `pending_bundles` holds full bundle bytes until retry | documented data location and "how to delete everything" (§11.4); `pending_bundles` purged after 7 days (§9.4) — `certain` | ☐ 9.3 PRIVACY.md |
| Denial of service | `pending_bundles` grows unbounded on repeated failures | 7-day purge; `unlimitedStorage` permission; user-visible *Retry write* — `likely` | ☐ 6.2 |
| Elevation of privilege | n/a (`certain`) | — | ✅ |

### B7 — Network

| Category | Threat considered | Mitigation / control | Status |
|---|---|---|---|
| Spoofing / Tampering / Repudiation / Information disclosure / Denial of service / Elevation of privilege | **Exfiltration by the extension or a dependency** is the single threat this boundary exists to eliminate | there is no boundary: CSP `connect-src 'none'`, no network API anywhere, `no-network-check` on the production bundle (NFR-006, ADR-012); a feature needing network requires an ADR that rewrites PRD §11 — `certain` (requirement) | ☐ 9.2 (the check ships at M9; the CSP lands at 1.9) |

### B8 — Distribution

| Category | Threat considered | Mitigation / control | Status |
|---|---|---|---|
| Spoofing | A look-alike zip | releases attached to annotated tags on the repository; `SHA256SUMS`; Edge Add-ons listing later (ADR-013) — `likely` | ☐ 9.4 |
| Tampering | Modified artifact | `SHA256SUMS` + CycloneDX SBOM per release (§13.5) — `likely` | ☐ 9.4 |
| Repudiation | Which commit shipped | tag on the merge commit; release drafted by the agent, published by the human (`AGENTS.md` §6.1) — `certain` | ✅ release.yml rendered |
| Information disclosure | n/a — public OSS artifact (`certain`) | — | ✅ |
| Denial of service | n/a — GitHub-hosted (`certain`) | — | ✅ |
| Elevation of privilege | "Load unpacked" in developer mode bypasses store review | accepted (RSK-10); documented developer-mode warning; Edge Add-ons when mature — `certain` (PRD) | ✅ accepted |

### B9 — Repository delivery surface (CI, dependencies, contributors)

| Category | Threat considered | Mitigation / control | Status |
|---|---|---|---|
| Spoofing | A PR from a non-owner lands unreviewed, or `main` is pushed directly | CODEOWNERS = owner; but `main` has **no protection or ruleset** and the repository still allows merge commits and rebase (API, 2026-09-11) — the contract's PR-required / squash-only / no-force-push guarantees rest on owner discipline until the one-time setup (`docs/workflow/github-setup.md` §1) — `certain` | ▢ RR-04 |
| Tampering | A mutable action tag is retargeted upstream (supply chain) | template-owned actions are SHA-pinned with version comments; the profile's `pnpm/action-setup@v4`, `actions/setup-node@v4` and one `actions/checkout@v6` are tag-pinned by upstream design (ADR-0009 §3), Dependabot-managed weekly — `certain` (grep of the rendered workflows) | ▢ RR-02 |
| Repudiation | Who changed what | signed-off Conventional Commits, PR body becomes the squash commit, assignee = owner — `certain` (`AGENTS.md` §6) | ✅ |
| Information disclosure | Secrets committed | no secret-like tokens in the tracked tree (scan 2026-09-11); `.gitignore` covers local harness files; CI uses `GITHUB_TOKEN` only — `certain` (scan + workflow read) | ✅ |
| Denial of service | Dependabot floods PRs | grouped weekly updates, `open-pull-requests-limit: 1` per ecosystem — `certain` (dependabot.yml) | ✅ |
| Elevation of privilege | CI job permissions broader than needed | `ci.yml` runs with `permissions: contents: read`; `release.yml` with `contents: write` (needed to draft releases) — least privilege holds — `certain` (verified 2026-09-11) | ✅ RR-03 closed |

## 3. Findings → the risk register

Threats that survive analysis are recorded in [`risk-register.md`](risk-register.md) with severity,
component, impact and mitigation (RR-01 … RR-06 at the bootstrap audit). No confirmed, reproducible
defect exists yet (no source code), so the bug ledger stays empty; no vulnerability warrants an
advisory. The ☐ cells above are design-time mitigations whose verification is a named roadmap item;
they are re-audited when that item ships.
