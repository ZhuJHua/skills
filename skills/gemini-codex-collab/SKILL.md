---
name: gemini-codex-collab
description: Coordinate Gemini-led planning in Antigravity with Codex plan review, implementation, and code review in the codex-plugin-agy plugin. Use when the user requests Gemini and Codex collaboration or Antigravity delegates a phase of that workflow. Select the model and reasoning effort for each phase.
---

# Gemini + Codex Collaboration

Use with the Antigravity `codex` plugin ([ZhuJHua/codex-plugin-agy](https://github.com/ZhuJHua/codex-plugin-agy)). Follow its rules and built-in skills (`codex-rescue`, `codex-review`, `codex-adversarial-review`, `codex-prompting`, and `codex-result-handling`) for invocation, permission boundaries, handoff formatting, and results.

Gemini owns design, plan approval, acceptance criteria, verification, and triage. Codex reviews the proposed plan, implements scoped tasks, and reviews the resulting code changes.

## Choose the model and effort for each step

### Plan review with read-only `codex-rescue` (omit `--write`)

| What the plan needs checked | Model and effort |
| --- | --- |
| Clear scope, complete acceptance criteria, and straightforward edge cases | `gpt-6.2-sol` / `medium` |
| A key assumption that requires checking call paths or contracts across modules or systems | `gpt-6.2-astra` / `high` |
| Interacting constraints or failure scenarios, such as mixed versions, retries, and rollback during a migration | Consider `gpt-6.2-astra` / `xhigh` |

### Implementation with `codex-rescue` (`--write`)

| Task | Model and effort |
| --- | --- |
| Routine feature or local fix with a clear plan and acceptance criteria | `gpt-6.2-sol` / `medium` |
| Multi-module implementation with a settled design and known dependencies | `gpt-6.2-astra` / `medium` |
| Difficult diagnosis or implementation that requires tracing behavior across systems or reconciling several constraints | `gpt-6.2-astra` / `high` |

Forward `--network` or `--full-access` only when explicitly requested by the user for that invocation.

### Code review with `gpt-6.2-sol`

| Change | Review mode and effort |
| --- | --- |
| Low-risk diff or small follow-up with straightforward behavior to verify | Standard `codex-review` |
| A consequential risk or design assumption that needs focused scrutiny, such as one backward-compatibility constraint | `codex-adversarial-review` / `high` |
| Interacting failure modes or assumptions that require evaluating several scenarios, such as retries, concurrent writes, and partial failure during a migration | `codex-adversarial-review` / `xhigh` |

A small patch that touches a consequential invariant can use Sol `medium` for implementation and Sol `high` for focused review. Use these selections unless the user specifies a model or effort.

These defaults draw on OpenAI's [Codex model use cases](https://learn.chatgpt.com/docs/whats-new/gpt-62-astra-sol), [GPT-6.2 model guide](https://developers.openai.com/api/docs/guides/latest-model?model=gpt-6.2-astra), and [reasoning effort guidance](https://learn.chatgpt.com/guides/prompting-gpt-6-2-astra).

## Gemini's review threshold

In plan and code review, skip fixes for paths that are unreachable under supported usage or whose prerequisites make them exceptionally unlikely in practice. A plausible issue with unresolved reachability warrants a targeted check before Gemini decides whether to act.

## Workflow

For feedback on a supplied plan, use step 2 and return actionable findings and open verification items. If the user asks for a plan, return the finalized plan after step 2. Continue to step 3 only when the user requests implementation. Trivial edits can stay with Gemini.

1. **Draft the plan in Gemini.** Set scope, key assumptions, and acceptance criteria. Include code review in the proposed workflow when it fits the user's requested scope.
2. **Challenge the draft plan.** Start a fresh, read-only `codex-rescue` task (`--fresh`, omit `--write`). Ask Codex which assumptions fail under plausible conditions, what evidence supports each concern, and what to verify before implementation. Gemini resolves blockers and finalizes the plan before delivering it or delegating implementation.
3. **Delegate the finalized plan.** Use `codex-rescue` (`--write`) directly from the main Antigravity session by default. Split large changes into independently verifiable tasks; serialize write tasks on the same repository worktree (workspace leases block overlapping writers on one worktree) and parallelize only across isolated worktrees or read-only analyses (using the `codex-rescue` subagent via `invoke_subagent`, or `--background` with `codex-await`).
4. **Verify in Gemini.** Inspect the diff and run relevant checks against the finalized acceptance criteria.
5. **Review the code under the plugin's invocation rules.** Select `codex-review` or `codex-adversarial-review` by risk. Pick an explicit review scope (`--scope working-tree` or `--base <ref>`) for each relevant diff. Run both scopes when the task spans committed and uncommitted work. For focused review, carry forward the riskiest assumption from the plan.
6. **Triage and close in Gemini.** Apply the review threshold and route authorized delegated fixes back to their implementation threads (`--thread <id>` or `--resume`). Re-review based on remaining risk. After two rounds, report open findings and revisit the plan before continuing. Summarize material plan changes, what was verified, and any actionable issues that remain open.
