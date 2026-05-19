# Litmus

> An AI agent skill that evaluates how well a documentation site works for AI agents executing real integration tasks.

## What it does

Litmus takes a documentation URL and produces an **Execution Score** — a measurement of whether an AI coding agent can complete real tasks using only the documentation provided. The score is paired with a per-task breakdown classifying failures by root cause and a prioritized list of doc sections to fix.

Unlike existing tools (AFDocs, Mintlify Agent Score, Fern Agent Score) that measure *readability* — can an agent parse the docs? — Litmus measures *execution* — can the agent actually do anything with them?

## Install

Litmus is a Claude Code skill, currently at version **1.1**. Install it once at the user level so it's available in every Claude Code session.

```bash
git clone https://github.com/luchobonatti/litmus-axscore.git /tmp/litmus
mkdir -p ~/.claude/skills
cp -r /tmp/litmus/.claude/skills/litmus ~/.claude/skills/
```

Or, to use it only inside a specific project, clone the repo and run Claude Code from the project root — the skill lives under `.claude/skills/litmus/` and Claude Code picks it up automatically.

**Requirements:** Node.js ≥ 20 (for `npx tsx`), `curl`, and either `turndown` (via `node -e` or `npx -y turndown-cli`) or `pandoc` for HTML→markdown conversion.

## Quickstart

In a Claude Code session inside any working directory (Litmus writes only under that directory), tell the agent:

```
run litmus on https://docs.example.com
```

Litmus crawls the site, generates 10 TypeScript tasks that test what the doc claims is possible, runs them in isolation, evaluates the outcomes, and prints a scorecard plus the path to a full report.

## What you get

After a run:

- An inline scorecard in chat with the Execution Score, grade, pass/fail/errored counts, top failure types, and top problem sections.
- A timestamped full report at `<cwd>/litmus-report-<TS>.md` containing the score, the prioritized fix list (sections ranked by failure count), per-task detail, and methodology.
- An append-only history index at `<cwd>/.litmus/reports-index.md` with one row per run (timestamp, hostname, score, grade, paths).
- Structured artifacts under `<cwd>/.litmus/run-<TS>/`: the ingested pages, the generated tasks, each task's execution (solution.ts, logs, result.json), and the evaluations.

See [`.claude/skills/litmus/templates/full-report.md`](.claude/skills/litmus/templates/full-report.md) for the report shape.

## Scope

Litmus v1.x measures **library-level TypeScript claims**. It does not validate:

- Interactive CLI wizards (`pnpm dlx <wizard>`, `vue create`, etc.) — skipped and reported as such.
- Tasks requiring API keys, paid services, or external credentials.
- Documentation for non-TypeScript ecosystems (Rust, Solidity, Python, etc.) — Litmus halts at the generate step with a `scope_mismatch` classification.

A doc that scores low because its surface is mostly non-TS or wizard-driven is an *agent-friendliness* signal, not a doc quality verdict.

## Repo layout

```
.claude/skills/
  litmus/      The Litmus skill (the deliverable)
  issue/       /sdlc:issue
  create-pr/   /sdlc:create-pr

docs/discovery/   Product brief, feasibility, roadmap
docs/validation/  Per-run validation reports

architecture.md   Pipeline data flow
CLAUDE.md         Agent operating manual for this repo
```

## Contributing

PRs are welcome. Each PR closes an issue, follows [Conventional Commits](https://www.conventionalcommits.org/), and is reviewed before merge. See [`CLAUDE.md`](CLAUDE.md) for the full operating manual.

## License

Apache-2.0. See [`LICENSE`](LICENSE).
