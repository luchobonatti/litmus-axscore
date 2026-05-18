# Litmus — Project Brief

**Phase:** 1 — Discovery
**Date:** 2026-05-15
**Status:** Draft — pending team sign-off

---

## What

Litmus is an AI agent skill that evaluates how well a documentation site works for AI agents executing real integration tasks.

The MVP produces a single score — the **Execution Score** — which measures whether an agent can actually complete tasks using only the documentation provided. The score is accompanied by a per-task report classifying failures by root cause, and a prioritized list of doc sections that need attention.

Litmus is distributed as a skill (`SKILL.md` + auxiliary prompts and templates) that runs inside Claude Code. It uses Claude Code's own authentication, file system, and execution environment. No API keys, no sandboxes, no hosted infrastructure.

## Why

AI coding agents read documentation millions of times per day to help developers integrate products. When docs fail agents, two distinct failure modes appear:

1. **Readability** — agent cannot parse, navigate, or discover the docs. Already measured by AFDocs, Mintlify Agent Score, and Fern Agent Score.
2. **Execution** — agent technically can read the docs, but cannot complete real tasks because examples break, context is missing, or terminology is ambiguous. **No existing tool measures this dimension.**

Litmus closes the second gap. That is its wedge.

## Who

| Persona | Use case |
|---|---|
| Doc maintainer at a developer-facing product | Wants to know whether their SDK/API docs work for agent users, get a prioritized fix list |
| Doc consultant or auditor (BootNode's primary persona) | Evaluates client documentation, uses the report as input for a remediation engagement |
| Developer evaluating an SDK | Wants a sanity check on whether their AI coding assistant will be able to help integrate the product |

Primary persona for MVP validation: **the consultant**. The MVP must be usable by BootNode on client engagements without further productization.

## MVP Scope (in)

- **Single command flow:** user provides a URL, Litmus produces Execution Score + per-task report.
- **Execution Score only.** Readability Score (via AFDocs) is deferred to v1.1.
- **Claude Code only** as host agent. Cursor and other clients deferred.
- **TypeScript only** as task execution language.
- **6-stage pipeline:** doc ingestion → task generation → task execution → evaluation → reporting (5 stages; AFDocs stage removed for MVP).
- **5-category failure taxonomy** as defined in the PRD (broken_example, missing_context, ambiguous_terminology, undocumented_gotcha, missing_decision_tree) + `other` fallback.
- **Markdown report** written to `<cwd>/litmus-report.md`, plus inline scorecard in chat.
- **Working directory under `.litmus/`** with manifest, ingested content, executions, evaluations.

## MVP Scope (out)

- AFDocs integration / Readability Score (v1.1).
- Cursor and other host agents (v1.2).
- Languages other than TypeScript (v2).
- Curated task sets per product category (v2).
- Mock environments for credentialed tasks (v2).
- CI integration, GitHub Actions triggers (v2).
- Historical comparison between runs (v2).
- Authentication against private docs (v2).
- Multi-page task dependencies (v2).
- "Fix mode" that suggests doc edits (v2).
- Web UI, standalone CLI, hosted SaaS (out of long-term scope).
- Distribution via marketplaces (initial: GitHub-only).

## Scope: non-interactive tasks only

Litmus measures **library-level claims that an AI agent can execute without human interaction**. Interactive CLI wizards (`pnpm dlx <wizard>`, `vue create`, `next create-next-app`, `prisma init`, `vite create`, and similar) are explicitly out of scope.

When a doc's primary install or setup flow is interactive, Litmus does NOT try to script it (no `expect`, no stdin injection, no `--yes` flag detection). Instead, Litmus generates tasks that test the *underlying library claims* — package existence, exported symbols, type shapes, function returns — and reports the interactive flow as a known limitation of the doc's agent-friendliness.

**Implication for doc maintainers:** if Litmus generates few or no executable tasks for a doc, the doc's flow is too dependent on interactive CLI steps. To improve the Execution Score, expose non-interactive equivalents: `--yes` flags, scriptable bootstraps, programmatic factory functions, or library-level entry points that bypass the wizard.

This decision is intentional. Litmus measures what AI agents can do in a single autonomous run. Anything requiring "click here in the wizard" is by definition outside the agent's reach in the current generation of tooling.

## Success Criteria (MVP)

- Installable and runnable in **under 2 minutes** from cold start by someone who has Claude Code already configured.
- A complete run (10 tasks) completes in **under 10 minutes** for a typical SDK doc.
- Execution Score and failure taxonomy correlate with subjective human assessment on **dappbooster.dev** (the validation doc).
- The 5-category taxonomy classifies **at least 80%** of failures into a non-`other` category.
- The pipeline runs reliably through all 5 stages **without manual intervention** for the validation doc.

## Non-Goals

- A general-purpose doc evaluator. Litmus is opinionated about agent-facing evaluation only.
- A replacement for human doc review.
- A static analyzer. Litmus runs tasks; it does not just lint markdown.

## Open Questions (deferred from PRD)

| Question | Defer to |
|---|---|
| Task count default (10?) | Execution phase — adjust after first runs |
| Cost transparency in report | v2 |
| Final repo location (`bootnode/litmus` vs personal) | Release phase |

## Sign-off

- [ ] Project lead approves brief
- [ ] Team reviewed scope and out-of-scope
- [ ] Open questions assigned an owner or deferred to a phase
