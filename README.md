# Harness Engineering for Agentic Coding

A spec-driven development harness built on **Model-Driven Development (MDD)** principles. Spec files serve as the single source of truth across sessions, enabling lightweight session lifecycles while preserving full context.

## Architecture

```
Human (orchestrator) ↔ Main Session (opus)
                            │
                            ├─ plan-writer  → plan.md (agent-facing)
                            ├─ implementer  → code changes
                            ├─ reviewer     → PASS/FAIL verdict
                            │
                            └─ spec.md (human-facing, persists across sessions)
```

**Key principle:** Sessions are disposable. Specs are permanent.

## Workflow

```
Session 1: Requirements → Plan (Waterfall) → Implement → Review
           └─ spec.md updated ─┘
                ↓
         [Human: manual testing + code review]
                ↓
Session 2: Read spec.md → Fix/improve → Review
           └─ spec.md updated ─┘
                ↓
         [Human: re-test + review approval]
                ↓
Session N: ...repeat until done...
```

## Agents

| Agent | Model | Role |
|-------|-------|------|
| **plan-writer** | opus | Explores codebase via Explore agent, produces detailed implementation plan (Waterfall) |
| **implementer** | haiku/sonnet | Translates plan into code. No design decisions — follows plan exactly |
| **reviewer** | sonnet | Validates implementation against plan.md + Trust-5 quality framework (TRUS) |

### Trust-5 Quality Framework (4 of 5 verified by reviewer)

- **T**ested — Tests pass, no regressions
- **R**eadable — Naming reveals intent, matches plan
- **U**nified — Consistent with existing codebase patterns
- **S**ecured — Proper authorization, data access boundaries
- ~~**T**rackable~~ — Meaningful commits (human responsibility)

## File Structure

```
.claude/
├── agents/
│   ├── plan-writer.md      # Waterfall planning agent
│   ├── implementer.md      # DDD-based implementation agent
│   ├── reviewer.md         # Trust-5 review agent
│   └── test-writer.md      # Standalone test writing agent
├── docs/
│   ├── architecture.md     # Project-level layer rules (stable)
│   ├── code-style.md       # Project-level code style (stable)
│   └── domains/            # Domain docs (accumulated over time)
│       └── chatnotification.md
├── specs/
│   ├── TEMPLATE.md         # Spec file template
│   └── RGD-XXXX/           # Per-ticket working directory
│       ├── spec.md          # Human-facing: status, decisions, requirements
│       └── plan.md          # Agent-facing: detailed implementation tasks
└── ...
```

## Spec Files

### spec.md (human-facing)

Captures **why** decisions were made. Read by humans between sessions.

- Current status
- Requirements
- Key decisions with rationale and date
- What is explicitly out of scope
- Remaining work

### plan.md (agent-facing)

Captures **how** to implement. Read by orchestrator, implementer, and reviewer.

- Per-task method signatures, implementation steps, reference code
- Success/failure criteria per task
- Task execution order (parallel/sequential)

## Domain Documents

Located in `.claude/docs/domains/`. Accumulated progressively as plan-writer explores new domains. Contains:

- Package structure
- API endpoints
- Dependencies on other domains
- Key design decisions
- Business rules

## External Integration

| Timing | Source |
|--------|--------|
| During work | Local spec files only (zero token cost) |
| After completion | Sync to Notion once (optional) |
| When needed | Fetch from Notion for meeting notes, external decisions |

## Design Decisions

| Decision | Rationale |
|----------|-----------|
| Spec files over session context | Sessions are disposable; specs persist across sessions |
| Waterfall for planning, XP for implementation | Thorough upfront design reduces reviewer burden and retry costs |
| DDD over TDD for existing projects | Existing codebase has established patterns; analyze domain first, then implement |
| opus for plan-writer | Plan quality determines overall quality; saves total tokens by reducing retries |
| haiku/sonnet for implementer | Following a detailed plan doesn't require expensive reasoning |
| Local spec over Jira/Notion during work | Eliminates integration token overhead |
