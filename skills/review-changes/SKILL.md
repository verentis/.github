---
description: Review the current change set with parallel finder sub-agents, adversarially validate every finding, and report only what survives. Use when the user asks for a review of what has just been built, staged, or changed — or to review a pull request and stage the findings into its pending review.
short-description: Review a change set with parallel finder sub-agents and adversarial validation.
argument-hint: Optional scope — a spec slug, an application, a folder, "staged", or a pull request ("PR #42 in <repo> over <refA>...<refB>")
requiredTools:
  - get_change_review
  - get_file_diffs
  - stage_pull_request_review
  - create_sub_agent
  - bash
---

# Review changes

Find real problems in the change set and **report only the ones that survive a challenge**. The value of this skill is the filter, not the finding: an unfiltered list of plausible-sounding issues costs the user more time than it saves.

$ARGUMENTS

## 0. Which mode are you in?

**Pull-request mode** when the scope above names a pull request — a number, a repository path, and a ref range. Everything below applies, plus: the range is fixed (§1), the layer classification drives who looks at what (§2a), and the findings are **staged into the pull request's pending review** rather than only reported in chat (§6). The working tree is out of scope entirely — you are reviewing what the author pushed, not what happens to be on this machine.

**Working-change mode** otherwise: the default. Report in chat and stage nothing.

## 1. Establish the change set — do this first, and concretely

You cannot review "the changes" in the abstract. Pin down exactly what changed before dispatching anyone:

- **Classified change set:** `get_change_review` — every changed file with its `FullyManaged` / `ManagedWithCustomisations` / `FullyCustom` classification, every changed designer element, and the requirement links over that range. Start here in **either** mode: it is computed, not judged, and §2a's dispatch depends on it. In pull-request mode pass the PR's own `refA` / `refB`; in working-change mode the defaults (merge-base → working tree) are right. If the repository you are running in holds no Intent solution, this tool has no model to reach: review from the diff alone, without §2a's layer weighting, and say so in the report.
- **Staged generated output:** `get_file_diffs` — the Software Factory changes not yet written to disk. Working-change mode only; a pull request has nothing staged.
- **The diff itself:** run `git diff --stat <refA>...<refB>` and then `git diff` over the files worth reading (`bash`, or `powershell` on Windows). In working-change mode use `git status --short` and the working-tree diff instead. Enumerate the change set — do not infer it. Only if the shell is unavailable to you should you fall back to `glob`/`read_file` over the areas the conversation has been working in, and in that case **say in your report that the change set was inferred rather than enumerated**.
- **Model changes:** the elements this work created or altered, via `get_designer_model_structure` / `find_designer_elements`. Code review that ignores the model misses the half of the change that generated the rest.

If the scope above narrowed things (a spec slug, an application, a folder), restrict the change set to it and say so. If you genuinely cannot determine a change set, ask the user rather than reviewing the whole repository.

## 2. Fan out finders — several dispatches in ONE response

Emit **all** the finder dispatches **in a single response** so they run concurrently. Each finder gets a **distinct lens** and the concrete change set from step 1 — never the same brief repeated.

**Which dispatch tool: your host's own, if it has one.** If the harness running you exposes native sub-agents (Claude Code's `Task` / `Agent`, or an equivalent), use that for the finder and validator waves. This is read-only parallel fan-out, which is exactly what those are for, and they run in-process with no extra hop. Use `create_sub_agent` with `agentId: discovery` when you have no native equivalent — which is the case for Intent's own in-app agents. The brief below is identical either way; only the tool changes. Do **not** dispatch the same lens through both.

The default lenses (drop any that the change set doesn't touch, add one when the change warrants it):

| Lens                           | What it hunts                                                                                                                                                                           |
| ------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Correctness**                | Logic that is wrong for a real input: null/empty/boundary cases, inverted conditions, wrong operator, mis-ordered operations, swallowed errors, resource leaks, races.                  |
| **Model ↔ code drift**         | Hand-written code that contradicts the model, structure inferred from files rather than designers, generated files edited by hand, elements the code needs that the model doesn't have. |
| **Conventions & instructions** | Violations of the repo's own `CLAUDE.md` / instruction files and of the surrounding code's established patterns — the rules this codebase actually states, not generic style opinion.   |
| **Contracts & integration**    | Callers left behind by a changed signature/DTO/endpoint, unhandled new failure modes, breaking changes to something already consumed.                                                   |

Every finder is **read-only** — a reviewer must not "fix" anything, so say so in the brief and, on a native dispatch, pick a read-only agent type if your host offers one. In each brief put: the exact files/elements in scope, the lens, the false-positive exclusions below, and **the required report shape** — one entry per finding with `file:line` (or element id), **the problem in one line**, what is wrong in two or three sentences, and a **severity** of `blocking`, `minor` or `nit` (§5). Tell them to report findings, not their search: no narration of what they read, no reasoning they can leave out, nothing about what they looked at and cleared.

### 2a. Weight the lenses by layer — this is what makes the review Intent's rather than generic

`get_change_review` told you what each changed file **is**. That classification decides who reviews it and how, and it is never inferred from the path or handed to a finder to guess:

| What the file is                     | How the review treats it                                                                                                                                                                                                                                                                                                                   |
| ------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **`FullyManaged`** (generated)       | **Do not dispatch a finder at it, and suppress any finding about it** — a regen fixes or reproduces it, so the fix is the model, never the file. One exception: the file was **hand-edited** despite being fully managed. That IS a finding (layer `FullyManaged`), and one only we can make — the edit is about to be silently reclaimed. |
| **`ManagedWithCustomisations`**      | Review with the model in hand: does the customisation fight what the template generates, or duplicate something the model could express?                                                                                                                                                                                                   |
| **`FullyCustom`**                    | Full finder attention. The generic bug lenses concentrate here — this is the code nobody generated and nobody checked.                                                                                                                                                                                                                     |
| **Changed model elements**           | The flagship lens: element changes whose generated consequences the author never inspected, and hand-written code that should have been modelled. Give this its own finder and tell it to read the elements, not just the diff.                                                                                                            |
| **Non-code** (docs, build, metadata) | Skim only. No finder of its own.                                                                                                                                                                                                                                                                                                           |

Say in the report how many generated files were suppressed. A suppressed file is not a reviewed file, and a reader who thinks it was has been misled.

**Cost shape:** finders are wide and mechanical; the validators in step 4 are the expensive half. That split is a **configuration** decision, not a per-dispatch one — the `discovery` persona already pins `effort: low`, so the fan-out is cheap without you doing anything, and a solution that wants its finders cheaper still says so in `discovery.agent.md` or a per-solution override of it. Where `create_sub_agent` exposes a `model` argument at all, don't guess at it: the keys are the user's own configured provider ids, nothing in your context lists them, and a wrong one either silently falls back to the current model or runs the review on the wrong one. Native sub-agents are the exception — if your host's own tool offers a fixed set of model tiers, a cheap tier for the finders and a strong one for the validators is exactly right.

**Tell every finder what is NOT a finding**, or you will spend the validation pass discarding the same noise every time:

- Style, naming, or formatting preference that the repo's own instructions don't state.
- "Consider adding a test / a comment / logging" with no defect behind it.
- Generated code that looks odd but is what the template produces — the fix is the model, not the file.
- Anything the change set doesn't actually touch.
- Speculation about code the finder did not read.

## 3. Read the finders' results before validating

The wave boundary is real: you cannot know what to validate until the finders report. Merge their entries, drop exact duplicates, and drop anything that plainly hits an exclusion above. What remains is the validation queue.

If the queue is empty, say so plainly and stop — do not manufacture findings to justify the review.

## 4. Validate every finding — one dispatch each, again all in one response

For each surviving finding, dispatch a sub-agent whose job is to **disprove it**, and emit them all in one response — through the same tool the finders used (§2). The brief is adversarial on purpose:

> Here is a claimed issue: `<finding>`. Read the actual code and model. Try to show it is NOT a real problem — that the case is already handled elsewhere, that the input cannot occur, that this is the generated/intended shape, or that the reviewer misread the code. Report `refuted` with the evidence, or `confirmed` with the concrete failing input/state and the observable consequence.

A finding is reported **only** if the validator confirms it **and** names a concrete failure — specific inputs or state, and what actually goes wrong. "Could be a problem" is a refutation, not a confirmation. When a finding could fail in more than one way, give its validators different lenses (does it reproduce / is it reachable / does it break a caller) rather than asking the same question twice.

## 5. Report

Report **only confirmed findings**, most severe first. Each one gets:

- **The problem in one line** — first, and it must stand alone. This is the headline the comment leads with and often all anyone reads: name what is wrong, in the reader's terms ("Declined credit checks are treated as approvals"), never what you did ("Investigated the retry path").
- **Where** — `file:line` or the designer element id. When the SAME issue is in several files, name the others alongside it rather than raising it once per file: they render as one row listing every file.
- **A severity** — `blocking` (must be fixed before this merges), `minor` (worth fixing, not blocking) or `nit` (a small improvement, take it or leave it). It leads the comment as a coloured glyph and is the findings table's first column, sharing the verdict's colours: 🔴 / 🟡 / 🔵. Grade honestly — a review where everything is blocking is one nobody reads twice.
- **What breaks** — the concrete failing input or state, and the consequence. Two or three sentences.
- **The fix** — the smallest correct change; for anything model-owned, the _model_ change, not a code patch.

**The order is verdict, then findings, then the account of the run** — the reader came for the list, so nothing goes between them.

Open with a **verdict** — `looks-good`, `minor-issues` or `changes-needed` — and one sentence saying where the change stands. The verdict follows from the severities and must not contradict them: one `blocking` finding makes it `changes-needed`, only `minor`/`nit` ones make it `minor-issues`, none at all makes it `looks-good`.

Close with the **account of how the run went about it**, as a short bulleted breakdown rather than a paragraph — a wall of prose here is the one thing a reader skips whole:

- **Coverage** — which finder passes ran and what each covered, plus the layer breakdown (files by classification, how many generated ones were suppressed, designer elements reviewed). If the change set was inferred rather than enumerated, say so here.
- **Findings** — N raised, N refuted, N confirmed.
- **Refuted** — what validation threw out, and why, a clause each.
- **Not checked** — a file you couldn't read, an area outside the change set; or say there was nothing.

### Write for the next reader, not about your own run

The reader is a developer deciding what to do next, or an agent about to fix it. Neither needs your process. Cut, from the report and from every comment you stage:

- Your reasoning and how you got there — what you read, what you suspected first. Which passes ran belongs in the **Coverage** bullet above and nowhere else.
- Anything a validator refuted, or that you considered and dropped. It is out; do not narrate it out.
- Confidence talk ("I'm fairly sure", "this may warrant investigation"), hedging, and self-assessment. A confirmed finding is stated flatly; if you cannot state it flatly, it is not confirmed.
- Restating the diff, quoting code the comment already sits on, or explaining what the changed code does.
- Praise, preamble, and closing offers of further help.

A finding is one headline plus two or three sentences. If it runs longer than that, either it is two findings or the extra text is about you.

**Change nothing.** This skill reviews. If the user wants the findings fixed, that is a separate instruction.

## 6. Pull-request mode only: stage it

Finish with **one** `stage_pull_request_review` call — **always, even at zero confirmed findings.** A counts-only summary is the record that the review ran; silence is indistinguishable from the button never having been pressed.

**Nothing you stage is posted.** The findings go into the user's own PENDING review, held on their machine: they read each one on the diff, edit or delete what they disagree with, and submit the review themselves with a verdict of their own. Write for a reader who will act on your findings, not as the last word.

- `repositoryRootPath` / `pullRequestNumber` — as the scope gave them.
- `headSha` — the range's `refB`. It is stamped into every comment's hidden marker, which is what makes a re-run safe.
- `verdict` — `looks-good`, `minor-issues` or `changes-needed`, from §5. It opens the summary, so a reader who reads nothing else still knows where the change stands. It is advisory and does **not** submit anything: the review's own verdict stays the user's to choose.
- `headline` — the one sentence from §5. No counts (they go in `summary`), no file names, no reasoning; it renders as a single line whatever you write.
- `findings` — the **confirmed** ones only, each with the `layer` from §2a, the `severity` from §5, the one-line `problem` headline, and a `body` carrying what breaks and the fix. Add `otherPaths` when the same issue is in several files — one finding listing them all, never the same finding once per file. Refuted findings must not appear; neither must anything about a `FullyManaged` file that wasn't hand-edited.
- `summary` — the §5 account of how the run went about it, as those four bullets. It renders LAST, under its own heading below the findings, so it is the part a reader chooses to read — not something they scroll past to reach the list.

The fields ARE the format — verdict first, then one sentence, then the findings, then the account — so put each part in its own field rather than writing a structured essay into `summary`.

The tool renders the severity glyph, the layer badge, the provenance footer, the marker and the summary's findings table — including shortening each path to a file name — so do not write any of them into a body yourself. It also does the two things you must not attempt by hand:

- **De-duplication.** It reads the pull request's existing threads AND what is already staged in the pending review, and skips any finding either of them already makes. Never work around this by re-wording a finding to get it staged again.
- **Folding.** A finding with no file/line to anchor to is moved into the summary rather than lost. Anchor findings to head-side lines the diff touches wherever you can; the fold is a safety net, not a plan.

Read the per-finding outcome list it returns and repeat it in chat — in particular say which findings were skipped as duplicates and which were folded. Then **tell the user to open the pull request's Conversation tab to read, edit and submit the staged review**, and stop. Do not submit anything yourself: the review is theirs to send, and the `verdict` you staged is advice for them to overrule.

## If you cannot dispatch sub-agents

Personas with neither a native sub-agent tool nor `create_sub_agent` run the same sequence inline: work the lenses one at a time, then re-check each finding against the real code before reporting it. The adversarial re-check is the part that must not be skipped — it is what makes this a review rather than a list of guesses.
