# Fennec CI/CD – Centralized Reusable Workflows

**Status:** Implemented and validated in practice — the first consumer migration
(`model.atlas`) runs fully green against these reusables (see §0)
**Applies to:** all `eclipse-fennec/*` Gradle/bnd projects (first validated consumer:
`model.atlas`; `emf.util`, `emf.odata` still to migrate)
**Purpose:** Single source of truth for how Fennec CI is structured, plus a step-by-step
checklist to migrate an individual project repo onto the shared workflows. It documents the
reusable workflows that live in this repo (`eclipse-fennec/.github`) **and** the thin caller
workflows that go into each project repo.

---

## 0. Migration status (as of 2026-07-30)

| Repo | State |
|---|---|
| `eclipse-fennec/.github` | All 5 reusables implemented, including the single-build release extensions (`extra-gradle-tasks`, `artifact-paths`, `gradle-parallel`, `publish-java-version`) from #13 (`43722f4`). Pending: merge #13 and tag, so consumers can move their pins from the branch SHA to a release tag. |
| `fennec-model.atlas` | **First consumer, validated.** Branch `ci/reusable-workflows` (tip `f1f8376`) is fully green: license gate + Gradle 9.6.1/JDK 25 build + `testOSGi` + bndrun export checks (run 30546108930). Thin callers: `build.yml` (verify-only), `snapshot.yml`/`release.yml` (verify → release with `do-release` false/true, `publish-java-version: '25'`, exports via `extra-gradle-tasks`, jars via `artifact-paths` → repo-local `reusable-container.yml` builds the container images from the `release-jars` artifact, docker only after the Maven publish). Pinned to `eclipse-fennec/.github@43722f4`. |
| `emf.util`, `emf.odata` | Not yet migrated — follow the checklist in §10. |

Validated in practice: the full verify path incl. `extra-gradle-tasks` on PRs/feature
branches, and the artifact upload plumbing. **Not yet exercised** (happens on the first push
to `snapshot` after the model.atlas PR merges): the credentialed half of
`reusable-release.yml` (Maven snapshot deploy) and the container job's
`download-artifact` + docker push.

---

## 1. Motivation

Previously every project repo carried ~7 near-identical workflow files with **SHA-pinned**
action versions. `emf.util` and `emf.odata` were almost byte-for-byte identical; the only real
differences were an extra ignored `initial` branch (emf.util only) and a missing
`dependabot.yml` in odata. Bumping one action (e.g. `harden-runner`) meant editing the same
SHA in ~14 files across N repos.

Goal: **all build logic and all action versions live in exactly one place**
(`eclipse-fennec/.github`). Project repos contain only thin callers.

---

## 2. Branch and publishing model (authoritative)

| Trigger | License + Build + Test + osgiTest | Release | Docs (VitePress) |
|---|---|---|---|
| **PR** (any target branch) | ✅ | – | – |
| **Push to a feature branch** (anything except `main`/`snapshot`) | ✅ | – | – |
| **Push to `snapshot`** (development branch) | ✅ | → **Maven Snapshot** (`DO_RELEASE=false`) | ✅ build + deploy |
| **Push to `main`** (release branch) | ✅ | → **Maven Central** (`DO_RELEASE=true`) | ✅ build + deploy |

Key points:

- **License check, build, test and osgiTest run everywhere the same** (all branches + PRs).
  This is the shared verify part, with no credentials whatsoever.
- **Specific to `main` and `snapshot`** is only (a) the respective release step and
  (b) the docs build/deploy.
- **`main` = release branch → Maven Central.** **`snapshot` = development branch → Maven Snapshot.**
- The only difference between the two releases is the `DO_RELEASE` flag (`true` for Central,
  `false` for Snapshot). Both invoke the same Gradle `release` task.

### Credential scoping (key design decision)

The release step lives in its **own** reusable workflow file (`reusable-release.yml`). Only the
release job is passed the Sonatype/GPG secrets. The verify job (including the JDK-25 matrix) and
the docs job **never** see those secrets. As a result:

- everything untrusted (matrix build, tests, docs) runs without publishing credentials,
- only **one** JDK (21) publishes, not the whole matrix,
- the attack surface is minimal (least privilege).

---

## 3. Target architecture

### Central repo `eclipse-fennec/.github`

Reusable workflows under `.github/workflows/`:

| File | `on` | Purpose | Secrets |
|---|---|---|---|
| `reusable-verify.yml` | `workflow_call` | License gate → Gradle `clean build testOSGi perfTest`, matrix JDK [21,25] | none |
| `reusable-release.yml` | `workflow_call` | GPG import → `build testOSGi release` (JDK 21), `DO_RELEASE` via input | Sonatype + GPG |
| `reusable-docs.yml` | `workflow_call` | VitePress build + GitHub Pages deploy | none |
| `reusable-scorecard.yml` | `workflow_call` | OpenSSF Scorecard | none |
| `reusable-dependency-review.yml` | `workflow_call` | Dependency Review (PR) | none |

A centralized default license config (`.licenserc.yaml`) is discussed as an **open proposal**
in §7 — it is **not** part of the current setup; license config stays project-local for now.

> The license check is the **first, gating job inside `reusable-verify.yml`** — no longer a
> separate consumer file. This applies the license gate to every branch and PR without a
> duplicate run.

### Project repo (consumer), e.g. `eclipse-fennec/emf.util`

Thin callers under `.github/workflows/`:

| File | `on` | Calls |
|---|---|---|
| `build.yml` | push (except `main`,`snapshot`) + PR | `reusable-verify` |
| `snapshot.yml` | push `snapshot` | `reusable-verify` → `reusable-release`(do-release=false) → `reusable-docs` |
| `release.yml` | push `main` | `reusable-verify` → `reusable-release`(do-release=true) → `reusable-docs` |
| `docs.yml` | `workflow_dispatch` | `reusable-docs` (manual rebuild) |
| `scorecard.yml` | schedule / push main / branch_protection_rule | `reusable-scorecard` |
| `dependency-review.yml` | PR | `reusable-dependency-review` |
| `dependabot.yml` | – | config (github-actions + gradle) |

Chain for a push to `main`:

```
release.yml
  └─ verify   (reusable-verify)         license → build/test matrix [21,25]   [no secrets]
       └─ release (reusable-release)    GPG + gradle release, DO_RELEASE=true  [Sonatype+GPG]
            └─ docs (reusable-docs)     VitePress build + Pages deploy         [no secrets]
```

`snapshot.yml` is identical, only `DO_RELEASE=false` (→ Maven Snapshot).

---

## 4. Versioning / pinning the central workflows

Consumers reference the central workflows. Two options:

- **Recommended (keeps the Scorecard "Pinned-Dependencies" posture):** pin by **SHA** and let
  **Dependabot** (`package-ecosystem: github-actions`) bump them automatically. The actual
  version work happens only in `eclipse-fennec/.github`; consumer PRs are trivial SHA bumps.
- **Alternative (more convenient, but Scorecard complains):** a moving tag `@v1`. One central
  edit, no consumer PRs — but an unpinned reference.

In the examples below `@<PIN>` is a placeholder. When migrating, replace it with the concrete
SHA (recommended) or `v1`.

---

## 5. Central reusable workflows (repo `eclipse-fennec/.github`)

> The action SHAs match the current state of the existing repos. Future updates happen **only
> here**. See the actual files under `.github/workflows/` in this repo — the snippets below are
> the abridged shapes.

### 5.1 `reusable-verify.yml`

License gate + build/test/osgiTest/perfTest, matrix JDK [21,25], no credentials. Inputs:
`java-versions` (JSON array, default `["21","25"]`), `run-perf-tests` (bool, default true),
`extra-gradle-tasks` (string, default empty — additional tasks appended to the build
invocation, e.g. bndrun resolve/export checks, so PRs validate them too) and
`gradle-parallel` (bool, default true — bnd workspaces with resolve/export tasks may
need false).

The license job runs the header check against the consumer repo's own `.licenserc.yaml`.
(A centralized-default-with-local-override variant is an open proposal — see §7.)

### 5.2 `reusable-release.yml`

Credential-scoped publish. Inputs: `do-release` (bool, required), `publish-java-version`
(default `21`), `extra-gradle-tasks` (string, default empty — additional Gradle tasks run in
the **same build invocation**, e.g. bndrun exports), `artifact-paths` (string, default empty —
when set, the paths are uploaded as workflow artifact `release-jars`, with
`if-no-files-found: error`) and `gradle-parallel` (bool, default true). Secrets:
`CENTRAL_SONATYPE_TOKEN_USERNAME`, `CENTRAL_SONATYPE_TOKEN_PASSWORD`, `GPG_PASSPHRASE`,
`GPG_KEY_ID`, `GPG_PRIVATE_KEY`. Runs GPG import → `./gradlew build testOSGi
<extra-gradle-tasks> release` with `DO_RELEASE=${{ inputs.do-release }}` → keyring cleanup.
Publishes with a single JDK.

**Single-build consistency:** tests, extra tasks and the release all run in one Gradle
invocation, so the jars that are tested, exported and published are identical. The extra
tasks are ordered **before** the `release` task, so a failing export aborts the build before
anything is published to Maven.

Repos that additionally build container images from bnd export outputs (e.g. `model.atlas`)
pass their export tasks via `extra-gradle-tasks` and the resulting jar paths via
`artifact-paths`; a downstream container job then fetches the `release-jars` artifact with
`actions/download-artifact` instead of rebuilding the workspace. This guarantees the jars
published to Maven and the jars baked into the images come from the **same build**. The same
export tasks can be passed to `reusable-verify`'s `extra-gradle-tasks` so PRs and feature
branches validate the bndrun exports as well (without any artifact upload).

### 5.3 `reusable-docs.yml`

VitePress build + GitHub Pages deploy. Input: `node-version` (default `20`). The repo-specific
publish path slug comes from each repo's `docs-site/config.mts` (via `DOCS_BRANCH`), not from
this workflow. The deploy job holds `pages: write` + `id-token: write`.

### 5.4 `reusable-scorecard.yml`

OpenSSF Scorecard analysis. `permissions: read-all` at workflow level; the analysis job holds
`security-events: write`, `id-token: write`, `contents: read`, `actions: read`.

### 5.5 `reusable-dependency-review.yml`

`actions/dependency-review-action` with `fail-on-severity: high`, PR comment on failure.

---

## 6. Consumer workflows (into each project repo)

`@<PIN>` = SHA (recommended) or `v1`. `secrets: inherit` forwards repo/org secrets only to the
called reusable; since only `reusable-release.yml` declares `secrets:`, verify and docs receive
**no** publishing credentials (credential scoping is preserved).

### 6.1 `build.yml` (feature branches + PR)

```yaml
name: CI Build
on:
  push:
    branches-ignore:
      - main
      - snapshot
  pull_request:
    branches:
      - '*'
permissions:
  contents: read
concurrency:
  group: ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
jobs:
  verify:
    uses: eclipse-fennec/.github/.github/workflows/reusable-verify.yml@<PIN>
```

### 6.2 `snapshot.yml` (development branch → Maven Snapshot)

```yaml
name: Snapshot Build
on:
  push:
    branches:
      - snapshot
permissions:
  contents: read
  pages: write
  id-token: write
concurrency:
  group: snapshot-publish
  cancel-in-progress: false
jobs:
  verify:
    uses: eclipse-fennec/.github/.github/workflows/reusable-verify.yml@<PIN>
  release:
    needs: verify
    uses: eclipse-fennec/.github/.github/workflows/reusable-release.yml@<PIN>
    with:
      do-release: false
    secrets: inherit
  docs:
    needs: release
    uses: eclipse-fennec/.github/.github/workflows/reusable-docs.yml@<PIN>
```

### 6.3 `release.yml` (release branch → Maven Central)

```yaml
name: Release Build
on:
  push:
    branches:
      - main
permissions:
  contents: read
  pages: write
  id-token: write
concurrency:
  group: release-publish
  cancel-in-progress: false
jobs:
  verify:
    uses: eclipse-fennec/.github/.github/workflows/reusable-verify.yml@<PIN>
  release:
    needs: verify
    uses: eclipse-fennec/.github/.github/workflows/reusable-release.yml@<PIN>
    with:
      do-release: true
    secrets: inherit
  docs:
    needs: release
    uses: eclipse-fennec/.github/.github/workflows/reusable-docs.yml@<PIN>
```

### 6.4 `docs.yml` (manual rebuild)

```yaml
name: Documentation
on:
  workflow_dispatch:
permissions:
  contents: read
  pages: write
  id-token: write
concurrency:
  group: pages
  cancel-in-progress: false
jobs:
  docs:
    uses: eclipse-fennec/.github/.github/workflows/reusable-docs.yml@<PIN>
```

### 6.5 `scorecard.yml`

```yaml
name: OpenSSF Scorecard
on:
  branch_protection_rule:
  schedule:
    - cron: '27 4 * * 1'
  push:
    branches:
      - main
permissions: read-all
concurrency:
  group: scorecard-${{ github.ref }}
  cancel-in-progress: true
jobs:
  scorecard:
    permissions:
      security-events: write
      id-token: write
      contents: read
      actions: read
    uses: eclipse-fennec/.github/.github/workflows/reusable-scorecard.yml@<PIN>
```

### 6.6 `dependency-review.yml`

```yaml
name: Dependency Review
on:
  pull_request:
    branches:
      - '*'
permissions:
  contents: read
  pull-requests: write
concurrency:
  group: dependency-review-${{ github.ref }}
  cancel-in-progress: true
jobs:
  dependency-review:
    uses: eclipse-fennec/.github/.github/workflows/reusable-dependency-review.yml@<PIN>
```

### 6.7 `dependabot.yml` (currently missing in odata!)

```yaml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
  - package-ecosystem: "gradle"
    directory: "/"
    schedule:
      interval: "weekly"
```

> Note: for SHA-bumping the central reusable references (`@<PIN>`), `eclipse-fennec/.github`
> also carries a `github-actions` Dependabot config, and Dependabot in the consumers updates
> the `uses:` SHA references in turn.

---

## 7. License configuration (`.licenserc.yaml`) — OPEN PROPOSAL, not implemented

> **Status:** open question, deliberately **not** part of the current setup. Today each repo
> keeps its own `.licenserc.yaml`. This section records the analysis and a possible future
> approach so the decision can be made separately.

The license header check uses `apache/skywalking-eyes`, which reads a **single**
`.licenserc.yaml` (no field-level merge). Across the org these files genuinely differ:

- most Foundation repos share the same EPL-2.0 header (only the copyright year varies, and the
  matcher `pattern` omits the year line, so that difference is harmless),
- a few older repos still carry a "Data In Motion" header,
- and `paths-ignore` is genuinely per-repo (e.g. `emf.util` ignores all `**/src-gen/**`, while
  `emf.odata` validates `src-gen` and ignores only `**/src-gen-parser/**`).

Because a partial merge is impossible and the variance is high, the only workable central model
would be a **central default with whole-file local override**: `eclipse-fennec/.github` holds a
shared `.licenserc.yaml`; `reusable-verify.yml` uses a consumer's repo-local file if present,
otherwise the central default. Trade-off to weigh: given nearly every repo has a distinct
`paths-ignore` (and three still use the DiM header), most repos would keep a local file anyway,
so the central default would mostly serve new/cleaned-up repos — modest benefit for a bit of
extra workflow mechanics. **Recommendation:** decide this only if/when the org intends to
converge repos onto a single Foundation header; until then, keep license config project-local.

> Aside (out of scope): `fennec-model.atlas/.licenserc.yaml` contains a literal TAB that makes
> it invalid YAML — worth fixing in that repo separately.

---

## 8. Secrets & permissions – overview

**Org/repo secrets** (must exist in each consumer repo or at org level):

- `CENTRAL_SONATYPE_TOKEN_USERNAME`
- `CENTRAL_SONATYPE_TOKEN_PASSWORD`
- `GPG_PASSPHRASE`
- `GPG_KEY_ID`
- `GPG_PRIVATE_KEY`

These flow **only** via `secrets: inherit` in `release.yml`/`snapshot.yml` into
`reusable-release.yml`. Verify and docs never receive them.

**Permissions:** a job that uses `uses:` (a reusable workflow) **cannot** set job-level
`permissions` — the ceiling comes from the **caller's workflow-level** `permissions`. Therefore:

- `snapshot.yml`/`release.yml`/`docs.yml` declare `contents: read`, `pages: write`,
  `id-token: write` at workflow level (for the docs deploy).
- The reusables declare the finer job-level permissions themselves.
- `scorecard.yml` is the exception — the calling job needs the Scorecard scopes, so they sit
  directly on `jobs.scorecard` (see §6.5).

---

## 9. GitHub prerequisites for org-internal reusable workflows

- Repo `eclipse-fennec/.github` must exist (it does — it already carries the profile README).
- All `eclipse-fennec` repos (this one and all consumers) are **public**, so calling these
  reusables via `uses:` works **without** any org "reusable workflow access" setting — that
  setting only governs private/internal repos.
- The action set is unchanged from the workflows already running today, so no additional org
  Actions-policy allowlisting is needed.
- Reusable-workflow nesting is capped at 4 levels (here: 1 level, non-issue).

---

## 10. Migration checklist (per project repo)

1. **Prerequisite:** `eclipse-fennec/.github` contains the 5 reusables (§5). Choose `@<PIN>`
   (the `.github` commit SHA, recommended).
2. Create a branch in the project repo (repos are PR-only).
3. Replace the old workflows with the thin callers from §6:
   - overwrite `build.yml`, `snapshot.yml`, `release.yml`, `docs.yml`, `scorecard.yml`,
     `dependency-review.yml`.
   - **delete** the standalone `license.yml` (the license gate now lives in `reusable-verify`).
4. Ensure `dependabot.yml` exists (§6.7) — it is missing in `emf.odata`.
5. Set `@<PIN>` in all 6 callers to the chosen SHA/tag.
6. License config: keep the repo's existing `.licenserc.yaml` as-is (license config stays
   project-local; a possible central default is an open proposal — see §7).
7. Repo-specific checks:
   - `docs-site/config.mts` sets the correct base-path slug (`/emf.util/`, `/emf.odata/`, …).
     The docs workflow is generic; the slug lives here.
   - `docs-site/package-lock.json` exists (npm cache path in the docs workflow).
   - Gradle tasks `build`, `testOSGi`, `perfTest`, `release` exist.
   - All 5 secrets from §8 are set (org or repo level).
8. **`initial` branch decision:** the old emf.util workflows special-cased `initial`. In the new
   model `initial` is an ordinary feature branch (verify only). If it should not build, add it to
   `branches-ignore` in `build.yml`.
9. Test order:
   - open a PR → only `verify` + `dependency-review` should run.
   - merge to `snapshot` → `verify` → `release`(Snapshot) → `docs`.
   - merge to `main` → `verify` → `release`(Central) → `docs`.
10. Verify branch protection on `main`/`snapshot` (PR-only, required checks = the `verify` jobs).

---

## 11. Intentional deviations from the previous state

- **`license.yml` is dropped** as a standalone consumer file (now part of `reusable-verify`).
- **Release publishes with a single JDK** (default 21, configurable via
  `publish-java-version` — model.atlas uses 25), no longer as a combined "build+release" matrix job.
  The full matrix (21+25) still runs in verify — just without credentials. This is the
  credential-scoping improvement.
- **`initial` is no longer special-cased** (see §10.8).
- `--scan` (Gradle build scan), which the old snapshot build set, is intentionally omitted so
  both release paths are identical. Add it as an input to `reusable-release.yml` if wanted.
