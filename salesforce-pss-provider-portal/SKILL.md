---
name: salesforce-pss-provider-portal
description: >-
  Reusable recipe for building an NGO / provider self-onboarding portal on
  Salesforce Public Sector Solutions (PSS) + Aura Experience Cloud + OmniScript.
  Use this skill whenever the user is building a public-facing portal where an
  external organisation submits an intake form to create Account + Contact +
  HealthcareProvider records with a Pending status awaiting agency approval.
  TRIGGER on requests to build a provider onboarding portal, NGO self-service
  intake, allied-health provider signup, external-org registration flow, or any
  public portal that creates a HealthcareProvider record from an OmniScript.
  Also TRIGGER when the user hits the specific pain points captured here —
  ProviderManagementPsl not assigned, PSL granted but object CRUD still
  missing, HealthcareProvider ProviderType / ProviderClass / Status picklist
  values, OmniScript Type/SubType alphanumeric-only rejection (no underscores or spaces), Callable Apex signature
  ("global" + "System.Callable"), args nested under input.AllData.<Step>.<Field>,
  Remote Action placement at Level 0 between Steps, "Build Your Own Aura"
  template choice for OmniScript-hosting portals. Bundled reference notes cover
  the full 5-step OmniScript structure, Callable Apex wrapper, permission set
  layering, agency-logo static resource pattern, and the Aura vs LWR decision.
  IMPORTANT — OmniScripts do NOT work reliably on LWR. This skill uses Aura
  Experience Cloud (Build Your Own template) exclusively for OmniScript
  hosting. DO NOT TRIGGER for LWR-only sites (use salesforce-experience-cloud-portal),
  PSS Investigative Case Management demos (use salesforce-pss-icm), or generic
  OmniStudio work (use omnistudio-retrieve-and-modify).
---

# PSS Provider Portal — Aura + OmniScript Recipe

## What this skill is for

A tested, end-to-end recipe for building a **public provider self-onboarding portal** on Salesforce Public Sector Solutions. External providers (NGOs, allied-health providers, community organisations) open a public URL, complete a 5-step OmniScript, and get their records created with a Pending status awaiting agency approval.

## When to use this skill

Trigger on any of these:
- "Build a provider onboarding portal"
- "NGO self-registration flow on PSS"
- "Public-facing form to create HealthcareProvider records"
- "External-org intake with agency approval workflow"
- "Provider Management PSL not working"
- "Portal user submits an OmniScript and it creates Account + Contact + HealthcareProvider"
- Any request naming: `HealthcareProvider`, `ProviderManagementPsl`, `Build Your Own Aura`, "provider onboarding"

## Critical constraint — Aura only, not LWR

**OmniScripts do NOT work reliably on LWR.** LWR portals hit "module not found" errors when trying to host an OmniScript. This recipe uses **Aura Experience Cloud with the "Build Your Own (Aura)" template** — the only path we've verified end-to-end.

If the user asks for an LWR portal that hosts an OmniScript, warn them explicitly and route them to Aura, or route to `salesforce-experience-cloud-portal` for the LWR gotchas.

## How to use this skill

1. **Read the bundled reference first** — `./references/pss-provider-portal-reference.md` covers PSS prerequisites (ProviderManagementPsl + Permission Set), HealthcareProvider fields, the 5-step OmniScript structure, the Callable Apex service class, permission set layering, and the Aura portal recipe.
2. **Follow the OmniScript build pattern from `omnistudio-retrieve-and-modify`** — retrieve first, modify existing patterns, activate manually via UI.
3. **Watch the sequencing** — PSL first, then object CRUD Permission Set, then OmniScript, then Callable Apex, then Aura portal.

## Key patterns captured in the reference

- **Data model** — Account (org) + Contact (primary contact) + HealthcareProvider. No custom objects needed.
- **HealthcareProvider fields** — ProviderType (`Onsite Service Provider` / `At Home Service Provider`), ProviderClass (`Medical Group` for NGOs), Status (`Pending` on submit).
- **Custom fields on Account** (not HealthcareProvider) — Trading_Name__c, ABN__c, ACN__c.
- **Prerequisites** — ProviderManagementPsl assigned to the running user AND a Permission Set granting object CRUD. PSL alone is not enough.
- **OmniScript structure** — 5 Steps + Remote Action + Confirmation Step. Type/SubType alphanumeric only, no underscores or spaces.
- **Callable Apex signature** — `global without sharing` + `implements System.Callable`. Args nested `args.input.AllData.<Step>.<Field>` — always unwrap.
- **Remote Action placement** — Level 0, `ParentElementId = null`, between the last Step and the Confirmation Step.
- **Aura portal recipe** — Build Your Own Aura template, 4 LWCs (portalHeader, portalHeroSection, portalContentCards, portalFooter), full-width CSS override, static resource for agency logo.
- **Guest user access** — configure network Guest User profile, assign object CRUD to the Guest User's permission set, add sharing rules if the OmniScript needs to read existing records.

## When NOT to use this skill

- LWR-only portals — use `salesforce-experience-cloud-portal` (which explicitly warns OmniScripts don't work on LWR)
- Justice / Courts / Investigations builds — use `salesforce-pss-icm`
- Generic OmniStudio work — use `omnistudio-retrieve-and-modify`
- Non-PSS Salesforce work

## Related skills

- `omnistudio-retrieve-and-modify` — parent OmniStudio workflow (retrieve-first)
- `salesforce-experience-cloud-portal` — LWR vs Aura decision, Experience Cloud gotchas
- `salesforce-pss-icm` — sister PSS skill for courts, tribunals, case management
- `platform-apex-generate` — Callable Apex class scaffolding
- `platform-permission-set-generate` — Permission Set for object CRUD
