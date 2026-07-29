---
name: bnd
description: Use the bnd CLI (java -jar biz.aQute.bnd.jar) instead of manual unzip/javap/manifest parsing whenever working with jars or OSGi bundles — inspecting manifests, imports/exports, DS components, metatype, capabilities, required Java version (EEs); searching which jar exports/imports a package; comparing two jar versions (diff/baseline, semantic versioning advice); verifying bundles; wrapping plain jars into bundles; indexing repos; resolving .bndrun files. Trigger words - jar analysis, MANIFEST.MF, Import-Package, Export-Package, bundle, OSGi, baseline, wrap, bndrun.
---

# bnd CLI: analyzing and building jars/OSGi bundles

bnd is the swiss army knife for jars and OSGi bundles. Prefer it over `unzip -p`,
`javap`, and hand-parsing manifests — it understands versions, uses-constraints,
DS components, and semantic versioning.

## Invocation

Use the current release (avoid `-SNAPSHOT`; RC is fine). Download once into the
local Maven repo if it is not already there:

```bash
BND_VERSION=7.3.0
BND="$HOME/.m2/repository/biz/aQute/bnd/biz.aQute.bnd/$BND_VERSION/biz.aQute.bnd-$BND_VERSION.jar"
[ -f "$BND" ] || curl -sL --create-dirs -o "$BND" \
  "https://repo1.maven.org/maven2/biz/aQute/bnd/biz.aQute.bnd/$BND_VERSION/biz.aQute.bnd-$BND_VERSION.jar"
java -jar "$BND" <command> ...
```

If a newer release exists in `~/.m2/repository/biz/aQute/bnd/biz.aQute.bnd/` (or in
Central's `maven-metadata.xml`), prefer it over the pinned version above.
`java -jar "$BND" help` lists all commands; `help <command>` shows its options.

## Inspecting a single jar

| Task | Command |
|---|---|
| Manifest (pretty-printed) | `print <jar>` (default) or `view <jar> META-INF/MANIFEST.MF` |
| Imports/exports incl. version ranges | `print -i <jar>` |
| DS components in detail | `print -C <jar>` |
| Metatype (configuration) data | `print -t <jar>` |
| Provided/required capabilities | `print -c <jar>` |
| Package uses / used-by graph | `print -u <jar>` / `print -b <jar>` |
| API usage constraints on exports | `print -a <jar>` |
| Everything at once (+`-v` verifies too) | `print -f <jar>` |
| Verify bundle correctness | `verify <jar>` |
| Required Java version (Execution Environments) | `ees <jar>` |
| List / extract contents (jar tf/xf equivalent) | `type -f <jar>`, `extract -f <jar> -c <dir>` |
| View one resource; .class files are disassembled | `view <jar> <resource-path>` |
| Expand Bundle-ClassPath into a flat jar | `flatten <in.jar> <out.jar>` |

## Searching across many jars

| Task | Command |
|---|---|
| Which jar exports/imports a package? | `find -e 'org.osgi.service.*' *.jar` / `find -i <glob> *.jar` |
| Grep manifest headers | `grep -h 'Bundle-Vendor' '<pattern>' *.jar`; `-b` bsn, `-e` exports, `-i` imports |
| Filter jars by manifest assertion, print headers | `select --where 'Bundle-Version=1.0.*' --header 'Bundle-SymbolicName' -n *.jar` |
| Class/package cross-references | `xref -t <jars>` (references to), `-f` (from), `--classes` for class level, `-m <glob>` to filter |

## Comparing versions

| Task | Command |
|---|---|
| Structural diff of two jars | `diff <newer.jar> <older.jar>` (`-m` manifest, `-r` resources, `-a` API, `-f` full tree) |
| Semantic-versioning verdict (what version bump is required, per package) | `baseline -d -v <newer.jar> <older.jar>` |
| Diff two OSGi XML repository indexes | `xmlrepodiff <newer.xml> <older.xml>` |

`baseline` is the tool of choice for "is this a breaking change / which version must
this package get" questions — do not answer those by eyeballing diffs.

## Creating and providing bundles

| Task | Command |
|---|---|
| Wrap a plain jar into an OSGi bundle | `wrap -b <bsn> -v <version> -o <out.jar> <in.jar>` (fine-grained control: write a `.bnd` file and run `bnd <file>.bnd`) |
| Generate an OSGi repository index for a dir of jars | `index -d <dir> -n <name> <dir>/*.jar` |

## Workspace / bndrun (bnd workspaces, e.g. our Fennec repos)

| Task | Command |
|---|---|
| Resolve a `.bndrun`, print bundles | `resolve -b <file.bndrun>` |
| Resolve and write `-runbundles` back into the file | `resolve -W <file.bndrun>` |
| List configured repositories / their content | `repo repos`, `repo list` (in a workspace/project dir; `-w <workspace>`) |
| Look up what a bnd header/instruction means | `syntax <header>` e.g. `syntax -exportcontents` |
| Expand a macro in project context | `macro '<macro>'` |
| Project info / diagnostics | `info`, `debug` (inside a project dir) |

Builds in our repos run through Gradle/Maven as usual — use the CLI's `build`/`run`/
`test` only for quick experiments, not as a replacement for the project's build.

## Notes

- Multiple jars are accepted by most analysis commands — glob freely.
- Exit code is non-zero when `verify`/`baseline` find problems; their stderr lists them.
- On Windows Git Bash, quote `$BND` and jar paths (spaces in `~/.m2` are rare but paths
  with drive letters work as `/c/Users/...`).
