# shadcn/ui Wrapper Design

## Context

`accelerate-data/shadcn-ui` is a fork of `shadcn-ui/ui`. The upstream repository
contains a root `skills/shadcn/SKILL.md`, but it does not provide Claude or Codex
plugin manifests for the Accelerate Data marketplace.

Unlike the thinking-skills collection, the shadcn skill is not purely narrative.
It includes Claude-style metadata such as `allowed-tools` and an inline command
injection block for `npx shadcn@latest info --json`. The skill body is valuable
for Codex, but Codex compatibility should be verified before the Codex
marketplace entry is treated as production-ready.

## Goals

- Publish the upstream shadcn skill through the Accelerate Data marketplace as
  `shadcn`.
- Keep the upstream `skills/shadcn/` directory as real root files.
- Add plugin manifests needed for marketplace installation.
- Make the repository's forked/upstream-owned nature explicit to AI agents.
- Require agents to warn before editing upstream-owned shadcn source or skill
  content.
- Add a sync workflow that opens reviewable pull requests for upstream changes.
- Include a Codex compatibility review step for Claude-specific skill metadata.

## Non-Goals

- Rewrite the shadcn skill during wrapper setup.
- Fork shadcn component source code behavior.
- Change templates, packages, registries, or app code.
- Automatically merge upstream changes without review.

## Repository Shape

The fork keeps the upstream layout intact:

```text
shadcn-ui/
  .claude-plugin/plugin.json
  .codex-plugin/plugin.json
  AGENTS.md
  README.md
  skills/
    shadcn/
      SKILL.md
      cli.md
      customization.md
      mcp.md
  packages/
  apps/
  templates/
  .github/workflows/sync-upstream.yml
```

The `skills/` directory remains a real root directory. Symlinks are avoided so
plugin installers and scanners do not need special filesystem behavior.

## Marketplace Identity

Use `shadcn` as the marketplace-facing plugin name. Both marketplace entries
should use a whole-repo source:

```json
{
  "source": "url",
  "url": "https://github.com/accelerate-data/shadcn-ui.git"
}
```

The Claude marketplace entry may use `strict: false` if needed for compatibility
with upstream skill metadata. The Codex marketplace entry should point at this
fork only after `.codex-plugin/plugin.json` exists and the skill has been smoke
tested in Codex.

## Codex Compatibility

The current `skills/shadcn/SKILL.md` contains:

- `allowed-tools` metadata with Claude-style Bash allowlists.
- An inline command injection block:
  ````markdown
  ```json
  !`npx shadcn@latest info --json`
  ```
  ````

Those constructs may be ignored or unsupported by Codex. The first wrapper
implementation should add plugin manifests and run an install smoke test. If
Codex does not support those constructs, create a small Codex-specific
compatibility patch only after confirming the failure mode.

## Agent Safety Note

`AGENTS.md` should state that this repository is a maintained fork of
`shadcn-ui/ui`. AI agents may edit wrapper-owned files such as `.claude-plugin/`,
`.codex-plugin/`, `.github/workflows/`, documentation, and validation scripts as
normal.

Before editing files under `skills/`, `packages/`, `apps/`, or `templates/`,
agents must warn the user that the file is upstream-owned and explain that the
change may diverge from future upstream syncs. They may proceed only when the
user explicitly confirms the local divergence or when the task is specifically
to resolve an upstream sync conflict.

## Upstream Sync

A scheduled GitHub Actions workflow should fetch `shadcn-ui/ui`, merge the
upstream default branch into a sync branch, run validation, and open a pull
request. The workflow should not push directly to `main`.

The sync PR gives the team a review point for upstream shadcn changes and any
conflicts against wrapper-owned plugin files.

## Validation

The implementation should verify:

- `.claude-plugin/plugin.json` is valid JSON.
- `.codex-plugin/plugin.json` is valid JSON.
- `skills/shadcn/SKILL.md` exists.
- The repository package manager can install or verify dependencies as required
  by existing shadcn workflows.
- The Codex marketplace can install the plugin from this fork.
- The Claude marketplace can install the plugin from this fork.

Marketplace changes belong in `plugin-marketplace`, not in this fork.
