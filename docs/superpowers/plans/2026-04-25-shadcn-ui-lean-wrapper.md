# shadcn/ui Lean Wrapper Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Convert `accelerate-data/shadcn-ui` from a full upstream-shaped fork into a lean plugin wrapper that synchronizes only `skills/shadcn/` from `shadcn-ui/ui`.

**Architecture:** Keep the plugin manifests, docs, workflow, and governance files as wrapper-owned content. Treat `skills/shadcn/` as the only upstream-owned payload, and update GitHub Actions to sparse-checkout that directory into a sync branch instead of merging the full upstream monorepo.

**Tech Stack:** GitHub Actions, `actions/checkout@v4` sparse checkout, Claude plugin manifest JSON, Codex plugin manifest JSON, shell validation, git.

---

## File Structure

- Modify: `.github/workflows/sync-upstream.yml`
  - Replace full upstream remote merge with sparse checkout of `skills/shadcn`.
- Modify: `AGENTS.md`
  - Narrow upstream-owned path guidance to `skills/shadcn/`.
- Modify: `README.md`
  - Reframe this repository as a lean Accelerate Data shadcn skill wrapper.
- Keep: `.claude-plugin/plugin.json`
  - Claude plugin manifest.
- Keep: `.codex-plugin/plugin.json`
  - Codex plugin manifest.
- Keep: `skills/shadcn/**`
  - Upstream-owned shadcn skill payload.
- Delete: `apps/`
  - Upstream app source not needed by the marketplace plugin.
- Delete: `packages/`
  - Upstream package source not needed by the marketplace plugin.
- Delete: `templates/`
  - Upstream templates not needed by the marketplace plugin.
- Delete: package and upstream monorepo config files that become unused after removing upstream source:
  - `.changeset/`
  - `.cursor/`
  - `.cursor-plugin/`
  - `.vscode/`
  - `.commitlintrc.json`
  - `.editorconfig`
  - `.eslintignore`
  - `.eslintrc.json`
  - `.kodiak.toml`
  - `.npmrc`
  - `.nvmrc`
  - `.prettierignore`
  - `CONTRIBUTING.md`
  - `package.json`
  - `pnpm-lock.yaml`
  - `pnpm-workspace.yaml`
  - `prettier.config.cjs`
  - `scripts/sync-templates.sh`
  - `tsconfig.json`
  - `turbo.json`
  - `vitest.config.ts`
  - `vitest.workspace.ts`
- Review and delete unrelated upstream workflows:
  - `.github/workflows/code-check.yml`
  - `.github/workflows/issue-stale.yml`
  - `.github/workflows/prerelease-comment.yml`
  - `.github/workflows/release.yml`
  - `.github/workflows/test.yml`
- Keep or delete after review:
  - `LICENSE.md`
  - `SECURITY.md`
  - `.github/workflows/validate-registries.yml`
  - `.github/FUNDING.yml`
  - `.github/ISSUE_TEMPLATE/**`
  - `.github/DISCUSSION_TEMPLATE/**`

## Task 1: Verify Baseline Repository State

**Files:**
- Read: `AGENTS.md`
- Read: `.github/workflows/sync-upstream.yml`
- Read: `.claude-plugin/plugin.json`
- Read: `.codex-plugin/plugin.json`
- Read: `skills/shadcn/SKILL.md`

- [ ] **Step 1: Confirm working tree is clean**

Run:

```bash
git status --short --branch
```

Expected: branch information only, with no modified or untracked files.

- [ ] **Step 2: Confirm the plugin payload exists**

Run:

```bash
test -f .claude-plugin/plugin.json
test -f .codex-plugin/plugin.json
test -f skills/shadcn/SKILL.md
test -f skills/shadcn/rules/styling.md
```

Expected: all commands exit 0.

- [ ] **Step 3: Record current top-level footprint**

Run:

```bash
du -sh . skills apps packages templates .git
```

Expected: output shows `skills` is much smaller than `apps`, `packages`, and the full repository.

## Task 2: Update Agent Guidance

**Files:**
- Modify: `AGENTS.md`

- [ ] **Step 1: Replace upstream ownership guidance**

Edit `AGENTS.md` to this content:

```markdown
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
```

- [ ] **Step 2: Review the guidance**

Run:

```bash
sed -n '1,180p' AGENTS.md
```

Expected: the only upstream-owned path is `skills/shadcn/`.

## Task 3: Replace Full Upstream Merge With Sparse Skill Sync

**Files:**
- Modify: `.github/workflows/sync-upstream.yml`

- [ ] **Step 1: Replace workflow content**

Edit `.github/workflows/sync-upstream.yml` to this content:

```yaml
name: Sync upstream shadcn skill

on:
  schedule:
    - cron: "0 9 * * 1"
  workflow_dispatch:

permissions:
  contents: write
  pull-requests: write

jobs:
  sync:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout wrapper repo
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Configure git identity
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"

      - name: Checkout upstream shadcn skill
        uses: actions/checkout@v4
        with:
          repository: shadcn-ui/ui
          ref: main
          path: upstream
          sparse-checkout: |
            skills/shadcn
          sparse-checkout-cone-mode: false

      - name: Create sync branch
        run: git checkout -B sync/upstream-shadcn-skill

      - name: Sync upstream skill
        run: |
          rm -rf skills/shadcn
          mkdir -p skills
          cp -R upstream/skills/shadcn skills/shadcn

      - name: Validate wrapper files
        run: |
          python3 -m json.tool .claude-plugin/plugin.json >/dev/null
          python3 -m json.tool .codex-plugin/plugin.json >/dev/null
          test -f skills/shadcn/SKILL.md
          test -f skills/shadcn/rules/styling.md

      - name: Detect changes
        id: changes
        run: |
          if git diff --quiet -- skills/shadcn; then
            echo "changed=false" >> "$GITHUB_OUTPUT"
          else
            echo "changed=true" >> "$GITHUB_OUTPUT"
          fi

      - name: Commit synced skill
        if: steps.changes.outputs.changed == 'true'
        run: |
          git add skills/shadcn
          git commit -m "chore: sync upstream shadcn skill"

      - name: Push sync branch
        if: steps.changes.outputs.changed == 'true'
        run: git push origin sync/upstream-shadcn-skill --force-with-lease

      - name: Open sync pull request
        if: steps.changes.outputs.changed == 'true'
        env:
          GH_TOKEN: ${{ github.token }}
        run: |
          gh pr view sync/upstream-shadcn-skill --json number >/dev/null 2>&1 || \
            gh pr create \
              --base main \
              --head sync/upstream-shadcn-skill \
              --title "Sync upstream shadcn skill" \
              --body-file - <<'BODY'
            Automated sync of `skills/shadcn/` from `shadcn-ui/ui`.

            Review upstream-owned skill changes before merge. This wrapper repo
            intentionally does not sync upstream `apps/`, `packages/`, or
            `templates/`.
          BODY
```

- [ ] **Step 2: Verify sparse checkout wording**

Run:

```bash
rg -n "sparse-checkout|skills/shadcn|upstream-shadcn-skill|git merge" .github/workflows/sync-upstream.yml
```

Expected: output includes `sparse-checkout`, `skills/shadcn`, and `upstream-shadcn-skill`; output does not include `git merge`.

## Task 4: Rewrite README for Lean Wrapper Purpose

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Replace README content**

Edit `README.md` to this content:

```markdown
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
```

- [ ] **Step 2: Review README**

Run:

```bash
sed -n '1,220p' README.md
```

Expected: README describes a lean skill wrapper and links to upstream shadcn docs/source.

## Task 5: Remove Full Upstream Monorepo Content

**Files:**
- Delete: `apps/`
- Delete: `packages/`
- Delete: `templates/`
- Delete: `.changeset/`
- Delete: `.cursor/`
- Delete: `.cursor-plugin/`
- Delete: `.vscode/`
- Delete: `.commitlintrc.json`
- Delete: `.editorconfig`
- Delete: `.eslintignore`
- Delete: `.eslintrc.json`
- Delete: `.kodiak.toml`
- Delete: `.npmrc`
- Delete: `.nvmrc`
- Delete: `.prettierignore`
- Delete: `CONTRIBUTING.md`
- Delete: `package.json`
- Delete: `pnpm-lock.yaml`
- Delete: `pnpm-workspace.yaml`
- Delete: `prettier.config.cjs`
- Delete: `scripts/sync-templates.sh`
- Delete: `tsconfig.json`
- Delete: `turbo.json`
- Delete: `vitest.config.ts`
- Delete: `vitest.workspace.ts`

- [ ] **Step 1: Remove upstream source directories**

Run:

```bash
git rm -r apps packages templates
```

Expected: git stages deletions for upstream app, package, and template files.

- [ ] **Step 2: Remove unused upstream repo scaffolding**

Run:

```bash
git rm -r .changeset .cursor .cursor-plugin .vscode
git rm .commitlintrc.json .editorconfig .eslintignore .eslintrc.json .kodiak.toml .npmrc .nvmrc .prettierignore
git rm CONTRIBUTING.md package.json pnpm-lock.yaml pnpm-workspace.yaml prettier.config.cjs tsconfig.json turbo.json vitest.config.ts vitest.workspace.ts
git rm scripts/sync-templates.sh
```

Expected: git stages deletions for upstream-only configuration and scripts.

- [ ] **Step 3: Review remaining tracked top-level files**

Run:

```bash
git ls-files | awk -F/ '{print $1}' | sort | uniq -c | sort -nr
```

Expected: remaining tracked files are concentrated under `skills`, `docs`, `.github`, `.claude-plugin`, `.codex-plugin`, and wrapper metadata.

## Task 6: Review and Prune Unrelated GitHub Files

**Files:**
- Delete: `.github/workflows/code-check.yml`
- Delete: `.github/workflows/issue-stale.yml`
- Delete: `.github/workflows/prerelease-comment.yml`
- Delete: `.github/workflows/release.yml`
- Delete: `.github/workflows/test.yml`
- Review: `.github/workflows/validate-registries.yml`
- Review: `.github/FUNDING.yml`
- Review: `.github/ISSUE_TEMPLATE/**`
- Review: `.github/DISCUSSION_TEMPLATE/**`

- [ ] **Step 1: Inspect GitHub workflows**

Run:

```bash
find .github/workflows -maxdepth 1 -type f -print | sort
```

Expected: output lists the current workflow files.

- [ ] **Step 2: Delete upstream-only workflows**

Run:

```bash
git rm .github/workflows/code-check.yml .github/workflows/issue-stale.yml .github/workflows/prerelease-comment.yml .github/workflows/release.yml .github/workflows/test.yml
```

Expected: git stages deletions for workflows that test or release the upstream monorepo.

- [ ] **Step 3: Inspect remaining GitHub metadata**

Run:

```bash
find .github -maxdepth 3 -type f | sort
```

Expected: output shows only wrapper-relevant GitHub files or files intentionally kept for repository governance.

- [ ] **Step 4: Decide whether `validate-registries.yml` is wrapper-relevant**

Run:

```bash
sed -n '1,220p' .github/workflows/validate-registries.yml
```

Expected: if this workflow validates marketplace/source metadata used by this wrapper, keep it; if it validates removed upstream registry code, delete it with `git rm .github/workflows/validate-registries.yml`.

## Task 7: Validate Lean Repository

**Files:**
- Read: `.claude-plugin/plugin.json`
- Read: `.codex-plugin/plugin.json`
- Read: `.github/workflows/sync-upstream.yml`
- Read: `AGENTS.md`
- Read: `README.md`
- Read: `skills/shadcn/SKILL.md`

- [ ] **Step 1: Validate plugin manifests**

Run:

```bash
python3 -m json.tool .claude-plugin/plugin.json >/dev/null
python3 -m json.tool .codex-plugin/plugin.json >/dev/null
```

Expected: both commands exit 0.

- [ ] **Step 2: Validate required skill files**

Run:

```bash
test -f skills/shadcn/SKILL.md
test -f skills/shadcn/cli.md
test -f skills/shadcn/customization.md
test -f skills/shadcn/mcp.md
test -f skills/shadcn/rules/styling.md
```

Expected: all commands exit 0.

- [ ] **Step 3: Validate removed upstream directories are gone**

Run:

```bash
test ! -e apps
test ! -e packages
test ! -e templates
```

Expected: all commands exit 0.

- [ ] **Step 4: Validate workflow uses sparse sync**

Run:

```bash
rg -n "sparse-checkout|skills/shadcn|git merge|upstream/main" .github/workflows/sync-upstream.yml
```

Expected: output includes `sparse-checkout` and `skills/shadcn`; output does not include `git merge` or `upstream/main`.

- [ ] **Step 5: Check final footprint**

Run:

```bash
du -sh . skills .git
```

Expected: repository working tree is materially smaller than the baseline recorded in Task 1.

- [ ] **Step 6: Review staged diff**

Run:

```bash
git diff --stat
git diff -- .github/workflows/sync-upstream.yml AGENTS.md README.md
```

Expected: diff shows lean wrapper docs/workflow updates and deletions of upstream monorepo content. No files under `skills/shadcn/` are modified.

## Task 8: Commit the Migration

**Files:**
- Modify: `.github/workflows/sync-upstream.yml`
- Modify: `AGENTS.md`
- Modify: `README.md`
- Delete: upstream monorepo files listed in Tasks 5 and 6

- [ ] **Step 1: Confirm `skills/shadcn/` is untouched**

Run:

```bash
git diff --name-only -- skills/shadcn
```

Expected: no output.

- [ ] **Step 2: Stage all changes**

Run:

```bash
git add -A
```

Expected: command exits 0.

- [ ] **Step 3: Review staged changes**

Run:

```bash
git diff --cached --stat
git diff --cached -- .github/workflows/sync-upstream.yml AGENTS.md README.md
```

Expected: staged diff matches the lean wrapper migration.

- [ ] **Step 4: Commit**

Run:

```bash
git commit -m "chore: convert shadcn wrapper to sparse skill sync"
```

Expected: commit succeeds.

## Self-Review Checklist

- [ ] The plan keeps `skills/shadcn/` unchanged during the lean migration.
- [ ] The sync workflow copies only `skills/shadcn/` from upstream.
- [ ] The repository no longer needs upstream `apps/`, `packages/`, or `templates/`.
- [ ] `AGENTS.md` reflects the new ownership boundary.
- [ ] `README.md` describes this repo as a wrapper, not the full shadcn source.
- [ ] Validation commands are local and deterministic.
