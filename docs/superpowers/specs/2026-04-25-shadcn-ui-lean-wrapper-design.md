# shadcn/ui Lean Wrapper Design

## Context

`accelerate-data/shadcn-ui` currently keeps a full fork-shaped checkout of
`shadcn-ui/ui` while the marketplace payload only needs the upstream skill at
`skills/shadcn/` plus Accelerate Data wrapper metadata.

The current repository has one distributable skill:

```text
skills/shadcn/
  SKILL.md
  agents/openai.yml
  assets/
  cli.md
  customization.md
  evals/evals.json
  mcp.md
  rules/
```

The upstream app, package, and template directories are useful for the upstream
project, but they are not needed for Claude or Codex marketplace installation of
this wrapper plugin.

## Goals

- Keep the repository as the source for the `shadcn` Claude and Codex plugin.
- Keep `skills/shadcn/` synchronized from `shadcn-ui/ui`.
- Remove full upstream app/package/template source from this wrapper repo.
- Replace full upstream merge sync with sparse upstream skill synchronization.
- Keep sync updates reviewable through pull requests.
- Make wrapper-owned and upstream-owned paths clearer for future agents.
- Keep validation small, deterministic, and focused on plugin installability.

## Non-Goals

- Change upstream shadcn skill behavior.
- Edit `skills/shadcn/` content as part of the lean migration.
- Package shadcn application code, registry code, test packages, or templates.
- Recreate the upstream repository's development workflow in this wrapper.
- Change marketplace registry entries outside this repository.

## Current Repository Shape

The repo currently resembles the upstream monorepo:

```text
shadcn-ui/
  .claude-plugin/plugin.json
  .codex-plugin/plugin.json
  .github/workflows/sync-upstream.yml
  AGENTS.md
  README.md
  apps/
  packages/
  skills/shadcn/
  templates/
```

This makes the clone/install payload much larger than the distributable plugin
surface and increases review noise when upstream changes unrelated code.

## Target Repository Shape

The target is a lean wrapper repository:

```text
shadcn-ui/
  .claude-plugin/plugin.json
  .codex-plugin/plugin.json
  .github/workflows/sync-upstream.yml
  AGENTS.md
  LICENSE.md
  README.md
  docs/superpowers/
  skills/
    shadcn/
      SKILL.md
      agents/openai.yml
      assets/
      cli.md
      customization.md
      evals/evals.json
      mcp.md
      rules/
```

Wrapper-owned files are all files outside `skills/shadcn/`, except license text
that documents upstream attribution. Upstream-owned content is limited to
`skills/shadcn/`.

## Sync Model

The sync workflow should stop merging `upstream/main` into this repository.
Instead, it should check out the wrapper repo and check out only
`skills/shadcn` from `shadcn-ui/ui` into a temporary path using GitHub Actions
sparse checkout.

The workflow should then replace the local `skills/shadcn/` directory with the
sparse upstream copy, validate wrapper files, push a sync branch, and open a
pull request.

Representative workflow steps:

```yaml
- name: Checkout wrapper repo
  uses: actions/checkout@v4
  with:
    fetch-depth: 0

- name: Checkout upstream shadcn skill
  uses: actions/checkout@v4
  with:
    repository: shadcn-ui/ui
    ref: main
    path: upstream
    sparse-checkout: |
      skills/shadcn
    sparse-checkout-cone-mode: false

- name: Sync upstream skill
  run: |
    rm -rf skills/shadcn
    mkdir -p skills
    cp -R upstream/skills/shadcn skills/shadcn
```

This keeps upstream updates reviewable without making this repository a full
monorepo fork.

## File Ownership

`AGENTS.md` should state:

- `skills/shadcn/` is upstream-owned content from `shadcn-ui/ui`.
- `.claude-plugin/`, `.codex-plugin/`, `.github/workflows/`, documentation, and
  validation scripts are wrapper-owned.
- Agents may edit wrapper-owned files normally.
- Agents must warn before editing `skills/shadcn/` unless the task is explicitly
  to resolve a sync conflict or intentionally diverge from upstream.

The old warning for `apps/`, `packages/`, and `templates/` should be removed
after those directories are deleted.

## README

`README.md` should no longer present this repository as the full shadcn source
repo. It should describe the repository as an Accelerate Data wrapper for the
upstream shadcn skill, with links to upstream shadcn documentation.

The README should include:

- what this repo packages,
- what is synchronized from upstream,
- how the sync workflow works,
- which paths are wrapper-owned,
- where marketplace registry changes belong.

## Validation

The lean repo should validate only the surfaces it owns:

- `.claude-plugin/plugin.json` is valid JSON.
- `.codex-plugin/plugin.json` is valid JSON.
- `skills/shadcn/SKILL.md` exists.
- `skills/shadcn/rules/styling.md` exists as a representative nested skill file.
- The sync workflow YAML exists and uses sparse checkout for `skills/shadcn`.

Optional marketplace smoke tests can still be run from `plugin-marketplace`, but
they are not required for the local lean-migration commit.

## Migration Strategy

The migration should happen in one focused repository change:

1. Update docs and guidance.
2. Replace the sync workflow.
3. Remove upstream monorepo files and directories not needed by the wrapper.
4. Update README.
5. Validate manifests and the lean tree.

The implementation should not touch `skills/shadcn/` content.
