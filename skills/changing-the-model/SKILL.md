---
description: How to mutate an Intent Architect model — run_designer_script (the primary mutation path), update_application_settings, apply_change_diagram_layout, and the validate-until-clean loop with get_designer_validation_errors. Load before making any model, settings, or diagram change: covers operation ordering, containment, type/package references, associations & cardinality, mapping, and diagram layout.
short-description: Mutate the model with run_designer_script (containment, ordering, associations & cardinality, mapping), settings, diagram layout, and the validate-until-clean loop.
---

# Changing the Model

The **model is the source of truth**. Apply changes through the designer tools — `run_designer_script` is the primary mutation path. Work in **phases** for a large design: split it into cohesive scripts (e.g. one per aggregate/package/designer), each its own `run_designer_script`, and **verify the scoped `errors[]` before the next phase**. Each script is one undo step, so keep them small. A script changes the **model only** — never lay out a diagram from a script (that's a separate step, below).

Before your first script, load the scripting API once with `get_designer_script_api` (identical for every designer — reuse it) and the designer's `get_designer_schema` once per designer (see the **exploring-the-model** skill).

## `run_designer_script` — the primary mutation path

- Call `get_designer_script_api` ONCE before your first script (identical for every designer, so reuse it). Call `get_designer_schema` ONCE per designer for its rules, stereotypes, and reference types — do NOT re-fetch.
- `get_designer_schema`'s **"Element types"** block is the source of truth for CONTAINMENT: per type it lists the exact child specialization names you may `addChild`/`createElementUnder`, marks any type that REQUIRES a type reference with `!` — INCLUDING on a child reference (e.g. `Class: Attribute!`), so set a type wherever you see `!` — and `¹` for max-one children. Use those exact names; don't guess a child type or skip a required type. (Valid type-reference targets aren't enumerated — resolve them by name with `resolveType`/`setType`.)
- Don't assume package names (e.g. "Domain"): get exact names from `get_designer_schema` or list them at runtime with `getPackages()`. Name/path lookups are **EDITABLE-FIRST with a REFERENCE FALLBACK**: they match your own editable types first (no false "ambiguous" errors), and if nothing editable matches they automatically find types from REFERENCED designers/packages — so in e.g. a Services designer, `lookupByName("Order")` or `lookupByPath("Domain/Ordering/Order")` resolves the referenced Domain type with no extra flag. Those results are READ-ONLY (use them as association endpoints / operation & attribute types / mapping ends; you cannot rename or restructure them). If a type you KNOW exists comes back null, it is almost always a name/path typo — verify with `getPackages()` / the schema, not by guessing a flag.
- **A package is not an element.** Element finders (`lookupByName` / `findElements` / `lookupByPath`) search only the children of editable packages — `lookupByName("Domain")` returns `null`. Address a package via `getPackages()` or `lookupPackage(nameOrId)`.
- **Remove a stereotype owner-side:** `owner.removeStereotype(nameOrDefinitionId)` (on both element and package handles) — there is no `stereotype.remove()`.
- Console output (`log`/`warn`/`error`) is captured and returned; a thrown error comes back with source-mapped stack frames (`(main script)` for your inline `script`, else the included path) and `executed: false`.
- The result reports **`changes`** — the elements actually `created`/`updated`/`deleted` (each with `kind`, `elementId`, `name`, `specialization`). This is populated even when `executed` is false: a script mutates incrementally and a mid-script throw STILL applies the steps before it to the **in-memory** model (not yet saved to disk — see "Persisting to disk" below). So on a failure do NOT assume nothing happened — read `changes` to see what already succeeded and write the follow-up to continue from there (don't blindly re-create elements that already exist). When recording spec traceability, forward each change's `elementId` and `name`.
- The result also reports **`errors`** — validation errors/warnings on the touched elements, each addressed by `path` (the package-rooted `Package/Folder/Name` you can feed back to lookups/finders) or `associationId`. A non-empty `errors` after a "successful" run means the model is invalid — fix it before moving on.

### Operation ordering (statement order is operation order)

- Create **parents before children**.
- Create **both endpoints before creating an association** between them. ALWAYS set cardinality: `createAssociation(spec, sourceId, targetId, { targetMultiplicity: "0..*", sourceMultiplicity: "1" })` (UML strings `"1"`, `"0..1"`, `"*"`, `"0..*"`, `"1..*"`). Omitting it defaults to 1-to-1, usually wrong. Associations are **unidirectional by default** (only the target end navigable) — do NOT pass `isBidirectional`; set `isBidirectional: true` (or `assoc.setBidirectional(true)`) ONLY when two-way navigation is an explicit part of the design.
- To INSPECT existing links, `el.getAssociations()` returns one handle per connected element, each oriented AWAY from `el`: `getAssociatedElement()` is the connected neighbour, `getName()` is `el`'s name for the link, `getMultiplicity()` is how many, `getSpecialization()` is the association's kind (what `getAssociations(type)` filters on). Find a link by the connected element (`...find(a => a.getAssociatedElement()?.getName() === "Payment")`), not by guessing the end name. For the reverse name/cardinality back toward `el`, use `a.getThisElementsEnd()`. (Avoid `getOtherEnd()` — it reads as "the neighbour's end" but hops back to `el` itself.)
- Create the element **before adding stereotypes**; add a stereotype before updating its properties. If a stereotype's apply mode is `Always`, it is created/applied automatically — don't add it. Stereotype property values must be valid: `get_designer_schema` lists each as `Name: allowed` — set with `setProperty("Verb", "POST")`; `ref to <Types>` = an element id of one of those types; `(multiple)` = a JSON array of ids (e.g. `setProperty("Security Roles", [roleId])`). Off-list values fail listing the allowed values; a ref id that isn't a valid option (wrong type, or in a package this designer doesn't reference) fails naming the allowed type — resolve the element by name, and add the package reference if it lives elsewhere. Read a value back with `getProperty(name).getValue()` (an array of elements for a `(multiple)` ref), not the `.value` snapshot. A property listed as `item-list of '<Row Type>'` is NOT set with `setProperty` — call `prop.addItem()` per row and set that row handle's own properties (the schema lists them under the stereotype's `itemTypes`); where several rows match, the LAST matching row wins, so add the broad rule first. Honour a property's authored hint (the text after ` — ` on its schema line).
- You are **MODELLING, not writing C#** — don't import language idioms.
- When moving children to a new parent, move them ALL before deleting the old parent. Don't delete-and-recreate to move — update the parent reference.
- To **map** elements (e.g. a Command's fields to a Domain entity), create both endpoints and the child fields first, then map. `get_designer_schema`'s "Mappings" block says which element/association types support mapping and the mapping-type names (designer-specific — never assume a universal name like "Map to Domain"); `get_designer_element_details` shows existing mappings. Basic: `el.setBasicMapping(targetId)`. Advanced: `const m = el.createAdvancedMapping("Mapping Type Name")` then `m.mapEnd(["SourceField"], ["TargetField"])` (the mapping type is optional and the last arg — omit it; a mapping almost always has one used automatically), verifying with `m.getMappedEnds()`. Path segments take names or ids; a wrong one throws and lists what was available.

If a designer rule is violated, the operation fails — adjust the design and retry.

### Package references

- PACKAGE REFERENCES (a package depending on another package's types) are ALSO changed via a script — there is no add/remove tool. Discover candidates with `await getAvailablePackageReferences()` (or the read-only `list_available_package_references` tool), then on the owning package handle call `pkg.addReference(absolutePath, module?)` — pass a candidate's `absolutePath`, plus its `source` as `module` when `sourceType === "module"` (omit `module` for a 'solution' package). Remove with `pkg.removeReference(idOrNameOrPath)`; list current ones with `pkg.getReferences()` (or `get_designer_package_references`). Get the package handle via `lookupPackage(name)`/`getPackages()`.

### Persisting to disk — in-memory until saved, and saves are per-application

`run_designer_script` and `update_application_settings` mutate only the target application's **in-memory** model — nothing is written to disk by the call itself, no matter how "final" a successful run or a validated `errors[]` looks. The model reaches disk automatically as the first step of a `run_software_factory` run, but **only for that same application** — a Software Factory run for Application A saves only A's own dirty designers, never a different application B's.

This matters whenever one application references another's package. Application A's reference to Application B resolves against **B's saved-to-disk model**, not B's live in-memory edits. So after modelling changes in B, running Software Factory for A does **not** make A see them — the new/changed types will look missing or stale. If you've just modelled B and now need A (which references B) to pick up those changes: run Software Factory for B **first** (this saves B), then model/generate in A. Don't assume a designer's changes are visible cross-application just because its own script run reported success or clean validation.

Both `run_designer_script` and `apply_change_diagram_layout` accept an optional `saveOnSuccess` flag to persist that one designer to disk immediately, **without** running the Software Factory — useful for pure-modelling work (e.g. spec/traceability recording, documentation-only modelling) where no generation step is needed yet. Only set it `true` on the **final** call of an already-verified sequence: the save happens unattended and does **not** gate on validation errors, so saving mid-fix would persist a half-corrected model.

### Reusing saved application scripts (`includedScriptPaths`)

- Call `get_scripts` first to discover available scripts; use the returned `path` values directly. Each entry is an absolute path or a path relative to the `.application.config` file. Scripts are prepended in order — a library loaded before your main body. To run a saved script with no extra logic, pass its `path` in `includedScriptPaths` and an empty string for `script`.

### Comments

Add comments to elements for non-obvious design decisions — express the purpose of the element and/or its expected behaviour if it represents an operation or function. This is passed on to coding agents that implement the logic.

## Verify — confirm it landed, then validate until clean

After every mutation confirm **both**, and don't proceed while either fails:

1. **It landed as intended.** Reconcile the returned `changes[]` against your plan: each new element appears **once** as `created`, edits show as `updated` (a stray `created` where you meant to modify is a **duplicate** — e.g. a second `FirstName` → `FirstName1`). When a mutation could have created-instead-of-updated, or you retried a throwing script (pre-throw steps already committed), re-read the container with `get_designer_model_structure` / `get_designer_element_details` to confirm.
2. **It's error-free.** Read the scoped `errors[]` after every script; broaden with `get_designer_validation_errors` after a larger change set. Resolve and re-check until clear.

`addChild` / `createElementUnder` / `createElement` never dedupe. To _ensure an element exists_, look it up first (`lookupByName` / `findElements` / `el.getChildren()`) and create only when absent.

## Diagram layout — a SEPARATE step, never in a script

`run_designer_script` places nothing on a diagram, and you must **never lay out from a script** (no `layoutVisuals`, no positioning visuals in-script). Lay out **after a designer's mutation/verify phases — and before moving on to the next designer** — with `apply_change_diagram_layout`, and **you own the result: it must be tidy with NO overlapping elements.**

**Interleave layout per designer, don't defer it.** When a design spans multiple designers, finish the apply → verify → lay-out loop for **one designer** before you start modelling the next.

- **Read first.** Call `get_designer_diagram_snapshot` to see the diagram's current nodes (each with x/y/width/height) and edges. Also honour any layout rules on the diagram's type from `get_designer_schema`.
- **Add the new elements.** Address each node by its **id, name, or package-rooted PATH** (e.g. `Domain/Orders/Order`) — whatever a tool result gave you; never invent a GUID (an ambiguous name errors with the candidate paths). Give clear `x, y, width, height`. Place a new element next to its most-connected related element. Some nodes `autoSized` — their width/height is computed, so position by their actual size. Only add elements this diagram type can actually represent — an element whose type has no visual configuration here (e.g. a DTO on a Domain diagram) fails the call with an explanatory error instead of silently being skipped.
- **Route the associations** you created so their lines render. Address edges by `associationId` (from `get_designer_diagram_snapshot`, `get_designer_model_structure`, or `get_designer_element_details`). An edge routes only when **both** its endpoint nodes are placed in the same call — add both endpoints as nodes alongside the edge; an edge to an element you didn't place won't appear. Routing is automatic — don't set waypoints unless asked.
- **Spacing & stability:** keep ≥150px horizontal and vertical gaps; nodes must never overlap; don't move existing elements unnecessarily — only include a node if you intend to move or resize it. Moving an existing element (to make room / clear an overlap) is exactly what this tool is for.
- The result reports final geometry and any overlaps, each with a collision-checked move to clear it. **Re-call applying those moves until no overlaps remain**, then verify a clean layout.

## `update_application_settings`

- Changes the per-application settings shown in the Solution Explorer "Settings…" dialog. Call `get_application_settings` first to discover current values and the exact `id`s of module settings before patching.
- **Partial patch** — only the parameters you supply change; everything else is untouched.
- Changing `name` ALSO renames the on-disk folder and metadata files (the solution is modified).
- `moduleSettings` is a flat list of `{ settingId, value }`; each `settingId` must match an `id` returned by `get_application_settings` (the tool resolves the group). Stringly-typed values: checkboxes/switches expect `"true"`/`"false"`; selects must use an option `value`; numbers are numeric strings.
- `persistenceFormatVersion` ∈ `V1Xml` (legacy), `V2Xml`, `V3Xml` (nested associations), `V3Yaml`. `iconType` ∈ `UrlImagePath`, `RelativeImagePath`, `FontAwesome`, `CharacterBox`.
