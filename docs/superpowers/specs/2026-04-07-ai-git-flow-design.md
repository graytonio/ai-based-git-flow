# AI-Based Git Flow — Design Spec

**Date:** 2026-04-07  
**Status:** Approved  

---

## Overview

A GitHub Actions-based autonomous development pipeline where AI agents handle the full issue-to-PR lifecycle. Two agents (planner, implementer), four workflows, one PR as the shared artifact. Labels are the only routing mechanism between phases.

Phase 1 of this project is a working implementation in a single repo. Phase 2 (not in scope here) is extracting the system into a reusable, deployable form for arbitrary repos.

---

## File Structure

```
.github/
  workflows/
    plan.yml               # fires on issues.opened
    revise-plan.yml        # fires on issue_comment + needs-plan-review label
    implement.yml          # fires on label applied: plan-approved
    revise-impl.yml        # fires on pr_review_comment + needs-code-review label

  actions/
    planner/
      action.yml           # composite action: sets up claude, runs planner agent
    implementer/
      action.yml           # composite action: sets up claude, runs implementer agent

  prompts/
    planner-initial.md     # prompt for plan.yml
    planner-revise.md      # prompt for revise-plan.yml
    implementer-initial.md # prompt for implement.yml
    implementer-revise.md  # prompt for revise-impl.yml

.claude/
  plugins/
    superpowers/           # plugin files committed here (pinned version)
```

The plan file is committed to the repo by the planner agent (the superpowers plugin decides the exact path). The implementer reads it from the repo at runtime — the PR description is not the source of truth.

---

## State Machine

Labels are the only routing mechanism. Each workflow guards on the correct label before doing anything.

```
issues.opened
    └─► plan.yml ──────────────────────► applies: needs-plan-review
                                          opens draft PR

issue_comment (PR has needs-plan-review)
    └─► revise-plan.yml ───────────────► re-commits revised plan
                                          keeps: needs-plan-review
                                          (loops until human applies plan-approved)

label applied: plan-approved
    └─► implement.yml ─────────────────► pushes code
                                          swaps label: plan-approved → needs-code-review

pr_review_comment (PR has needs-code-review)
    └─► revise-impl.yml ───────────────► pushes fixes
                                          keeps: needs-code-review
                                          (loops until human approves + merges)
```

### Guard Conditions

| Workflow | Guard |
|---|---|
| `revise-plan.yml` | Comment is on a PR (not a plain issue — check `github.event.issue.pull_request` is set) AND PR has `needs-plan-review` AND comment author is not `github-actions[bot]` |
| `implement.yml` | Label applied is `plan-approved` AND PR is in draft state |
| `revise-impl.yml` | PR has `needs-code-review` AND reviewer is not `github-actions[bot]` AND PR has at least one approved GitHub PR review (not just a comment) |

The "plain issue" guard on `revise-plan.yml` is necessary because `issue_comment` events fire on both plain issues and PRs on GitHub. Without it, a comment on the original issue (not the draft PR) could trigger the revision workflow.

The PR review approval guard on `revise-impl.yml` prevents a stray inline comment from triggering the implementer before a human has explicitly approved the PR.

---

## Composite Actions

Both composite actions share the same setup sequence. The `ANTHROPIC_API_KEY` is passed as a secret input.

### Setup Steps (both actions)

1. Install Node.js
2. `npm install -g @anthropic-ai/claude-code`
3. Export `ANTHROPIC_API_KEY` from secret input
4. Copy superpowers plugin into place:
   ```bash
   cp -r .claude/plugins/superpowers ~/.claude/plugins/superpowers
   ```
5. Run the agent (see invocation below)

### Agent Invocation

```yaml
- run: |
    claude --dangerously-skip-permissions \
           --max-turns 30 \
           -p "$(cat ${{ inputs.prompt-file }})"
```

### Planner Action Inputs

| Input | Description |
|---|---|
| `issue-body` | The issue text |
| `issue-number` | Used to find/open the PR |
| `comment-thread` | Optional — passed by `revise-plan.yml` |
| `prompt-file` | Path to the prepared prompt file |

### Implementer Action Inputs

| Input | Description |
|---|---|
| `plan-path` | Path to the committed plan file in the repo |
| `pr-number` | The PR to push to |
| `review-thread` | Optional — passed by `revise-impl.yml` |
| `prompt-file` | Path to the prepared prompt file |

### Context Injection

Each workflow prepares a temp prompt file before calling the action, injecting runtime values via `sed`:

```yaml
- run: |
    sed "s|{{ISSUE_BODY}}|${{ github.event.issue.body }}|g" \
        .github/prompts/planner-initial.md > /tmp/prompt.md
```

The action receives `prompt-file: /tmp/prompt.md`. This keeps prompt files readable and avoids shell escaping issues.

### Plugin Availability

The superpowers plugin is committed to `.claude/plugins/superpowers/` in the repo (pinned version). The setup step copies it to `~/.claude/plugins/superpowers` before the agent runs, guaranteeing availability without any marketplace network call.

**Pre-implementation checklist:**
- Inspect `~/.claude/plugins/` locally to confirm the exact directory layout before committing
- Verify the superpowers plugin license permits redistribution in a private repo

---

## Agent Prompts

### `planner-initial.md`

Purpose: read the issue, produce a structured plan, commit it, open a draft PR.

Key instructions:
- Output the plan as a markdown file (let the plugin decide the path)
- Use `git commit` and `git push` to persist it
- Open a draft PR against `main` titled `[Plan] <issue title>`
- Do not write any implementation code

### `planner-revise.md`

Purpose: given the current plan and a comment thread, make targeted revisions.

Key instructions:
- Read the existing plan from the repo before doing anything
- Make targeted edits — do not rewrite from scratch
- Re-commit and push
- Do not respond to comments requesting implementation — flag them as out of scope in the plan

### `implementer-initial.md`

Purpose: read the committed plan, implement everything it describes.

Key instructions:
- Read the plan file at `{{PLAN_PATH}}` before writing any code
- The plan is the source of truth — do not infer requirements from the PR description
- Push all changes to the existing PR branch
- Do not modify the plan file
- If no plan file exists at `{{PLAN_PATH}}`, stop immediately and comment on the PR

### `implementer-revise.md`

Purpose: given the current diff and a review thread, address each comment.

Key instructions:
- Read the review thread carefully before making any changes
- Make targeted fixes — do not refactor surrounding code unless explicitly asked
- Push fixes to the PR branch
- If a review comment is ambiguous, make a conservative interpretation and note it in the commit message

---

## GITHUB_TOKEN Permissions

Declared explicitly in each workflow's `permissions:` block:

```yaml
permissions:
  contents: write
  pull-requests: write
```

No org-level permissions. No stored long-lived secrets beyond `ANTHROPIC_API_KEY`.

---

## Error Handling & Edge Cases

### Bot Comment Guard (self-triggering prevention)

Both revision workflows skip runs where the triggering comment came from the bot. This is the very first step in each workflow:

```yaml
- if: github.event.comment.user.login == 'github-actions[bot]'
  run: echo "Bot comment — skipping" && exit 0
```

### `--max-turns` Budget Exhaustion

If the agent hits the turn cap without completing, the workflow step exits non-zero. The workflow catches this and applies a `needs-human-intervention` label rather than silently failing, keeping the state machine unstuck.

### Push Conflicts on Revision Loops

Concurrent revision runs on the same branch cause push conflicts. Mitigation: add `concurrency` groups to revision workflows so only one run executes at a time and the second cancels:

```yaml
concurrency:
  group: pr-${{ github.event.issue.number }}   # issue_comment uses issue.number; pr_review_comment uses pull_request.number
  cancel-in-progress: true
```

### Plan File Not Found

If `implement.yml` fires but no plan file exists (e.g., the planner failed mid-run), the implementer exits early with a clear error. The prompt explicitly instructs: "If no plan file exists at `{{PLAN_PATH}}`, stop immediately and comment on the PR."

---

## Runner Infrastructure

Phase 1 targets GitHub-hosted runners. The ARC (Actions Runner Controller) hardening described in the original brief — ephemeral self-hosted runners, network egress allowlist, OIDC for cloud auth — is the intended phase 2 security upgrade and is documented here as the target state but not required for the initial implementation.

---

## Open Questions (to resolve before/during implementation)

1. **Superpowers plugin directory layout** — inspect `~/.claude/plugins/` locally before committing the plugin to the repo
2. **Plugin license** — verify redistribution is permitted in a private repo
3. **Plan file path discovery** — the superpowers plugin decides where to commit the plan; the implementer needs a reliable way to find it. Options: (a) the planner writes the path to a known file (e.g., `.claude/plan-path`), (b) the implementer searches for the most recently modified file in `docs/` or `.claude/specs/`, (c) a fixed convention is established in the planner prompt
4. **Phase 2 packaging** — form TBD; deferred until phase 1 is working
