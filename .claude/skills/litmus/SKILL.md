---
name: litmus
description: "Trigger: run litmus on, litmus check, evaluate docs at, score doc for agents. Evaluate how well a docs site works for AI agents via a 5-stage pipeline producing an Execution Score and prioritized fix list."
license: Apache-2.0
metadata:
  author: bootnode
  version: "1.0"
---

# litmus

## Activation Contract

Run when the user asks to evaluate, score, or audit a documentation site for AI agent use.

**Input:** one HTTP(S) URL pointing to a **documentation site or section** (e.g. `docs.example.com`, `example.com/docs`). NOT a marketing landing, repo, demo, or aggregator.

**Output:** inline scorecard, `<cwd>/litmus-report-<TS>.md` (timestamped, one per run), an append-only `.litmus/reports-index.md`, and structured artifacts under `<cwd>/.litmus/run-<TS>/`.

Do NOT run when: the URL is missing/invalid, clearly not a docs site, or the user wants only a quick readability lint (use AFDocs for that — Litmus is heavier).

## Hard Rules

- Write only under `<cwd>/.litmus/` and `<cwd>/litmus-report-<TS>.md`. The single exception is `<cwd>/.gitignore`, which the rule below explicitly authorizes the runner to append to (idempotently). Never write anywhere else. Never overwrite a prior report — each run gets its own timestamped file.
- Never read `.env`, credentials, or env vars beyond what `npm install` needs.
- All fetches via `curl -fsSL` (Bash) or `fetch()` (Node). NEVER use model-mediated fetch (e.g. WebFetch) for existence checks, raw content, OR conversion.
- HTML→markdown via a deterministic local tool: `turndown` (via `node -e` or `npx -y turndown-cli`) or `pandoc`. Record the tool in `manifest.json` under `conversion_method`.
- Each stage produces a structured artifact. Later stages MUST only read prior artifacts — never recompute.
- Add `.litmus/` to `.gitignore` IF a `.git` directory is present AND `.litmus/` is not already an exact line in `.gitignore`. Idempotent — never appends duplicates. Same rule applies for `litmus-report*.md`: only add when absent.
- `<TS>` is ISO-8601 UTC compact: `YYYYMMDDTHHMMSSZ` (e.g. `20260518T145200Z`).
- All paths passed to Bash MUST be absolute. `cwd` is not reliable between calls.
- Tasks must be runnable in isolation: no `@/src/...` imports, no scaffold-dependent paths. Test library-level claims only.
- Tasks MUST NOT require interactive input (CLI wizards, prompts, stdin reads). Skip those flows and record them in `manifest.json` under `interactive_flows_skipped`.
- After Stage 3 captures results for a task, delete that task's `node_modules/` via `trash` or `find ... -delete`. Recursive-force-delete commands may be blocked by host hooks.

## Decision Gates

| Condition | Action |
|-----------|--------|
| URL missing, non-HTTP(S), or clearly not a docs site | Halt; ask user for valid docs URL |
| `llms.txt` or `sitemap.xml` lists docs only on a DIFFERENT hostname | Halt; ask user to provide that hostname as input |
| `<cwd>/.litmus/run-<TS>/` exists | Append `-N` suffix |
| Stage artifact invalid or unparseable | Halt; report which stage failed |
| Fewer than 3 pages selectable in Stage 1 | Halt; insufficient content |
| Task generation produces ≠ 10 tasks | Halt; report Stage 2 failure |
| No portable timeout mechanism available | Set `enforced_timeout: false` in `result.json`; continue |
| `<cwd>/litmus-report-<TS>.md` already exists (same `<TS>`) | Append `-N` suffix to the timestamp |
| `.litmus/reports-index.md` does not exist | Create it from the template header, then append this run's row |

## Execution Steps

1. **Validate input.** Reject non-HTTP(S) URLs or obviously non-doc URLs.
2. **Initialize run dir.** Create `<cwd>/.litmus/run-<TS>/`. If `.git` is present, ensure `.gitignore` contains `.litmus/` and `litmus-report*.md` — append each only when not already an exact line (idempotent). Write `manifest.json` with `{input_url, ts, skill_version, conversion_method, interactive_flows_skipped: []}`.
3. **Stage 1 — Ingest.**
   1. Try `curl -fsSL <url>/llms.txt`. If 200 + non-HTML, parse links under sections titled `Documentation`/`Docs`/`Reference`/`Guides`. Exclude `Optional`/`GitHub`/`Repository`/`Demo`. Keep input-hostname URLs only.
   2. Else try `curl -fsSL <url>/sitemap.xml`. Filter to URLs under the input URL's path prefix.
   3. Else BFS from input URL: max 50 pages, same hostname, max depth 3.
   4. For each candidate page (up to 50):
      1. **Try native markdown first.** Fetch `<page-url>.md` (append `.md` to the page URL). If 200 with `Content-Type: text/markdown` (or `text/plain` when the path already ends in `.md`), use the response body directly. Set `conversion_method: 'native-markdown'` in `manifest.json`.
      2. **Else fetch the HTML version** and convert via the deterministic local tool (turndown or pandoc per Hard Rules). Set `conversion_method` to the tool used (`turndown`, `pandoc`, etc.).
      3. Strip nav/footer/scripts (HTML path only — native markdown is already clean).
      4. Write `ingested/content/<slug>.md` and append to `ingested/pages.json` with `[{url, slug, title, headings, char_count, category}]`. Category labels follow `prompts/task-generation.md`.
4. **Stage 2 — Generate.** Apply [`prompts/task-generation.md`](prompts/task-generation.md) to `pages.json` and the ingested markdown. Produce exactly 10 TypeScript tasks, library-level only, diversified across pages and difficulty. Write `tasks.json`.
5. **Stage 3 — Execute.** For each task, apply [`prompts/execution.md`](prompts/execution.md):
   1. Create `executions/task-NNN/`.
   2. Write `solution.ts` and `package.json` (`{name, private: true, type: "module", dependencies}`). No `tsx` dep. No `tsconfig.json`.
   3. Run `npm install --prefer-offline --no-audit --no-fund --silent` (capture to `install.log`).
   4. Run with portable 60s timeout: `gtimeout 60 npx tsx solution.ts` (macOS), `timeout 60 npx tsx solution.ts` (Linux), or fall back per Decision Gate.
   5. Capture stdout, stderr, exit code in `result.json`.
   6. Delete `node_modules/`.
6. **Stage 4 — Evaluate.** Apply [`prompts/evaluation.md`](prompts/evaluation.md) to each `result.json`. Build `evaluations.json` per the schema defined there.
7. **Stage 5 — Report.** Compute Execution Score: `round(passed / total * 100)`. Render [`templates/scorecard.md`](templates/scorecard.md) inline in the chat. Render [`templates/full-report.md`](templates/full-report.md) to `<cwd>/litmus-report-<TS>.md` (timestamped, never overwrites prior runs). Append one row to `<cwd>/.litmus/reports-index.md` using [`templates/reports-index.md`](templates/reports-index.md) — create the file with its header if missing. Aggregate fix suggestions by `responsible_section`; prioritize sections by failure count.
8. **Summarize.** Print one line: `Litmus complete. Execution Score: <N>/100 (<grade>). Full report: <cwd>/litmus-report-<TS>.md. History: <cwd>/.litmus/reports-index.md`.

## Output Contract

- `<run-dir>/manifest.json`
- `<run-dir>/ingested/pages.json`, `<run-dir>/ingested/content/*.md`
- `<run-dir>/tasks.json`
- `<run-dir>/executions/task-NNN/{solution.ts, package.json, result.json, install.log, stdout.log, stderr.log}` (×10, no `node_modules/`)
- `<run-dir>/evaluations.json`
- `<cwd>/litmus-report-<TS>.md` (one per run; never overwritten)
- `<cwd>/.litmus/reports-index.md` (append-only history index across runs)
- One-line summary in chat.

## Grade mapping

| Score | Grade |
|-------|-------|
| ≥ 95 | A+ |
| ≥ 90 | A |
| ≥ 80 | B |
| ≥ 70 | C |
| ≥ 60 | D |
| < 60 | F |

## Common Mistakes

- Accepting a marketing landing, repo, or demo URL instead of a docs URL.
- Using model-mediated fetch (e.g. WebFetch) for any I/O — destroys reproducibility.
- Following all `llms.txt` URLs blindly. Filter to doc-section URLs on input hostname.
- Generating scaffold-dependent tasks (`@/src/*` imports). Use library-level claims.
- Trying to script interactive CLIs via `expect` or stdin injection. Skip them and record in `manifest.json`.
- Adding `tsx` as a task dependency. Use `npx tsx`.
- Forgetting to delete `node_modules/` after Stage 3 (~80MB bloat per run).
- Recomputing in a later stage what an earlier stage already produced.
- Using relative paths in Bash arguments — `cwd` is not reliable between calls.

## References

- [`docs/discovery/project-brief.md`](../../../docs/discovery/project-brief.md) — product scope (non-interactive)
- [`architecture.md`](../../../architecture.md) — pipeline data flow
- [`docs/discovery/feasibility.md`](../../../docs/discovery/feasibility.md) — risk register
- [`.claude/skills/litmus-spike/SKILL.md`](../litmus-spike/SKILL.md) — proven base contract from spike #2
- `prompts/task-generation.md` — Stage 2 instructions
- `prompts/execution.md` — Stage 3 instructions
- `prompts/evaluation.md` — Stage 4 instructions (with worked examples)
- `templates/scorecard.md` — inline scorecard format
- `templates/full-report.md` — per-run full report format
- `templates/reports-index.md` — historical index row format and table header
