# AI Git Flow Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a GitHub Actions pipeline where AI agents handle the full issue-to-PR lifecycle — planner creates a plan file and draft PR, implementer turns the plan into code, both agents respond to human review in revision loops.

**Architecture:** Two composite actions (`planner`, `implementer`) are invoked by four thin workflow files. Labels are the only routing mechanism. The superpowers plugin is pinned in the repo at `.claude/plugins/superpowers/` and copied into place at runner setup time. Plan files live at a fixed convention path `.claude/plans/issue-{n}.md`.

**Tech Stack:** GitHub Actions, Claude Code CLI (`@anthropic-ai/claude-code`), `actionlint` (workflow linting), `gh` CLI (pre-installed on ubuntu-latest runners), Python 3 (pre-installed, used for multiline prompt injection)

---

## File Map

| File | Action | Responsibility |
|---|---|---|
| `.github/actions/planner/action.yml` | Create | Composite action: install Claude Code, copy plugin, run planner agent |
| `.github/actions/implementer/action.yml` | Create | Composite action: install Claude Code, copy plugin, run implementer agent |
| `.github/prompts/planner-initial.md` | Create | Prompt: read issue, write plan, open draft PR |
| `.github/prompts/planner-revise.md` | Create | Prompt: targeted plan revisions from review comment |
| `.github/prompts/implementer-initial.md` | Create | Prompt: read plan file, implement everything |
| `.github/prompts/implementer-revise.md` | Create | Prompt: targeted code fixes from review comment |
| `.github/workflows/plan.yml` | Create | Trigger: issues.opened → run planner, apply needs-plan-review |
| `.github/workflows/revise-plan.yml` | Create | Trigger: issue_comment on PR with needs-plan-review → run planner |
| `.github/workflows/implement.yml` | Create | Trigger: label plan-approved → run implementer, swap label |
| `.github/workflows/revise-impl.yml` | Create | Trigger: pr_review_comment on PR with needs-code-review → run implementer |
| `.claude/plugins/superpowers/` | Create | Pinned plugin files copied from local ~/.claude/plugins/superpowers |

---

### Task 1: Spike — Resolve open questions before writing any files

**Files:**
- Read: `~/.claude/plugins/` (local inspection only, no commits)

This task resolves three open questions from the spec before any code is written. Complete all three steps before moving on.

- [ ] **Step 1: Inspect the superpowers plugin directory layout**

Run:
```bash
ls -la ~/.claude/plugins/
ls -la ~/.claude/plugins/superpowers/ 2>/dev/null || echo "No superpowers dir found — check cache path"
ls -la ~/.claude/plugins/cache/claude-plugins-official/superpowers/ 2>/dev/null || true
```

Record the actual directory structure. You need to know: (a) the top-level folder name, (b) whether there's a `package.json` or similar manifest, (c) the approximate size of the directory. This determines exactly what gets committed in Task 3.

- [ ] **Step 2: Check plugin license**

Run:
```bash
find ~/.claude/plugins/ -name "LICENSE*" -o -name "license*" | head -5
cat ~/.claude/plugins/cache/claude-plugins-official/superpowers/*/LICENSE 2>/dev/null || echo "No license file found"
```

If a license file is present, read it. If it prohibits redistribution, pause and ask the user how to proceed before continuing. If no license file is found, proceed — the plugin is from an official Anthropic source and committing it to a private repo is acceptable by default.

- [ ] **Step 3: Install actionlint**

Run:
```bash
brew install actionlint
actionlint --version
```

Expected output: version string like `actionlint 1.x.x`. This is the linter used to validate every workflow file in subsequent tasks.

- [ ] **Step 4: Commit nothing — document decisions here**

The plan file path convention is now fixed: `.claude/plans/issue-{ISSUE_NUMBER}.md`
The plugin source path (from the inspection above) is what gets committed in Task 3.

No commit needed for this task.

---

### Task 2: Repo scaffolding

**Files:**
- Create: `.github/workflows/.gitkeep`
- Create: `.github/actions/planner/.gitkeep`
- Create: `.github/actions/implementer/.gitkeep`
- Create: `.github/prompts/.gitkeep`
- Create: `.claude/plans/.gitkeep`
- Create: `.gitignore`

- [ ] **Step 1: Create directory structure**

Run:
```bash
mkdir -p .github/workflows \
          .github/actions/planner \
          .github/actions/implementer \
          .github/prompts \
          .claude/plans
touch .github/workflows/.gitkeep \
      .github/actions/planner/.gitkeep \
      .github/actions/implementer/.gitkeep \
      .github/prompts/.gitkeep \
      .claude/plans/.gitkeep
```

- [ ] **Step 2: Write .gitignore**

Create `.gitignore`:
```
# OS
.DS_Store

# Temporary prompt files (generated at CI runtime)
/tmp/prompt.md
```

- [ ] **Step 3: Commit**

```bash
git add .github .claude .gitignore
git commit -m "chore: scaffold directory structure"
```

---

### Task 3: Commit superpowers plugin

**Files:**
- Create: `.claude/plugins/superpowers/` (contents from local)

Use the directory path discovered in Task 1.

- [ ] **Step 1: Copy plugin into repo**

Replace `<SOURCE_PATH>` with the actual path found in Task 1 (e.g., `~/.claude/plugins/cache/claude-plugins-official/superpowers/5.0.7`):

```bash
cp -r <SOURCE_PATH> .claude/plugins/superpowers
```

- [ ] **Step 2: Verify copy**

```bash
ls -la .claude/plugins/superpowers/
```

Expected: the same file structure seen in Task 1.

- [ ] **Step 3: Commit**

```bash
git add .claude/plugins/superpowers
git commit -m "chore: commit pinned superpowers plugin"
```

---

### Task 4: Planner composite action

**Files:**
- Create: `.github/actions/planner/action.yml`
- Delete: `.github/actions/planner/.gitkeep`

- [ ] **Step 1: Verify actionlint rejects an empty file**

```bash
echo "invalid: yaml: [" > /tmp/bad-action.yml
actionlint /tmp/bad-action.yml 2>&1 | head -5
```

Expected: error output. Confirms linter is working.

- [ ] **Step 2: Write the composite action**

Create `.github/actions/planner/action.yml`:
```yaml
name: 'AI Planner Agent'
description: 'Installs Claude Code, copies the superpowers plugin, and runs the planner agent'
inputs:
  anthropic-api-key:
    description: 'Anthropic API key'
    required: true
  prompt-file:
    description: 'Absolute path to the prepared prompt file'
    required: true
runs:
  using: 'composite'
  steps:
    - name: Set up Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '20'

    - name: Install Claude Code
      shell: bash
      run: npm install -g @anthropic-ai/claude-code

    - name: Configure git identity
      shell: bash
      run: |
        git config --global user.email "github-actions[bot]@users.noreply.github.com"
        git config --global user.name "github-actions[bot]"

    - name: Copy superpowers plugin
      shell: bash
      run: |
        mkdir -p ~/.claude/plugins
        cp -r "$GITHUB_WORKSPACE/.claude/plugins/superpowers" ~/.claude/plugins/superpowers

    - name: Run planner agent
      shell: bash
      env:
        ANTHROPIC_API_KEY: ${{ inputs.anthropic-api-key }}
      run: |
        claude --dangerously-skip-permissions \
               --max-turns 30 \
               -p "$(cat '${{ inputs.prompt-file }}')"
```

- [ ] **Step 3: Run actionlint**

```bash
rm .github/actions/planner/.gitkeep
actionlint .github/actions/planner/action.yml
```

Expected: no output (clean). If errors appear, fix them before continuing.

- [ ] **Step 4: Commit**

```bash
git add .github/actions/planner/
git commit -m "feat: add planner composite action"
```

---

### Task 5: Implementer composite action

**Files:**
- Create: `.github/actions/implementer/action.yml`
- Delete: `.github/actions/implementer/.gitkeep`

- [ ] **Step 1: Write the composite action**

Create `.github/actions/implementer/action.yml`:
```yaml
name: 'AI Implementer Agent'
description: 'Installs Claude Code, copies the superpowers plugin, and runs the implementer agent'
inputs:
  anthropic-api-key:
    description: 'Anthropic API key'
    required: true
  prompt-file:
    description: 'Absolute path to the prepared prompt file'
    required: true
runs:
  using: 'composite'
  steps:
    - name: Set up Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '20'

    - name: Install Claude Code
      shell: bash
      run: npm install -g @anthropic-ai/claude-code

    - name: Configure git identity
      shell: bash
      run: |
        git config --global user.email "github-actions[bot]@users.noreply.github.com"
        git config --global user.name "github-actions[bot]"

    - name: Copy superpowers plugin
      shell: bash
      run: |
        mkdir -p ~/.claude/plugins
        cp -r "$GITHUB_WORKSPACE/.claude/plugins/superpowers" ~/.claude/plugins/superpowers

    - name: Run implementer agent
      shell: bash
      env:
        ANTHROPIC_API_KEY: ${{ inputs.anthropic-api-key }}
      run: |
        claude --dangerously-skip-permissions \
               --max-turns 30 \
               -p "$(cat '${{ inputs.prompt-file }}')"
```

- [ ] **Step 2: Run actionlint**

```bash
rm .github/actions/implementer/.gitkeep
actionlint .github/actions/implementer/action.yml
```

Expected: no output.

- [ ] **Step 3: Commit**

```bash
git add .github/actions/implementer/
git commit -m "feat: add implementer composite action"
```

---

### Task 6: Agent prompts

**Files:**
- Create: `.github/prompts/planner-initial.md`
- Create: `.github/prompts/planner-revise.md`
- Create: `.github/prompts/implementer-initial.md`
- Create: `.github/prompts/implementer-revise.md`
- Delete: `.github/prompts/.gitkeep`

Prompts use `{{PLACEHOLDER}}` syntax. Values are injected by the calling workflow's prepare-prompt step using Python string replacement before the action runs.

- [ ] **Step 1: Write planner-initial.md**

Create `.github/prompts/planner-initial.md`:
```markdown
You are an AI planning agent. Your job is to read a GitHub issue and produce a structured implementation plan.

## What to do

1. Read the Runtime Context section at the bottom of this prompt.
2. Create a new git branch named `plan/issue-{{ISSUE_NUMBER}}` from `main`.
3. Write a structured implementation plan to `.claude/plans/issue-{{ISSUE_NUMBER}}.md` using the format below.
4. Commit the plan file: `git add .claude/plans/issue-{{ISSUE_NUMBER}}.md && git commit -m "plan: add implementation plan for issue #{{ISSUE_NUMBER}}"`
5. Push the branch: `git push -u origin plan/issue-{{ISSUE_NUMBER}}`
6. Open a draft pull request against `main`:
   - Title: `[Plan] {{ISSUE_TITLE}}`
   - Body: `Closes #{{ISSUE_NUMBER}}`

## Plan file format

The plan at `.claude/plans/issue-{{ISSUE_NUMBER}}.md` must contain these sections:

- **Goal**: one sentence describing what needs to be built
- **Background**: why this is being built (from the issue)
- **Approach**: high-level technical approach (2-4 sentences)
- **Assumptions**: any ambiguities in the issue and how you resolved them
- **Files**: list of files to create or modify, with one-line descriptions
- **Tasks**: numbered list of implementation steps. Each task must include: what to do, which files to touch, and acceptance criteria.

## Rules

- Do NOT write any implementation code
- Do NOT modify any existing source files
- The plan is the source of truth for the implementer — write it as if the implementer has never read the issue

## Runtime Context

Issue number: {{ISSUE_NUMBER}}
Issue title: {{ISSUE_TITLE}}
Issue body:
{{ISSUE_BODY}}
```

- [ ] **Step 2: Write planner-revise.md**

Create `.github/prompts/planner-revise.md`:
```markdown
You are an AI planning agent. Your job is to revise an existing implementation plan based on a reviewer comment.

## What to do

1. Read the current plan at `{{PLAN_PATH}}`.
2. Read the reviewer comment in the Runtime Context section below.
3. Make targeted edits to address the feedback — do not rewrite sections unrelated to the comment.
4. Commit: `git add {{PLAN_PATH}} && git commit -m "plan: revise based on review feedback"`
5. Push to the current branch.

## Rules

- Make targeted changes only — preserve all content not mentioned in the feedback
- If the comment requests implementation (writing code, modifying source files), do not act on it. Instead, add a note to the plan's Assumptions section: "Reviewer requested [X] — this is deferred to implementation."
- If the comment is ambiguous, make a conservative interpretation and document it in the plan's Assumptions section
- Do NOT open a new PR — the existing draft PR is the artifact
- Do NOT modify any source files

## Runtime Context

Issue number: {{ISSUE_NUMBER}}
Plan path: {{PLAN_PATH}}
Comment author: {{COMMENT_AUTHOR}}
Comment:
{{COMMENT_BODY}}
```

- [ ] **Step 3: Write implementer-initial.md**

Create `.github/prompts/implementer-initial.md`:
```markdown
You are an AI implementation agent. Your job is to implement everything described in an existing plan file.

## What to do

1. Read the plan at `{{PLAN_PATH}}`. This is your only source of truth — do not infer requirements from the PR description or issue body.
2. If `{{PLAN_PATH}}` does not exist: post a comment on PR #{{PR_NUMBER}} with the text "Implementation aborted: plan file not found at {{PLAN_PATH}}. Please re-run the planner." Then stop.
3. Implement each task in the plan in order.
4. Commit after each task: use a descriptive message like `feat: implement <task name from plan>`
5. Push all commits to the current branch when done.

## Rules

- The plan file at `{{PLAN_PATH}}` is the source of truth. Do not deviate from it.
- Do NOT modify the plan file at `{{PLAN_PATH}}`
- Do NOT open a new PR — push to the existing branch
- Commit frequently — at minimum once per task in the plan

## Runtime Context

Plan path: {{PLAN_PATH}}
PR number: {{PR_NUMBER}}
```

- [ ] **Step 4: Write implementer-revise.md**

Create `.github/prompts/implementer-revise.md`:
```markdown
You are an AI implementation agent. Your job is to address a specific code review comment on a pull request.

## What to do

1. Read the review comment in the Runtime Context section below.
2. Locate the file at `{{COMMENT_PATH}}`, line `{{COMMENT_LINE}}`.
3. Make the targeted fix described in the review.
4. Commit: `git add {{COMMENT_PATH}} && git commit -m "fix: address review comment on {{COMMENT_PATH}}:{{COMMENT_LINE}}"`
5. Push to the current branch.

## Rules

- Make targeted fixes only — do not refactor surrounding code unless the review explicitly asks for it
- If the comment is ambiguous, make a conservative interpretation and explain it in the commit message body
- Do NOT modify the plan file
- Do NOT open a new PR

## Runtime Context

PR number: {{PR_NUMBER}}
Reviewer: {{COMMENT_AUTHOR}}
File: {{COMMENT_PATH}}
Line: {{COMMENT_LINE}}
Comment:
{{COMMENT_BODY}}
```

- [ ] **Step 5: Commit all prompts**

```bash
rm .github/prompts/.gitkeep
git add .github/prompts/
git commit -m "feat: add agent prompt files"
```

---

### Task 7: plan.yml

**Files:**
- Create: `.github/workflows/plan.yml`
- Delete: `.github/workflows/.gitkeep` (only if no other workflow files exist yet)

- [ ] **Step 1: Write the workflow**

Create `.github/workflows/plan.yml`:
```yaml
name: AI Plan

on:
  issues:
    types: [opened]

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

      - name: Prepare prompt
        env:
          ISSUE_NUMBER: ${{ github.event.issue.number }}
          ISSUE_TITLE: ${{ github.event.issue.title }}
          ISSUE_BODY: ${{ github.event.issue.body }}
        run: |
          python3 -c "
          import os
          t = open('.github/prompts/planner-initial.md').read()
          t = t.replace('{{ISSUE_NUMBER}}', os.environ['ISSUE_NUMBER'])
          t = t.replace('{{ISSUE_TITLE}}', os.environ['ISSUE_TITLE'])
          t = t.replace('{{ISSUE_BODY}}', os.environ['ISSUE_BODY'])
          open('/tmp/prompt.md', 'w').write(t)
          "

      - name: Run planner agent
        uses: ./.github/actions/planner
        with:
          anthropic-api-key: ${{ secrets.ANTHROPIC_API_KEY }}
          prompt-file: /tmp/prompt.md

      - name: Apply needs-plan-review label
        if: success()
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          ISSUE_NUMBER: ${{ github.event.issue.number }}
        run: |
          BRANCH="plan/issue-${ISSUE_NUMBER}"
          PR_NUMBER=$(gh pr list --head "$BRANCH" --json number --jq '.[0].number')
          gh pr edit "$PR_NUMBER" --add-label "needs-plan-review"

      - name: Apply needs-human-intervention on failure
        if: failure()
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          ISSUE_NUMBER: ${{ github.event.issue.number }}
        run: |
          BRANCH="plan/issue-${ISSUE_NUMBER}"
          PR_NUMBER=$(gh pr list --head "$BRANCH" --json number --jq '.[0].number' 2>/dev/null || echo "")
          if [ -n "$PR_NUMBER" ]; then
            gh pr edit "$PR_NUMBER" --add-label "needs-human-intervention"
          fi
```

- [ ] **Step 2: Run actionlint**

```bash
actionlint .github/workflows/plan.yml
```

Expected: no output. If `actionlint` reports errors about the composite action inputs, that is expected and acceptable — it cannot resolve local composite actions at lint time.

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/plan.yml
git commit -m "feat: add plan.yml workflow"
```

---

### Task 8: revise-plan.yml

**Files:**
- Create: `.github/workflows/revise-plan.yml`

- [ ] **Step 1: Write the workflow**

Create `.github/workflows/revise-plan.yml`:
```yaml
name: AI Revise Plan

on:
  issue_comment:
    types: [created]

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
          # Check PR has needs-plan-review label
          HAS_LABEL=$(gh pr view "$ISSUE_NUMBER" --json labels --jq '[.labels[].name] | contains(["needs-plan-review"]) | if . then "1" else "0" end')
          if [ "$HAS_LABEL" != "1" ]; then
            echo "should-run=false" >> "$GITHUB_OUTPUT"
            exit 0
          fi
          echo "should-run=true" >> "$GITHUB_OUTPUT"
          echo "pr-number=$ISSUE_NUMBER" >> "$GITHUB_OUTPUT"
          echo "plan-path=.claude/plans/issue-${ISSUE_NUMBER}.md" >> "$GITHUB_OUTPUT"

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
        run: |
          python3 -c "
          import os
          t = open('.github/prompts/planner-revise.md').read()
          t = t.replace('{{ISSUE_NUMBER}}', os.environ['ISSUE_NUMBER'])
          t = t.replace('{{PLAN_PATH}}', os.environ['PLAN_PATH'])
          t = t.replace('{{COMMENT_BODY}}', os.environ['COMMENT_BODY'])
          t = t.replace('{{COMMENT_AUTHOR}}', os.environ['COMMENT_AUTHOR'])
          open('/tmp/prompt.md', 'w').write(t)
          "

      - name: Run planner agent
        uses: ./.github/actions/planner
        with:
          anthropic-api-key: ${{ secrets.ANTHROPIC_API_KEY }}
          prompt-file: /tmp/prompt.md

      - name: Apply needs-human-intervention on failure
        if: failure()
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          PR_NUMBER: ${{ needs.guard.outputs.pr-number }}
        run: gh pr edit "$PR_NUMBER" --add-label "needs-human-intervention"
```

- [ ] **Step 2: Run actionlint**

```bash
actionlint .github/workflows/revise-plan.yml
```

Expected: no output.

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/revise-plan.yml
git commit -m "feat: add revise-plan.yml workflow"
```

---

### Task 9: implement.yml

**Files:**
- Create: `.github/workflows/implement.yml`

- [ ] **Step 1: Write the workflow**

Create `.github/workflows/implement.yml`:
```yaml
name: AI Implement

on:
  pull_request:
    types: [labeled]

permissions:
  contents: write
  pull-requests: write

jobs:
  guard:
    runs-on: ubuntu-latest
    if: github.event.label.name == 'plan-approved'
    outputs:
      should-run: ${{ steps.check.outputs.should-run }}
      plan-path: ${{ steps.check.outputs.plan-path }}
    steps:
      - name: Check conditions
        id: check
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          PR_NUMBER: ${{ github.event.pull_request.number }}
        run: |
          # Only run if PR is still in draft state
          IS_DRAFT=$(gh pr view "$PR_NUMBER" --json isDraft --jq '.isDraft')
          if [ "$IS_DRAFT" != "true" ]; then
            echo "should-run=false" >> "$GITHUB_OUTPUT"
            exit 0
          fi
          echo "should-run=true" >> "$GITHUB_OUTPUT"
          # Derive plan path from branch name convention: plan/issue-{n}
          BRANCH=$(gh pr view "$PR_NUMBER" --json headRefName --jq '.headRefName')
          ISSUE_NUM="${BRANCH#plan/issue-}"
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

      - name: Checkout PR branch
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          PR_NUMBER: ${{ github.event.pull_request.number }}
        run: gh pr checkout "$PR_NUMBER"

      - name: Prepare prompt
        env:
          PLAN_PATH: ${{ needs.guard.outputs.plan-path }}
          PR_NUMBER: ${{ github.event.pull_request.number }}
        run: |
          python3 -c "
          import os
          t = open('.github/prompts/implementer-initial.md').read()
          t = t.replace('{{PLAN_PATH}}', os.environ['PLAN_PATH'])
          t = t.replace('{{PR_NUMBER}}', os.environ['PR_NUMBER'])
          open('/tmp/prompt.md', 'w').write(t)
          "

      - name: Run implementer agent
        uses: ./.github/actions/implementer
        with:
          anthropic-api-key: ${{ secrets.ANTHROPIC_API_KEY }}
          prompt-file: /tmp/prompt.md

      - name: Swap labels on success
        if: success()
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          PR_NUMBER: ${{ github.event.pull_request.number }}
        run: |
          gh pr edit "$PR_NUMBER" \
            --remove-label "plan-approved" \
            --add-label "needs-code-review"

      - name: Apply needs-human-intervention on failure
        if: failure()
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          PR_NUMBER: ${{ github.event.pull_request.number }}
        run: gh pr edit "$PR_NUMBER" --add-label "needs-human-intervention"
```

- [ ] **Step 2: Run actionlint**

```bash
actionlint .github/workflows/implement.yml
```

Expected: no output.

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/implement.yml
git commit -m "feat: add implement.yml workflow"
```

---

### Task 10: revise-impl.yml

**Files:**
- Create: `.github/workflows/revise-impl.yml`
- Delete: `.github/workflows/.gitkeep`

- [ ] **Step 1: Write the workflow**

Create `.github/workflows/revise-impl.yml`:
```yaml
name: AI Revise Implementation

on:
  pull_request_review_comment:
    types: [created]

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
      plan-path: ${{ steps.check.outputs.plan-path }}
    steps:
      - name: Check conditions
        id: check
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          COMMENT_AUTHOR: ${{ github.event.comment.user.login }}
          PR_NUMBER: ${{ github.event.pull_request.number }}
        run: |
          # Skip bot comments
          if [ "$COMMENT_AUTHOR" = "github-actions[bot]" ]; then
            echo "should-run=false" >> "$GITHUB_OUTPUT"
            exit 0
          fi
          # Check PR has needs-code-review label
          HAS_LABEL=$(gh pr view "$PR_NUMBER" --json labels --jq '[.labels[].name] | contains(["needs-code-review"]) | if . then "1" else "0" end')
          if [ "$HAS_LABEL" != "1" ]; then
            echo "should-run=false" >> "$GITHUB_OUTPUT"
            exit 0
          fi
          # Check PR has at least one approved human review
          HAS_APPROVAL=$(gh pr view "$PR_NUMBER" --json reviews --jq '[.reviews[] | select(.state == "APPROVED" and .author.login != "github-actions[bot]")] | length')
          if [ "$HAS_APPROVAL" = "0" ]; then
            echo "should-run=false" >> "$GITHUB_OUTPUT"
            exit 0
          fi
          echo "should-run=true" >> "$GITHUB_OUTPUT"
          BRANCH=$(gh pr view "$PR_NUMBER" --json headRefName --jq '.headRefName')
          ISSUE_NUM="${BRANCH#plan/issue-}"
          echo "plan-path=.claude/plans/issue-${ISSUE_NUM}.md" >> "$GITHUB_OUTPUT"

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
        run: |
          python3 -c "
          import os
          t = open('.github/prompts/implementer-revise.md').read()
          t = t.replace('{{PR_NUMBER}}', os.environ['PR_NUMBER'])
          t = t.replace('{{COMMENT_BODY}}', os.environ['COMMENT_BODY'])
          t = t.replace('{{COMMENT_AUTHOR}}', os.environ['COMMENT_AUTHOR'])
          t = t.replace('{{COMMENT_PATH}}', os.environ['COMMENT_PATH'])
          t = t.replace('{{COMMENT_LINE}}', os.environ.get('COMMENT_LINE') or 'unknown')
          open('/tmp/prompt.md', 'w').write(t)
          "

      - name: Run implementer agent
        uses: ./.github/actions/implementer
        with:
          anthropic-api-key: ${{ secrets.ANTHROPIC_API_KEY }}
          prompt-file: /tmp/prompt.md

      - name: Apply needs-human-intervention on failure
        if: failure()
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          PR_NUMBER: ${{ github.event.pull_request.number }}
        run: gh pr edit "$PR_NUMBER" --add-label "needs-human-intervention"
```

- [ ] **Step 2: Run actionlint**

```bash
rm .github/workflows/.gitkeep
actionlint .github/workflows/revise-impl.yml
```

Expected: no output.

- [ ] **Step 3: Lint all workflows together**

```bash
actionlint .github/workflows/*.yml
```

Expected: no output.

- [ ] **Step 4: Commit**

```bash
git add .github/workflows/
git commit -m "feat: add revise-impl.yml workflow"
```

---

### Task 11: GitHub repo setup and end-to-end smoke test

**Files:** None — this task is repo configuration and a live test.

Before the workflows can run, the GitHub repo needs to exist with the right configuration.

- [ ] **Step 1: Create the GitHub repository**

```bash
gh repo create ai-based-git-flow --private --source=. --remote=origin --push
```

If the repo already exists on GitHub:
```bash
git remote add origin https://github.com/<YOUR_USERNAME>/ai-based-git-flow.git
git push -u origin master
```

- [ ] **Step 2: Set the ANTHROPIC_API_KEY secret**

```bash
gh secret set ANTHROPIC_API_KEY
```

You will be prompted to enter the value. Paste your Anthropic API key.

- [ ] **Step 3: Create the required labels**

```bash
gh label create "needs-plan-review"       --color "0075ca" --description "Plan is awaiting human review"
gh label create "plan-approved"           --color "0e8a16" --description "Plan approved, ready for implementation"
gh label create "needs-code-review"       --color "e4e669" --description "Implementation awaiting human review"
gh label create "needs-human-intervention" --color "d93f0b" --description "Agent hit an error, human input required"
```

- [ ] **Step 4: File a smoke-test issue**

```bash
gh issue create \
  --title "Add a hello world script" \
  --body "Create a simple bash script at scripts/hello.sh that prints 'Hello, world!' when run."
```

- [ ] **Step 5: Observe plan.yml**

Go to the Actions tab on GitHub. You should see `AI Plan` running within a few seconds.

Wait for it to complete, then verify:
- A new branch `plan/issue-1` exists
- A draft PR titled `[Plan] Add a hello world script` exists
- The PR has the `needs-plan-review` label
- The file `.claude/plans/issue-1.md` is committed on the branch

If the workflow failed, check the logs for the specific step that failed and refer back to the relevant task above.

- [ ] **Step 6: Test the revision loop**

Comment on the draft PR: `Can you add an explanation of what the script does in the plan?`

Wait for `AI Revise Plan` to complete. Verify the plan file was updated on the branch with a new commit.

- [ ] **Step 7: Approve the plan and trigger implementation**

Apply the `plan-approved` label to the PR. Watch `AI Implement` run. Verify:
- `scripts/hello.sh` is committed on the branch
- The PR label swapped from `plan-approved` to `needs-code-review`

- [ ] **Step 8: Test the implementation revision loop**

Leave an inline review comment on `scripts/hello.sh` requesting a change (e.g., `change the message to 'Hello from AI!'`). Then submit a PR review with "Approve".

Wait for `AI Revise Implementation` to run. Verify the change was pushed to the branch.
