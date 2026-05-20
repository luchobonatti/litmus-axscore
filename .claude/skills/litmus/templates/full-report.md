# Full Report Template

> Rendered to `<cwd>/litmus-report-<TS>.md` at the end of a Litmus run. Each run gets its own file — prior runs are never overwritten. A summary row is appended to `.litmus/reports-index.md` by the report step; see `reports-index.md` template for that format.
>
> Placeholders use `{{double_braces}}`. The agent substitutes from `manifest.json`, `tasks.json`, and `evaluations.json`.

<!-- BEGIN TEMPLATE CONTENT — everything between this marker and the matching END marker is rendered verbatim to the report file, with `{{placeholders}}` substituted per the rules below. Inner ``` code fences are part of the report and must NOT be treated as template delimiters. -->

# Litmus Report — {{hostname}}

**Generated:** {{ts}}
**Input URL:** {{input_url}}
**Litmus version:** {{skill_version}}

---

## Readability

{{#if readability}}
**Score:** {{readability.overall_score}}/100 — **Grade {{readability.overall_grade}}**

**Pages tested:** {{readability.pages_tested}} · **Tool:** {{readability.tool}} {{readability.afdocs_version}}

| Category | Score |
|----------|-------|
| content-discoverability | {{readability.categories.content-discoverability}} |
| markdown-availability | {{readability.categories.markdown-availability}} |
| page-size | {{readability.categories.page-size}} |
| content-structure | {{readability.categories.content-structure}} |
| url-stability | {{readability.categories.url-stability}} |
| observability | {{readability.categories.observability}} |
| authentication | {{readability.categories.authentication}} |

{{#if readability.failed_checks.length}}
### Failing checks

{{#each readability.failed_checks}}
- **`{{id}}`** ({{category}}, {{status}}): {{message}}
{{/each}}
{{/if}}

**Raw output:** `{{cwd}}/.litmus/run-{{ts}}/readability.json`
{{/if}}

{{#if readability_unavailable}}
**Readability score unavailable.**

- Reason: `{{readability_unavailable.reason}}`
- Detail: {{readability_unavailable.detail}}
{{/if}}

---

## Execution Score

**Score:** {{score}}/100 — **Grade {{grade}}**

| Bucket | Count |
|--------|-------|
| Passed  | {{passed}} |
| Failed  | {{failed}} |
| Errored | {{errored}} |
| **Total** | **{{total}}** |

{{#if interactive_flows_skipped}}
### Interactive flows skipped

The following CLI wizard flows are referenced by the doc but were not validated (Litmus measures library-level claims only):

{{#each interactive_flows_skipped}}
- `{{this}}`
{{/each}}

To improve agent-friendliness, expose non-interactive equivalents (`--yes` flags, scriptable bootstraps, or library-level factory functions).
{{/if}}

---

## Prioritized fix list

Sections are ranked by the number of failing tasks attributed to them. Address the top entries first for the largest score impact.

{{#each prioritized_sections}}
### {{rank}}. `{{section_slug}}` — {{failure_count}} failing task(s)

{{#each failures_in_section}}
- **{{task_id}}** ({{root_cause}}): {{fix_suggestion}}
{{/each}}

{{/each}}

{{#if no_failures}}
No failures detected. The doc successfully supported all {{total}} generated tasks.
{{/if}}

---

## Per-task detail

{{#each tasks_with_evaluations}}
### {{id}} — {{status_emoji}} {{status}}

**Description:** {{description}}

**Difficulty:** {{difficulty}} · **Category:** {{category}}

**Relevant sections:** {{#each relevant_sections}}`{{this}}`{{#unless @last}}, {{/unless}}{{/each}}

**Dependencies:** {{#if expected_dependencies.length}}{{#each expected_dependencies}}`{{this}}`{{#unless @last}}, {{/unless}}{{/each}}{{else}}none{{/if}}

**Success criterion:** {{success_criterion}}

**Outcome:** exit code `{{exit_code}}`, duration {{duration_ms}}ms

**Stdout:**
```
{{stdout_excerpt}}
```

{{#if stderr_excerpt}}
**Stderr:**
```
{{stderr_excerpt}}
```
{{/if}}

{{#if failed}}
**Root cause:** `{{root_cause}}`
**Responsible section:** `{{responsible_section}}`
**Fix suggestion:** {{fix_suggestion}}
{{/if}}

{{#if errored}}
**Error phase:** {{error_phase}}
**Error message:** {{error_message}}
{{/if}}

---

{{/each}}

## Methodology

Litmus evaluates documentation by generating executable tasks that test what the doc claims is possible, then running them and classifying outcomes. Run-to-run scores can vary by ~10-15 points because task generation is LLM-driven.

- **Scope:** library-level claims only. Tasks requiring interactive CLI wizards, scaffold-dependent paths, paid services, or credentials are skipped.
- **Tasks per run:** {{task_count}}, diversified across pages and difficulty (target ~3 basic / ~4 intermediate / ~3 advanced).
- **Failure taxonomy:** `broken_example`, `missing_context`, `ambiguous_terminology`, `undocumented_gotcha`, `missing_decision_tree`, plus `other` as a last resort.
- **HTML→markdown conversion:** `{{conversion_method}}`.
- **Pages ingested:** {{ingested_pages_count}}.

## Artifacts

All structured outputs for this run are under `{{cwd}}/.litmus/run-{{ts}}/`:

- `manifest.json` — run metadata
- `ingested/pages.json` and `ingested/content/*.md` — ingested pages and per-page content
- `tasks.json` — generated tasks
- `executions/task-NNN/` — per-task execution artifacts (solution.ts, package.json, result.json, logs)
- `evaluations.json` — per-task evaluations

---

*Generated by Litmus v{{skill_version}}.*

<!-- END TEMPLATE CONTENT. The sections below are meta-documentation for renderers and reviewers; they do NOT appear in the rendered report. -->

## Substitution rules

- `{{ts}}` — ISO-8601 UTC compact from the `ts` field in `manifest.json`.
- `{{hostname}}`, `{{input_url}}`, `{{skill_version}}`, `{{conversion_method}}`, `{{interactive_flows_skipped}}` — from `manifest.json`.
- `{{readability}}` — truthy when `manifest.readability` is populated; its fields map directly to the sub-keys (e.g. `{{readability.overall_score}}`, `{{readability.overall_grade}}`, `{{readability.afdocs_version}}`).
- `{{readability.failed_checks}}` — array from `manifest.readability.failed_checks`. Each entry has `id`, `category`, `status` (one of `fail`, `warn`), `message`. Block is omitted when empty.
- `{{readability_unavailable}}` — truthy when `manifest.readability_unavailable` is populated.
- `{{score}}`, `{{grade}}`, `{{passed}}`, `{{failed}}`, `{{errored}}`, `{{total}}` — computed from `evaluations.json`.
- `{{prioritized_sections}}` — group `evaluations[]` where `status === "failed"` by `responsible_section`, sort by group size desc, then by section slug asc. Each entry contains `section_slug`, `failure_count`, `failures_in_section[]`.
- `{{tasks_with_evaluations}}` — `tasks.json` joined with `evaluations.json` on `task_id`. Order: by `id` asc.
- `{{status_emoji}}` — `✅` for passed, `❌` for failed, `⚠️` for errored.
- `{{stdout_excerpt}}` / `{{stderr_excerpt}}` — first 500 chars from `evidence.stdout` / `evidence.stderr`. Truncate with `…` if longer.
- `{{no_failures}}` — boolean: `failed === 0`.
- `{{ingested_pages_count}}` — `pages.json.length`.
- `{{cwd}}` — current working directory absolute path.

## Edge cases

- **All tasks errored.** Render the header with `Score: N/A — Grade: —` and a note explaining no score is computable. Skip the prioritized fix list and per-task detail sections; instead include an "Errored tasks" section listing each error_phase + error_message.
- **Zero failures.** Skip the prioritized fix list section. Add the `no_failures` line.
- **More than 50 sections in fix list.** Cap at top 20 by failure count; add a footnote: `(Showing top 20 of N sections. Full data in evaluations.json.)`

## Length budget

Target ≤ 5000 lines for a 10-task run. If the report exceeds this, truncate per-task stdout/stderr excerpts further (down to 200 chars) and indicate truncation.
