# fennec-quality-review

A Claude Code skill that reviews an Eclipse Fennec OSGi repository against
**SOLID principles** (adapted to OSGi/DS/bnd) and **Eclipse Foundation
guidelines** — naming and API/internal hygiene, EPL-2.0 license headers,
API evolution/versioning, and fennec release-readiness (root documents,
IP Dash / `DEPENDENCIES`, CI/publishing shape, baselining).

The review is **read-only**: it never modifies code, it produces a findings
report. Every finding carries a `file:line` reference, a severity
(blocker / major / minor / info), and a concrete fix suggestion.

## Installation

Copy this folder into your skills directory:

- personal (all repos): `~/.claude/skills/fennec-quality-review/`
- or per repo, checked in: `<repo>/.claude/skills/fennec-quality-review/`

It appears as `/fennec-quality-review` in your next Claude Code session.

## Commands

| Command | What it does |
|---|---|
| `/fennec-quality-review` | Quick single-pass review of the whole repo + repo-level release-readiness checks. Writes `docs/reviews/quality-review-<date>.md`. |
| `/fennec-quality-review <scope>` | Same, limited to bundles matching `<scope>` (a path or a bundle-name substring, e.g. `management.git`, `rest`). |
| `/fennec-quality-review changes` | Pre-commit review of your pending work only (staged + unstaged + untracked files). Prints findings to the terminal and ends with a "ready to commit" verdict. |
| `/fennec-quality-review changes <ref>` | Reviews everything the current branch adds since `merge-base HEAD <ref>` — e.g. `changes main` before opening a PR. |
| `/fennec-quality-review refresh` | No review — re-fetches the guideline source URLs and updates the bundled snapshot in `references/`. |

### `deep` modifier

Add `deep` to any review to escalate from a single pass to a multi-agent
audit: one reviewer agent per bundle, cross-bundle deduplication, then an
adversarial verification pass that tries to refute each finding before it is
reported. Much more thorough, much more expensive — use it for periodic
audits, not everyday checks.

Examples:

```
/fennec-quality-review deep                     # full-repo audit
/fennec-quality-review management.git deep      # deep audit of selected bundles
/fennec-quality-review changes deep main        # deep review of a large branch diff
```

### Rules of thumb

- `changes` before you commit
- a scoped quick run while working on a subsystem
- `deep` for the periodic whole-project audit
- `refresh` when the guideline docs change upstream

## How it decides what to flag

The rules live in two files the review reads on every run:

- [`references/solid-osgi.md`](references/solid-osgi.md) — SOLID translated
  to OSGi/DS reality (e.g. DIP = depend on service APIs via `@Reference`,
  never import a foreign `*Impl` package), plus DS lifecycle correctness
  (deactivate must undo activate, no blocking in `activate()`, …).
- [`references/eclipse-guidelines.md`](references/eclipse-guidelines.md) —
  Eclipse Platform coding/naming conventions, license headers (matching the
  repo's `.licenserc.yaml`), API vs `internal` hygiene translated to bnd
  exports, API evolution / semantic versioning, and the fennec
  release-readiness checklist (reference implementation: `emf.osgi`).

Both files contain an explicit **do-not-flag list** (EMF-generated code, the
`emf.nsURI` property, documented read-only backends, …) so intentional
fennec idioms don't show up as findings.

### Updating the guidelines

The source URLs are listed at the top of `references/eclipse-guidelines.md`
(Eclipse Platform docs, the fennec release guide, the IP Dash guide). To add
your own: append the URL to that list and run
`/fennec-quality-review refresh` — the snapshot is regenerated from all
listed sources. Rule tweaks (severities, extra idioms to ignore) can be
edited directly in the two reference files.
