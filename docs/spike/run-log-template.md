# Litmus Spike — Run Log Template

> Duplicate this file as `run-<N>.md` for each of the 5 validation runs against `docs.dappbooster.dev` in fresh Claude Code sessions. Findings from all 5 runs feed the go/no-go decision in Issue #2 and PR #3.

---

## Metadata

- **Run number:** 1 of 5
- **Date / time (UTC):** YYYY-MM-DDTHH:MM:SSZ
- **SKILL.md version under test:** 0.3
- **Input URL:** https://docs.dappbooster.dev/
- **Session kind:** fresh (no prior context loaded)
- **Branch:** spike/2-pipeline-reliability
- **Run dir produced:** `.litmus/run-<TS>/`

## Pipeline completion

| Stage | Reached without manual prompt? | Output artifact valid? | Notes |
|-------|---|---|---|
| 0 — Setup (manifest, gitignore) | yes / no | yes / no | |
| 1 — Ingest | yes / no | yes / no | conversion_method used: turndown / pandoc / other |
| 2 — Generate (3 tasks) | yes / no | yes / no | |
| 3 — Execute | yes / no | yes / no | timeout enforced: yes / no |
| 4 — Evaluate | yes / no | yes / no | |
| Summary line printed | yes / no | — | |

## Per-task outcomes

| Task | Page (category) | Status | Duration (s) | Notes |
|------|-----------------|--------|--------------|-------|
| 001 | / (quickstart) | passed / failed / errored | | |
| 002 | / (recipe) | passed / failed / errored | | |
| 003 | / (advanced) | passed / failed / errored | | |

If any failed: which `root_cause` from taxonomy? (broken_example / missing_context / ambiguous_terminology / undocumented_gotcha / missing_decision_tree / other)

## Observed variance

To compare across runs, record:

- **Selected URLs** (which 3 paths)
- **Task descriptions** (one-line each)
- **Task dependencies** (which npm packages got installed)
- **Total wall-clock time**
- **Run dir size after Stage 3 cleanup**

## Frictions encountered

> A "friction" is anywhere the SKILL.md didn't give a clear decision and the agent had to improvise, OR anywhere the agent did something the SKILL.md forbid, OR anywhere the agent had to be prompted to continue.

For each friction:

- Stage / location
- Description
- Severity (low / medium / high)
- Suggested SKILL.md fix

## Go / no-go signal for this run

- Pipeline completed end-to-end without manual intervention: **yes / no**
- All output artifacts structurally valid: **yes / no**
- No artifacts written outside `.litmus/`: **yes / no**

## Free-form observations

(Anything else worth noting: agent behavior quirks, ambiguous moments, unexpected workarounds, etc.)
