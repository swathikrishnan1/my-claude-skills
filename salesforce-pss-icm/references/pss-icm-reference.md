# PSS ICM — Reference Knowledge Base

**Purpose:** Reusable reference for any Salesforce PSS Courts, Tribunals, or Investigative Case Management demo build. Covers data model, typical baseline org inventory, OmniStudio patterns, and architecture patterns. Load this file at the start of any PSS ICM build.

**Sister references (separate skills):**
- `salesforce-pss-provider-portal` — reusable pattern for building an NGO / provider self-onboarding portal on PSS + Aura + OmniScript. Covers Provider Management PSL, HealthcareProvider data model, Callable Apex, Remote Action placement, and the "Build Your Own Aura" portal recipe.

---

## 1. PSS Court & Investigative Case Management — Data Model

Based on Salesforce PSS official data model diagram.

### Core Objects

| Object | API Name | Purpose |
|---|---|---|
| Case | `Case` | Core matter record — all court types |
| Public Complaint | `PublicComplaint` | Citizen-facing lodgement / complaint intake |
| Complaint Case | `ComplaintCase` | Junction: links Public Complaint to Case |
| Complaint Participant | `ComplaintParticipant` | Parties on a complaint before it becomes a case |
| Case Participant | `CaseParticipant` | All parties to a matter (applicant, respondent, legal rep, witness) |
| Case Proceeding | `CaseProceeding` | Hearings, directions, mentions, conciliation sessions |
| Case Proceeding Participant | `CaseProceedingParticipant` | Who appeared at a specific hearing + their role |
| Case Proceeding Result | `CaseProceedingResult` | Outcomes, decisions, orders from a proceeding |
| Case Proceeding Infraction | `CaseProceedingInfraction` | Breaches/violations found at a proceeding |
| Case Proceeding Complaint | `CaseProceedingComplaint` | Complaints linked to a specific proceeding |
| Care Plan | `CarePlan` | Post-hearing orders, compliance actions, support plans |
| Case Episode | `CaseEpisode` | Discrete phases/episodes within a matter |
| Regulatory Code Violation | `RegulatoryCodeViolation` | Specific breach of a regulatory code / legislation |
| Violation Enforcement Action | `ViolationEnforcementAction` | Enforcement orders, directions, penalties |
| Regulatory Transaction Fee | `RegulatoryTransactionFee` | Filing fees, fines, bond amounts, penalties |
| Custody Item | `CustodyItem` | Physical or digital evidence items |
| Custody Chain Entry | `CustodyChainEntry` | Chain of custody log — who handled evidence, when |
| Custody Item Relation | `CustodyItemRelation` | Links evidence to case/proceeding |
| Custody Item Regulatory Code Violation | `CustodyItemRegulatoryCodeViolation` | Evidence tied to a specific violation |
| Visit / Inspection Visit | `Visit` | Site inspections, property visits |
| Assessment | `Assessment` | Structured assessments (eligibility, risk, social) |
| Location | `Location` | Physical locations (courtrooms, addresses) |
| Address | `Address` | Address records |
| Business License | `BusinessLicense` | Licences and regulatory authorisations |
| Person (Account) | `Account` (IsPersonAccount=true) | Individual parties — applicants, respondents |
| Contact | `Contact` / `PersonContactId` | Contact records; for Person Accounts use PersonContactId |
| Household | `Account` (Household RT) | Household grouping |

### Key Relationships

```
PublicComplaint ──(ComplaintCase)──► Case
Case ──► CaseParticipant (multiple, by role)
Case ──► CaseProceeding (multiple hearings)
CaseProceeding ──► CaseProceedingParticipant
CaseProceeding ──► CaseProceedingResult
CaseProceeding ──► CaseProceedingInfraction
Case ──► RegulatoryCodeViolation
RegulatoryCodeViolation ──► ViolationEnforcementAction
Case ──► CustodyItem (via CustodyItemRelation)
CustodyItem ──► CustodyChainEntry (multiple log entries)
CustodyItem ──► CustodyItemRegulatoryCodeViolation
Case ──► CarePlan (post-hearing actions)
Case ──► Case (ParentId — appeal hierarchy)
```

---

## 2. example court & Tribunal Structure

### Hierarchy
1. **Supreme Court** — highest court, appellate + original jurisdiction
2. **Magistrates Court** — summary criminal + civil matters AUD 25k–250k
3. **civil and administrative tribunal** — civil disputes under AUD 25k, tenancy, discrimination, guardianship, government decision review

### Magistrates Court Divisions
- Children's Court
- Coroner's Court
- Family Violence Court
- Industrial Court

### civil tribunal Matter Types (18+)
Civil disputes, tenancy, energy & water, discrimination, guardianship, mental health, motor accident injuries, occupational regulation, retirement villages, unit titles, government decision review, fence disputes, voluntary assisted dying, fair trading

### Appeal Pathways
- civil tribunal → Supreme Court (limited grounds)
- Magistrates Court → Supreme Court (single judge)
- Supreme Court → Court of Appeal (panel of 3 Supreme Court judges)
- Court of Appeal → higher appellate court

### Key Legislation
- [Records Act — jurisdiction-specific] 
- [Tribunal Act — jurisdiction-specific]
- [Tenancies Act — jurisdiction-specific] 
- [Magistrates Court Act — jurisdiction-specific] 
- [Supreme Court Act — jurisdiction-specific] 
- [Court Procedures Act — jurisdiction-specific] 
- [Privacy Act — jurisdiction-specific] 

### Physical Infrastructure
All courts commonly housed in a single building (example layout):
- 11 Magistrate courtrooms (left side)
- 8 Supreme Court rooms (right side)
- Children's Court (separate entrance)
- Single administration: central Courts & Tribunal Administration

---

## 3. Existing Org Metadata — example inventory from a working PSS ICM demo org

**Org alias:** `<your-org-alias>`
**Namespace:** Core (OmniProcess, OmniDataTransform, OmniUiCard — no vlocity prefix)

### OmniScripts — Courts-Relevant (active unless noted)

| Name | Type / SubType | Status | Notes |
|---|---|---|---|
| Create New Court Case | CourtCaseManagement / CreateNewCourtCase | Active | Core case creation — reuse |
| Create New Court Case Proceeding | CourtCaseManagement / CreateNewCourtCaseProceeding | Active | Hearing creation — reuse |
| PoliceComplaintIntake | CourtCaseManagement / PoliceComplaintIntake | Active (multiple versions) | Base for citizen intake — reskin |
| Complaint Intake | CaseManagement / ComplaintIntake | Active | General complaint pattern |
| Case Proceeding Deferral | CaseProceeding / ServiceRequest | Active | Adjournment/deferral |
| Create New Investigation | InvestigativeCaseManagement / CreateNewInvestigativeCase | Active | Investigation creation |
| Property Owner Interrogation | InvestigativeCaseManagement / PropertyAssessment | Active | Assessment pattern |
| Search Property | InvestigativeCaseManagement / PropertySearch | Active | Search pattern |
| Witness Interviews | Investigation / Safety | Active | Interview capture |
| Check-in Assessment | YouthJustice / Assessment | Active | Assessment pattern |
| DocGenComponent | DocGen / Component | Active | DocGen trigger — reuse |
| SignatureCapture | ChangeOfCircumstances / SignatureCapture | Active | Digital signature — reuse as-is |

**No existing OmniScript for:** Evidence/Custody capture, Copy Participants to Proceeding, Fine/Bond calculation, Participant-centric view

### Integration Procedures — Courts-Relevant

| Name | Type / SubType | Notes |
|---|---|---|
| CaseProceedingExtension | CaseProceeding / ServiceRequest | Proceeding extensions |
| IPGetDocumentTemplateDetailsForDocGen | Document / Generation | DocGen template lookup |
| DocumentServiceGateway_DocGeneration | DocumentServiceGateway / DocGeneration | Core DocGen |
| DocumentServiceGateway_DocGenerationPDF | DocumentServiceGateway / DocGenerationPDF | PDF generation — use for orders |
| DocumentServiceGateway_DocGenerationWithTokenData | DocumentServiceGateway / DocGenerationWithTokenData | Secure DocGen |
| ResusableDocuSignSendEnvelope | Document / ResusableDocuSignSendEnvelope | DocuSign integration |
| sfiArchResusableDocuSignSendEnvelope | sfriArch / ResusableDocuSignSendEnvelope | Newer DocuSign variant |
| mergepdfVersionIds | Merge / VersionIds | PDF merging |
| LPI Generate Signature Files | LPI / GenerateSignatureFiles | Signature file generation |
| Search for a Client | — | Party search |
| Search for a Referrer | — | Agent/rep search |
| Attach Files to a Referral | — | File attachment |
| FetchPersonalInformation | BenefitChangeOfCircumstance / FetchPersonalInformation | Personal info prefill |
| IPRegulatoryFeeCalculator | Fee / Calculator | Fee/fine calculation |
| PSSServiceExcellenceAlertsGet | — | Alerts/notifications |

### DataRaptors — Courts-Relevant (activate before use)

| Name | Status | Notes |
|---|---|---|
| CCMDRGetCaseDetailsForCaseProceeding | Check | Case details for proceeding |
| CCMDRGetCaseParticipants | Check | All case participants |
| CCMDRGetComplaintCase | Check | Fetch complaint case |
| CCMDRGetPublicComplaint | Check | Fetch public complaint |
| CCMDRGetUserDetails | Check | User details |
| CCMDRGetComplaintDetailsForCase | Inactive — activate | Complaint details |
| CCMDRGetComplaintParticipants | Inactive — activate | Complaint participants |
| CCMDRGetRecordTypeId | Check | Record type utility |
| DRGetAccountAndContactForUser | Check | Account + contact for logged-in user |
| DRGetContactDetailsforUser | Check | Contact details |
| DRGetUserInfo | Check | User info |
| PSSCaseworkOverviewGetCaseProceedingsIDO | Inactive — activate | All proceedings |
| PSSCaseworkOverviewGetCaseProceedingResultIDO | Inactive — activate | Proceeding outcomes |
| PSSCaseworkOverviewGetCaseProceedingInfractionsIDO | Inactive — activate | Infractions |
| PSSCaseworkOverviewGetCaseParticipantsIDO | Inactive — activate | Participants per proceeding |
| DocGenSampleExtractDocumentTemplatesLWC | Check | DocGen template list |
| ExtractDocumentTemplateByTemplateType | Check | Template fetch by type |
| LPIGetSignatureFilesInspectionReport | Check | Signature files |
| GetComplaintDetailsForCasePrepopulation | Check | Pre-populate case from complaint |

### FlexCards — Courts-Relevant

| Name | Status | Notes |
|---|---|---|
| CaseParticipants | Active | Party list — reuse |
| ComplaintParticipants | Active | Complaint parties |
| PSSCaseworkOverviewShowCaseProceedingsIDO | Active | Proceedings timeline — core 360° component |
| PSSCaseworkOverviewShowCaseProceedingDetailsIDO | Active | Proceeding detail |
| PSSCaseworkOverviewShowCaseProceedingParticipantsIDO | Active | Participants per proceeding |
| PSSCaseworkOverviewShowCaseProceedingParticipantsInDataTableIDO | Active | Tabular participants |
| PSSCaseworkOverviewShowCaseProceedingResultsIDO | Active | Proceeding outcomes |
| PSSCaseworkOverviewShowCaseProceedingResultsInDataTableIDO | Active | Tabular outcomes |
| PSSCaseworkOverviewShowCaseProceedingInfractionsIDO | Active | Infractions per proceeding |
| PSSCaseworkOverviewShowCaseProceedingsInDataTableIDO | Active | Tabular proceedings |
| PSSCaseworkOverviewShowEmptyStateIDO | Active | Empty state |
| PSSServiceExcellenceAlertCard | Active | Alerts/deadlines |
| PSSServiceExcellenceGroupedAlert | Active | Grouped alerts |
| PSSServiceExcellenceSingleAlert | Active | Single alert |
| SearchParticipants | Active | Participant search |
| LaunchFundingAwardDocGen | Active | DocGen launcher pattern |

### Existing Portal Users (reuse for demos)

| User | ID | Contact | Type | Profile | Persona (first name or role only — do NOT use real last names) |
|---|---|---|---|---|---|
| Portal User A | [USER_ID] | [CONTACT_ID] | PowerCustomerSuccess | SDO-Customer Community Plus | Citizen (Citizen) |
| Portal User B | [USER_ID] | [CONTACT_ID] | PowerCustomerSuccess | SDO-Customer Community Plus | Lawyer (Lawyer) |
| Portal User C | [USER_ID] | [CONTACT_ID] | PowerCustomerSuccess | SDO-Customer Community Plus | SC Officer (SC Officer) |
| Admin User | [USER_ID] | — | Standard | System Administrator | Judge/Tribunal Member — labelled "Judge" |
| Service User | [USER_ID] | — | Standard | SDO-Service | Registry Officer (Registry Officer) |

**Note:** Person Accounts are enabled in this org and portal login works with Person Accounts (confirmed via Test Person Account — [ACCOUNT_ID]). Safe to convert Portal User A and Portal User B contacts to Person Accounts.

**Justice Portal Profile** already exists (ID: [PROFILE_ID]) — use as base for citizen portal access.

---

## 4. OmniStudio Build Reference

### Component Order (always follow this sequence)
1. DataRaptors (Extract, Transform, Load, Turbo Extract)
2. Integration Procedures (server-side orchestration)
3. OmniScripts (guided UI)
4. FlexCards (presentation layer)

### Namespace — Core (this org)
```
OmniProcess (OmniScript + IP)
OmniDataTransform (DataRaptor)
OmniUiCard (FlexCard)
OmniProcessElement (elements within OmniScript/IP)
OmniDataTransformItem (items within DataRaptor)
```

### OmniScript Key Patterns

**Type/SubType/Language triplet** — uniquely identifies an OmniScript
- Type: PascalCase domain (e.g. `CourtCaseManagement`)
- SubType: PascalCase variant (e.g. `TenancyDisputeIntake`)
- Language: `English`

**Data prefill:** DataRaptor Extract Action as first element in Step, `executionConditionFormula` to skip re-fetch on back navigation

**Merge field syntax:** `%FieldName%` or `%StepName:FieldName%`

**Embedding:** OmniScript Action element, pass params via inputMap

**Save and resume:** enabled at OmniScript level — `allowSaveForLater: true` on Step

**Max elements per Step:** 7–10 (performance)

### DataRaptor Types
- **Extract** — read from Salesforce, SOQL-based
- **Turbo Extract** — high-volume reads (10x faster, no formula fields)
- **Transform** — in-memory JSON reshaping (no DML/callouts)
- **Load** — write to Salesforce (insert/update/upsert/delete)

**Naming convention:** `DR_[Type]_[Object]_[Purpose]`
Example: `DR_Load_TenancyDisputeCase`, `DR_Extract_CaseParticipants`

### DataRaptor Output Variable Naming — DRId Convention

When a **DataRaptor Load** creates a record, it outputs the new Salesforce record ID as `DRId_[ObjectName]`. This is a standard CCM naming convention baked into the pre-built DRs:
- Create a Case → outputs `DRId_Case`
- Create a PublicComplaint → outputs `DRId_PublicComplaint`
- Create a CaseParticipant → outputs `DRId_CaseParticipant`

Reference this in downstream elements using `%DRId_Case%` etc.

### OmniScript Confirmation Screen — Full Pattern

To show a created record's fields (e.g. CaseNumber) on a confirmation screen after Submit:

1. **DR Post Action** at Level 0, `validationRequired: Submit` — creates the record, outputs `DRId_Case`. Do NOT set `responseJSONNode` — causes a "true" error popup.
2. **DR Turbo Action** at Level 0, `validationRequired: Submit` — input param: `DRId_Case` → `caseId`. Set `responseJSONNode: CreatedCase`. Uses `GetCaseID` bundle.
3. **Confirmation Step** (Level 0, next Seq) — with Text Block child.
4. **Text Block** — reference case number as `%CreatedCase|1:CaseNumber%`

**Merge field syntax for DR Turbo Action response:** `%ResponseNode|arrayIndex:FieldName%`
- `|1` = first record in the response array
- If DR returns a flat object (not array), use `%ResponseNode:FieldName%` without the index

**Why `responseJSONNode` on Post Action causes "true" error:** A Load DR returns a boolean success flag, not JSON data. If `responseJSONNode` is set, OmniScript parses `true` as the response and displays it as an error. Always leave `responseJSONNode` empty on DataRaptor Post Actions.

### Integration Procedure Key Patterns

**OmniProcessType = `Integration Procedure`** (with space) — NOT `IntegrationProcedure`. Querying with `IsIntegrationProcedure=true` misses all records in this org. Correct SOQL: `WHERE OmniProcessType='Integration Procedure'`. 160+ IPs typically exist in a fully-baseline PSS ICM org.

**Response mapping:** `%ElementName:keyPath%` to reference upstream element output

**Caching:** `cacheType: Platform`, `cacheTTL: 3600` for read-heavy IPs

**Naming:** `Domain/Purpose` format following OmniProcess Type/SubType convention — e.g. Type=`CourtCaseManagement`, SubType=`CreateTenancyMatter`

**FlexCard VersionNumber required:** Must be a positive integer (e.g. `1`). Field is required on create.

**FlexCard AuthorName:** Must be alphanumeric + underscores only, no spaces, starts with letter. Use format like `ACT_CourtsDemo`.

### FlexCard Key Patterns

**Data source config field:** `DataSourceConfig` (NOT `Definition`)
**IP data source type:** `IntegrationProcedures` (plural, capital P)
**Merge field syntax:** `{datasource.fieldName}`, `{recordId}`
**Max nesting depth:** 2 levels

### CLI Commands
```bash
# List OmniScripts
sf data query --query "SELECT Id,Type,SubType,Language,IsActive FROM OmniProcess WHERE IsIntegrationProcedure=false" -o <alias>

# List IPs
sf data query --query "SELECT Id,Type,SubType,IsActive FROM OmniProcess WHERE IsIntegrationProcedure=true" -o <alias>

# List FlexCards
sf data query --query "SELECT Id,Name,IsActive FROM OmniUiCard" -o <alias>

# List DataRaptors
sf data query --query "SELECT Id,Name,Type,IsActive FROM OmniDataTransform" -o <alias>

# Deploy
sf project deploy start -m OmniScript:<Name> -o <alias>
sf project deploy start -m OmniIntegrationProcedure:<Name> -o <alias>
sf project deploy start -m OmniUiCard:<Name> -o <alias>
sf project deploy start -m OmniDataTransform:<Name> -o <alias>
```

---

## 5. Skills Reference (from GitHub)

### Jaganpro/sf-skills — OmniStudio Skills
Full OmniStudio reference covering DataMapper, Integration Procedure, OmniScript, FlexCard.
- Scoring: DataMapper 100pt, IP 110pt, OmniScript 120pt, FlexCard 130pt
- All thresholds: 90+ deploy, 67-89 review, <67 block
- Dependency order: Analyze → DataMapper → IP → OmniScript → FlexCard
- Repo: https://github.com/Jaganpro/sf-skills

### Weytani/sf-claude-skills — Salesforce Platform Reference
LWC, Apex, SOQL, Flows, Metadata API, SF CLI, SLDS2 reference skills.
- PICKLES framework for LWC
- Wire service patterns, event communication (CustomEvent, LMS, @api)
- Repo: https://github.com/weytani/sf-claude-skills

### Weytani/sf-skills — Extended Salesforce Skills
19+ skills, 150+ validation points, LSP engine, Code Analyzer V5.
- OmniStudio and Experience Cloud skills planned (not yet released)
- Repo: https://github.com/weytani/sf-skills

---

## 6. Capabilities Commonly Requested in Courts & Tribunals Demos

The following capabilities recur in courts & tribunals demos and should be covered in most PSS ICM builds:

### Civil Courts & Tribunals
- Dedicated data model (Employee, Job History, Degree, Commercial relationships, credentials, authorisations, contracts)
- Fine Management — Fines Calculator, Beneficiary Assignment
- Contract Lifecycle Management — Document Generation and Versioning Management

### Criminal Court
- Evidence Management — Physical and Digital Evidence, Document Management, Chain of Custody
- Detention & Probation — episodes, anti-recidivism
- Investigation & Expert Management

### Cross-cutting (all court types)
- Citizen & Relationship Data Model — Citizen Snapshot, Social Security Centric, Effective Dating
- Justice Case Centric View — visual tree (Proceedings → Participants → Complaints → Code Violations → Custody Relations) + stage path progress bar
- Justice Participant Centric View — Person Account with graphical 360 relationship tab, Action Launcher, Interaction Summaries, Prior Assessments
- Evidence Guided Flow OmniScript — Custody Item Details → Chain of Custody → Regulatory Code Decisions → Confirmation + preview/stream digital evidence
- AWS S3 External Storage — bi-directional link, upload from Salesforce UI, files stored in S3
- Complaint Intake Guided Flow — citizen self-service, evidence upload, confirmation
- Proceeding Management — Copy Participants from Case to Proceeding guided flow
- Asset Management — Unified Asset Record, link to Custody Item, maintenance plans

---

## 7. Case Status Lifecycle (civil tribunal)

Matches real civil tribunal process:
1. Lodged
2. Assessment (registry jurisdiction check)
3. Conciliation (parties invited to resolve before hearing)
4. Hearing Scheduled
5. Hearing in Progress
6. Decision Reserved
7. Order Made
8. Under Appeal
9. Closed
10. Withdrawn

---

## 8. Permission Architecture

### Permission Set Group pattern (recommended over individual permission sets)

| PSG | Role | Key Access |
|---|---|---|
| PSG_Citizen | Tenant / Complainant | Own matter, own docs, civil tribunal portal |
| PSG_Lawyer | Legal Representative | Client matters, shared orders, appeal lodgement, both portals |
| PSG_RegistryOfficer | Registry staff | All civil tribunal cases, correspondence, scheduling, DocGen |
| PSG_TribunalMember | Judge / Member | Assigned hearings, benchsheet, order generation |
| PSG_SupremeCourtOfficer | SC Officer | All cases + appellate layer, elevated document access |

### Record Types on Case
- `civil tribunal_Matter` — civil tribunal civil disputes
- `SupremeCourt_Appeal` — Supreme Court appeals

### OWD
- Case = Private
- Sharing rules layer access on top by role

---

## 9. Architecture Decisions

- **One org** — all courts and tribunals on single PSS org. Security by Record Type + Permission Sets, not separate systems
- **Standard Case object** — confirmed as the base object in this org (not a custom Matter object)
- **Person Accounts** — enabled and working with portal logins. Use for citizen and lawyer contacts
- **Actionable Relationship Centre (ARC)** — native Salesforce Industries Lightning Web Component, typically already installed in a PSS ICM org. Needs configuration (which objects/relationships to display, node types, relationship groups). Add to Lightning record page via App Builder. This is the graphical relationship map — distinct from and complementary to the PSSCaseworkOverview FlexCards. Do NOT rebuild this as a FlexCard — configure the native component.
- **DocGen** — infrastructure exists (DocumentServiceGateway IPs, DocGen OmniScript). No court-specific templates yet — build from LPI Inspection Report pattern
- **Signature** — `ChangeOfCircumstances / SignatureCapture` OmniScript exists and works — embed in benchsheet OmniScript
- **S3 External Storage** — configure separately if available; fallback to Salesforce Files + CustodyItem link
- **SCV (Service Cloud Voice)** — separate org, smoke-and-mirror into main demo with matching data

---

## 10. Salesforce PSS Data Model Reference

Official documentation: https://developer.salesforce.com/docs/platform/data-models/guide/public-sector-solutions-category.html

Key page: PSS Court & Investigative Case Management (ICM) data model diagram

---

## 12. OmniScript on LWR Experience Cloud — Critical Gotchas (2026-06-29)

### Runtime Type: Standard vs Managed
This org runs **OmniStudio Core Standard Runtime** (no vlocity/managed package namespace).
- `WebComponentKey` is **always null** on standard runtime — this is CORRECT and expected. Do NOT chase it.
- On standard runtime, the LWC bundle is generated dynamically by the platform, not stored as a keyed reference.
- `IsActive: true` is the only field that matters for standard runtime deployment.

### Creating OmniScripts via API — Element IsActive gotcha
Elements created via REST API default to `IsActive: false`. The OmniScript designer and compile cycle ignore inactive elements — canvas shows blank, no steps visible.
**Fix:** After creating all `OmniProcessElement` records via API, PATCH each one to `IsActive: true` before opening in designer.

### Creating OmniScripts via API — Correct approach
1. POST `OmniProcess` with `IsActive: false`, `OmniProcessType: "OmniScript"`
2. POST all `OmniProcessElement` records with `IsActive: true` in level order (Level 0 first, then 1, 2, 3)
3. Set `ParentElementId` using IDs returned from Level 0/1 creates
4. DO NOT include `IsOmniScriptEmbeddable` in POST payload — read-only field, causes `INVALID_FIELD` error
5. Use `PropertySetConfig` (not `PropertySet` — doesn't exist)
6. Deactivate via API before patching PropertySetConfig or elements
7. Open in OmniScript designer UI and click Activate

### Valid OmniProcessElement Type values (confirmed in a PSS ICM Core-namespace org)
`Step`, `Block`, `Text`, `Text Area` (with space), `Select`, `Radio`, `Telephone`, `Email`, `Currency`, `Checkbox`, `Date`, `File`, `Headline`, `Display Text` — NOT `TextArea` (no space), NOT `TextBlock` for display

### OmniScript metadata deploy via sf project deploy
- Requires `<name>`, `<uniqueName>`, `<type>`, `<subType>`, `<language>`, `<isActive>`, `<omniProcessType>`, `<propertySetConfig>` in `.os-meta.xml`
- Does NOT deploy child elements — elements must be created separately via REST API after deploy
- Retrieve via `sf project retrieve start --metadata "OmniScript:<Name>"` often fails for Core namespace — use API directly

### Deactivating OmniScript via API
Always deactivate (`IsActive: false`) before patching `PropertySetConfig` or element records. Active OmniScript records reject PATCH with `FIELD_INTEGRITY_EXCEPTION`.

### LWR portal embedding — Standard Runtime
Use `<lightning-omnistudio-omniscript>` tag (NOT `<omnistudio-omniscript>` — not resolvable as LWC module in Core namespace orgs).
Required attributes: `type`, `sub-type`, `language`, `theme="lightning"`, `lwr`
Wrap in a custom LWC with targets `lightningCommunity__Default` and `lightningCommunity__Page`.
`experience__AppPage` / `experience__HomePage` / `experience__RecordPage` targets are NOT valid in this org's API version.

### LWR3008 error — `false` in bundle path
Error path `omniScript_...__false__...` means `IsWebCompEnabled=false` was baked into the page at publish time.
Fix: Set `IsWebCompEnabled=true` via REST (deactivate first), reactivate via UI, then republish the page.
On standard runtime this flag must be `true` for LWR to resolve the dynamic bundle path correctly.

### CRITICAL — Must publish via Experience Builder UI, not Connect API
The Connect API publish (`POST /connect/communities/$ID/publish`) does NOT trigger the LWR static asset compiler for OmniScript component paths. Even with `IsWebCompEnabled=true` and the LWC wrapper deployed, the page will still serve a stale/broken bundle path until the Experience Builder UI Publish button is used.

**Correct end-to-end sequence for OmniScript on LWR:**
1. Create OmniProcess + elements via API — do NOT touch `IsWebCompEnabled`
2. Open in OmniStudio designer → click **Activate** (sets flag + compiles + generates key atomically)
3. Deploy LWC wrapper with hardcoded `lightning-omnistudio-omniscript` + `lwr` attribute
4. Open **Experience Builder** → drag wrapper component onto canvas → click **Publish** (UI button)
5. Wait 2–3 minutes → hard-refresh browser (Cmd+Shift+R)

---

## 11. Data Creation Patterns (API / REST — confirmed working)

### CustodyItem
- `ReferenceItemRecordId` does NOT accept Case — it's a restricted polymorphic field. Create CustodyItem standalone, then link via `CustodyItemRelation`.
- `CustodyItemRelation`: use `ContextId` (not `CaseId` — that field is read-only on insert). Status values: `New`, `Qualified`, `Disqualified`.
- `CustodyChainEntry`: `CustodianId` accepts User, Contact, Account, ServiceTerritory.

### RegulatoryCodeViolation
- `Name` field is **auto-generated** — do NOT include it in POST payload (causes insert error).
- Required fields on insert: `RegulatoryCodeId`, `ViolationTypeId`, `Status`, `Priority`, `DateCreated`.
- `ResponseContextId` — use this to link to the Case (civil tribunal matter). This is how the violation appears in Justice Case Centric View Casework Overview tree.
- `ViolationType` requires `Type` (Health and Safety / Electrical / Building) and `Severity` (Issue / Minor Violation / Major Violation).

### RegulatoryCode
- `EffectiveFrom` is **required** — always include it.
- `IsActive` is **read-only on insert** — omit it (defaults to true).
- Create hierarchy: Title → Chapter → Section using `ParentCodeId`.

### CustodyItemRgltyCodeVio (junction: evidence → violation)
- Fields: `CustodyItemId`, `RegulatoryCodeViolationId`, `Status`, `Description`.
- Status values mirror CustodyItem: `New`, `Qualified`, `Disqualified`.

### Case — linking matters
- Use standard `ParentId` to link a Supreme Court appeal to its parent civil tribunal matter.
- `CourtLevel__c` custom picklist: civil tribunal / Magistrates Court / Supreme Court — drives visibility and portal display.

### PermissionSet FLS for new custom fields
- New custom fields are NOT visible to any profile by default (even System Admin in enhanced profile orgs).
- Deploy a PermissionSet with `<fieldPermissions>` entries for each new field, then assign to all relevant users.
- Without this, SOQL queries on new fields return INVALID_FIELD even though Tooling API shows them as deployed.

### Session token expiry
- Tokens expire during long builds. Always refresh with: `sf org display --target-org "<your-org-alias>" --json | python3 -c "import sys,json; print(json.load(sys.stdin)['result']['accessToken'])"`
