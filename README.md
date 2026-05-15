# Litmus

> An AI agent skill that evaluates how well a documentation site works for AI agents executing real integration tasks.

**Status:** Phase 1 (Discovery) complete. Implementation begins after the Spike M1 lands. See [`docs/discovery/`](docs/discovery/) for product context.

## What it does

Litmus takes a documentation URL and produces an **Execution Score** — a measurement of whether an AI coding agent can actually complete real tasks using only the documentation provided. The score is accompanied by a per-task report classifying failures by root cause, and a prioritized list of doc sections that need attention.

Unlike existing tools (AFDocs, Mintlify Agent Score, Fern Agent Score) which measure *readability* — can an agent parse the docs? — Litmus measures *execution* — can the agent actually do anything with them?

## Status

This is a working repository, not a released product. Tracking issues live in [Issues](../../issues). The current milestone is **M1 — Spike: Pipeline Reliability Validation** (see [`docs/discovery/roadmap.md`](docs/discovery/roadmap.md)).

A v1.0 release is gated on:

1. Spike validation that Claude Code can reliably drive the pipeline from a `SKILL.md` (M1).
2. MVP implementation against `dappbooster.dev` (M2).
3. Single-doc benchmark calibration (M3).

## Stack

| Layer | Choice |
|---|---|
| Distribution | Claude Code skill (`.claude/skills/litmus/`) |
| Language | TypeScript (strict) |
| Runtime | Node.js ≥ 20 |
| Task executor | `npx tsx` |
| Host agent (MVP) | Claude Code only |

See [`architecture.md`](architecture.md) for details.

## Repo Layout

```
.claude/skills/         Skills installed in this repo
  litmus/               The Litmus skill (the deliverable — built during M2)
  issue/                /sdlc:issue (from bootnode-sdlc starter-kit)
  create-pr/            /sdlc:create-pr (from bootnode-sdlc starter-kit)

docs/discovery/         Phase 1 outcomes (brief, feasibility, roadmap)

.github/                Issue and PR templates
```

## Contributing

This project follows [BootNode's AI-enhanced SDLC](https://github.com/bootnode/sdlc): Discovery → Definition → Execution → Review → Release. Every PR closes an issue. Every issue is born from a template via `/sdlc:issue`.

- Conventional Commits enforced
- One issue, one PR
- See [`CLAUDE.md`](CLAUDE.md) for the full operating manual

## License

TBD — to be decided at M4 (public release).
