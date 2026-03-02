# loacher-solution/.github

The central hub for the loacher-solution GitHub organization. Contains the
org profile, shared GitHub Actions workflows (agentic workflow engine), and
default community health files.

## What's in here

| Path | Purpose |
|------|---------|
| `profile/README.md` | Org homepage at github.com/loacher-solution |
| `.github/workflows/orchestrator-trigger.yml` | Entry point for project board automation |
| `.github/workflows/orchestrator.yml` | Reusable orchestrator (routes issues to agents) |
| `.github/workflows/developer.yml` | Reusable developer agent workflow |
| `.github/workflows/reviewer.yml` | Reusable reviewer agent workflow |
| `.claude/agents/` | Org-wide agent definitions (Dave, Rick) |
| `workflow-templates/` | Suggested templates shown in Actions tab of org repos |
| `ISSUE_TEMPLATE/` | Default issue templates for all org repos |
| `PULL_REQUEST_TEMPLATE.md` | Default PR template for all org repos |
| `config.yml` | Org-wide agentic workflow defaults |

## Adding a new target repo to the agentic workflow

1. In the target repo, create `.github/workflows/agentic-developer.yml` and
   `agentic-reviewer.yml` — use the templates from `workflow-templates/` as
   your starting point (or select them from the Actions tab in GitHub).

2. Create `.claude/agents/developer.md` and `reviewer.md` in the target repo.
   Copy from `.claude/agents/` in this repo and customize the
   `Tools & Commands` section with your repo's actual test/lint/build commands.

3. Connect the target repo to the GitHub Projects V2 board.

4. Ensure the self-hosted runners (`agentic-developer`, `agentic-reviewer`)
   are registered and running.

## Infrastructure requirements

| Component | Status | Notes |
|-----------|--------|-------|
| GitHub org secrets (`CLAUDE_CODE_OAUTH_TOKEN`, `WORKFLOW_PAT`) | Required | Set at org level |
| Self-hosted runners (`agentic-developer`, `agentic-reviewer`) | Required | Linux, Claude Code CLI installed |
| Cloudflare Worker webhook relay | Required | Forwards project board events to this repo |
| GitHub Projects V2 board | Required | Status columns: Backlog, Ready, In Progress, In Review, Done |
