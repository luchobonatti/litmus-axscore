---
name: litmus-spike
description: "Trigger: litmus spike, run litmus spike on, validate pipeline against. Run minimal 4-stage Litmus pipeline against a docs URL to validate Claude Code can drive it end-to-end."
license: Apache-2.0
metadata:
  author: bootnode
  version: "0.3"
---

# litmus-spike

## Activation Contract

Run when the user asks to validate the Litmus pipeline against a documentation URL.

**Input:** one HTTP(S) URL pointing to a **documentation site or section** (e.g. `docs.example.com`, `example.com/docs`). NOT a marketing landing, repo, demo app, or aggregator.

**Output:** structured artifacts under `<cwd>/.litmus/run-<TS>/` and a one-line summary.

Do NOT run when: the URL is missing, invalid, or clearly not a docs site; the user wants a full Litmus report (this is the spike — no reporting stage).

## Hard Rules

- Write only under `<cwd>/.litmus/`. Never write elsewhere.
- Never read `.env`, credentials, or env vars beyond what `npm install` needs.
- All fetches via `curl -fsSL` (Bash) or `fetch()` (Node). NEVER use model-mediated fetch (e.g. WebFetch) for existence checks, raw content, OR conversion.
- HTML→markdown via a deterministic local tool: `turndown` (invoke through `node -e` or `npx -y turndown-cli`) or `pandoc`. Record the tool used in `manifest.json` under `conversion_method`.
- Each stage produces a structured artifact. Later stages MUST only read prior artifacts — never recompute.
- Add `.litmus/` to `.gitignore` if a `.git` directory is present.
- `<TS>` is ISO-8601 UTC compact: `YYYYMMDDTHHMMSSZ`.
- Tasks must be runnable in isolation: no `@/src/...` imports, no scaffold-dependent paths. Test library-level claims, not project-level.
- Tasks MUST NOT require interactive input (CLI wizards, prompts, stdin reads). If the doc's primary flow is interactive (e.g. `pnpm dlx <wizard>`), generate library-level tasks that test claims about packages, exports, and types — NOT the wizard flow itself. Record skipped interactive flows in `manifest.json` under `interactive_flows_skipped`.
- After Stage 3 captures results for a task, delete that task's `node_modules/` to keep run-dir bounded. Keep `package.json`, `solution.ts`, logs, `result.json`.

## Decision Gates

| Condition | Action |
|-----------|--------|
| URL missing, non-HTTP(S), or clearly not a docs site | Halt; ask user for valid docs URL |
| `llms.txt` or `sitemap.xml` lists docs only on a DIFFERENT hostname | Halt; ask user for that hostname |
| `<cwd>/.litmus/run-<TS>/` exists | Append `-N` suffix |
| Stage artifact invalid or unparseable | Halt; report which stage failed |
| Fewer than 3 pages selectable in Stage 1 | Halt; insufficient content |
| Task generation produces ≠ 3 tasks | Halt; report Stage 2 failure |
| No portable timeout mechanism available | Set `enforced_timeout: false` in `result.json`; continue |

## Execution Steps

1. **Validate input.** Reject non-HTTP(S) URLs or obviously non-doc URLs.
2. **Initialize run dir.** Create `<cwd>/.litmus/run-<TS>/`. Update `.gitignore` if in git. Write `manifest.json` with `{input_url, ts, skill_version, conversion_method}`.
3. **Stage 1 — Ingest.**
   1. Try `curl -fsSL <url>/llms.txt`. If 200 + non-HTML, parse links under sections titled `Documentation`/`Docs`/`Reference`/`Guides`. Exclude `Optional`/`GitHub`/`Repository`/`Demo`. Keep input-hostname URLs only.
   2. Else try `curl -fsSL <url>/sitemap.xml`. Filter to URLs under the input URL's path prefix.
   3. Else BFS from input URL: max 3 pages, same hostname, max depth 2.
   4. **Select exactly 3 URLs maximizing diversity:** one matching `/install*|/start*|/intro*|/quickstart*|/getting-started*` (quickstart), one matching `/guide*|/recipe*|/tutorial*|/components*` (recipe), one matching `/advanced*|/reference*|/api*|/internals*` (advanced). If fewer categories available, take first 3 from source order.
   5. Fetch each, convert HTML→markdown per Hard Rules. Strip nav/footer/scripts. Write `ingested/content/<slug>.md` per page and `ingested/pages.json` with `[{url, slug, title, headings, char_count, category}]`.
4. **Stage 2 — Generate.** Read `pages.json`. Produce exactly 3 TypeScript tasks. Each: `{ id, description, success_criterion, relevant_sections, expected_dependencies }`. One task per page; diversify by difficulty (basic/intermediate/advanced). Tasks MUST be library-level (no `@/src/*`, no scaffold paths). Write `tasks.json`.
5. **Stage 3 — Execute.** For each task:
   1. Create `executions/task-NNN/`.
   2. Write `solution.ts` (imports only declared dependencies). Write `package.json` with `{name, private: true, type: "module", dependencies}`. Do NOT add `tsx` as a dependency. Do NOT add `tsconfig.json`.
   3. Run `npm install --prefer-offline --no-audit --no-fund --silent` (capture to `install.log`).
   4. Run with portable 60s timeout: `gtimeout 60 npx tsx solution.ts` (macOS), `timeout 60 npx tsx solution.ts` (Linux), or fall back per Decision Gate.
   5. Capture stdout, stderr, exit code in `result.json`.
   6. Delete `node_modules/`.
6. **Stage 4 — Evaluate.** For each `result.json`, classify status `passed | failed | errored`. Build `evaluations.json` as an array of `{task_id, status, duration_ms, responsible_section, evidence: {stdout, stderr, exit_code}}`. If `failed`: also `root_cause` from taxonomy `broken_example | missing_context | ambiguous_terminology | undocumented_gotcha | missing_decision_tree`, plus `fix_suggestion`. If `errored`: also `error_phase` (`install`|`run`).
7. **Summarize.** Print one line: `Run complete. N passed, N failed, N errored. Artifacts in <run-dir>.`

## Output Contract

- `<run-dir>/manifest.json`
- `<run-dir>/ingested/pages.json`, `<run-dir>/ingested/content/*.md` (×3)
- `<run-dir>/tasks.json`
- `<run-dir>/executions/task-NNN/{solution.ts, package.json, result.json, install.log, stdout.log, stderr.log}` (×3, no `node_modules/`)
- `<run-dir>/evaluations.json`
- One-line summary in chat. NO report file.

## Common Mistakes

- Accepting a marketing landing, repo, or demo URL instead of a docs URL.
- Using model-mediated fetch (e.g. WebFetch) for any I/O — destroys reproducibility.
- Following all `llms.txt` URLs blindly. Filter to doc sections on input hostname.
- Picking the first 3 URLs without diversification. Apply the category selector.
- Generating scaffold-dependent tasks (`@/src/*` imports). Use library-level claims.
- Trying to script interactive CLIs via `expect` or stdin injection. Treat interactive flows as untestable and skip them.
- Adding `tsx` as a task dependency (use `npx tsx`).
- Forgetting to delete `node_modules/` after Stage 3 → disk bloat (~80MB per run).
- Running stages out of order or recomputing prior outputs in a later stage.

## References

- `docs/discovery/project-brief.md` — Litmus product scope
- `architecture.md` — pipeline data flow
- `docs/discovery/feasibility.md` — Risk R1 (spike rationale)
