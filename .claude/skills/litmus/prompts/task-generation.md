# Stage 2 — Task Generation Prompt

You are the task generator for Litmus. Read the ingested documentation and produce exactly **10 executable TypeScript tasks** that test what the doc claims is possible.

## Inputs available

- `<run-dir>/ingested/pages.json` — index of ingested pages with `{url, slug, title, headings, char_count, category}`.
- `<run-dir>/ingested/content/<slug>.md` — clean markdown per page.

## Output

Write `<run-dir>/tasks.json` as an array of 10 task objects. Schema:

```json
{
  "id": "task-001",
  "description": "One concise sentence describing what to verify.",
  "language": "typescript",
  "success_criterion": "Objective condition the run must satisfy. e.g. 'Process exits 0 and stdout contains foo=bar'.",
  "relevant_sections": ["page-slug-1", "page-slug-2"],
  "expected_dependencies": ["pkg-name"],
  "difficulty": "basic | intermediate | advanced",
  "category": "quickstart | recipe | reference | advanced"
}
```

`expected_dependencies` is an array of **bare package names** (no version spec). Stage 3 resolves each entry to `"latest"` when authoring `package.json`, unless the relevant doc section pins a specific version, in which case the agent uses that version verbatim.

## Hard Rules

- **Exactly 10 tasks.** No more, no less.
- **Library-level only.** Tasks MUST run from a fresh empty directory with only `solution.ts` + `package.json`. No `@/src/*` imports. No assumptions about a pre-existing scaffold. If a doc claim only works inside a scaffolded project, generate a task that tests the *underlying library claim* (e.g. `verify package X exports symbol Y`) — not the project-level orchestration.
- **Non-interactive only.** Do NOT generate tasks that require CLI wizards, prompts, stdin reads, or human input. If the doc's flow is wizard-driven (`pnpm dlx <wizard>`, `vue create`, `vite create`, etc.), generate library-level tasks that test the published artifacts of that flow — package metadata, exported functions, expected types — and append the wizard command to `manifest.json` under `interactive_flows_skipped`.
- **No credentials.** Do NOT generate tasks requiring API keys, paid services, or external state the doc does not declare as free.
- **No web search.** Tasks should rely only on what the ingested markdown states. The runner won't have web access beyond npm registry + the documented endpoints.
- **Diversify across pages.** Use every ingested page at least once (when ≥ 10 pages available). When < 10 pages, balance allocation: never let one page own more than `ceil(10 / page_count) + 1` tasks.
- **Diversify by difficulty.** Target ~3 basic, ~4 intermediate, ~3 advanced. Hard limit: no fewer than 2 per band.
- **Each task tests a distinct claim.** No two tasks may share `success_criterion` semantically.

## Category labels for pages

When tagging `category` in `pages.json` (Stage 1) and `tasks.json`, use:

| Page path matches | category |
|---|---|
| `/install*`, `/start*`, `/intro*`, `/quickstart*`, `/getting-started*` | `quickstart` |
| `/guide*`, `/recipe*`, `/tutorial*`, `/example*` | `recipe` |
| `/api*`, `/reference*` | `reference` |
| `/advanced*`, `/internals*`, `/architecture*` | `advanced` |
| anything else | use the page's top-level heading as category |

## Positive examples (good tasks)

```json
{
  "id": "task-001",
  "description": "Verify the `dappbooster` npm package is published and resolvable, and print its current latest version.",
  "language": "typescript",
  "success_criterion": "Process exits 0 and stdout matches `dappbooster@<semver>`.",
  "relevant_sections": ["introduction-installation"],
  "expected_dependencies": [],
  "difficulty": "basic",
  "category": "quickstart"
}
```

```json
{
  "id": "task-005",
  "description": "Import `createConfig` from `wagmi` and instantiate a config with the `sepolia` chain. Print whether the resulting object has the expected `chains` array length.",
  "language": "typescript",
  "success_criterion": "Process exits 0 and stdout contains `chains.length=1`.",
  "relevant_sections": ["recipes-my-first-dapp"],
  "expected_dependencies": ["wagmi", "viem"],
  "difficulty": "intermediate",
  "category": "recipe"
}
```

```json
{
  "id": "task-009",
  "description": "Use the documented `@bootnodedev/db-subgraph` factory and verify it returns a config with the keys the recipe claims (`schema`, `generates`, `pluckConfig`).",
  "language": "typescript",
  "success_criterion": "Process exits 0 and stdout lists all three keys, one per line.",
  "relevant_sections": ["advanced-subgraph-plugin"],
  "expected_dependencies": ["@bootnodedev/db-subgraph"],
  "difficulty": "advanced",
  "category": "advanced"
}
```

## Negative examples (do NOT generate)

```json
{
  "id": "bad-1",
  "description": "Run `pnpm dlx dappbooster` and verify the project scaffolds correctly.",
  "success_criterion": "The scaffold succeeds."
}
```
Bad: requires interactive wizard input; success is subjective.

```json
{
  "id": "bad-2",
  "description": "Check that the docs are well-written and easy to understand."
}
```
Bad: not testable; not a library claim; subjective.

```json
{
  "id": "bad-3",
  "description": "Copy the entire WETH dApp example from the recipe into a fresh dAppBooster scaffold and verify it renders.",
  "success_criterion": "The dApp renders the WETH page."
}
```
Bad: requires scaffold; involves UI rendering; tests integration not library claim.

```json
{
  "id": "bad-4",
  "description": "Use the `useTokenInput` hook from `@/src/components/sharedComponents/TokenInput/useTokenInput` and verify it returns the expected state.",
  "success_criterion": "stdout matches the doc's example output."
}
```
Bad: `@/src/*` import path requires a scaffold; the hook is project-internal, not a published package.

## Process

1. Read `pages.json` and the markdown content under `ingested/content/`.
2. List every concrete claim the doc makes about a published package, exported symbol, configuration shape, or chain/library constant.
3. From that list, select 10 claims that satisfy the Hard Rules above.
4. For each, draft the task object per the schema.
5. Validate: count = 10, distinct success_criteria, diversified pages, difficulty banding holds.
6. Write `tasks.json`. Halt if validation fails (per Decision Gate in `SKILL.md`).

## Failure mode

If the doc has fewer than 10 testable library-level claims, STOP and record this in `manifest.json` under `task_generation_shortfall: <count>`. Halt Stage 2.
