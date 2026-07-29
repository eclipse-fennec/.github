---
name: fennec-quality-review
description: Code-quality review of an Eclipse Fennec OSGi repository against SOLID principles (adapted to OSGi/DS) and Eclipse Foundation guidelines (naming, API/internal hygiene, license headers, API evolution). Produces a ranked markdown findings report. Use when asked for a quality review, SOLID review, or Eclipse-guidelines audit of a fennec project — including a pre-commit review of only the pending changes (`changes` argument) or a follow-up that re-checks whether a previous review's findings were resolved (`followup` argument).
---

# Fennec quality review

Review the current repository's code quality against two rule sets bundled with this skill:

- `references/solid-osgi.md` — SOLID principles translated to OSGi/DS/bnd reality
- `references/eclipse-guidelines.md` — Eclipse Foundation / Platform conventions (distilled snapshot; source URLs listed inside)

**Read both reference files first, before looking at any code.** They define what counts as a finding and — just as important — the fennec idioms that must NOT be flagged.

This is a **read-only review**: never modify source code during a run. The deliverable is the report.

## Arguments

`$ARGUMENTS` may contain, in any order:

- `changes` (alias `diff`) — review only the pending git changes instead of bundles (pre-commit use). Optionally followed by a ref: `changes main` reviews everything on this branch since `merge-base HEAD main`; bare `changes` reviews the working tree (staged + unstaged + untracked).
- `deep` — run the multi-agent workflow audit instead of the default single-pass review.
- `refresh` — do NOT review. Re-fetch the source URLs listed at the top of `references/eclipse-guidelines.md` (WebFetch), update the distilled snapshot in place (keep the same structure, note the refresh date), report what changed, and stop.
- `followup` (alias `previous`) — also check the previous review: re-verify its findings against the current code and use the still-open ones as first hints. Optionally followed by a path to a specific report; otherwise use the most recent `docs/reviews/quality-review-*.md`. If none exists, say so and run a normal review. See "Follow-up pass" below.
- anything else — a **scope**: bundle directory name(s), path(s), or a substring matching bundle names (e.g. `management.git` scopes to all `*management.git*` bundles). No scope = whole repository.

## Procedure — quick mode (default)

1. Read both reference files.
2. Discover the review targets: bundle projects are directories containing a `bnd.bnd` file. Apply the scope argument. Exclude generated code (`generated/`, EMF-generated model code — packages with `@generated` javadoc tags get structural findings suppressed, but license-header and API-export findings still apply), and exclude `*.tests` bundles from SOLID structure checks (test code still gets header/naming checks).
3. **Repo-level pass** (always, regardless of scope): check the release-readiness section of `references/eclipse-guidelines.md` — root documents, license-check + Dash workflows, `DEPENDENCIES` (including `restricted` entries), CI/publishing shape, baselining. These are checked once against the repo root and `cnf/`, not per bundle.
4. For each target bundle, review against every dimension in the two reference files. Read `bnd.bnd` first (exports, private packages, buildpath) — several checks are about the bundle manifest, not the Java.
5. **Verify before reporting**: re-read the exact code location for every candidate finding and drop anything that is speculative, generated-code noise, or an allowed fennec idiom. Prefer fewer, solid findings over volume.
6. Write the report (format below) and print a short terminal summary (counts by severity + top 3 findings).

For a whole-repo quick run on a large workspace, prioritize: non-test bundles' API surface (exported packages), DS components, and bnd.bnd files — say explicitly in the report which bundles got full reads vs. skimmed.

## Procedure — changes mode (`changes` / `diff`)

Pre-commit review of the delta only; same rule sets, different scoping:

1. Read both reference files.
2. Collect the changed files: bare `changes` → `git status --porcelain` (staged, unstaged, untracked); `changes <ref>` → `git diff --name-only $(git merge-base HEAD <ref>)..HEAD` plus the working tree. Keep source-relevant files (`.java`, `bnd.bnd`, `*.bndrun`, `cnf/**`, workflow/config files); list anything ignored (docs, generated) in the report's skipped section.
3. For each changed file, read the **full file** (a diff hunk without context produces false positives) plus the diff itself, but only report findings that are **in changed code or directly caused by the change** (e.g. the change makes an existing sibling impl violate LSP, or an edited `bnd.bnd` now exports an impl package). Pre-existing issues in untouched code are out of scope — at most one `info` note if something severe is spotted incidentally.
4. Run the repo-level release-readiness pass ONLY if the change touches root documents, `.github/workflows/`, `.licenserc.yaml`, `tools/`, or `cnf/` — and then only the touched aspects. New `.java` files always get the license-header check.
5. Verify each candidate finding against the actual code before reporting (as in quick mode).
6. Output: **terminal report only** by default — findings in the quick-mode format, printed, most-severe first, ending with a one-line verdict ("ready to commit" / "N findings worth fixing first"). Write the report file too only when the user asks or when there are ≥ 10 findings.

`changes` composes with `deep` (`changes deep <ref>`): fan out one agent per changed bundle instead of per file — only worth it for large branch diffs.

## Follow-up pass (`followup`)

Composes with every mode; runs **before** the fresh review so the old findings can steer it:

1. Read the previous report and extract its findings (id, severity, location, claim).
2. Re-verify each one against the **current** code — never trust stale line numbers, re-locate the code first (it may have moved or been deleted). Classify: **resolved**, **still open**, **partially resolved** (fixed at the reported spot but the same root cause remains elsewhere), or **no longer applicable** (code removed / rule changed on `refresh`).
3. Findings from the previous report that fall outside the current scope/changes arguments are still status-checked (cheap), but only re-reported in full if still open AND in scope — otherwise one line in the status table is enough.
4. Run the normal review for the mode, using still-open findings as hints for where to look harder (same bundle, same author pattern, sibling classes) — but every reported finding must be independently verified as usual; do not copy text from the old report.
5. Report additions: a **"Previous findings status"** table right after the Summary (previous id → status → one-line note, with resolved/open counts), and still-open findings re-listed as regular findings marked `carried over from <date>` (keep their evidence fresh). A finding open across ≥2 consecutive reviews gets an explicit note — recurring findings are a signal the suggested fix isn't landing.

## Procedure — deep mode (`deep`)

Use the **Workflow tool** to orchestrate (this skill is your authorization to call it):

1. Scout inline: list target bundles (as in quick mode step 2), and run the repo-level release-readiness pass yourself (quick mode step 3) — it is cheap and doesn't need an agent.
2. Phase "Review": one reviewer agent **per bundle** (pipeline, structured output schema: findings with `bundle`, `file`, `line`, `category`, `severity`, `claim`, `evidence`, `fix`). Each agent gets: the bundle path, and instructions to read this skill's two reference files (`~/.claude/skills/fennec-quality-review/references/`) before reviewing.
3. Barrier + dedup findings across bundles (same rule violated by the same root cause in many files = ONE systemic finding listing occurrences).
4. Phase "Verify": one adversarial verifier agent per deduped finding, prompted to REFUTE it against the actual code and the reference files' allowed-idioms list; drop refuted findings.
5. Synthesize into the same report format as quick mode, plus a "Systemic issues" section for cross-bundle patterns.

## Report format

Write to `docs/reviews/quality-review-<YYYY-MM-DD>.md` in the repo root (create the directory if needed; get the date from `date +%F`, append `-2`, `-3`… if the file exists). Structure:

```markdown
# Quality review — <repo name> — <date>
Mode: quick|deep · Scope: <scope or "whole repo"> · Rule sets: SOLID/OSGi + Eclipse Foundation (see skill references)

## Summary
<counts table: severity × category, one paragraph of overall assessment>

## Previous findings status   (followup mode only)
<table: previous id · status (resolved / still open / partial / n/a) · note — plus resolved/open totals>

## Findings
### F1 · <severity> · <category> · <bundle>
- **Where:** path/File.java:line
- **What:** <one-sentence claim>
- **Why it matters:** <consequence>
- **Suggested fix:** <concrete, minimal>
...

## Systemic issues        (deep mode, or when a pattern spans ≥3 bundles)
## Skipped / not reviewed (anything scoped out or only skimmed)
```

Severities: **blocker** (broken API contract, missing/wrong license header on shipped code, exported internal package), **major** (clear SOLID violation with concrete blast radius, unversioned exported API package, DS lifecycle leak), **minor** (naming, javadoc gaps on API, ordering conventions), **info** (observations, improvement ideas). Sort findings most-severe first. Every finding needs a real `file:line` — no "throughout the codebase" findings outside the Systemic section.
