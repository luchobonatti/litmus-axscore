# Reports Index Template

> Rendered to `<cwd>/.litmus/reports-index.md`. Append-only — each Litmus run appends one row at the bottom. Prior rows are never modified or removed.
>
> If the file does not exist, the report step creates it with the header block below, then appends the first row.

## Header block (write once on file creation)

```markdown
# Litmus Reports Index

Append-only log of Litmus runs in this working directory. Each row records one run.

Columns:

- **TS** — ISO-8601 UTC timestamp from the `ts` field in `manifest.json`, also used as the `<TS>` suffix in the report filename.
- **Hostname** — host portion of the doc URL evaluated.
- **Input URL** — full URL passed to Litmus.
- **Score** — Execution Score (0-100) or `N/A` when no tasks ran.
- **Grade** — letter grade derived from Score per the SKILL.md mapping; `—` when score is N/A.
- **Tasks (p / f / e)** — passed / failed / errored counts.
- **Report** — relative path from cwd to the per-run report file.
- **Run dir** — relative path from cwd to the structured artifacts dir.

| TS | Hostname | Input URL | Score | Grade | Tasks (p / f / e) | Report | Run dir |
|----|----------|-----------|-------|-------|-------------------|--------|---------|
```

## Row format (append one per run)

```markdown
| {{ts}} | {{hostname}} | {{input_url}} | {{score}} | {{grade}} | {{passed}} / {{failed}} / {{errored}} | [report](./litmus-report-{{ts}}.md) | [artifacts](./.litmus/run-{{ts}}/) |
```

## Substitution rules

- `{{ts}}` — from the `ts` field in `manifest.json`, ISO-8601 UTC compact (`YYYYMMDDTHHMMSSZ`).
- `{{hostname}}` — `URL(manifest.input_url).hostname`.
- `{{input_url}}` — `manifest.input_url`, displayed verbatim.
- `{{score}}` — `round(passed / total * 100)` from `evaluations.json`. When `total === 0` (no tasks ran), write `N/A`.
- `{{grade}}` — per the grade mapping in SKILL.md. When score is N/A, write `—`.
- `{{passed}}`, `{{failed}}`, `{{errored}}` — counts from `evaluations.json`.

## Append protocol

1. If `.litmus/reports-index.md` does NOT exist:
   1. Create the file with the header block above (including the table header row).
   2. Append the row for the current run.
2. If the file DOES exist:
   1. Open in append mode. Do NOT read-then-rewrite — that risks losing concurrent rows.
   2. Append a single newline followed by the row for the current run.

## Edge cases

- **All tasks errored.** Score `N/A`, Grade `—`. Row still gets appended.
- **Stage halted before evaluation.** Append a row with `Score: HALTED`, `Grade: —`, `Tasks: — / — / —`, and the run dir path.
- **File truncated or invalid.** Do NOT auto-repair. Halt with a clear error message and instruct the user to inspect the file.
