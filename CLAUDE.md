# Litmus — Agent Configuration

Litmus is an AI agent skill that evaluates how well a documentation site works for AI agents executing real integration tasks. See [`docs/discovery/project-brief.md`](docs/discovery/project-brief.md) for product context.

This file is the agent operating manual. Architectural shape lives in [`architecture.md`](architecture.md). The product brief, feasibility, and roadmap live in [`docs/discovery/`](docs/discovery/).

---

## Stack & Conventions

| Category | Technology | Notes |
|----------|-----------|-------|
| Language | TypeScript (strict mode) | All scripts and task code |
| Runtime | Node.js ≥ 20 | Required for `npx`, `tsx` |
| Task executor | `npx tsx <file>` | Zero-config TS execution, no compile step |
| Package manager | npm (per-task) | Each task installs its own deps; no global lockfile |
| Distribution | Claude Code skill | `.claude/skills/litmus/` is the deliverable |
| Linting | oxlint | Fast, strict, ESM-first |
| Formatting | oxfmt | Pair with oxlint |
| Testing | vitest | Colocated `*.test.ts` |
| Naming | camelCase vars/functions, PascalCase types, kebab-case files | |

## Code Style

- **Semicolons:** no
- **Quotes:** single
- **Print width:** 100
- **Trailing commas:** all
- **Indent:** spaces, width 2
- **Imports:** ESM only (`"type": "module"`), absolute imports preferred

## Working Rules

- **Litmus is a skill, not a binary.** The deliverable is `SKILL.md` + auxiliary prompts and templates. No CLI, no hosted service.
- **No LLM API calls from Litmus code.** All LLM work is performed by the host agent following the prompts defined under `prompts/`. Litmus is a contract, not a runtime.
- **All run state lives under `.litmus/`.** Never write outside that directory during a run. Never read user credentials, `.env` files, or environment variables beyond what `npm install` needs.
- **Host agent is Claude Code only for MVP.** Cursor and other clients are deferred to v1.2 (see [`roadmap.md`](docs/discovery/roadmap.md)).
- **Working directory under `.litmus/run-<timestamp>/`** per PRD §6.2.
- **TypeScript only for task execution** in MVP. Other languages deferred to v2.

## Skill Authoring

When creating or modifying skills inside `.claude/skills/`, use the `skill-creator` skill — never hand-write `SKILL.md` from scratch. It enforces the Agent Skills spec (frontmatter, triggers, structure) and avoids drift.

The Litmus deliverable itself (`.claude/skills/litmus/`) MUST be authored through `skill-creator` when work on M2 begins.

### Project Skills in this Repo

| Skill | Path | Purpose |
|-------|------|---------|
| `sdlc:issue` | `.claude/skills/issue/` | Create GitHub issues from briefs against the repo's templates |
| `sdlc:create-pr` | `.claude/skills/create-pr/` | Create pull requests filled from git context and linked issue |
| `litmus` | `.claude/skills/litmus/` | Evaluate a docs site for AI-agent use and produce an Execution Score |

## Architecture

See [`architecture.md`](architecture.md) for the skill's structural shape, prompt layout, and pipeline stages.

For product scope and decisions, see [`docs/discovery/`](docs/discovery/).

## Testing

- **Framework:** vitest
- **Run tests:** `npm test`
- **What to test:** Skill helper scripts (ingestion, parsing, scoring math), prompt-output schema validators, report rendering.
- **What not to test:** The LLM-driven stages themselves. Those are validated empirically against `dappbooster.dev` (see roadmap M3).
- **Coverage:** Meaningful coverage on deterministic code paths. Do not chase a number.

## Commit Standards

Use [Conventional Commits](https://www.conventionalcommits.org/):

**Format:** `type(scope): subject`

- **Scope** is optional
- **Subject** uses imperative mood, lowercase after the colon, no trailing period
- **Body** (optional): separated by a blank line, explains *why*

**Prefixes:**

| Prefix | Purpose |
|--------|---------|
| `feat` | New feature |
| `fix` | Bug fix |
| `chore` | Maintenance, dependencies, config |
| `docs` | Documentation only |
| `refactor` | Code change that neither fixes a bug nor adds a feature |
| `test` | Adding or updating tests |
| `style` | Formatting, whitespace, semicolons |
| `ci` | CI/CD pipeline changes |
| `perf` | Performance improvement |
| `build` | Build system or external dependencies |
| `revert` | Reverts a previous commit |
| `wip` | Work in progress (avoid on main) |
| `release` | Release-related changes |

Never add `Co-Authored-By` or AI attribution lines.

## PR Workflow

- Every PR must reference an issue (`Closes #N`)

  > No related issue? Use `No related issue.` as the first line of the Summary section.

- Mirror the issue's acceptance criteria in the PR
- Self-review your diff before requesting peer review
- Keep PRs small and focused — one issue, one PR
- PR titles use the same conventional commit format (`feat: add task generation prompt`)
- Use `/sdlc:create-pr` to create PRs — it reads the template and fills every section automatically
- Use `/sdlc:issue` to create issues — never hand-craft `gh issue create` invocations

## Label Conventions

GitHub form dropdowns only work through the web UI. Labels are the API-reliable mechanism for structured metadata.

**Priority** (bugs, features, and epics):

| Label | Description |
|-------|-------------|
| `priority: critical` | Blocking work, system down, or security issue |
| `priority: high` | Must be addressed in current sprint |
| `priority: medium` | Should be addressed soon |
| `priority: low` | Nice to have, can wait |

The `/sdlc:issue` skill applies these labels automatically when creating issues via CLI.

## Guardrails

- Do not commit secrets, API keys, or credentials
- Do not modify CI/CD pipelines without team review
- Do not skip tests or linting to make a build pass
- Never use `rm -rf`; use `trash` for recoverable deletes
- Prefer `rg` / `fd` / `bat` over `grep` / `find` / `cat`
- When in doubt, ask — don't assume

## Change Strategy

- Prefer small, focused diffs over broad refactors
- Avoid introducing new patterns when a project pattern already exists
- Update docs only when behavior or workflow changes

## Validation Checklist

Run before declaring work done:

- `npm run lint` (when configured)
- `npm test` (when tests exist)
- Manual end-to-end pipeline run against `dappbooster.dev` (for skill-affecting changes)

## References

- [Project brief](docs/discovery/project-brief.md)
- [Technical feasibility](docs/discovery/feasibility.md)
- [Roadmap](docs/discovery/roadmap.md)
- [BootNode SDLC process](https://github.com/bootnode/sdlc) (internal reference)
