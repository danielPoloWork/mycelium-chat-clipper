# RFC-0001: Mycelium Chat Clipper — system design and repository shape

- **Status:** Accepted (rev 3, 2026-09-11) — the owner decided D1–D4 on 2026-09-11; D2 was decided against the tech-lead's recommendation and the dissent is recorded in D2
- **Author:** tech-lead (agent, drafted 2026-09-11) · **Reviewers:** reviewer, enterprise-architect (cross-cutting: repository layout and contract literals) · **Approver:** tech-lead, after the owner's decision
- **Date:** 2026-09-11
- **Related:** the product specification `.draft-specs/SPEC-LCA-001.md` Rev 3.0 (the PRD; Italian prose, English code; Product Owner: Andrea); the project manifest `orchestrator/project.yaml` (`manifest_rev 1`, `phase: design`); bootstrap PR #1; milestone **M1 — Project bootstrap & CI**; the factory's generated ADR-0001 (record decisions) and ADR-0002 (cross-language source layout); lesson L-0005 (never reserve ADR numbers ahead of authoring).

> Written *before* the code. The engineering design is fixed by the PRD (spec §5–§11) and was
> imported into the manifest's `spec.*` fields on 2026-09-11; this RFC restates it in the design
> folds the plan phase derives from, and **settles the four repository-shape decisions the import
> left open** (D1–D4). Approving this RFC encodes those four decisions; everything in D0 is a
> restatement of decisions the PRD already made and is referenced, not re-decided. Confidence tags
> follow `AGENTS.md` §10: `certain` = verified in a file or by a command this session, `likely` =
> an inference from named evidence, `guessing` = flagged as such.

## Context

Mycelium Chat Clipper is a Chromium Manifest V3 extension (Chrome/Edge >= 120) that captures the
conversation open in the active tab — ChatGPT, Claude, or Gemini — from the page DOM, normalizes it
into a provider-independent canonical JSON, and writes a self-contained, **write-once export bundle**
(`raw.json` → `capture.json` → `manifest.json`) into an inbox folder that an external ingestion
module (the owner's agentic OS) consumes. It is local-first: zero network, zero telemetry, zero
private provider endpoints, zero budget. It is an **exporter, not an archive** (spec ADR-021):
archive identity, reconciliation, indexing, chunking, search and Markdown projection belong to the
ingestion module.

The PRD is complete to implementation depth: layers and dependency matrix (§5.1), MV3 contexts
(§5.2), the export flow and session state machine (§5.4–5.5), the single Port channel (§5.6), the
eight mandatory patterns (§5.7), the export contract as Zod schemas (§6), the capture algorithm
(§7), normalization and the three hashes (§8), persistence (§9), UI (§10), the threat model and
permissions (§11), observability (§12), the quality pyramid and CI gate order (§13), standards and
approved dependencies (§14), and a risk-first work-package plan WP-00…WP-10 (§15). All of it is in
the manifest (`spec.objective`, `functional_reqs` FR-001…021, `nonfunctional_reqs` NFR-001…018 +
CON-001…008, `nfr_budgets`, `architecture`, `patterns`, `public_api`, `verification`,
`milestones` 2–11) and was confirmed by the owner on 2026-09-11.

What the import could **not** settle, and what forces a decision **now** — before WP-00 lays down
the workspace and WP-01 freezes the schema literals:

1. **Naming.** The PRD uses the working code `lce` as a *machine-visible identifier*: the npm scope
   `@lce/*`, the contract literal `manifest.json → producer.name: z.literal('lce')` (§6.5, PRD line
   502), the `producer` object of `export-root.json` (§6.1, line 347), the IndexedDB database name
   (§9.4, line 705), the Port name (§5.6, line 303), the release zip (§13.5, line 842), the CLI
   (`lce renormalize`, Appendix E Q2, line 1218), and the **normative Appendix A fixture**
   (`"producer": { "name": "lce" }`, line 1092). The owner named the repository
   `mycelium-chat-clipper`, confirmed the slug `chatclipper` and the group path `it/mycelium-labs`.
   The PRD itself says the final name is the PO's decision (line 10). The PRD is also internally
   inconsistent about its own id: the header, the ID row and the closing line say `LCA-SPEC-001`
   (lines 1, 7, 1238) while the file name, §14.1 and Appendix F say `SPEC-LCA-001` (lines 857, 1228).
2. **Layout.** PRD §14.1 prescribes root-level `packages/{core,adapters,extension,tools}`; the
   governance layer the owner adopted prescribes the Maven-style cross-language tree
   `src/main/typescript/it/mycelium-labs/chatclipper/` (generated ADR-0002, normative for every
   sibling project) with the four packages seeded as its layered skeleton (`spec.layers`), and the
   generated `consistency_lint` requires every pattern's code location to sit somewhere under
   `src/` — any `src/` path, not the derived root (`.eados-core/templates/tools/consistency_lint.py`,
   `check_patterns`, `certain`). The renderer derives the source root from language, group path
   and slug with **no manifest override** (`.eados-core/tools/render.py` line 304, `certain`), so the
   scaffold always renders the ADR-0002 tree; any other tree is a superseding decision.
3. **ADR numbering.** The PRD carries a decision table ADR-001…024 (010, 019, 020 transferred to
   the ingestion module); the scaffold seeds ADR-0001 and ADR-0002 with fixed content, and ADR-0001
   makes numbering sequential and ADRs immutable once Accepted (a change is a new, superseding ADR).
4. **The two draft documents.** The generated repository is English-on-disk (`AGENTS.md` §2); the
   PRD is a human-authored Italian document and the source of product truth, and it travels with a
   draft operating-instructions file (`.draft-specs/CLAUDE.md`) whose golden rules still name the
   PRD's `packages/` layout and `@lce/*` scope. Both need a governed home and a governed mechanism.

Constraints that bound every option: exporter-not-archiver (spec ADR-021), zero network (ADR-012),
MV3 (CON-001), exactly eight patterns (CON-004), one maintainer (CON-007), a contract consumable
from any language (CON-008), and the governance contract: English on disk, one PR at a time, the
human holds every terminal gate, and the manifest is the single source of truth the scaffold
renders from (`orchestrator/project.yaml` line 1; the English spec is generated from it).

## Decision

### D0 — The system design the plan derives from (restated; normative source: spec §5–§11)

**Style: Hexagonal (Ports & Adapters)**, lint-enforced (spec ADR-003, `eslint-plugin-boundaries`).
Four pnpm workspace packages: `core` (domain: schemas, normalizer, canon/hash, sanitizeUrl, path,
deterministic serialize, invariants, Result/errors — zero browser APIs, runs in Node), `adapters`
(`CaptureEngine` + one `ProviderAdapter` per platform), `extension` (WXT app: entrypoints, app
use-cases and ports, infrastructure, Preact UI, `_locales/{en,it}`), `tools` (validate-bundle,
no-network-check, schema-gen, fixture-anonymizer, perf-fixture-gen). Dependency matrix: domain →
nothing; application → domain (+ ports); adapters → domain; infrastructure → domain, application,
adapters; ui → application + domain types. One composition root per MV3 context; manual DI; no bus.

**MV3 contexts:** content script (isolated world) detects, captures, normalizes, hashes, validates
and streams chunks over one Port; the Side Panel hosts the use cases, assembles the bundle, writes
through an `ExportSink`, keeps the non-authoritative cache; Options holds settings and the FSA
picker; the service worker is lifecycle-only (it has no `DOMParser`, CON-001).

**Export contract as product** (spec §6): `inbox/<platform>/<conversation_key>/<capture_id>/` with
`raw.json` (DOM fragments, untrusted), `capture.json` (CanonicalCapture 1.0.0), `manifest.json`
(written **last**; its presence means complete; file hashes; `write_verified`); `export-root.json`,
`schema/*.v1.json`, `README.md` at the root; `state/*.anchor.json` written by the consumer (Phase
2). Zod is the single schema definition; JSON Schema is generated. Additive within a major.

**Session FSM, write sequence, three hashes, capture algorithm:** as in the manifest's
`spec.architecture` and the folds below. **Eight mandatory patterns**, `pattern_discipline:
enforced` (spec §5.7 / ADR-018): Ports & Adapters, Provider Adapter by composition, ordered parser
chain of pure functions, Repository, write sequence with completion marker, Finite State Machine,
Result type, manual DI. Anything else requires an ADR.

### D1 — Product identifier: `chatclipper` replaces `lce` wherever it is machine-visible

| Surface | PRD (Rev 3.0) | Decision |
|---|---|---|
| npm scope + packages | `@lce/core`, `@lce/adapters`, `@lce/extension`, `@lce/tools` | `@mycelium-labs/chatclipper-core`, `-adapters`, `-extension`, `-tools` |
| `manifest.json → producer.name` (§6.5, line 502) | `z.literal('lce')` | `z.literal('chatclipper')` |
| `export-root.json → producer` (§6.1, line 347) | `producer` object, no schema given | `producer: { name: 'chatclipper', version }`, with a Zod schema added to `core` (the PRD gives none) |
| Appendix A example `manifest.json` (line 1092; a **normative** fixture that roadmap 2.5 turns into an Ajv test) | `"producer": { "name": "lce", ... }` | `"chatclipper"` — Rev 3.1 must edit Appendix A, or the fixture fails against the new literal |
| IndexedDB database name (§9.4, line 705) | `lce` | `chatclipper` |
| Port name (§5.6, line 303) | `tabs.connect(tabId, { name: 'lce' })` | `{ name: 'chatclipper' }` |
| Release zip (§13.5, line 842) | `lce-chromium-vX.Y.Z.zip` | `chatclipper-chromium-vX.Y.Z.zip` |
| CLI names (§6.7; WP-10, line 1218) | `validate-bundle`, `lce renormalize` | `validate-bundle`, `chatclipper renormalize` |
| Downloads sink folder (§9.1, line 664; a first-class sink, not a fallback) | `Downloads/LLM-Exports/` | **unchanged** — user-facing, provider-neutral, not an identifier |
| Spec document id | `LCA-SPEC-001` (header, ID row, closing line) vs `SPEC-LCA-001` (file name, §14.1, Appendix F) | **`SPEC-LCA-001`** — matches the file and every cross-reference; Rev 3.1 aligns the header |
| Display name | `__MSG_extName__` | "Mycelium Chat Clipper" in `_locales/{en,it}` |

Only `producer.name` (and the new `export-root.json` schema) is contract-visible, and both are
literals in schemas that do not exist yet. Deciding them **before WP-01** is free **provided no
consumer code has been written against Rev 3.0** — `likely`; the PO is asked to confirm (open
question Q6). After v0.1.0 the same change would be a consumer-visible one under §6.7. The rename
is recorded as an ADR at WP-00 (D3), and the PO is asked to issue PRD **Rev 3.1** with these
literals, the Appendix A fixture, and the aligned header id (see Consequences).

### D2 — Repository layout: a project tree `src/mycelium/`, superseding the generated ADR-0002

**Owner's decision (2026-09-11):** the four packages live at **`src/mycelium/{core, adapters,
extension, tools}`**, with a **160-character budget** for tracked repo-relative paths. This was
decided against the tech-lead's recommendation; per `AGENTS.md` §10.4 the decision stands and the
dissent is recorded here, not relitigated.

> **Recorded dissent (tech-lead, 2026-09-11).** Position: keep the derived tree
> `src/main/typescript/it/mycelium-labs/chatclipper/` as generated — it is the one property the
> adopted governance exists to reproduce (an identical tree across every sibling project), it is
> what the scaffold renders, and the generated lint keys the version file off it. Alternative
> offered: if depth was the concern, shorten `group_path` in the manifest so the derived tree
> becomes `src/main/typescript/mycelium/chatclipper/` while ADR-0002 keeps its shape. Risk accepted
> by the owner: the generated ADR-0002 must be **superseded** by a new ADR at WP-00; the scaffold
> still renders the derived tree, which WP-00 must remove; the rendered `AGENTS.md` §5 and the
> generated lint's `version_file` configuration must be re-pointed; the cross-project identical
> tree is given up for this repository.

The resulting layout reconstitutes PRD §14.1 under a `src/mycelium/` prefix, each package owning its
own sources, tests and (where applicable) benchmarks — which also removes the test-resolution
problem of the earlier draft, because tests live inside the package they exercise:

```text
src/mycelium/
├── core/          @mycelium-labs/chatclipper-core       package.json · tsconfig.json · src/ (incl. src/version.ts) · test/ · bench/
├── adapters/      @mycelium-labs/chatclipper-adapters   src/ · test/ (adapter DOM-contract tests over fixtures/)
├── extension/     @mycelium-labs/chatclipper-extension  WXT project root: wxt.config.ts, entrypoints/, app/, infrastructure/, ui/, public/_locales/ · test/
└── tools/         @mycelium-labs/chatclipper-tools      validate-bundle · no-network-check · schema-gen · fixture-anonymizer · perf-fixture-gen · test/ (incl. the path-budget test)
e2e/                                                     Playwright suite + synthetic-provider/ (a test-only component)
fixtures/                                                anonymized DOM snapshots per platform/selectors_version (data, not code)
pnpm-workspace.yaml · package.json (root scripts: check, build, test:*, schema:gen, validate-bundle) · tsconfig.base.json · eslint.config.js · vitest.config.ts (test.projects) · playwright.config.ts
```

Rules that make this concrete and checkable:

- **Workspace glob** (`pnpm-workspace.yaml`): `src/mycelium/*`. Each package resolves its own
  dependencies; cross-package imports are `workspace:*` dependencies.
- **Test runners.** Vitest: `vitest.config.ts` at the root with `test.projects` = the four packages
  (the separate `vitest.workspace.ts` file is deprecated in Vitest 3.2 and removed in 4 — `likely`,
  not verified this session); coverage thresholds per package (NFR-010). Playwright: `testDir` =
  `e2e/`. `fixtures/` holds data only; code outside `src/mycelium/`, `e2e/` and the root config
  files is a review failure (the superseding ADR states it).
- **The scaffold's seeded tree.** The renderer will still create the derived
  `src/main/typescript/it/mycelium-labs/chatclipper/` (and its test mirror) because the root is not
  overridable. To keep that seed minimal, the architect sets `governance.capabilities.layered:
  false` and `spec.layers: []` in the manifest before scaffold (no four empty layer folders are
  seeded); WP-00 removes the residual `src/main` / `src/test` skeleton in the same PR that authors
  the superseding ADR.
- **Version lockstep.** The version file is `src/mycelium/core/src/version.ts`; every package's
  `package.json` version is asserted equal to it by a workspace script, and the provenance fields
  `core_version` / `extension_version` (PRD §6.3) read their own package's version. The generated
  `consistency_lint`'s `CONFIG["version_file"]` points at the derived root (the template's SRC_MAIN placeholder joined with `version.ts`)
  and, finding no file there, falls back to the README badge (`certain`, from the template's
  `check_version_lockstep`); WP-00 re-points that configuration to `src/mycelium/core/src/version.ts`
  as part of superseding ADR-0002, so the lockstep check stays real.
- **Ignore rules.** The generated `.gitignore` anchors `/node_modules/`, `/dist/`, `/coverage/` at
  the root; pnpm creates per-package `node_modules/` under `src/mycelium/*`, WXT writes `.output/`
  and `.wxt/` inside the extension package, Playwright writes `test-results/` and
  `playwright-report/`. WP-00 (1.6) adds the unanchored patterns `node_modules/`, `.output/`,
  `.wxt/`, `coverage/`, `test-results/`, `playwright-report/` (`certain`, from
  `.eados-core/templates/gitignore.tmpl`).
- **Path budget, derived.** Windows' default limit is 260 characters for an absolute path. The
  source prefix is 13 characters (`src/mycelium/`); the deepest expected tracked file is under 100.
  The rule the owner set: **no tracked repo-relative path exceeds 160 characters** (checked by a
  unit test in `tools`), which leaves **~90 characters for the clone root**. README/CONTRIBUTING
  (1.12) state it: clone under a short root, or enable `git config core.longpaths true` and Windows
  `LongPathsEnabled`. Generated paths are the longer ones and scale with the clone root: pnpm's
  virtual store sits at the workspace root (`node_modules/.pnpm/<name>@<version>_<peers>/node_modules/<name>/...`)
  and per-package `node_modules/` hold only symlinks; WXT's `.output/` is ~35 characters deep in the
  extension package. `likely` safe under a ~90-character clone root. The Windows CI job is a
  **smoke**, not a proof: the runner checks out under a 49-character root.

### D3 — ADR numbering: sequential at authoring time; the PRD table is mapped, not copied

- The repository's ADR sequence is the generated one: ADR-0001 (record decisions) and ADR-0002
  (source layout) are seeded by the scaffold; every further ADR takes the **next number when it is
  authored** (L-0005 — no reservations), one decision per file (the RFC's own Alternatives reject
  multi-decision ADRs, so spec 002 and 014 stay **separate**: toolchain vs UI framework).
- The PRD's ADR-001…024 are **decisions**, not files. The manifest's roadmap item 1.10 lists the
  eight WP-00 decisions the PRD names for WP-00 (spec 001, 002, 003, 004, 018, 021, 022, 023; PRD
  line 902). This RFC **proposes to the plan phase** the following amendment of 1.10 and the
  placement of the rest — an amendment, not something already recorded:
  - **WP-00 (1.10):** spec 001 MV3 Chromium-first; 002 TypeScript/WXT/pnpm; 003 Ports & Adapters
    with lint-enforced boundaries; 004 Zod single definition, generated JSON Schema; 012 zero
    network permanently (the CSP lands at WP-00); 013 Apache-2.0 + GitHub Releases + Edge Add-ons
    (the LICENSE lands at WP-00); 014 Preact + signals, native CSS, `_locales`; 018 the eight
    mandatory patterns; 021 exporter-not-archiver; 022 write-once bundle with final manifest; 023
    archive-stateless extension; plus this RFC's **D1** (identifier), **D2** (the `src/mycelium/`
    tree — a new ADR that **supersedes** the generated ADR-0002, which ADR-0001 requires for any
    change to an Accepted ADR) and **D4** (documents and precedence).
  - **WP-01 (M2):** spec 004's schemas realize 008 ULID identity within a capture, 009 the three
    hashes, 011 RAW as redacted DOM fragments, 016 attachments as metadata only, 024 the file-based
    anchor protocol (its schema is authored there).
  - **WP-02 / CP1 (M3):** spec 005 sinks — **provisional until CP1**, authored then.
  - **WP-03 (M4):** 006 DOM-only, no private endpoints; 007 normalization in the content script
    behind `HtmlParser`; 015 active branch only; 017 fixtures + synthetic provider, no live
    provider in CI.
- `docs/adr/README.md` carries a **Spec mapping** column (`spec ADR-0xx → ADR-00NN`) written as
  **plain text**, never as a link, so the generated `adr-index` lint (which reads `NNNN-slug`
  presence and `](NNNN-slug.md)` links) is unaffected (`certain`). The PRD's ids remain valid
  references in prose, written as "spec ADR-021".

### D4 — The two draft documents: the PRD is imported verbatim; the draft instructions become house rules recorded in the manifest

- **`.draft-specs/SPEC-LCA-001.md` → `docs/prd/SPEC-LCA-001.md`, verbatim** (id per D1). The
  `software` domain's product artifact is the PRD; `docs/prd/**` is owned by the product-manager
  role (authority map). The PO acts in that role; no authority record maps a person to it yet —
  the plan phase confirms it (`guessing` until then). The PRD is the **source of product truth**.
  The English `docs/specs/01_spec_chatclipper.md`, generated at scaffold from the manifest, is the
  **engineering spec**.
- **Precedence, scoped.** On product requirements (FR/NFR/CON, §6 semantics) the **PRD prevails**.
  On the D1–D4 surfaces the **repository's ADRs prevail until Rev 3.1 aligns the PRD** (so Rev
  3.0's `lce` does not override D1). A PRD-vs-spec conflict is a **manifest defect first**: the fix
  runs PRD → manifest `spec.*` (state writer: enterprise-architect, `manifest_rev` + 1) → the
  regenerated or edited English spec, in the same PR that finds it — never silently. An
  irreconcilable conflict stops the work and asks the PO (PRD §0.1).
- **Language exception.** The PRD is one imported, human-authored Italian document; everything
  agent-authored stays English. The generated `AGENTS.md.tmpl` has no template hook for such an
  import (only the comment-language, documentation-language and i18n exceptions, `certain`), so
  the exception is recorded as a **house rule** (§15, below) rather than an edit of the rendered
  `AGENTS.md`. The PRD is **not** an i18n artifact: `docs/i18n/{it,zh,ja}/` are, by the generated
  contract's own definition, derived copies of English sources (`AGENTS.md.tmpl` §2); the PRD is
  a source, is not listed in `translation-status.md`, and the freshness lint ignores unlisted files.
- **`.draft-specs/CLAUDE.md` → house rules in the manifest, before scaffold.** Its fourteen golden
  rules and the "things that look reasonable but are wrong" list are project non-negotiables. The
  mechanism EADOS provides is `governance.house_rules` in `orchestrator/project.yaml`, which the
  renderer emits as the generated `AGENTS.md` **§15 Organization House Rules** when non-empty
  (`.eados-core/tools/render.py` lines 336 and 358, `certain`); house rules win over defaults.
  Therefore: (a) the text is **rewritten** to the D1/D2/D4 names (`@mycelium-labs/chatclipper-*`,
  the tree path, `docs/prd/SPEC-LCA-001.md`) and **stripped** of the sections that other governed
  surfaces own — the layout (ADR-0002 / `AGENTS.md` §5), the command list (the roadmap), and the
  draft's own precedence line (replaced by the scoped rule above); (b) it is written into
  `governance.house_rules` by the enterprise-architect as state writer (`manifest_rev` + 1)
  **during the plan phase, before `/eados scaffold` renders `AGENTS.md`** — not at WP-00, which
  runs after the render; (c) the vendored `.eados-core/config/house-rules.md` is **not** the
  destination (it is the factory's overlay file inside the vendored copy; editing it is drift).
  The draft file is retired: the generated `CLAUDE.md` is the host adapter pointing at `AGENTS.md`.
- **Roadmap item 1.12** is thereby **amended** (proposed to the plan phase; its manifest text today
  is "root documents"): PRD import to `docs/prd/`, the README/CONTRIBUTING path-budget note, and
  the retirement of `.draft-specs/` once both moves have landed.

### API contract (`api` / `systemdesign`) — aligned with spec §5 (Public Interface)

- **Files and layout** — `inbox/<platform>/<conversation_key>/<capture_id>/{raw.json,
  capture.json, manifest.json}`; `export-root.json` (`export_layout_version: 1`, `producer {name:
  'chatclipper', version}`, `created_at`, `schema_versions`); `schema/{capture,raw-capture,manifest,
  anchor}.schema.v1.json`; `README.md`; `state/<platform>/<conversation_key>.anchor.json`
  (consumer-written). `conversation_key` = sanitized provider id (`[a-z0-9-_]`, <= 64) or `nokey`;
  `capture_id` = ULID. The extension never reads, modifies or deletes under `inbox/`.
- **Payloads** — `capture.json` = CanonicalCapture 1.0.0 (`capture`, `conversation`,
  `user_annotations`, `messages[]` with typed content blocks, `attachments[]`, `provenance`);
  `raw.json` = RawCapture 1.0.0 (untrusted `outer_html` per message + page/adapter/stats);
  `manifest.json` = Manifest 1.0.0 (`files[] {name, sha256, bytes}`, `schema_versions`,
  `producer {name: 'chatclipper', version}`, `write_verified`); `anchor.json` = Anchor 1.0.0 (3–10
  anchor messages; read-only, <= 256 kB, <= 90 days, hostile). Required vs optional: absent value =
  `null`, never omitted; unknown fields tolerated by consumers.
- **Internal protocol** — Side Panel ↔ content script Port v1 (`name: 'chatclipper'`): `detect` /
  `detect.result`, `capture.start {session_id, scan_scope, options, anchor}`, `capture.cancel`,
  `capture.progress`, `capture.chunk` (<= 50 messages or 512 kB, monotonic `seq`),
  `capture.complete`, `capture.error`; every message `{protocol_version: 1, type, payload}`
  Zod-validated on both sides.
- **Library and CLI** — `@mycelium-labs/chatclipper-core`: `normalize(raw, ctx): Result<CanonicalDraft>`,
  `canon`, `Hasher`, `sanitizeUrl`, `sanitizePathSegment` / `buildBundlePath`, `serialize`,
  `validateInvariants(capture): Violation[]`, `Result` / `DomainError`, ports `Clock`,
  `IdGenerator`, `Hasher`, `HtmlParser`; `validate-bundle <dir>` (Node, zero extension deps).
- **Error model** — coded errors with a localized remedy: AdapterNotFound, ConversationNotDetected,
  AdapterBroken, AdapterDegraded, ContentScriptUnavailable, ContentScriptDisconnected,
  HistoryLoadTimeout, ValidationFailed, ExportRootNotConnected, PermissionDenied,
  ExportRootLayoutAhead, WriteFailed, WriteVerificationFailed, DownloadInterrupted,
  IllegalStateTransition, Cancelled; coded warnings gap_detected, adapter_degraded, expand_failed,
  timestamp_relative_ignored, math_rendered_only, empty_message, unknown_block, branch_detected,
  truncated_by_provider, url_query_stripped, conversation_key_missing, anchor_not_found,
  anchor_invalid, write_not_verified. `Result<T, DomainError>` in domain/application; `throw` only
  for invariant bugs and at browser boundaries.
- **Versioning** — SemVer `schema_version` in every file; within a major only optional/nullable
  fields and documented enum values are added; consumers tolerate unknown fields and treat unknown
  block types as `unknown`. **MAJOR** = any removal, rename, type change, or a change to file
  names, folder layout, or completion semantics; a new major ships as `schema/*.v2.json` and
  consumers refuse unknown majors. Every contract change requires an ADR, a `schema_version` bump,
  regenerated JSON Schema, an updated `docs/CONTRACT.md`, and a green `validate-bundle`.

### Data & schema (`database`)

No relational store; no database profile (within ADR-0004's frame, a store would be a secondary
component and none is declared). Persistent state is:

- **The export folder** — write-once files, the contract above; the consumer owns everything after
  `manifest.json` is written. Migration policy: `export_layout_version` at the root (a higher
  version is refused, `ExportRootLayoutAhead`) and `schema_version` per file (additive within a
  major, new major = new schema files).
- **IndexedDB `chatclipper`, version 1** (non-authoritative except `handles`): `exports`
  (`[platform+conversation_key]` → last capture id, date, count, annotations — UX only),
  `pending_bundles` (`capture_id` → serialized bytes + step reached; purged after 7 days),
  `handles` (`export_root` → `FileSystemDirectoryHandle`), `logs` (ring buffer, 2,000 events).
  Migration policy: `idb` versioned upgrades; the cache **never decides what to export** (P1, spec
  ADR-023) and losing it is harmless by requirement (FR-020, Appendix B "Statelessness").
- **`storage.local`** — Zod-validated settings.

### Scalability budgets (`scalability`)

The `software` domain declares no hard NFR axis, so the factory's `nfr-budgets` audit gate skips
these; they are enforced by **the project's own CI** from roadmap item 6.1 (warnings in Phase 1,
blocking from Phase 2), as `spec.nfr_budgets` records:

| Axis | Metric | Target | Source |
|---|---|---|---|
| performance | capture of 600 already-loaded messages | <= 20 s (excluding provider waits) | NFR-001 |
| performance | normalize + hash per message (median) | <= 5 ms | NFR-001 |
| performance | longest task on the page UI thread | <= 50 ms | NFR-002 |
| performance | additional tab memory per 1,000 messages | <= 200 MB | NFR-003 |
| performance | content script, minified | <= 300 kB (`size-limit`) | NFR-017 |
| security | network API occurrences in the production bundle | 0 (`no-network-check`) | NFR-006 |
| portability | relative bundle path length in the export folder | <= 120 characters (max combination 113) | CON-006 / §9.3 |
| portability | tracked repo-relative path length | <= 160 characters (D2, owner's decision) | this RFC |
| quality | coverage: core / adapters | >= 90 % (canon, hash, slug, path, sanitizeUrl 100 %) / >= 80 % | NFR-010 |

### Algorithm sketch (`pseudocode`) — `CaptureEngine.runCapture` (spec §7.3–7.4)

```text
accumulated := ordered sequence (merge by overlap); passes := stable := gaps := 0
container := adapter.getScrollContainer(doc) ?? fail(ConversationNotDetected)
observe mutations on adapter.getMessageContainer(doc) → lastMutationAt

PASS():                                   -- EXPAND → WAIT → COLLECT → MERGE, for THIS window
  for el in adapter.findCollapsed(visible scope):
    if not retry(expandRetries, adapter.expand(el)): mark collapsed_unresolved + warn expand_failed
  if any expansion: await quiet(quietMs, cap 4×settleMs)
  window := adapter.collectVisibleMessages(doc)          -- mounted elements only, DOM order
  merge(accumulated, window):                             -- key: provider_message_id, else dom_text_sha256 + relative position
    overlap found → insert the new, replace common keys with the latest observation if expanded/changed
    no overlap and not adjacent to an end → gaps += 1; warn gap_detected(pass); scroll step /= 2 (min 0.2)
  onProgress(counts, pass, elapsed)

LOOP upward:
  guard: aborted → Cancelled; passes > maxPasses or elapsed > maxDurationMs → HistoryLoadTimeout
  PASS()
  if scanScope == tail and anchorFound(accumulated, anchor, anchorMin): break
  atTop := scrollTop <= 1 and not adapter.isHistoryLoading(doc)
  if atTop: stable := (new == 0) ? stable + 1 : 0; if stable >= stablePasses: break
  else: stable := 0; scrollTop -= scrollStepRatio × clientHeight
  await settle(settleMs); await quiet(quietMs); passes += 1

SWEEP downward (whole conversation in full; anchor → bottom in tail): PASS() at every step; final PASS()
effective scan_scope := (tail requested and anchor found before the top) ? tail : full
capture_status := gaps == 0 ? complete : incomplete
assign ordinals after the final merge; normalize · hash · validate in chunks with cooperative yield
```

Invariants: expansion happens **before** collecting the same window (expanding once at the end
misses messages a virtualized list has unmounted); an expanded re-observation replaces a collapsed
one; nothing but scrolling and expansion is ever performed on the page; a gap can only make the
bundle `incomplete`, never lose a message silently. Property test: overlapping contiguous windows
in mixed order reconstruct the exact sequence.

### Cross-cutting

- **Security (spec §11.1)** — page-content injection: RAW is never rendered, `outer_html` is
  declared untrusted; forged Port messages: isolated world, Zod on every message, random
  `session_id`, monotonic `seq`, Port opened by the Side Panel; exfiltration: CSP
  `connect-src 'none'` + `no-network-check` on the production bundle; credentials at rest: URL
  sanitization in RAW and canonical; path traversal: segment sanitization, FSA forbids `..`;
  hostile anchor file: Zod, 256 kB cap, read-only, ignored with a warning; supply chain: approved
  list §14.4 (+ axe-core, approved 2026-09-11), Dependabot, CodeQL, SBOM. Permissions: `storage`,
  `unlimitedStorage`, `sidePanel`, `downloads`, `scripting`, `activeTab`; host permissions for the
  three providers only; ISOLATED world; no remote code. **Any change to permissions, host
  permissions or CSP requires the PO's approval** (PRD §11.2–11.3) **and an ADR**, because it
  changes the public surface (generated `AGENTS.md` §7) — not because of the governance posture,
  which is `standard`.
- **Privacy** — no collection, transmission or telemetry; logs carry ids, counts, hashes, codes only
  (`LogSafeValue`); `PRIVACY.md` and the enterprise-account/ToS notice ship at v1.0.0.
- **Performance** — chunking and cooperative yield (NFR-002); budgets above; `size-limit`.
- **Accessibility** — keyboard, ARIA, AA contrast (NFR-015), scanned with axe-core in the Playwright
  e2e, blocking from the release milestone.
- **Resilience** — every failure path has a coded error and a tested recovery (retry write with
  byte-identical output; disconnection → `incomplete` bundle on confirmation; downloads
  `onChanged` ordering).

## Alternatives

**Skip the RFC and scaffold from the manifest (the classic one-shot path).** Rejected: D1 changes a
contract literal and D2 the workspace path; after WP-01 the first is a consumer-visible change under
§6.7 and after WP-00 the second is a mass move. Both are cheap today (`likely`, under the Q6
assumption) and expensive next month.

**One RFC per decision (four RFCs).** Rejected: the decisions are coupled — the package names live
at the layout path, the layout shim and the naming both become ADRs under the numbering rule, and
the PRD's home decides where the renamed literals get their Rev 3.1 — and the plan phase needs one
design document to cover.

**D1 alternatives.** *Keep `lce` everywhere*: rejected — the PO named the product, and `lce` inside
a public, polyglot contract would be a permanent fossil that every consumer hard-codes. *A second
scope `@chatclipper/*`*: rejected — a scope to own and register for no gain; `@mycelium-labs` is
the confirmed group path. *Generic names `@mycelium-labs/core`*: rejected — collides with any other
Mycelium package under the same scope. *Keep the PRD header's `LCA-SPEC-001`*: rejected — the file
name, §14.1 and Appendix F already use `SPEC-LCA-001`; renaming the file would break every
cross-reference for a header fix.

**D2 alternatives.** *The derived tree `src/main/typescript/it/mycelium-labs/chatclipper/*`
(the tech-lead's recommendation)*: **rejected by the owner** (2026-09-11) in favor of the shallow,
ecosystem-idiomatic `src/mycelium/` tree; the recommendation and its risks are recorded in D2.
*Root-level `packages/*` as in PRD §14.1*: rejected — it is the one location the generated
`consistency_lint` refuses outright (pattern code locations must sit under `src/`, `certain`), and
`src/mycelium/` gives the same shallow, idiomatic paths without that cost. *A shorter derived tree
via `group_path: mycelium`*: rejected by the owner — it keeps ADR-0002's shape but changes the
project's reverse-domain identity, which was answered as `it/mycelium-labs`. *A `packages` symlink
to any tree*: rejected — two names for one thing, and symlinked directories in git need Developer
Mode or elevation on Windows (`likely`). *Separate test mirrors (`src/test/...`)*: rejected —
ADR-0002 is superseded, so its test-mirror rule falls with it; colocated `test/` folders resolve the
package under test without extra workspace globs.

**D3 alternatives.** *Keep the PRD's numbering*: rejected — collides with the seeded ADR-0001/0002,
carries transferred gaps (010, 019, 020), and violates ADR-0001's sequential-immutable rule. *Import
the whole table as one ADR*: rejected — a 20-decision ADR cannot be superseded per decision, which
is the only reason to have ADRs. *Author all mapped decisions at WP-00*: rejected — fourteen of them
concern schemas, sinks and adapters that do not exist until WP-01…WP-03; an ADR authored before its
subject is a reservation by another name (L-0005).

**D4 alternatives.** *`docs/i18n/it/` for the PRD*: rejected — the generated contract defines that
directory as derived copies of English sources; the PRD is a source. *Translate the PRD into English
as the PRD*: rejected — 1,200 lines of PO-authored normative text would become a second source of
truth, and the manifest already carries the faithful English mapping the engineering spec is
generated from. *Keep `.draft-specs/`*: rejected — an ungoverned location with no owner in the
authority map. *Import the draft golden rules verbatim as house rules*: rejected — they name the
rejected `packages/` layout and the retired `@lce/*` scope, and house rules win over defaults, so a
verbatim import would override D1/D2. *Edit the rendered `AGENTS.md` §2 to record the language
exception*: rejected — no template hook exists, and a post-render edit does not survive a
re-render; a house rule lives in the manifest.

## Consequences

**Easier.** WP-00 has an unambiguous target: the workspace globs, the four package names, the
contract literals, the ADR list and order, the PRD's home, the house rules. The plan phase maps PRD
§15 onto the manifest's milestones 2–11 (already recorded) and references RFC-0001 from each. The
contract ships without a working-title fossil. The PRD stays the PO's document and keeps its id.

**Harder.** The generated ADR-0002 must be superseded at WP-00 and the rendered `AGENTS.md` §5 and
the generated lint's `version_file` configuration re-pointed; the scaffold's residual
`src/main` / `src/test` skeleton must be removed in that same PR; this repository gives up the
identical cross-project tree (the recorded dissent in D2); six ignore patterns and the Vitest /
Playwright configuration are needed; the clone root must stay under ~90 characters on Windows
without long-path support.

**Migration path.** None — greenfield. The rename touches only the PRD's text: the PO is asked to
issue **Rev 3.1** of SPEC-LCA-001 with (a) the D1 literals (§5.6, §6.5, §9.4, §13.5, Appendix E Q2)
and a `producer` schema for `export-root.json` (§6.1); (b) the **Appendix A** example
`manifest.json` (`"producer": { "name": "chatclipper" }`); (c) the header, ID row and closing line
aligned to `SPEC-LCA-001`; (d) the stale `dompurify` mention dropped. Until Rev 3.1 lands, the D1
table is the source for the literals (the scoped precedence rule in D4).

**State writer hand-off (design.md step 6).** `RFC-0001` is written to `delivery_state.refs.rfcs`
by the enterprise-architect (the manifest's state writer), together with the D2 scaffold inputs
`governance.capabilities.layered: false` and `spec.layers: []` (so the scaffold seeds no layer
folders in a tree WP-00 will remove). `toolchain.version_file` stays at the profile default: the
generated lint's configuration is re-pointed by WP-00 when ADR-0002 is superseded. The house rules
(D4) follow during plan, before scaffold.

**Roadmap amendments proposed to the plan phase** (the manifest's `spec.milestones` is the
producer's input, not this RFC's output): **1.6** — the `src/mycelium/*` workspace glob, removal of
the scaffold's residual `src/main` / `src/test` skeleton, the re-pointed lint `version_file`, the six
ignore patterns; **1.10** — the WP-00 ADR list of D3 (eleven spec decisions + D1, D2 as the ADR
superseding ADR-0002, D4); **1.12** — PRD import to `docs/prd/SPEC-LCA-001.md`, README/CONTRIBUTING
path-budget note, retirement of `.draft-specs/`; **new** — a `tools` unit test for the
160-character repo-path budget; **2.1** — `producer.name: 'chatclipper'` and the `export-root.json`
schema; **3.5** — spec ADR-005 authored at CP1; **M2 / M4** — the remaining spec ADRs at the WP that
realizes them (D3).

**Open questions carried to the PO (PRD Appendix E, plus two from this RFC), none blocking this
design.** Q1 — does the ingestion module watch `inbox/` and move or mark ingested bundles? (layout
of `inbox/`, quarantine rule). Q2 — its stack (Node or not → the `renormalize` CLI, WP-10). Q3 —
will it write anchors, and how often? (enables WP-09). Q4 — should its Markdown projection stay
compatible with Appendix D.6? Q5 — keep RAW always on (`includeRaw: true` proposed)? **Q6** — has
any consumer code been written against Rev 3.0's `producer.name: 'lce'`? (the D1 zero-cost
assumption). **Q7** — does the PO act as the `product-manager` role for `docs/prd/**`? They gate CP2
and Phase 3, not WP-00–WP-04.

## Approval

Filled by the approver **after** review and after the owner's decision on D1–D4 (this is the
`rfc-approved` gate's record). The owner approved the RFC and decided D1–D4 on 2026-09-11 (D1
"everywhere", D2 the literal `src/mycelium/` tree, D3 and D4 as written); the approver records it:

```
approved-by: tech-lead (2026-09-11)
```

Reviewers (structured findings addressed): reviewer — **resolved** (rev 2, 2026-09-11: B1, S1–S10,
N1–N9 — see the review log); enterprise-architect — **resolved** (cross-cutting lens: agreed D1 and
the rev-2 D2; the completeness items S1, S2 and S9 are addressed, and the owner's rev-3 D2 decision
is recorded with the tech-lead's dissent).

### Review log (rev 1 → rev 2 → rev 3, 2026-09-11)

| Id | Finding (short) | Resolution |
|---|---|---|
| Owner | D2 decided as the literal `src/mycelium/` tree, against the tech-lead's recommendation of the derived tree | Rev 3: D2 rewritten to the owner's decision; dissent recorded (position / alternative / risk); D3 layout ADR now supersedes ADR-0002; scaffold inputs `layered: false`, `layers: []`; S1 becomes moot (colocated tests) |
| B1 | D4 pointed house rules at a non-existent root file, after the render that consumes them, with content that would override D1/D2 | D4 rewritten: `governance.house_rules` in the manifest, written by the architect during plan before scaffold; rules rewritten to D1/D2/D4 names and stripped of layout/commands/precedence |
| S1 | Test mirrors outside the workspace glob cannot resolve the packages under test | Second and third workspace globs; mirrors are private packages with `workspace:*` devDependencies; Vitest projects rooted in the mirrors |
| S2 | 160-character budget underived; CI run called a proof; CON-006 non-sequitur | Budget derived (260 − 160 → ~90-character clone root, long-path opt-in documented at 1.12); CI is a smoke; CON-006 sentence removed |
| S3 | Spec id misattributed: the PRD header says `LCA-SPEC-001`, the file `SPEC-LCA-001` | Conflict recorded; canonical `SPEC-LCA-001`; header alignment added to the Rev 3.1 request |
| S4 | D1 missed Appendix A's fixture and `export-root.json`'s `producer`; zero-cost untagged | Both rows added; Appendix A in the Rev 3.1 request; claim tagged `likely` with Q6 |
| S5 | D3 re-scoped 1.10 (8 → 22 ADRs) as if already recorded; 002+014 merged | Presented as proposed plan-phase amendments; WP-00 keeps eleven spec decisions + D1/D2/D4; the rest at WP-01/WP-02/WP-03; 002 and 014 separate |
| S6 | "Addendum to ADR-0002" contradicts ADR immutability | A new ADR recording the shim, cross-referencing ADR-0002 (neither edited nor superseded) |
| S7 | Precedence rule skipped the manifest, was unscoped, dropped the PRD's terminal rule | Scoped rule: PRD on requirements, ADRs on D1–D4 until Rev 3.1; fixes run PRD → manifest → spec; irreconcilable → ask the PO |
| S8 | Language exception has no template hook in the generated `AGENTS.md` | Recorded as a house rule (§15); owner named |
| S9 | Generated `.gitignore` anchors at the root; nested tool outputs unmatched | Six unanchored patterns added to D2 and to the 1.6 amendment |
| S10 | Load-bearing claims untagged; the i18n freshness lint claim was inaccurate | Claims tagged; the lint sentence replaced by the categorical reason |
| N1 | `vitest.workspace.ts` deprecated | `vitest.config.ts` with `test.projects` (`likely`) |
| N2 | `version.ts` would sit outside every package | `toolchain.version_file: core/src/version.ts` set by the architect before scaffold |
| N3 | Downloads called a fallback | "first-class sink" |
| N4 | `e2e/` could read as a fifth layer | Marked a test-only component |
| N5 | Spec-mapping cell could break the adr-index lint if linked | Plain text, never a link |
| N6 | "the PO holds the product-manager role" unbacked | Tagged `guessing`; confirmation added as Q7 |
| N7 | `config/README.md` says §14 (stale) | The template (§15) is cited; the stale upstream text is noted for a later report |
| N8 | `refs.rfcs` hand-off unmentioned | State-writer hand-off paragraph added |
| N9 | Security ADR obligation attributed to posture | Attributed to the public-surface rule (§7) and PRD §11.3 |

## References

- Product specification: `.draft-specs/SPEC-LCA-001.md` Rev 3.0 (§5–§15, Appendices A–E); to become
  `docs/prd/SPEC-LCA-001.md` (D4).
- Project manifest: `orchestrator/project.yaml` (`spec.*`, `nfr_budgets`, `patterns`, `layers`,
  `milestones`, `governance.house_rules`, `toolchain.version_file`); bootstrap PR #1.
- Governance: `AGENTS.md` §2 (English on disk), §6 (git and PR contract), §10 (interaction
  contract); the generated ADR-0001 and ADR-0002 templates (`.eados-core/templates/docs/adr/`); the
  generated `AGENTS.md.tmpl` (§2 exceptions, §7, §15 house rules); the generated
  `consistency_lint.py` and `gitignore.tmpl`; `.eados-core/tools/render.py` (`HOUSE_RULES`,
  `IF_HOUSE_RULES`); the house-rules overlay (`.eados-core/config/README.md`); lesson L-0005
  (`.eados-core/learning/lessons.yaml`).
- Procedure: `.eados-core/orchestrator/commands/design.md`; RFC protocol
  `.eados-core/orchestrator/os/rfc/review-protocol.md`; routing advice for this work:
  frontier-reasoning / extra (labels `adr`, flag `decision-heavy`, steps `design` and `review`).
- Upstream factory issues found while framing this project: pgs-eados#387, pgs-eados#388.
