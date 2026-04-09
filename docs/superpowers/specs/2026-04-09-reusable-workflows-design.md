# Reusable Workflows — Design Spec

**Date:** 2026-04-09
**Status:** Approved

---

## Overview

Package the four AI git flow workflows for public distribution as GitHub reusable workflows. Consuming repos call them via `uses: graytonio/ai-based-git-flow/.github/workflows/<name>.yml@v1` with only an `ANTHROPIC_API_KEY` secret required for zero-config adoption. Label names and agent prompts are overridable via inputs for consumers who need customization.

This is Phase 2 of the project. Phase 1 (single-repo working implementation) is complete.

---

## Structure

No new files are created. Each of the four existing workflow files gets a `workflow_call` trigger added alongside its current event trigger, making it both self-triggering (for this repo's own use) and callable by external repos.

```
.github/
  workflows/
    plan.yml            # on: issues.opened + workflow_call
    revise-plan.yml     # on: issue_comment + workflow_call
    implement.yml       # on: pull_request.labeled + workflow_call
    revise-impl.yml     # on: pull_request_review_comment + workflow_call
  actions/
    planner/action.yml       # updated: plugin path, no other changes
    implementer/action.yml   # updated: plugin path, no other changes
```

---

## Consumer Setup

A consuming repo creates four thin wrapper files. Zero-config example:

```yaml
# .github/workflows/plan.yml in consumer repo
on:
  issues:
    types: [opened]
jobs:
  plan:
    uses: graytonio/ai-based-git-flow/.github/workflows/plan.yml@v1
    secrets:
      anthropic-api-key: ${{ secrets.ANTHROPIC_API_KEY }}
```

With customization:

```yaml
jobs:
  plan:
    uses: graytonio/ai-based-git-flow/.github/workflows/plan.yml@v1
    secrets:
      anthropic-api-key: ${{ secrets.ANTHROPIC_API_KEY }}
    with:
      needs-plan-review-label: ai-plan-review
      planner-initial-prompt: .github/prompts/my-custom-planner.md
```

Consumer prerequisites:
1. Add `ANTHROPIC_API_KEY` repository secret
2. Create four labels (or configure custom names via inputs)
3. Create the four wrapper workflow files

---

## Inputs & Secrets

All four workflows declare the following on their `workflow_call` trigger.

**Secret (all workflows):**

| Name | Required | Description |
|---|---|---|
| `anthropic-api-key` | yes | Anthropic API key for Claude |

**Inputs:**

| Input | Default | Used in |
|---|---|---|
| `needs-plan-review-label` | `needs-plan-review` | `plan.yml`, `revise-plan.yml` |
| `plan-approved-label` | `plan-approved` | `implement.yml` |
| `needs-code-review-label` | `needs-code-review` | `implement.yml`, `revise-impl.yml` |
| `needs-human-intervention-label` | `needs-human-intervention` | all four |
| `planner-initial-prompt` | _(empty)_ | `plan.yml` |
| `planner-revise-prompt` | _(empty)_ | `revise-plan.yml` |
| `implementer-initial-prompt` | _(empty)_ | `implement.yml` |
| `implementer-revise-prompt` | _(empty)_ | `revise-impl.yml` |

When a workflow fires from its native event (not `workflow_call`), inputs are empty. All label references use the pattern `${{ inputs.label-name || 'default-value' }}` to handle both cases.

When a prompt input is empty, the workflow uses the default prompt from the sparse checkout (see Resource Fetching). When provided, the value is a path relative to the consumer's repo root (e.g. `.github/prompts/custom-planner.md`).

---

## Resource Fetching

Each workflow adds a sparse checkout step immediately after checking out the consumer's repo. This fetches only the prompts directory and superpowers plugin from this repo into `_ai-git-flow/`:

```yaml
- name: Fetch ai-git-flow resources
  uses: actions/checkout@v4
  with:
    repository: graytonio/ai-based-git-flow
    ref: v1
    path: _ai-git-flow
    sparse-checkout: |
      .github/prompts
      .claude/plugins/superpowers
```

The `ref` matches the workflow version being run, preventing skew between the workflow logic and the prompts/plugin it loads. During development and pre-release testing the ref is `main`; it switches to the version tag at release time.

This step runs in every workflow including this repo's own runs, where it fetches from itself. This is a minor redundancy but keeps the code path consistent and avoids special-casing.

`_ai-git-flow/` is added to `.gitignore` so agents running inside the workflow cannot accidentally commit the temporary checkout.

**Prompt resolution** in each prepare-prompt step:

```python
import os
custom = os.environ.get('CUSTOM_PROMPT_PATH', '').strip()
path = custom if custom else '_ai-git-flow/.github/prompts/planner-initial.md'
t = open(path).read()
```

**Plugin path** in composite actions: updated from `$GITHUB_WORKSPACE/.claude/plugins/superpowers` to `$GITHUB_WORKSPACE/_ai-git-flow/.claude/plugins/superpowers`.

---

## Composite Action Changes

Both `planner/action.yml` and `implementer/action.yml` change only the plugin copy step:

```yaml
# Before
- name: Copy superpowers plugin
  shell: bash
  run: |
    mkdir -p ~/.claude/plugins
    cp -r "$GITHUB_WORKSPACE/.claude/plugins/superpowers" ~/.claude/plugins/superpowers

# After
- name: Copy superpowers plugin
  shell: bash
  run: |
    mkdir -p ~/.claude/plugins
    cp -r "$GITHUB_WORKSPACE/_ai-git-flow/.claude/plugins/superpowers" ~/.claude/plugins/superpowers
```

No other changes to the composite actions. The `uses: ./.github/actions/planner` references in the reusable workflows resolve to this repo's actions (not the caller's), so no path changes are needed there.

---

## Versioning

The repo uses semantic versioning with GitHub releases:

- `@v1.0.0` — exact pinned version
- `@v1` — floating major version tag; gets non-breaking updates automatically (recommended for consumers)

Breaking changes (new required inputs, removed inputs, label name changes) increment the major version. The sparse checkout `ref` in each workflow always matches its own tag so prompts and plugin are always version-consistent.

Initial release: cut `v1.0.0` and move the `v1` floating tag to the same commit after implementation is complete and tested.

---

## What Is Not Changing

- Workflow logic, guard conditions, concurrency groups, and error handling are unchanged
- The four-label state machine is unchanged
- Branch naming convention (`plan/issue-{N}`) is unchanged
- Agent invocation flags (`--dangerously-skip-permissions --max-turns 30`) are unchanged
- Runner is fixed at `ubuntu-latest` (not configurable)
