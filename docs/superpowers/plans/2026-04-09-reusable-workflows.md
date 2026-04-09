# Reusable Workflows Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `workflow_call` triggers to all four workflows so external repos can call them as reusable workflows, with sensible defaults for labels and prompts and the ability to override both.

**Architecture:** Each workflow gains a `workflow_call` trigger alongside its existing event trigger. A sparse checkout of `graytonio/ai-based-git-flow@main` into `_ai-git-flow/` provides the default prompts and superpowers plugin for every run — including runs in this repo. Composite actions are updated to read the plugin from `_ai-git-flow/` instead of the workspace root. All hardcoded label names become `${{ inputs.label-name || 'default' }}` expressions.

**Tech Stack:** GitHub Actions YAML, `actionlint` for validation, `gh` CLI for GitHub API calls, Python 3 for prompt injection.

---

## File Map

| File | Change |
|---|---|
| `.github/actions/planner/action.yml` | Update plugin copy path |
| `.github/actions/implementer/action.yml` | Update plugin copy path |
| `.github/workflows/plan.yml` | Add `workflow_call`, sparse checkout, parameterize labels + prompt |
| `.github/workflows/revise-plan.yml` | Add `workflow_call`, sparse checkout, parameterize labels + prompt |
| `.github/workflows/implement.yml` | Add `workflow_call`, sparse checkout, move label guard to step, parameterize labels + prompt |
| `.github/workflows/revise-impl.yml` | Add `workflow_call`, sparse checkout, parameterize labels + prompt |

---

### Task 1: Update composite actions plugin path

Both composite actions currently copy the plugin from `$GITHUB_WORKSPACE/.claude/plugins/superpowers`. After this change they read from `$GITHUB_WORKSPACE/_ai-git-flow/.claude/plugins/superpowers`, which is where the sparse checkout step deposits it.

**Files:**
- Modify: `.github/actions/planner/action.yml`
- Modify: `.github/actions/implementer/action.yml`

- [ ] **Step 1: Update planner action plugin path**

Replace the "Copy superpowers plugin" step in `.github/actions/planner/action.yml`:

```yaml
    - name: Copy superpowers plugin
      shell: bash
      run: |
        if [ ! -d "$GITHUB_WORKSPACE/_ai-git-flow/.claude/plugins/superpowers" ]; then
          echo "ERROR: superpowers plugin not found at $GITHUB_WORKSPACE/_ai-git-flow/.claude/plugins/superpowers" >&2
          exit 1
        fi
        mkdir -p ~/.claude/plugins
        cp -r "$GITHUB_WORKSPACE/_ai-git-flow/.claude/plugins/superpowers" ~/.claude/plugins/superpowers
```

- [ ] **Step 2: Update implementer action plugin path**

Apply the identical change to `.github/actions/implementer/action.yml` — same step, same new content.

- [ ] **Step 3: Validate both actions with actionlint**

```bash
actionlint .github/actions/planner/action.yml .github/actions/implementer/action.yml
```

Expected: no output (clean).

- [ ] **Step 4: Commit**

```bash
git add .github/actions/planner/action.yml .github/actions/implementer/action.yml
git commit -m "fix: read superpowers plugin from _ai-git-flow sparse checkout"
```

---

### Task 2: Update `plan.yml`

Adds `workflow_call` trigger with secret + three inputs. Adds sparse checkout step. Updates prompt resolution to use `CUSTOM_PROMPT` env var with fallback to `_ai-git-flow/` default. Parameterizes both label names.

**Files:**
- Modify: `.github/workflows/plan.yml`

- [ ] **Step 1: Replace `plan.yml` with the updated content**

Write the following to `.github/workflows/plan.yml`:

```yaml
name: AI Plan

on:
  issues:
    types: [opened]
  workflow_call:
    secrets:
      anthropic-api-key:
        description: 'Anthropic API key'
        required: true
    inputs:
      needs-plan-review-label:
        description: 'Label to apply when plan is ready for review'
        required: false
        default: 'needs-plan-review'
        type: string
      needs-human-intervention-label:
        description: 'Label to apply on agent failure'
        required: false
        default: 'needs-human-intervention'
        type: string
      planner-initial-prompt:
        description: 'Path to custom planner prompt (relative to repo root). Omit to use built-in default.'
        required: false
        default: ''
        type: string

permissions:
  contents: write
  pull-requests: write

jobs:
  plan:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          fetch-depth: 0

      - name: Fetch ai-git-flow resources
        uses: actions/checkout@v4
        with:
          repository: graytonio/ai-based-git-flow
          ref: main
          path: _ai-git-flow
          sparse-checkout: |
            .github/prompts
            .claude/plugins/superpowers

      - name: Prepare prompt
        env:
          ISSUE_NUMBER: ${{ github.event.issue.number }}
          ISSUE_TITLE: ${{ github.event.issue.title }}
          ISSUE_BODY: ${{ github.event.issue.body }}
          CUSTOM_PROMPT: ${{ inputs.planner-initial-prompt }}
        run: |
          python3 -c "
          import os
          custom = os.environ.get('CUSTOM_PROMPT', '').strip()
          path = custom if custom else '_ai-git-flow/.github/prompts/planner-initial.md'
          t = open(path).read()
          t = t.replace('{{ISSUE_NUMBER}}', os.environ['ISSUE_NUMBER'])
          t = t.replace('{{ISSUE_TITLE}}', os.environ['ISSUE_TITLE'])
          t = t.replace('{{ISSUE_BODY}}', os.environ['ISSUE_BODY'])
          open(os.environ['RUNNER_TEMP'] + '/prompt.md', 'w').write(t)
          "

      - name: Run planner agent
        uses: ./.github/actions/planner
        with:
          anthropic-api-key: ${{ secrets.anthropic-api-key || secrets.ANTHROPIC_API_KEY }}
          prompt-file: ${{ env.RUNNER_TEMP }}/prompt.md

      - name: Apply needs-plan-review label
        if: success()
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          ISSUE_NUMBER: ${{ github.event.issue.number }}
          LABEL: ${{ inputs.needs-plan-review-label || 'needs-plan-review' }}
        run: |
          BRANCH="plan/issue-${ISSUE_NUMBER}"
          PR_NUMBER=$(gh pr list --head "$BRANCH" --json number --jq '.[0].number' 2>/dev/null || echo "")
          if [ -n "$PR_NUMBER" ] && [ "$PR_NUMBER" != "null" ]; then
            gh pr edit "$PR_NUMBER" --add-label "$LABEL"
          fi

      - name: Apply needs-human-intervention on failure
        if: failure()
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          ISSUE_NUMBER: ${{ github.event.issue.number }}
          LABEL: ${{ inputs.needs-human-intervention-label || 'needs-human-intervention' }}
        run: |
          BRANCH="plan/issue-${ISSUE_NUMBER}"
          PR_NUMBER=$(gh pr list --head "$BRANCH" --json number --jq '.[0].number' 2>/dev/null || echo "")
          if [ -n "$PR_NUMBER" ]; then
            gh pr edit "$PR_NUMBER" --add-label "$LABEL"
          fi
```

- [ ] **Step 2: Validate with actionlint**

```bash
actionlint .github/workflows/plan.yml
```

Expected: no output (clean).

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/plan.yml
git commit -m "feat: add workflow_call trigger to plan.yml"
```

---

### Task 3: Update `revise-plan.yml`

Same pattern as Task 2. The label check in the guard step uses `--arg` to pass the label name into jq rather than hardcoding it.

**Files:**
- Modify: `.github/workflows/revise-plan.yml`

- [ ] **Step 1: Replace `revise-plan.yml` with the updated content**

```yaml
name: AI Revise Plan

on:
  issue_comment:
    types: [created]
  workflow_call:
    secrets:
      anthropic-api-key:
        description: 'Anthropic API key'
        required: true
    inputs:
      needs-plan-review-label:
        description: 'Label that marks a PR as awaiting plan review'
        required: false
        default: 'needs-plan-review'
        type: string
      needs-human-intervention-label:
        description: 'Label to apply on agent failure'
        required: false
        default: 'needs-human-intervention'
        type: string
      planner-revise-prompt:
        description: 'Path to custom plan revision prompt (relative to repo root). Omit to use built-in default.'
        required: false
        default: ''
        type: string

permissions:
  contents: write
  pull-requests: write

concurrency:
  group: pr-${{ github.event.issue.number }}
  cancel-in-progress: true

jobs:
  guard:
    runs-on: ubuntu-latest
    outputs:
      should-run: ${{ steps.check.outputs.should-run }}
      pr-number: ${{ steps.check.outputs.pr-number }}
      plan-path: ${{ steps.check.outputs.plan-path }}
    steps:
      - name: Check conditions
        id: check
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          COMMENT_AUTHOR: ${{ github.event.comment.user.login }}
          ISSUE_NUMBER: ${{ github.event.issue.number }}
          PR_URL: ${{ github.event.issue.pull_request.url }}
          NEEDS_PLAN_REVIEW_LABEL: ${{ inputs.needs-plan-review-label || 'needs-plan-review' }}
        run: |
          # Skip bot comments
          if [ "$COMMENT_AUTHOR" = "github-actions[bot]" ]; then
            echo "should-run=false" >> "$GITHUB_OUTPUT"
            exit 0
          fi
          # Skip if this is a plain issue, not a PR
          if [ -z "$PR_URL" ]; then
            echo "should-run=false" >> "$GITHUB_OUTPUT"
            exit 0
          fi
          # Check PR has the plan-review label
          HAS_LABEL=$(gh pr view "$ISSUE_NUMBER" --json labels \
            --jq --arg label "$NEEDS_PLAN_REVIEW_LABEL" \
            '[.labels[].name] | contains([$label]) | if . then "1" else "0" end')
          if [ "$HAS_LABEL" != "1" ]; then
            echo "should-run=false" >> "$GITHUB_OUTPUT"
            exit 0
          fi
          {
            echo "should-run=true"
            echo "pr-number=$ISSUE_NUMBER"
            echo "plan-path=.claude/plans/issue-${ISSUE_NUMBER}.md"
          } >> "$GITHUB_OUTPUT"

  revise:
    needs: guard
    if: needs.guard.outputs.should-run == 'true'
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          fetch-depth: 0

      - name: Fetch ai-git-flow resources
        uses: actions/checkout@v4
        with:
          repository: graytonio/ai-based-git-flow
          ref: main
          path: _ai-git-flow
          sparse-checkout: |
            .github/prompts
            .claude/plugins/superpowers

      - name: Checkout PR branch
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          PR_NUMBER: ${{ needs.guard.outputs.pr-number }}
        run: gh pr checkout "$PR_NUMBER"

      - name: Prepare prompt
        env:
          ISSUE_NUMBER: ${{ github.event.issue.number }}
          PLAN_PATH: ${{ needs.guard.outputs.plan-path }}
          COMMENT_BODY: ${{ github.event.comment.body }}
          COMMENT_AUTHOR: ${{ github.event.comment.user.login }}
          CUSTOM_PROMPT: ${{ inputs.planner-revise-prompt }}
        run: |
          python3 -c "
          import os
          custom = os.environ.get('CUSTOM_PROMPT', '').strip()
          path = custom if custom else '_ai-git-flow/.github/prompts/planner-revise.md'
          t = open(path).read()
          t = t.replace('{{ISSUE_NUMBER}}', os.environ['ISSUE_NUMBER'])
          t = t.replace('{{PLAN_PATH}}', os.environ['PLAN_PATH'])
          t = t.replace('{{COMMENT_BODY}}', os.environ['COMMENT_BODY'])
          t = t.replace('{{COMMENT_AUTHOR}}', os.environ['COMMENT_AUTHOR'])
          open(os.environ['RUNNER_TEMP'] + '/prompt.md', 'w').write(t)
          "

      - name: Run planner agent
        uses: ./.github/actions/planner
        with:
          anthropic-api-key: ${{ secrets.anthropic-api-key || secrets.ANTHROPIC_API_KEY }}
          prompt-file: ${{ env.RUNNER_TEMP }}/prompt.md

      - name: Apply needs-human-intervention on failure
        if: failure()
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          PR_NUMBER: ${{ needs.guard.outputs.pr-number }}
          LABEL: ${{ inputs.needs-human-intervention-label || 'needs-human-intervention' }}
        run: gh pr edit "$PR_NUMBER" --add-label "$LABEL"
```

- [ ] **Step 2: Validate with actionlint**

```bash
actionlint .github/workflows/revise-plan.yml
```

Expected: no output (clean).

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/revise-plan.yml
git commit -m "feat: add workflow_call trigger to revise-plan.yml"
```

---

### Task 4: Update `implement.yml`

The existing job-level guard `if: github.event.label.name == 'plan-approved'` is moved into the guard step so the label name can be parameterized. All other logic is unchanged.

**Files:**
- Modify: `.github/workflows/implement.yml`

- [ ] **Step 1: Replace `implement.yml` with the updated content**

```yaml
name: AI Implement

on:
  pull_request:
    types: [labeled]
  workflow_call:
    secrets:
      anthropic-api-key:
        description: 'Anthropic API key'
        required: true
    inputs:
      plan-approved-label:
        description: 'Label that triggers implementation'
        required: false
        default: 'plan-approved'
        type: string
      needs-code-review-label:
        description: 'Label to apply after successful implementation'
        required: false
        default: 'needs-code-review'
        type: string
      needs-human-intervention-label:
        description: 'Label to apply on agent failure'
        required: false
        default: 'needs-human-intervention'
        type: string
      implementer-initial-prompt:
        description: 'Path to custom implementer prompt (relative to repo root). Omit to use built-in default.'
        required: false
        default: ''
        type: string

permissions:
  contents: write
  pull-requests: write

jobs:
  guard:
    runs-on: ubuntu-latest
    outputs:
      should-run: ${{ steps.check.outputs.should-run }}
      plan-path: ${{ steps.check.outputs.plan-path }}
    steps:
      - name: Check conditions
        id: check
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          PR_NUMBER: ${{ github.event.pull_request.number }}
          EVENT_LABEL: ${{ github.event.label.name }}
          PLAN_APPROVED_LABEL: ${{ inputs.plan-approved-label || 'plan-approved' }}
        run: |
          # Only run if the label applied is the plan-approved label
          if [ "$EVENT_LABEL" != "$PLAN_APPROVED_LABEL" ]; then
            echo "should-run=false" >> "$GITHUB_OUTPUT"
            exit 0
          fi
          # Only run if PR is still in draft state
          IS_DRAFT=$(gh pr view "$PR_NUMBER" --json isDraft --jq '.isDraft')
          if [ "$IS_DRAFT" != "true" ]; then
            echo "should-run=false" >> "$GITHUB_OUTPUT"
            exit 0
          fi
          # Derive plan path from branch name convention: plan/issue-{n}
          BRANCH=$(gh pr view "$PR_NUMBER" --json headRefName --jq '.headRefName')
          ISSUE_NUM="${BRANCH#plan/issue-}"
          if ! [[ "$ISSUE_NUM" =~ ^[0-9]+$ ]]; then
            echo "Branch '$BRANCH' does not follow plan/issue-{n} convention — skipping" >&2
            echo "should-run=false" >> "$GITHUB_OUTPUT"
            exit 0
          fi
          echo "should-run=true" >> "$GITHUB_OUTPUT"
          echo "plan-path=.claude/plans/issue-${ISSUE_NUM}.md" >> "$GITHUB_OUTPUT"

  implement:
    needs: guard
    if: needs.guard.outputs.should-run == 'true'
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          fetch-depth: 0

      - name: Fetch ai-git-flow resources
        uses: actions/checkout@v4
        with:
          repository: graytonio/ai-based-git-flow
          ref: main
          path: _ai-git-flow
          sparse-checkout: |
            .github/prompts
            .claude/plugins/superpowers

      - name: Checkout PR branch
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          PR_NUMBER: ${{ github.event.pull_request.number }}
        run: gh pr checkout "$PR_NUMBER"

      - name: Prepare prompt
        env:
          PLAN_PATH: ${{ needs.guard.outputs.plan-path }}
          PR_NUMBER: ${{ github.event.pull_request.number }}
          CUSTOM_PROMPT: ${{ inputs.implementer-initial-prompt }}
        run: |
          python3 -c "
          import os
          custom = os.environ.get('CUSTOM_PROMPT', '').strip()
          path = custom if custom else '_ai-git-flow/.github/prompts/implementer-initial.md'
          t = open(path).read()
          t = t.replace('{{PLAN_PATH}}', os.environ['PLAN_PATH'])
          t = t.replace('{{PR_NUMBER}}', os.environ['PR_NUMBER'])
          open(os.environ['RUNNER_TEMP'] + '/prompt.md', 'w').write(t)
          "

      - name: Run implementer agent
        uses: ./.github/actions/implementer
        with:
          anthropic-api-key: ${{ secrets.anthropic-api-key || secrets.ANTHROPIC_API_KEY }}
          prompt-file: ${{ env.RUNNER_TEMP }}/prompt.md

      - name: Swap labels on success
        if: success()
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          PR_NUMBER: ${{ github.event.pull_request.number }}
          REMOVE_LABEL: ${{ inputs.plan-approved-label || 'plan-approved' }}
          ADD_LABEL: ${{ inputs.needs-code-review-label || 'needs-code-review' }}
        run: |
          gh pr edit "$PR_NUMBER" \
            --remove-label "$REMOVE_LABEL" \
            --add-label "$ADD_LABEL"

      - name: Apply needs-human-intervention on failure
        if: failure()
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          PR_NUMBER: ${{ github.event.pull_request.number }}
          LABEL: ${{ inputs.needs-human-intervention-label || 'needs-human-intervention' }}
        run: gh pr edit "$PR_NUMBER" --add-label "$LABEL"
```

- [ ] **Step 2: Validate with actionlint**

```bash
actionlint .github/workflows/implement.yml
```

Expected: no output (clean).

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/implement.yml
git commit -m "feat: add workflow_call trigger to implement.yml"
```

---

### Task 5: Update `revise-impl.yml`

Same pattern as revise-plan.yml. Label check uses `--arg` in jq.

**Files:**
- Modify: `.github/workflows/revise-impl.yml`

- [ ] **Step 1: Replace `revise-impl.yml` with the updated content**

```yaml
name: AI Revise Implementation

on:
  pull_request_review_comment:
    types: [created]
  workflow_call:
    secrets:
      anthropic-api-key:
        description: 'Anthropic API key'
        required: true
    inputs:
      needs-code-review-label:
        description: 'Label that marks a PR as awaiting code review'
        required: false
        default: 'needs-code-review'
        type: string
      needs-human-intervention-label:
        description: 'Label to apply on agent failure'
        required: false
        default: 'needs-human-intervention'
        type: string
      implementer-revise-prompt:
        description: 'Path to custom implementer revision prompt (relative to repo root). Omit to use built-in default.'
        required: false
        default: ''
        type: string

permissions:
  contents: write
  pull-requests: write

concurrency:
  group: pr-${{ github.event.pull_request.number }}
  cancel-in-progress: true

jobs:
  guard:
    runs-on: ubuntu-latest
    outputs:
      should-run: ${{ steps.check.outputs.should-run }}
    steps:
      - name: Check conditions
        id: check
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          COMMENT_AUTHOR: ${{ github.event.comment.user.login }}
          PR_NUMBER: ${{ github.event.pull_request.number }}
          NEEDS_CODE_REVIEW_LABEL: ${{ inputs.needs-code-review-label || 'needs-code-review' }}
        run: |
          # Skip bot comments
          if [ "$COMMENT_AUTHOR" = "github-actions[bot]" ]; then
            echo "should-run=false" >> "$GITHUB_OUTPUT"
            exit 0
          fi
          # Check PR has the code-review label
          HAS_LABEL=$(gh pr view "$PR_NUMBER" --json labels \
            --jq --arg label "$NEEDS_CODE_REVIEW_LABEL" \
            '[.labels[].name] | contains([$label]) | if . then "1" else "0" end')
          if [ "$HAS_LABEL" != "1" ]; then
            echo "should-run=false" >> "$GITHUB_OUTPUT"
            exit 0
          fi
          # Check PR has at least one approved human review
          HAS_APPROVAL=$(gh pr view "$PR_NUMBER" --json reviews \
            --jq '[.reviews[] | select(.state == "APPROVED" and (.author.login // "") != "github-actions[bot]")] | length')
          if [ "$HAS_APPROVAL" = "0" ]; then
            echo "should-run=false" >> "$GITHUB_OUTPUT"
            exit 0
          fi
          BRANCH=$(gh pr view "$PR_NUMBER" --json headRefName --jq '.headRefName')
          ISSUE_NUM="${BRANCH#plan/issue-}"
          if ! [[ "$ISSUE_NUM" =~ ^[0-9]+$ ]]; then
            echo "Branch '$BRANCH' does not follow plan/issue-{n} convention — skipping" >&2
            echo "should-run=false" >> "$GITHUB_OUTPUT"
            exit 0
          fi
          echo "should-run=true" >> "$GITHUB_OUTPUT"

  revise:
    needs: guard
    if: needs.guard.outputs.should-run == 'true'
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          fetch-depth: 0

      - name: Fetch ai-git-flow resources
        uses: actions/checkout@v4
        with:
          repository: graytonio/ai-based-git-flow
          ref: main
          path: _ai-git-flow
          sparse-checkout: |
            .github/prompts
            .claude/plugins/superpowers

      - name: Checkout PR branch
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          PR_NUMBER: ${{ github.event.pull_request.number }}
        run: gh pr checkout "$PR_NUMBER"

      - name: Prepare prompt
        env:
          PR_NUMBER: ${{ github.event.pull_request.number }}
          COMMENT_BODY: ${{ github.event.comment.body }}
          COMMENT_AUTHOR: ${{ github.event.comment.user.login }}
          COMMENT_PATH: ${{ github.event.comment.path }}
          COMMENT_LINE: ${{ github.event.comment.line }}
          CUSTOM_PROMPT: ${{ inputs.implementer-revise-prompt }}
        run: |
          python3 -c "
          import os
          custom = os.environ.get('CUSTOM_PROMPT', '').strip()
          path = custom if custom else '_ai-git-flow/.github/prompts/implementer-revise.md'
          t = open(path).read()
          t = t.replace('{{PR_NUMBER}}', os.environ['PR_NUMBER'])
          t = t.replace('{{COMMENT_BODY}}', os.environ['COMMENT_BODY'])
          t = t.replace('{{COMMENT_AUTHOR}}', os.environ['COMMENT_AUTHOR'])
          t = t.replace('{{COMMENT_PATH}}', os.environ['COMMENT_PATH'])
          t = t.replace('{{COMMENT_LINE}}', os.environ.get('COMMENT_LINE') or 'unknown')
          open(os.environ['RUNNER_TEMP'] + '/prompt.md', 'w').write(t)
          "

      - name: Run implementer agent
        uses: ./.github/actions/implementer
        with:
          anthropic-api-key: ${{ secrets.anthropic-api-key || secrets.ANTHROPIC_API_KEY }}
          prompt-file: ${{ env.RUNNER_TEMP }}/prompt.md

      - name: Apply needs-human-intervention on failure
        if: failure()
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          PR_NUMBER: ${{ github.event.pull_request.number }}
          LABEL: ${{ inputs.needs-human-intervention-label || 'needs-human-intervention' }}
        run: gh pr edit "$PR_NUMBER" --add-label "$LABEL"
```

- [ ] **Step 2: Validate with actionlint**

```bash
actionlint .github/workflows/revise-impl.yml
```

Expected: no output (clean).

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/revise-impl.yml
git commit -m "feat: add workflow_call trigger to revise-impl.yml"
```

---

### Task 6: Push, verify, and cut v1.0.0 release

- [ ] **Step 1: Run actionlint across all changed files**

```bash
actionlint .github/workflows/plan.yml .github/workflows/revise-plan.yml .github/workflows/implement.yml .github/workflows/revise-impl.yml .github/actions/planner/action.yml .github/actions/implementer/action.yml
```

Expected: no output (clean).

- [ ] **Step 2: Push to main**

```bash
git push origin master:main
```

- [ ] **Step 3: Open a test issue to verify the native trigger still works**

In `graytonio/ai-based-git-flow`, create a GitHub issue with a simple task (e.g., "Add a hello world bash script to scripts/hello.sh"). Verify:
- `plan.yml` workflow triggers and completes
- A draft PR is opened with a plan file at `.claude/plans/issue-{N}.md`
- The `needs-plan-review` label is applied

- [ ] **Step 4: Cut v1.0.0 release**

Once the native trigger is confirmed working:

```bash
gh release create v1.0.0 --title "v1.0.0" --notes "Initial public release. Adds workflow_call triggers to all four workflows for reusable workflow distribution."
```

- [ ] **Step 5: Create floating v1 tag**

```bash
git tag v1
git push origin v1
```
