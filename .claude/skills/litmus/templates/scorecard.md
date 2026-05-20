# Inline Scorecard Template

> Rendered to chat at the end of a Litmus run. Keep compact and scannable.
>
> Placeholders are wrapped in `{{double_braces}}`. The agent substitutes them from `evaluations.json` and `tasks.json`.

```
Litmus Report — {{hostname}}

  Readability:  {{readability_score}}/100  (Grade {{readability_grade}})
  Execution:    {{score}}/100  (Grade {{grade}})
  Overall:      **Grade {{overall_grade}}**

  Tasks executed:   {{total}}
  Passed:           {{passed}}
  Failed:           {{failed}}
  Errored:          {{errored}}
  {{#if interactive_flows_skipped}}
  Interactive flows skipped: {{interactive_flows_count}}  (see manifest.json){{/if}}

  Top failure types:
{{#each top_failure_types}}
    {{category_padded}} {{count}}
{{/each}}

  Top problem sections:
{{#each top_problem_sections}}
    {{section_padded}} {{count}} failures
{{/each}}

  Methodology: {{conversion_method}} for HTML→md, {{task_count}} tasks library-level, non-interactive scope.

Full report: {{cwd}}/litmus-report-{{ts}}.md
History:     {{cwd}}/.litmus/reports-index.md
```

## Substitution rules

- `{{hostname}}` — `URL(manifest.input_url).hostname`.
- `{{score}}` — `round(passed / total * 100)`.
- `{{grade}}` — per the grade mapping in `SKILL.md`.
- `{{readability_score}}` — `manifest.readability.overall_score`; show `—` when `manifest.readability_unavailable` is set.
- `{{readability_grade}}` — `manifest.readability.grade`; show `—` when unavailable.
- `{{overall_grade}}` — `min(readability_grade, execution_grade)` when both present; `<available_grade> (readability unavailable)` or `<available_grade> (execution unavailable)` when one axis is missing; `—` when both unavailable.
- `{{total}}`, `{{passed}}`, `{{failed}}`, `{{errored}}` — counts from `evaluations.json`.
- `{{top_failure_types}}` — top 3 by frequency, descending. Source: `evaluations[].root_cause` (failures only). Omit the block if empty.
- `{{top_problem_sections}}` — top 3 by failure frequency, descending. Source: `evaluations[].responsible_section` (failures only). Omit if empty.
- `{{interactive_flows_skipped}}` — boolean from `manifest.interactive_flows_skipped.length > 0`.
- `{{interactive_flows_count}}` — `manifest.interactive_flows_skipped.length`.
- `{{conversion_method}}` — from `manifest.conversion_method`.
- `{{cwd}}` — current working directory absolute path.
- `{{category_padded}}` / `{{section_padded}}` — the corresponding `{{category}}` / `{{section}}` value left-aligned in a 30-character-wide field, padded with spaces. If the value is ≥ 30 chars, append exactly one space before the count. Computed by the renderer, not a literal placeholder.

## Edge cases

- **All tasks errored.** Render the scorecard with `score: N/A`, `grade: —`, and a note `Pipeline errored — no Execution Score computable. See manifest.json for stage failures.`
- **Zero failures.** Omit the `Top failure types` and `Top problem sections` blocks entirely. Add a line: `No failures detected.`
- **Score below threshold.** When `score < 60` (grade F), append a line: `This doc is not currently agent-friendly. Start with the Top problem sections above.`

## Formatting notes

- Render in a fenced code block (the agent prints inside ```` ```text ```` so it appears monospaced in chat).
- Right-align counts under columns where possible (use whitespace padding).
- Total visible width should be ≤ 70 chars for terminal-friendliness.
