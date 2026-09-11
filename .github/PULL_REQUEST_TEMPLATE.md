## Summary

One or two sentences: what changes and why it matters.

## Motivation

Link to the spec section, ADR, roadmap item, or issue that prompted this work.

## Changes

- bulleted list of meaningful changes (not a file list)

## Design Patterns

- list every pattern adopted/refined/rejected in this PR, with a one-line rationale and a
  link to the ADR.
- if none, write "None — straightforward implementation."

## Verification

- [ ] Builds cleanly on the full CI matrix (Linux and Windows on Node 22 and 24 LTS; Chromium (Chrome/Edge) >= 120 for the extension)
- [ ] Unit tests pass; new/changed behavior covered (≥ 80% line)
- [ ] `Prettier` clean; `ESLint (typescript-eslint type-aware, eslint-plugin-boundaries for the layer matrix) + tsc --noEmit --strict + commitlint` clean on the diff
- [ ] tsc --strict (type soundness); eslint --max-warnings 0; no-network-check (static scan of the production bundle, content script included); size-limit (content script <= 300 kB); pnpm audit; CodeQL green (where applicable)
- [ ] Benchmark numbers attached (when perf-relevant)
- [ ] `python tools/consistency_lint.py` passes

## Documentation Impact

- [ ] README.md updated (if user-facing surface changed)
- [ ] ROADMAP.md checkbox flipped
- [ ] ADR added/updated (if a non-trivial design decision was made)
- [ ] docs/patterns/README.md updated (if a pattern was introduced, refined, or rejected)
- [ ] Spec updated (if behavior diverges from `docs/specs/`)
- [ ] CHANGELOG.md updated (for user-visible changes)
- [ ] PR metadata set — assignee (the owner), one type label, release milestone, project (where present)

## Lesson

<!-- Optional, one line. A generalizable rule the next contributor should inherit — captured
here at review time, while the knowledge is hot and the human gate is already open. Squash-merge
takes this PR body as the permanent commit on `main`, so a merged lesson is
owner-approved by construction. Write "none" if there is nothing durable to carry forward. -->

Lesson: none
