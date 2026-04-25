# shadcn skill wrapper

This repository packages the upstream `shadcn-ui/ui` skill for Accelerate Data
Claude and Codex marketplace installation.

## What is included

- `.claude-plugin/plugin.json` for Claude marketplace installation.
- `.codex-plugin/plugin.json` for Codex marketplace installation.
- `skills/shadcn/`, synchronized from upstream `shadcn-ui/ui`.
- Repository guidance and GitHub Actions for maintaining the wrapper.

## What is not included

This repository intentionally does not vendor the full upstream shadcn monorepo.
It does not include upstream `apps/`, `packages/`, or `templates/` because those
files are not needed for marketplace installation of the skill.

For shadcn documentation and source code, use the upstream project:

- Documentation: https://ui.shadcn.com/docs
- Source: https://github.com/shadcn-ui/ui

## Upstream sync

The weekly `Sync upstream shadcn skill` workflow sparse-checks out
`skills/shadcn/` from `shadcn-ui/ui`, copies it into this wrapper repository,
validates the plugin manifests, and opens a reviewable pull request when the
skill changes.

## Ownership

`skills/shadcn/` is upstream-owned content. Wrapper metadata, documentation, and
workflows are maintained by Accelerate Data.

Marketplace registry changes belong in the marketplace repository, not in this
wrapper source repository.
