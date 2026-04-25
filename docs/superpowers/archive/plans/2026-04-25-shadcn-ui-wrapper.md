# shadcn/ui Wrapper Implementation Plan

> Archived: this plan described the initial full-fork wrapper setup. It is
> superseded by `docs/superpowers/plans/2026-04-25-shadcn-ui-lean-wrapper.md`,
> which documents the current lean `skills/shadcn/` wrapper migration.

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Package the forked shadcn skill for Accelerate Data Claude and Codex marketplaces while preserving upstream shadcn source content.

**Architecture:** Keep upstream source and `skills/shadcn/` in place. Add plugin manifests, agent governance notes, an upstream sync workflow, and a Codex smoke-test gate because the skill includes Claude-style metadata.

**Tech Stack:** Claude plugin manifest JSON, Codex plugin manifest JSON, GitHub Actions, pnpm workspace, shadcn CLI, marketplace `url` sources.

---

## File Structure

- Create: `.claude-plugin/plugin.json`
  - Claude plugin manifest for the fork.
- Create: `.codex-plugin/plugin.json`
  - Codex plugin manifest for the fork.
- Create: `AGENTS.md`
  - Agent guidance for upstream-owned shadcn content.
- Create: `.github/workflows/sync-upstream.yml`
  - Scheduled/manual workflow that syncs from `shadcn-ui/ui`.
- Read: `skills/shadcn/SKILL.md`
  - Runtime skill file with Claude-style metadata requiring compatibility review.
- Modify later in `/Users/hbanerjee/src/plugin-marketplace`: `.claude-plugin/marketplace.json`
  - Point the Claude marketplace entry at `accelerate-data/shadcn-ui`.
- Modify later in `/Users/hbanerjee/src/plugin-marketplace`: `.agents/plugins/marketplace.json`
  - Point the Codex marketplace entry at `accelerate-data/shadcn-ui` after smoke testing.

## Task 1: Add Claude Plugin Manifest

**Files:**
- Create: `.claude-plugin/plugin.json`

- [x] **Step 1: Create manifest directory**

Run:

```bash
mkdir -p .claude-plugin
```

Expected: command exits 0.

- [x] **Step 2: Add `.claude-plugin/plugin.json`**

Create `.claude-plugin/plugin.json` with:

```json
{
  "name": "shadcn",
  "version": "0.1.0",
  "description": "shadcn/ui skill packaged for Accelerate Data Claude marketplace installation.",
  "author": {
    "name": "Accelerate Data",
    "url": "https://github.com/accelerate-data"
  },
  "repository": "https://github.com/accelerate-data/shadcn-ui"
}
```

- [x] **Step 3: Validate JSON**

Run:

```bash
python3 -m json.tool .claude-plugin/plugin.json
```

Expected: pretty-printed JSON and exit 0.

- [x] **Step 4: Commit**

Run:

```bash
git add .claude-plugin/plugin.json
git commit -m "feat: add Claude plugin manifest"
```

Expected: commit succeeds.

## Task 2: Add Codex Plugin Manifest

**Files:**
- Create: `.codex-plugin/plugin.json`

- [x] **Step 1: Create manifest directory**

Run:

```bash
mkdir -p .codex-plugin
```

Expected: command exits 0.

- [x] **Step 2: Add `.codex-plugin/plugin.json`**

Create `.codex-plugin/plugin.json` with:

```json
{
  "name": "shadcn",
  "version": "0.1.0",
  "description": "shadcn/ui skill packaged for Accelerate Data Codex marketplace installation.",
  "author": {
    "name": "Accelerate Data",
    "url": "https://github.com/accelerate-data"
  },
  "repository": "https://github.com/accelerate-data/shadcn-ui"
}
```

- [x] **Step 3: Validate JSON**

Run:

```bash
python3 -m json.tool .codex-plugin/plugin.json
```

Expected: pretty-printed JSON and exit 0.

- [x] **Step 4: Commit**

Run:

```bash
git add .codex-plugin/plugin.json
git commit -m "feat: add Codex plugin manifest"
```

Expected: commit succeeds.

## Task 3: Add Agent Guidance for Upstream-Owned Content

**Files:**
- Create: `AGENTS.md`

- [x] **Step 1: Create `AGENTS.md`**

Create `AGENTS.md` with:

```markdown
# AGENTS.md

This repository is an Accelerate Data-maintained fork of `shadcn-ui/ui`.

## Vendored Upstream Content

The shadcn source under `skills/`, `packages/`, `apps/`, and `templates/` is
upstream-owned content from `shadcn-ui/ui`.

AI agents may edit wrapper-owned files such as `.claude-plugin/`,
`.codex-plugin/`, `.github/workflows/`, documentation, and validation scripts as
normal.

Before editing files under `skills/`, `packages/`, `apps/`, or `templates/`, AI
agents must warn the user that the file is upstream-owned and explain that the
change may diverge from future upstream syncs. Proceed only when the user
explicitly confirms the local divergence or when the task is specifically to
resolve an upstream sync conflict.

## Repository Purpose

This repository packages the upstream shadcn skill for marketplace installation
in Claude and Codex. It should preserve upstream shadcn behavior unless a
reviewed local divergence is explicitly requested.
```

- [x] **Step 2: Review the rendered guidance**

Run:

```bash
sed -n '1,140p' AGENTS.md
```

Expected: the upstream-owned content warning is present and clear.

- [x] **Step 3: Commit**

Run:

```bash
git add AGENTS.md
git commit -m "docs: add vendored shadcn guidance"
```

Expected: commit succeeds.

## Task 4: Add Scheduled Upstream Sync Workflow

**Files:**
- Create: `.github/workflows/sync-upstream.yml`

- [x] **Step 1: Create workflow directory**

Run:

```bash
mkdir -p .github/workflows
```

Expected: command exits 0.

- [x] **Step 2: Add workflow**

Create `.github/workflows/sync-upstream.yml` with:

```yaml
name: Sync upstream shadcn/ui

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
      - name: Checkout fork
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Configure git identity
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"

      - name: Add upstream remote
        run: git remote add upstream https://github.com/shadcn-ui/ui.git

      - name: Fetch upstream
        run: git fetch upstream main

      - name: Create sync branch
        run: git checkout -B sync/upstream-shadcn-ui

      - name: Merge upstream
        run: git merge upstream/main --no-edit

      - name: Validate wrapper files
        run: |
          python3 -m json.tool .claude-plugin/plugin.json >/dev/null
          python3 -m json.tool .codex-plugin/plugin.json >/dev/null
          test -f skills/shadcn/SKILL.md

      - name: Push sync branch
        run: git push origin sync/upstream-shadcn-ui --force-with-lease

      - name: Open sync pull request
        env:
          GH_TOKEN: ${{ github.token }}
        run: |
          gh pr view sync/upstream-shadcn-ui --json number >/dev/null 2>&1 || \
            gh pr create \
              --base main \
              --head sync/upstream-shadcn-ui \
              --title "Sync upstream shadcn-ui/ui" \
              --body-file - <<'BODY'
            Automated sync from `shadcn-ui/ui`.

            Review upstream-owned `skills/`, `packages/`, `apps/`, and `templates/`
            changes before merge.
          BODY
```

- [x] **Step 3: Validate workflow text**

Run:

```bash
python3 - <<'PY'
from pathlib import Path

path = Path(".github/workflows/sync-upstream.yml")
text = path.read_text()
for required in [
    "name: Sync upstream shadcn/ui",
    "https://github.com/shadcn-ui/ui.git",
    "test -f skills/shadcn/SKILL.md",
    "git push origin sync/upstream-shadcn-ui --force-with-lease",
    "gh pr create",
]:
    if required not in text:
        raise SystemExit(f"missing required workflow text: {required}")
print("workflow text checks passed")
PY
```

Expected: `workflow text checks passed`.

- [x] **Step 4: Commit**

Run:

```bash
git add .github/workflows/sync-upstream.yml
git commit -m "ci: add upstream sync workflow"
```

Expected: commit succeeds.

## Task 5: Smoke Test Plugin Shape and Codex Compatibility

**Files:**
- Read: `.claude-plugin/plugin.json`
- Read: `.codex-plugin/plugin.json`
- Read: `skills/shadcn/SKILL.md`

- [x] **Step 1: Run local validation checks**

Run:

```bash
python3 -m json.tool .claude-plugin/plugin.json >/dev/null
python3 -m json.tool .codex-plugin/plugin.json >/dev/null
test -f skills/shadcn/SKILL.md
```

Expected: command exits 0.

- [x] **Step 2: Inspect Claude-specific skill metadata**

Run:

```bash
sed -n '1,40p' skills/shadcn/SKILL.md
```

Expected: the output shows the current `allowed-tools` and command injection metadata for review.

- [x] **Step 3: Install from local marketplace source in Claude and Codex**

Run the repository-specific plugin install smoke checks available in the current
environment. At minimum, verify that Codex can read `.codex-plugin/plugin.json`
and discover `skills/shadcn/SKILL.md`.

Expected: Codex installation succeeds or produces a specific incompatibility
error tied to `allowed-tools` or command injection syntax.

- [x] **Step 4: Decide whether a Codex compatibility patch is needed**

If Codex installation succeeds, do not change `skills/shadcn/SKILL.md`.

If Codex installation fails because of Claude-specific metadata, document the
exact error and add the smallest compatibility patch that preserves Claude
behavior. Re-run the smoke check after the patch.

- [x] **Step 5: Commit compatibility patch if needed**

If Step 4 required a change, commit it:

```bash
git add skills/shadcn/SKILL.md
git commit -m "fix: make shadcn skill compatible with Codex"
```

Expected: commit succeeds only if a compatibility patch was required.

## Task 6: Update Plugin Marketplace

**Files:**
- Modify: `/Users/hbanerjee/src/plugin-marketplace/.claude-plugin/marketplace.json`
- Modify: `/Users/hbanerjee/src/plugin-marketplace/.agents/plugins/marketplace.json`
- Modify if needed: `/Users/hbanerjee/src/plugin-marketplace/.claude-plugin/plugin.json`

- [x] **Step 1: Update Claude marketplace entry**

In `/Users/hbanerjee/src/plugin-marketplace/.claude-plugin/marketplace.json`, ensure the `shadcn` entry points at the maintained fork:

```json
{
  "name": "shadcn",
  "description": "Skills to manage shadcn components and projects.",
  "strict": false,
  "source": {
    "source": "url",
    "url": "https://github.com/accelerate-data/shadcn-ui.git"
  }
}
```

- [x] **Step 2: Update Codex marketplace entry**

In `/Users/hbanerjee/src/plugin-marketplace/.agents/plugins/marketplace.json`, ensure the `shadcn` entry points at the same maintained fork:

```json
{
  "name": "shadcn",
  "source": {
    "source": "url",
    "url": "https://github.com/accelerate-data/shadcn-ui.git"
  },
  "policy": {
    "installation": "AVAILABLE",
    "authentication": "ON_INSTALL"
  },
  "category": "Design"
}
```

- [x] **Step 3: Keep marketplace versions aligned**

If `.claude-plugin/marketplace.json` `metadata.version` changes, update
`/Users/hbanerjee/src/plugin-marketplace/.claude-plugin/plugin.json` to the same version.

- [x] **Step 4: Validate marketplace**

Run from `/Users/hbanerjee/src/plugin-marketplace`:

```bash
python3 scripts/validate-marketplace.py --base-ref HEAD
python3 -m unittest tests/test_validate_marketplace.py
codex plugin marketplace add .
```

Expected:

```text
marketplace validation passed
Ran 21 tests
OK
Marketplace `ad-internal-marketplace` is already added
```

- [x] **Step 5: Commit marketplace update**

Run from `/Users/hbanerjee/src/plugin-marketplace`:

```bash
git add .claude-plugin/marketplace.json .agents/plugins/marketplace.json .claude-plugin/plugin.json
git commit -m "feat: add maintained shadcn plugin source"
```

Expected: commit succeeds.

## Self-Review

- Spec coverage: The plan covers plugin manifests, agent warning guidance,
  upstream sync workflow, Codex compatibility review, validation, and marketplace
  source updates.
- Placeholder scan: No placeholder markers or unspecified implementation steps remain.
- Scope check: The plan is one focused wrapper-and-marketplace task. shadcn
  source behavior changes are intentionally out of scope unless Codex smoke
  testing proves a compatibility patch is required.
