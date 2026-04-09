# AI-Based Git Flow

An AI-powered GitHub Actions pipeline that automates the full issue-to-PR lifecycle using Claude agents. When you open an issue, a planner agent creates a structured implementation plan. After you approve the plan, an implementer agent writes the code. Both agents respond to feedback in revision loops.

## How It Works

The pipeline has four phases, each driven by a GitHub Actions workflow. **Labels are the only routing mechanism** between phases.

```
Issue Opened
    │
    ▼ plan.yml
Planner creates plan → opens draft PR
    │ [needs-plan-review label applied]
    │
    ├── Human reviews and leaves comments
    │       │
    │       ▼ revise-plan.yml (loops until approved)
    │   Planner revises plan based on feedback
    │
    ├── Human applies [plan-approved] label
    │
    ▼ implement.yml
Implementer reads plan → writes code → pushes commits
    │ [plan-approved removed, needs-code-review applied]
    │
    ├── Human reviews code and leaves inline comments
    │       │
    │       ▼ revise-impl.yml (loops until merged)
    │   Implementer fixes code based on review
    │
    └── Human approves and merges
```

### Workflows

| Workflow | Trigger | What It Does |
|---|---|---|
| `plan.yml` | Issue opened | Runs planner agent; creates `plan/issue-{N}` branch, commits plan to `.claude/plans/issue-{N}.md`, opens draft PR |
| `revise-plan.yml` | Comment on PR with `needs-plan-review` | Runs planner agent again to revise plan based on human feedback |
| `implement.yml` | `plan-approved` label applied to PR | Runs implementer agent to implement all tasks from the plan |
| `revise-impl.yml` | Inline code review comment on PR with `needs-code-review` | Runs implementer agent to fix specific code per review feedback |

### Labels

| Label | Meaning |
|---|---|
| `needs-plan-review` | Plan is ready for human feedback |
| `plan-approved` | Plan approved; triggers implementation |
| `needs-code-review` | Implementation ready for human review |
| `needs-human-intervention` | Agent hit an error; manual action required |

### Agents

Both agents run as Claude Code CLI instances with `--max-turns 30` (prevents infinite loops).

**Planner** — reads issue context, produces a structured plan with: Goal, Background, Approach, Assumptions, Files, and Tasks. Never writes source code.

**Implementer** — reads the committed plan file as the sole source of truth, implements tasks in order, commits per task with descriptive messages. Never modifies the plan.

### Prompts

Prompts live in `.github/prompts/`. Each workflow prepares a prompt at runtime by injecting context (issue number, PR number, comment body, file path, etc.) using Python string replacement into `{{PLACEHOLDER}}` slots, then writes the result to `$RUNNER_TEMP/prompt.md` before passing it to the agent.

```
.github/prompts/
├── planner-initial.md      # Issue → Plan
├── planner-revise.md       # Review comment → Updated plan
├── implementer-initial.md  # Plan → Code
└── implementer-revise.md   # Review comment → Code fix
```

### Composite Actions

`.github/actions/planner/` and `.github/actions/implementer/` are identical in structure:

1. Install Node.js v20
2. Install Claude Code CLI globally (`@anthropic-ai/claude-code`)
3. Configure git identity as `github-actions[bot]`
4. Copy the pinned superpowers plugin to `~/.claude/plugins/superpowers`
5. Run: `claude --dangerously-skip-permissions --max-turns 30 -p "$(cat $PROMPT_FILE)"`

### Superpowers Plugin

A pinned copy of the [superpowers](https://github.com/anthropics/claude-code-superpowers) plugin lives at `.claude/plugins/superpowers/` (MIT license). It's copied to the runner at CI time so agents have access to planning, implementation, and code-review skills without network calls during execution.

## Repository Layout

```
.github/
├── actions/
│   ├── planner/action.yml          # Composite action: setup + run planner
│   └── implementer/action.yml      # Composite action: setup + run implementer
├── prompts/
│   ├── planner-initial.md
│   ├── planner-revise.md
│   ├── implementer-initial.md
│   └── implementer-revise.md
└── workflows/
    ├── plan.yml
    ├── revise-plan.yml
    ├── implement.yml
    └── revise-impl.yml
.claude/
├── plugins/superpowers/            # Pinned plugin (committed to repo)
├── plans/issue-{N}.md              # Auto-generated plan files
└── settings.local.json             # Claude Code permission whitelist
```

## Setup

**1. Add a repository secret:**
- `ANTHROPIC_API_KEY` — your Anthropic API key

**2. Create four labels in the repository:**
- `needs-plan-review`
- `plan-approved`
- `needs-code-review`
- `needs-human-intervention`

**3. Ensure GitHub Actions has write permissions** (Settings → Actions → General → Workflow permissions → Read and write).

That's it. Open an issue and the pipeline starts automatically.

## Safety & Design Notes

- **Self-triggering prevention:** Both revision workflows skip comments made by `github-actions[bot]` to avoid loops.
- **Concurrency guards:** Revision workflows use `group: pr-{N}` with `cancel-in-progress: true`; concurrent revision runs are cancelled.
- **Draft state guard:** `implement.yml` only runs if the PR is still in draft, preventing accidental re-runs.
- **Turn budget:** `--max-turns 30` caps agent execution; if hit, the workflow fails and applies `needs-human-intervention`.
- **Minimal secrets:** Only `ANTHROPIC_API_KEY` is required. No deploy keys, no long-lived tokens beyond the default `GITHUB_TOKEN`.
- **Human-in-the-loop:** Agents never merge. Every phase gate requires a human action (comment, label, review approval, or merge).
