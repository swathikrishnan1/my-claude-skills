---
name: salesforce-experience-cloud-portal
description: >-
  Salesforce Experience Cloud portal build guide covering LWR (Lightning Web
  Runtime) vs Aura decision, theming, LWC hosting, guest user setup, publish
  gotchas, and CSP/branding patterns. ALWAYS start by asking the user "Is this
  LWR or Aura?" — the approach is completely different and getting it wrong
  wastes a day. TRIGGER on requests to build an Experience Cloud portal, public
  site, community, or citizen-facing portal on Salesforce; on LWR vs Aura
  choice questions; on Experience Builder publish failures; on guest user
  access issues; on portal branding, header, footer, hero LWC work; on portal
  URL prefix, custom domain, or my.site.com setup; on "why doesn't my
  Experience Builder change appear on the site" (usually the publish button);
  on portal LWC full-width CSS overrides; on CMT (Custom Metadata Type)-driven
  portal config; on CSP Trusted Sites for external images. TRIGGER also on
  requests about OmniScript embedding in a portal — but WARN LOUDLY that
  OmniScripts do NOT work reliably on LWR (module not found errors,
  IsWebCompEnabled complications, Connect API publish does not compile the
  static bundle path). For OmniScript-hosting portals, use Aura Experience
  Cloud "Build Your Own" template. Bundled reference notes cover LWR vs Aura
  comparison, section 6b (what does NOT work in LWR), Aura + OmniScript
  verified path, agency-logo static resource pattern, and CMT-based portal
  config. DO NOT TRIGGER for internal-only Lightning Experience apps (use
  platform-lightning-app-coordinate), for the OmniScript itself (use
  omnistudio-retrieve-and-modify), or for the Callable Apex behind an
  OmniScript Remote Action (use platform-apex-generate).
---

# Salesforce Experience Cloud — LWR vs Aura Portal Skill

## CRITICAL WARNING — OmniScripts and LWR

**OmniScripts do NOT work reliably on LWR.** This is the single most expensive lesson in this skill. Every time you try to host an OmniScript on an LWR site you hit some combination of:

- "Module not found" errors resolving `<lightning-omnistudio-omniscript>`
- `IsWebCompEnabled=false` baked into the bundle path
- Connect API publish not triggering the LWR static asset compiler
- Stale bundle paths surviving hard refresh
- LWR3008 error with `__false__` in the module path

**If the user wants a portal that hosts an OmniScript, ALWAYS route to Aura Experience Cloud** ("Build Your Own (Aura)" template — see `salesforce-pss-provider-portal` for the full recipe).

Only build on LWR when the portal does NOT need to host an OmniScript.

## Layer 0 — First Question to Ask

Before touching anything, ask the user:

> **"Is this LWR (Lightning Web Runtime) or Aura?"**

The answer changes everything — metadata folder, CSS scoping, theming approach, deployment commands, publish mechanism, full-width CSS trick, LWC support, gotchas.

If they don't know, help them decide:
- **Pick LWR when:** modern SEO-friendly site, marketing / brochure / general public content, no OmniScript hosting, custom LWCs only.
- **Pick Aura when:** hosting an OmniScript, needs Aura components, needs the "Build Your Own" flexibility, or you're extending an existing IDO portal.

## How to use this skill

1. **Read the bundled reference** — `./references/experience-cloud-reference.md` covers the LWR vs Aura comparison, LWR full-width trick, LWR publish gotchas, section 6b ("What Does NOT Work in LWR"), the verified Aura + OmniScript path, agency-logo static resource pattern, and CMT-based portal config.
2. **Ask the LWR vs Aura question first** if the user hasn't specified.
3. **If OmniScript hosting is required, WARN LOUDLY** and route to Aura + `salesforce-pss-provider-portal`.

## Key patterns captured in the reference

- **LWR vs Aura comparison table** — stack, component types, metadata folder, CSS, theming, deployment
- **LWR full-width CSS override** — `:host { display: block; width: 100vw; ... }` — required to break out of LWR's fixed-width layout
- **Deploy + Publish sequence** — metadata deploy does NOT make changes live; must publish via Experience Builder UI (Connect API publish does NOT trigger the LWR compiler for OmniScript paths)
- **What does NOT work in LWR** — OmniScripts (see warning above), certain Aura components, some Salesforce Industries LWCs
- **Guest user setup** — sharing rules, network access, object CRUD via Guest Permission Set
- **Agency logo static resource pattern** — download official SVG from a public agency source, deploy as static resource, import via `@salesforce/resourceUrl/<Name>`, reference in template
- **CMT-driven portal config** — deploy `Portal_Config__mdt` with configurable fields (API keys, feature flags), including the Layout metadata file so fields actually appear on the CMT edit page
- **CSP Trusted Sites** — required for external images / API calls from the portal; hand-edit `.cspTrustedSite-meta.xml` with `endpointUrl` / `description` / `isActive` / `context` (in that order — Salesforce is strict about element order)
- **URL prefix conventions** — short, lowercase, single segment (e.g. `provider`, not `provider-portal-v2`)

## When NOT to use this skill

- Internal-only Lightning Experience apps — use `platform-lightning-app-coordinate`
- The OmniScript inside a portal — use `omnistudio-retrieve-and-modify`
- Building the provider-onboarding portal specifically — use `salesforce-pss-provider-portal`
- Non-Experience-Cloud Salesforce work

## Related skills

- `salesforce-pss-provider-portal` — verified Aura + OmniScript portal recipe
- `omnistudio-retrieve-and-modify` — OmniStudio retrieve-first workflow
- `platform-lightning-app-coordinate` — internal Lightning apps (not portals)
- `experience-lwr-site-generate` — bare-metal LWR site scaffolding
- `platform-metadata-deploy` — deploying portal metadata bundles
