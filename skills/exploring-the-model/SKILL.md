---
description: How to inspect an Intent Architect designer model — get_applications, get_designer_schema, find_designer_elements, get_designer_model_structure, get_designer_element_details, get_application_settings. Load before exploring or discovering any part of the model, and to read the shared element/association notation the read tools emit.
short-description: Inspect the model (schemas, structure, elements, settings) and the read-tool notation.
---

# Exploring the Model

The **model is the source of truth**; the codebase is a generated artifact — never infer model structure, naming, or hierarchy from files on disk. Discover the relevant part of the model thoroughly before you design or answer: keep inspecting until you would make no assumptions.

## Discovery order

1. **Locate the application & designers.** The workspace context you were given already lists the solution's applications, their designers, output locations, and codebase roots — start there, and call `get_applications` when it was elided or you need the full list. Each designer has an id you pass to the read tools.
2. **`get_designer_schema` ONCE per designer** — load its rules, settings, stereotypes, reference types, current diagram, and package list. The schema does **not** change as you edit elements — reuse the result; do not re-fetch after mutating.
3. **Find elements** — prefer `find_designer_elements` (regex + specialization filter) when you know what you're after. Use `get_designer_model_structure` only when you genuinely need topology ("show me all packages and their classes"); always scope it with `specializations` or `packageId`.
4. **`get_designer_element_details`** on a specific element before referencing or modifying it, to confirm its properties, type references, stereotypes, mappings, and parent constraints. Its `elementPath` argument takes a `Folder/Name` PATH, a bare NAME, or an id.

## The read-tool notation (shared legend)

`get_designer_model_structure` and `get_designer_element_details` render elements and associations in one shared notation:

```
Shared notation for model elements and associations:
- Element line: `Name [Type] (memberCounts) +Stereotype(prop=value) — comment [#id]`. `[Type]` is the specialization; a trailing `/` on the name marks a container (folder / an element with members); `(3 Attribute, 1 Operation)`, when present, COUNTS child members that are not expanded here; `+Stereotype(prop=value)` lists APPLIED stereotypes showing only set / non-default property values (`+Name` alone when it has none); `— comment` is the description; a trailing `#id` appears ONLY where you need it to act (see the tool's notes).
- Association line: `Source →"navName" Target (srcMult→tgtMult) [Type] #id`. `→` reads source→target (`↔` = bidirectional, with the back-navigation name after `/`); `"navName"` is the Source's name for the link; `Source`/`Target` are the element's bare NAME, or its full `Package/Folder/Name` PATH when that name is duplicated or cross-package (address either verbatim via lookupByPath / get_designer_element_details); `(srcMult→tgtMult)` are the multiplicities; `[Type]` is shown only for non-default association types; the trailing `#id` is the association's id — an association has NO name/path address, so this is its only handle (and the `associationId` apply_change_diagram_layout needs to route the edge). In a script `Source.getAssociations()` returns this link — `getName()` = navName, `getElement()` = Target (the connected element, NOT Source), `getMultiplicity()` = the target multiplicity.
```

## Per-tool notes

### `get_designer_schema`

- Returns the designer's schema only — rules, settings, stereotypes, reference types, current diagram, package list (no elements, associations, or errors).
- A reference type shown with a `<...>` signature is GENERIC and needs that many type arguments (e.g. `Dictionary<TKey, TValue>` → `setType("Dictionary<string, Order>")`); a plain name takes none.
- Stereotype definitions list `applicableTo` and, when present, `applicableToReferencedTypes` — or `appliesTo` (a filter-function rule resolved to the element types it accepts) or `applicability` (`any element`, or `dynamic` when a per-element filter function decides and we could not resolve it; read the description for where it applies); apply one by its `id` (names can collide). `availableIn` appears only when a stereotype is limited to certain packages. A stereotype applies at most ONCE per element unless it carries `allowMultipleApplies: true`.
- A property line's text after ` — ` is the module author's own **hint** — what the value means, what a path is relative to, how it is spelt. It is authoritative and frequently un-guessable; read it before setting the value rather than inferring from the property name.
- A property shown as `item-list of '<Row Type>'` is a list of ROWS, not a value: `setProperty` on it stores labels and no rows, and reading its value gives you that stale label cache. Add rows with `prop.addItem()` and set the returned handle's own properties — which are listed under the stereotype's `itemTypes`, keyed by row type name — and read them back with `prop.getItems()`.
- Applied stereotypes on elements are NOT here — use `get_designer_element_details` for per-element values. Check validation with `get_designer_validation_errors`, not by re-fetching the schema.

### `find_designer_elements` (preferred for targeted lookups)

- Search elements by any text field with a **case-insensitive regex** (`query`). `fields` restricts the search: `name`, `specialization`, `comment`, `value`, `typeReference`, `stereotype` (all stereotype names + property values); omit it for all fields.
- `specializations` filters to matching element types (e.g. `["Class"]`, `["Repository"]`); omit it for all types. Searches every element (top-level, children, package references).
- Each result carries `matchedOn` (which fields matched) and a **`path`** — its package-rooted `Package/Folder/Name` address. **Prefer passing this `path`** (not the bare name) as `get_designer_element_details`'s `elementPath`, and to scripts — it's unambiguous, so duplicate names never collide. Bare name is safe only when exactly one match was returned.
- Results cap at 100 — if hit, narrow the regex, restrict `fields`, or add `specializations`.

### `get_designer_model_structure` (topology only)

- Terse indented TEXT tree per package (packages → folders → elements) plus a compact association list. Stereotype DEFINITIONS/rules/settings are NOT here (that's `get_designer_schema`) — but each element's APPLIED stereotypes are shown inline.
- The overview carries NO element ids — address an element by its NAME or PATH (the indentation IS the path, e.g. `Catalog/Product`). `(memberCounts)` advertise child members NOT shown — call `get_designer_element_details` for those.
- A line shown with `@ Package/Folder/Name` has a DUPLICATED name — address it by THAT exact path; names without an `@` qualifier are unique, so the bare name is safe.
- `totalElements` is the whole model's count; the tree shows only top-level nodes. `truncated: true` (with `shownNodes`) appears only when the element cap was hit — narrow with `packageId`/`specializations` or use `find_designer_elements`. Defaults are slim (no comments, cap 200, hard cap 500).

### `get_designer_element_details`

- Reads ONE element in full — pass its `Folder/Name` PATH, its exact NAME, or its id, all as the single `elementPath` argument. Returns the element plus a terse `members` tree, applied `stereotypes` (current values), `associations`, any `mapping`/`mappings`, and `suggestions`/`codeFiles`/`errors` when present.
- A PACKAGE name resolves here too (you get its applied stereotypes; the element-only surfaces — members tree, associations, mappings — are omitted).
- `members` are children indented by containment (each with a trailing `#id`); a member's own associations and stereotype values are not expanded — call the tool again on that member to drill in.
- Existing mappings: `mapping` is a BASIC projection (`→ targetPath [Mapping Setting]`); `mappings` are ADVANCED — a `[Mapping Type]` header then one indented `sourcePath → targetPath [= expression]` per mapped end (name-based paths). An association's advanced mappings show in the same notation beneath its `associations` line.
- `stereotypes` shows applied stereotypes with CURRENT values only — the allowed values/types per property live in `get_designer_schema`. Change one via `run_designer_script`: `el.getStereotype("Name").setProperty("prop", value)`.

### `get_application_settings`

- Reads per-application settings + architecture configuration. Returns basic fields (`name`, `description`, `metadataLocation`, `outputLocation`) and dynamic `moduleSettingGroups` from installed modules.
- `outputLocation` is the resolved absolute on-disk path for generated code; `metadataLocation` the resolved absolute metadata path — use these directly when you need an absolute path; don't guess the solution root.
- Every module setting has a unique `id` (per application); pass that exact id to `update_application_settings`. `controlType` says how the value is read: `Checkbox`/`Switch` → `"true"`/`"false"`; `Select`/`MultiSelect` → one of `options[].value`; `Number` → numeric string; the rest are free text.

## Designer Quick Reference

- **User Interface designer** — Pages, Components, Layouts.
- **Services designer** — Commands, Queries, DTOs, Services, Operations (CQRS / API surface).
- **Domain designer** — Entities, Value Objects, Aggregates, Repositories.
- **Codebase Structure designer** — Folder/project layout and template output anchors.

Folder names in a designer map to namespaces or output paths — they may not match disk folders. Trust the designer.
