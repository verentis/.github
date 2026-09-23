---
description: How to add an application to the already-open Intent Architect solution with create_application. Load before adding an application. High-stakes: confirm the architecture, the optional components, and the name with the user first.
short-description: Add an application to the already-open solution.
---

# Creating Applications

Adding an application is **high-stakes and hard to reverse** — always confirm the **architecture**, the **optional components**, and the **name** with the user before executing (never invent a name).

## `create_application` — add to the open solution

Use to add an application to the ALREADY-OPEN solution.

**Ask in TWO `ask_user_question` calls, never three.** Only the component and setting choices depend on anything (`get_architecture_details`, which needs the architecture); bundle everything else into the first call.

1. **Call 1 — target, architecture and name, bundled.** If the user has NOT specified an architecture, call `search_architectures` and **present the matches — do NOT pick one silently**; omit the `query` to list everything, and if a query returns nothing, list them all rather than guessing at another search term. Put the architecture, the application **name** (never invent one), and the target application if that's ambiguous, into a SINGLE `ask_user_question` call — the UI auto-appends a free-text "Other" option, so a typed-in name needs no call of its own.
2. **Call 2 — components & settings.** Call `get_architecture_details` for the chosen architecture, then **always offer the optional choices** in ONE further call (never silently accept only the defaults):
   - **Optional components** — the components returned with `required: false` **and** `includedByDefault: false`. **You MUST present these** and ask which, if any, to add — a single `multiSelect` question listing each optional component (with its description). Do NOT skip this question just because the defaults would produce a working app.
   - **Optional modules** — within otherwise-included components, surface any modules marked `default-off` the same way when they're relevant to the user's request.
   - **Non-default settings** — any additional settings surfaced (e.g. target framework); confirm them or let the user change them, and flag any default that looks wrong for this solution rather than passing it through silently.

   Pass the user's picks through as `componentOverrides` / `settingOverrides`. Skip this Q&A **only** when the user's initial request explicitly opted out of questions (e.g. "just give me everything, don't ask") — then take the recommended/default set without prompting.
3. **Create.** Call `create_application` with the confirmed parameters. It installs the modules and returns once they are installed and the Software Factory has **started** — it does NOT wait for the run to finish. Call `run_software_factory` to wait until the SF reaches staging; its generated files are **staged** (not on disk until applied) — inspect with `get_file_diffs` and write to disk with `apply_staged_file_changes` (see the **generating-and-applying-code** skill).
4. **Finish the application — do not stop at the scaffold.** The architecture gives you projects, configuration and infrastructure, not an entrypoint or any behaviour. Deliver whatever the user's words imply on top of it (an entrypoint, a first working endpoint, a hosted service) in the **same turn**, dispatching a `coding` sub-agent when it is more than trivial. Never end the turn *offering* to do it.
5. **Build before you report.** Run the configured build task with `run_task` and report the outcome; never claim an application runs off the back of a successful file write.

Skip a confirmation step only when the user has already supplied that detail explicitly (e.g. they named the architecture, so there's no need to ask which one), or — for components/settings — when the user's initial request said to skip questions entirely.
