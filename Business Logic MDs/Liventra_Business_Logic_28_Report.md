# Liventra — Business Logic Documentation
## Model 28: `Report`

**Project:** Liventra — Smart Livestock Farm Management System  
**Module:** Reporting & Analytics  
**Collection Status:** Final

> This document follows the finalized Liventra DB model. The finalized `Report` collection contains: `_id`, `farmId`, `reportType`, `title`, `generatedBy`, `generatedDate`, `parameters`, `fileUrl`, `status`, `createdAt`. fileciteturn27file0turn27file1

---

# 1. Finalized Model

```text
Report:
_id,
farmId,

reportType:
    Animal
    Health
    Vaccination
    Treatment
    Breeding
    Pregnancy
    Feed
    Milk
    Inventory
    Finance
    Sensor
    Device
    Custom,

title,

generatedBy,

generatedDate,

parameters,

fileUrl,

status:
    Generating
    Ready
    Failed,

createdAt
```

The finalized source explicitly defines these fields and enum values. fileciteturn27file0

---

# 2. What Is `Report`?

`Report` stores the **metadata and lifecycle of a generated report**.

It answers:

```text
Which farm?
What type of report?
What is the report called?
Who generated it?
When was it generated?
What parameters were used?
Where is the generated file?
What is the generation status?
When was the report record created?
```

The important distinction is:

```text
Report
=
generated-report metadata / reference

Underlying models
=
actual business data used to generate the report
```

---

# 3. Why Do We Need a Separate Report Collection?

Suppose a user generates:

```text
Monthly Milk Production Report
```

The system may need to remember:

```text
Report title
Report type
Farm
Generator
Date
Filters/parameters
Generated file
Generation status
```

Without a Report collection, the system would have to regenerate or reconstruct this information every time.

The finalized model explicitly provides a dedicated Report collection. fileciteturn27file0

---

# 4. Report Is NOT the Actual Business Data

Very important:

```text
MilkRecord
=
milk production data
```

while:

```text
Report
=
a generated representation of selected milk data
```

Similarly:

```text
Animal
=
animal master data

Report
=
generated animal report
```

So Report should not become a duplicate storage location for all business data.

---

# 5. `_id`

### Type
MongoDB `ObjectId`

### Required
Yes

### Purpose

Unique identifier of the report document.

Example:

```text
ObjectId("...")
```

---

# 6. `farmId`

### Type
ObjectId → Farm

### Required
Yes

### Purpose

Identifies the farm for which the report was generated.

Example:

```text
farmId = FARM-001
```

This is essential because Liventra supports multiple farms and farm-level data isolation.

---

# 7. Why `farmId` Is Important

Suppose the same user has access to:

```text
Farm A
Farm B
```

A report must clearly belong to one farm.

Example:

```text
Report:
Monthly Finance Report
farmId = FARM-A
```

The user should not accidentally receive:

```text
Farm B
```

data inside that report.

---

# 8. Farm-Level Authorization

Before generating a report:

```text
Authenticated User
        ↓
FarmMember
        ↓
Permission / Role
        ↓
Farm
        ↓
Report Query
```

The report generation service should enforce farm isolation.

The finalized project architecture explicitly emphasizes farm isolation and role-based access control. fileciteturn27file3

---

# 9. `reportType`

### Type
Enum

### Required
Yes

### Finalized Values

```text
Animal
Health
Vaccination
Treatment
Breeding
Pregnancy
Feed
Milk
Inventory
Finance
Sensor
Device
Custom
```

These values are explicitly defined in the finalized DB model. fileciteturn27file0

---

# 10. Why `reportType` Exists

It tells the system:

```text
What kind of report is this?
```

Example:

```text
reportType = Milk
```

means the report is related to milk-production information.

---

# 11. `Animal` Report

Example:

```text
reportType = Animal
title = "Active Animal Report"
```

Could contain selected information such as:

```text
Animal Code
Name
Breed
Gender
Purpose
Weight
Health Status
Reproductive Status
Life Status
```

The exact columns are report-template logic, not separately defined by the DB model.

---

# 12. `Health` Report

Example:

```text
reportType = Health
title = "Animal Health Summary"
```

Potential source data:

```text
Animal
Disease
Treatment
SensorData
```

A Health report can summarize health-related records, but the finalized model does not define the exact report template.

---

# 13. `Vaccination` Report

Example:

```text
reportType = Vaccination
title = "Vaccination History"
```

Potential information:

```text
Animal
Vaccine
Vaccination date
Next due date
Doctor
Remarks
```

The exact report columns depend on the implementation.

---

# 14. `Treatment` Report

Example:

```text
reportType = Treatment
title = "Treatment History"
```

Potential source models:

```text
Treatment
Disease
Medicine
Animal
```

This report can summarize treatment history by animal/date/status.

---

# 15. `Breeding` Report

Example:

```text
reportType = Breeding
title = "Breeding Activity Report"
```

Potential information:

```text
Animal
Heat Date
Insemination Date
Method
Status
```

The actual fields come from the Breeding model.

---

# 16. `Pregnancy` Report

Example:

```text
reportType = Pregnancy
title = "Pregnancy Status Report"
```

Potential information:

```text
Animal
Insemination Date
Pregnancy Confirmation
Expected Delivery
Actual Delivery
```

Again, the Report model stores report metadata; the underlying Pregnancy model supplies the business data.

---

# 17. `Feed` Report

Example:

```text
reportType = Feed
title = "Feed Consumption Report"
```

Potential source models:

```text
FeedRecord
FeedFormula
FeedIngredient
FeedRecommendation
Animal
```

Possible report outputs:

```text
Animal
Feed quantity
Feeding date
Cost
Formula
Ingredient
```

The exact report layout is application-level.

---

# 18. `Milk` Report

Example:

```text
reportType = Milk
title = "Monthly Milk Production Report"
```

Potential source:

```text
MilkRecord
```

Possible analytics:

```text
Morning milk
Evening milk
Total milk
Fat
Protein
Quality
```

The finalized Report model does not prescribe the exact calculations.

---

# 19. `Inventory` Report

Example:

```text
reportType = Inventory
title = "Inventory Stock Report"
```

Potential source models:

```text
Inventory
InventoryTransaction
```

Possible output:

```text
Item
Category
Current Stock
Minimum Stock
Supplier
Stock Status
```

Exact report columns depend on the implementation.

---

# 20. `Finance` Report

Example:

```text
reportType = Finance
title = "Monthly Farm Finance Report"
```

Potential source:

```text
Income
Expense
```

Possible calculations:

```text
Total Income
Total Expense
Net Balance
```

The Report model itself does not store these calculated numbers as dedicated fields.

---

# 21. `Sensor` Report

Example:

```text
reportType = Sensor
title = "Animal Sensor Monitoring Report"
```

Potential source:

```text
SensorData
```

Possible information:

```text
Temperature
Activity
Rumination
Water Intake
Feed Intake
Milk Yield
Heart Rate
Respiration
Battery
Alerts
```

The finalized SensorData model defines the available metrics. fileciteturn27file0

---

# 22. `Device` Report

Example:

```text
reportType = Device
title = "Device Status Report"
```

Potential source:

```text
Device
DeviceAssignment
SensorData
```

Possible information:

```text
Device Code
Device Type
Serial Number
Connection Status
Health Status
Battery
Assigned Animal
Last Sync
```

The exact template is application-level.

---

# 23. `Custom` Report

### Purpose

Allows a report that does not fit the predefined report types.

Example:

```text
reportType = Custom
title = "Farm Performance Summary"
```

A Custom report can combine multiple business areas.

For example:

```text
Animal
+
Milk
+
Feed
+
Finance
```

The exact Custom-report builder behavior is not defined by the finalized DB model.

---

# 24. `title`

### Type
String

### Required
Yes

### Example

```text
"Monthly Milk Production — July 2026"
```

### Purpose

Human-readable name of the generated report.

---

# 25. Why `title` Is Separate from `reportType`

Two reports can have the same type but different titles.

Example:

```text
reportType = Milk
title = "Milk Production — July"
```

and:

```text
reportType = Milk
title = "Milk Production — August"
```

Therefore:

```text
reportType
=
machine-readable category

title
=
human-readable report name
```

---

# 26. Title Generation

The frontend may suggest:

```text
"Monthly Milk Production Report"
```

but the backend should validate/sanitize the value before storing it.

A report title should not contain:

```text
unsafe HTML
script content
```

if it will later be rendered in a web UI.

---

# 27. `generatedBy`

### Type
ObjectId → User

### Required
Yes

### Purpose

Identifies the user who requested/generated the report.

Example:

```text
generatedBy = USER-001
```

---

# 28. Why `generatedBy` Matters

Suppose three farm users generate reports:

```text
Farm Owner
Farm Manager
Farm Staff
```

The system can know:

```text
Who generated this report?
```

This is useful for:

```text
History
Accountability
Support
Audit context
```

---

# 29. `generatedDate`

### Type
Date

### Required
Yes

### Purpose

Stores when the report was generated/requested according to the finalized model.

Example:

```text
2026-08-17 10:30
```

---

# 30. `generatedDate` vs `createdAt`

These are similar but conceptually different.

### `generatedDate`

```text
Business timestamp for report generation.
```

### `createdAt`

```text
Database document creation timestamp.
```

For a simple synchronous report they may be almost identical.

For asynchronous reports:

```text
Report requested:
10:00

Report document created:
10:00

Report finished:
10:05
```

The finalized model does not have a separate `completedAt` field.

---

# 31. `parameters`

### Type
Object / JSON-like structure

### Required
Optional

### Purpose

Stores the filters/options used to generate the report.

This is one of the most important fields for reproducibility.

Example:

```json
{
  "startDate": "2026-08-01",
  "endDate": "2026-08-31",
  "animalId": "ANM-001"
}
```

The finalized model defines `parameters` but does not define a fixed internal schema. fileciteturn27file0

---

# 32. Why Parameters Matter

Suppose the user generates:

```text
Milk Report
```

with:

```text
August 1 → August 31
```

The report should preserve:

```text
startDate
endDate
```

so the system knows what dataset was requested.

---

# 33. Possible Parameters

Depending on report type:

```text
startDate
endDate
animalId
breedId
status
gender
purpose
diseaseId
vaccineId
deviceId
supplierId
category
```

These are examples of possible filters, not finalized Report schema fields.

---

# 34. Parameters Must Be Flexible

Different reports need different filters.

For example:

### Milk

```json
{
  "startDate": "...",
  "endDate": "...",
  "animalId": "..."
}
```

### Inventory

```json
{
  "category": "Feed",
  "lowStockOnly": true
}
```

### Sensor

```json
{
  "deviceId": "...",
  "startDate": "...",
  "endDate": "..."
}
```

Therefore a generic `parameters` object is appropriate.

---

# 35. Parameters vs Report Data

Very important:

```text
parameters
=
how the report was generated
```

not:

```text
parameters
=
all report results
```

The actual report output should be represented by the generated file referenced through:

```text
fileUrl
```

or generated dynamically according to the application's architecture.

---

# 36. `fileUrl`

### Type
String

### Required
Optional / depends on status

### Purpose

Stores the location/reference of the generated report file.

Example:

```text
/storage/reports/FARM-001/monthly-milk-2026-08.pdf
```

The finalized model explicitly defines `fileUrl`. fileciteturn27file0

---

# 37. Why `fileUrl` Is Optional

While a report is being generated:

```text
status = Generating
fileUrl = null
```

After successful generation:

```text
status = Ready
fileUrl = "...generated-file..."
```

If generation fails:

```text
status = Failed
```

and `fileUrl` may remain null.

This is a recommended lifecycle interpretation based on the finalized status enum.

---

# 38. `fileUrl` Should Not Be the Report Data

The Report document should not contain a giant PDF/Excel file as a MongoDB string.

Instead:

```text
Report
   ↓
fileUrl
   ↓
Object Storage / File Storage
```

The exact storage provider is not defined by the finalized DB model.

---

# 39. File Security

A report file may contain sensitive farm/business data.

Therefore a raw publicly accessible URL should generally be avoided unless intentionally designed that way.

A secure application may use:

```text
authenticated download
signed URL
permission check
```

This is an implementation/security recommendation.

---

# 40. `status`

### Type
Enum

### Required
Yes

### Finalized Values

```text
Generating
Ready
Failed
```

These are explicitly defined in the finalized model. fileciteturn27file0

---

# 41. `Generating`

Meaning:

```text
The report has been requested
but generation is not complete.
```

Typical state:

```text
status = Generating
fileUrl = null
```

---

# 42. `Ready`

Meaning:

```text
The report has successfully been generated
and can be accessed.
```

Typical:

```text
status = Ready
fileUrl != null
```

---

# 43. `Failed`

Meaning:

```text
Report generation failed.
```

Example causes:

```text
Database query error
File generation error
Storage error
Invalid parameters
Unexpected server error
```

The finalized model does not contain an error-message field, so detailed failure information must be handled elsewhere unless the schema is later expanded.

---

# 44. Status Lifecycle

Normal:

```text
Generating
     ↓
Ready
```

Failure:

```text
Generating
     ↓
Failed
```

A failed report can potentially be regenerated as a new report request.

The finalized model does not define a retry field or retry counter.

---

# 45. Why We Should Not Overwrite a Failed Report

Recommended:

```text
Report #001
status = Failed
```

Then user retries:

```text
Report #002
status = Generating
```

This preserves the history of report-generation attempts.

The finalized schema does not explicitly require append-only report attempts, but it is a useful implementation pattern.

---

# 46. `createdAt`

### Type
Date

### Required
Yes

### Purpose

Records when the Report document was created.

Example:

```text
2026-08-17 10:00
```

---

# 47. Complete Example — Milk Report

```json
{
  "farmId": "FARM-001",
  "reportType": "Milk",
  "title": "Monthly Milk Production — August 2026",
  "generatedBy": "USER-001",
  "generatedDate": "2026-08-17T10:00:00Z",

  "parameters": {
    "startDate": "2026-08-01",
    "endDate": "2026-08-31"
  },

  "fileUrl": "/reports/FARM-001/milk-2026-08.pdf",

  "status": "Ready",

  "createdAt": "2026-08-17T10:00:02Z"
}
```

This is an illustrative example based on the finalized field structure.

---

# 48. Complete Example — Sensor Report

```json
{
  "farmId": "FARM-001",
  "reportType": "Sensor",
  "title": "Animal Sensor Monitoring — ANM-001",
  "generatedBy": "USER-001",
  "generatedDate": "2026-08-17T12:00:00Z",

  "parameters": {
    "animalId": "ANM-001",
    "startDate": "2026-08-10",
    "endDate": "2026-08-17"
  },

  "fileUrl": "/reports/FARM-001/sensor-ANM-001.pdf",

  "status": "Ready",

  "createdAt": "2026-08-17T12:00:03Z"
}
```

---

# 49. Complete Example — Generating State

```json
{
  "farmId": "FARM-001",
  "reportType": "Finance",
  "title": "Monthly Finance Report",
  "generatedBy": "USER-002",
  "generatedDate": "2026-08-17T13:00:00Z",

  "parameters": {
    "startDate": "2026-08-01",
    "endDate": "2026-08-31"
  },

  "fileUrl": null,

  "status": "Generating",

  "createdAt": "2026-08-17T13:00:00Z"
}
```

---

# 50. Complete Example — Failed State

```json
{
  "farmId": "FARM-001",
  "reportType": "Finance",
  "title": "Monthly Finance Report",
  "generatedBy": "USER-002",
  "generatedDate": "2026-08-17T13:00:00Z",

  "parameters": {
    "startDate": "2026-08-01",
    "endDate": "2026-08-31"
  },

  "fileUrl": null,

  "status": "Failed",

  "createdAt": "2026-08-17T13:00:00Z"
}
```

The current finalized model does not provide a dedicated `errorMessage` field.

---

# 51. Report Generation Workflow

Recommended:

```text
User selects report
        ↓
Selects parameters
        ↓
Backend validates farm access
        ↓
Create Report document
status = Generating
        ↓
Fetch source data
        ↓
Generate report file
        ↓
Upload/store file
        ↓
Update Report
status = Ready
fileUrl = generated file
```

If anything fails:

```text
status = Failed
```

---

# 52. Why Create the Report Record First?

For long-running reports:

```text
User clicks Generate
```

should not require the browser to wait until the PDF is finished.

Instead:

```text
Create Report
status = Generating
```

then process asynchronously.

This supports:

```text
background jobs
progress tracking
retry
report history
```

The finalized schema's `Generating` status naturally supports this workflow.

---

# 53. Report Generation Queue

For heavier reports:

```text
Frontend
   ↓
Create Report
Generating
   ↓
Job Queue
   ↓
Report Worker
   ↓
Generate PDF/Excel
   ↓
Storage
   ↓
Update Report → Ready
```

The exact queue technology is not defined by the source.

---

# 54. Report Download Workflow

```text
User opens Report
        ↓
Check farm access
        ↓
Find Report
        ↓
status == Ready?
        ↓
Get fileUrl
        ↓
Authorize download
        ↓
Download file
```

Do not expose a report belonging to another farm.

---

# 55. Report History Page

A useful UI can show:

| Title | Type | Generated By | Date | Status |
|---|---|---|---|---|
| Monthly Milk | Milk | Manager | Aug 17 | Ready |
| Health Summary | Health | Owner | Aug 16 | Ready |
| Finance Report | Finance | Manager | Aug 15 | Failed |

All information can be derived from:

```text
title
reportType
generatedBy
generatedDate
status
```

---

# 56. Report Filtering

A Report list can be filtered by:

```text
farmId
reportType
status
generatedBy
generatedDate
```

The finalized model supports all of these except that `farmId`, `reportType`, `generatedBy`, `generatedDate`, and `status` are the actual fields.

---

# 57. Recommended Indexes

Useful indexes:

```javascript
{ farmId: 1, generatedDate: -1 }
```

for:

```text
recent reports of a farm
```

```javascript
{ farmId: 1, reportType: 1, generatedDate: -1 }
```

for:

```text
reports by type
```

```javascript
{ generatedBy: 1, generatedDate: -1 }
```

for:

```text
reports generated by a user
```

These are implementation recommendations; indexes are not specified in the finalized model.

---

# 58. Most Important Report Index

For a farm report-history page:

```javascript
{ farmId: 1, generatedDate: -1 }
```

is a strong candidate.

---

# 59. Report Parameters and Reproducibility

Suppose:

```text
Milk Report
```

was generated using:

```text
Aug 1–Aug 31
Animal A-001
```

and six months later the user opens the report record.

The stored:

```text
parameters
```

can show:

```text
what filters were used
```

This is valuable for understanding report history.

---

# 60. Report Parameters Are Not a Snapshot of Source Data

This is an important distinction from the FeedRecommendation model.

FeedRecommendation deliberately stores an:

```text
inputSnapshot
```

because the recommendation itself must preserve the exact input state used by the rule engine.

Report currently has:

```text
parameters
```

but **not a full source-data snapshot**.

Therefore a Report record does not guarantee that the current underlying data is identical to the data used when the report was generated.

The generated file itself is the historical output.

---

# 61. Report vs FeedRecommendation

### FeedRecommendation

```text
Recommendation result
+
inputSnapshot
+
ruleVersion
+
recommendedItems
```

### Report

```text
Report metadata
+
parameters
+
file reference
+
generation status
```

These are intentionally different responsibilities.

---

# 62. Report vs AuditLog

### Report

```text
What report was generated?
```

### AuditLog

```text
What important system/user action happened?
```

A report record already gives reporting-specific history.

Audit behavior should be defined separately.

---

# 63. Current AuditLog Consideration

The finalized AuditLog `entityType` currently includes:

```text
User
Farm
FarmMember
Animal
FeedFormula
FeedRecommendation
FeedRecord
MilkRecord
WeightRecord
Vaccine
Vaccination
Disease
Medicine
Treatment
Breeding
Pregnancy
Inventory
InventoryTransaction
Expense
Income
Notification
```

but does **not** include:

```text
Report
```

fileciteturn27file0turn27file1

Therefore report creation/download/update is not directly represented as a Report entity in the current AuditLog enum.

This is a consideration, not a change to the finalized model.

---

# 64. Current Notification Consideration

Notification's finalized `referenceType` includes:

```text
Animal
FeedRecommendation
Vaccination
Treatment
Inventory
Pregnancy
SensorData
Manual
```

but not:

```text
Report
```

fileciteturn27file0

Therefore report-specific notifications are not directly represented by the current reference-type enum.

Again, do not modify the finalized model here.

---

# 65. Report and SensorData

The finalized model explicitly supports:

```text
reportType = Sensor
```

and SensorData is the source of telemetry.

Flow:

```text
SensorData
    ↓
Query/Aggregation
    ↓
Sensor Report
    ↓
Report
```

---

# 66. Report and Device

The finalized model explicitly supports:

```text
reportType = Device
```

Possible source:

```text
Device
+
DeviceAssignment
+
SensorData
```

This can produce:

```text
Device inventory
Device status
Device assignments
Battery/connection overview
```

The exact template is application-level.

---

# 67. Report and Finance

Finance report:

```text
Income
+
Expense
```

can calculate:

```text
Total Income
Total Expense
Net Balance
```

The Report document stores the report metadata and generated file, not the entire financial dataset.

---

# 68. Report and Inventory

Inventory report may combine:

```text
Inventory
+
InventoryTransaction
```

to show:

```text
Current stock
Purchases
Consumption
Damage
Low-stock status
```

Again, calculations are generated from the underlying collections.

---

# 69. Report and Feed

Feed report may use:

```text
FeedRecord
+
FeedFormula
+
FeedIngredient
+
FeedRecommendation
```

This can support:

```text
Feed usage
Feed cost
Recommended vs actual feeding
```

The exact calculations are not part of the Report model.

---

# 70. Report and Health

Health report may combine:

```text
Disease
Treatment
Vaccination
Animal
SensorData
```

but this is a reporting composition, not a direct database relationship.

---

# 71. Report and Custom Reporting

Custom reporting could eventually allow:

```text
Select modules
Select fields
Select filters
Select date range
Generate report
```

For example:

```text
Animal
+
Milk
+
Feed
+
Finance
```

The current Report model supports:

```text
reportType = Custom
parameters
```

but does not define a full report-builder schema.

---

# 72. Important Current Schema Limitation — No `format`

The finalized Report model does not contain:

```text
format
```

such as:

```text
PDF
Excel
CSV
```

Therefore the file format must be inferred from:

```text
fileUrl
```

or handled by the report-generation system.

Do not add `format` to the finalized model unless requirements change.

---

# 73. Important Current Schema Limitation — No `fileName`

There is no:

```text
fileName
```

field.

Therefore the actual filename is represented by the storage location/fileUrl or generated by the storage service.

---

# 74. Important Current Schema Limitation — No `completedAt`

There is:

```text
generatedDate
createdAt
```

but no:

```text
completedAt
```

For long-running asynchronous reports, completion time is therefore not directly stored.

This may be useful later if detailed performance monitoring is required.

---

# 75. Important Current Schema Limitation — No `errorMessage`

Status supports:

```text
Failed
```

but the Report model has no:

```text
errorMessage
```

field.

Therefore detailed failure reason must currently be handled through:

```text
server logs
job logs
monitoring
```

unless the schema is later extended.

---

# 76. Important Current Schema Limitation — No `expiresAt`

The finalized model has no:

```text
expiresAt
```

for generated files.

Therefore file retention/expiration is not directly represented.

---

# 77. Important Current Schema Limitation — No Report Version

There is no:

```text
version
```

field.

If the same report template changes later, the current model does not directly store the template version.

This may matter for research/reproducibility but is not part of the finalized schema.

---

# 78. Important Current Schema Limitation — No Data Snapshot

Unlike FeedRecommendation:

```text
Report
```

does not contain a complete source-data snapshot.

Therefore:

```text
parameters
```

tell us what was requested, while:

```text
fileUrl
```

points to the generated output.

The current Report record itself does not reconstruct the exact historical source dataset.

---

# 79. Report Generation and Data Consistency

Suppose:

```text
User requests:
Milk report for August
```

while data is actively changing.

The report generation process should use a consistent query/time boundary.

The exact transaction/isolation strategy is not defined by the DB model.

For a production system, the reporting service should ensure the generated file is internally consistent.

---

# 80. Report File Naming

Recommended example:

```text
FARM-001/
  reports/
    milk/
      monthly-milk-2026-08.pdf
```

This is an implementation/storage convention, not a finalized schema rule.

---

# 81. Report Security

Reports can contain:

```text
financial data
animal health data
farm information
production information
```

Therefore:

```text
Report access
```

must be checked against:

```text
farm membership
role/permission
```

before returning the file.

---

# 82. Report Download Authorization

Bad:

```text
GET /reports/{fileUrl}
```

with no permission check.

Better:

```text
User
 ↓
Verify farm membership
 ↓
Verify report belongs to farm
 ↓
Verify permission
 ↓
Generate/return authorized file access
```

The exact API/storage implementation is outside the finalized model.

---

# 83. Report Generation Authorization

Not every farm member necessarily needs permission to generate every report.

For example:

```text
Finance
```

may require higher permissions than:

```text
Animal
```

The exact role-permission matrix is not defined in the Report model.

It should be handled by the application's RBAC layer.

---

# 84. Report Deletion

The finalized Report model does not define:

```text
Deleted
```

status.

Therefore report deletion/retention behavior needs a separate application policy.

A report record should not be physically deleted casually if report history is important.

---

# 85. Report Regeneration

If the user wants the same report again:

```text
Generate new report
```

Recommended:

```text
new Report document
```

rather than silently overwriting the old report.

This preserves report-generation history.

The finalized schema does not explicitly mandate this behavior.

---

# 86. Report History Example

```text
Report #001
Milk
Aug 1–Aug 31
Ready

Report #002
Milk
Aug 1–Aug 31
Ready
```

They may have been generated at different times from different underlying datasets.

Keeping both records can be useful.

---

# 87. Report vs Live Dashboard

### Dashboard

```text
Current/live data
```

### Report

```text
Generated historical output
```

Do not force every dashboard request to create a Report document.

A Report record should represent an intentionally generated report.

---

# 88. Report vs Export

An export may simply provide:

```text
CSV download
```

while a Report is a named generated artifact with:

```text
reportType
title
generatedBy
generatedDate
parameters
status
fileUrl
```

The exact distinction should be defined by the frontend/backend product behavior.

---

# 89. Report Generation Flow — Detailed

```text
1. User opens Reports.
        ↓
2. Selects report type.
        ↓
3. Selects filters.
        ↓
4. Backend validates farm access.
        ↓
5. Backend validates parameters.
        ↓
6. Create Report document.
        ↓
7. status = Generating.
        ↓
8. Generate report.
        ↓
9. Store generated file.
        ↓
10. Set fileUrl.
        ↓
11. status = Ready.
        ↓
12. User downloads/views report.
```

Failure:

```text
Generation error
      ↓
status = Failed
```

---

# 90. Report Service Responsibilities

A dedicated Report Service can handle:

```text
Report parameter validation
Farm authorization
Data querying
Aggregation
Template selection
File generation
File storage
Status updates
```

The Report collection stores the resulting metadata.

---

# 91. Example Backend Logic

Conceptually:

```text
createReport(request):

    authenticate user

    validate farm access

    validate reportType

    validate parameters

    create Report:
        status = Generating

    generate output

    upload file

    update Report:
        status = Ready
        fileUrl = ...

    return Report
```

This is an implementation outline.

---

# 92. Failed Generation Logic

```text
try:
    generate report
except error:
    update Report:
        status = Failed

    log detailed error
```

Because the finalized Report model has no `errorMessage`, detailed technical errors should not be inserted into the normal report metadata unless the schema is intentionally expanded.

---

# 93. Report API Concept

Possible endpoints:

```text
POST   /api/reports
GET    /api/reports
GET    /api/reports/:id
GET    /api/reports/:id/download
```

These are example routes only; the finalized DB model does not define API endpoints.

---

# 94. Report List Query

Conceptually:

```text
farmId = currentFarm
sort = generatedDate DESC
```

Optional filters:

```text
reportType
status
generatedBy
date range
```

---

# 95. Report Detail Page

Possible display:

```text
Title
Type
Farm
Generated By
Generated Date
Parameters
Status
Download
```

The fields map directly to the finalized Report model.

---

# 96. Report Status UI

```text
Generating
→ Show spinner / processing

Ready
→ Show Download button

Failed
→ Show Failed state / Retry option
```

The exact UI is not part of the DB model.

---

# 97. Report Notification

Current Notification reference types do not include `Report`. fileciteturn27file0

Therefore if the system later wants:

```text
"Your report is ready."
```

the notification would need either:

```text
referenceType = Manual
```

or a future schema update to support Report explicitly.

Do not silently modify the finalized enum.

---

# 98. Report and Auditability

Even though Report is not currently listed in AuditLog's entityType enum, the Report document itself preserves:

```text
generatedBy
generatedDate
parameters
status
```

So report-generation history has a dedicated source of metadata.

---

# 99. Report and Research Reproducibility

The strongest reproducibility fields currently are:

```text
reportType
title
generatedBy
generatedDate
parameters
fileUrl
```

However, because there is no:

```text
dataSnapshot
templateVersion
ruleVersion
completedAt
```

the current model does not provide perfect reproducibility metadata.

This is a current design limitation, not something to silently fix here.

---

# 100. Report and Data Privacy

Potentially sensitive report categories:

```text
Health
Treatment
Finance
```

Therefore report files should follow the same farm-level authorization policy as the underlying data.

---

# 101. Report and File Storage Failure

Possible flow:

```text
Database report created
status = Generating

File generation succeeds

Storage upload fails

↓

status = Failed
fileUrl = null
```

This should be handled by the report-generation service.

---

# 102. Report and Database Failure

If:

```text
file generated
```

but:

```text
database update fails
```

the system may end up with an orphaned file.

Therefore production implementation should consider:

```text
transactional metadata update
+
file cleanup
+
retry
```

The exact strategy depends on the storage system.

---

# 103. Report and Background Jobs

Large reports should preferably use background processing:

```text
HTTP Request
    ↓
Create Report
Generating
    ↓
Queue Job
    ↓
Worker
    ↓
Generate File
    ↓
Update Report
Ready
```

This prevents request timeouts.

---

# 104. Report Generation Performance

Reports involving:

```text
SensorData
MilkRecord
FeedRecord
InventoryTransaction
```

may scan large collections.

Therefore report services should use:

```text
proper indexes
aggregation pipelines
date filters
pagination where appropriate
pre-aggregation/caching if necessary
```

The finalized model itself does not define these performance mechanisms.

---

# 105. Sensor Report Example

```text
Report:
"Animal Sensor Monitoring — August"

reportType:
Sensor

parameters:
{
    animalId: ANM-001,
    startDate: 2026-08-01,
    endDate: 2026-08-31
}

↓

SensorData query

↓

Aggregation

↓

PDF

↓

fileUrl

↓

status = Ready
```

---

# 106. Finance Report Example

```text
Report:
"August Finance Summary"

reportType:
Finance

parameters:
{
    startDate: 2026-08-01,
    endDate: 2026-08-31
}

↓

Income + Expense

↓

Total Income
Total Expense
Net Balance

↓

PDF/Excel

↓

Report.status = Ready
```

---

# 107. Inventory Report Example

```text
Report:
"Current Inventory"

reportType:
Inventory

parameters:
{
    category: "Feed"
}

↓

Inventory + InventoryTransaction

↓

Stock analysis

↓

Generated file
```

---

# 108. Custom Report Example

```text
reportType:
Custom

title:
"Farm Performance Summary"

parameters:
{
    modules: [
        "Animal",
        "Milk",
        "Feed",
        "Finance"
    ],
    startDate: "...",
    endDate: "..."
}
```

This illustrates why `parameters` needs to remain flexible.

The exact custom-report builder is not defined by the finalized schema.

---

# 109. Common Mistakes to Avoid

### Mistake 1
Putting all report results inside the Report document.

**Problem:** duplication and huge documents.

### Mistake 2
Removing `parameters`.

**Problem:** report-generation context is lost.

### Mistake 3
Overwriting old report records.

**Problem:** history becomes unclear.

### Mistake 4
Making `fileUrl` mandatory at creation.

**Problem:** asynchronous `Generating` state becomes difficult.

### Mistake 5
Ignoring farm authorization.

**Problem:** sensitive cross-farm data exposure.

### Mistake 6
Treating report as live dashboard data.

**Problem:** unnecessary report generation and storage.

### Mistake 7
Storing sensitive technical errors directly in user-facing report metadata.

**Problem:** information leakage.

---

# 110. Business Rules Summary

```text
1. Every report belongs to a farm.
2. Report type must be one of the finalized enum values.
3. Every report has a human-readable title.
4. generatedBy identifies the requesting/generating user.
5. generatedDate records the report generation timestamp.
6. parameters store the filters/options used for the report.
7. fileUrl references the generated output.
8. Generating means report generation is in progress.
9. Ready means the generated output is available.
10. Failed means generation did not complete successfully.
11. Farm access must be validated before report generation.
12. Report should not duplicate entire business datasets.
13. Underlying collections remain the source of truth.
14. Sensor reports use SensorData as their source.
15. Device reports can combine Device/Assignment/SensorData.
16. Finance reports can use Income and Expense.
17. Inventory reports can use Inventory and InventoryTransaction.
18. Feed reports can use Feed-related collections.
19. Historical report records should preferably not be silently overwritten.
20. Generated files should be stored outside the Report document.
21. Report downloads must respect farm authorization.
22. Large reports should preferably be generated asynchronously.
23. Report generation failures should result in status = Failed.
24. Detailed technical errors are not represented by the finalized Report schema.
25. Report currently has no explicit file format field.
26. Report currently has no completedAt field.
27. Report currently has no expiresAt field.
28. Report currently has no template/rule version field.
29. Report currently has no full source-data snapshot.
30. Current AuditLog does not explicitly support Report as an entityType.
31. Current Notification does not explicitly support Report as a referenceType.
```

The field structure and enum values are source-derived; workflow, security, indexing, storage, and lifecycle rules above are implementation recommendations based on those fields. fileciteturn27file0turn27file1

---

# 111. Current Schema Review — Important Considerations

## Consideration 1 — `parameters` Is Flexible but Unstructured

Current:

```text
parameters
```

is generic.

This is useful, but validation must happen according to:

```text
reportType
```

Example:

```text
Milk
→ date range / animal

Inventory
→ category / stock filters

Sensor
→ device/animal/date range
```

---

## Consideration 2 — No `format`

Current schema does not tell us:

```text
PDF
Excel
CSV
```

So format must be handled by the report-generation layer.

---

## Consideration 3 — No `completedAt`

For asynchronous report jobs, knowing:

```text
requested/generated time
```

but not:

```text
completion time
```

limits generation-performance analytics.

---

## Consideration 4 — No `errorMessage`

`Failed` exists, but failure details are outside the model.

---

## Consideration 5 — No Snapshot

Unlike FeedRecommendation, Report has no full data snapshot.

The generated file therefore becomes the primary historical representation of the report output.

---

# 112. Final Architecture

```text
                     USER
                       │
                       ▼
                Report Request
                       │
                       ▼
               Authorization
                       │
                       ▼
                 Parameters
                       │
                       ▼
                Report Document
               status=Generating
                       │
                       ▼
                Source Collections
        ┌──────────────┼──────────────┐
        │              │              │
       Animal         Milk          Finance
        │              │              │
      Health         Feed         Inventory
        │              │              │
      Sensor         Device       Treatment
        └──────────────┼──────────────┘
                       ▼
                Report Generator
                       │
                       ▼
                  File Storage
                       │
                       ▼
                Report.status
                     Ready
                       │
                       ▼
                   fileUrl
                       │
                       ▼
                    User
```

---

# 113. Final Responsibility Separation

```text
Animal
────────────────────────
Master animal information
```

```text
MilkRecord
────────────────────────
Milk production records
```

```text
SensorData
────────────────────────
Historical IoT measurements
```

```text
Income / Expense
────────────────────────
Financial source data
```

```text
Inventory
────────────────────────
Inventory source data
```

```text
Report
────────────────────────
Generated report metadata
+
parameters
+
file reference
+
generation status
```

This keeps the Report model clean and avoids making it a duplicate database for every module.

---

# 114. Final Status

**Model:** `Report`  
**Number:** 28  
**Status:** ✅ Final

```text
Identity                  ✓
Farm relationship         ✓
Report type               ✓
Title                     ✓
Generated by              ✓
Generated date            ✓
Parameters                ✓
File reference            ✓
Generation status         ✓
Async workflow            ✓
Farm authorization        ✓
Report history            ✓
Source-data separation    ✓
Sensor reporting          ✓
Device reporting          ✓
Finance reporting         ✓
Inventory reporting       ✓
Custom reporting          ✓
Index recommendations     ✓
Security considerations   ✓
Current schema gaps       ✓
```

---

# 115. Complete Project Model Progress

```text
01 User
02 Farm
03 FarmMember
04 Breed
05 Animal
06 FeedIngredient
07 FeedFormula
08 FeedRecommendation
09 FeedRecord
10 MilkRecord
11 WeightRecord
12 Vaccine
13 Vaccination
14 Disease
15 Medicine
16 Treatment
17 Breeding
18 Pregnancy
19 Inventory
20 InventoryTransaction
21 Expense
22 Income
23 Notification
24 AuditLog
25 Device
26 DeviceAssignment
27 SensorData
28 Report ← FINAL
```

# 🎯 FINAL MODEL COMPLETE

All **28 Liventra DB models** have now been documented from the business-logic perspective.

The final separation is:

```text
MASTER DATA
User
Farm
FarmMember
Breed
Disease
Medicine
Vaccine
FeedIngredient
Device

ANIMAL & LIVESTOCK
Animal
WeightRecord
MilkRecord
Breeding
Pregnancy

FEED
FeedFormula
FeedRecommendation
FeedRecord

HEALTH
Vaccination
Treatment

INVENTORY & FINANCE
Inventory
InventoryTransaction
Income
Expense

NOTIFICATION / AUDIT
Notification
AuditLog

IOT
Device
DeviceAssignment
SensorData

REPORTING
Report
```

The key architectural principle is:

```text
Master Models
      ↓
Operational Records
      ↓
Historical/Analytical Data
      ↓
Notifications / Audit
      ↓
Reports
```

And importantly, `Report` is **not another source-of-truth collection**. The underlying models remain the source of truth; Report records the generated artifact's metadata, parameters, status, and file reference.
