# Litmus — Technical Feasibility Assessment

**Phase:** 1 — Discovery
**Date:** 2026-05-15
**Companion to:** [project-brief.md](./project-brief.md)

---

## Summary

Litmus is technically feasible **conditional on one unverified assumption**: that Claude Code can reliably execute a multi-stage pipeline driven by a single `SKILL.md`. This assumption is gated by **Spike #1** (see Roadmap M1). All other risks are non-blocking.

## Risks

### R1 — CRITICAL — Pipeline reliability under SKILL.md driving

**Description.** The entire Litmus architecture rests on the assumption that Claude Code, given a `SKILL.md` describing a 5-stage pipeline, will faithfully execute all stages in order, write artifacts to the correct paths, classify outcomes using the provided taxonomy, and produce a final report. Skills typically perform best on short, atomic flows. A 5-stage pipeline with intermediate file artifacts, package installation, TypeScript execution, and structured classification is substantially more demanding.

**Failure modes observed in similar setups.**
- Agent skips stages it considers "obvious" or redundant.
- Agent compresses multi-step instructions into a single creative interpretation.
- Agent loses track of the working directory mid-run.
- Agent fabricates structured output (e.g., taxonomy classifications) rather than evaluating evidence.

**Mitigation.** Resolved before any MVP work begins via **Spike #1** (Roadmap M1). The spike implements a minimal end-to-end pipeline against `dappbooster.dev` and exits with a binary verdict: pipeline reliable enough for MVP, or not. If "not", the architecture must be revised (e.g., split into multiple smaller skills coordinated by an orchestrator, or backed by deterministic scripts at each stage).

**Acceptance signal.** Three consecutive runs of the minimal pipeline complete all stages without manual intervention and produce structurally valid output.

---

### R2 — HIGH — Task generation bias (same-LLM artifact)

**Description.** The same Claude model both generates tasks from the docs and executes them. This creates a circular bias: the model generates tasks it knows it can solve, inflating the Execution Score and reducing the diagnostic value of the report.

**Mitigation strategy (not in MVP, documented for Execution phase).**
- Separate the two stages into subagent invocations with restricted context.
- Task generation receives only the doc *index* (page titles + headings), not the full content.
- Task execution receives only the specific section referenced by the task, not the full doc.
- Optionally, vary the model between stages in future versions.

**MVP posture.** Accept the bias as a known limitation. Document explicitly in the report's methodology section. Address structurally in v1.1 or later if benchmark validation confirms the bias is significant.

---

### R3 — MEDIUM — Run-to-run variance

**Description.** PRD targets two consecutive runs within 10 score points. Typical LLM-driven evaluation pipelines exhibit 15–25 point variance. The 10-point target may be unachievable without seeding, score normalization, or multi-run averaging.

**Mitigation strategy.**
- Measure actual variance during Spike #1 across 5 runs.
- If variance > 15 points, revise the success criterion to "median of 3 runs within 15 points" or similar.
- Document variance characteristics in the report so users interpret single-run scores correctly.

**MVP posture.** Adjust the criterion based on measured variance; do not assume the PRD's number.

---

### R4 — MEDIUM — Taxonomy overlap and inconsistent classification

**Description.** The 5-category taxonomy has overlapping definitions:

- `broken_example` and `undocumented_gotcha` both describe code that fails — the difference is whether the doc *acknowledges* the constraint elsewhere.
- `missing_context` overlaps with `missing_decision_tree` when the missing information is "which option to choose."

An LLM evaluator will classify the same failure differently across runs.

**Mitigation strategy.**
- Add explicit disambiguation rules in `prompts/evaluation.md`: each category gets a primary signal that distinguishes it from neighbors, plus a tiebreaker.
- Provide 2–3 worked examples per category in the evaluation prompt.
- Track per-category confidence in the evaluation output; flag low-confidence classifications in the report.

**MVP posture.** Keep 5 categories. Invest in prompt engineering to reduce overlap. If classification consistency on dappbooster.dev is below 80% non-`other`, consider collapsing to 3 categories + `other` in v1.1.

---

### R5 — LOW — Supply chain risk during task execution

**Description.** Tasks execute `npm install` against package names declared by the doc. A malicious package matching an expected name could be installed.

**Mitigation.** Document the risk explicitly in README. v2 will add registry signing checks or an allowlist.

**MVP posture.** Acceptable for MVP given the user runs against trusted docs.

---

### R6 — LOW — Working directory hygiene

**Description.** Litmus writes to `.litmus/` in the user's cwd. If the user runs in a sensitive directory (production deployment, secret-containing repo), unexpected writes could cause confusion.

**Mitigation.** Add `.litmus/` to `.gitignore` automatically if a git repo is detected. Refuse to run if cwd contains common secret files (`.env`, `secrets.json`) without explicit user override.

**MVP posture.** Implement the `.gitignore` addition. Defer the secrets-check to v1.1.

---

## Dependencies

| Dependency | Type | Risk | Notes |
|---|---|---|---|
| Claude Code | Host runtime | Low | Versioned, stable. Spike #1 validates compatibility. |
| Node.js ≥ 20 | User-side prereq | Low | Standard. Documented in README. |
| `tsx` (npm) | Per-task install | Low | Stable, widely used. Installed per task. |
| `npm registry` | Network | Low | Standard. Permission scoped to per-task install. |
| The target doc site | External | Variable | Site can be down, change format, or block crawlers. Handle gracefully. |
| Skill format spec | Spec compatibility | Low for MVP, Medium for Cursor port | Claude Code's current skill conventions are stable enough for MVP. |

## Infrastructure Gaps

None. Litmus runs entirely inside the user's Claude Code session. No hosted infrastructure, no CI requirements for the MVP runtime. The repo itself needs:

- Standard GitHub setup (templates, branch protection).
- A README with installation instructions.
- The skill files at the conventional path (`.claude/skills/litmus/`).

## Tech Stack Decisions

| Layer | Decision | Rationale |
|---|---|---|
| Skill format | Claude Code skill conventions | Host agent is Claude Code only for MVP |
| Task execution language | TypeScript only | PRD scope. Covers majority of SDK docs |
| Task runner | `npx tsx <file.ts>` | Zero-config TS execution, no compile step |
| Package manager for tasks | `npm` (per-task `npm install`) | Per-task isolation, no global state |
| Repo source of truth | `github.com/luchobonatti/litmus-axscore` | Confirmed by user during Phase 1 brainstorming |
| Skill internal scripts | TypeScript with `tsx` | Same toolchain as task runner; no Python or Bash beyond simple shell calls |
| Validation doc | `dappbooster.dev` | BootNode-controlled, accessible, representative |

## Pre-existing Tech Debt

None. Greenfield project.

## Compliance and Security Considerations

- No PII processing. Litmus reads public documentation only.
- No credential handling. Skill explicitly forbids reading `.env` or environment variables beyond what `npm install` requires.
- No outbound network calls except to the target doc, npm registry, and (per task) endpoints the doc itself documents.

## Design Decisions

This section records architectural choices made during M2/v1.0 development, surfaced by validation runs. They are documented here (not in `architecture.md`) because they belong to the rationale layer — *why* the skill behaves a given way, not *how* it's structured.

### DD-1 — Out-of-scope docs are rejected at task generation, not by a pre-ingest language check

**Context.** The Foundry validation run (M3, see `docs/validation/foundry-run-1.md`) halted at task generation with `task_generation_shortfall: 0`. Foundry's documentation is Rust + Solidity, well outside Litmus v1.0's TS-only execution scope. The question surfaced: should Litmus pre-filter these docs with a language-fit detection step before ingestion?

**Decision.** **Keep the task-generation halt as the rejection point.** No pre-ingest language-fit check.

**Rationale.**
- A pre-ingest language detector would rely on heuristics (code-fence languages, import-path patterns, top-level URL structure). Each heuristic has plausible false positives: a TS doc with a Bash quickstart, a polyglot SDK doc with separate sections per language, a doc that documents both client TS and server Solidity.
- Ingestion provides *evidence* for the rejection: the ingested pages and their categories are written to `manifest.json` and `pages.json`. A maintainer reading the halt output sees "49 pages ingested, 0 library-level claims found" — which is more diagnostic than "rejected by heuristic before reading the doc."
- The cost of running ingestion against a misfit doc is small (≤ 30 seconds of fetches + conversion), bounded by the URL cap.
- Friction #22 (closed alongside this decision) enriches the halt with a `halt_classification` (`scope_mismatch` vs `low_quality` vs `insufficient_content`), so the task-generation halt now distinguishes "wrong tool for the doc" from "doc has gaps." That distinction is what users actually need.

**Implications.**
- v1.0 keeps a uniform 4-source ingest path (llms.txt → sitemap → BFS → cap-and-select), independent of doc language.
- A future v2 that supports multiple execution languages would change the question, not the architecture: the language scope expands, but the rejection still happens at task generation based on what was found, not by a pre-ingest guess.

**Status.** Implemented in SKILL.md v1.0 via the task-generation halt-classification rule.

## Sign-off

- [ ] Risks reviewed and accepted as documented
- [ ] R1 (Spike #1) acknowledged as blocking for all other work
- [ ] Tech stack decisions approved
- [ ] No additional risks identified during review
