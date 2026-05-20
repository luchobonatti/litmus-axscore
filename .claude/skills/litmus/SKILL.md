---
name: litmus
description: "Trigger: run litmus on, litmus check, evaluate docs at, score doc for agents. Evaluate how well a docs site works for AI agents and produce an Execution Score with a prioritized fix list."
license: Apache-2.0
metadata:
  author: bootnode
  version: "1.1"
---

# litmus

## Activation Contract

Run when the user asks to evaluate, score, or audit a documentation site for AI agent use.

**Input:** one HTTP(S) URL pointing to a **documentation site or section** (e.g. `docs.example.com`, `example.com/docs`). NOT a marketing landing, repo, demo, or aggregator.

**Output:** inline scorecard, `<cwd>/litmus-report-<TS>.md` (timestamped, one per run), an append-only `.litmus/reports-index.md`, and structured artifacts under `<cwd>/.litmus/run-<TS>/`.

Do NOT run when: the URL is missing/invalid, or clearly not a docs site.

## Hard Rules

- Write only under `<cwd>/.litmus/`, `<cwd>/litmus-report-<TS>.md`, and `<cwd>/.gitignore` (per the rule below). Never write elsewhere. Never overwrite a prior report.
- Never read `.env`, credentials, or env vars beyond what `npm install` needs.
- All fetches via `curl -fsSL` (Bash) or `fetch()` (Node). NEVER use model-mediated fetch (e.g. WebFetch) for existence checks, raw content, OR conversion.
- HTML→markdown via a deterministic local tool: `turndown` (via `node -e` or `npx -y turndown-cli`) or `pandoc`. Record the tool in `manifest.json` under `conversion_method`.
- Each step produces a structured artifact. Later steps MUST only read prior artifacts — never recompute.
- Add `.litmus/` and `litmus-report*.md` to `.gitignore` if a `.git` directory is present. Append each entry only when it is not already an exact line in `.gitignore`.
- `<TS>` is ISO-8601 UTC compact: `YYYYMMDDTHHMMSSZ` (e.g. `20260518T145200Z`).
- All paths passed to Bash MUST be absolute. `cwd` is not reliable between calls.
- Tasks must be runnable in isolation: no `@/src/...` imports, no scaffold-dependent paths. Test library-level claims only.
- Tasks MUST NOT require interactive input (CLI wizards, prompts, stdin reads). Skip those flows and record them in `manifest.json` under `interactive_flows_skipped`.
- After the execute step captures results for a task, delete that task's `node_modules/` via `trash` or `find ... -delete`.

## Decision Gates

| Condition | Action |
|-----------|--------|
| URL missing, non-HTTP(S), or clearly not a docs site | Halt; ask user for valid docs URL |
| `llms.txt` or `sitemap.xml` lists docs only on a DIFFERENT hostname | Halt; ask user to provide that hostname as input |
| `<cwd>/.litmus/run-<TS>/` exists | Append `-N` suffix |
| Step artifact invalid or unparseable | Halt; report which step failed |
| Fewer than 3 pages selectable during ingest | Halt; insufficient content |
| Task generation produces < 10 library-level claims | Halt; record `manifest.halt_classification` per the generate-step rule |
| Task generation validation fails 3 consecutive times | Halt; record `task_generation_validation_failed: { failed_check, attempts }` in `manifest.json` |
| No portable timeout mechanism available | Set `enforced_timeout: false` in `result.json`; continue |
| `<cwd>/litmus-report-<TS>.md` already exists (same `<TS>`) | Append `-N` suffix to the timestamp |
| `.litmus/reports-index.md` does not exist | Create it from the template header, then append this run's row |
| Node major < 22 (measure step) | Set `manifest.readability_unavailable.reason = "node_version"`; continue to ingest |
| AFDocs install fails (npx exits nonzero, stderr matches `E404\|ENOTFOUND\|npm error`) | Set `manifest.readability_unavailable.reason = "afdocs_install_failed"`; continue |
| AFDocs returns valid JSON, `.testedPages >= 3` | Populate `manifest.readability` |
| AFDocs returns valid JSON, `.testedPages < 3` | Populate `manifest.readability_partial` with `reason: "low_sample_count"` |
| AFDocs exits nonzero and `jq` fails to parse output | Set `manifest.readability_unavailable.reason = "afdocs_runtime_error"`; continue |
| AFDocs exits 0 but `jq` fails to parse output | Set `manifest.readability_unavailable.reason = "afdocs_invalid_output"`; continue |
| `readability_unavailable` is set (render step) | Show `—` in Readability column; Overall = execution_grade with `(readability unavailable)` marker |
| `readability_partial` is set (render step) | Show `partial (<n> pages tested)` in Readability column; Overall = execution_grade with `(readability partial)` marker |
| Multiple of `manifest.readability`, `manifest.readability_partial`, `manifest.readability_unavailable` are set, or none is set, after step 3 | Halt; report invariant violation |

## Execution Steps

1. **Validate input.** Reject non-HTTP(S) URLs or obviously non-doc URLs.
2. **Initialize run dir.** Create `<cwd>/.litmus/run-<TS>/`. If `.git` is present, ensure `.gitignore` contains `.litmus/` and `litmus-report*.md` — append each only when not already an exact line (idempotent). Write `manifest.json` with `{input_url, ts, skill_version, conversion_method, interactive_flows_skipped: []}`.
3. **Measure readability.**
   Get the Node.js major version:
   ```
   NODE_MAJOR_RAW="$(node --version)"
   NODE_MAJOR="${NODE_MAJOR_RAW#v}"
   NODE_MAJOR="${NODE_MAJOR%%.*}"
   ```
   If `NODE_MAJOR` is less than 22, write to `manifest.json`:
   ```
   manifest.readability_unavailable = { reason: "node_version", detail: "<actual node version>", timestamp: <ISO 8601> }
   ```
   Then skip to step 4.

   Otherwise run with portable 300s timeout: `gtimeout 300 ...` (macOS), `timeout 300 ...` (Linux), or omit the prefix if neither is available:
   ```
   gtimeout 300 npx --yes afdocs@0.18.7 check "<docs_url>" --format json --score --max-links 50 --sampling deterministic > <cwd>/.litmus/run-<TS>/readability.json 2> <cwd>/.litmus/run-<TS>/readability.stderr.log
   ```
   Capture the exit code. Validate with:
   ```
   jq -e '.scoring.overall' <cwd>/.litmus/run-<TS>/readability.json
   ```
   Read `.testedPages` (integer) from the same file to branch between `readability` and `readability_partial`. Map results per Decision Gates. Always continue to step 4.

4. **Ingest.**
   1. Try `curl -fsSL <url>/llms.txt`. If 200 + non-HTML, parse links under sections titled `Documentation`/`Docs`/`Reference`/`Guides`. Exclude `Optional`/`GitHub`/`Repository`/`Demo`. Keep input-hostname URLs only.
   2. Else try `curl -fsSL <url>/sitemap.xml`. Filter to URLs under the input URL's path prefix.
   3. Else BFS from input URL: same hostname, max depth 3.
   4. **Cap and select.** If the candidate set from steps 1-3 exceeds 50, apply this deterministic selection:
      1. Categorize each URL using the path patterns in `prompts/task-generation.md` (quickstart, recipe, reference, advanced, other).
      2. Take up to 12 per category in source order, iterating categories in this priority: quickstart → recipe → reference → advanced → other.
      3. If still under 50 after categorical sampling, fill the remainder from the largest underused category, preserving source order. Tie-breaker when two or more categories share the same remaining count: pick by the priority order above (quickstart → recipe → reference → advanced → other).
      4. Stop at 50 total. Record the kept and dropped counts in `manifest.json` under `selection: {candidates_total, kept, dropped, per_category}`.
   5. For each candidate page (up to 50):
      1. **Try native markdown first.** If the candidate URL already ends in `.md`, fetch it as-is. Otherwise fetch `<page-url>.md`. In either case, if the response is 200 with `Content-Type` matching `text/markdown` or `text/plain`, use the body directly and set `conversion_method: 'native-markdown'` in `manifest.json`.
      2. **Else fetch the HTML version** and convert via the deterministic local tool (turndown or pandoc per Hard Rules). Set `conversion_method` to the tool used (`turndown`, `pandoc`, etc.).
      3. Strip nav/footer/scripts when converting from HTML.
      4. Write `ingested/content/<slug>.md` and append to `ingested/pages.json` with `[{url, slug, title, headings, char_count, category}]`. Category labels follow `prompts/task-generation.md`.
5. **Generate.** Apply [`prompts/task-generation.md`](prompts/task-generation.md) to `pages.json` and the ingested markdown. Produce exactly 10 TypeScript tasks, library-level only, diversified across pages and difficulty. Write `tasks.json`.
   - If fewer than 10 library-level claims are found, **halt** and record in `manifest.json`:
     - `task_generation_shortfall: <count>` — the count of library-level claims found (0 to 9).
     - `halt_classification` — one of:
       - `scope_mismatch` when `pages_ingested >= 5 AND library_level_claims == 0`.
       - `low_quality` when `pages_ingested >= 5 AND 0 < library_level_claims < 10`.
       - `insufficient_content` when `pages_ingested < 5`.
   - If post-draft validation fails 3 consecutive times, **halt** and record `task_generation_validation_failed: { failed_check, attempts: 3 }` in `manifest.json`. `failed_check` is one of `count`, `distinct_success_criteria`, `diversified_pages`, `difficulty_banding`; when multiple checks fail in the same attempt, record the first failing check in that order.
6. **Execute.** For each task, apply [`prompts/execution.md`](prompts/execution.md):
   1. Create `executions/task-NNN/`.
   2. Write `solution.ts` and `package.json` (`{name, private: true, type: "module", dependencies}`). No `tsx` dep. No `tsconfig.json`.
   3. Run `npm install --prefer-offline --no-audit --no-fund --silent` (capture to `install.log`).
   4. Capture `start_ts = Date.now()` immediately before launching the run command.
   5. Run with portable 60s timeout: `gtimeout 60 npx tsx solution.ts` (macOS), `timeout 60 npx tsx solution.ts` (Linux), or fall back per Decision Gate. Capture `end_ts = Date.now()` immediately after the process exits.
   6. Capture stdout, stderr, exit code, and `duration_ms` (= `end_ts - start_ts`) in `result.json`. Set `duration_ms_captured: true` in `manifest.json`.
   7. Delete `node_modules/`.
7. **Evaluate.** Apply [`prompts/evaluation.md`](prompts/evaluation.md) to each `result.json`. Build `evaluations.json` per the schema defined there.
8. **Report.** Compute Execution Score: `round(passed / total * 100)`. Render [`templates/scorecard.md`](templates/scorecard.md) inline in the chat. Render [`templates/full-report.md`](templates/full-report.md) to `<cwd>/litmus-report-<TS>.md` (timestamped, never overwrites prior runs). Append one row to `<cwd>/.litmus/reports-index.md` using [`templates/reports-index.md`](templates/reports-index.md) — create the file with its header if missing. Aggregate fix suggestions by `responsible_section`; prioritize sections by failure count.
9. **Summarize.** Print one line: `Litmus complete. Execution Score: <N>/100 (<grade>). Readability Score: <N>/100 (<grade>) [or: readability unavailable]. Full report: <cwd>/litmus-report-<TS>.md. History: <cwd>/.litmus/reports-index.md`.

## Output Contract

- `<run-dir>/manifest.json`
- `<run-dir>/ingested/pages.json`, `<run-dir>/ingested/content/*.md`
- `<run-dir>/tasks.json`
- `<run-dir>/executions/task-NNN/{solution.ts, package.json, result.json, install.log, stdout.log, stderr.log}` (×10, no `node_modules/`)
- `<run-dir>/evaluations.json`
- `<run-dir>/readability.json` (AFDocs raw output; may be absent, empty, or invalid when `readability_unavailable` is set)
- `<run-dir>/readability.stderr.log` (AFDocs stderr; always written during measure step)
- `<cwd>/litmus-report-<TS>.md` (one per run; never overwritten)
- `<cwd>/.litmus/reports-index.md` (append-only history index across runs)
- One-line summary in chat.

### Manifest: `readability` block

Populated when AFDocs produces valid JSON and samples at least 3 pages. Exactly one of `readability`, `readability_partial`, or `readability_unavailable` is set after step 3 — never multiple, never none.

```
"readability": {
  "tool": "afdocs",
  "afdocs_version": "0.18.7",
  "overall_score": <0-100>,                       // maps to .scoring.overall
  "overall_grade": <"A" | "B" | "C" | "D" | "F">, // maps to .scoring.grade (clamp "A+" → "A")
  "pages_tested": <integer>,                      // maps to .testedPages
  "categories": {
    "content-discoverability": <0-100>,  // .scoring.categoryScores.content-discoverability.score
    "markdown-availability": <0-100>,    // .scoring.categoryScores.markdown-availability.score
    "page-size": <0-100>,                // .scoring.categoryScores.page-size.score
    "content-structure": <0-100>,        // .scoring.categoryScores.content-structure.score
    "url-stability": <0-100>,            // .scoring.categoryScores.url-stability.score
    "observability": <0-100>,            // .scoring.categoryScores.observability.score
    "authentication": <0-100>            // .scoring.categoryScores.authentication.score
  },
  "failed_checks": [                     // .results[] filtered where .status in {"fail", "warn"}
    { "id": <string>, "category": <string>, "status": <"fail" | "warn">, "message": <string> }
  ],
  "raw_path": "readability.json",
  "timestamp": <ISO 8601>
}
```

### Manifest: `readability_partial` block

Populated when AFDocs produces valid JSON but samples fewer than 3 pages.

```
"readability_partial": {
  "tool": "afdocs",
  "afdocs_version": "0.18.7",
  "reason": "low_sample_count",
  "pages_tested": <integer>,        // < 3
  "raw_path": "readability.json",
  "timestamp": <ISO 8601>
}
```

### Manifest: `readability_unavailable` block

Populated when AFDocs cannot run or produces unparseable output.

```
"readability_unavailable": {
  "reason": "afdocs_install_failed" | "afdocs_runtime_error" | "afdocs_invalid_output" | "node_version",
  "detail": <string>,
  "timestamp": <ISO 8601>
}
```

### Overall Grade

"Worse" uses the ordering `F < D < C < B < A`.

- Both axes present: the worse of `readability_grade` and `execution_grade`.
- Readability unavailable: the execution grade with a `(readability unavailable)` suffix — e.g. `B (readability unavailable)`.
- Readability partial: the execution grade with a `(readability partial)` suffix — e.g. `B (readability partial)`.

## Grade mapping

| Score | Grade |
|-------|-------|
| ≥ 90 | A |
| ≥ 80 | B |
| ≥ 70 | C |
| ≥ 60 | D |
| < 60 | F |

AFDocs emits `"A+"` for scores ≥ 95; clamp to `"A"` when populating `manifest.readability.overall_grade`.

## References

- `prompts/task-generation.md`
- `prompts/execution.md`
- `prompts/evaluation.md`
- `templates/scorecard.md`
- `templates/full-report.md`
- `templates/reports-index.md`
