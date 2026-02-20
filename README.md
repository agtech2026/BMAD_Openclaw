# BMad Method × OpenClaw

> ## ⚠️ NOTICE: `dev` branch available
> A **`dev` branch** is under active development that more closely mirrors the official [BMad Method v6](https://github.com/bmadcode/BMAD-METHOD) structure — with proper `agents/`, `templates/`, `checklists/`, and `workflow/` directories aligned to upstream naming. It also includes 11 additional OpenClaw-specific execution agents under `agents-openclaw/`.
>
> **👉 Check out the [`dev` branch](../../tree/dev) for the latest.**
>
> This `master` branch is the original v1 release and is kept for backward compatibility.

---

A full implementation of the [BMad Method](https://github.com/bmadcode/BMAD-METHOD) using [OpenClaw](https://github.com/openclaw/openclaw)'s `sessions_spawn` capability. Ship production-quality software with a 12-agent AI development team — all orchestrated from a single chat session.

## What is this?

The BMad Method is a structured approach to AI-driven software development. It uses specialized agents (product owner, architect, developer, reviewer, etc.) to take a product idea from brief → PRD → architecture → stories → implementation → review.

This repo adapts BMad to run natively on OpenClaw, using sub-agent spawning instead of CLI tools. The result: **lower token cost, crash recovery, and a single orchestrator that stays responsive while agents work in parallel.**

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Main Session (Orchestrator)               │
│  • Always responsive to user                                 │
│  • Spawns sub-agents for each workflow step                  │
│  • Handles HALT conditions and retries                       │
│  • Tracks sprint status across all agents                    │
└─────────────────────────────────────────────────────────────┘
                              │
                    sessions_spawn (isolated)
                              │
    ┌──────────┬──────────┬──────────┬──────────┬──────────┐
    ▼          ▼          ▼          ▼          ▼          ▼
 product    business   architect  ux-design  scrum     readiness
  owner     analyst                er       master     check
    │          │          │          │          │          │
    ▼          ▼          ▼          ▼          ▼          ▼
  Brief      PRD     Architecture  UX Spec   Epics    GO/NO-GO
                                            Stories
    │
    └──────────────────────────────────────────────────────────┐
                                                               │
    ┌──────────┬──────────┬──────────┬──────────┐              │
    ▼          ▼          ▼          ▼          ▼              │
 create     dev-story   code      ux-review  qa-tester  retrospective
  story                 review
    │          │          │          │          │              │
    ▼          ▼          ▼          ▼          ▼              ▼
 Story.md   Code +    Approve/   UX Pass/   QA Pass/    Sprint
            Tests     Reject     Fail       Fail        Learnings
```

## 12 Agents

### Planning Phase
| Agent | Prompt | Output |
|-------|--------|--------|
| **Product Owner** | `prompts/product-owner.md` | Product Brief |
| **Business Analyst** | `prompts/business-analyst.md` | PRD (requirements, user journeys, FRs) |
| **Architect** | `prompts/architect.md` | Architecture doc (stack, data model, APIs) |
| **UX Designer** | `prompts/ux-designer.md` | UX Design Spec (screens, flows, components) |
| **Scrum Master** | `prompts/scrum-master.md` | Epics & Stories with acceptance criteria |
| **Readiness Check** | `prompts/readiness-check.md` | GO/NO-GO decision for implementation |

### Execution Phase
| Agent | Prompt | Output |
|-------|--------|--------|
| **Create Story** | `prompts/create-story.md` | Story file with tasks, AC, dependencies |
| **Dev Story** | `prompts/dev-story.md` | Implementation + tests (red-green-refactor) |
| **Code Review** | `prompts/code-review.md` | Adversarial review (3-10 issues minimum) |
| **UX Review** | `prompts/ux-review.md` | UX compliance check |
| **QA Tester** | `prompts/qa-tester.md` | Test execution and validation |
| **Retrospective** | `prompts/retrospective.md` | Sprint retrospective and learnings |

## Setup

### Prerequisites

- [OpenClaw](https://github.com/openclaw/openclaw) installed and running
- An LLM API key configured (Claude recommended)

### 1. Clone this repo into your OpenClaw workspace

```bash
cd ~/.openclaw/workspace
git clone git@github.com:agtech2026/BMAD_Openclaw.git
```

### 2. Create a project config

Copy the example config and customize for your project:

```bash
cp bmad-openclaw/config/example.yaml bmad-openclaw/config/my-project.yaml
```

Edit `my-project.yaml`:

```yaml
project:
  name: "My SaaS App"
  description: "Brief description of what you're building"
  
paths:
  root: "/home/you/.openclaw/workspace/my-project"
  planning: "/home/you/.openclaw/workspace/my-project/_bmad-output/planning-artifacts"
  implementation: "/home/you/.openclaw/workspace/my-project/_bmad-output/implementation-artifacts"

stack:
  frontend: "Next.js, Tailwind, ShadCN"
  backend: "Supabase"
  language: "TypeScript"
```

### 3. Add orchestrator instructions to your AGENTS.md

Add to your workspace `AGENTS.md` or `SOUL.md`:

```markdown
## BMad Orchestrator

When asked to run BMad workflows, use the prompts in `bmad-openclaw/prompts/`.
Spawn each agent as an isolated sub-agent with `sessions_spawn`.

Workflow order:
1. product-owner → Brief
2. business-analyst → PRD  
3. architect → Architecture
4. ux-designer → UX Spec
5. scrum-master → Epics & Stories
6. readiness-check → GO/NO-GO
7. For each story: create-story → dev-story → code-review → [ux-review] → [qa-tester]
8. retrospective → Sprint learnings
```

### 4. Run it

From your OpenClaw chat:

```
Run the BMad product-owner workflow for my-project.
The idea: [describe your product]
```

OpenClaw spawns the product-owner agent, which produces a brief. Then:

```
Run the BMad business-analyst workflow. Use the brief we just created.
```

Continue through the pipeline. Each agent reads previous artifacts and produces the next one.

## Usage Examples

### Start a new project
```
Run BMad product-owner for my-project. 
Idea: A SaaS that lets agencies generate branded presentations from templates.
```

### Generate architecture
```
Spawn BMad architect. Read the PRD at _bmad-output/planning-artifacts/prd.md 
and produce an architecture document.
```

### Implement a story
```
Spawn BMad dev-story for story 2-1-workspace-management.
Follow red-green-refactor. Run tests after each task.
```

### Review code
```
Spawn BMad code-review for story 2-1.
Be adversarial — find at least 3 issues.
```

## How It Works

### Sub-Agent Spawning

Each BMad agent runs as an isolated OpenClaw session via `sessions_spawn`:

```javascript
sessions_spawn({
  task: `You are executing the architect workflow for My Project.
  
  ## Context
  - PROJECT_ROOT: /path/to/project
  - Read the PRD at: _bmad-output/planning-artifacts/prd.md
  
  ## Instructions
  ${architectPrompt}
  
  Begin now. Follow all steps. Report completion or HALT.`,
  label: "bmad-architect"
})
```

The sub-agent works autonomously, reads/writes files, and announces results back to the main session.

### HALT Protocol

When an agent can't proceed, it returns:

```
HALT: Missing database schema | Context: Need schema before implementing data layer
```

The orchestrator (main session) can:
1. **Resolve and respawn** — fix the issue, re-run the agent
2. **Ask the user** — for ambiguous decisions
3. **Mark blocked** — log it, move to next story

### State Tracking

All state lives in files (not in memory):
- Planning artifacts in `_bmad-output/planning-artifacts/`
- Story files in `_bmad-output/implementation-artifacts/`
- Sprint status in `sprint-status.yaml`

This means agents can crash and be respawned without losing context.

## Comparison with Original BMad

| Feature | Original BMad | OpenClaw BMad |
|---------|---------------|---------------|
| Orchestrator | BMad Master (menu-driven) | Main session (conversational) |
| Agent execution | Claude Code CLI in PTY | `sessions_spawn` (isolated) |
| Token cost | 3x (main → CLI → main) | 1x (sub-agent only) |
| Crash recovery | Lost context | Orchestrator respawns |
| Interactive prompts | ask/wait in workflow | HALT → orchestrator decides |
| Parallel work | Sequential only | Multiple agents simultaneously |
| YOLO mode | Built-in flag | Not needed (agents are autonomous) |

## Quality Guarantees

- ✅ Red-green-refactor methodology
- ✅ Definition of Done checklists  
- ✅ Adversarial code review (3-10 issues minimum)
- ✅ Sprint status tracking
- ✅ Given/When/Then acceptance criteria
- ✅ Task completion verification

## License

MIT

## Credits

- [BMad Method](https://github.com/bmadcode/BMAD-METHOD) by BMad Code
- [OpenClaw](https://github.com/openclaw/openclaw) — the AI agent platform
