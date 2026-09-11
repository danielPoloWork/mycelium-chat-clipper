# Translation Status

Canonical source language: **en** — the on-disk sources are English; the
rows below track *derived* translations under `docs/i18n/<code>/`.

**Legend** — `pending`: not yet translated (ignored by the freshness check) · `translated`:
current as of the recorded source commit · the lint flags it stale automatically once the
English source advances past that commit.

## Tracked pages

One row per `(source page × target language)`. When a translation lands, change its status
to `translated` and set the **Source commit** to the SHA of the English page it was made
from. The link in a `translated` row is resolved **relative to this file's directory**
(`docs/i18n/`) and must point at the English source (e.g. `../../README.md`).

| Source page | Language | Status | Source commit |
|---|---|---|---|
| — | — | `pending` | — |

## Target languages

- **Italian** (`it`) — `docs/i18n/it/`
- **Chinese** (`zh`) — `docs/i18n/zh/`
- **Japanese** (`ja`) — `docs/i18n/ja/`

