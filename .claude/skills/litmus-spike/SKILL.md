---
name: litmus-spike
description: "Trigger: litmus spike, run litmus spike on, validate pipeline against. Run minimal 4-stage Litmus pipeline against a docs URL to validate Claude Code can drive it end-to-end."
license: Apache-2.0
metadata:
  author: bootnode
  version: "0.2"
---

# litmus-spike

## Activation Contract

Run when the user asks to validate the Litmus pipeline against a documentation URL.

**Input:** one HTTP(S) URL pointing to a **documentation site or section** (e.g. `docs.example.com`, `example.com/docs`). NOT a marketing landing, GitHub repo, demo app, or aggregator.

**Output:** structured artifacts under `<cwd>/.litmus/run-<TS>/` and a one-line summary.

Do NOT run when: the URL is missing, invalid, or clearly not a docs site; the user wants a full Litmus report (this is the spike — no reporting stage).

## Hard Rules

- Write only under `<cwd>/.litmus/`. Never write elsewhere.
- Never read `.env`, credentials, or environment variables beyond what `npm install` needs.
- Never perform web searches. Use only the input URL, its in-domain pages, and the npm registry.
- Use direct fetch (`curl -fsSL` in Bash, or Node `fetch()`) for existence and content checks. NEVER use model-mediated fetch (e.g. WebFetch) for raw verification — paraphrasing destroys the signal.
- Each stage produces a structured artifact. Later stages MUST only read prior artifacts — never recompute.
- Add `.litmus/` to `.gitignore` if a `.git` directory is present.
- `<TS>` is ISO-8601 UTC compact: `YYYYMMDDTHHMMSSZ` (e.g. `20260518T145200Z`).

## Decision Gates

| Condition | Action |
|-----------|--------|
| URL missing or not HTTP(S) | Ask user for valid docs URL; do not proceed |
| URL clearly a landing/repo/demo (no `/docs` path, no `docs.*` host, and no doc-shaped content on root) | Halt; ask user for the docs URL |
| `llms.txt` or `sitemap.xml` lists doc URLs exclusively on a DIFFERENT hostname | Halt; ask user to provide that hostname as input |
| `<cwd>/.litmus/run-<TS>/` already exists | Append `-N` suffix |
| Stage N artifact invalid or unparseable | Halt; report which stage failed |
| Fewer than 3 pages ingested in Stage 1 | Halt; report insufficient content |
| Task generation produces ≠ 3 tasks | Halt; report Stage 2 failure |

## Execution Steps

1. **Validate input.** Reject non-HTTP(S) URLs. Abort if missing or obviously not a docs URL.
2. **Initialize run dir.** Create `<cwd>/.litmus/run-<TS>/`. Add `.litmus/` to `.gitignore` if in a git repo. Write `manifest.json` with `{input_url, ts, skill_version}`.
3. **Stage 1 — Ingest.**
   1. Try `curl -fsSL <url>/llms.txt`. If 200 + non-HTML, parse links under sections titled `Documentation`, `Docs`, `Reference`, or `Guides`. Exclude URLs under `Optional`, `GitHub`, `Repository`, `Demo`. Keep only URLs on the input hostname.
   2. Else try `curl -fsSL <url>/sitemap.xml`. Filter to URLs whose path is under the input URL's path prefix.
   3. Else BFS from input URL: max 3 pages, same hostname, max depth 2.
   4. Pick up to 3 URLs. Fetch each, strip nav/footer/scripts, convert to clean markdown. Write `ingested/content/<slug>.md` per page and `ingested/pages.json` with `[{url, title, headings, char_count}]`.
4. **Stage 2 — Generate.** Read `pages.json`. Produce exactly 3 TypeScript tasks. Each: `{ id, description, success_criterion, relevant_sections, expected_dependencies }`. Diversify across pages. Write `tasks.json`.
5. **Stage 3 — Execute.** For each task: create `executions/task-NNN/`, write `solution.ts` and `package.json`, run `npm install --prefer-offline` then `npx tsx solution.ts` (60s timeout). Capture stdout, stderr, exit code in `result.json`.
6. **Stage 4 — Evaluate.** For each `result.json`, classify status `passed | failed | errored`. If failed, assign `root_cause` from the taxonomy: `broken_example | missing_context | ambiguous_terminology | undocumented_gotcha | missing_decision_tree`. Write `evaluations.json`.
7. **Summarize.** Print one line: `Run complete. N passed, N failed, N errored. Artifacts in <run-dir>.`

## Output Contract

- `<run-dir>/manifest.json`
- `<run-dir>/ingested/pages.json` and `<run-dir>/ingested/content/*.md`
- `<run-dir>/tasks.json`
- `<run-dir>/executions/task-NNN/{solution.ts, package.json, result.json}` (×3)
- `<run-dir>/evaluations.json`
- One-line summary in chat. NO report file.

## Common Mistakes

- Accepting a marketing landing or repo URL instead of a docs URL.
- Using model-mediated fetch (e.g. WebFetch) for raw content verification.
- Following ALL URLs in `llms.txt` blindly — filter to doc-section URLs on the input hostname only.
- Running stages out of order or recomputing prior outputs in a later stage.
- Writing artifacts outside `.litmus/`.
- Filling gaps in docs with prior training knowledge instead of letting tasks fail.
- Producing fewer or more than 3 tasks in Stage 2.

## References

- `docs/discovery/project-brief.md` — Litmus product scope and MVP boundary
- `architecture.md` — pipeline data flow and stage isolation rules
- `docs/discovery/feasibility.md` — Risk R1 (spike rationale)
