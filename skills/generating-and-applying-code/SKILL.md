---
description: How to run the Software Factory and get its generated code onto disk — run_software_factory → get_file_diffs → apply_staged_file_changes, including staging vs write-through and the destructive-change branch. Load after model changes are verified and you need to generate and apply code.
short-description: Run the Software Factory, inspect diffs, apply staged changes; the destructive-change branch.
---

# Generating and Applying Code

After model changes are verified and the requirement is met, run the Software Factory (SF) to generate the boilerplate and architectural code. The SF **stages** its output for review — it is NOT on disk until you apply it.

The flow: **`run_software_factory` → inspect with `get_file_diffs` → `apply_staged_file_changes`.**

## The Software Factory implements NO business logic — assume nothing, dispatch `coding`

**Never assume the Software Factory implements ANY business logic.** It typically generates **contracts and infrastructure only** — interfaces, DTOs, entity/property scaffolding, WebApi/controller wiring, DI registrations, EF configuration, and method **signatures** with empty or `NotImplementedException`/`TODO` bodies. It does **not** automatically write operation bodies, command/query handlers, domain method bodies, event handlers, mapping logic, validation logic, UI views, or any behaviour. Treat every generated business logic method body as a stub until you have read it and proven otherwise.

Dispatch `coding` sub-agents to implement business logic, mappings, UI components, tests, etc.

## 1. `run_software_factory`

- Run it for an application **after** making model changes. It returns when the SF reaches staging (changes calculated and visible in the UI) or when it completes/fails before staging.
- **Do NOT call it repeatedly in a loop** to retry the same unresolved error — investigate and fix the underlying (usually modelling) issue first; never patch generated code to make it compile, that masks the error and is overwritten on regeneration. This does **not** mean never re-run: any model/code change made after a run, including one made from `coding` feedback, invalidates it — run it again.
- The response includes a `changes` list (relative path + change type, no file contents) and an `outputBasePath`. Get a change's absolute path by combining `outputBasePath` with the change's `relativePath`.
- **Staging vs write-through.** By default changes are **staged** (not on disk until applied). When **write-through checkpoints** are enabled the SF writes changes DIRECTLY to the codebase — the response says so and the listed changes are ALREADY on disk; do NOT call `apply_staged_file_changes` in that case, just build/test against the updated code.

### Destructive changes

Each change carries a `destructive` tri-state: `"yes"` = a file was DELETED or hand-written/customised code was OVERWRITTEN; `"unknown"` = could NOT be verified as safe (no previous template output to compare); `"no"` = safe (absent when no check was performed). **Do NOT assume `"yes"`/`"unknown"` are safe.** Whenever any are present, load the **`resolve-destructive-changes`** skill and follow its playbook — inspect each as a diff with `get_file_diffs`, decide whether the loss is intended, and prefer reproducing unintended losses by MODELLING them (code-management directives only for genuinely bespoke code). In staging mode resolve them BEFORE you apply; in write-through mode they are already on disk, so revert unwanted ones first.

## 2. `get_file_diffs`

- Shows what the SF did to one or more files as a Git-style unified diff against the file's PRE-RUN content. Works in BOTH modes:
  - **Staging mode** — staged content (not yet on disk) vs the file on disk.
  - **Write-through mode** — file on disk vs the shadow-git checkpoint taken just before the most recent run (i.e. exactly what that run changed).
- This is the tool to reach for when a change came back `destructive: "yes"`/`"unknown"` — read the diff to see exactly what was deleted/overwritten, then follow the `resolve-destructive-changes` skill.
- Provide ABSOLUTE paths inside an application's output directory. A file with no change yields an empty diff (`status: unchanged`). The aggregated result is a readable multi-file unified diff; per-file metadata (`status`: added / modified / deleted / unchanged) is also returned for programmatic checks.

## 3. `apply_staged_file_changes`

- Writes the given SF staged changes to disk so build and test tools see the latest code. Call it (in staging mode) before running a build/test task, or when a `read_file` result notes staged content and you need an on-disk view. In write-through mode there is nothing to apply.
- **Pass the exact relative paths you reviewed** (from `run_software_factory`'s `changes` list or `get_file_diffs`) via `relativePaths` — required and non-empty. It applies ONLY those files; anything else still pending is left untouched. Never guess or assume it applies "everything pending".

## Stop conditions

The task is complete only when **all** hold:

- The requested capability is represented in the appropriate designer(s), with no validation errors (see the **changing-the-model** skill).
- The Software Factory has been run and its changes applied successfully.
- The codebase compiles and existing tests pass.
- Required bespoke logic is in place — no `NotImplementedException`, `TODO`, or stubs remain in new files.
- **A fresh `run_software_factory` re-run, taken right before declaring done, proposes zero changes to the bespoke code** (via `get_file_diffs` or an empty `changes` list) — an earlier run in the conversation doesn't count, especially after edits made from `coding` feedback. If it proposes changes, protection is missing/wrong — fix it (model it, or add a code-management ignore directive) and repeat until clean. Ignore any diff that is purely the SF reordering or removing C# `using` declarations (e.g. one becomes implicitly unnecessary) — that is normal housekeeping, not a destructive change or a sign that protection is missing.
