---
name: salesforce-pss-icm
description: >-
  Salesforce Public Sector Solutions (PSS) Investigative Case Management (ICM)
  reference and demo-build guide. Use this skill whenever the user is building,
  extending, or troubleshooting a PSS ICM demo — courts, tribunals, complaint
  intake, case proceedings, evidence and custody chain, regulatory violations,
  care plans, or citizen portals on Case / PublicComplaint / CaseProceeding /
  CustodyItem / RegulatoryCodeViolation. TRIGGER on requests to build a courts
  or tribunal demo, model case proceedings and hearings, wire up complaint
  intake, capture evidence with chain of custody, generate orders and
  benchsheets, build a citizen or lawyer portal on PSS Case, deploy a Justice
  Case Centric or Participant Centric view, or configure Actionable Relationship
  Centre (ARC) for a PSS org. Bundled reference notes cover PSS ICM data model
  (24 objects), OmniStudio component inventory typical to a baseline PSS org,
  ICM-specific OmniScript / IP / DataRaptor / FlexCard patterns, permission set
  group architecture, record types for courts vs tribunals, and OmniScript on
  LWR Experience Cloud gotchas (Core standard runtime, WebComponentKey null,
  IsWebCompEnabled, Experience Builder Publish button). DO NOT TRIGGER for
  generic PSS work outside courts / tribunals / investigations, for provider
  portals on PSS + Aura (use salesforce-pss-provider-portal), for generic
  OmniStudio work (use omnistudio-retrieve-and-modify), or for non-PSS
  Salesforce work.
---

# Salesforce PSS ICM — Investigative Case Management Skill

## What this skill is for

Salesforce Public Sector Solutions ships a **Court & Investigative Case Management (ICM)** data model and a baseline OmniStudio inventory. This skill captures the reusable knowledge for building a PSS ICM demo — data model, component inventory, build patterns, and gotchas — so Claude doesn't re-research it every time.

Sister skills for related domains:
- `salesforce-pss-provider-portal` — NGO / provider self-onboarding on PSS + Aura + OmniScript
- `omnistudio-retrieve-and-modify` — generic OmniStudio retrieve-first workflow
- `salesforce-experience-cloud-portal` — LWR vs Aura decision and Experience Cloud gotchas

## When to use this skill

Trigger on any of these:
- "Build a courts / tribunal demo on PSS"
- "Model case proceedings / hearings"
- "Wire up complaint intake OmniScript"
- "Capture evidence with chain of custody"
- "Generate orders from a proceeding"
- "Build a Justice Case Centric view / Participant Centric view"
- "Configure ARC (Actionable Relationship Centre) for PSS Cases"
- "What's the PSS ICM data model?"
- "How do I link a Supreme Court appeal to a tribunal matter?"
- Any request naming PSS objects: `PublicComplaint`, `ComplaintCase`, `CaseProceeding`, `CustodyItem`, `RegulatoryCodeViolation`, `CarePlan`, `CaseParticipant`

## How to use this skill

1. **Read the bundled reference first** — `./references/pss-icm-reference.md` covers the full data model, existing org inventory patterns, OmniScript / IP / DataRaptor / FlexCard build patterns, permission architecture, LWR gotchas, and data-creation patterns for CustodyItem, RegulatoryCodeViolation, etc.
2. **Then retrieve the target org** — a real PSS ICM org already has 100+ OmniScripts, 160+ IPs, dozens of DataRaptors and FlexCards. Query first, don't generate from scratch.
3. **Follow the retrieve-and-modify pattern** — see `omnistudio-retrieve-and-modify` for the docs → notes → retrieve workflow.

## Key patterns captured in the reference

- **Data model (24 objects)** — `PublicComplaint` → `ComplaintCase` → `Case` → `CaseProceeding` → `CaseProceedingResult` / `CaseProceedingInfraction` → `RegulatoryCodeViolation` → `ViolationEnforcementAction`; `CustodyItem` → `CustodyChainEntry` + `CustodyItemRelation` + `CustodyItemRgltyCodeVio`; `CarePlan` for post-hearing orders
- **OmniScript inventory** — what typically ships baseline in a PSS ICM org and what's missing (Evidence/Custody capture, Copy Participants to Proceeding, Fine/Bond calculation, Participant-centric view)
- **Namespace patterns** — Core (`OmniProcess`, `OmniDataTransform`, `OmniUiCard`) vs vlocity_cmt; naming conventions
- **DR outputs** — `DRId_[ObjectName]` naming convention baked into pre-built CCM DRs
- **Confirmation screen pattern** — DR Post → DR Turbo Extract → Text Block referencing `%CreatedCase|1:CaseNumber%`
- **OmniProcessType gotcha** — `Integration Procedure` (with space), not `IsIntegrationProcedure=true`
- **Standard vs Managed runtime** — `WebComponentKey` is always null on Core standard runtime; do not chase it
- **LWR portal embedding** — `<lightning-omnistudio-omniscript>` + `theme="lightning" lwr` + Experience Builder UI Publish (Connect API publish does NOT trigger the LWR compiler)
- **Permission architecture** — PSG pattern (Citizen, Lawyer, Registry Officer, Tribunal Member, SC Officer)
- **Record types** — `<Tribunal>_Matter` vs `SupremeCourt_Appeal`; `ParentId` links appeal to parent matter
- **ARC (Actionable Relationship Centre)** — configure the native Industries LWC, do NOT rebuild as a FlexCard
- **FLS gotcha** — new custom fields invisible to all profiles by default, even System Admin in enhanced-profile orgs — must deploy PermissionSet FLS entries

## When NOT to use this skill

- Non-PSS Salesforce work — use the matching skill
- Generic OmniStudio not tied to ICM data model — use `omnistudio-retrieve-and-modify`
- Provider / NGO self-onboarding portal — use `salesforce-pss-provider-portal`
- Sales, Service, Marketing, Commerce Cloud — use the matching skill

## Related skills

- `omnistudio-retrieve-and-modify` — parent workflow for any OmniStudio work in a PSS org
- `salesforce-pss-provider-portal` — sister skill for provider self-onboarding
- `platform-metadata-deploy` — deploying `Case` record types, permission sets, FLS
- `experience-lwr-site-generate` — LWR-specific site scaffolding
