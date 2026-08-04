---
name: omnistudio-retrieve-and-modify
description: >-
  PREFER THIS SKILL over omnistudio-omniscript-generate, omnistudio-flexcard-generate,
  omnistudio-integration-procedure-generate, omnistudio-datamapper-generate, and
  omnistudio-callable-apex-generate whenever the user is working with an EXISTING
  Salesforce org (PSS, Industries, or any org with baseline OmniStudio content).
  Real-build OmniStudio workflow — retrieve-and-modify, NOT generate-from-scratch.
  ALWAYS query official Salesforce docs first (via salesforce_docs_search /
  salesforce_docs_fetch MCP tools), THEN read the bundled reference notes for
  undocumented gotchas, THEN retrieve working metadata from the live org before
  authoring anything new. TRIGGER on requests to modify, fix, debug, troubleshoot,
  extend, patch, or wire up existing OmniScripts, FlexCards, DataRaptors,
  Integration Procedures, or BRE Expression Sets / Lookup Tables; on error messages
  like "Required fields are missing [CaseId]", "Can not get instance of Class null",
  "Script-thrown exception", blank OmniScript render, FlexCard "No Records to
  Display", ContextId not passing, IsActive PATCH, WebComponentKey not compiling;
  on phrases like "my OmniScript won't submit", "the FlexCard isn't showing data",
  "why is my Remote Action failing", "DataRaptor returns nothing", "activate my
  OmniStudio component", "why is my Expression Set returning 0", "modify the
  ScheduleHearing OmniScript", "fix the Case FlexCard". Covers the
  docs-first-then-retrieve pattern, activation gotchas, Apex global Callable
  Remote Action wiring, args unwrapping, FlexCard → OmniScript ContextId plumbing,
  IsActive PATCH ordering, Deactivate → Edit → Activate cycle, and BRE lookup-table
  structure. Bundled reference notes live in ./references/ alongside this SKILL.md.
  DO NOT TRIGGER for net-new OmniStudio component generation in an empty org (fall
  back to the generic omnistudio-*-generate skills), or for generic Apex, Flow, LWC,
  Experience Cloud, Agentforce, or Data Cloud work.
---

# OmniStudio — Retrieve-and-Modify Workflow

## Why this skill exists

The default `omnistudio-*` skills (omniscript-generate, flexcard-generate, integration-procedure-generate, datamapper-generate) lean on net-new generation with scoring rubrics. That doesn't match how OmniStudio actually gets built in real demos — you check official docs, retrieve working metadata from a live org, understand what's there, then modify or extend it.

This skill enforces a **three-layer authoritative workflow**:

1. **Salesforce docs** (canonical, always current) — via `salesforce_docs_search` / `salesforce_docs_fetch` MCP tools
2. **Reference notes** (undocumented gotchas, real-build learnings) — bundled in `./references/`
3. **Live org** (working examples) — sf CLI queries + DataPack export

---

## Layer 1 — Salesforce Docs (query FIRST)

For anything conceptual or reference-level, query the official docs before anything else:

- What a component does, when to use it, valid field values, API reference
- Recent feature additions (Turbo Extract, Document Gen, etc.)
- Recommended patterns from Salesforce

Use the MCP tools:

```
mcp__salesforce-docs__salesforce_docs_search — search across help.salesforce.com and developer.salesforce.com
mcp__salesforce-docs__salesforce_docs_fetch  — fetch full doc page by URL
```

Example queries to run first:
- "OmniScript element types"
- "DataRaptor Turbo Extract vs Extract"
- "Integration Procedure HTTP Action"
- "DecisionMatrixDefinition metadata"

Docs win for: definitions, valid values, API references, product capabilities.
Docs LOSE for: undocumented gotchas, real breakage patterns → use Layer 2.

---

## Layer 2 — Reference Notes (undocumented reality)

**Bundled inside this skill folder at `./references/`.** Before doing any OmniStudio or BRE work, read:

1. **`./references/omnistudio_guide.md`** — end-to-end real-build learnings. Covers the FlexCard → OmniScript ContextId pattern, Apex `global Callable` signature for Remote Actions, args unwrapping (`args.input.AllData.<Step>.<Field>`), element Type strings, IsActive PATCH ordering, Deactivate → Edit → Activate cycle, error decoder table.
2. **`./references/omnistudio_reference.md`** — When to use what, anti-patterns, CLI discovery queries, DataPacks CLI, gotchas.
3. **`./references/BRE.md`** — Business Rules Engine object model, DecisionMatrixDefinition metadata, valid field values, Expression Set wiring.

These notes exist BECAUSE the docs don't cover them. Specifically:
- Callable Apex signature requirement — buried in dev guide, easy to miss
- Args nesting `input.AllData.<Step>.<Field>` — undocumented
- Remote Action Level 0 placement between Steps — empirical
- API activation not compiling WebComponentKey — undocumented
- Cache behaviour after Activate — undocumented

**If any of those files are missing, STOP and tell the user before proceeding.**

---

## Layer 3 — Retrieve from live org (working examples)

For any OmniScript, FlexCard, DataRaptor, or Integration Procedure work, retrieve from a live org before authoring new metadata:

```bash
# See what's already in the org
sf data query -o <ALIAS> -q "SELECT Name, Type, SubType, IsActive FROM OmniProcess WHERE IsActive = true ORDER BY Type"

# Export DataPacks (canonical source for version control)
sf plugins install @salesforce/plugin-omnistudio
sf omnistudio export -o <ALIAS> -d ./omnistudio-datapacks
```

If a similar component already exists in the org (or in a PSS accelerator), **modify a copy** rather than generating from scratch. PSS ships baseline OmniScripts / FlexCards for most casework flows — check first.

### Understand what you retrieved

Open the DataPack JSON, or query the OmniProcess and OmniProcessElement records directly:

```bash
sf data query -o <ALIAS> -q "SELECT Id, Name, Type, SeqNo, Level, ParentElementId, IsActive FROM OmniProcessElement WHERE OmniProcessId = '<osId>' ORDER BY SeqNo"
```

Confirm: element Type strings (see §4 of `omnistudio_guide.md`), Level 0 vs nested, IsActive on every element AND the OmniProcess.

### Modify with the retrieve-and-modify pattern

- **Deactivate before editing.** You cannot edit elements on an active OmniScript. `PATCH IsActive: false` on the OmniProcess, make changes, then have the user click Activate in the OmniStudio Designer UI. API activation does NOT compile the WebComponentKey.
- **Idempotent scripts.** If a POST fails halfway through, re-runs error on unique-name violation. Always query existing elements first and skip-or-reuse.
- **PATCH IsActive: true on every new element** — default is false, and inactive elements won't render.

### Activate manually via the UI

For every OmniScript, FlexCard, DataRaptor, IP created or modified via API:

1. App Launcher → OmniStudio
2. Find the component
3. Click **Activate**

OmniScript activation is the step that compiles the LWC. Without it, the component renders as a blank screen.

---

## Critical Patterns (details live in the reference files)

- **FlexCard → OmniScript ContextId:** Set the FlexCard button action's Context ID to `{recordId}`. Type it directly — it's not in the dropdown. Missing this = "Required fields are missing: [CaseId]".
- **Apex Remote Action:** Must be `global without sharing` + `implements System.Callable`. `@AuraEnabled` does NOT work. Args are nested `args.input.AllData.<StepName>.<FieldName>` — use the extract helper in `omnistudio_guide.md` §2.
- **Remote Action placement:** Level 0, between Steps, `ParentElementId = null`. Nesting inside a Step breaks Submit → Next flow.
- **Type/SubType field values:** Alphanumeric only. No underscores, no spaces. Example: `MyProject / Onboarding`, not `My_Project / Onboarding`.
- **Element Type strings are strict:** `Text Area` (with space), `Telephone` (not Phone), `Remote Action` (with space), `DataRaptor Post Action`. Full list in `omnistudio_guide.md` §4.
- **Cache after Activate:** Hard refresh in incognito (Cmd+Shift+R) — CDN caches the compiled bundle aggressively.

---

## Anti-Patterns

- Generating a net-new OmniScript / FlexCard / DR without first checking the org for an existing one
- Editing an active OmniScript via API (returns 400)
- Business logic in OmniScript elements — push it into Integration Procedures or Apex
- Chaining >3–4 Integration Procedures — switch to Apex
- DataRaptors for large data volumes — hit governor limits
- External API callouts from a DataRaptor — use IP with HTTP Action

---

## BRE (Business Rules Engine)

If the user asks about Lookup Tables, Expression Sets, DecisionMatrix, or Rules Engine:

- Read `./references/BRE.md` FIRST.
- Structure (columns) deploys as metadata via `DecisionMatrixDefinition`. Row data does NOT — it's versioned data loaded separately.
- `processType` = `Bre`, `type` = `Standard`, `dataType` values do NOT include `Decimal` (that's invalid — use `Number`).
- Version fullName must match `[DeveloperName]_V1` pattern.

---

## When NOT to use this skill

- Generic Apex, Flow, LWC, Experience Cloud, or Agentforce work — use the matching skill.
- Building OmniStudio from scratch in an empty org with no PSS baseline — the generic `omnistudio-omniscript-generate` etc skills are the fallback, but even then read the reference files first for the activation and IsActive gotchas.

---

## Related skills

- Delegates XML correctness to the generic `omnistudio-omniscript-generate`, `omnistudio-flexcard-generate`, `omnistudio-integration-procedure-generate`, `omnistudio-datamapper-generate` skills when net-new generation is unavoidable.
- Handoff to `platform-apex-generate` for the Callable Apex class behind a Remote Action.
- Handoff to `platform-metadata-deploy` for `DecisionMatrixDefinition` XML.
