---
name: omnistudio-guide
description: "How OmniStudio components work together — FlexCards, OmniScripts, DataRaptors, Integration Procedures — real-build learnings"
metadata:
  type: reference
---

# OmniStudio — How It All Works Together

## The Four Components

### 1. DataRaptors
- **What they do:** Read or write Salesforce data. Two types used here:
  - **Extract** — queries Salesforce records and outputs JSON
  - **Load** — takes input JSON and creates/updates Salesforce records
- **Activation:** Must be activated in OmniStudio UI before they work. API creation alone is not enough.
- **Context:** They receive input parameters (key/value pairs) and output named JSON nodes.
- **Example:** `PSSCaseworkOverviewGetCaseProceedingsIDO` takes `recordId` (Case ID) as input, returns `CaseProceedings` list as output.

### 2. FlexCards
- **What they do:** Display data on a Lightning record page. Like a mini dashboard component — can show a data table, buttons, child FlexCards.
- **Data Source:** Each FlexCard points to a DataRaptor (Extract) or Integration Procedure. The data source receives an **Input Map** (key → value bindings using `{token}` syntax).
- **Input Map tokens:** Use `{recordId}` to pass the current page's record ID to the DataRaptor. This is a special FlexCard page-context variable — it is NOT in the data source dropdown but can be typed directly.
- **Activation:** Must be activated in OmniStudio UI. API-created FlexCards are "In Active" by default.
- **Actions:** Buttons on FlexCards can have actions:
  - `Open OmniScript` — launches an OmniScript
  - The **Context ID** field on the action passes a value into the OmniScript as `%ContextId%`
  - Context ID should be `{recordId}` to pass the current Case ID
  - The **Input Parameters** section passes additional named values
- **Dropdown limitation:** The Context ID field's dropdown only shows data source fields (e.g. `{TotalRecordsCount}`). For page-context variables like `{recordId}`, type it directly — it works even though it's not in the dropdown.
- **Version management:** FlexCards have versions. "In Active" = draft. Must click Activate to go live.

### 3. OmniScripts
- **What they do:** Guided multi-step forms/flows. Used for data entry (schedule hearing, create proceeding, benchsheet, lodge appeal, etc.)
- **Activation:** CRITICAL — OmniScripts built via API must be activated in OmniStudio UI (open → click Activate). API creation alone does not compile the WebComponentKey needed to render. Without activation, the OmniScript is a blank screen.
- **Context variable:** `%ContextId%` — receives the ID passed from the FlexCard's Context ID field or Action Launcher. This is how the OmniScript knows which Case it's running on.
- **SetValues element:** Used at the start of OmniScripts to capture context. Example: `SetCaseID` element maps `CaseID = %ContextId%`. If ContextId is empty, CaseID is blank and all downstream DataMapper saves fail with "Required fields are missing: [CaseId]".
- **DataMapper Extract:** Queries Salesforce using values set earlier in the script. Uses the input like `%CaseID%`.
- **DataMapper Post (Load):** Creates/updates records. Receives field values from the OmniScript form elements.
- **Error "Required fields are missing: [CaseId]"** — means %ContextId% was empty when the OmniScript launched. Fix: ensure the FlexCard or Action Launcher passes the record ID as Context ID.
- **DO NOT patch active OmniScripts via API** — will break them. Only edit via OmniStudio UI or create a new version.
- **Patching IsActive via API:** After creating an OmniScript element via API, must PATCH `IsActive=true` on each element, then PATCH `IsActive=true` on the OmniScript itself.

### 4. Integration Procedures
- **What they do:** Server-side orchestration. Like a flow but for OmniStudio — chains DataRaptors, HTTP callouts, conditional logic.
- **Used for:** Complex multi-step saves that span multiple objects. Example: `ScheduleHearingAndNotify` chains CaseProceeding create + email send.
- **Activation:** Must be activated separately in OmniStudio UI.

---

## How They Wire Together (the pattern)

```
Lightning Record Page (Case)
    └── FlexCard embedded as component
            └── DataRaptor Extract (data source) ← receives {recordId}
                    └── returns list of related records (e.g. CaseProceedingsIDO)
            └── Button (New) → Action: Open OmniScript
                    └── Context ID = {recordId}  ← THIS IS THE CRITICAL LINK
                            └── OmniScript receives %ContextId% = Case ID
                                    └── SetValues: CaseID = %ContextId%
                                    └── DataMapper Extract: get case details using %CaseID%
                                    └── Screen step: user fills in form
                                    └── DataMapper Post: create CaseProceeding with CaseId = %CaseID%
```

---

## The casework demo FlexCard Stack

The standard PSS `PSSCaseworkOverviewShowCaseProceedingsIDO` FlexCard:
- Data source: `PSSCaseworkOverviewGetCaseProceedingsIDO` DataRaptor Extract
- Input Map: `recordId = {recordId}` (passes Case page record ID)
- Displays: CaseProceedings data table (Name, Description, CaseFilingDateTime)
- New button action: Open OmniScript `CourtCaseManagement/CreateNewCourtCaseProceeding`
- Context ID: `{recordId}` — must be set for the OmniScript to receive the Case ID

**The fix we made:** The FlexCard's New button Context ID was empty. Setting it to `{recordId}` fixed the "Required fields are missing: [CaseId]" error in the OmniScript.

---

## Common Errors and Fixes

| Error | Cause | Fix |
|---|---|---|
| "Required fields are missing: [CaseId]" | OmniScript %ContextId% is empty | Set FlexCard Action Context ID = {recordId} |
| OmniScript renders blank | Not activated via UI | Open in OmniStudio → Activate |
| FlexCard shows "No Records to Display" | DataRaptor not activated, or wrong input | Activate DataRaptor; check Input Map uses {recordId} |
| Context ID dropdown doesn't show {recordId} | It's a page variable, not a data field | Type {recordId} directly — it works |
| "Role: bad value for restricted picklist" | OmniScript saving a role value not in the picklist | Add the value to the picklist in Setup → Object Manager |
| "Required fields are missing: [InfractionId]" | GetRegulatoryCodeViolation DataRaptor can't find RCV because RCV has no CaseId field | Known bug in standard PSS OmniScript — click Continue, CaseProceeding still saves |

---

## Activation Checklist (manual UI steps — cannot be automated via API)

For every OmniScript, FlexCard, DataRaptor, Integration Procedure created via API:
1. Go to App Launcher → OmniStudio
2. Find the component by name
3. Open it
4. Click **Activate**

OmniScript activation also compiles the LWC — required for the component to render on a page.

---

## DataRaptor Naming Convention (casework demo)

- `PSSCaseworkOverview*IDO` — standard PSS cluster, reads case-related data for FlexCards
- `CCMDRCreate*` — Load DataRaptors for creating records (CaseParticipant, CustodyItem)
- `CCMDRGet*` — Extract DataRaptors for reading records
- `GetDataForCaseSummary` — Extract for Agentforce case summary

---

## Integration Procedures (casework demo — example pattern)

| Name | Purpose |
|---|---|
| CreateMatter | Creates Case from portal intake |
| ScheduleAndNotify | Creates child record + sends email |
| GenerateAndSealDocument | DocGen + document sealing |
| CreateAppeal | Creates escalation Case |

---

## OmniScript Remote Actions — Building From Scratch via API (provider onboarding demo, 2026-07-22)

Building a from-scratch OmniScript with a Remote Action to Apex, deployed to an Aura portal. Painful learnings, all captured here so future demos take minutes not hours.

### 1. Apex signature — MUST be `global` + implement `System.Callable`

**Not** `@AuraEnabled`. Not `public`. OmniStudio's Remote Action runtime does `Type.forName(namespace, className).newInstance()` — that requires:

```apex
global without sharing class TKProviderOnboardingService implements System.Callable {
    global Object call(String action, Map<String, Object> args) {
        if (action == 'onboardProvider') { return onboardProvider(args); }
        return new Map<String, Object>{'success' => false, 'error' => 'Unknown action'};
    }
    ...
}
```

**If class is `public` (not `global`):** OmniStudio throws `Can not get instance of Class null.<ClassName>` on Submit. Deploys and compiles fine, but instantiation fails at runtime.

**If class doesn't implement `Callable`:** Same error — `call()` is what OmniStudio invokes.

### 2. Args are wrapped AND step-nested — must be unwrapped BEFORE reading fields

The `Map<String, Object> args` passed to `call()` has this shape:

```json
{
  "input": {
    "AllData": {
      "Step1LegalReg": { "LegalName": "Example Co", "TradingName": null },
      "Step2Address":  { "Street": "123 Main St", "City": "Anytown" },
      "Step3Services": { "ServicesOffered": "ServiceA" },
      "Step4Contact":  { "ContactFirstName": "Jane", ... },
      "localTimeZoneName": "Australia/Sydney",
      "omniProcessId": "...",
      ...
    },
    "omniScriptId": "0jNgL...",
    "elementName": "SubmitProvider"
  },
  "output": { "error": "OK" },
  "options": { "useQueueableApexRemoting": false, "vlcClass": "...", ... }
}
```

**Fields you configured in the OmniScript are nested TWO levels deep**: `args.input.AllData.<StepName>.<FieldName>`.

**Standard extract helper:**

```apex
private static Map<String, Object> extractFormData(Map<String, Object> args) {
    if (args == null) return new Map<String, Object>();
    Map<String, Object> rawData;
    Object inputObj = args.get('input');
    if (inputObj instanceof Map<String, Object>) {
        Map<String, Object> input = (Map<String, Object>) inputObj;
        Object all = input.get('AllData');
        rawData = (all instanceof Map<String, Object>) ? (Map<String, Object>) all : input;
    } else {
        rawData = args;
    }
    // Flatten step-nested keys — each entry whose value is a Map is a Step wrapper.
    Map<String, Object> flat = new Map<String, Object>();
    for (String key : rawData.keySet()) {
        Object v = rawData.get(key);
        if (v instanceof Map<String, Object>) {
            Map<String, Object> nested = (Map<String, Object>) v;
            for (String innerKey : nested.keySet()) {
                flat.put(innerKey, nested.get(innerKey));
            }
        } else {
            flat.put(key, v);
        }
    }
    return flat;
}
```

After this, `data.get('LegalName')` returns the actual value regardless of which Step wraps it.

**Wrong approaches that failed:**
- `args.get('legalName')` → null (data is nested under `input.AllData.<Step>.<Field>`)
- Wrapper class `public class Input { public String legalName; }` + `onboardProvider(Input in)` → doesn't work with Callable; the signature must be `Map<String, Object>`

### 3. Return type — `Map<String, Object>` (avoid throwing)

The OmniScript client treats a thrown exception as opaque "Script-thrown exception" — you can't debug from the popup. Better pattern: catch everything, return a result map:

```apex
Map<String, Object> result = new Map<String, Object>();
try {
    // ... do work, catch specific errors
    result.put('success', true);
    result.put('recordId', hcp.Id);
    result.put('message', 'Onboarded successfully.');
} catch (Exception e) {
    Database.rollback(sp);
    result.put('success', false);
    result.put('error', e.getMessage());
    result.put('type', e.getTypeName());
}
return result;
```

Then read `%SubmitResult:success%` and `%SubmitResult:error%` in later OmniScript steps for conditional display.

### 4. Element Type strings (case + spacing matters — Salesforce is strict)

Correct API values for the `Type` field on `OmniProcessElement`:

| Meaning | Correct value |
|---|---|
| Text input | `Text` |
| Text area | `Text Area` (space) |
| Email input | `Email` |
| Phone input | `Telephone` |
| Rich HTML block | `Text Block` |
| Multi-step container | `Step` |
| Static defaults | `Set Values` |
| Apex Callable action | `Remote Action` |
| DataRaptor Load | `DataRaptor Post Action` |
| DataRaptor Extract | `DataRaptor Extract Action` |
| Integration Procedure | `Integration Procedure Action` |

**Wrong values that fail:** `TextArea` (no space), `Phone` (should be Telephone), `RemoteAction` (needs space).

Grab the definitive list from the org: `SELECT Type, COUNT(Id) c FROM OmniProcessElement WHERE OmniProcess.OmniProcessType = 'OmniScript' GROUP BY Type ORDER BY COUNT(Id) DESC`.

### 5. Type + SubType field values — alphanumeric only

`OmniProcess.Type` and `OmniProcess.SubType` **cannot contain underscores or spaces**. Salesforce error: `Enter a value that starts with a letter and contains only alphanumeric characters without spaces or underscores`.

Correct: `TKProvider / Onboarding`.
Wrong: `TK_Provider / Onboarding`.

### 6. Remote Action placement in the OmniScript — Level 0, standalone, between Steps

**Wrong (what I built first):** Remote Action nested inside Step 5 (`ParentElementId = Step5Review`, Level 1). Renders a giant Submit button in the step body, doesn't advance to the next step automatically.

**Right (what actually works):** Remote Action as a **Level 0, top-level element BETWEEN Step 5 and Step 6**, `ParentElementId = null`. Runtime flows Step 5 → executes Remote Action inline → advances to Step 6. This is the pattern SPCM SubmitReferral uses.

**Sequence example (real, verified working):**

| SeqNo | Level | Type | Name | Parent |
|---|---|---|---|---|
| 4 | 0 | Step | Step5Review | null |
| **5** | **0** | **Remote Action** | **SubmitProvider** | **null** ← between steps |
| 6 | 0 | Step | Step6Confirmation | null |

### 7. IsActive PATCH is required on every new element (default is false)

After creating any `OmniProcessElement` via API, immediately `PATCH IsActive: true` on the new record. Elements default to inactive and won't render in the designer canvas or at runtime.

### 8. Building via API is idempotent-hostile — build in idempotent scripts

If the first POST fails halfway through, re-running the script tries to recreate elements that already exist and errors on unique-name violation. Always check existing elements first:

```python
existing = query("SELECT Id, Name FROM OmniProcessElement WHERE OmniProcessId = '<osId>'")
# Skip create if name already exists — reuse the Id instead.
```

### 9. Confirmation step pattern

Add a Step 6 with `hideNextBtn: true` and `hidePrevBtn: true` so it's a dead-end. Inside, put a `Text Block` element with inline HTML (styles allowed, no `<script>`).

```json
{
  "name": "ConfirmationMessage",
  "HTMLTemplate": "<div style='padding:40px;text-align:center;'><h2>Application Submitted</h2><p>...</p></div>"
}
```

### 10. Cache behaviour — Preview vs Portal

- **OmniStudio Designer Preview** reads the OmniScript record directly — no publish cycle needed for updates.
- **Aura portal** picks up the *active* OmniScript version live — no site republish needed either.
- **But**: browser + CDN cache the compiled bundle aggressively. Hard refresh (Cmd+Shift+R in incognito) after every Activate cycle.

### 11. Deactivate → Edit → Activate cycle

You **cannot** edit elements on an active OmniScript. Even patching PSC via API returns 400. Always:
1. PATCH `IsActive: false` on the OmniProcess
2. Make your changes
3. Have the user click **Activate** in the OmniStudio Designer UI (API activation doesn't compile the WebComponentKey correctly)

### 12. Common error message decoder

| Error | Real cause | Fix |
|---|---|---|
| `Can not get instance of Class null.<ClassName>` | Apex isn't `global` and/or doesn't implement `Callable` | Rewrite Apex per §1 |
| `Script-thrown exception` (no detail) | Apex threw an unhandled exception. Get the log to see what | Trace flag on user, re-run, `sf apex get log --log-id <id>` |
| Submit button doesn't advance | Remote Action nested inside the Step (Level 1) | Move to Level 0 between Steps |
| Element renders as empty blue rectangle | `label: ""` on Remote Action | Give it a proper label |
| Same OmniScript works in Preview but not Portal | Different active versions, or portal is on browser cache | Confirm `IsActive: true`, hard refresh |
| `Enter a value that starts with a letter and contains only alphanumeric characters` | Underscore or space in Type/SubType | Use PascalCase, no separators |
| Deploy error: `Element {isApplicableToScriptSrc} invalid at this location` | CSPTrustedSite XML has fields in wrong order | Salesforce is strict about XML element order — use minimal 4-field version (endpointUrl, description, isActive, context) |
| Custom Metadata Type field doesn't show on edit page | Field not added to the CMT's layout | Deploy a Layout metadata file, OR add via Setup UI |

