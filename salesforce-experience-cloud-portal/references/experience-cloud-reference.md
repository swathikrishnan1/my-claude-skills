# Experience Cloud Build Guide

> Reference this file whenever building an Experience Cloud portal or site.
> **Always ask the user upfront: "Is this LWR or Aura?"** — the approach is completely different.

---

## 0. First Question to Ask

> "Is this LWR (Lightning Web Runtime) or Aura (Visualforce-era)?"

| | LWR | Aura |
|---|---|---|
| Stack | Lightning Web Runtime | Aura Components |
| Component type | LWC only | Aura or LWC |
| Metadata folder | `digitalExperiences/site/<SiteName>/` | `experiences/<SiteName>/` |
| CSS scope | Shadow DOM + `:host` | Global, no shadow |
| Theming | `sfdc_cms__theme`, `sfdc_cms__themeLayout` | `ExperienceBundle` (branding etc.) |
| Full-width trick | Required (see §4) | Not needed |
| Deployment | `sf project deploy start` + Publish API | `sf project deploy start` |

---

## 1. LWR Metadata Structure

```
digitalExperiences/site/<SiteName>/
  sfdc_cms__site/          ← site config (content.json)
  sfdc_cms__theme/         ← theme name + brandingSet reference
  sfdc_cms__themeLayout/
    scopedHeaderAndFooter/ ← header + footer regions (content.json)
  sfdc_cms__view/
    home/                  ← home page component layout (content.json)
  sfdc_cms__route/         ← URL routing entries
  sfdc_cms__brandingSet/   ← colour + font tokens
```

### view content.json — how to place components

```json
{
  "type": "sfdc_cms__view",
  "contentBody": {
    "component": {
      "definition": "community_byo:page",
      "children": [{
        "definition": "community_layout:section",
        "attributes": {
          "sectionConfig": "{\"UUID\":\"<uuid>\",\"columns\":[{\"UUID\":\"<uuid>\",\"columnKey\":\"content\",\"columnWidth\":\"12\"}]}"
        },
        "children": [{
          "id": "<uuid>",
          "name": "content",
          "type": "region",
          "children": [{
            "definition": "c:myLwcComponent",
            "id": "<uuid>",
            "type": "component",
            "attributes": {}
          }]
        }]
      }]
    }
  }
}
```

### themeLayout content.json — header + footer regions

The `scopedHeaderAndFooter` layout has two top-level regions: `header` and `footer`. Wire custom LWCs into the `headerSection` / `footerSection` column children. See the ACAT1 file as a reference template.

### UUIDs

All IDs in view/themeLayout JSON **must be valid hex UUIDs** (no letters outside a-f, no custom strings like `acat0001-...`). Generate with:

```bash
python3 -c "import uuid; print(uuid.uuid4())"
```

---

## 2. LWC for Experience Cloud

### js-meta.xml targets

```xml
<targets>
    <target>lightningCommunity__Page</target>
    <target>lightningCommunity__Default</target>
</targets>
```

**Never** add `lightningCommunity__Header` — it is not a valid target and will cause deploy failure.

### Full-width override (critical)

Experience Cloud wraps every component in a column div. Without this, the component is constrained to the column width:

```css
:host {
    display: block;
    width: 100vw;
    position: relative;
    left: 50%;
    transform: translateX(-50%);
}
```

Add this to the top of every LWC CSS file that needs edge-to-edge layout (header, hero, footer).

---

## 3. Images and CSP

**Salesforce blocks all external image URLs** via CSP. This includes:
- `url('https://www.any-external-site.com/image.jpg')` in CSS
- `<img src="https://...">` in HTML
- Any `background-image` pointing outside the org

### Solution: Static Resources

1. Download the image locally
2. Create `force-app/main/default/staticresources/<name>.jpg` + `<name>.jpg-meta.xml`
3. Deploy it: `sf project deploy start -o "Alias" --source-dir force-app/main/default/staticresources`
4. Reference in JS:

```js
import myImage from '@salesforce/resourceUrl/myImage';

export default class MyComponent extends LightningElement {
    get heroStyle() {
        return `background-image: url('${myImage}'); background-size: cover;`;
    }
    imgUrl = myImage;
}
```

5. Bind in HTML:
```html
<!-- Background image via inline style -->
<section style={heroStyle}></section>

<!-- Img src -->
<img src={imgUrl} alt="" />
```

**Never** use CSS `background-image: url(...)` with external URLs — they will be silently blocked and show nothing.

### meta.xml for static resources

```xml
<?xml version="1.0" encoding="UTF-8"?>
<StaticResource xmlns="http://soap.sforce.com/2006/04/metadata">
    <cacheControl>Public</cacheControl>
    <contentType>image/jpeg</contentType>  <!-- or image/png, image/svg+xml -->
</StaticResource>
```

---

## 4. CSS in LWC (Shadow DOM Rules)

- CSS is **scoped to the component** — classes do not leak in or out
- `!important` is often needed to override Experience Cloud's base theme styles
- Fonts (Google Fonts etc.) may not load — declare `font-family` with system fallbacks, or import via the org's branding theme
- Hover on child elements: use `.parent:hover .child { }` — works fine in scoped CSS
- **No `<style>` tags in HTML** — Salesforce htmlEditor strips them. All CSS goes in the `.css` file.

---

## 5. Publish After Every Deploy

Deploying metadata does **not** make changes live. Always publish after deploy:

```bash
# 1. Get network ID
curl "$INSTANCE/services/data/v62.0/connect/communities" \
  -H "Authorization: Bearer $TOKEN"

# 2. Publish
curl -X POST "$INSTANCE/services/data/v62.0/connect/communities/$NETWORK_ID/publish" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json"

# 3. Poll until complete
# Query: SELECT Id, Status FROM BackgroundOperation WHERE Id = '<jobId>'
# Wait for Status = 'Complete'
```

---

## 6. Embedding OmniScript on LWR (Standard Runtime — Core Namespace)

### The correct LWC wrapper pattern
```html
<template>
    <lightning-omnistudio-omniscript
        type="CourtCaseManagement"
        sub-type="TenancyRentalBondIntake"
        language="English"
        theme="lightning"
        lwr>
    </lightning-omnistudio-omniscript>
</template>
```

```xml
<LightningComponentBundle>
    <apiVersion>62.0</apiVersion>
    <isExposed>true</isExposed>
    <targets>
        <target>lightningCommunity__Default</target>
        <target>lightningCommunity__Page</target>
    </targets>
</LightningComponentBundle>
```

### Key rules
- Use `lightning-omnistudio-omniscript` tag — NOT `omnistudio-omniscript` (not resolvable as LWC module in Core namespace)
- The `lwr` boolean attribute is required for LWR sites
- `experience__AppPage` / `experience__HomePage` / `experience__RecordPage` targets are NOT valid — deploy fails
- `WebComponentKey` will be null on standard runtime — this is correct and expected
- `IsWebCompEnabled` must be `true` for the LWR bundle path to resolve correctly
- LWR3008 error with `false` in path = `IsWebCompEnabled` was false at publish time. Fix: deactivate → set `IsWebCompEnabled=true` → reactivate → republish

### CRITICAL — Phase 4: Publish must go through Experience Builder UI
**DO NOT** use the Connect API publish after adding an OmniScript wrapper to a page. The API publish does NOT trigger the LWR compiler's static analysis for OmniScript component paths.

The correct sequence end-to-end:
1. Create OmniProcess via API (leave `IsWebCompEnabled` alone — do not touch it)
2. Create all OmniProcessElements via API
3. Open OmniScript in **OmniStudio designer** → click **Activate** — this is the only step that sets `IsWebCompEnabled=true` AND runs the compile AND generates `WebComponentKey` atomically
4. Create and deploy a hardcoded LWC wrapper with `lightning-omnistudio-omniscript` + `lwr` attribute + `lightningCommunity__` targets
5. In **Experience Builder**: open the target page → drag the wrapper component from Custom Components panel onto the canvas → click **Publish** (top-right UI button)
6. Wait 2–3 minutes for the CDN routing tables to regenerate
7. Hard-refresh browser (Cmd+Shift+R)

**Why the UI publish matters**: Experience Builder's Publish button triggers the static asset compiler which resolves the LWC module paths at build time. The Connect API publish (POST to `/connect/communities/$ID/publish`) does NOT run this compiler pass for OmniScript components.

### IsWebCompEnabled — how to set it
```bash
# Must deactivate first
curl -X PATCH "$INSTANCE/services/data/v62.0/sobjects/OmniProcess/$OS_ID" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"IsActive": false}'

curl -X PATCH "$INSTANCE/services/data/v62.0/sobjects/OmniProcess/$OS_ID" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"IsWebCompEnabled": true}'
# Then activate via UI
```

---

## 6b. What Does NOT Work in LWR

| Thing | What happens | Fix |
|---|---|---|
| External CSS/image URLs | Silently blocked by CSP | Static Resources (§3) |
| `<style>` blocks in HTML | Stripped by Salesforce | Use `.css` file |
| Custom page routes via metadata | `standard__namedPage` fails for new pages | Use Experience Builder UI to add pages |
| `lightningCommunity__Header` target | Deploy error | Remove — not a valid target |
| Non-hex UUIDs in view JSON | Metadata error | Generate with `python3 -c "import uuid; print(uuid.uuid4())"` |
| `geoBotsAllowed` in site config | Schema validation error | Remove the property |
| SVG static resources with HTML content | Renders as broken image | Use emoji or inline SVG in HTML instead |

---

## 7. ACAT Site — Exact Brand Tokens

These are extracted from the real `www.acat.act.gov.au` site for reference:

```css
/* Colours */
--acat-navy:      hsla(216, 98%, 20%, 1.0);   /* primary navy */
--acat-navy-dark: hsla(216, 98%, 14%, 1.0);   /* footer bottom bar */
--acat-navy-mid:  hsla(216, 98%, 35%, 1.0);   /* icon circles */
--acat-gold:      hsla(39, 98.9%, 62.9%, 1.0); /* gold accent */
--acat-light-bg:  hsla(0, 0%, 95%, 1.0);       /* case type section bg */

/* Header gradient (diagonal white → gold stripe → navy) */
background: linear-gradient(125deg,
    hsla(0, 0%, 98%, 1.0)   22%,
    hsla(39, 98.9%, 62.9%, 1.0) 22%,
    hsla(39, 98.9%, 62.9%, 1.0) 24%,
    hsla(216, 98%, 20%, 1.0) 24%);

/* Fonts */
--font-heading: 'Montserrat', sans-serif;    /* h1, h2, nav, buttons */
--font-body:    'Source Sans Pro', sans-serif; /* body text */
```

---

## 8. LWR Deploy Checklist

- [ ] All UUIDs in view/themeLayout JSON are valid hex UUIDs
- [ ] All images are Static Resources (no external URLs)
- [ ] All LWC CSS files have `:host { width:100vw; left:50%; transform:translateX(-50%); }` if full-width
- [ ] `js-meta.xml` targets are `lightningCommunity__Page` + `lightningCommunity__Default` only
- [ ] Deploy: `sf project deploy start -o "<Alias>" --source-dir <path>`
- [ ] Publish via Connect API + poll `BackgroundOperation` until `Status = Complete`
- [ ] Hard-refresh browser (Cmd+Shift+R) to see changes — Experience Cloud aggressively caches

---

## 9. Aura Portal — LWC Build Rules (ACAT Aura Portal)

The ACAT Aura portal (`acataura`, Network ID `0DBKc000000PCqYOAW`) is an **Aura-based** Experience Cloud site.

### Key differences from LWR
- URL prefix: `/acataura/s/` (note the `/s/` — Aura sites always include it)
- No metadata deploy for page structure — add components via **Experience Builder UI**
- No Connect API publish needed — deploy LWC + refresh in Experience Builder
- **Full-width CSS is still required** — Aura wraps components in columns just like LWR:
```css
:host {
    display: block;
    width: 100vw;
    position: relative;
    left: 50%;
    transform: translateX(-50%);
}
```
- `js-meta.xml` targets must include `lightningCommunity__Page` + `lightningCommunity__Default`
- Guest/public pages: Apex controller must be `public without sharing` and exposed as a guest user accessible Apex class

### Page URL pattern
All page slugs follow `/acataura/s/<page-slug>` — e.g.:
- Home: `/acataura/s/`
- Hearings list: `/acataura/s/acat-hearings-list`
- Tenancy bond intake: `/acataura/s/tenancy-rental-bond-dispute`

### Deploy pattern for Aura portal LWCs
1. `sf project deploy start --source-dir force-app/main/default/lwc/<component> --target-org "ACT Courts"`
2. Open Experience Builder for the ACAT Portal site
3. Navigate to the page, drag the component onto the canvas if new
4. Click **Publish**

---

## 10. SFDX Project Layout (ACT Courts)

```
/tmp/act-courts-fields/
  force-app/main/default/
    lwc/
      acatHeader/          ← gradient header, logo, search, nav
      acatHeroSection/     ← hero with background photo + 2x2 tiles
      acatCaseTypeCards/   ← 8-card get-started grid
      acatFooter/          ← Acknowledgement of Country + link bar
    staticresources/
      acatLogo             ← horizontal ACAT wordmark
      acatLandingImage     ← hero background photo (contains 3D shield)
      acatTileBg           ← dark navy polygon facet texture (primary)
      acatTileBgAlt        ← alternate tile texture
    digitalExperiences/
      site/ACAT1/          ← all LWR site metadata
```

Org alias: `ACT Courts` | Instance: `https://storm-183b2819edf361.my.salesforce.com`
ACAT network ID: `0DBKc000000PCqEOAW`
Supreme Court network ID: `0DBKc000000PCqJOAW`

---

## 11. Aura Portal + OmniScript — Verified Path

Confirmed pattern for a from-scratch, purpose-built Aura Experience Cloud portal that hosts an OmniScript. Use this instead of retrofitting an existing IDO portal (which comes with locked template regions that reject custom LWCs).

### 11.1 Template choice — Build Your Own Aura

For portals that need custom LWCs + OmniScript, choose **Build Your Own (Aura)**. Skip:
- **Customer Service (Napili)** — has stock header/nav that fights your branding
- **LWR templates** — OmniScript still doesn't fully work on LWR (module not found errors)

### 11.2 Site creation checklist

1. Setup → All Sites → **New** → Build Your Own (Aura)
2. Name: `<Portal Name>` (readable)
3. URL prefix: `provider` (short, lowercase, single segment)
4. Base URL becomes `https://<org>.my.site.com/provider/s/...`

### 11.3 Header, hero, footer, content-cards LWC recipe

**Four LWCs, dropped into Content Header / Content / Content Footer regions:**

- **`portalHeader`** — top strip "A <Government> website" (or agency label) + <government> logo (via static resource) + site title + nav
- **`portalHeroSection`** — full-width navy hero with headline + tagline + centred CTAs. Skip 2-col tile grid here (content cards handle that)
- **`portalContentCards`** — 4-tile grid, all neutral by default, turn accent on hover. Uses `.portal-ct:hover { background: linear-gradient... color: #FFFFFF; }` to swap colours. Also swap child element colours in the same hover rule.
- **`portalFooter`** — dark footer with optional acknowledgement block (jurisdictions have their own conventions — customise per demo) + 4 link columns

All four use the full-width CSS override at the top:
```css
:host { display: block; width: 100vw; position: relative; left: 50%; transform: translateX(-50%); }
```

### 11.4 <Government> logo — static resource pattern

**Never** hotlink external images (CSP blocks them). **Never** hand-draw an SVG (users will call out that it looks wrong).

Correct pattern:

```bash
# 1. Download official SVG from a public <government> source
curl -sSL -o /tmp/agency.svg "https://www.example-agency.gov.example/content/dam/agency/logo/AgencyLogo.svg"

# 2. Drop into force-app/main/default/staticresources/AgencyLogo.svg
# 3. Create resource-meta.xml with contentType=image/svg+xml
# 4. In LWC:
```

```js
import AGENCY_LOGO from '@salesforce/resourceUrl/AgencyLogo';
export default class PortalHeader extends LightningElement {
    agencyLogo = AGENCY_LOGO;
}
```

```html
<img src={agencyLogo} alt="<Government>" class="portal-header__logo-img" />
```

### 11.5 CSP Trusted Sites for Google Maps

**Required for Maps JavaScript API + Geocoding API.** Deploy this schema (minimal fields — full field set errors on element-order validation):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<CspTrustedSite xmlns="http://soap.sforce.com/2006/04/metadata">
    <endpointUrl>https://maps.googleapis.com</endpointUrl>
    <description>Google Maps JavaScript API</description>
    <isActive>true</isActive>
    <context>All</context>
</CspTrustedSite>
```

Deploy two — one for `maps.googleapis.com`, one for `maps.gstatic.com`.

**Do NOT include** `isApplicableToScriptSrc`, `isApplicableToStyleSrc`, `isApplicableToConnectSrc` etc. — those fields exist but have strict ordering rules that fail on deploy. The minimal 4-field version works everywhere.

### 11.6 Custom Metadata Type for API key storage

For any 3rd-party API key (Google Maps, Mapbox, etc.):

1. Create `Portal_Config__mdt` Custom Metadata Type
2. Add fields: `Google_Maps_API_Key__c` (Text 255), etc.
3. **Deploy the CMT layout too** — Salesforce doesn't auto-add fields to the CMT edit page. Layout file: `layouts/Portal_Config__mdt-Portal Config Layout.layout-meta.xml`
4. Deploy an Apex service that reads via `Portal_Config__mdt.getInstance('Default').Google_Maps_API_Key__c`
5. User creates the "Default" record via Setup UI (Custom Metadata Types → Portal Config → Manage Records → New → paste key → Save)

**Never** put the API key in a file or in git. It lives only in the org.

### 11.7 CMT layout gotcha (very easy to miss)

When you add a custom field to a Custom Metadata Type via metadata deploy, **it will not appear on the CMT's edit page**. Salesforce doesn't auto-add. You must also deploy a Layout metadata file:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Layout xmlns="http://soap.sforce.com/2006/04/metadata">
    <layoutSections>
        <label>Information</label>
        <editHeading>true</editHeading>
        <layoutColumns>
            <layoutItems>
                <behavior>Required</behavior>
                <field>MasterLabel</field>
            </layoutItems>
            <layoutItems>
                <behavior>Required</behavior>
                <field>DeveloperName</field>
            </layoutItems>
            <layoutItems>
                <behavior>Edit</behavior>
                <field>Your_Custom_Field__c</field>
            </layoutItems>
        </layoutColumns>
        <style>OneColumn</style>
    </layoutSections>
</Layout>
```

Filename must be exactly `<CMT>-<Layout Label>.layout-meta.xml` where the layout label defaults to `<CMT display name> Layout` — e.g. `Portal_Config__mdt-Portal Config Layout.layout-meta.xml`.

### 11.8 Existing IDO portal — extend vs replace

**Extending an IDO's default portal (e.g. `Service_Provider_Portal1`) fights the template**:
- Content Header / Content / Content Footer regions may reject custom LWC drops (locked by template)
- Existing OOTB nav ("Government of the Future" strip) is not easily removable

**Better:** Create a fresh **Build Your Own (Aura)** site from scratch. All settings are yours to control. Migration is just adding pages + dropping the same LWCs.

### 11.9 OmniScript on Aura — no publish required for OmniScript changes

Aura sites read the *active* OmniScript version directly from `OmniProcess` records. Changes take effect the moment you Activate in OmniStudio Designer — **no site republish needed**.

Corollary: if the portal doesn't reflect your OmniScript changes, the fix is usually one of:
1. OmniScript is not actually Active (`SELECT IsActive FROM OmniProcess`)
2. Browser cache — hard refresh incognito

### 11.10 Deploy order for the portal

1. **Static resources** (logos, images) — deploy first, other things reference them
2. **Custom Metadata Types + fields + layouts** — deploy fields FIRST, then layout
3. **Apex classes** — deploy before LWCs that use `@salesforce/apex/...`
4. **LWCs** — bundle deploy (all 4 files each)
5. **CSP Trusted Sites** — before any HTTP callouts fire from LWCs
6. **Permission sets** — assign to user after deployment
7. **OmniScript metadata (OmniProcess + Elements)** — build via API script; then user activates via UI
8. **Site publish** — only after all page-level components are in the LWC library

### 11.11 Field-level security + Apex access permission set — one bundle

For anything a demo user runs (Apex classes, custom fields on standard objects), a single permission set is cleanest:

```xml
<PermissionSet ...>
    <label>Provider Portal Access</label>
    <classAccesses>
        <apexClass>ProviderOnboardingService</apexClass>
        <enabled>true</enabled>
    </classAccesses>
    <objectPermissions>
        <object>HealthcareProvider</object>
        <allowCreate>true</allowCreate><allowRead>true</allowRead><allowEdit>true</allowEdit>
    </objectPermissions>
    <fieldPermissions>
        <field>Account.ABN__c</field>
        <readable>true</readable><editable>true</editable>
    </fieldPermissions>
</PermissionSet>
```

Assign to the running user post-deploy via `sf data create record --sobject PermissionSetAssignment ...`.

### 11.12 Trace Flag for Apex debug — mandatory before hunting Apex errors

Setup does NOT auto-capture logs. Set a trace flag on the user first:

```bash
DEBUG_LEVEL_ID=$(sf data query --query "SELECT Id FROM DebugLevel WHERE DeveloperName = 'SFDC_DevConsole'" --use-tooling-api --json | jq -r .result.records[0].Id)
sf data create record --sobject TraceFlag --values "TracedEntityId=<userId> DebugLevelId=$DEBUG_LEVEL_ID LogType=USER_DEBUG StartDate=<now> ExpirationDate=<now+1h>" --use-tooling-api
```

Then trigger the Apex + `sf apex get log --log-id <id>` to see the actual exception.
