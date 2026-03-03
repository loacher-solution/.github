# loacher-solution .github Repo Setup

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Set up the `loacher-solution/.github` repository as the central hub for the GitHub organization — org profile, agentic workflow engine (consolidated), default issue/PR templates, and workflow templates for target repos.

**Architecture:** All reusable GitHub Actions workflows (orchestrator, developer, reviewer) live directly in this `.github` repo, eliminating the need for a separate `bc-agentic-workflow` repo. The trigger workflow calls the orchestrator within the same repo using `loacher-solution/.github/.github/workflows/orchestrator.yml@main`. Target repos reference workflows from `loacher-solution/.github` as well.

**Tech Stack:** GitHub Actions (reusable workflows), GitHub Projects V2, Cloudflare Worker (webhook relay, already deployed in `kai-loacher-org`), Claude Code CLI (`--agent` flag), `gh` CLI, `jq`, `python3`

---

## Repository Structure (target state)

```
.github/                              <- repo root on disk
├── .github/                          <- GitHub special folder
│   └── workflows/                    <- workflows that run IN this repo
│       ├── orchestrator-trigger.yml  <- entry point: receives project board events
│       ├── orchestrator.yml          <- reusable: routes issue to developer or reviewer
│       ├── developer.yml             <- reusable: runs Claude Code as developer agent
│       └── reviewer.yml              <- reusable: runs Claude Code as reviewer agent
├── .claude/                          <- org-wide agent defaults
│   ├── agents/
│   │   ├── developer.md              <- developer agent "Dave"
│   │   └── reviewer.md               <- reviewer agent "Rick"
│   ├── agent-memory/
│   │   ├── developer/.gitkeep
│   │   ├── reviewer/.gitkeep
│   │   └── logs/.gitkeep
│   ├── hooks/
│   │   ├── inject-default-agent.sh
│   │   └── inject-recent-logs.sh
│   └── settings.json
├── profile/
│   └── README.md                     <- shown on github.com/loacher-solution
├── workflow-templates/               <- suggested templates for new org repos
│   ├── agentic-developer.yml
│   ├── agentic-developer.properties.json
│   ├── agentic-reviewer.yml
│   └── agentic-reviewer.properties.json
├── ISSUE_TEMPLATE/                   <- default for all org repos
│   ├── feature.md
│   └── bug.md
├── PULL_REQUEST_TEMPLATE.md          <- default for all org repos
├── config.yml                        <- org-wide agentic workflow defaults
├── README.md                         <- internal repo documentation
└── tmp/                              <- staging area (delete after setup)
    ├── BusinessIdea.md
    └── IdeaOfThisRepo.md
```

---

## Task 1: Move Planning Files to tmp/

**Files:**
- Move: `BusinessIdea.md` → `tmp/BusinessIdea.md`
- Move: `IdeaOfThisRepo.md` → `tmp/IdeaOfThisRepo.md`

**Step 1: Create tmp/ folder and move files**

```bash
mkdir -p tmp
git mv BusinessIdea.md tmp/BusinessIdea.md
git mv IdeaOfThisRepo.md tmp/IdeaOfThisRepo.md
```

**Step 2: Commit**

```bash
git add tmp/
git commit -m "chore: move planning files to tmp/ for later cleanup"
```

---

## Task 2: Org Profile README

**Files:**
- Create: `profile/README.md`

**Context:** This file is displayed publicly at `https://github.com/loacher-solution`. It should be friendly and professional — introduce the person and the business direction. Do NOT include internal revenue targets, subcontractor plans, or phase details from `tmp/BusinessIdea.md`.

**Step 1: Create profile/ directory and README**

```markdown
# loacher.solution

Hi, I'm Kai — a Business Central consultant and developer based near Dinkelsbühl, Germany.

I help small and mid-sized companies in the DACH region get more out of Microsoft Dynamics 365 Business Central. My focus is on practical implementations, clean customizations, and getting things done efficiently — with the help of modern AI tooling.

## What I work on

- **Business Central implementations** — full projects from requirements to go-live
- **BC extensions & customizations** — AL development, integrations, reporting
- **AI-powered development workflows** — using Claude Code and GitHub Actions to automate the boring parts

## This organization

This GitHub org is where I build and share:
- Reusable GitHub Actions workflows for AI-assisted BC development
- Open tooling for BC consultants and developers

## Get in touch

- [LinkedIn](https://www.linkedin.com/in/kai-loacher) <!-- update with real URL -->
- [YouTube](https://www.youtube.com/@kai-loacher) <!-- update with real URL -->
```

**Step 2: Commit**

```bash
git add profile/README.md
git commit -m "feat: add org profile README"
```

---

## Task 3: Orchestrator Trigger Workflow

**Files:**
- Create: `.github/workflows/orchestrator-trigger.yml`

**Context:** This is the entry point. A Cloudflare Worker (already deployed for `kai-loacher-org`, needs re-deployment or re-pointing for `loacher-solution`) sends `repository_dispatch` events of type `projects_v2_item` to this repo. This trigger workflow catches them and calls the orchestrator.

The key change from `kai-loacher-org`: instead of calling `kai-loacher-org/bc-agentic-workflow/.github/workflows/orchestrator.yml@main`, this calls `loacher-solution/.github/.github/workflows/orchestrator.yml@main` (the orchestrator lives IN this repo).

**Step 1: Create .github/workflows/ directory and trigger file**

```yaml
name: Project Board Trigger

on:
  repository_dispatch:
    types: [projects_v2_item]

permissions:
  contents: write
  pull-requests: write
  issues: write
  actions: write

jobs:
  orchestrate:
    # Only trigger for issues when a single-select field (e.g. Status) changes
    if: >-
      github.event.client_payload.projects_v2_item.content_type == 'Issue' &&
      github.event.client_payload.changes.field_value.field_type == 'single_select'
    uses: loacher-solution/.github/.github/workflows/orchestrator.yml@main
    with:
      item_node_id: ${{ github.event.client_payload.projects_v2_item.node_id }}
      content_node_id: ${{ github.event.client_payload.projects_v2_item.content_node_id }}
      project_node_id: ${{ github.event.client_payload.projects_v2_item.project_node_id }}
    secrets: inherit
```

**Step 2: Commit**

```bash
git add .github/workflows/orchestrator-trigger.yml
git commit -m "feat: add orchestrator trigger workflow"
```

---

## Task 4: Orchestrator Reusable Workflow

**Files:**
- Create: `.github/workflows/orchestrator.yml`

**Context:** Copied 1:1 from `C:\Repos\bc-agentic-workflow\.github\workflows\orchestrator.yml`. No changes needed — it already uses generic `${{ inputs.* }}` references and dispatches to `agentic-developer.yml` and `agentic-reviewer.yml` in the TARGET repo (not in this repo). The only change: the `runner` default can remain `self-hosted`.

**Step 1: Copy orchestrator.yml**

Copy the full content of `C:\Repos\bc-agentic-workflow\.github\workflows\orchestrator.yml` verbatim into `.github/workflows/orchestrator.yml`. No substitutions needed.

**Step 2: Commit**

```bash
git add .github/workflows/orchestrator.yml
git commit -m "feat: add reusable orchestrator workflow"
```

---

## Task 5: Developer Reusable Workflow

**Files:**
- Create: `.github/workflows/developer.yml`

**Context:** Copied 1:1 from `C:\Repos\bc-agentic-workflow\.github\workflows\developer.yml`. No changes needed — it is fully parameterized via `workflow_call` inputs.

**Step 1: Copy developer.yml**

Copy the full content of `C:\Repos\bc-agentic-workflow\.github\workflows\developer.yml` verbatim into `.github/workflows/developer.yml`. No substitutions needed.

**Step 2: Commit**

```bash
git add .github/workflows/developer.yml
git commit -m "feat: add reusable developer agent workflow"
```

---

## Task 6: Reviewer Reusable Workflow

**Files:**
- Create: `.github/workflows/reviewer.yml`

**Context:** Copied 1:1 from `C:\Repos\bc-agentic-workflow\.github\workflows\reviewer.yml`. No changes needed.

**Step 1: Copy reviewer.yml**

Copy the full content of `C:\Repos\bc-agentic-workflow\.github\workflows\reviewer.yml` verbatim into `.github/workflows/reviewer.yml`. No substitutions needed.

**Step 2: Commit**

```bash
git add .github/workflows/reviewer.yml
git commit -m "feat: add reusable reviewer agent workflow"
```

---

## Task 7: Org-Wide Agent Defaults (.claude/)

**Files:**
- Create: `.claude/agents/developer.md`
- Create: `.claude/agents/reviewer.md`
- Create: `.claude/agent-memory/developer/.gitkeep`
- Create: `.claude/agent-memory/reviewer/.gitkeep`
- Create: `.claude/agent-memory/logs/.gitkeep`
- Create: `.claude/hooks/inject-default-agent.sh`
- Create: `.claude/hooks/inject-recent-logs.sh`
- Create: `.claude/settings.json`
- Create: `config.yml`

**Context:** These are the org-wide defaults. Each target repo can override them with its own `.claude/agents/developer.md` etc. Copy from `C:\Repos\bc-agentic-workflow\.claude\` — no changes needed since the agents are already generic.

**Step 1: Create .claude/agents/developer.md**

Copy verbatim from `C:\Repos\bc-agentic-workflow\.claude\agents\developer.md`.

**Step 2: Create .claude/agents/reviewer.md**

Copy verbatim from `C:\Repos\bc-agentic-workflow\.claude\agents\reviewer.md`.

**Step 3: Create .claude/agent-memory/ placeholder files**

```bash
mkdir -p .claude/agent-memory/developer .claude/agent-memory/reviewer .claude/agent-memory/logs
touch .claude/agent-memory/developer/.gitkeep
touch .claude/agent-memory/reviewer/.gitkeep
touch .claude/agent-memory/logs/.gitkeep
```

**Step 4: Create .claude/hooks/inject-default-agent.sh**

Copy verbatim from `C:\Repos\bc-agentic-workflow\.claude\hooks\inject-default-agent.sh`.

**Step 5: Create .claude/hooks/inject-recent-logs.sh**

Copy verbatim from `C:\Repos\bc-agentic-workflow\.claude\hooks\inject-recent-logs.sh`.

**Step 6: Create .claude/settings.json**

Copy verbatim from `C:\Repos\bc-agentic-workflow\.claude\settings.json`.

**Step 7: Create config.yml at repo root**

```yaml
# Organization-wide defaults for the agentic workflow.
# Each target repo can override these by placing a .claude/config.yml in its root.

max_review_cycles: 3
yolo_mode: false
dedicated_branch: ""
recent_logs_count: 3
default_agent: developer

# Runner labels (configured in the reusable workflows)
runners:
  orchestrator: "self-hosted"
  developer: "agentic-developer"
  reviewer: "agentic-reviewer"
```

**Step 8: Commit**

```bash
git add .claude/ config.yml
git commit -m "feat: add org-wide agent defaults and config"
```

---

## Task 8: Default Issue & PR Templates

**Files:**
- Create: `ISSUE_TEMPLATE/feature.md`
- Create: `ISSUE_TEMPLATE/bug.md`
- Create: `PULL_REQUEST_TEMPLATE.md`

**Context:** These files in the `loacher-solution/.github` repo serve as defaults for ALL repos in the org that don't have their own templates. GitHub looks for them in the root, `.github/`, or `docs/` directory of the `.github` repo.

**Step 1: Create ISSUE_TEMPLATE/feature.md**

```markdown
---
name: Feature request
about: Describe a new feature or improvement
title: ''
labels: ''
assignees: ''
---

## What should be implemented?

<!-- Describe the feature clearly. What is the expected behavior? -->

## Why is this needed?

<!-- What problem does it solve? -->

## Acceptance criteria

<!-- List the conditions that must be true for this issue to be considered done. -->
- [ ]
- [ ]

## Additional context

<!-- Screenshots, links, references — anything that helps. -->
```

**Step 2: Create ISSUE_TEMPLATE/bug.md**

```markdown
---
name: Bug report
about: Something is not working as expected
title: ''
labels: bug
assignees: ''
---

## What happened?

<!-- Describe the bug. What did you expect vs. what actually happened? -->

## Steps to reproduce

1.
2.
3.

## Expected behavior

<!-- What should have happened? -->

## Actual behavior

<!-- What happened instead? -->

## Environment

- BC version:
- Extension version:
- Browser / client:

## Additional context

<!-- Logs, screenshots, anything helpful. -->
```

**Step 3: Create PULL_REQUEST_TEMPLATE.md**

```markdown
## Summary

<!-- What does this PR do? Keep it short. -->

## Related issue

Ref #<!-- issue number -->

## Changes

<!-- List the key changes made. -->
-

## Test plan

<!-- How was this tested? What should reviewers check? -->
- [ ]

## Notes for reviewers

<!-- Anything specific reviewers should pay attention to? -->
```

**Step 4: Commit**

```bash
git add ISSUE_TEMPLATE/ PULL_REQUEST_TEMPLATE.md
git commit -m "feat: add default issue and PR templates"
```

---

## Task 9: Workflow Templates for Target Repos

**Files:**
- Create: `workflow-templates/agentic-developer.yml`
- Create: `workflow-templates/agentic-developer.properties.json`
- Create: `workflow-templates/agentic-reviewer.yml`
- Create: `workflow-templates/agentic-reviewer.properties.json`

**Context:** These appear in the Actions tab of any repo in the `loacher-solution` org as suggested templates. The `.yml` files are the template content; the `.properties.json` files describe them (name, description, icon). The templates call the reusable workflows from THIS repo (`loacher-solution/.github`).

**Step 1: Create workflow-templates/agentic-developer.yml**

```yaml
name: Agentic Developer

on:
  workflow_dispatch:
    inputs:
      issue_number:
        description: "Issue number to implement"
        type: string
        required: true
      review_cycle:
        description: "Current review cycle"
        type: string
        required: true
        default: "0"
      max_review_cycles:
        description: "Max review cycles"
        type: string
        required: false
        default: "3"
      project_node_id:
        description: "Project node ID"
        type: string
        required: false
        default: ""
      item_node_id:
        description: "Project item node ID"
        type: string
        required: false
        default: ""
      runner:
        description: "Runner label"
        type: string
        required: false
        default: "self-hosted"
      dedicated_branch:
        description: "Dedicated branch (empty = per-issue branches)"
        type: string
        required: false
        default: ""

permissions:
  contents: write
  pull-requests: write
  issues: write
  actions: write

jobs:
  develop:
    uses: loacher-solution/.github/.github/workflows/developer.yml@main
    with:
      issue_number: ${{ inputs.issue_number }}
      review_cycle: ${{ inputs.review_cycle }}
      max_review_cycles: ${{ inputs.max_review_cycles }}
      project_node_id: ${{ inputs.project_node_id }}
      item_node_id: ${{ inputs.item_node_id }}
      runner: ${{ inputs.runner }}
      dedicated_branch: ${{ inputs.dedicated_branch }}
    secrets: inherit
```

**Step 2: Create workflow-templates/agentic-developer.properties.json**

```json
{
    "name": "Agentic Developer",
    "description": "Dispatches Claude Code as an autonomous developer agent to implement a GitHub Issue.",
    "iconName": "octicon-cpu",
    "categories": [
        "Automation",
        "AI"
    ]
}
```

**Step 3: Create workflow-templates/agentic-reviewer.yml**

```yaml
name: Agentic Reviewer

on:
  workflow_dispatch:
    inputs:
      issue_number:
        description: "Issue number being reviewed"
        type: string
        required: true
      pr_number:
        description: "PR number to review"
        type: string
        required: true
      review_cycle:
        description: "Current review cycle"
        type: string
        required: true
        default: "0"
      max_review_cycles:
        description: "Max review cycles"
        type: string
        required: false
        default: "3"
      project_node_id:
        description: "Project node ID"
        type: string
        required: false
        default: ""
      item_node_id:
        description: "Project item node ID"
        type: string
        required: false
        default: ""
      runner:
        description: "Runner label"
        type: string
        required: false
        default: "self-hosted"

permissions:
  contents: write
  pull-requests: write
  issues: write
  actions: write

jobs:
  review:
    uses: loacher-solution/.github/.github/workflows/reviewer.yml@main
    with:
      issue_number: ${{ inputs.issue_number }}
      pr_number: ${{ inputs.pr_number }}
      review_cycle: ${{ inputs.review_cycle }}
      max_review_cycles: ${{ inputs.max_review_cycles }}
      project_node_id: ${{ inputs.project_node_id }}
      item_node_id: ${{ inputs.item_node_id }}
      runner: ${{ inputs.runner }}
    secrets: inherit
```

**Step 4: Create workflow-templates/agentic-reviewer.properties.json**

```json
{
    "name": "Agentic Reviewer",
    "description": "Dispatches Claude Code as an autonomous reviewer agent to review a Pull Request.",
    "iconName": "octicon-eye",
    "categories": [
        "Automation",
        "AI"
    ]
}
```

**Step 5: Commit**

```bash
git add workflow-templates/
git commit -m "feat: add agentic developer and reviewer workflow templates"
```

---

## Task 10: Update README.md

**Files:**
- Modify: `README.md`

**Context:** The current README.md is just `# .github`. Replace it with useful internal documentation that describes what this repo does and how to use it for onboarding new target repos.

**Step 1: Rewrite README.md**

```markdown
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

See `C:\Repos\bc-agentic-workflow\README.md` (locally) or the bc-agentic-workflow
repo in `kai-loacher-org` for detailed setup instructions.
```

**Step 2: Commit**

```bash
git add README.md
git commit -m "docs: update README with repo overview and onboarding guide"
```

---

## Task 11: Cleanup — Delete tmp/ Files

**Files:**
- Delete: `tmp/BusinessIdea.md`
- Delete: `tmp/IdeaOfThisRepo.md`
- Delete: `tmp/` (directory)

**Pre-condition:** Only do this once all information from those files has been properly absorbed into the new structure. At this point:
- The internal business strategy from `BusinessIdea.md` is private and does NOT belong in this repo at all — it should be stored elsewhere (e.g., a private Notion/OneNote, or a private separate repo).
- `IdeaOfThisRepo.md` content is now reflected in `README.md`.

**Step 1: Remove tmp/ files**

```bash
git rm tmp/BusinessIdea.md tmp/IdeaOfThisRepo.md
rmdir tmp/ 2>/dev/null || true
```

**Step 2: Commit**

```bash
git commit -m "chore: remove tmp planning files — content migrated to structured locations"
```

---

## Post-Setup Checklist

After all tasks are committed and pushed, verify:

- [ ] `https://github.com/loacher-solution` shows the profile README
- [ ] Creating a new repo in the org shows the issue templates by default
- [ ] The Actions tab of a new org repo shows "Agentic Developer" and "Agentic Reviewer" as suggested templates
- [ ] A `repository_dispatch` event (type `projects_v2_item`) to this repo triggers the orchestrator-trigger workflow (test with `gh api repos/loacher-solution/.github/dispatches ...`)

## Notes

**Cloudflare Worker:** The existing worker from `kai-loacher-org` must be reconfigured (or a new one deployed) to forward events to `loacher-solution/.github` instead. The PAT in the worker (`GITHUB_PAT`) must have `actions:write` on the `loacher-solution/.github` repo. The GitHub org webhook must point to the worker URL.

**Self-hosted runners:** Runners need to be registered under the `loacher-solution` org (not `kai-loacher-org`). Labels: `agentic-developer` and `agentic-reviewer`.

**Org secrets:** `CLAUDE_CODE_OAUTH_TOKEN` and `WORKFLOW_PAT` must be set as org-level secrets under `loacher-solution`.
