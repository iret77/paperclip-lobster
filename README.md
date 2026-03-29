# paperclip-plugin-lobster

A [Paperclip](https://github.com/paperclipai/paperclip) plugin that brings deterministic hard constraints and approval gates to AI agent teams — inspired by the [Lobster](https://github.com/openclaw/lobster) system from [OpenClaw](https://github.com/openclaw/openclaw).

## Problem

Paperclip orchestrates autonomous agent teams with budget policies and task-level approvals. But it lacks **workflow-step-level hard constraints** — the ability to halt an agent mid-execution, require explicit human approval, and resume deterministically.

OpenClaw solves this with Lobster, a typed workflow engine with approval gates and state persistence. However, OpenClaw's full stack is overkill for pure dev-team orchestration where Paperclip excels.

This plugin bridges the gap: Lobster-style hard constraints, native in Paperclip's plugin system.

## Features

- **Approval Gates** — halt agent execution completely until a human approves or denies (not a warning, a hard stop)
- **Constraint Rules** — declarative YAML-based constraints scoped to company, project, or agent
- **Deterministic Pipelines** — multi-step workflows controlled by code/state machines, not LLM routing
- **Checkpoint/Resume** — token-based state persistence so workflows resume exactly where they stopped
- **Built-in Constraint Types:**
  - `approval_gate` — require explicit human approval before proceeding
  - `file_scope` — restrict which paths an agent may modify
  - `budget_check` — per-action cost limits with approval thresholds
  - `time_window` — restrict agent activity to defined time windows

## Architecture

```
Paperclip Agent
  → calls tool "constraint_check" (action: "deploy")
  → Plugin evaluates matching constraints
  → Hard constraint matched → status: "needs_approval", resumeToken returned
  → Agent halts, parks task

[Human approves via UI or API]

  → Plugin job detects approval
  → Agent wakeup triggered
  → Agent resumes with token → status: "approved"
  → Agent proceeds
```

The plugin integrates via Paperclip's native extension points:

| Capability | Usage |
|------------|-------|
| `agent.tools.register` | `constraint_check`, `approval_gate`, `pipeline_run` tools |
| `jobs.schedule` | Periodic constraint violation scans |
| `ui.slots` | Dashboard for active constraints and pending approvals |

## Constraint Definition

```yaml
constraints:
  - id: deploy-approval
    scope: project
    trigger: before_deploy
    type: approval_gate
    config:
      approvers: ["ceo-agent", "human"]
      timeout: "24h"
      on_timeout: deny

  - id: file-scope
    scope: agent
    agent_role: engineer
    type: file_scope
    config:
      allowed_paths: ["src/**", "tests/**"]
      denied_paths: ["infrastructure/**", ".env*"]

  - id: cost-guard
    scope: company
    type: budget_check
    config:
      max_per_action_cents: 500
      require_approval_above_cents: 200
```

## Setup

```bash
git clone git@github.com:iret77/paperclip-lobster.git
cd paperclip-lobster
./script/setup   # configures git hooks
```

## Status

**Early development** — this plugin is being designed and built. Contributions and feedback welcome.

## License

MIT
