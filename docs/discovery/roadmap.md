# Litmus — Roadmap

**Phase:** 1 — Discovery
**Date:** 2026-05-15
**Companion to:** [project-brief.md](./project-brief.md), [feasibility.md](./feasibility.md)

> This roadmap captures **milestones, dependencies, and sequencing** — not dates. Estimation happens at the Epic/Issue level in Phase 2.

---

## Milestone Graph

```
M1 (Spike)  ──blocks──▶  M2 (MVP)  ──blocks──▶  M3 (Validation)  ──blocks──▶  M4 (Public Release)
                                                                                       │
                                                                                       ▼
                                                                          M5 (AFDocs / Readability)
                                                                                       │
                                                                                       ▼
                                                                          M6 (Cursor & cross-client)
                                                                                       │
                                                                                       ▼
                                                                          M7+ (v2: mocks, CI, history)
```

---

## M1 — Spike: Pipeline Reliability Validation

**Goal.** Resolve risk R1 (see [feasibility.md](./feasibility.md)). Determine, with evidence, whether Claude Code can reliably drive a multi-stage pipeline from a `SKILL.md` against a real doc.

**Scope.**
- Minimal `SKILL.md` covering 4 stages: ingest → generate (3 tasks only) → execute → evaluate.
- No reporting stage, no scorecard — just structured output to verify each stage ran.
- Run against `dappbooster.dev`.
- Execute 5 consecutive runs and measure: stage completion rate, working-dir consistency, output structural validity.

**Outcomes.**
- Verdict: pipeline reliable for MVP **or** architecture revision needed.
- Measured variance baseline (informs R3 mitigation).
- Initial draft of `SKILL.md` patterns that work and patterns that don't.

**Exit criteria.**
- ≥ 4/5 runs complete all 4 stages without manual intervention.
- All 5 runs produce valid JSON output at each stage.
- Variance in task count and classification across runs documented.

**Blocks.** Everything downstream. If exit criteria fail, M2 scope must be revised.

**Owner.** TBD (BootNode).

---

## M2 — MVP: Execution Score End-to-End

**Goal.** Deliver the MVP as defined in [project-brief.md](./project-brief.md).

**Scope.**
- Complete 5-stage pipeline: ingest → generate (10 tasks) → execute → evaluate → report.
- Working directory layout under `.litmus/` per PRD §6.2.
- Markdown report (`litmus-report.md`) + inline scorecard.
- 5-category failure taxonomy with disambiguation rules.
- All prompts (`task_generation.md`, `execution.md`, `evaluation.md`) refined from Spike learnings.

**Outcomes.**
- Litmus skill fully functional against any TypeScript-oriented doc site.
- Skill installable into a Claude Code session.
- README with install + quickstart.

**Exit criteria.**
- End-to-end run against `dappbooster.dev` completes in < 10 minutes.
- Report contains Execution Score, per-task breakdown, and prioritized fix list.
- Taxonomy classifies ≥ 80% of failures into a non-`other` category.

**Blocks.** M3 (validation cannot start without a functional MVP).

**Depends on.** M1.

---

## M3 — Validation: Single-Doc Benchmark + Calibration

**Goal.** Validate that Litmus's output correlates with human judgment on the chosen validation doc.

**Scope.**
- Human evaluator (BootNode) runs subjective assessment on `dappbooster.dev`: 5-point scale across 3 dimensions (completeness, clarity, executability).
- Run Litmus 3 times against the same doc; compare scores and failure distributions.
- Calibrate prompts based on divergence between Litmus and human judgment.
- Document variance characteristics for the methodology section of the report.

**Outcomes.**
- Calibration report: where Litmus agrees with humans, where it diverges, why.
- Updated prompts incorporating calibration learnings.
- Documented variance bounds.

**Exit criteria.**
- Litmus's failure taxonomy aligns with human classification on ≥ 80% of failures.
- Run-to-run score variance documented and reported in `litmus-report.md` methodology.
- No remaining critical defects in the pipeline.

**Blocks.** M4 (cannot release without validation).

**Depends on.** M2.

---

## M4 — Public Release (v1.0)

**Goal.** Make Litmus publicly available and usable by external doc maintainers.

**Scope.**
- Polish README, installation docs, and example outputs.
- Decide final repo location (BootNode org vs personal).
- Public announcement (post, blog, etc.).
- Tag `v1.0`.

**Outcomes.**
- v1.0 tag and release notes.
- Litmus usable by anyone with Claude Code.

**Exit criteria.**
- External user (non-BootNode) successfully runs Litmus end-to-end.
- README rated complete by 2 internal reviewers.

**Blocks.** Nothing downstream is strictly blocked — M5+ proceed independently after release.

**Depends on.** M3.

---

## M5 — AFDocs Integration (v1.1)

**Goal.** Add Readability Score by integrating AFDocs as a pre-ingest step.

**Scope.**
- Pin to a tested AFDocs version.
- Implement the readability check per PRD §7.2.
- Combined scorecard (Readability + Execution + Overall Grade).
- Graceful degradation if AFDocs fails.

**Outcomes.**
- v1.1 with both scores.

**Exit criteria.**
- AFDocs failures do not break the pipeline.
- Combined report renders correctly inline and in `litmus-report.md`.

**Depends on.** M4.

---

## M6 — Cursor & Cross-Client Compatibility (v1.2)

**Goal.** Port Litmus to Cursor (and validate cross-client parity).

**Scope.**
- Cursor skill installation instructions.
- Per-client variance measurement on the same doc.
- Documentation of cross-client behavioral differences.

**Outcomes.**
- v1.2 with documented Cursor support.

**Exit criteria.**
- Scores within 10 points across Claude Code and Cursor on the validation doc.

**Depends on.** M4.

---

## M7+ — v2 Backlog (sequence TBD)

Independent tracks, prioritized after v1.2 ships:

| Track | Description |
|---|---|
| Languages | Python, Go, Rust task execution |
| Curated task sets | Per product category (SDK, REST, framework, protocol) |
| Mock environments | For tasks requiring credentials |
| CI integration | GitHub Actions trigger, webhook |
| Historical runs | Compare across time, score deltas |
| Fix mode | Suggest or apply doc edits |
| Cost transparency | LLM token cost in report |

---

## Sequencing Rationale

- **Spike first, always.** R1 is the single highest-leverage risk; resolving it cheaply protects against catastrophic rework downstream.
- **MVP before AFDocs.** Execution Score is the unique wedge. AFDocs is a "completeness" addition that's better added once the core pipeline is proven.
- **Single-doc validation before public release.** External users need confidence that scores mean something. Benchmark of one well-known doc gives that.
- **Cursor after Claude Code.** Cross-client portability is a v1.2 concern, not MVP.

## Sign-off

- [ ] Sequencing and dependencies approved
- [ ] Milestone exit criteria agreed
- [ ] M1 owner identified
- [ ] v2 backlog deferred explicitly (no scope creep into MVP)
