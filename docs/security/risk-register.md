# Risk register — mycelium-chat-clipper

> The **outcome** side of the security story (see [`README.md`](README.md)): the scored findings of
> each concrete audit run, owned by the **security-auditor** role. One section per audit; a finding
> keeps its id across audits until closed. Severity vocabulary: low / medium / high / critical
> (`.eados-core/orchestrator/os/risk/risk.yaml`). Confidence tags per `AGENTS.md` §12.

## Audit 2026-09-11 — bootstrap audit (phase `audit`, after PR #4)

### Scope and score

| Item | Value |
|---|---|
| Change audited | PR #4 (the scaffold render): 50 paths, 3,470 lines; cumulative PR #1–#4: 374 paths, 44,510 lines |
| Risk score (`risk_score.py --domain software`) | **critical** — factors: security-surface (`.github/**`, `tools/**`, `SECURITY.md`), large-change, wide-blast-radius; `certain` (tool output) |
| Security-auditor gate | **REQUIRED** → the deep audit and the STRIDE pass below were run |
| Traceability-lint (`traceability.py ROADMAP.md RFC-0001 --links links.yaml`) | **FAIL at first run — 1 dangling edge:** `pr-no-rfc: PR 1` (RR-01) → **OK after the same-day remediation** (links re-derived; all four PRs carry RFC-0001 + M1); RFC-0001 → M1…M12 coverage OK |
| Threat model | [`threat-model.md`](threat-model.md): 9 boundaries, 54 STRIDE cells, none blank; mitigated by design with the verifying test named, deferred to a named roadmap item, or open (RR-02, RR-04) |
| Source code audited | none exists yet — the audit covers the delivery surface (CI, dependencies, policy, secret hygiene, repository settings) and the design-time controls the PRD fixes |
| Secret hygiene | no secret-like tokens in the tracked tree, vendored factory included (`git grep` for GitHub/AWS/Slack/OpenAI token shapes and private-key headers, 2026-09-11) — `certain` |
| Policy | `SECURITY.md` present; private vulnerability reporting to `danielPoloWork` is the channel; supported window = latest `0.x`; severity → SemVer per `docs/workflow/maintenance.md` — `certain` (file read) |
| Dependencies | none yet; `.github/dependabot.yml` wired for `github-actions` and `npm`, grouped weekly, 1 open PR per ecosystem — `certain` |
| CI permissions | `ci.yml` `permissions: contents: read`; `release.yml` `contents: write` (needed to draft releases) — least privilege holds — `certain` |

### Findings

| Id | Severity | Component | Finding | Realistic impact | Mitigation | Owner / where | Status |
|---|---|---|---|---|---|---|---|
| **RR-01** | medium | delivery record (traceability graph) | PR #1 (the init bootstrap) carries no RFC reference: it predates RFC-0001 by design, so the `RFC → milestone → PR` graph has a dangling `pr-no-rfc` edge and the `traceability-lint` gate is red — `certain` | the `audit → migrate` transition is blocked while the edge stands; the graph cannot be used as an audit trail for the bootstrap PR | either (a) add a retroactive cross-link to PR #1's body — "RFC-0001 (authored in the design phase after this bootstrap; cross-link added at the 2026-09-11 audit)" — so `derive_links.py` picks it up, or (b) record the edge as an accepted exception in this register and re-run the lint from PR #2 onward | owner approved option (a) on 2026-09-11; the cross-link was added to PR #1's body with the original text preserved; `traceability-lint` re-run → OK | ✅ closed 2026-09-11 |
| **RR-02** | low | CI supply chain (`.github/workflows/ci.yml`, `release.yml`) | template-owned actions are SHA-pinned with version comments, but the profile's fragments are tag-pinned — `pnpm/action-setup@v4`, `actions/setup-node@v4`, and `actions/checkout@v6` in the profile's `lint` job (the template's checkout is `@3d3c42e… # v7.0.1`) — `certain` (grep) | a retargeted tag could run different code in CI; the blast radius is CI only (`contents: read`), no secrets beyond `GITHUB_TOKEN` | accepted trade-off upstream (EADOS ADR-0009 §3: profile actions stay tag-pinned and Dependabot-managed); tighten at WP-00 item 1.11 by SHA-pinning the three fragments and aligning the lint job's checkout with the template's | roadmap 1.11 | ▢ open, accepted until 1.11 |
| RR-03 | — | CI job permissions | `ci.yml` runs with `contents: read`; `release.yml` with `contents: write` | none — least privilege verified — `certain` | none needed | — | ✅ closed |
| **RR-04** | medium | repository settings (`main`) | `main` has **no branch protection and no ruleset** (API: "Branch not protected", 0 rulesets); the repository allows **merge commits and rebase** in addition to squash, and `delete_branch_on_merge` is off — `certain` (API reads 2026-09-11). Consequence already visible: PRs #1–#4 landed as **merge commits**, not squashes, so the "PR body becomes the permanent commit" property `AGENTS.md` §6.4 relies on did not hold for them | the contract's terminal guarantees (PR-required `main`, squash-only, no force-push) rest on owner discipline, not on the platform; a direct push or a merge-commit history is possible by accident | the owner's one-time setup in `docs/workflow/github-setup.md` §1: ruleset on `main` (require PR, block force-push and deletion, linear history), allow **squash only**, enable delete-branch-on-merge | owner (admin action; not the agent's) — the owner chose to keep it OPEN rather than accept it (2026-09-11) | ▢ open, owner action pending |
| **RR-05** | low | `.gitignore` | no ignore patterns for local secret files (`.env*`, `*.pem`, `*.key`); the generated file covers harness scratch, IDE and tool caches only — `certain` (grep) | an accidentally created local env or key file could be staged with `git add -A` | add `.env*`, `*.pem`, `*.key`, `*.p12` to `.gitignore` in WP-00 item 1.6 (which already edits the file) | roadmap 1.6 | ▢ open |
| **RR-06** | low | CI gates not yet wired | `no-network-check`, `size-limit`, CodeQL, SBOM and the a11y scan are design-time controls until their roadmap items (1.11, 6.1, 9.2, 9.4) — `certain` (rendered `ci.yml` read) | until then NFR-006/NFR-017 are promises, not gates; expected at bootstrap (no code) | ship them with their items; the bootstrap guard already skips toolchain jobs honestly rather than failing | roadmap 1.11 / 6.1 / 9.2 / 9.4 | ▢ open, tracked |
| **RR-07** | low | GitHub labels | `.github/labels.yml` is rendered but only `feat`, `docs`, `chore` exist on the repository; Dependabot's `labels: ["ci"]` / `["build"]` are **silently dropped** by GitHub when the label is missing — `certain` (label list + dependabot.yml comment) | the first bot PRs arrive unlabelled; the one-type-label PR contract cannot be met for `fix`, `ci`, `build`, `test`, `perf`, `refactor`, `security` PRs | import the canonical set (`github-setup.md` §2), or let the agent create the seven missing labels idempotently | owner approved on 2026-09-11; the seven labels were created with the canonical colors and descriptions (all ten now exist) | ✅ closed 2026-09-11 |

No finding is a confirmed, reproducible **defect** (the bug ledger stays empty) and none is a
**vulnerability** warranting an advisory (nothing to disclose; no code). RR-04 is the highest-value
fix: it converts the contract's guarantees from discipline into platform enforcement.

### Disposition

- **Bug ledger:** no entry (no reproducible defect).
- **Advisories:** none drafted.
- **Migration:** none needed (greenfield) — `audit → migrate` is not proposed; the audit is the
  standing gate before a release.
- **Next audit:** at WP-00 completion (roadmap 1.1–1.14) — the first source code, the `src/mycelium/`
  workspace, the superseding ADR and the CSP; re-check RR-02, RR-05, RR-06 there.
