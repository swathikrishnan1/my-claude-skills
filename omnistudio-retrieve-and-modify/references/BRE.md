# Business Rules Engine (BRE) — Build Guide

This file captures everything learned building BRE Lookup Tables and Expression Sets in Salesforce PSS orgs. Update this file every time you build or troubleshoot BRE components.

---

## Object Model

### Key Objects

| Object | Purpose |
|---|---|
| `DecisionMatrixDefinition` | The parent "Lookup Table" record — holds label, type, and column definitions |
| `DecisionMatrixDefinitionVersion` | The version of the matrix (V1, V2 etc) — holds column schema and status |
| `CalculationMatrixVersion` | The runtime version that holds actual row data. Child of `DecisionMatrixDefinitionVersion` via `DecisionMatrixDefinitionVerId` |
| `CalculationMatrixRow` | Each individual data row. Child of `CalculationMatrixVersion`. Stores `InputData` and `OutputData` as JSON strings |
| `CalcMatrixColumnRange` | Column range metadata (child of `CalculationMatrixVersion`) |

### Relationship Chain
```
DecisionMatrixDefinition
  └── DecisionMatrixDefinitionVersion (child: DecisionMatrixDefinitionVer)
        └── CalculationMatrixVersion (child: DecisionMatrixDefinitionVerId)
              └── CalculationMatrixRow (child: CalculationMatrixRows)
```

---

## Lookup Tables — Metadata (Structure)

### What deploys as metadata
Only the **structure** (column definitions) deploys via `DecisionMatrixDefinition` metadata. Row data does NOT deploy as metadata — it is versioned data loaded separately.

### Metadata file location
`force-app/main/default/decisionMatrixDefinition/[Name].decisionMatrixDefinition-meta.xml`

### Valid field values

| Field | Valid values |
|---|---|
| `processType` | `Bre` |
| `type` | `Standard` |
| `columnType` | `Input`, `Output` |
| `dataType` | `Text`, `Number`, `Currency`, `Boolean`, `Date`, `DateTime` — **NOT** `Decimal` (invalid) |
| `status` | `Active`, `Draft` |

### Required fields in version
- `fullName` — must match `[DeveloperName]_V1` pattern
- `label`
- `rank`
- `startDate` — **required**, deploy fails without it. Use ISO format: `2026-07-02T00:00:00.000Z`
- `status`
- `versionNumber`
- `decisionMatrixDefinition` — must match parent DeveloperName exactly
- At least one `columns` entry

### Rows are NOT metadata
`<rows>` elements inside the version block will cause: `Element rows invalid at this location in type DecisionMatrixDefinitionVersion`. Remove all `<rows>` from the XML before deploying.

### Example metadata file (structure only)
```xml
<?xml version="1.0" encoding="UTF-8"?>
<DecisionMatrixDefinition xmlns="http://soap.sforce.com/2006/04/metadata">
    <label>RF2 ACM Condition</label>
    <processType>Bre</processType>
    <type>Standard</type>
    <versions>
        <fullName>RF2_ACM_Condition_V1</fullName>
        <columns>
            <columnType>Input</columnType>
            <dataType>Text</dataType>
            <displaySequence>1</displaySequence>
            <isWildcardColumn>false</isWildcardColumn>
            <name>ACMCondition</name>
        </columns>
        <columns>
            <columnType>Output</columnType>
            <dataType>Number</dataType>
            <displaySequence>2</displaySequence>
            <isWildcardColumn>false</isWildcardColumn>
            <name>Score</name>
        </columns>
        <decisionMatrixDefinition>RF2_ACM_Condition</decisionMatrixDefinition>
        <label>RF2 ACM Condition V1</label>
        <rank>1</rank>
        <startDate>2026-07-02T00:00:00.000Z</startDate>
        <status>Active</status>
        <versionNumber>1</versionNumber>
    </versions>
</DecisionMatrixDefinition>
```

---

## Lookup Tables — Row Data

### How row data is stored
Each row is a `CalculationMatrixRow` record with:
- `CalculationMatrixVersionId` — ID of the parent `CalculationMatrixVersion`
- `InputData` — JSON string of input column name → value pairs
- `OutputData` — JSON string of output column name → value pairs
- `Name` — any unique string (use an MD5 hash or sequential name)

### Example row record
```json
{
  "CalculationMatrixVersionId": "0lNJ2000000XbV7MAK",
  "InputData": "{\"ProductRiskLevel\": \"Level 1 - Reinforced resins, plastics\"}",
  "OutputData": "{\"Score\": 3.5}",
  "Name": "rf1_r1"
}
```

### How to find the CalculationMatrixVersion ID
```soql
SELECT Id, Name, IsEnabled, LoadProcessStatus, DecisionMatrixDefinitionVerId
FROM CalculationMatrixVersion
WHERE Name LIKE '%RF%'
```

### Editing is blocked when version is active
Before inserting rows, **deactivate** the version:
```bash
sf data update record --sobject CalculationMatrixVersion --record-id [ID] --values "IsEnabled=false" --target-org "[alias]"
```
After inserting all rows, **reactivate**:
```bash
sf data update record --sobject CalculationMatrixVersion --record-id [ID] --values "IsEnabled=true" --target-org "[alias]"
```

### Inserting rows — use REST API, NOT sf data create
`sf data create record` cannot handle JSON values in `--values` flag — it throws `INVALID_INPUT: Enter input data in JSON format.`

Use the REST API directly:
```bash
ACCESS_TOKEN=$(sf org display --target-org "[alias]" --json | python3 -c "import json,sys; print(json.load(sys.stdin)['result']['accessToken'])")
INSTANCE="https://[your-instance].my.salesforce.com"

curl -s -X POST \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"CalculationMatrixVersionId":"[VERSION_ID]","InputData":"{\"ColumnName\":\"value\"}","OutputData":"{\"Score\":3.5}","Name":"row_name"}' \
  "$INSTANCE/services/data/v67.0/sobjects/CalculationMatrixRow"
```

> **CRITICAL — JSON formatting:** The UI stores `InputData` and `OutputData` as **pretty-printed JSON** with newlines and spaces (e.g. `{\n  "ACMCondition": "Stable"\n}`). API-inserted rows with compact JSON (e.g. `{"ACMCondition": "Stable"}`) appear to save but the BRE runtime does not match them at lookup time — steps return 0. Always use `json.dumps(data, indent=2)` when inserting via API.

### Python script pattern for bulk row insert
```python
import urllib.request, json

def insert(instance, token, version_id, name, input_data, output_data):
    body = json.dumps({
        "CalculationMatrixVersionId": version_id,
        "InputData": json.dumps(input_data),
        "OutputData": json.dumps(output_data),
        "Name": name
    }).encode()
    req = urllib.request.Request(
        f"{instance}/services/data/v67.0/sobjects/CalculationMatrixRow",
        data=body,
        headers={"Authorization": f"Bearer {token}", "Content-Type": "application/json"},
        method="POST"
    )
    with urllib.request.urlopen(req) as r:
        return json.loads(r.read()).get("success", False)
```

---

## End-to-End Checklist for a New Lookup Table

1. Create metadata file in `decisionMatrixDefinition/` with column structure only (no rows)
2. Ensure `startDate` is set, `dataType` uses `Number` not `Decimal`, and **`status` is `Draft`** — NOT `Active`
3. Deploy: `sf project deploy start --source-dir force-app/main/default/decisionMatrixDefinition --target-org "[alias]"`
4. Query the `CalculationMatrixVersion` ID for the deployed table
5. Insert rows via REST API (table is already inactive as Draft — no need to deactivate first)
6. **Activate the table manually in the UI** — Setup → Business Rules Engine → Lookup Tables → open the version → Activate
7. Verify rows are visible in the Matrix tab

> **CRITICAL:** Deploy as `Draft`, insert rows, THEN activate in the UI. If you deploy as `Active` first and then insert rows (even after deactivating via API and reactivating), the lookup steps in Expression Set simulations return 0. The table must be activated AFTER rows are loaded, and activation must be done via the UI not the API.

---

## Common Errors

| Error | Cause | Fix |
|---|---|---|
| `'Decimal' is not a valid value for the enum 'DecisionMatrixDataType'` | Used `Decimal` as dataType | Change to `Number` |
| `Element rows invalid at this location` | `<rows>` elements included in metadata XML | Remove all `<rows>` blocks — rows are data, not metadata |
| `Required field is missing: startDate` | Missing `<startDate>` in version | Add `<startDate>2026-07-02T00:00:00.000Z</startDate>` |
| `INVALID_INPUT: Enter input data in JSON format` | Using `sf data create record` with JSON fields | Use REST API directly instead |
| Version cannot be edited | Version is active (`IsEnabled=true`) | Deactivate first, insert rows, reactivate |
| Lookup steps return 0 in ES simulation | Table was deployed as `Active` before rows were inserted | Delete and recreate as `Draft`, insert rows, then activate in UI |

---

## Decision Explainer — Full Setup

### Overview
Decision Explainer shows a log of ES execution results on a record page via a related list component. It requires:
1. An `ApplicationSubtypeDefinition` (max 10 char DeveloperName)
2. An `ExplainabilityActionDef` linked to the subtype and a `BusinessProcessTypeDef`
3. An `ExplainabilityActionVersion` (active, child of the def)
4. `ExplainabilityMsgTemplate` records — one per step, per result type (Passed/Failed)
5. `ShouldShowExplExternally: true` on the ES version
6. Two extra `inputParameters` on the `runExpressionSet` flow action call
7. The Decision Explainer component added to the Lightning Record Page in App Builder

### Step 1 — Create via Tooling API

```python
# ApplicationSubtypeDefinition (DeveloperName max 10 chars)
POST /tooling/sobjects/ApplicationSubtypeDefinition
{
  "DeveloperName": "RiskAssess",       # max 10 chars
  "MasterLabel": "Risk Assessment Risk Assessment",
  "Language": "en_US",
  "ApplicationUsageType": "ExplainabilityService"
}

# ExplainabilityActionDef
POST /tooling/sobjects/ExplainabilityActionDef
{
  "DeveloperName": "RiskAssess",       # use same name for simplicity
  "MasterLabel": "Risk Assessment Risk Assessment",
  "Language": "en_US",
  "ApplicationType": "PublicSector",
  "ApplicationSubtypeId": "<subtype_id>",
  "ProcessTypeId": "<business_process_type_def_id>",  # get from tooling query on BusinessProcessTypeDef
  "ActionLogSchemaType": "ExpressionSet",
  "IsInternal": false
}

# ExplainabilityActionVersion
POST /tooling/sobjects/ExplainabilityActionVersion
{
  "DeveloperName": "RiskAssess_V1",
  "MasterLabel": "Risk Assessment V1",
  "Language": "en_US",
  "ExplainabilityActionDefId": "<def_id>",
  "ActionSpecification": "{}",
  "IsActive": true,
  "DefinitionVersion": 1
}
```

> **Note:** The `__explainabilitySpecName` in the flow must match the `DeveloperName` of the `ExplainabilityActionDef` exactly — e.g. `RiskAssess`.

### Step 2 — ExpressionSetVersion flag

Set `ShouldShowExplExternally = true` on the `ExpressionSetVersion` record. Can be done via REST API:
```bash
PATCH /sobjects/ExpressionSetVersion/<id>
{"ShouldShowExplExternally": true}
```
Or check the ES in the BRE UI — it surfaces as "Show in Decision Explainer" at the version level.

### Step 3 — Explainability Message Templates

Create one template per step per result type via Setup → Business Rules Engine → Explainability Message Templates (or REST API on `ExplainabilityMsgTemplate`).

Key fields:
- `DeveloperName` — unique name
- `MasterLabel` — display label
- `ExpressionSetStepType` — `Calculation` or `Condition`
- `ResultType` — `Passed` or `Failed`
- `EmtUsageType` — `Bre`
- `IsDefault` — `false` (unless you want it as default for all steps of that type)
- `Message` — the explanation text shown in the related list

Example (Risk Assessment):
```
Risk_Assessment_Risk_Score     | Calculation | Passed | Bre | "The Risk Calculation has been completed based on RF1-4 from the Lookup Tables"
Risk_Assessment_Risk_ScoreFail | Calculation | Failed | Bre | "The Risk Calculation was not completed due to missing or Incorrect information"
```

Then in the ES designer, open each Calculation/Condition step → Decision Explainer section → check **Show decision explanation** → assign the template for "When Step Returns Output" and "When Step Errors".

### Step 4 — Flow inputParameters for runExpressionSet

Two extra params are required on the `runExpressionSet` action call — these must use `inputConfiguratorMode: Resource` in the XML (set via UI, not hardcoded text):

```xml
<inputParameters>
    <name>__explainabilitySpecName</name>
    <value>
        <inputConfiguratorMode>Resource</inputConfiguratorMode>
        <stringValue>RiskAssess</stringValue>   <!-- DeveloperName of ExplainabilityActionDef -->
    </value>
</inputParameters>
<inputParameters>
    <name>__actionContextCode</name>
    <value>
        <elementReference>Get_Risk_Register_Item.Id</elementReference>
        <inputConfiguratorMode>Resource</inputConfiguratorMode>
    </value>
</inputParameters>
```

> **CRITICAL:** `__explainabilitySpecName` must be set as a **Resource** (not literal text) in the Flow Builder UI, otherwise it shows as Missing. Do NOT type the value directly into the text box — use the resource picker and select the string value. The `__actionContextCode` links the explanation log to the parent record (the Risk Register Item Id).

> **CRITICAL:** The other ES input parameters (e.g. `RiskRegisterItem.ProductRiskLevel`) must also use the Resource picker in the UI — referencing `{!Get_Risk_Register_Item.Product_Risk_Level__c}` etc. If typed directly as text they show as Missing even though they work at runtime.

### Step 5 — Lightning Record Page

Add the **Decision Explainer** component from the Components panel in Lightning App Builder onto the record page. No configuration needed — it automatically shows explanation logs linked via `__actionContextCode`.

---

## Calling an Expression Set from a Screen Flow

### Correct pattern — `runExpressionSet` with named input parameters

Use `actionType: runExpressionSet`. Do NOT use `calculateExpressionSetWithObjAlias` (not valid).

#### Key rules
1. **Get the record first** via a `recordLookups` step before calling the ES — do not rely on `{!$Record}` in a Quick Action flow, it is not reliably populated
2. **Pass each field as a named input parameter** using the ES Object Alias variable name (e.g. `RiskRegisterItem.ProductRiskLevel`) — do NOT use the `__buildContext` pattern
3. **Use `nameSegment` + `storeOutputAutomatically=true`** — do not set `versionSegment` or `versionNumber`
4. **Reference outputs** directly as `{!ActionCallName.OutputVariableName}` (e.g. `{!Run_Expression_Set.FinalRiskScore}`)

#### Flow XML pattern
```xml
<!-- Step 1: Get the record -->
<recordLookups>
    <name>Get_Record</name>
    <label>Get Record</label>
    <filterLogic>and</filterLogic>
    <filters>
        <field>Id</field>
        <operator>EqualTo</operator>
        <value><elementReference>recordId</elementReference></value>
    </filters>
    <getFirstRecordOnly>true</getFirstRecordOnly>
    <object>Risk_Register_Item__c</object>
    <storeOutputAutomatically>true</storeOutputAutomatically>
    <connector><targetReference>Run_Expression_Set</targetReference></connector>
</recordLookups>

<!-- Step 2: Call the ES -->
<actionCalls>
    <name>Run_Expression_Set</name>
    <actionName>Risk_Assessment_Final_Risk_Score_Calculation</actionName>
    <actionType>runExpressionSet</actionType>
    <inputParameters>
        <name>RiskRegisterItem.ProductRiskLevel</name>
        <value><elementReference>Get_Record.Product_Risk_Level__c</elementReference></value>
    </inputParameters>
    <!-- repeat for each input field -->
    <nameSegment>Risk_Assessment_Final_Risk_Score_Calculation</nameSegment>
    <storeOutputAutomatically>true</storeOutputAutomatically>
    <connector><targetReference>Update_Record</targetReference></connector>
</actionCalls>

<!-- Step 3: Update fields with ES output -->
<recordUpdates>
    <name>Update_Record</name>
    <filterLogic>and</filterLogic>
    <filters>
        <field>Id</field>
        <operator>EqualTo</operator>
        <value><elementReference>recordId</elementReference></value>
    </filters>
    <inputAssignments>
        <field>Final_Risk_Score__c</field>
        <value><elementReference>Run_Expression_Set.FinalRiskScore</elementReference></value>
    </inputAssignments>
    <inputAssignments>
        <field>Building_Rating__c</field>
        <value><elementReference>Run_Expression_Set.BuildingRating</elementReference></value>
    </inputAssignments>
    <object>Risk_Register_Item__c</object>
</recordUpdates>
```

### Expression Set variable requirements for flow integration

For ES outputs to be accessible in a flow via `storeOutputAutomatically`, each output variable in the ES **must have "Include in Output" enabled**. This is set on the **Calculation step** (not the Resource Manager), via the step's Output Variables section → "Include in Output" toggle on each variable. Variables with `output=false` are invisible to the flow caller even though they appear in ES simulations.

### Object Alias variable naming in ES

When an ES uses an Object Alias (e.g. `RiskRegisterItem` aliasing `Risk_Register_Item__c`), input field references inside the ES use the pattern `ObjectAliasName.FieldName` — e.g. `RiskRegisterItem.ProductRiskLevel`. These same names are used as the `<name>` of `<inputParameters>` in the flow's `actionCalls` element.

### Input/Output mapping on `GetOutputsFromDecisionMatrix` steps

When the ES input comes from an Object Alias variable, you **must** explicitly map:
- **Input**: the Object Alias field → the lookup table's input column variable
- **Output**: the lookup table's output column → a local ES variable (e.g. `RF1_Product_Risk_Level__Score`)

Without explicit output mapping, the lookup step runs but its result is discarded. Without explicit input mapping, the step always returns 0 (no match).

---

## LWC — Displaying BRE Scores Visually

### Key pattern: derive display state from the numeric score, not a status field

When building an LWC scorecard that shows BRE output (e.g. a risk score), always derive the colour/label from the **numeric score field** directly — not from a picklist status field like `Risk_Status__c`. The status field is only updated when the flow runs; if it's stale, the LWC will show the wrong colour even though the score is correct.

```js
get riskLevel() {
    const score = this._score;
    if (score >= 69) return 'HIGH';
    if (score > 50) return 'MEDIUM';
    return 'LOW';
}
get bannerClass() {
    const score = this._score;
    if (score >= 69) return 'banner--high';   // red
    if (score > 50)  return 'banner--medium'; // amber
    return 'banner--low';                     // green
}
```

### Auto-refresh via LDS

Wire `@wire(getRecord)` with all displayed fields including the score field. LDS will automatically refresh the component when the flow updates the record — no manual refresh or pubsub needed.

```js
@wire(getRecord, { recordId: '$recordId', fields: FIELDS })
record;
```

### Card grid alignment

When RF cards are side-by-side and titles have different lengths (some wrap to 2 lines, some don't), `.rf-value` elements will misalign vertically. Fix: set `min-height` on `.rf-header` so all value rows start at the same level:

```css
.rf-header {
    min-height: 2.4rem;
    align-items: flex-start;
}
```

---

## Risk Assessment — Lookup Tables Built (2026-07-02)

| Table | DeveloperName | Input Column | Output Column | Rows |
|---|---|---|---|---|
| RF1 Product Risk Level | `RF1_Product_Risk_Level` | `ProductRiskLevel` | `Score` | 10 |
| RF2 ACM Condition | `RF2_ACM_Condition` | `ACMCondition` | `Score` | 4 |
| RF3 Disturbance Potential | `RF3_Disturbance_Potential` | `DisturbancePotential` | `Score` | 4 |
| RF4a Public Access | `RF4a_Public_Access` | `PublicAccess` | `Score` | 2 |
| RF4b Frequency of Use | `RF4b_Frequency_of_Use` | `FrequencyOfUse` | `Score` | 6 |
| RF4c Daily Duration | `RF4c_Daily_Duration` | `DailyDuration` | `Score` | 5 |
| RF4d Level of Activity | `RF4d_Level_of_Activity` | `LevelOfActivity` | `Score` | 5 |
| RF4e Mobile Plant | `RF4e_Mobile_Plant` | `MobilePlant` | `Score` | 2 |
