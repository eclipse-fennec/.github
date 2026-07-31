# Fennec Claude Code plugin marketplace

This repo doubles as a [Claude Code plugin marketplace](https://code.claude.com/docs/en/plugins)
for shared Fennec dev tooling. Like the reusable workflows, the goal is a single source of
truth: skills are maintained here once and every fennec developer/repo picks them up from here.

## What's in it

| Plugin | Contents |
|---|---|
| `fennec-dev` | `fennec-quality-review` skill — code-quality review of a Fennec OSGi repo against SOLID (adapted to OSGi/DS) and Eclipse Foundation guidelines. Supports whole-repo, scoped, pending-changes (`changes`), and follow-up (`followup`) reviews. |
| `fennec-dev` | `bnd` skill — jar/OSGi bundle analysis via the bnd CLI (pinned to the current release, 7.3.0, auto-downloaded from Maven Central): manifests, imports/exports incl. version ranges, DS components, metatype, capabilities, EEs, `baseline`/`diff` semantic-versioning verdicts, wrapping plain jars, repo indexing, `.bndrun` resolving. |

Marketplace and plugin manifests: `.claude-plugin/marketplace.json` (repo root) and
`plugins/fennec-dev/.claude-plugin/plugin.json`. Skill sources live under
`plugins/fennec-dev/skills/`.

## One-time install (per developer)

In any Claude Code session:

```
/plugin marketplace add eclipse-fennec/.github
/plugin install fennec-dev@fennec
```

The skill is then available in **every** repository, and `/plugin` update pulls new versions
from this repo.

## Per-repo auto-suggest (consumer repos)

To have Claude Code offer the plugin automatically to anyone who opens a fennec project repo,
commit this `.claude/settings.json` to the project repo:

```json
{
  "extraKnownMarketplaces": {
    "fennec": {
      "source": {
        "source": "github",
        "repo": "eclipse-fennec/.github"
      }
    }
  },
  "enabledPlugins": {
    "fennec-dev@fennec": true
  }
}
```

When a teammate trusts the project folder, Claude Code prompts them to install the marketplace
and enables the plugin. This file is a candidate addition to the per-repo migration checklist
in [ci-cd-reusable-workflows.md](./ci-cd-reusable-workflows.md).

## Maintaining the skills

- Edit the files under `plugins/fennec-dev/skills/<skill>/` and bump `version`
  in `plugins/fennec-dev/.claude-plugin/plugin.json` so installed copies see the update.
- The quality-review skill's Eclipse-guidelines reference is a distilled snapshot; it
  documents its own refresh procedure (`refresh` argument) in `SKILL.md`.
