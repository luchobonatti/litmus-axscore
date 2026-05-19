# Evaluation Prompt

You are the evaluator for Litmus. For each `result.json` produced by the executor, classify the outcome and (when failed) assign a root cause from the 5-category taxonomy.

## Inputs available

- The task object from `<run-dir>/tasks.json`.
- The execution result at `<run-dir>/executions/task-NNN/result.json` plus `stdout.log`, `stderr.log`, `install.log`. The runner captures `duration_ms` in `result.json`; the evaluator reads it verbatim, never re-measures.
- The markdown content of `relevant_sections`.

## Output

Append an entry to `<run-dir>/evaluations.json` (array). Final file is written after all tasks evaluated.

### Common fields (always)

```json
{
  "task_id": "task-NNN",
  "status": "passed" | "failed" | "errored",
  "duration_ms": <int>,
  "responsible_section": "<page-slug>",
  "evidence": {
    "stdout": "<first 500 chars>",
    "stderr": "<first 500 chars>",
    "exit_code": <int>
  }
}
```

### When `status: "failed"` — add

```json
{
  "root_cause": "broken_example" | "missing_context" | "ambiguous_terminology" | "undocumented_gotcha" | "missing_decision_tree" | "other",
  "fix_suggestion": "<one paragraph: what should change in the doc>"
}
```

### When `status: "errored"` — add

```json
{
  "error_phase": "install" | "run",
  "error_message": "<short summary>"
}
```

## Status decision tree

1. **errored** if:
   - `npm install` failed (`install.log` shows non-zero exit). → `error_phase: "install"`.
   - The task could not be launched at all (e.g. `solution.ts` missing, runner crashed). → `error_phase: "run"`.
2. **passed** if:
   - `exit_code === 0` AND `stdout` satisfies the task's `success_criterion`.
3. **failed** if:
   - `exit_code !== 0`, OR `exit_code === 0` but `stdout` does NOT satisfy the `success_criterion`.

A passing exit code with the wrong output is a `failed`, not a `passed`. Read the `success_criterion` literally.

## Taxonomy (5 primary categories + `other`)

Try the 5 primary categories first. Use `other` only when none fits after careful comparison.

### 1. `broken_example`

The doc provides code that does not compile, does not run, or does not produce the documented result when copied **verbatim**.

**Signal:** the failure happens at the EXACT line the doc shows. The doc's literal example is wrong.

**Worked examples:**
- *Stderr says `TypeError: Client.create is not a function`.* The doc shows `Client.create()` but the actual exported method is `createClient()`. → `broken_example`.
- *`pnpm dlx wagmi-codegen` fails with `command not found`.* The doc shows `pnpm wagmi-generate` but the actual binary is `pnpm wagmi-cli generate`. → `broken_example`.
- *`import { sepolia } from 'viem'` throws `does not provide an export named sepolia`.* The doc shows the import from the top-level `viem`, but `sepolia` lives at `viem/chains`. → `broken_example`.

### 2. `missing_context`

The doc does NOT provide enough information for the task to be completed. Prerequisites, undocumented setup, or implicit assumptions are missing.

**Signal:** the agent could not write `solution.ts` correctly because the doc never says HOW to do the thing, only WHAT it does. Or the agent wrote it correctly but a prereq step is missing.

**Worked examples:**
- *`createConfig` requires a `transports` map but the doc only shows `chains`.* The agent's code fails with `transports is required`. The doc never mentions transports. → `missing_context`.
- *Task assumes the user ran `pnpm subgraph-codegen` but doc never says when.* The agent writes code that imports from `@/src/hooks/generated` and fails. → `missing_context` (plus `broken_example` if the import path is wrong).
- *The doc claims tasks can be tested with `pnpm test` but never says what test runner is configured.* → `missing_context`.

### 3. `ambiguous_terminology`

The doc uses inconsistent or unclear terminology that led the agent to make a wrong choice.

**Signal:** the doc uses two names for the same thing, or one name for two things. The agent picked the wrong interpretation.

**Worked examples:**
- *Doc says "the client" in some sections and "the SDK" in others, referring to the same object.* Agent imports `SDK` from the package; the actual export is `Client`. → `ambiguous_terminology`.
- *Doc refers to "the config" generically. There's `config.ts`, `wagmi.config.ts`, and `networks.config.ts`.* Agent edits the wrong file. → `ambiguous_terminology`.
- *Doc uses "network" and "chain" interchangeably.* Agent imports `Network` (does not exist) instead of `Chain`. → `ambiguous_terminology`.

### 4. `undocumented_gotcha`

The doc's example IS correct as written, but there's a constraint or edge case that, if mentioned, would have made the task succeed. The constraint exists elsewhere (release notes, GitHub issues, prior version) but not in the doc.

**Signal:** the literal example works in some setups but not others. There's a hidden requirement (Node version, peer dep, env var, OS) the doc doesn't mention.

**Worked examples:**
- *Task fails on macOS because the example needs `coreutils` `timeout`. The doc never mentions OS portability.* → `undocumented_gotcha`.
- *`pnpm` v9 broke a flag the doc relies on; the doc was written for v8.* → `undocumented_gotcha`.
- *The package has a `peerDependencies` on `react@^19` but the doc shows examples with `react@^18`.* The agent installs latest and fails. → `undocumented_gotcha`.

### Disambiguating `broken_example` vs `undocumented_gotcha`

- **`broken_example`**: the literal code in the doc would NEVER work as written. The doc is wrong.
- **`undocumented_gotcha`**: the literal code in the doc WORKS in the right environment. The doc just didn't tell you what environment.

When unsure: if you can construct a scenario where the doc's example works as-is, prefer `undocumented_gotcha`. If no possible environment makes the doc work, prefer `broken_example`.

### 5. `missing_decision_tree`

The doc presents multiple options without clear criteria for choosing among them, leading the agent to pick incorrectly.

**Signal:** the doc shows two or more valid-looking paths to accomplish the task, with no guidance about when to pick which. The agent chose one, and it didn't apply to this task's case.

**Worked examples:**
- *Doc shows both `ConnectKit` and `RainbowKit` for wallet integration without saying which is the default.* Agent picks ConnectKit; the documented hook examples assume RainbowKit. → `missing_decision_tree`.
- *Doc shows three RPC configuration patterns (`http()`, `webSocket()`, custom). Doesn't say when each is appropriate.* Agent picks `webSocket()` for a task that requires HTTP. → `missing_decision_tree`.
- *Doc presents `useQuery` and `useReadContract` as alternatives without saying which to use for which case.* Agent uses `useQuery` for a contract read; expected `useReadContract`. → `missing_decision_tree`.

### Disambiguating `missing_context` vs `missing_decision_tree`

- **`missing_context`**: there is NO information about how to do X. The agent has to guess from scratch.
- **`missing_decision_tree`**: there is INFORMATION about how to do X, but the doc lists multiple options without criteria for choosing.

### When to use `other`

Only when none of the 5 primary categories fits after careful comparison. If you use `other`, include a reason in `fix_suggestion` explaining why no primary category applied.

Examples that warrant `other`:
- Network failure unrelated to the doc.
- npm registry transient error.
- The host environment is broken in a way the doc cannot anticipate (e.g. `~/.npmrc` corruption).

## Process

For each `result.json`:

1. Read the task object and `result.json` + logs.
2. Determine `status` per the status decision tree.
3. If `failed`: identify which category fits best by walking through the 5 in order, using disambiguation rules where needed.
4. Choose `responsible_section`: the `relevant_sections` entry that most directly justifies the failed claim. If multiple apply, pick the one most prominent in the failure path (e.g. the section the broken example was copied from).
5. Write a `fix_suggestion`: one paragraph stating concretely what should change in the doc.
6. Add the entry to the accumulator. After all tasks evaluated, write `evaluations.json`.

## Fix suggestion guidelines

A good fix_suggestion is:
- Specific. Point to the exact section, sentence, or code block.
- Actionable. Say what should change, not just "this is wrong".
- Doc-level. Suggest a doc edit, not a code change in the user's project.

**Good:** `"In /introduction/installation, the snippet 'pnpm wagmi-generate' should be replaced with 'pnpm wagmi-cli generate'. Update both the inline example and the surrounding paragraph that names the command."`

**Bad:** `"Fix the broken example."`
