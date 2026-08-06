# PSS Provider Portal — Reusable Build Recipe

> Reusable recipe for building a provider self-onboarding portal on **PSS + Aura + OmniScript**.
> Everything here is verified working end-to-end in a Public Sector IDO.

---

## What This Portal Is

- **User story:** An NGO or provider organisation opens a public URL, fills a 5-step OmniScript form, hits Submit. Records are created: an **Account** (org), a **Contact** (primary contact), a **HealthcareProvider** (PSS provider record, `Status = Pending` awaiting agency approval).
- **Runtime:** Aura Experience Cloud (Build Your Own template). LWR does not yet reliably host OmniScripts.
- **Data model:** Standard PSS **HealthcareProvider** + Salesforce **Account** (via `AccountId` lookup) + **Contact** (via `PractitionerId` lookup). No custom objects needed.

---

## PSS Prerequisites

| Requirement | How to verify | If missing |
|---|---|---|
| **Provider Management PSL** — enables `HealthcareProvider` object | `sf data query --query "SELECT DeveloperName, TotalLicenses, UsedLicenses FROM PermissionSetLicense WHERE DeveloperName = 'ProviderManagementPsl'"` | Not present in some PS demo orgs — request enablement from Salesforce |
| **PSL assigned to running user** | `sf data query --query "SELECT Id FROM PermissionSetLicenseAssign WHERE Assignee.Username = '<user>' AND PermissionSetLicense.DeveloperName = 'ProviderManagementPsl'"` | `sf data create record --sobject PermissionSetLicenseAssign --values "AssigneeId=<userId> PermissionSetLicenseId=<pslId>"` |
| **Object CRUD** on HealthcareProvider, Account, Contact | `sf sobject describe --sobject HealthcareProvider \| grep createable` — must be `True` | PSL alone grants feature, but a Permission Set is required for object CRUD. Build & assign one (see §5 below). |

**Reality check:** even with the PSL assigned, the user still needs Object CRUD from a Permission Set. Assigning the PSL alone does NOT grant record-level access.

---

## HealthcareProvider — The Fields That Matter

**Standard fields we populate:**

| Field | Type | Use |
|---|---|---|
| `Name` | Text | Provider display name (same as legal business name for orgs) |
| `AccountId` | Lookup(Account) | The NGO org record |
| `PractitionerId` | Lookup(Contact) | Primary contact for the provider |
| `ProviderType` | Picklist | Options: Onsite Service Provider, At Home Service Provider, Hospital, Ambulatory Care, Laboratory, Pharmacy, Dentist, Pharmacist, Medical Doctor. Example: use **Onsite Service Provider** or **At Home Service Provider**. |
| `ProviderClass` | Picklist | IPA, Medical Group, Solo Practitioner. NGO providers = **Medical Group**. |
| `Status` | Picklist | Active, Inactive, **Pending**. New submissions = **Pending** (awaits agency approval). |
| `InitialStartDate` | Date | `Date.today()` on submission |
| `EffectiveFrom` | Date | `Date.today()` |
| `Description` | Text Area | Service description |

**Standard fields to remove from layout** (not typically needed in a provider portal demo): `TotalLicensedBeds`, `EhrSystem`, `TerminationReason`, `SourceSystem`, `SourceSystemIdentifier`, `EffectiveTo`, `TerminationDate`, `ReferredByContactId`, `CaqhIdentifier`, `IsNotSearchable`.

**Custom fields to add to Account** (not on HealthcareProvider — data model separation is deliberate):
- `Trading_Name__c` — Text(255)
- `ABN__c` — Text(11)
- `ACN__c` — Text(9)

**Custom field on HealthcareProvider:**
- `Services_Offered__c` — Long Text Area(1000)

**All address / phone / website fields** live on Account (standard: `BillingStreet`, `BillingCity`, `BillingState`, `BillingPostalCode`, `Phone`, `Website`). No duplication onto HealthcareProvider.

**Contact:** standard fields only (`FirstName`, `LastName`, `Email`, `Phone`) + `AccountId` linked to the same Account.

---

## The OmniScript Structure (verified working)

Type: **Provider** (no underscore — API rejects). SubType: **Onboarding**. Language: **English**.

**Structure (Level 0 = top-level, Level 1 = nested inside Step):**

| SeqNo | Level | Type | Name | Parent |
|---|---|---|---|---|
| 0 | 0 | Step | Step1LegalReg | — |
| 110 | 1 | Text | LegalName (required) | Step1LegalReg |
| 120 | 1 | Text | TradingName | Step1LegalReg |
| 130 | 1 | Text | ABN | Step1LegalReg |
| 140 | 1 | Text | ACN | Step1LegalReg |
| 1 | 0 | Step | Step2Address | — |
| 160-210 | 1 | Text / Telephone | Street, City, State, Postcode, OrgPhone, Website | Step2Address |
| 2 | 0 | Step | Step3Services | — |
| 230 | 1 | Text Area | ServicesOffered | Step3Services |
| 3 | 0 | Step | Step4Contact | — |
| 250-280 | 1 | Text / Email / Telephone | ContactFirstName, ContactLastName, ContactEmail, ContactPhone | Step4Contact |
| 4 | 0 | Step | Step5Review | — |
| **5** | **0** | **Remote Action** | **SubmitProvider** | **— (NOT nested in a Step)** |
| 6 | 0 | Step | Step6Confirmation (hidePrev + hideNext) | — |
| 320 | 1 | Text Block | ConfirmationMessage | Step6Confirmation |

**Critical**: The Remote Action must be a **standalone top-level element between the last Step and the confirmation Step**, not nested inside a Step. Otherwise it either renders a duplicate button OR doesn't advance to the next step.

---

## The Apex Class — Full, Reusable, Copy-Paste

```apex
/**
 * ProviderOnboardingService (reusable pattern for OmniScript → Multi-object save)
 *
 * MUST be `global` + implement `System.Callable`.
 * MUST accept args as `Map<String, Object>` (OmniStudio wraps input at args.input.AllData).
 * MUST return `Map<String, Object>` — throwing gives opaque "Script-thrown exception" on the client.
 */
global without sharing class ProviderOnboardingService implements System.Callable {

    global Object call(String action, Map<String, Object> args) {
        if (action == 'onboardProvider') {
            return onboardProvider(args);
        }
        return new Map<String, Object>{'success' => false, 'error' => 'Unknown action: ' + action};
    }

    private Map<String, Object> onboardProvider(Map<String, Object> args) {
        Map<String, Object> data = extractFormData(args);
        System.debug('Onboarding — form data keys: ' + data.keySet());
        System.debug('Onboarding — full form data: ' + JSON.serialize(data));

        String legalName        = str(data, 'LegalName');
        String tradingName      = str(data, 'TradingName');
        String abn              = str(data, 'ABN');
        String acn              = str(data, 'ACN');
        String street           = str(data, 'Street');
        String city             = str(data, 'City');
        String state            = str(data, 'State');
        String postcode         = str(data, 'Postcode');
        String orgPhone         = str(data, 'OrgPhone');
        String website          = str(data, 'Website');
        String servicesOffered  = str(data, 'ServicesOffered');
        String contactFirstName = str(data, 'ContactFirstName');
        String contactLastName  = str(data, 'ContactLastName');
        String contactEmail     = str(data, 'ContactEmail');
        String contactPhone     = str(data, 'ContactPhone');

        Map<String, Object> result = new Map<String, Object>();
        if (String.isBlank(legalName)) {
            result.put('success', false);
            result.put('error', 'Legal business name is required. Data keys received: ' + data.keySet());
            return result;
        }

        Savepoint sp = Database.setSavepoint();
        try {
            // Account (the NGO)
            Account acc = new Account(
                Name              = legalName,
                Trading_Name__c   = tradingName,
                ABN__c            = abn,
                ACN__c            = acn,
                BillingStreet     = street,
                BillingCity       = city,
                BillingState      = state,
                BillingPostalCode = postcode,
                Phone             = orgPhone,
                Website           = website
            );
            insert acc;

            // Primary Contact
            Contact con;
            if (String.isNotBlank(contactFirstName) || String.isNotBlank(contactLastName)) {
                con = new Contact(
                    AccountId = acc.Id,
                    FirstName = contactFirstName,
                    LastName  = String.isBlank(contactLastName) ? 'Unknown' : contactLastName,
                    Email     = contactEmail,
                    Phone     = contactPhone
                );
                insert con;
            }

            // HealthcareProvider (PSS)
            HealthcareProvider hcp = new HealthcareProvider(
                Name                = legalName,
                AccountId           = acc.Id,
                Services_Offered__c = servicesOffered,
                ProviderType        = 'Onsite Service Provider',
                ProviderClass       = 'Medical Group',
                Status              = 'Pending',
                InitialStartDate    = Date.today(),
                EffectiveFrom       = Date.today(),
                Description         = 'Example provider — allied health, youth counselling, early intervention, OT, speech pathology.'
            );
            if (con != null) hcp.PractitionerId = con.Id;
            insert hcp;

            result.put('success',              true);
            result.put('healthcareProviderId', hcp.Id);
            result.put('accountId',            acc.Id);
            result.put('contactId',            con == null ? null : con.Id);
            result.put('message',              'Provider onboarded. Awaiting agency approval. Reference: ' + hcp.Id);
        } catch (Exception e) {
            Database.rollback(sp);
            result.put('success', false);
            result.put('error',   e.getMessage());
            result.put('type',    e.getTypeName());
            result.put('stack',   e.getStackTraceString());
        }
        return result;
    }

    /**
     * OmniScript args are wrapped: { input: { AllData: { Step1: {...}, Step2: {...} } } }
     * This helper unwraps + flattens step-nested keys into a single top-level map.
     */
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

    private static String str(Map<String, Object> m, String key) {
        Object v = (m == null) ? null : m.get(key);
        return (v == null) ? null : String.valueOf(v);
    }
}
```

**Class meta.xml:**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<ApexClass xmlns="http://soap.sforce.com/2006/04/metadata">
    <apiVersion>62.0</apiVersion>
    <status>Active</status>
</ApexClass>
```

---

## The Remote Action PSC (verified working)

```json
{
  "name": "SubmitProvider",
  "label": "Submit Provider Application",
  "remoteClass": "ProviderOnboardingService",
  "remoteMethod": "onboardProvider",
  "remoteOptions": {},
  "remoteTimeout": 30000,
  "inputMap": [
    { "inputParam": "legalName",        "value": "%LegalName%" },
    { "inputParam": "tradingName",      "value": "%TradingName%" },
    { "inputParam": "abn",              "value": "%ABN%" },
    { "inputParam": "acn",              "value": "%ACN%" },
    { "inputParam": "street",           "value": "%Street%" },
    { "inputParam": "city",             "value": "%City%" },
    { "inputParam": "state",            "value": "%State%" },
    { "inputParam": "postcode",         "value": "%Postcode%" },
    { "inputParam": "orgPhone",         "value": "%OrgPhone%" },
    { "inputParam": "website",          "value": "%Website%" },
    { "inputParam": "servicesOffered",  "value": "%ServicesOffered%" },
    { "inputParam": "contactFirstName", "value": "%ContactFirstName%" },
    { "inputParam": "contactLastName",  "value": "%ContactLastName%" },
    { "inputParam": "contactEmail",     "value": "%ContactEmail%" },
    { "inputParam": "contactPhone",     "value": "%ContactPhone%" }
  ],
  "validationRequired": "Submit",
  "sendJSONNode": "AllData",
  "invokeMode": "RemoteAction",
  "responseJSONNode": "SubmitResult",
  "failureNextElement": "",
  "failureAbortLabel": "Abort",
  "failureGoBackLabel": "Go Back",
  "failureNextLabel": "Continue",
  "redirectNextLabel": "Next",
  "redirectPreviousLabel": "Previous"
}
```

**Note:** `sendJSONNode: "AllData"` is what makes OmniScript send the flat form data. The `inputMap` uses `%FieldName%` merge syntax — the field name is the raw element `Name`, not qualified by Step.

---

## Reusing for Another Program

To adapt this pattern for another program (Aged Care, Foster Care, Disability Services, etc.):

1. **Copy the Apex class**, rename e.g. `AgedCareProviderOnboardingService`, adjust `ProviderType/Class/Status` defaults.
2. **Adjust custom fields on Account/HealthcareProvider** if the domain needs different metadata (e.g. NDIS Registration Number).
3. **Rename OmniScript** Type/SubType (no underscores — PascalCase alphanumeric only).
4. **Adjust the confirmation copy** in the Step 6 Text Block.
5. **Adjust portal branding** (LWCs — swap <government> logo static resource for the relevant agency's).

The Callable pattern + args-unwrap + Level-0 Remote Action placement + Aura hosting is **the same for every version**. Don't re-litigate those.

---

## Related Skills

- `omnistudio-retrieve-and-modify` — OmniStudio retrieve-first workflow (sibling skill)
- `salesforce-experience-cloud-portal` — Aura + LWR Experience Cloud build guide (sibling skill)
- `salesforce-pss-icm` — reusable PSS Court & Investigative Case Management reference (sibling skill)

---

## Example artefact names

- **Org alias:** `<your-org-alias>`
- **Portal URL:** `https://<org-domain>.my.site.com/<url-prefix>/s/`
- **OmniScript:** `Provider / Onboarding / English`
- **Apex:** `ProviderOnboardingService` (global + Callable)
- **Permission Set:** `Provider_Portal_Access` (grants Apex + Account/Contact/HealthcareProvider CRUD)
- **PSL:** `Provider Management` (`ProviderManagementPsl`)
- **LWCs:** `portalHeader`, `portalHeroSection`, `portalContentCards`, `portalFooter`
- **Static resource:** `AgencyLogo` (agency SVG — swap per demo)
- **CMT:** `Portal_Config__mdt` with configurable fields (e.g. Google Maps API Key)
