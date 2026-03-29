# paperclip-lobster — Concept

> Hard constraints and approval gates for Paperclip agent teams.

## Context

**Paperclip** orchestrates autonomous AI agent teams. It manages budgets, task assignments, heartbeat-driven execution, and governance. But its constraint model is coarse: budget hard-stops pause entire agents, and issue approvals gate entire tasks. There is no mechanism to enforce arbitrary rules on *what an agent does within a task* — file access boundaries, deployment gates, cost-per-action limits, or time-based restrictions.

**Lobster** (from the OpenClaw ecosystem) solves this for OpenClaw agents through deterministic pipelines with approval gates, conditional execution, and stateful checkpoint/resume. But OpenClaw's full orchestration stack is overkill for pure dev teams where Paperclip is the better fit.

**paperclip-lobster** brings Lobster's hard constraint model into Paperclip as a native plugin — using Paperclip's own plugin API, not as a fork or core modification.

## Problem Statement

Without step-level constraints, Paperclip agent teams operate on trust: the agent's instructions say "don't deploy without approval" but nothing *enforces* it. A misbehaving or poorly prompted agent can:

- Push to production without review
- Modify infrastructure files outside its scope
- Burn through budget on a single expensive action
- Operate outside intended hours
- Skip required approval workflows

The cost of these failures scales with team size and autonomy level.

---

## Goals

### G1 — Declarative Constraint Definitions

Operators can define hard constraints as structured data (YAML), scoped to company, project, or individual agent. Constraints are version-controlled alongside project configuration.

**Success criteria:**
- Constraints are defined in a single YAML schema, validated at load time
- Each constraint has: `id`, `scope` (company/project/agent), `type`, and type-specific `config`
- Invalid constraint definitions produce clear error messages with line references
- Constraints can be added, modified, or removed without restarting the plugin or agents
- A JSON Schema is published so editors provide autocompletion and validation

### G2 — Approval Gates with Hard Halt

Agents can be stopped mid-workflow at defined checkpoints. Execution does not continue until a human (or authorized agent) explicitly approves or denies. Denial terminates the workflow branch.

**Success criteria:**
- When an agent hits an approval gate, the plugin returns a `needs_approval` status with a resume token
- The agent's tool call returns immediately — the agent is not blocked in a polling loop
- Approval state persists across agent restarts and heartbeat cycles (via Paperclip's plugin state API)
- Approvals can be granted through: plugin UI slot, API call, or issue comment (e.g. `/approve`)
- Denied gates produce an audit trail entry with the denier's identity and reason
- Resume tokens are opaque, tamper-evident, and expire after a configurable TTL (default: 24h)
- An approval gate that times out without decision resolves to a configurable default (`deny` or `escalate`)

### G3 — Enforceable File Scope Boundaries

Agents can be restricted to specific file paths. Attempts to operate outside the allowed scope are blocked before execution, not detected after the fact.

**Success criteria:**
- File scope rules use glob patterns: `allowed_paths` and `denied_paths`
- `denied_paths` takes precedence over `allowed_paths` (deny-wins)
- The constraint is checked when the agent calls `constraint_check` before a file operation
- Violations return a clear error naming the violated rule, the blocked path, and the allowed scope
- A scheduled plugin job can scan execution workspaces for post-hoc violations and create incidents

### G4 — Per-Action Cost Guards

Individual agent actions can be cost-limited, independent of the agent's overall monthly budget.

**Success criteria:**
- Operators define `max_per_action_cents` and `require_approval_above_cents` thresholds
- Actions below the threshold proceed without friction
- Actions between `require_approval_above_cents` and `max_per_action_cents` trigger an approval gate (→ G2)
- Actions above `max_per_action_cents` are denied immediately
- Cost estimation is provided by the agent (self-reported) — the plugin validates against the threshold, not the actual cost

### G5 — Time Window Restrictions

Agent activity can be restricted to defined time windows.

**Success criteria:**
- Windows are defined as `{ days: [...], hours: "HH:MM-HH:MM", timezone: "..." }`
- Actions outside the window are handled according to a configurable policy: `deny`, `queue`, or `warn`
- `queue` means the action is deferred and auto-retried when the window opens (via plugin job)
- Timezone is explicit per rule — no implicit server-timezone assumptions

### G6 — Agent-Facing Tool Interface

Agents interact with the constraint system through standard Paperclip tools, not through custom protocol knowledge.

**Success criteria:**
- The plugin registers exactly three tools via `agent.tools.register`:
  - `constraint_check` — "May I do X?" → returns `allow`, `deny`, or `needs_approval`
  - `approval_gate` — "Halt here and wait for approval" → returns resume token
  - `constraint_resume` — "Continue with this token" → returns `approved` or `denied`
- Tool schemas are self-descriptive: an agent with no prior knowledge of lobster can use them from the tool description alone
- All tool responses follow a consistent envelope: `{ status, message, resumeToken?, constraint? }`

### G7 — Audit Trail

Every constraint evaluation, approval decision, and violation produces a durable, queryable record.

**Success criteria:**
- Events are stored via Paperclip's plugin state API with structured keys
- Each event contains: timestamp, agent ID, constraint ID, action, decision, context
- Events are queryable by agent, constraint, time range, and decision outcome
- The UI slot (→ G8) can display the audit log filtered by these dimensions
- Retention follows the company's existing data retention policy (no separate config)

### G8 — Operator Dashboard (UI Slot)

Operators can see active constraints, pending approvals, and recent violations at a glance.

**Success criteria:**
- A UI slot registered via `ui.slots` shows:
  - Pending approvals with one-click approve/deny
  - Active constraints per scope (company → project → agent drill-down)
  - Recent violations with agent, constraint, and timestamp
- The dashboard updates without full page reload (via plugin bridge SSE stream)
- Approval actions from the dashboard produce the same audit trail as API-based approvals

---

## Non-Goals

These are explicitly out of scope for the initial version:

- **LLM-based constraint evaluation** — constraints are deterministic rules, not AI judgments
- **Lobster workflow file compatibility** — we use the constraint model, not the pipeline DSL
- **Cross-Paperclip-instance constraints** — scoping is within a single Paperclip deployment
- **Automated constraint generation** — operators write constraints, agents don't self-constrain
- **Real-time cost verification** — cost is self-reported by agents, not measured by the plugin

## Dependencies

| Dependency | Type | Notes |
|------------|------|-------|
| Paperclip Plugin API v1 | Required | `agent.tools.register`, `jobs.schedule`, `ui.slots` capabilities |
| Paperclip Plugin State API | Required | For approval state and audit trail persistence |
| Paperclip Plugin Bridge | Required | For UI slot data and actions |
| Paperclip Agent Wakeup API | Required | To wake agents when approvals are granted |

## Constraint Schema (Reference)

```yaml
# Example: constraints.yaml
version: 1
constraints:

  - id: deploy-approval
    scope: project
    trigger: before_deploy
    type: approval_gate
    config:
      approvers: ["ceo-agent", "human"]
      timeout: "24h"
      on_timeout: deny

  - id: source-only
    scope: agent
    agent_role: engineer
    type: file_scope
    config:
      allowed_paths: ["src/**", "tests/**"]
      denied_paths: ["infrastructure/**", ".env*", "*.pem"]

  - id: cost-guard
    scope: company
    type: budget_check
    config:
      max_per_action_cents: 500
      require_approval_above_cents: 200

  - id: business-hours
    scope: company
    type: time_window
    config:
      windows:
        - days: [mon, tue, wed, thu, fri]
          hours: "06:00-22:00"
          timezone: Europe/Berlin
      on_violation: queue
```

## Success Metric (Overall)

The plugin is successful when an operator can:
1. Write a `constraints.yaml` file
2. Install the plugin into a Paperclip company
3. Have agents automatically check constraints before sensitive actions
4. Approve or deny gated actions through the UI or API
5. Audit every constraint decision after the fact

— all without modifying Paperclip core or the agents' adapter code.
