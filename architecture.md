# Litmus — Architecture Overview

> Structural knowledge of the Litmus skill. This document will be filled in detail during Phase 3 (Execution). For now, it captures the shape committed during Phase 1 Discovery so future contributors and agents start with the right mental model.

For product context and decisions, see [`docs/discovery/`](docs/discovery/).

## Tech Stack

| Category | Technology | Notes |
|----------|-----------|-------|
| Distribution format | Claude Code skill | `.claude/skills/litmus/` |
| Language | TypeScript (strict) | All helper scripts and task code |
| Runtime | Node.js ≥ 20 | Required by `tsx` and the skill's helpers |
| Task executor | `npx tsx` | Zero-config TS execution per task |
| Package manager | npm | Per-task installs under `.litmus/run-*/executions/task-NNN/` |
| Testing | vitest | Colocated `*.test.ts` for deterministic code only |
| Linting | oxlint + oxfmt | |

## Project Structure (planned)

```
litmus-axscore/
├── .claude/
│   └── skills/
│       ├── litmus/                  Main skill — the Litmus deliverable
│       │   ├── SKILL.md             Entry contract for the host agent
│       │   ├── prompts/
│       │   │   ├── task-generation.md
│       │   │   ├── execution.md
│       │   │   └── evaluation.md
│       │   ├── templates/
│       │   │   ├── scorecard.md     Inline scorecard
│       │   │   └── full-report.md   Final report
│       │   └── scripts/             Deterministic helpers (TS, run via tsx)
│       │       ├── ingest.ts
│       │       ├── crawl.ts
│       │       ├── render-report.ts
│       │       └── schemas/         JSON schemas for stage outputs
│       ├── issue/                   sdlc:issue (from bootnode-sdlc starter-kit)
│       └── create-pr/               sdlc:create-pr (from bootnode-sdlc starter-kit)
├── docs/
│   └── discovery/                   Phase 1 outcomes
├── tests/                           vitest suites for helper scripts
├── architecture.md                  This file
├── CLAUDE.md                        Agent operating manual
├── AGENTS.md                        Pointer for non-Claude agents
└── README.md                        Public-facing install + usage
```

> Skeleton only. The actual directories appear as milestones progress (`scripts/` and `prompts/` materialize during M2).

## Pipeline Stages (MVP)

The Litmus skill drives Claude Code through five stages. Each stage produces a structured artifact under `.litmus/run-<timestamp>/`:

| # | Stage | Driver | Output |
|---|-------|--------|--------|
| 1 | Doc ingestion | Helper script (`crawl.ts`) | `ingested/pages.json`, `ingested/content/*.md` |
| 2 | Task generation | LLM (via `prompts/task-generation.md`) | `tasks.json` |
| 3 | Task execution | LLM + `npx tsx` per task | `executions/task-NNN/result.json` |
| 4 | Evaluation | LLM (via `prompts/evaluation.md`) | `evaluations.json` |
| 5 | Reporting | Helper script (`render-report.ts`) | `litmus-report.md` + inline scorecard |

Readability Score (AFDocs) is **out of MVP**. It joins as a pre-ingest step in v1.1 (see [roadmap M5](docs/discovery/roadmap.md)).

## Key Abstractions

### The skill contract

`SKILL.md` is the single source of truth for what the host agent does. It must:

- Define triggers explicitly.
- Enumerate pipeline stages in imperative order.
- Reference prompt files for LLM-driven stages.
- Reference helper scripts for deterministic stages.
- Forbid out-of-scope behavior (web search, credential reading, writes outside `.litmus/`).

### The working directory

All run state under `<cwd>/.litmus/run-<timestamp>/` per PRD §6.2. The skill adds `.litmus/` to `.gitignore` on first run if a git repo is detected.

### Stage isolation

Each stage produces a typed artifact validated by a JSON schema under `scripts/schemas/`. A later stage reads only artifacts from prior stages — never recomputes them. This makes failures isolable and runs resumable.

## Routes

Not applicable. Litmus is a skill, not a web service.

## Data Flow

```
User URL
    │
    ▼
crawl.ts ──▶ ingested/*.md
    │
    ▼
prompts/task-generation.md (LLM via host agent) ──▶ tasks.json
    │
    ▼
prompts/execution.md (LLM writes TS) ──▶ executions/task-NNN/{solution.ts, result.json}
    │
    ▼
prompts/evaluation.md (LLM classifies) ──▶ evaluations.json
    │
    ▼
render-report.ts ──▶ litmus-report.md + inline scorecard
```

Each arrow is a stage boundary. No stage reaches back across boundaries.

## Environment Variables

None required from the user. The skill explicitly forbids reading `.env` files or environment variables beyond what `npm install` needs.

## Scripts

| Command | Purpose |
|---------|---------|
| `npm test` | Run vitest suites for deterministic helpers |
| `npm run lint` | oxlint over `scripts/` and `tests/` |
| `npm run fmt` | oxfmt across the repo |

No `dev` or `build` script — the skill is consumed directly by Claude Code, not bundled.

---

## Notes for Future Maintainers

- Detailed architecture (interfaces between stages, schema definitions, error contracts) will be filled during M2 (MVP execution).
- The shape above is a commitment from Phase 1. Changes to the stage boundaries or the working-dir layout require updating both this file and the PRD.
