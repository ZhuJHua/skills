---
name: claude-codex-collab
description: Division of labor between Claude and Codex, plus which Codex model and reasoning effort to use for each kind of delegation. Claude owns design and planning; Codex implements (gpt-6.2-astra, medium) and reviews (gpt-6.2-sol, xhigh for deep review, standard review for light changes). Use whenever Claude is about to hand work to Codex through the codex plugin (/codex:rescue, /codex:review, /codex:adversarial-review, codex-rescue agent), or when the user asks Claude and Codex to collaborate, to "let Codex do it", to have Codex implement or review a change, or to plan with Claude and execute with Codex, even if they never name a model.
---

# Claude + Codex Collaboration

Written for the `codex` plugin from the `zhujhua-codex` marketplace. The upstream OpenAI plugin treats `--effort` in review commands as focus text and makes them user-invoked only, so these flag combinations do not carry over to it.

Claude plans; Codex executes and reviews. Claude's context holds the user's intent and the design decisions, so judgment-heavy work stays here. Codex gets bounded, well-specified work and a fresh-eyed review, so each model is used where it is strongest.

## Roles and models

| Phase | Owner | Model / effort |
| --- | --- | --- |
| Understand request, design, plan, split tasks, define acceptance criteria | Claude | (this session) |
| Implement a planned task | Codex | `--model gpt-6.2-astra --effort medium` |
| Review the result | Codex | `--model gpt-6.2-sol`; `--effort xhigh` for deep review (standard review cannot set effort) |
| Triage findings, decide fixes, final report to the user | Claude | (this session) |

Pass these flags explicitly on every delegation (standard review is the one exception: it has no `--effort`). Omitting them silently falls back to whatever `~/.codex/config.toml` defaults to (currently a different effort), so the split would not actually happen.

If Codex rejects a slug, check the exact names in `~/.codex/models_cache.json` or `~/.codex/config.toml` (they may carry a prefix such as `codex/`) and adjust; keep the astra-for-implementation / sol-for-review split.

## Workflow

1. **Design and plan in Claude.** Settle scope, file boundaries, constraints, and "done" criteria before delegating. Do not ask Codex to design or to choose between approaches; that is the part Claude is here for.
2. **Delegate implementation** with `/codex:rescue` (or `Agent(subagent_type: "codex:codex-rescue")` for parallel or context-isolated tasks):
   ```
   /codex:rescue --write --model gpt-6.2-astra --effort medium <task>
   ```
   Use `--write` only for authorized implementation; omit it for read-only investigation. The task text must carry the agreed plan, the exact files or modules in scope, constraints, and acceptance criteria (tests to run, behavior expected), because Codex does not see this conversation. Keep each task small enough to verify on its own; split large plans into several tasks, and run independent ones in parallel only if they touch different files.
3. **Check the output.** Read the diff and run the relevant tests yourself before moving on. Codex's report is a claim, not proof.
4. **Delegate review** after the implementation is in the working tree or branch (see review scope below). Choose the mode by how much the change can hurt:
   - **Deep review (default for non-trivial changes):** challenges the design and assumptions, takes focus text, and is the only mode that accepts `--effort`.
     ```
     /codex:adversarial-review --model gpt-6.2-sol --effort xhigh [--base <ref>] [focus text]
     ```
     Use it for new features, refactors, concurrency, data migrations, auth/security, public APIs, or anything hard to roll back. Put the intent and risky areas in the focus text so the reviewer challenges the right assumptions.
   - **Standard review (light):** Codex's built-in reviewer; looks for ordinary bugs, takes no focus text and rejects `--effort` (it runs at the config default, currently `high`).
     ```
     /codex:review --model gpt-6.2-sol [--base <ref>]
     ```
     Use it for small, low-risk diffs, intermediate checkpoints in a multi-task plan, or re-reviewing small fixes after triage.
   - When unsure, pick deep review. Say which mode you chose and why in the final report. Reviews are read-only: never add `--write`.
5. **Triage in Claude.** For each finding decide: valid, already handled, or wrong. If the user asked only for a review, stop here and report; fixing needs their go-ahead (or an accepted plan that includes it). Otherwise fix valid ones either directly (small) or by sending Codex another implementation task with the astra/medium settings. Re-review only if the fixes were substantial (a standard review is enough for small fixes).
6. **Report** to the user what was planned, what Codex changed, what the review found, and what you accepted or rejected.

## Beyond implement and review

Command mechanics (`--write`, `--resume`/`--fresh`, `--background`, `/codex:status|await|result|cancel|setup|transfer`, permission flags, exit codes) are defined by the codex plugin's own commands and runtime skill; follow those rather than restating them here. These are the choices that carry policy:

- **Hard bug or plan second opinion:** `/codex:rescue --model gpt-6.2-sol --effort xhigh` without `--write`. Stronger setting, and read-only so Codex cannot edit.
- **Independent tasks on different files:** one `Agent(subagent_type: "codex:codex-rescue")` per task, each prompt carrying the full task and the astra/medium flags.
- **Follow-up to a Codex task:** note the thread ID each implementation result reports. `--resume` picks the most recently updated thread, which is wrong when several tasks ran; use `--thread <id>` for a specific one and send only the new instruction. Without an ID for an older task, use `--fresh` with full context. `--fresh` also suits unrelated work.
- **Review scope:** uncommitted work needs `--scope working-tree` and no `--base`: a base compares committed history only and would silently skip the new changes. Use `--base <ref>` for committed branch work; if both exist, review each scope separately.
- **Background runs:** only when Claude has real independent work meanwhile; otherwise wait.

## Review loops

After a review, route each valid finding by size: tiny fixes Claude makes directly; anything larger goes back to the same Codex thread with `/codex:rescue --write --thread <id> --model gpt-6.2-astra --effort medium <finding + intended fix>`. Then re-review with the lighter mode unless the fix reshaped the design. Stop after two rounds and report what remains rather than looping, since repeated rounds usually mean the plan, not the code, is wrong.

## Rules of thumb

- Don't hand Codex vague tasks like "improve this". If you cannot state acceptance criteria, the planning is not finished.
- Don't use sol for implementation or astra for review; the pairing is deliberate (cheap, fast execution; slow, thorough scrutiny).
- Trivial edits (a typo, one-line fix) are faster done directly by Claude; skip delegation.
- If the user names a different model or effort for a task, that overrides the defaults here.
