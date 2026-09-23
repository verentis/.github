---
description: Heal the gaps a verify review recorded — fix each failing requirement so it is genuinely satisfied, one gap at a time, without re-running work that already passed. The repair step of the built-in SDD flow, run when a verdict has gaps.
short-description: Fix the gaps a verify review recorded, one at a time.
requiredTools:
  - read_spec
  - todo_update
  - record_spec_traceability
  - create_sub_agent
---

# SDD — Heal

You are running the **heal** step — the targeted repair after `/sdd-verify` recorded a verdict with **gaps**. Your job is to make each failing requirement genuinely satisfied, fixing **only** what verify judged deficient. When you finish, **stop** — `/sdd-verify` re-runs to re-judge the gaps (automatically in the Specs panel; otherwise the user runs it).

This is **gap-driven, not a task re-run**: the implementation's `tasks.md` checkboxes stay exactly as they are (`[x]`). **Never tick or untick a task** (`complete_spec_task`) and **never call `advance_spec_phase`** — the heal is proven by the re-run of verify, not by checkboxes, and the phase gate belongs to the user. The tasks already passed; the review found specific defects, and those are all you touch. The spec is in the **verification** phase throughout — healing is part of the review loop, and it stays there until the review passes.

You already know how to model, generate, write code, and build/test — run **your normal model → generate → code → verify workflow**, scoped to each gap. This skill only tells you _which_ defects to fix, _how to track_ the work, and _what not to touch_.

## Honour the solution's governance

When the spec's framework declares governance documents — Spec Kit's `.specify/memory/constitution.md`, say — `read_spec` returns them as `governanceDocuments` and the Specs panel lists them. **Read them first and treat their rules as binding on what you produce.** They are the user's standing constraints, not background reading. If a requirement and a governance rule genuinely conflict, say so and ask rather than silently picking one.

## What to do

**First, resolve the slug — never guess it.** If you weren't handed an exact slug, call **`read_spec` with no `slug`**: it lists every spec in the solution with its `slug`, `phase` and progress. Match what the user named against that list. Specs live under `intent/.specs/` and are reachable **only** through `read_spec` — never glob or grep the workspace hunting for spec artifacts. If nothing there plausibly matches what they named, `ask_user_question` offering the closest specs as options rather than inventing a slug.

1. **Load the gaps and the context.**
   - `read_spec(slug, artifact: "verdict")` — the per-requirement verdict. Each **gap** carries your brief: `reason` (the summary), `expected` (what the acceptance criterion requires — your goal), `actual` (what's there now / the deficiency), and `location` (the file(s)/element(s) to fix). Ignore the `pass` rows.
   - `read_spec(slug, artifact: "requirements")` — the EARS acceptance criteria for the failing requirements.
   - `read_spec(slug, artifact: "design")` — the intended model changes + realization plan, so your fix fits the design.
   - `read_spec(slug, artifact: "traceability")` — which tasks/elements/files already realize each failing requirement (i.e. **where** the fix lands). Compact target summaries by default; `read_spec(slug, artifact: "traceability", requirementId: "Rn")` pulls one failing requirement's full targets.
2. **Turn the gaps into a todo list.** Call `todo_update` with **one entry per gap** (e.g. `Heal R2 — <short reason>`), all `pending`. This gap-level list is the live progress the user watches — one item per failing requirement, nothing for the requirements that passed.
3. **Heal each gap, one at a time** (mark it `in_progress`, fix it, mark it `completed`). For each gap:
   - **Target the specific defect.** The gap's `expected` is the goal, `actual` is what's wrong, and `location` + the requirement's traceability targets tell you where. Make the **smallest change that closes the gap** — do **not** rebuild work that already passed, and do **not** re-do requirements that verified clean.
   - **Run your normal workflow** on that defect: model the public surface with `run_designer_script` (Domain → Services → UI order when it spans designers) and validate the scoped `errors[]`; if you changed the model, run the Software Factory **once** and apply the staged changes so the code is on disk; dispatch a **`coding`** sub-agent for operation bodies / custom logic / tests (pass it the `slug`, the `applicationId`, the exact files/operations, the gap's `expected` **and** `actual`, the acceptance criteria, and require it to report the relative paths it wrote, any **model gap** it needs you to fill, and any hand-written member a regeneration would reclaim). It has no designer tools, so a gap comes back to you: model it, regenerate, and **inspect `get_file_diffs` before applying so your fix doesn't discard the code it just wrote**, then resume the same agent to finish. **Tell it explicitly that this is a heal, so it must NOT tick or untick any task** — the coding agent can call `complete_spec_task`, but heal's no-ticking rule governs, and a heal dispatch that ticks would fake progress the verify re-run never confirmed. Then run the build and test task(s) and **fix the cause and re-run** until green.
   - **Re-record traceability for what you touched.** For the task that satisfies the healed requirement, `record_spec_traceability(slug, taskId, requirementIds, targets, designerId)` with the targets you changed this run (`{ id, name }` for model elements, `{ kind: "file", relativePath }` for code) — passing `targets` replaces that task's links; passing **none** preserves them. If you only edited already-linked files and added no new element/file, there's nothing new to record.
   - **A broken/failed link is healed by re-recording the corrected target, never by editing `traceability.json` by hand.** If the gap was a broken link (an id/path that no longer resolves), re-record the task with the right element id (from `run_designer_script`'s `changes`) or the corrected `relativePath` — re-recording replaces the task's links. Then confirm the record call comes back with **no `failures`** before ticking the gap done: a `failures` entry means the corrected target was **rejected too, and nothing was stored**, so the requirement is still unlinked. Fix the target itself, not the message.
   - **Tick the gap's todo `completed` only once the defect is genuinely addressed and the build passes.** If you can't close a gap, **leave its todo open** and call it out in your report — never mark it done to move on.
4. **Idempotency check before you finish.** Re-run the Software Factory and inspect `get_file_diffs` **before applying**. If regeneration would delete or overwrite hand-written code, either model that surface (regenerate, re-fill the body) or protect it with Intent's code-management _ignore_ directive the way it's already used elsewhere. Re-run until regeneration is idempotent (no destructive diff) and the build passes.

## Then stop

When every gap's todo is `completed` (or you've done all you can), **reply with a concise report**: which gaps you closed and the elements/files you touched for each, and **any gap you couldn't close and why**. Change nothing else — `tasks.md` stays exactly as implementation left it, and you do **not** record a verdict.

Then **stop**. Do not re-verify yourself and do not advance the phase — `/sdd-verify` re-judges the gaps once your conversation ends (automatically in the Specs panel; otherwise the user runs it).

## Rules

- Keep the **model the source of truth**: if a fix needs structural change, adjust the model and re-run the Software Factory rather than hand-writing structural code.
- **Never touch the `tasks.md` checklist** (`complete_spec_task`) and never `record_spec_verdict` — heal repairs; verify judges. This applies to any `coding` sub-agent you dispatch too: it has `complete_spec_task` for spec-wave work, so say in its instructions that a heal dispatch ticks nothing.
- Only use ids / slugs returned from tool responses. Never invent them.
