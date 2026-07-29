# Eclipse Foundation / Platform guidelines — distilled snapshot

Snapshot date: 2026-07-23. On `refresh`, re-fetch these sources and update the sections below:

- Coding conventions: https://raw.githubusercontent.com/eclipse-platform/eclipse.platform/master/docs/Coding_Conventions.md
- Naming conventions: https://raw.githubusercontent.com/eclipse-platform/eclipse.platform/master/docs/Naming_Conventions.md
- Evolving Java-based APIs: https://raw.githubusercontent.com/eclipse-platform/eclipse.platform/master/docs/Evolving-Java-based-APIs-2.md
- IP / copyright headers: https://www.eclipse.org/projects/handbook/#ip-copyright-headers
- Fennec Eclipse release guide: https://raw.githubusercontent.com/eclipse-fennec/emf.osgi/snapshot/docs/eclipse-release-guide.md
- Fennec IP Dash / license check: https://raw.githubusercontent.com/eclipse-fennec/emf.osgi/snapshot/docs/ip-dash-license-check.md

## 1. License headers (severity: blocker when missing on shipped .java)

Every `.java` source file must carry the EPL-2.0 header. The fennec repos enforce it with SkyWalking Eyes (`.licenserc.yaml` at repo root — read it for the authoritative pattern and `paths-ignore` list; typical ignores: `package-info.java`, `packageinfo`, `module-info.java`, `*.bnd`, `*.bndrun`, XML/XMI/YAML). Canonical form:

```java
/*
 * Copyright (c) 2012 - <year> Data In Motion and others.
 * All rights reserved.
 *
 * This program and the accompanying materials are made
 * available under the terms of the Eclipse Public License 2.0
 * which is available at https://www.eclipse.org/legal/epl-2.0/
 *
 * SPDX-License-Identifier: EPL-2.0
 *
 * Contributors:
 *      Data In Motion - initial API and implementation
 */
```

Check: header present, `SPDX-License-Identifier: EPL-2.0` present, copyright owner and Contributors section present. Do not flag files matched by `paths-ignore`.

## 2. Package naming & API/internal hygiene

- General form `org.eclipse.<project>.<component>[.*]`, lowercase ASCII alphanumerics only, no `_`/`$`.
- Reserved qualifiers, placed **immediately after the project/component name**: `internal` (implementation, no API), `tests`, `examples`. E.g. `org.eclipse.jdt.internal.core.compiler` is correct; `org.eclipse.jdt.core.internal.compiler` and `org.eclipse.internal.*` are wrong.
- API packages must never contain `internal`, `tests`, `examples` segments. Everything `public` in an API package IS API.
- **OSGi translation (this is the check that matters in fennec):** a bundle's API = its `Export-Package`/`-exportcontents` in `bnd.bnd`. Findings:
  - exported package containing impl-only classes (`*Impl`, DS components) → **blocker** unless it is an EMF-generated model bundle (EMF `impl`/`util` packages are conventionally exported — do not flag).
  - implementation packages not marked private (`-privatepackage`/absent from exports) → major.
  - exported package without a package version (`package-info.java` `@Version` / `packageinfo` file) → major.
- Avoid names referencing a particular company or commercial product in code identifiers (copyright headers naming Data In Motion are fine).

## 3. Naming conventions (severity: minor)

- Classes: nouns, UpperCamelCase, whole words; abbreviations only when ubiquitous (URL, HTML).
- Interfaces: same rules; Eclipse does NOT require an `I` prefix outside Platform-proper code — do not flag its absence. Impl classes conventionally `<Interface>Impl`.
- Methods: verbs, lowerCamelCase. Constants: `UPPER_SNAKE`. Type params: single capital letter.
- Indentation: tabs (per Eclipse convention). Modifier order per JLS: `public protected private abstract default static final synchronized native strictfp transient volatile`.
- Eclipse workspace project name = bundle symbolic name (fennec follows this: directory name == BSN).

## 4. Javadoc & API quality

- All API (exported) types and their public/protected members need javadoc: what it does, param/return/throws, thread-safety and nullability where relevant.
- Non-API/internal code: javadoc optional; do not flag.
- API interfaces intended for implementation by clients vs. only for use should be documented as such (affects evolution rules below).

## 5. Evolving APIs / binary compatibility (severity: blocker on released API)

From "Evolving Java-based APIs": once API is released, in a minor/micro release you must not:
- delete or rename API packages/types/members; reduce visibility; change method signatures/return types; add checked exceptions to API methods.
- add abstract methods to API classes clients subclass, or (non-default) methods to interfaces clients implement — breaks implementors.
- change constants' values that clients may have inlined; change serialization contracts.

Allowed compatibly: adding API packages/types; adding members to classes clients don't subclass; adding `default` methods (with care — diamond clashes); widening input types.

**OSGi translation:** exported-package **semantic versioning** — micro = no API change, minor = API added (consumers fine, implementors maybe not), major = breaking. Check that package versions were bumped when API changed (bnd baselining does this mechanically; flag exported packages stuck at `1.0.0` across obviously different releases only as info).

## 6. Repository release-readiness (fennec release guide; category `release-readiness`, checked ONCE per repo, not per bundle)

The reference implementation for all of this is `eclipse-fennec/emf.osgi` — findings should say "copy from emf.osgi and adapt" where applicable.

**Root documents** — all six must exist at the repo root and be adapted (not template-verbatim where adaptation is required):
`LICENSE` (EPL-2.0 full text), `NOTICE.md` (project home, repo URL, notable deps), `README.md` (links docs, states branch model + Maven coordinates), `CONTRIBUTING.md` (ECA + DCO sign-off + Dash sections; security reports must point at `SECURITY.md`, not a mailing list), `CODE_OF_CONDUCT.md` (Eclipse Community CoC 2.0), `SECURITY.md` (GitHub advisories URL for THIS repo; supported-versions table current). Missing file = **major**; present-but-unadapted (wrong repo URL, stale supported versions) = **minor**.

**License-header enforcement**: `.licenserc.yaml` present and adapted (mind `paths-ignore`) + `.github/workflows/license.yml` (apache/skywalking-eyes) enabled. Missing enforcement = **major** even if headers themselves are fine.

**IP cleanliness (Dash)**: `tools/dash-licenses.sh`/`.bat` present; `.github/workflows/dash-licenses.yml` enabled; `DEPENDENCIES` file generated, reviewed and committed at the repo root; zero `restricted` entries at release time. Missing `DEPENDENCIES`/workflow = **major**; `restricted` entries present = **blocker** for a release, **major** otherwise (they need IP review via `dash-licenses.sh --review --project <PMI id>` — note the id is the dotted PMI id, e.g. `technology.fennec`, NOT the GitHub org). The DEPENDENCIES list is generated from the bnd workspace via `bnd repo deps` — do not suggest hand-maintained manifests.

**CI/publishing shape**: `build.yml`, `license.yml`, `snapshot.yml`, `release.yml` present; `snapshot` = development branch publishing `-SNAPSHOT` to Sonatype Central; a push to `main` IS the release trigger (immutable Maven Central) — flag PR-triggered publishing (secrets reaching fork code) as **blocker**. `cnf/build.bnd` sets `github-orga`, `github-project`, `-groupid`, `maven-central: true`.

**Baselining/OBR** (only after a repo's first release exists): `-plugin.release` + `-releaserepo.obr` in `cnf/build.bnd`; release OBR published as orphan branch `release-obr` from the SAME run that released to Maven Central (never a second `gradlew release` run); baseline block (`-plugin.baseline`, `-baseline: *`, `-diffpackages: *;threshold=MINOR`) enabled against it. Repos with releases but no baselining = **major** (API evolution rules in §5 are then unenforced).

## 7. Fennec idioms — do NOT flag

- EMF-generated code (`@generated`, `*.impl`/`*.util` EMF packages, `*Factory`/`*Package` classes): structural/naming findings suppressed; header findings still apply (generators stamp headers).
- The `emf.nsURI` service property spelling; scope/stage/registry workflow model; read-only storage backends rejecting writes with `UnsupportedOperationException` (documented contract, not an LSP violation).
- bnd conventions: `bnd.bnd` per bundle, BSN == directory name, `-runee`/`-runrequires` in `.bndrun` files.
