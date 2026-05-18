---
name: litmus-spike
description: "Trigger: litmus spike, run litmus spike on, validate pipeline against. Run minimal 4-stage Litmus pipeline against a docs URL to validate Claude Code can drive it end-to-end."
license: Apache-2.0
metadata:
  author: bootnode
  version: "0.1"
---

# litmus-spike

## Activation Contract

Run when the user asks to validate the Litmus pipeline against a docs URL. Input: one HTTP(S) URL. Output: structured artifacts under `<cwd>/.litmus/run-<timestamp>/` and a one-line summary.

Do NOT run when: the URL is missing or invalid; the user wants a full Litmus report (this is the spike — no reporting stage).

## Hard Rules

- Write only under `<cwd>/.litmus/`. Never write elsewhere.
- Never read `.env`, credentials, or environment variables beyond what `npm install` needs.
- Never perform web searches. Use only the target URL and the npm registry.
- Each stage produces a structured artifact. Later stages MUST only read prior artifacts — never recompute.
- Add `.litmus/` to `.gitignore` if a `.git` directory is present.

## Decision Gates

| Condition | Action |
|-----------|--------|
| URL missing or not HTTP(S) | Ask user for valid URL; do not proceed |
| `<cwd>/.litmus/run-<timestamp>/` already exists | Increment suffix |
| Stage N artifact invalid | Halt, report which stage failed |
| Fewer than 3 pages ingested in Stage 1 | Halt, report insufficient content |
| Task generation produces ≠ 3 tasks | Halt, report Stage 2 failure |

## Execution Steps

1. **Validate input.** Reject non-HTTP(S) URLs. Abort if missing.
2. **Initialize run dir.** Create `<cwd>/.litmus/run-<timestamp>/`. Add `.litmus/` to `.gitignore` if in a git repo.
3. **Stage 1 — Ingest.** Try `<url>/llms.txt`; fall back to `<url>/sitemap.xml`; fall back to BFS from URL (max 3 pages, same hostname, max depth 2). Convert each page to clean markdown. Write `ingested/content/<slug>.md` per page and `ingested/pages.json` index with `{url, title, headings, char_count}`.
4. **Stage 2 — Generate.** Read `pages.json`. Produce exactly 3 TypeScript tasks. Each: `{ id, description, success_criterion, relevant_sections, expected_dependencies }`. Diversify across pages. Write `tasks.json`.
5. **Stage 3 — Execute.** For each task: create `executions/task-NNN/`, write `solution.ts` and `package.json`, run `npm install --prefer-offline` then `npx tsx solution.ts` (60s timeout). Capture stdout, stderr, exit code in `result.json`.
6. **Stage 4 — Evaluate.** For each `result.json`, classify status `passed | failed | errored`. If failed, assign `root_cause` from the taxonomy: `broken_example | missing_context | ambiguous_terminology | undocumented_gotcha | missing_decision_tree`. Write `evaluations.json`.
7. **Summarize.** Print one line: `Run complete. N passed, N failed, N errored. Artifacts in <run-dir>.`

## Output Contract

- `<run-dir>/ingested/pages.json` and `<run-dir>/ingested/content/*.md`
- `<run-dir>/tasks.json`
- `<run-dir>/executions/task-NNN/{solution.ts, package.json, result.json}` (×3)
- `<run-dir>/evaluations.json`
- One-line summary in chat. NO report file.

## Common Mistakes

- Running stages out of order or recomputing prior outputs in a later stage.
- Writing artifacts outside `.litmus/`.
- Filling gaps in docs with prior training knowledge instead of letting tasks fail.
- Skipping the `.gitignore` update.
- Producing fewer or more than 3 tasks in Stage 2.

## References

- `docs/discovery/project-brief.md` — Litmus product scope and MVP boundary
- `architecture.md` — pipeline data flow and stage isolation rules
- `docs/discovery/feasibility.md` — Risk R1 (spike rationale)
