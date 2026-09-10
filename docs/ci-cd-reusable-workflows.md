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

## 0. Migration status (as of 2026-09-08)

| Repo | State |
|---|---|
| `eclipse-fennec/.github` | Seven reusables implemented: verify, test-report, release, docs, scorecard, dependency-review and — new, not yet in a release tag — `reusable-container.yml` (§5.7). Latest release: `v1.3.0` (`2a98ad6`), moving tag `v1` points at it. Adding the container workflow is a new optional reusable, so it goes out as a **minor** bump (`v1.4.0`) per §4.1, followed by re-pointing `v1` (§4.2). |
| `fennec-model.atlas` | **First consumer, validated.** Branch `ci/reusable-workflows` (tip `f1f8376`) is fully green: license gate + Gradle 9.6.1/JDK 25 build + `testOSGi` + bndrun export checks (run 30546108930). Thin callers: `build.yml` (verify-only), `snapshot.yml`/`release.yml` (verify → release with `do-release` false/true, `publish-java-version: '25'`, exports via `extra-gradle-tasks`, jars via `artifact-paths` → repo-local `reusable-container.yml` builds the container images from the `release-jars` artifact, docker only after the Maven publish). Pinned to `eclipse-fennec/.github@43722f4`. |
| `event.atlas`, `model.atlas`, `data.atlas`, `dcat.atlas` | On the central verify/release reusables, but each still carries its **own** repo-local `reusable-container.yml` — the four copies this repo's `reusable-container.yml` generalizes (input mapping per repo in §5.7, wiring in §6.8). Migrating them means swapping the `uses:` to the central workflow and deleting the local file. |
| `emf.util`, `emf.odata` | Not yet migrated — follow the checklist in §10. |

Validated in practice: the full verify path incl. `extra-gradle-tasks` on PRs/feature
branches, the artifact upload plumbing, and — through the repo-local copies in the four atlas
repos — the whole `release-jars` → `download-artifact` → docker push chain that
`reusable-container.yml` now centralizes. **Not yet exercised:** the central container
workflow itself, on the first `snapshot` push of the first repo that switches to it.

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
| `reusable-verify.yml` | `workflow_call` | License gate → Gradle `clean build testOSGi perfTest`, matrix JDK [21,25], JUnit XML as artifact + job summary | none |
| `reusable-test-report.yml` | `workflow_call` | Opt-in: renders the verify artifacts as one check run per JDK (needs `checks: write` from the caller) | none |
| `reusable-release.yml` | `workflow_call` | GPG import → `build testOSGi release` (JDK 21), `DO_RELEASE` via input | Sonatype + GPG |
| `reusable-docs.yml` | `workflow_call` | Shared-theme drift gate → VitePress build + GitHub Pages deploy | none |
| `reusable-scorecard.yml` | `workflow_call` | OpenSSF Scorecard | none |
| `reusable-dependency-review.yml` | `workflow_call` | Dependency Review (PR) | none |
| `reusable-container.yml` | `workflow_call` | Docker image build + multi-arch push to Docker Hub and GHCR, from the `release-jars` artifact | Docker Hub |

Shared docs assets under `docs-theme/`:

| File | Synced into the consumer as | Purpose |
|---|---|---|
| `index.ts` | `docs-site/docs/.vitepress/theme/index.ts` | Theme entry; renders the Eclipse Foundation footer in `layout-bottom` |
| `custom.css` | `docs-site/docs/.vitepress/theme/custom.css` | Fennec brand palette; hides VitePress' own footer |
| `EclipseFooter.vue` | `docs-site/docs/.vitepress/theme/EclipseFooter.vue` | The footer component |
| `public/eclipse-foundation-*.svg` | `docs-site/docs/public/` | Official EF logos, light and dark |

These files are **committed in every consumer repo** (so `npm run docs:dev` works without a
sync step) but **owned here**. `reusable-docs.yml` compares them byte-for-byte before the
build and fails on any difference — see §5.3. Theme changes therefore go into this repo
first, then get synced outwards.

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
| `snapshot.yml` | push `snapshot` | `reusable-verify` → `reusable-release`(do-release=false) → `reusable-docs`, and for image repos `reusable-container`(docker-label=snapshot) |
| `release.yml` | push `main` | `reusable-verify` → `reusable-release`(do-release=true) → `reusable-docs`, and for image repos `reusable-container`(docker-label=latest) |
| `docs.yml` | `workflow_dispatch` | `reusable-docs` (manual rebuild) |
| `scorecard.yml` | schedule / push main / branch_protection_rule | `reusable-scorecard` |
| `dependency-review.yml` | PR | `reusable-dependency-review` |
| `dependabot.yml` | – | config (github-actions + gradle) |

Chain for a push to `main`:

```
release.yml
  └─ verify   (reusable-verify)         license → build/test matrix [21,25]   [no secrets]
       └─ release (reusable-release)    GPG + gradle release, DO_RELEASE=true  [Sonatype+GPG]
            ├─ docs (reusable-docs)     VitePress build + Pages deploy         [no secrets]
            └─ container (reusable-container)  image from the release-jars     [Docker Hub]
                                               artifact, tag `latest`          (image repos only)
```

`snapshot.yml` is identical, only `DO_RELEASE=false` (→ Maven Snapshot) and the moving image
tag is `snapshot`. `docs` and `container` both hang off `release` and run in parallel.

---

## 4. Versioning / pinning the central workflows

Consumers reference the central workflows. Two options:

- **Recommended (keeps the Scorecard "Pinned-Dependencies" posture):** pin by **SHA** and let
  **Dependabot** (`package-ecosystem: github-actions`) bump them automatically. The actual
  version work happens only in `eclipse-fennec/.github`; consumer PRs are trivial SHA bumps.
- **Alternative (more convenient, but Scorecard complains):** the moving tag `@v1` (exists, see
  §4.2). One central edit, no consumer PRs — but an unpinned reference.

In the examples below `@<PIN>` is a placeholder. When migrating, replace it with the concrete
SHA (recommended) or `v1`.

### 4.1 Cutting a new version

Releases are plain annotated tags on `main` plus a GitHub release; there is no separate release
branch. Pick the level by what changed in `.github/workflows/reusable-*.yml`:

| Level | When |
| --- | --- |
| patch (`v1.1.1` → `v1.1.2`) | Dependabot action-pin bumps, doc fixes — no input or behaviour change. |
| minor (`v1.1.x` → `v1.2.0`) | New optional inputs, new reusable workflow, additional build steps. |
| major (`v1.x` → `v2.0.0`) | Removed or renamed inputs, changed defaults, anything that breaks a caller. |

```bash
git checkout main && git pull --ff-only
git tag -a v1.1.2 -m "Action SHA updates from Dependabot (harden-runner 2.20.1, setup-java 5.7.0, ...)"
git push origin v1.1.2
gh release create v1.1.2 --verify-tag --title v1.1.2 --notes "..."
```

### 4.2 Keeping the moving `v1` tag current

`v1` is a **moving** tag that always points at the newest `v1.x.y`. It does not update itself —
every new v1 release has to re-point it, otherwise `@v1` consumers silently stay on the old
workflows:

```bash
git tag -f -a v1 -m "Moving tag: latest v1.x reusable workflows (currently v1.1.2)" <commit>
git push origin v1 --force
```

Notes:

- The force-push is expected here and only ever applies to `v1` — the immutable `v1.x.y` tags are
  never moved or deleted.
- A new major line gets its own moving tag (`v2`); `v1` then stays frozen on the last v1 release
  so existing consumers keep working.
- `gh release view v1.1.2` / `git ls-remote origin 'refs/tags/v1^{}'` are the quick checks that
  release and moving tag ended up on the same commit.

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

Each JDK leg uploads its JUnit XML as artifact `test-results-java-<version>` (both
`**/build/test-results/` and bnd's `**/generated/test-reports/{test,testOSGi}/`) and then
renders it with `mikepenz/action-junit-report` in *annotate-only* mode: a job summary
(failed/skipped tests only, empty suites hidden) plus inline annotations, **no check run**.
That mode needs no permission beyond `contents: read`, so it is a drop-in for every caller.
The renderer is skipped when no XML exists (the build died before any test ran — that run is
already red) and never fails the job itself; the Gradle step is the gate. Check runs are the
job of `reusable-test-report.yml` (§5.6).

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
`artifact-paths`; `reusable-container.yml` (§5.7) then fetches the `release-jars` artifact
instead of rebuilding the workspace. This guarantees the jars
published to Maven and the jars baked into the images come from the **same build**. The same
export tasks can be passed to `reusable-verify`'s `extra-gradle-tasks` so PRs and feature
branches validate the bndrun exports as well (without any artifact upload).

### 5.3 `reusable-docs.yml`

VitePress build + GitHub Pages deploy. The repo-specific publish path slug comes from each
repo's `docs-site/config.mts` (via `DOCS_BRANCH`), not from this workflow. The deploy job
holds `pages: write` + `id-token: write` and is the **only** place that declares the
concurrency group `pages`. A caller must **not** declare `group: pages` at workflow level as
well: the caller's run then holds the group the deploy job waits for, and GitHub cancels the
job before its first step — the run fails in *Deploy to GitHub Pages* with zero steps and no
annotation (#35, first seen in model.atlas). Callers that want to serialise their own run use
a different group name (`snapshot-publish`, `release-publish` in §6.2/§6.3).

| Input | Default | Purpose |
|---|---|---|
| `node-version` | `20` | Node version for the build |
| `version` | *ref name* | URL segment to publish under. Pass it when the branch name is not the segment — a release workflow on `main` publishing `latest`. |
| `deploy` | `true` | `false` builds the slice and uploads it as a plain artifact instead of deploying. |
| `slice-artifact` | `pages-docs` | Artifact name for that slice. |

**Slice mode (`deploy: false`).** A Pages deploy replaces the *entire* site, so a repo that
publishes more than the docs cannot let this workflow deploy. `emf.m2x` is the case: it
publishes the OCL p2 update site under `ocl/<channel>/p2/` alongside the docs under
`<channel>/`, for both `snapshot` and `latest`. It calls this workflow with `deploy: false`
to get the docs slice, and its repo-local `pages-deploy.yml` merges all `pages-*` slices with
the previously published site and deploys once. In slice mode the root `index.html` redirect
is **not** written — with several channels the root has to point somewhere deterministic, so
that decision belongs to the merging deploy job.

Before `npm ci`, the build job checks out this repo at `github.job_workflow_sha` — the commit
of the reusable workflow itself, so theme and workflow can never be at different versions —
and compares every file under `docs-theme/` with its counterpart in the consumer. Any
difference fails the build with the resync command in the log. Files in
`docs-site/docs/public/` that the shared source does not carry (e.g. `fennec-logo.png`) are
left alone.

**Consequence for the rollout:** a consumer cannot bump its pin to a version carrying the
gate before it has the theme files. Add the files and bump the pin in the same PR. Repos still
pinned to an older SHA are unaffected.

### 5.4 `reusable-scorecard.yml`

OpenSSF Scorecard analysis. `permissions: read-all` at workflow level; the analysis job holds
`security-events: write`, `id-token: write`, `contents: read`, `actions: read`.

### 5.5 `reusable-dependency-review.yml`

`actions/dependency-review-action` with `fail-on-severity: high`, PR comment on failure.

---

### 5.6 `reusable-test-report.yml` (opt-in check runs)

Downloads the `test-results-java-<version>` artifacts that `reusable-verify.yml` uploaded and
publishes each JDK leg as its own **check run** on the commit ("Test results (Java 21)"), with
per-test annotations anchored to the source. Input: `java-versions` (JSON array, default
`["21","25"]` — must match the value passed to verify). The two legs stay separate on
purpose: they run the same suite, so a merged check would double every count. It writes no
job summary (verify already does) and never fails the run on a failing test (verify is the
gate) — but it *does* fail when the artifact is present and no test was parsed, so a changed
report layout is loud instead of silently empty.

This is a separate workflow rather than an input on verify because a check run needs
`checks: write`, and a called workflow cannot request a permission conditionally: a job-level
`checks: write` inside `reusable-verify` would be demanded from every caller, and consumers
call it with `contents: read` only. Opting in is therefore a second `uses:` job in the caller
that grants the permission on that job alone:

```yaml
jobs:
  verify:
    uses: eclipse-fennec/.github/.github/workflows/reusable-verify.yml@<PIN>
  test-report:
    needs: verify
    # Render whatever verify produced, red or green — a failing run is exactly
    # when the per-test annotations are worth having.
    if: ${{ !cancelled() }}
    permissions:
      contents: read
      checks: write
    uses: eclipse-fennec/.github/.github/workflows/reusable-test-report.yml@<PIN>
```

The job skips itself on pull requests from forks and on Dependabot PRs, where `GITHUB_TOKEN`
has no `checks: write` regardless of what the workflow declares. Trialled repo-locally in
`emf.m2x` (eclipse-fennec/emf.m2x#241) before moving here; see eclipse-fennec/.github#34.

### 5.7 `reusable-container.yml`

Builds **one** container image and pushes it to Docker Hub *and* GHCR. Repos that publish
several variants of the same image call the workflow once per variant, so the variants build
in parallel and a broken one does not block the others.

The image never rebuilds the workspace: the runtime jar comes from the `release-jars`
artifact that `reusable-release.yml` uploaded (`extra-gradle-tasks` + `artifact-paths`, §5.2),
so the jar inside the image is byte-identical to the one published to Maven. **The caller must
therefore run this after a release job, never straight after verify** — without that artifact
the download step fails.

Inputs:

| Input | Default | Purpose |
|---|---|---|
| `image-name` | required | Image repository name in both registries, e.g. `event.atlas`. |
| `docker-label` | required | Moving tag — `snapshot` from `snapshot.yml`, `latest` from `release.yml`. |
| `docker-context` | required | Docker build context, e.g. `docker/eventatlas`. |
| `version-jar` | required | Jar whose `Bundle-Version` becomes the immutable tag. |
| `variant` | `''` | Tag prefix for multi-variant repos (`apicurio`, `file`, `atlas`, …). |
| `runtime-jar` | `''` | Exported runtime jar to stage. Required unless `prepare-command` is set. |
| `runtime-jar-target` | `''` | Name the jar gets under `content/`; defaults to the `runtime-jar` file name. |
| `runtime-dir` | `''` | Repo-local directory staged into `content/runtime/`. |
| `prepare-command` | `''` | Stages the context itself, e.g. a Gradle `prepareDocker` task. |
| `java-version` | `'21'` | JDK for the version lookup and the prepare command. |
| `platforms` | `linux/amd64,linux/arm64/v8` | Multi-arch target list. |
| `bnd-version` | `'7.2.1'` | bnd CLI used to read `Bundle-Version`. |
| `docker-hub-namespace` | `eclipsefennec` | Docker Hub owner. |
| `ghcr-namespace` | `''` | GHCR owner; empty means the calling repository's owner. |

Secrets: `DOCKER_USERNAME`, `DOCKER_API_TOKEN` (GHCR uses the run's own `GITHUB_TOKEN`).
Outputs: `version` (the `Bundle-Version` used as the tag) and `digest` of the pushed manifest.

**Tagging.** Four tags per call — the moving label and the immutable bundle version, in both
registries, each prefixed with `<variant>-` when `variant` is set:

```
docker.io/eclipsefennec/model.atlas:apicurio-snapshot        # moving
docker.io/eclipsefennec/model.atlas:apicurio-1.2.3.20260908… # immutable
ghcr.io/eclipse-fennec/model.atlas:apicurio-snapshot
ghcr.io/eclipse-fennec/model.atlas:apicurio-1.2.3.20260908…
```

**Two ways to stage the build context**, mutually exclusive:

- *declarative* (default) — `runtime-jar` is copied to
  `<docker-context>/content/<runtime-jar-target>` and the **contents** of `runtime-dir` into
  `<docker-context>/content/runtime/`. Nothing else is needed in the consumer repo. The copy
  is a glob, so a top-level dotfile — every one of these trees carries a development-time
  `.gitignore` — stays out of the image, while the `.keep` placeholders that hold mount-point
  directories in git are one level down and travel with their subdirectory.
- *`prepare-command`* — an arbitrary command (in practice `./gradlew --no-daemon
  :docker:<x>:prepareDocker`) stages `content/` itself. It wins over the declarative copy, and
  it is the only mode that also runs Gradle wrapper validation and enables the Gradle cache.

**Jars are located by name, not by path.** `actions/upload-artifact` roots the archive at the
least common ancestor of the uploaded paths, so the layout inside `release-jars` depends on
*which* paths the release job uploaded. `version-jar`/`runtime-jar` are therefore used as-is
when the path exists and otherwise searched for by file name below the workspace. A
`prepare-command` has no such freedom — it reads the jar from the exact path its Gradle task
expects, which survives the round-trip only as long as the release job uploads **at least two
paths from different top-level directories** (the usual runtime jar + version jar pair keeps
the ancestor at the repository root).

Consumer values for the repos that currently carry a repo-local copy of this workflow:

| Repo | Calls | Notable inputs |
|---|---|---|
| `event.atlas` | 1 | `docker-context: docker/eventatlas`, `runtime-jar: eventatlas.runtime_docker.jar`, `runtime-dir: org.eclipse.fennec.event.atlas.mapping.runtime/runtime`, `version-jar: org.eclipse.fennec.event.atlas.mapping.jar` |
| `model.atlas` | 2 (`apicurio`, `file`) | `java-version: '25'`, per-variant `docker-context: docker/modelatlas_<variant>` and `runtime-jar: modelatlas.runtime_docker_<variant>.jar`, both renamed via `runtime-jar-target: modelatlas.runtime_docker.jar` |
| `data.atlas` | 2 (`file`, `atlas`) | `docker-context: docker/dataatlas` / `docker/dataatlas-atlas`, `runtime-jar: dataatlas.runtime_docker.jar` / `dataatlas.runtime_docker_atlas.jar` (Dockerfiles COPY the variant name, so no `runtime-jar-target`) |
| `dcat.atlas` | 1 | `prepare-command: ./gradlew --no-daemon :docker:dcatatlas:prepareDocker`, `version-jar: org.eclipse.fennec.dcat.atlas.api.jar`, no `runtime-dir` (the image carries only the jar) |

---

## 6. Consumer workflows (into each project repo)

`@<PIN>` = SHA (recommended) or `v1`. `secrets: inherit` forwards repo/org secrets only to the
called reusable, and a reusable only ever receives the secrets it declares: `reusable-release.yml`
the Sonatype/GPG ones, `reusable-container.yml` the Docker Hub ones, while verify and docs
receive **no** credentials at all (credential scoping is preserved).

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

Test results appear as a job summary on the verify jobs by default. For check runs with
per-test annotations add the opt-in `test-report` job from §5.6.

### 6.2 `snapshot.yml` (development branch → Maven Snapshot)

Repos that publish a container image add a `container` job and `packages: write` here — see
§6.8.

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

Repos that publish a container image add a `container` job and `packages: write` here — see
§6.8.

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
# No workflow-level concurrency here, deliberately: the reusable's deploy job
# already serialises on the job-level group `pages`. Declaring the same group
# here makes the deploy job deadlock against its own run (see §5.3, #35).
jobs:
  docs:
    uses: eclipse-fennec/.github/.github/workflows/reusable-docs.yml@<PIN>
```

Consumers whose `docs.yml` still carries the `concurrency: group: pages` block from an
earlier version of this template must remove it (model.atlas did in
eclipse-fennec/model.atlas#249; emf.codec, emf.osgi-mcp and event.atlas still have it).

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

### 6.8 Container publishing (image repos only)

Three additions to `snapshot.yml` / `release.yml`, nothing else:

1. `packages: write` in the workflow-level `permissions` (for the GHCR push),
2. the export tasks and jar paths on the `release` job, so the image gets its jar out of the
   very build that published to Maven,
3. a `container` job per image variant, `needs: release`.

`snapshot.yml` for a single-image repo (`event.atlas`):

```yaml
permissions:
  contents: read
  pages: write
  id-token: write
  packages: write # GHCR push
jobs:
  # verify: … (as in §6.2)
  release:
    needs: verify
    uses: eclipse-fennec/.github/.github/workflows/reusable-release.yml@<PIN>
    with:
      do-release: false
      # bnd resolve/export tasks are not parallel-safe in this workspace.
      gradle-parallel: false
      extra-gradle-tasks: >-
        org.eclipse.fennec.event.atlas.mapping.runtime:export.eventatlas.runtime_docker
      # The runtime jar the image runs, plus the jar the container job reads the
      # Bundle-Version from. Two paths from different top-level directories, so the
      # artifact keeps its directory layout (§5.7).
      artifact-paths: |
        org.eclipse.fennec.event.atlas.mapping.runtime/generated/distributions/executable/eventatlas.runtime_docker.jar
        org.eclipse.fennec.event.atlas.mapping/generated/org.eclipse.fennec.event.atlas.mapping.jar
    secrets: inherit
  container:
    needs: release
    uses: eclipse-fennec/.github/.github/workflows/reusable-container.yml@<PIN>
    with:
      image-name: event.atlas
      docker-label: snapshot
      docker-context: docker/eventatlas
      runtime-jar: eventatlas.runtime_docker.jar
      runtime-dir: org.eclipse.fennec.event.atlas.mapping.runtime/runtime
      version-jar: org.eclipse.fennec.event.atlas.mapping.jar
    secrets: inherit
  docs:
    needs: release
    uses: eclipse-fennec/.github/.github/workflows/reusable-docs.yml@<PIN>
```

`release.yml` is the same with `do-release: true` and `docker-label: latest`.

A multi-variant repo repeats the `container` job per variant (`model.atlas`):

```yaml
  container-apicurio:
    needs: release
    uses: eclipse-fennec/.github/.github/workflows/reusable-container.yml@<PIN>
    with:
      image-name: model.atlas
      docker-label: snapshot
      variant: apicurio
      java-version: '25'
      docker-context: docker/modelatlas_apicurio
      runtime-jar: modelatlas.runtime_docker_apicurio.jar
      # The Dockerfile COPYs a variant-independent name.
      runtime-jar-target: modelatlas.runtime_docker.jar
      runtime-dir: org.eclipse.fennec.model.atlas.runtime/runtime
      version-jar: org.eclipse.fennec.model.atlas.rest.application.jar
    secrets: inherit
  container-file:
    # … identical, variant: file, docker-context: docker/modelatlas_file,
    #    runtime-jar: modelatlas.runtime_docker_file.jar
```

And a repo that stages its context with Gradle (`dcat.atlas`):

```yaml
  container:
    needs: release
    uses: eclipse-fennec/.github/.github/workflows/reusable-container.yml@<PIN>
    with:
      image-name: dcat.atlas
      docker-label: snapshot
      docker-context: docker/dcatatlas
      prepare-command: ./gradlew --no-daemon :docker:dcatatlas:prepareDocker
      version-jar: org.eclipse.fennec.dcat.atlas.api.jar
    secrets: inherit
```

The repo-local `reusable-container.yml` is **deleted** in the same PR; `secrets: inherit`
forwards `DOCKER_USERNAME`/`DOCKER_API_TOKEN` exactly as it did before.

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

Image-publishing repos additionally need the Docker Hub credentials, which reach
`reusable-container.yml` the same way (and nothing else — the release job declares no Docker
secrets, the container job no Sonatype/GPG ones):

- `DOCKER_USERNAME`
- `DOCKER_API_TOKEN`

The GHCR push needs no secret at all; it authenticates with the run's own `GITHUB_TOKEN`.

**Permissions:** a called workflow can only **downgrade** the `GITHUB_TOKEN` permissions it
receives from the calling job, never elevate them, and it cannot make a permission
conditional on an input. The ceiling is the caller's workflow-level `permissions`, or the
job-level `permissions` on the `uses:` job where one is set. Therefore:

- `snapshot.yml`/`release.yml`/`docs.yml` declare `contents: read`, `pages: write`,
  `id-token: write` at workflow level (for the docs deploy), plus `packages: write` in the
  image-publishing repos (for the GHCR push, see §6.8).
- The reusables declare the finer job-level permissions themselves.
- Anything a reusable needs beyond `contents: read` that not every caller wants is a
  separate reusable the caller opts into with a job-level grant — `reusable-test-report.yml`
  with `checks: write` (§5.6) is the pattern.
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

1. **Prerequisite:** `eclipse-fennec/.github` contains the reusables of §5. Choose `@<PIN>`
   (the `.github` commit SHA, recommended).
2. Create a branch in the project repo (repos are PR-only).
3. Replace the old workflows with the thin callers from §6:
   - overwrite `build.yml`, `snapshot.yml`, `release.yml`, `docs.yml`, `scorecard.yml`,
     `dependency-review.yml`.
   - **delete** the standalone `license.yml` (the license gate now lives in `reusable-verify`).
   - image repos: wire the `container` job(s) per §6.8 and **delete** the repo-local
     `reusable-container.yml`.
4. Ensure `dependabot.yml` exists (§6.7) — it is missing in `emf.odata`.
5. Set `@<PIN>` in all 6 callers to the chosen SHA/tag.
6. License config: keep the repo's existing `.licenserc.yaml` as-is (license config stays
   project-local; a possible central default is an open proposal — see §7).
7. Repo-specific checks:
   - `docs-site/config.mts` sets the correct base-path slug (`/emf.util/`, `/emf.odata/`, …).
     The docs workflow is generic; the slug lives here.
   - `docs-site/package-lock.json` exists (npm cache path in the docs workflow).
   - Gradle tasks `build`, `testOSGi`, `perfTest`, `release` exist.
   - All 5 secrets from §8 are set (org or repo level) — plus `DOCKER_USERNAME` and
     `DOCKER_API_TOKEN` in image repos.
   - image repos: the bndrun export tasks and the exported jar paths passed to
     `reusable-release` still match the Dockerfile's `COPY content/…` lines.
8. **`initial` branch decision:** the old emf.util workflows special-cased `initial`. In the new
   model `initial` is an ordinary feature branch (verify only). If it should not build, add it to
   `branches-ignore` in `build.yml`.
9. Test order:
   - open a PR → only `verify` + `dependency-review` should run.
   - merge to `snapshot` → `verify` → `release`(Snapshot) → `docs` (+ `container`, tag
     `snapshot`).
   - merge to `main` → `verify` → `release`(Central) → `docs` (+ `container`, tag `latest`).
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
- **One image per `reusable-container.yml` call.** `model.atlas`' repo-local copy built both
  variants in a single job; centrally each variant is its own job. It re-downloads the
  `release-jars` artifact per variant (seconds) and buys parallel builds plus a failure that
  stays local to one variant.
