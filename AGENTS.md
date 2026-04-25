# AGENTS.md

This repository is an Accelerate Data-maintained wrapper for the upstream
`shadcn-ui/ui` skill.

## Vendored Upstream Content

The shadcn skill under `skills/shadcn/` is upstream-owned content from
`shadcn-ui/ui`.

AI agents may edit wrapper-owned files such as `.claude-plugin/`,
`.codex-plugin/`, `.github/workflows/`, documentation, and validation scripts as
normal.

Before editing files under `skills/shadcn/`, AI agents must warn the user that
the file is upstream-owned and explain that the change may diverge from future
upstream syncs. Proceed only when the user explicitly confirms the local
divergence or when the task is specifically to resolve an upstream sync
conflict.

## Repository Purpose

This repository packages the upstream shadcn skill for marketplace installation
in Claude and Codex. It should preserve upstream shadcn skill behavior unless a
reviewed local divergence is explicitly requested.
