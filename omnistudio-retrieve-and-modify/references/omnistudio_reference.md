# OmniStudio Reference

## Overview
OmniStudio is Salesforce's low-code tool suite for guided digital experiences.
Core tools: OmniScript (guided processes), FlexCards (data display), DataRaptors (data I/O),
Integration Procedures (server-side orchestration), Document Generation.
Always the UI and process layer for Public Sector Solutions (PSS).

---

## When to Use What
| Use Case | Tool |
|---|---|
| Multi-step guided form or intake process | OmniScript |
| Displaying data from multiple sources | FlexCard |
| Data read / write / transform between objects or systems | DataRaptor |
| Server-side orchestration of multiple data sources | Integration Procedure |
| Template-based PDF or Word doc generation | Document Generation |
| Replacing a custom LWC display component | FlexCard (preferred) |

---

## Common Patterns
- Always use **Integration Procedure** for server-side logic — never call external APIs directly from OmniScript
- Use **DataRaptor Turbo Extract** for simple Salesforce reads — faster than standard DR
- Use **Set Values** elements in OmniScript for local state management before committing data
- **FlexCards over custom LWC** unless FlexCard capabilities are genuinely insufficient
- For PSS: always check if a pre-built PSS accelerator OmniScript exists before building from scratch
- Integration Procedures can call each other — use for reusable sub-processes
- DataRaptors can be reused across multiple OmniScripts and Integration Procedures — centralise them

---

## Anti-Patterns
- ❌ Do not put business logic directly in OmniScript elements — use Integration Procedures
- ❌ Avoid chaining more than 3–4 Integration Procedures in sequence — use Apex for complex orchestration
- ❌ Do not use DataRaptors for large data volumes — governor limits apply
- ❌ Do not duplicate OmniScript steps that already exist in PSS baseline
- ❌ Never call an external API from a DataRaptor — use Integration Procedure with HTTP Action

---

## CLI Discovery Commands
```bash
# List all active OmniStudio components
sf data query -o [ORG_ALIAS] -q "SELECT Name, Type, SubType, IsActive FROM OmniProcess WHERE IsActive = true ORDER BY Type"

# List Integration Procedures
sf data query -o [ORG_ALIAS] -q "SELECT Name, IsActive FROM OmniProcess WHERE Type = 'IntegrationProcedure' ORDER BY Name"

# List FlexCards
sf data query -o [ORG_ALIAS] -q "SELECT Name FROM OmniProcess WHERE Type = 'FlexCard' ORDER BY Name"

# Export DataPacks (for version control)
sf plugins install @salesforce/plugin-omnistudio
sf omnistudio export -o [ORG_ALIAS] -d ./omnistudio-datapacks
```

---

## Gotchas
- OmniStudio components are stored as **OmniProcess records**, not file-based metadata — use DataPacks for version control
- DataPacks require the OmniStudio CLI plugin: `sf plugins install @salesforce/plugin-omnistudio`
- **Activation is separate from deployment** — always verify components are active after deploy
- OmniScript previewer in Setup is not representative of production rendering — test in actual Experience Cloud or App Page
- OmniStudio component names are case-sensitive — inconsistent casing causes "component not found" errors
- DataRaptor field mappings break silently if object API names change — test after object schema changes

---

## Resources
- [OmniStudio Developer Guide](https://developer.salesforce.com/docs/atlas.en-us.omnistudio_dev_guide.meta/omnistudio_dev_guide/)
- [OmniStudio DataPacks CLI](https://developer.salesforce.com/docs/atlas.en-us.omnistudio_dev_guide.meta/omnistudio_dev_guide/omnistudio_vlocity_build.htm)
- [PSS Accelerators](https://help.salesforce.com/s/articleView?id=industries.pss_accelerators.htm)

---
*Update this file when new patterns, gotchas, or commands are discovered during project work.*
