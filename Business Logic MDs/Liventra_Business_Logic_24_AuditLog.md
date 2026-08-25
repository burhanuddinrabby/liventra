# Liventra — Business Logic Documentation
## Model 24: `AuditLog`

**Document Version:** Business Logic v1.0  
**Project:** Liventra — Smart Livestock Farm Management System  
**Database:** MongoDB  
**Architecture:** MERN  
**Module:** System / Audit & Activity Tracking

---

# 1. Source / Finalized Model

The latest finalized Liventra DB Models define `AuditLog` as:

```text
AuditLog:
_id,
logNo,
farmId,
userId,
entityType:
    Animal
    FeedFormula
    FeedRecommendation
    FeedRecord
    WeightRecord
    MilkRecord
    Vaccination
    Treatment
    Inventory
    InventoryTransaction
    Expense
    Income
    Notification
    Farm
    User

entityId,

action:
    Create
    Update
    Delete
    Archive
    Login
    Logout
    Export
    Import
    Approve
    Reject

module:
    Animal
    Feed
    Health
    Inventory
    Finance
    Farm
    Authentication
    Administration

oldData,
newData,
changedFields:[],

ipAddress,
userAgent,
deviceType:
    Desktop
    Mobile
    Tablet
    API

browser,
operatingSystem,

location:{
    country,
    city,
    latitude,
    longitude
},

requestId,

status:
    Success
    Failed
    Rolled Back

createdAt
```

These fields and enum values are present in the latest finalized DB Models document. fileciteturn23file0turn23file1

The SRS also requires **Error Logging**, and its security requirements include preventing cross-farm access and enforcing role-based access on protected routes. fileciteturn23file5

---

# 2. Purpose

`AuditLog` is Liventra's **historical accountability record**.

It answers:

> **Who did what, to which record, in which module, when, from where, through which device/request, what changed, and whether the operation succeeded or failed?**

Example:

```text
User:
Farm Manager

Action:
Update

Entity:
Animal

Entity ID:
ANM-001

Changed:
healthStatus
currentWeight

Old:
Healthy
180 kg

New:
Sick
175 kg

Status:
Success
```

This is fundamentally different from normal application data.

---

# 3. Why AuditLog Is Important in Liventra

Liventra handles:

```text
Animal data
Health records
Vaccination
Treatment
Feed recommendations
Inventory
Expense
Income
User/farm management
```

Many of these records can affect:

```text
Animal history
Health decisions
Financial reports
Inventory balances
Feed recommendations
Farm operations
```

Therefore, simply storing the current state is not enough.

Example:

```text
Animal.healthStatus
```

currently says:

```text
Sick
```

But the normal Animal document does not answer:

```text
Who changed it?
When?
What was the previous value?
Why did the change happen?
```

AuditLog provides that historical context.

---

# 4. AuditLog vs Normal Collection

### Normal Collection

Stores:

```text
Current / operational state
```

Example:

```text
Animal
healthStatus = Sick
```

### AuditLog

Stores:

```text
Historical event
```

Example:

```text
User X
changed healthStatus
Healthy → Sick
at 10:30 AM
```

So:

```text
Operational Data
=
What is true now?

Audit Data
=
What happened over time?
```

---

# 5. AuditLog vs Error Log

These are related but not identical.

## AuditLog

Tracks user/system actions:

```text
Create
Update
Delete
Login
Export
Approve
```

## Error Logging

Tracks technical failures:

```text
Database connection failure
Unhandled exception
API timeout
```

The finalized AuditLog has:

```text
status = Failed
```

so failed business operations can be represented.

However, **AuditLog should not be treated as a complete replacement for application/server error logging**.

The SRS explicitly requires Error Logging. fileciteturn23file5

---

# 6. AuditLog vs Realtime Audit Viewer

This is important for the earlier requirement of seeing changes in realtime.

```text
AuditLog
    ↓
Persistent database history

WebSocket / Socket.IO
    ↓
Realtime delivery mechanism
```

Therefore:

```text
Database AuditLog
+
Realtime event
```

should be used together.

Example:

```text
User changes Expense
        ↓
AuditLog saved
        ↓
audit.created event
        ↓
Admin/Auditor dashboard updates instantly
```

If the auditor is offline:

```text
AuditLog remains in database
```

and can be viewed later.

---

# 7. AuditLog Is the Core of the Audit System

The overall architecture can be:

```text
                  Liventra Action
                        │
                        ▼
                 Business Service
                        │
             ┌──────────┴──────────┐
             ▼                     ▼
        Main Database          AuditLog
             │                     │
             └──────────┬──────────┘
                        ▼
                 Commit Successful
                        │
                        ▼
                 Realtime Event
                        │
                        ▼
                Audit Dashboard
```

For multi-document operations, the main data change and AuditLog creation should ideally be handled consistently within the same transaction where practical.

---

# 8. ` _id`

### Type

MongoDB `ObjectId`

### Required

Yes

### Purpose

Unique internal identifier for the audit record.

Example:

```text
ObjectId(...)
```

---

# 9. `logNo`

### Type

String

### Required

Yes

### Example

```text
LOG-000001
```

### Purpose

Human-readable audit log identifier.

Useful for:

```text
Audit search
Support
Investigation
Reports
Export
Administrative review
```

---

# 10. Log Number Generation

Recommended format:

```text
LOG-{SEQUENCE}
```

Examples:

```text
LOG-000001
LOG-000002
LOG-000003
```

The backend should generate this.

Do not rely on frontend-generated IDs for audit integrity.

---

# 11. `farmId`

### Type

ObjectId / Farm reference

### Required

Usually required for farm-scoped actions

### Purpose

Identifies the farm context in which the action happened.

Example:

```text
farmId = FARM-001
```

---

# 12. Why Farm Scope Is Critical

Liventra supports multiple farms and explicitly requires prevention of cross-farm data access. fileciteturn23file5

Suppose:

```text
User X
has access to:
Farm A
```

and attempts:

```text
Update Animal from Farm B
```

The application should:

```text
Reject operation
```

and ideally create an audit/security event indicating the failed attempt.

---

# 13. Cross-Farm Security Audit

Example:

```text
User:
U-001

Requested:
Update Animal A-005

Animal belongs to:
Farm B

User context:
Farm A
```

Result:

```text
Operation rejected
status = Failed
```

The audit trail can help demonstrate that the system prevented unauthorized cross-farm access.

---

# 14. `userId`

### Type

ObjectId / User reference

### Required

Conditional

### Purpose

Identifies the user responsible for the action.

Example:

```text
userId = USER-001
```

For authenticated actions:

```text
userId
```

should normally be populated.

---

# 15. System-Generated Actions

Some actions may occur without a human directly clicking a button.

Example:

```text
Scheduled vaccination reminder
```

or:

```text
Automatic notification generation
```

In such cases, the action may be performed by a system process rather than a normal user.

The finalized schema does not define a dedicated:

```text
actorType
```

field.

Therefore, the implementation should establish a consistent convention for system-generated actions, rather than inventing a new field without changing the model.

---

# 16. `entityType`

### Type

Enum

### Required

Yes

### Finalized Values

```text
Animal
FeedFormula
FeedRecommendation
FeedRecord
WeightRecord
MilkRecord
Vaccination
Treatment
Inventory
InventoryTransaction
Expense
Income
Notification
Farm
User
```

These exact values are defined in the finalized DB model. fileciteturn23file0

---

# 17. Meaning of `entityType`

It tells the audit system:

> **What kind of record was affected?**

Example:

```text
entityType = Animal
```

means:

```text
The action affected an Animal record.
```

---

# 18. `entityId`

### Type

ObjectId / entity reference

### Required

Conditional

### Purpose

Identifies the exact record that was affected.

Example:

```text
entityType = Animal
entityId = ANM-001
```

means:

```text
Animal ANM-001 was affected.
```

---

# 19. `entityType + entityId`

These two fields should always be interpreted together.

```text
entityType
+
entityId
```

Example:

```text
Expense
EXP-001
```

means:

```text
Expense document EXP-001
```

Invalid example:

```text
entityType = Animal
entityId = Expense._id
```

The audit service should never create mismatched pairs.

---

# 20. Actions Supported by the Finalized Model

The finalized action enum is:

```text
Create
Update
Delete
Archive
Login
Logout
Export
Import
Approve
Reject
```

fileciteturn23file0

Each action has a different business meaning.

---

# 21. `action = Create`

Use when a new record is created.

Example:

```text
Create Animal
```

Audit:

```text
entityType = Animal
action = Create
```

Typical:

```text
oldData = null
newData = created document/snapshot
```

---

# 22. `action = Update`

Use when an existing record is modified.

Example:

```text
Animal.healthStatus:
Healthy → Sick
```

Audit:

```text
action = Update

oldData:
{
    healthStatus: "Healthy"
}

newData:
{
    healthStatus: "Sick"
}

changedFields:
[
    "healthStatus"
]
```

---

# 23. `action = Delete`

Use when a record is deleted.

For a delete event:

```text
oldData
=
record before deletion
```

and:

```text
newData
=
null
```

if using a full-snapshot approach.

---

# 24. Hard Delete vs Audit

Even if a record is hard-deleted:

```text
Main record disappears
```

the AuditLog should preserve:

```text
what existed before deletion
```

where legally and technically appropriate.

This is one reason `oldData` is valuable.

---

# 25. `action = Archive`

Use when a record is intentionally archived instead of deleted.

Example:

```text
Animal
lifeStatus → Archived
```

or:

```text
Notification
status → Archived
```

The AuditLog records the archival event.

---

# 26. `action = Login`

Records successful or failed authentication-related login events.

Example:

```text
entityType = User
action = Login
status = Success
```

Useful metadata:

```text
ipAddress
deviceType
browser
operatingSystem
requestId
createdAt
```

---

# 27. `action = Logout`

Example:

```text
entityType = User
action = Logout
status = Success
```

This provides authentication activity history.

---

# 28. Login/Logout and `entityId`

For:

```text
Login
Logout
```

there is not always a traditional business entity being modified.

A practical mapping is:

```text
entityType = User
entityId = authenticated user's ID
```

This keeps the audit record consistent with the finalized schema.

---

# 29. `action = Export`

Use when data/report is exported.

Examples:

```text
Export Income Report
Export Audit Log
Export Animal Data
```

The Report model supports report formats:

```text
PDF
Excel
CSV
JSON
```

and includes `Audit Log` as a report type. fileciteturn23file0

Therefore exports should be auditable.

---

# 30. Export Example

```text
entityType:
Income

action:
Export

module:
Finance

status:
Success

requestId:
REQ-00125
```

Additional information can be represented through available fields, but the current model has no dedicated:

```text
exportFormat
reportId
```

field.

Therefore those details should be captured through the associated request/report context if required.

---

# 31. `action = Import`

Use when data is imported into the system.

Examples:

```text
Animal CSV import
Inventory import
Milk records import
```

Audit:

```text
action = Import
status = Success
```

This is especially important because bulk import can modify many records.

---

# 32. Bulk Import Auditing

Suppose:

```text
500 animals imported
```

Do not necessarily create only one vague audit record.

The exact strategy depends on the implementation.

Possible approach:

```text
Import operation audit
+
individual record audits
```

or:

```text
import-level audit
+
batch/request identifier
```

The current model provides:

```text
requestId
```

which is useful for correlating related audit entries.

---

# 33. `action = Approve`

Use when a supported business workflow contains approval.

Example:

```text
FeedRecommendation
status:
draft → confirmed
```

If an explicit approval action is performed:

```text
action = Approve
```

The FeedRecommendation model includes:

```text
status:
draft
confirmed
expert_approved
rejected
superseded
```

and `reviewedBy/reviewedAt`. The audit action should document important approval/rejection events. 

---

# 34. `action = Reject`

Use when a supported workflow rejects something.

Example:

```text
FeedRecommendation
→ rejected
```

Audit:

```text
action = Reject
```

The current AuditLog model does not contain a dedicated:

```text
reason
```

field.

Therefore, rejection explanation would need to be captured through the affected record's own fields or other available context.

---

# 35. `module`

### Type

Enum

### Required

Yes

### Finalized Values

```text
Animal
Feed
Health
Inventory
Finance
Farm
Authentication
Administration
```

fileciteturn23file0

---

# 36. Why `module` Is Needed

`entityType` tells:

```text
WHAT record
```

`module` tells:

```text
WHICH functional area
```

Example:

```text
entityType = Vaccination
module = Health
```

Another:

```text
entityType = Expense
module = Finance
```

---

# 37. Example Module Mapping

```text
Animal
→ Animal

FeedFormula
→ Feed

FeedRecommendation
→ Feed

FeedRecord
→ Feed

WeightRecord
→ Animal

MilkRecord
→ Animal

Vaccination
→ Health

Treatment
→ Health

Inventory
→ Inventory

InventoryTransaction
→ Inventory

Expense
→ Finance

Income
→ Finance

Notification
→ Administration

Farm
→ Farm

User
→ Authentication / Administration
```

These are implementation mappings based on the finalized enum structure.

---

# 38. `oldData`

### Type

Object / JSON-like snapshot

### Required

Conditional

### Purpose

Stores the previous state of the affected record.

Example:

```json
{
  "healthStatus": "Healthy",
  "currentWeight": 180
}
```

before update.

---

# 39. Why `oldData` Is Important

Without `oldData`:

```text
Audit says:
Animal updated
```

but cannot answer:

```text
What was changed from?
```

With `oldData`:

```text
Healthy
```

can be compared against:

```text
Sick
```

This is essential for investigations.

---

# 40. `newData`

### Type

Object / JSON-like snapshot

### Required

Conditional

### Purpose

Stores the resulting state after the operation.

Example:

```json
{
  "healthStatus": "Sick",
  "currentWeight": 175
}
```

---

# 41. Old vs New Snapshot

Example:

```text
oldData:
{
    currentWeight: 180,
    healthStatus: "Healthy"
}

newData:
{
    currentWeight: 175,
    healthStatus: "Sick"
}
```

This creates a clear historical transition:

```text
180 → 175 kg
Healthy → Sick
```

---

# 42. Snapshot Strategy

The finalized model does not specify whether:

```text
oldData/newData
```

must contain the **full document** or only changed fields.

A practical implementation choice is:

### Full snapshot

Store the relevant document state.

Advantages:

```text
Strong historical reconstruction
```

Disadvantages:

```text
Large audit records
```

### Changed-field snapshot

Store only affected fields.

Advantages:

```text
Smaller records
```

Disadvantages:

```text
Less complete reconstruction
```

For Liventra, a balanced strategy should be selected and applied consistently.

---

# 43. Recommended Snapshot Strategy for Liventra

For ordinary updates:

```text
oldData = only relevant previous fields
newData = only corresponding new fields
changedFields = exact changed paths
```

Example:

```text
oldData:
{
    healthStatus: "Healthy"
}

newData:
{
    healthStatus: "Sick"
}

changedFields:
[
    "healthStatus"
]
```

This is more storage-efficient than copying the entire Animal document for every small change.

For critical financial/security operations, a fuller snapshot may be justified.

---

# 44. `changedFields`

### Type

Array of strings

### Required

For Update operations

### Example

```text
[
    "healthStatus",
    "currentWeight"
]
```

### Purpose

Quickly identifies which fields changed without comparing complete snapshots.

---

# 45. Nested `changedFields`

For nested fields, use field paths.

Example:

```text
location.shed
```

or:

```text
gallery
```

depending on the update.

This makes the audit viewer easier to implement.

---

# 46. Create Operation and `changedFields`

For:

```text
Create
```

there is no previous state.

Therefore:

```text
oldData = null
```

and `changedFields` can be:

```text
[]
```

or contain the created fields, depending on the implementation convention.

Choose one convention and use it consistently.

---

# 47. Delete Operation

Recommended:

```text
oldData = relevant deleted record state
newData = null
changedFields = []
```

This allows the audit viewer to show:

```text
Record deleted
```

with the previous values.

---

# 48. Archive Operation

For:

```text
Archive
```

example:

```text
oldData:
{
    status: "Active"
}

newData:
{
    status: "Archived"
}

changedFields:
[
    "status"
]
```

---

# 49. `ipAddress`

### Type

String

### Required

Optional / conditional

### Example

```text
103.xxx.xxx.xxx
```

### Purpose

Records the network address from which the request originated.

Useful for:

```text
Security investigation
Suspicious activity
Login auditing
Administrative investigation
```

---

# 50. Important IP Consideration

If Liventra is deployed behind:

```text
Nginx
Cloudflare
Load Balancer
Proxy
```

the backend may see a proxy address rather than the original client IP.

Therefore, production configuration must correctly handle trusted proxy headers.

Do not blindly trust arbitrary client-supplied IP values.

---

# 51. `userAgent`

### Type

String

### Required

Optional

### Purpose

Stores the client user-agent string.

Example:

```text
Mozilla/5.0 ...
```

Useful for:

```text
Browser/device investigation
Security analysis
Support
```

---

# 52. `deviceType`

### Type

Enum

### Required

Optional / conditional

### Finalized Values

```text
Desktop
Mobile
Tablet
API
```

fileciteturn23file0

---

# 53. Device Type Examples

```text
Chrome on laptop
→ Desktop

Android phone
→ Mobile

iPad
→ Tablet

Machine-to-machine request
→ API
```

---

# 54. `browser`

### Type

String

### Required

Optional

### Example

```text
Chrome
```

Purpose:

```text
Security diagnostics
Audit display
```

---

# 55. `operatingSystem`

### Type

String

### Required

Optional

### Example

```text
Windows
Android
iOS
Linux
macOS
```

Useful for security investigation and support.

---

# 56. `location`

### Type

Object

### Required

Optional

### Fields

```text
country
city
latitude
longitude
```

The finalized model explicitly defines these nested fields. fileciteturn23file0

---

# 57. Why Location Exists

Location can provide contextual information for:

```text
Login
Sensitive financial changes
Administrative changes
Suspicious activity
```

Example:

```text
Login from:
Dhaka, Bangladesh
```

---

# 58. Location Must Be Treated Carefully

Because location can be sensitive, it should not be collected or displayed unnecessarily.

Recommended:

```text
Only collect when justified
Restrict access
Use secure storage
Apply retention policy
```

The DB model defines the fields, but the exact privacy/retention policy is not specified in the source.

---

# 59. `requestId`

### Type

String

### Required

Recommended for API-driven operations

### Example

```text
REQ-20260817-8F31
```

### Purpose

Correlates:

```text
API request
+
AuditLog
+
application logs
+
errors
+
realtime events
```

This is extremely useful for debugging.

---

# 60. Why `requestId` Is Powerful

Suppose a user reports:

> "I changed an animal's status and something went wrong."

Support can search:

```text
requestId
```

and find:

```text
API log
AuditLog
Error log
Database operation
```

This makes debugging much easier.

---

# 61. `status`

### Type

Enum

### Required

Yes

### Finalized Values

```text
Success
Failed
Rolled Back
```

These exact values are defined in the finalized model. fileciteturn23file0

---

# 62. `status = Success`

Means:

```text
The audited operation completed successfully.
```

Example:

```text
Update Animal
status = Success
```

---

# 63. `status = Failed`

Means:

```text
The attempted operation failed.
```

Example:

```text
User attempted unauthorized update
status = Failed
```

This can be particularly valuable for security-related investigations.

---

# 64. `status = Rolled Back`

Means:

```text
The operation began,
but the database transaction was rolled back.
```

Example:

```text
Animal sale
    ↓
Income creation
    ↓
Animal update
    ↓
Second operation failed
    ↓
Transaction rollback
```

Audit can record:

```text
status = Rolled Back
```

---

# 65. Failed vs Rolled Back

These are not identical.

### Failed

```text
Operation did not successfully complete.
```

### Rolled Back

```text
Operation may have started,
but its database changes were reverted.
```

This distinction is valuable for transactional business workflows.

---

# 66. `createdAt`

### Type

Date

### Required

Yes

### Purpose

Records exactly when the audit event was created.

This is the primary timeline field.

---

# 67. Audit Timeline

Example:

```text
10:00
User logged in

10:05
Animal created

10:20
Weight updated

10:25
Vaccination recorded

10:30
Expense created
```

AuditLog provides this chronological history.

---

# 68. Create Audit Example — Animal

```javascript
{
    logNo: "LOG-000001",

    farmId: "FARM-001",

    userId: "USER-001",

    entityType: "Animal",

    entityId: "ANIMAL-001",

    action: "Create",

    module: "Animal",

    oldData: null,

    newData: {
        animalCode: "A-001",
        name: "Raja",
        gender: "Male",
        purpose: "Beef"
    },

    changedFields: [],

    deviceType: "Desktop",

    requestId: "REQ-001",

    status: "Success",

    createdAt: "2026-08-17T10:00:00Z"
}
```

Values are illustrative.

---

# 69. Update Audit Example — Animal

```javascript
{
    logNo: "LOG-000002",

    farmId: "FARM-001",

    userId: "USER-002",

    entityType: "Animal",

    entityId: "ANIMAL-001",

    action: "Update",

    module: "Animal",

    oldData: {
        currentWeight: 180,
        healthStatus: "Healthy"
    },

    newData: {
        currentWeight: 175,
        healthStatus: "Sick"
    },

    changedFields: [
        "currentWeight",
        "healthStatus"
    ],

    deviceType: "Mobile",

    requestId: "REQ-002",

    status: "Success",

    createdAt: "2026-08-17T10:30:00Z"
}
```

---

# 70. Financial Audit Example — Expense

Suppose:

```text
Expense:
৳5,000
```

changes to:

```text
৳5,500
```

Audit:

```text
entityType = Expense
module = Finance
action = Update

oldData:
{
    amount: 5000
}

newData:
{
    amount: 5500
}

changedFields:
[
    "amount"
]
```

This is important because financial modifications should be traceable.

---

# 71. Financial Audit Example — Income

Example:

```text
Income amount:
৳8,000 → ৳8,500
```

Audit:

```text
entityType = Income
module = Finance
action = Update
```

This provides a historical trail for financial reporting.

---

# 72. Inventory Audit Example

Suppose:

```text
Inventory.currentStock
100 kg → 70 kg
```

The audit can record:

```text
entityType = Inventory
module = Inventory
action = Update
changedFields:
[
    "currentStock"
]
```

However, stock movement should also be represented through the finalized `InventoryTransaction` model. AuditLog should not replace operational inventory transactions.

---

# 73. InventoryTransaction Audit

The finalized AuditLog includes:

```text
entityType = InventoryTransaction
```

Therefore:

```text
Purchase
Consumption
Damage
Adjustment
Return
```

can be audited.

This is important because:

```text
Inventory
=
current state

InventoryTransaction
=
stock movement history

AuditLog
=
who performed/changed the stock operation
```

---

# 74. Feed Recommendation Audit

The finalized AuditLog supports:

```text
entityType = FeedRecommendation
```

This is especially valuable because FeedRecommendation is intended to preserve historical recommendation snapshots.

Example:

```text
Recommendation R-001
status:
draft → confirmed
```

Audit:

```text
action = Approve
```

or:

```text
action = Update
```

depending on the actual operation.

---

# 75. Why Feed Recommendation Needs Audit

The FeedRecommendation model already preserves:

```text
inputSnapshot
recommendedItems
ruleVersion
status
```

AuditLog adds:

```text
Who confirmed/rejected/modified it?
When?
```

This strengthens:

```text
Expert validation
Research evaluation
Historical traceability
```

---

# 76. Vaccination Audit

Possible events:

```text
Vaccination created
Vaccination updated
Vaccination cancelled
Vaccination completed
```

Example:

```text
entityType = Vaccination
module = Health
```

This provides medical-history accountability.

---

# 77. Treatment Audit

Treatment can contain:

```text
Disease
Medicines
Dosage
Estimated Cost
Status
Follow-up Date
```

Changes to these fields can materially affect animal care.

Therefore treatment modifications should be audited.

Example:

```text
changedFields:
[
    "status",
    "followUpDate"
]
```

---

# 78. Notification Audit

The finalized AuditLog supports:

```text
entityType = Notification
```

Therefore important notification lifecycle actions can be audited.

Example:

```text
Notification archived
```

Audit:

```text
action = Archive
module = Administration
```

---

# 79. User Audit

The finalized AuditLog supports:

```text
entityType = User
```

This is especially important for:

```text
Login
Logout
Account changes
Administrative changes
```

Example:

```text
entityType = User
action = Login
status = Success
```

---

# 80. Farm Audit

The finalized AuditLog supports:

```text
entityType = Farm
```

Therefore:

```text
Farm created
Farm updated
Farm archived
```

can be audited.

---

# 81. Important Coverage Limitation

The current finalized `entityType` enum does **not** include every collection in the finalized DB Models.

It includes:

```text
Animal
FeedFormula
FeedRecommendation
FeedRecord
WeightRecord
MilkRecord
Vaccination
Treatment
Inventory
InventoryTransaction
Expense
Income
Notification
Farm
User
```

but does not directly include:

```text
Breed
FeedIngredient
Vaccine
Disease
Medicine
Breeding
Pregnancy
```

The finalized collection list confirms these are separate collections. fileciteturn23file0

Therefore, under the **current schema**, do not claim that these omitted entities can be directly represented by `entityType`.

---

# 82. Why the Coverage Gap Matters

Suppose:

```text
Breed
```

is updated.

Current AuditLog enum has no:

```text
entityType = Breed
```

Therefore a clean direct audit record cannot be represented using the finalized enum.

The same issue exists for:

```text
Vaccine
Disease
Medicine
Breeding
Pregnancy
FeedIngredient
```

This is an architectural consideration to review before implementation.

**No schema update is being applied here; this is only being flagged as a consideration, consistent with the current finalized model.**

---

# 83. AuditLog Should Not Be Used as a Workaround

Do not do:

```text
entityType = Farm
```

just because:

```text
Breed
```

is not available.

That would make the audit trail semantically incorrect.

If those entities must be audited directly, the `entityType` enum would need to be expanded in a future schema revision.

---

# 84. Login Audit

Recommended:

```text
entityType = User
entityId = User._id
action = Login
module = Authentication
```

Metadata:

```text
ipAddress
deviceType
browser
operatingSystem
location
requestId
status
createdAt
```

This gives a useful login history.

---

# 85. Failed Login

Example:

```text
entityType = User
action = Login
module = Authentication
status = Failed
```

The exact reason:

```text
wrong password
inactive account
unknown email
```

is not represented by a dedicated field in the finalized AuditLog model.

Therefore, detailed authentication error information should remain in the application's security/error logging system rather than being forced into `oldData/newData`.

---

# 86. Logout Audit

```text
entityType = User
action = Logout
module = Authentication
status = Success
```

This provides session activity history.

---

# 87. Export Audit

When a user exports:

```text
Finance Report
```

create:

```text
entityType = Income
action = Export
module = Finance
```

or use the most appropriate report-related entity context supported by the current enum.

Because `Report` is not currently an `entityType`, the audit design should use the actual exported business data context.

---

# 88. Import Audit

For:

```text
Animal CSV import
```

the audit can record:

```text
entityType = Animal
action = Import
module = Animal
```

and use:

```text
requestId
```

to correlate all records affected by that import batch.

---

# 89. Approve / Reject Audit

For workflows that support approval:

```text
action = Approve
```

or:

```text
action = Reject
```

should be recorded.

Example:

```text
FeedRecommendation
status:
confirmed → expert_approved
```

Audit:

```text
action = Approve
```

---

# 90. Audit on Every CRUD?

Not every database read needs an AuditLog.

The finalized actions do not include:

```text
View
Read
Search
List
```

This strongly suggests AuditLog is primarily for meaningful state/security actions, not every GET request.

This is also important for performance.

---

# 91. Do Not Audit Every Read

Bad approach:

```text
Every GET request
→ AuditLog
```

If the dashboard sends:

```text
50 API requests
```

one page load could generate:

```text
50 audit logs
```

This creates massive noise.

Better:

```text
Write/administrative/security actions
→ AuditLog

Normal reads
→ no audit
```

Exceptions can be defined for particularly sensitive records.

---

# 92. What Should Definitely Be Audited?

Recommended high-value actions:

```text
Create
Update
Delete
Archive
Approve
Reject
Login
Logout
Export
Import
```

These are already the finalized action enum.

---

# 93. Financial Operations — High Audit Priority

Definitely audit:

```text
Expense Create
Expense Update
Expense Cancel
Income Create
Income Update
Income Cancel
Inventory Purchase
Inventory Adjustment
```

because these affect financial/stock integrity.

---

# 94. Health Operations — High Audit Priority

Audit:

```text
Vaccination Create/Update
Treatment Create/Update
Treatment Status Change
```

because these affect animal health history.

---

# 95. Feed Operations — High Audit Priority

Audit:

```text
FeedFormula Create/Update
FeedRecommendation Create/Confirm/Reject
FeedRecord Create/Update
```

because recommendations and feeding history can affect decision-making and research evaluation.

---

# 96. User/Security Operations — High Audit Priority

Audit:

```text
Login
Logout
User changes
Farm membership changes
Administrative actions
Exports
Imports
```

The current entity/action enums cover some of these directly, while FarmMember is not currently an `entityType`.

---

# 97. FarmMember Coverage Limitation

The finalized `entityType` enum includes:

```text
Farm
User
```

but not:

```text
FarmMember
```

Therefore direct membership changes such as:

```text
Add member
Change role
Remove member
```

do not have a dedicated entity type in the current AuditLog schema.

This is another schema consideration to review before implementation.

---

# 98. AuditLog Data Immutability

Audit logs should generally be:

```text
Append-only
```

Meaning:

```text
Create AuditLog
       ↓
Never casually edit
       ↓
Never casually delete
```

Why?

Because if an administrator can modify audit history freely, the audit system loses its purpose.

---

# 99. No Normal CRUD Editing of AuditLog

Recommended:

```text
No normal:
PATCH /api/audit-logs/:id
```

and:

```text
No normal:
DELETE /api/audit-logs/:id
```

Audit records should be system-generated.

Administrative access should be read-oriented.

---

# 100. AuditLog Immutability

Example:

```text
Expense changed:
5000 → 5500
```

Audit:

```text
LOG-100
```

should not later be modified to:

```text
5000 → 5200
```

That would compromise historical integrity.

---

# 101. AuditLog Retention

The current DB model does not define:

```text
expiresAt
retentionDays
archivedAt
```

Therefore, the source does not specify a retention period.

A future production policy should define:

```text
How long audit records are retained
Who can access them
When/if they can be archived
```

Do not invent a specific retention period as part of the finalized model.

---

# 102. AuditLog Privacy

Fields such as:

```text
ipAddress
userAgent
location
```

can contain sensitive operational metadata.

Therefore access should be restricted.

Example:

```text
Worker
→ no full audit access

Farm Manager
→ limited audit access

Farm Owner
→ farm-level audit

Admin
→ system-level audit
```

The exact permission policy should align with the role system in the SRS. The SRS defines Admin, Farm Owner, Farm Manager, and Worker access levels. fileciteturn23file5

---

# 103. Farm-Level Audit Visibility

Farm Owner should normally see:

```text
their farm's audit logs
```

not:

```text
another farm's logs
```

This follows the SRS's cross-farm data isolation requirement. fileciteturn23file5

---

# 104. Admin-Level Audit

Admin may need system-wide access for:

```text
Security investigation
User activity
Farm activity
System administration
```

But this should be protected because audit data itself is sensitive.

---

# 105. AuditLog Query Filters

Useful filters:

```text
farmId
userId
entityType
entityId
action
module
status
deviceType
date range
requestId
```

Example:

```text
Finance
+
Expense
+
Update
+
August 2026
```

---

# 106. Audit Search Example

User searches:

```text
Expense
```

Then:

```text
entityType = Expense
```

Searches:

```text
Update
```

Then:

```text
action = Update
```

Searches:

```text
Failed
```

Then:

```text
status = Failed
```

---

# 107. Audit Timeline

A useful UI:

```text
17 Aug 2026 10:30
USER-002
Updated Animal A-001

Changed:
healthStatus
currentWeight

Healthy → Sick
180 → 175 kg

Status:
Success
```

This is much easier for human investigation than raw JSON.

---

# 108. Audit Detail View

Clicking an audit event can show:

```text
Log No
User
Farm
Entity
Entity ID
Action
Module
Status
Changed Fields
Old Data
New Data
IP
Device
Browser
OS
Location
Request ID
Timestamp
```

---

# 109. Changed Field Visualization

For:

```text
oldData
newData
```

the UI can display:

```text
healthStatus
Healthy  →  Sick

currentWeight
180 kg   → 175 kg
```

This makes the audit system very understandable.

---

# 110. Realtime Audit Dashboard

Because the original requirement was to see changes in realtime, use:

```text
MongoDB AuditLog
+
Socket.IO/WebSocket
```

Architecture:

```text
User Action
    ↓
Service
    ↓
DB Transaction
    ↓
AuditLog saved
    ↓
Commit
    ↓
audit.created event
    ↓
Audit Dashboard
```

The dashboard can show:

```text
"Farm Manager updated Animal A-001"
```

immediately.

---

# 111. Realtime Audit Must Not Depend Only on Socket

Bad:

```text
User action
→ socket event
→ dashboard
```

If socket delivery fails:

```text
audit history disappears
```

Better:

```text
User action
→ AuditLog persisted
→ realtime event
```

The database is the source of truth.

---

# 112. Realtime Channel Security

Do not broadcast every audit event to every connected user.

Example:

```text
Farm A audit event
```

should only be delivered to authorized Farm A viewers.

Conceptually:

```text
farm:{farmId}:audit
```

and user/role authorization controls who can subscribe.

---

# 113. Audit Event Payload

Realtime event can be lightweight:

```json
{
  "logId": "LOG-001",
  "farmId": "FARM-001",
  "entityType": "Animal",
  "entityId": "ANIMAL-001",
  "action": "Update",
  "module": "Animal",
  "status": "Success",
  "createdAt": "2026-08-17T10:30:00Z"
}
```

The frontend can fetch the full audit record if necessary.

This avoids broadcasting sensitive:

```text
oldData
newData
location
IP
```

to every client.

---

# 114. Audit + Transaction

For multi-model operations, use:

```text
MongoDB transaction
```

where required.

Example:

```text
Animal Sale
```

could involve:

```text
Animal update
+
Income creation
+
AuditLog
```

Desired behavior:

```text
BEGIN
   ↓
Update Animal
   ↓
Create Income
   ↓
Create AuditLog(s)
   ↓
COMMIT
```

If critical operation fails:

```text
ROLLBACK
```

---

# 115. Rolled Back Audit

A subtle point:

If the MongoDB transaction itself rolls back, an AuditLog created **inside that same transaction** will also roll back.

Therefore, if Liventra needs a durable record saying:

```text
Operation was rolled back
```

the architecture must deliberately handle that event outside the rolled-back transaction or through a separate reliable logging mechanism.

This is an implementation-level concern; the finalized schema only provides the `Rolled Back` status.

---

# 116. Example — Animal Sale Failure

Suppose:

```text
Create Income
    ↓
Update Animal
    ↓
Failure
```

Transaction:

```text
ROLLBACK
```

The final database state should be:

```text
No partial Income
Animal unchanged
```

A separate application/error/audit mechanism can record:

```text
status = Rolled Back
```

if required.

---

# 117. AuditLog and Error Logging

Recommended architecture:

```text
Business Operation
      │
      ├── AuditLog
      │
      └── Error Logger
```

AuditLog answers:

```text
What business action happened?
```

Error logger answers:

```text
What technical error occurred?
```

Both can share:

```text
requestId
```

for correlation.

---

# 118. Request Correlation

Example:

```text
requestId = REQ-123
```

can appear in:

```text
AuditLog
Application Log
Error Log
API Response Log
```

This creates a trace:

```text
User action
→ API request
→ DB operation
→ error
```

---

# 119. Audit Logging Middleware vs Service Layer

A generic Express middleware can capture:

```text
request
user
IP
device
requestId
```

But it cannot always know:

```text
what exact business fields changed
```

Therefore the best design is usually:

```text
Middleware
→ request/security metadata

Service/domain layer
→ business event + old/new data
```

Then combine the information into AuditLog.

---

# 120. Why Pure Middleware Is Not Enough

Suppose:

```text
PATCH /animals/A-001
```

Middleware knows:

```text
PATCH
URL
User
IP
```

but may not know:

```text
healthStatus:
Healthy → Sick
```

The Animal service does know that.

Therefore:

```text
Audit creation
```

should be integrated closely with the business service or repository transaction.

---

# 121. Generic Audit Service

Recommended:

```text
auditService.log({
    farmId,
    userId,
    entityType,
    entityId,
    action,
    module,
    oldData,
    newData,
    changedFields,
    requestContext,
    status
})
```

This centralizes audit creation.

---

# 122. Request Context

The request context can provide:

```text
ipAddress
userAgent
deviceType
browser
operatingSystem
location
requestId
```

The business service provides:

```text
entityType
entityId
action
module
oldData
newData
changedFields
status
```

This is a clean separation.

---

# 123. Audit Diff Generation

For an update:

```text
oldData
newData
```

the backend can calculate:

```text
changedFields
```

automatically.

Example:

```text
old:
{
  amount: 5000,
  status: "Paid"
}

new:
{
  amount: 5500,
  status: "Paid"
}
```

Diff:

```text
["amount"]
```

This reduces frontend manipulation and makes the audit trustworthy.

---

# 124. Never Trust Frontend Audit Data

Bad:

```text
Frontend sends:
oldData
newData
```

Backend blindly saves it.

A malicious client could say:

```text
oldData = 100
newData = 200
```

even if actual DB value was:

```text
5000
```

Better:

```text
Backend reads old DB state
        ↓
Performs update
        ↓
Determines new state
        ↓
Creates AuditLog
```

This makes the audit trail server-controlled.

---

# 125. Financial Audit Integrity

For:

```text
Expense
Income
```

audit data should be generated by the backend.

Example:

```text
Database:
Expense.amount = 5000

Request:
amount = 5500

Backend:
old = 5000
new = 5500
```

Then AuditLog is created from trusted state.

---

# 126. AuditLog and Soft Delete

If the main collection uses:

```text
isActive = false
```

or:

```text
status = Archived
```

then:

```text
action = Archive
```

can record the transition.

This is preferable to simply deleting data when historical continuity matters.

---

# 127. AuditLog and Data Restoration

If an archived/deleted record is restored, the finalized action enum has no:

```text
Restore
```

Therefore restoration would need to be represented using an available action such as:

```text
Update
```

if the current schema is retained.

This is a current-schema limitation.

---

# 128. AuditLog and Bulk Updates

Suppose:

```text
100 animals
```

are updated.

Possible strategies:

### Individual audit records

```text
100 AuditLogs
```

Best traceability.

### Batch-level audit

```text
1 AuditLog
```

Less storage, less granularity.

The current schema has no:

```text
batchId
```

field.

`requestId` can provide some correlation, but it is not necessarily a perfect business batch identifier.

---

# 129. Recommended Bulk Strategy

For high-value business changes:

```text
Individual entity AuditLog
```

is preferable.

For large imports:

```text
requestId
```

can correlate the batch.

The exact strategy should depend on scale.

---

# 130. AuditLog Performance

Audit logs can become very large.

If Liventra has:

```text
100 animals
+
daily milk entries
+
feed records
+
inventory
+
financial operations
```

the AuditLog collection can grow rapidly.

Therefore:

```text
Indexes
Pagination
Date filters
Selective auditing
Retention strategy
```

are essential.

---

# 131. Recommended Indexes

Primary timeline query:

```javascript
{
    farmId: 1,
    createdAt: -1
}
```

User activity:

```javascript
{
    farmId: 1,
    userId: 1,
    createdAt: -1
}
```

Entity history:

```javascript
{
    farmId: 1,
    entityType: 1,
    entityId: 1,
    createdAt: -1
}
```

Module filtering:

```javascript
{
    farmId: 1,
    module: 1,
    createdAt: -1
}
```

Request tracing:

```javascript
{
    requestId: 1
}
```

---

# 132. Entity History Query

For:

```text
Animal A-001
```

query:

```text
farmId = FARM-001
entityType = Animal
entityId = A-001
sort createdAt DESC
```

Result:

```text
Created
Updated
Updated
Archive
```

This creates an Animal audit timeline.

---

# 133. User Activity Query

For:

```text
User U-001
```

query:

```text
farmId = FARM-001
userId = U-001
```

This can show:

```text
Login
Animal update
Expense create
Report export
Logout
```

---

# 134. Finance Audit Query

```text
module = Finance
```

can show:

```text
Expense Create
Expense Update
Income Create
Income Update
```

This is useful for financial review.

---

# 135. Security Audit Query

```text
module = Authentication
```

can show:

```text
Login
Logout
Failed login
```

subject to the exact implementation mapping.

---

# 136. Administration Audit Query

```text
module = Administration
```

can show administrative events such as:

```text
User changes
Notification management
Administrative exports
```

according to the entity/action mapping.

---

# 137. Audit Report

The finalized Report model explicitly supports:

```text
reportType = Audit Log
```

and formats:

```text
PDF
Excel
CSV
JSON
```

fileciteturn23file0

Therefore AuditLog can be exported as formal reports.

---

# 138. Audit Report Workflow

```text
User selects:
Audit Log

Date:
Aug 1 → Aug 31

Filters:
Finance
Update

        ↓

Generate Report
        ↓
Report collection
        ↓
PDF / Excel / CSV / JSON
```

The report itself should also be auditable if the application considers report generation an audit-worthy action.

---

# 139. Audit Report Security

Audit reports can contain:

```text
IP
Location
Financial changes
User activity
```

Therefore only authorized users should be able to generate/download them.

---

# 140. Audit Export Auditing

When an authorized user exports audit data:

```text
action = Export
```

should itself be logged.

This creates:

```text
Audit log
   ↓
Audit report exported
   ↓
Another AuditLog
```

This is useful for accountability.

---

# 141. AuditLog + Notification

If a critical audit event occurs, the system could generate a Notification.

Example:

```text
Multiple failed authentication attempts
```

However, the current Notification reference enum does not include:

```text
AuditLog
```

so any such notification would need to use:

```text
Manual
```

under the current finalized schema.

---

# 142. AuditLog + Realtime Security Alert

For high-risk events:

```text
Failed authentication
Suspicious cross-farm access
Large financial modification
```

architecture can be:

```text
AuditLog
    ↓
Rule Engine
    ↓
Notification
    ↓
Realtime Alert
```

This is an architectural extension, not a direct DB relationship.

---

# 143. Audit Severity Limitation

The finalized AuditLog has:

```text
status
```

but no:

```text
severity
riskLevel
```

Therefore the current schema cannot directly classify audit events as:

```text
Low
Medium
High
Critical
```

without using external/application logic.

---

# 144. Audit Reason Limitation

There is no:

```text
reason
comment
remarks
```

field.

Therefore:

```text
Why was this record deleted?
Why was this recommendation rejected?
Why was this expense cancelled?
```

cannot be stored in a dedicated AuditLog field.

If a business workflow requires a reason, it should ideally be captured in the affected business record or a future schema enhancement.

---

# 145. Audit Source Limitation

There is no explicit:

```text
source
```

field such as:

```text
Web
Mobile
API
System
Scheduler
IoT
```

The current `deviceType` provides:

```text
Desktop
Mobile
Tablet
API
```

but does not fully distinguish:

```text
System scheduler
IoT
Background worker
```

Therefore automated/system-generated actions need a consistent implementation convention.

---

# 146. Audit Correlation Limitation

`requestId` is excellent for:

```text
HTTP request correlation
```

but not necessarily enough for:

```text
long-running background job
scheduled task
multi-step workflow
```

There is no dedicated:

```text
jobId
transactionId
batchId
```

field.

This should be considered for future scaling.

---

# 147. AuditLog and IoT

Future SensorData will generate potentially huge event volumes.

Do **not** blindly create an AuditLog for every sensor reading.

Example:

```text
Sensor:
1 reading/second
```

would produce:

```text
86,400 audit logs/day
```

for one sensor.

That is inappropriate.

Instead:

```text
SensorData
=
telemetry history

AuditLog
=
important configuration/administrative events
```

For example:

```text
Device assignment
Device configuration
Manual sensor-data import
Critical alert acknowledgement
```

can be audited.

---

# 148. AuditLog + Device Future Scope

The finalized `entityType` does not currently include:

```text
Device
DeviceAssignment
SensorData
```

even though these collections are upcoming. fileciteturn23file0

Therefore future IoT auditing will require either:

```text
future entityType expansion
```

or another audit architecture.

Do not force IoT telemetry into the current AuditLog design.

---

# 149. AuditLog Should Be Business-Event Oriented

A good rule:

> **Audit meaningful changes, not every piece of data movement.**

Examples:

### Audit

```text
Animal health status changed
Expense changed
Income cancelled
Feed recommendation approved
User logged in
Report exported
Inventory adjusted
```

### Do not normally audit individually

```text
Sensor reading received
Dashboard loaded
Animal list viewed
Notification opened
```

unless a specific compliance/security requirement exists.

---

# 150. Audit Event Naming

The finalized schema uses:

```text
action = Create | Update | Delete | ...
```

Therefore keep action values standardized.

Do not create inconsistent values such as:

```text
Created
Created_New
AnimalCreated
UPDATED
```

outside the enum.

---

# 151. Audit Module Consistency

Every AuditLog should have a valid:

```text
module
```

matching the functional area.

Example:

```text
Expense
→ Finance

Vaccination
→ Health

FeedRecommendation
→ Feed
```

This makes reporting/filtering predictable.

---

# 152. Audit Entity Consistency

Every AuditLog should have:

```text
entityType
entityId
```

consistent with the actual operation.

This is particularly important because MongoDB does not automatically enforce polymorphic reference correctness.

---

# 153. Audit Creation Pattern

Recommended high-level service pattern:

```text
1. Authenticate user
2. Determine farm context
3. Authorize action
4. Read old state if needed
5. Perform business operation
6. Determine new state
7. Calculate changed fields
8. Create AuditLog
9. Commit transaction
10. Emit realtime audit event
```

For failed authorization:

```text
Authorization fails
    ↓
Create security/audit event if appropriate
    ↓
Return Forbidden
```

---

# 154. Update Pattern

Example:

```text
PATCH /animals/:id
```

Flow:

```text
Request
  ↓
Authentication
  ↓
Farm authorization
  ↓
Load old Animal
  ↓
Validate update
  ↓
Update Animal
  ↓
Build diff
  ↓
Create AuditLog
  ↓
Commit
  ↓
Emit audit.created
```

---

# 155. Delete Pattern

```text
Request
  ↓
Authorization
  ↓
Load old record
  ↓
Delete/archive
  ↓
Audit old state
  ↓
Commit
  ↓
Realtime event
```

For hard deletes, capturing the old state before deletion is essential if the audit record needs historical detail.

---

# 156. Create Pattern

```text
Validate request
    ↓
Create entity
    ↓
Audit:
oldData = null
newData = relevant created state
    ↓
Commit
```

---

# 157. Export Pattern

```text
User requests export
    ↓
Authorize
    ↓
Generate export
    ↓
Success?
   /   \
 Yes    No
 |       |
Audit   Audit
Success Failed
```

---

# 158. Import Pattern

```text
Upload file
    ↓
Validate
    ↓
Import records
    ↓
Audit import event
    ↓
Use requestId to correlate operation
```

If partial imports are allowed, the system should clearly distinguish:

```text
successful records
failed records
```

The current AuditLog model only has a single:

```text
status
```

per log record, so batch-level reporting may need additional application-level handling.

---

# 159. AuditLog and Authentication Security

Login audits should never store:

```text
password
password hash
OTP
refresh token
JWT
secret keys
```

inside:

```text
oldData
newData
```

The finalized model does not explicitly prohibit this, but from a security perspective sensitive credentials should never enter audit snapshots.

---

# 160. AuditLog and Personal Data

Avoid storing unnecessary sensitive data in:

```text
oldData
newData
location
userAgent
```

Audit data should contain enough information for accountability without becoming a second uncontrolled copy of every personal record.

---

# 161. Audit Snapshot Sanitization

Before storing:

```text
oldData
newData
```

the backend should remove fields that should never be logged.

For example:

```text
User.password
```

must not appear in AuditLog.

A reusable:

```text
sanitizeForAudit()
```

utility is recommended.

---

# 162. Audit Logging Rules by Entity

Recommended:

```text
User
→ Login, Logout, Create, Update, Archive

Farm
→ Create, Update, Archive

Animal
→ Create, Update, Archive, Delete

FeedFormula
→ Create, Update, Archive, Delete

FeedRecommendation
→ Create, Update, Approve, Reject, Archive

FeedRecord
→ Create, Update, Delete

WeightRecord
→ Create, Update, Delete

MilkRecord
→ Create, Update, Delete

Vaccination
→ Create, Update, Archive, Delete

Treatment
→ Create, Update, Archive, Delete

Inventory
→ Create, Update, Archive

InventoryTransaction
→ Create, Update, Import

Expense
→ Create, Update, Archive/Delete, Approve/Reject where applicable

Income
→ Create, Update, Archive/Delete, Approve/Reject where applicable

Notification
→ Create, Archive

```

These are recommended audit priorities based on the finalized action/entity structure.

---

# 163. AuditLog Should Not Replace History Collections

Important distinction:

```text
WeightRecord
=
actual historical weight measurements

AuditLog
=
who changed the WeightRecord
```

Similarly:

```text
MilkRecord
=
milk production history

AuditLog
=
modification history of MilkRecord
```

Do not use AuditLog as a substitute for operational history.

---

# 164. Animal History Example

Animal history may combine:

```text
WeightRecord
MilkRecord
Vaccination
Treatment
Breeding
Pregnancy
```

AuditLog adds:

```text
who modified each record
```

So:

```text
Operational history
+
Audit history
```

are complementary.

---

# 165. Financial History Example

Finance:

```text
Expense
Income
```

Audit:

```text
Expense changed by User A
Income cancelled by User B
```

This is especially useful during financial review.

---

# 166. Audit Dashboard Metrics

A useful admin/farm-owner dashboard can show:

```text
Today's audit events
Failed operations
Financial changes
User activity
Recent deletions
Recent exports
Recent approvals
```

These are derived metrics.

---

# 167. Recent Audit Activity

Example:

```text
10:30 — Manager updated Animal A-001
10:25 — Worker created MilkRecord
10:20 — Owner exported Finance Report
10:15 — Manager updated Inventory
10:10 — User login
```

This provides operational visibility.

---

# 168. Failed Action Monitoring

Filter:

```text
status = Failed
```

can identify:

```text
Unauthorized attempts
Validation failures
System failures
```

However, technical error details should remain in the application's error logs.

---

# 169. Rolled Back Monitoring

Filter:

```text
status = Rolled Back
```

can identify:

```text
Transaction failures
Multi-model consistency problems
```

This is particularly valuable for:

```text
Animal sale
Inventory purchase
Financial operations
```

---

# 170. Audit Report Example

```text
Audit Log Report
Farm: Green Valley Farm
Date: 01–31 August 2026

Total Events:
1,248

Create:
530

Update:
612

Delete:
18

Export:
8

Approve:
15

Reject:
6

Failed:
49

Rolled Back:
10
```

These figures would be derived from the AuditLog collection.

---

# 171. Audit Data Pagination

Never return thousands of logs in one API response.

Use:

```text
page
limit
```

or:

```text
cursor
```

Example:

```text
GET /api/audit-logs?page=1&limit=50
```

---

# 172. Audit Date Filtering

Very important:

```text
startDate
endDate
```

because audit collections grow rapidly.

Example:

```text
GET /api/audit-logs?from=2026-08-01&to=2026-08-31
```

---

# 173. Audit Entity History API

Example:

```text
GET /api/audit-logs/entity/Animal/ANIMAL-001
```

This can return:

```text
Animal created
Animal updated
Health status changed
Weight changed
Animal archived
```

The exact endpoint naming is implementation-specific.

---

# 174. Audit User Activity API

Example:

```text
GET /api/audit-logs/user/:userId
```

Useful for:

```text
User activity
Security review
Administrative investigation
```

---

# 175. Audit Finance API

Example:

```text
GET /api/audit-logs?module=Finance
```

This can show:

```text
Expense
Income
```

changes.

---

# 176. Audit Security API

Example:

```text
GET /api/audit-logs?module=Authentication
```

This can show:

```text
Login
Logout
```

events.

---

# 177. Recommended Internal API Methods

```text
auditService.createLog()
auditService.logCreate()
auditService.logUpdate()
auditService.logDelete()
auditService.logArchive()
auditService.logLogin()
auditService.logLogout()
auditService.logExport()
auditService.logImport()
auditService.logApproval()
auditService.logRejection()

auditService.getLogs()
auditService.getEntityHistory()
auditService.getUserActivity()
auditService.getFailedEvents()
auditService.getModuleActivity()
```

These are service-level responsibilities.

---

# 178. Audit Repository Methods

Possible:

```text
findByFarm()
findByUser()
findByEntity()
findByRequestId()
findByModule()
findByAction()
findByStatus()
findByDateRange()
```

This keeps query logic separate from business logic.

---

# 179. Audit Controller Responsibility

Controller should handle:

```text
Authentication
Authorization
Query validation
Pagination
Filter parsing
Response formatting
```

It should not calculate:

```text
oldData
newData
changedFields
```

Those belong to the audit/business layer.

---

# 180. Audit Middleware Responsibility

Middleware is ideal for:

```text
requestId
IP
userAgent
device detection
authentication context
```

Then the service layer combines those with:

```text
entity
action
old/new data
```

---

# 181. Audit Event Processing

A scalable architecture:

```text
Business Service
      ↓
Audit Event
      ↓
Queue
      ↓
Audit Worker
      ↓
MongoDB AuditLog
```

However, for critical financial/security operations, asynchronous logging must be designed carefully because you do not want the business action to succeed while the audit event is silently lost.

For the project's current scale, synchronous transactional logging for important state changes is simpler and safer.

---

# 182. Audit Logging Strategy for Liventra

Recommended priority:

### Synchronous / transaction-coupled

```text
Expense
Income
InventoryTransaction
Animal sale
Treatment
Vaccination
FeedRecommendation approval
User/farm security changes
```

### Lower-priority / asynchronous where appropriate

```text
Large export events
Bulk import processing
Some administrative events
```

### Do not audit individually

```text
Sensor telemetry
Normal GET requests
Dashboard reads
```

---

# 183. Critical Audit Principle

> **If the business operation is successful, the audit event for that operation should not be silently lost.**

For important changes:

```text
Business update
+
AuditLog
```

should be treated as one logical operation.

---

# 184. Audit Consistency Example

Bad:

```text
Expense updated
    ↓
Audit service crashes
    ↓
No audit record
```

Better:

```text
Expense update
+
AuditLog
```

within a MongoDB transaction where practical.

---

# 185. AuditLog and MongoDB Transactions

Example:

```text
session.startTransaction()

Update Expense

Create AuditLog

session.commitTransaction()
```

If something fails:

```text
session.abortTransaction()
```

For rolled-back operations requiring a durable failure record, use an external/application log or post-rollback mechanism.

---

# 186. AuditLog and Realtime Ordering

Correct order:

```text
DB commit
   ↓
AuditLog exists
   ↓
Emit realtime event
```

Not:

```text
Emit realtime
   ↓
DB save
```

because the database should remain the source of truth.

---

# 187. Audit Realtime Failure

Suppose:

```text
AuditLog saved
```

but:

```text
Socket connection fails
```

Result:

```text
Audit history is still safe.
```

When the user reloads:

```text
GET audit logs
```

will retrieve it.

This is why realtime is a delivery optimization, not the source of truth.

---

# 188. AuditLog and Offline Support

If an admin is offline:

```text
Audit events continue being stored
```

When the admin returns:

```text
Fetch audit history
```

and sees the missed events.

---

# 189. AuditLog Data Size

`oldData` + `newData` can make documents large.

Therefore:

```text
Do not blindly snapshot massive fields
```

Examples:

```text
Animal.gallery
large attachments
Sensor rawPayload
```

should not unnecessarily be copied into every audit record.

This is particularly important for future IoT data.

---

# 190. Audit Snapshot Exclusion

For large/non-auditable fields, the audit service can exclude:

```text
large image arrays
binary content
file content
raw sensor payload
passwords
tokens
```

The current DB model does not define a dedicated snapshot schema, so this must be handled in the audit service.

---

# 191. Audit of Attachments

For:

```text
Expense.attachment
Income.attachment
Animal.profilePhoto
```

Audit should generally store:

```text
file URL/path changed
```

rather than:

```text
actual file binary
```

This avoids massive audit documents.

---

# 192. Audit of Gallery Changes

For Animal:

```text
gallery[]
```

can contain multiple images.

Audit should ideally describe:

```text
gallery item added
gallery item removed
gallery caption changed
```

rather than duplicating every image object in old/new snapshots.

---

# 193. AuditLog and Data Versioning

AuditLog provides historical changes, but it is not a full version-control system.

Example:

```text
Animal version 1
Animal version 2
Animal version 3
```

Audit records can reconstruct meaningful changes, but the current schema does not contain:

```text
versionNumber
```

Therefore do not claim built-in document version numbering.

---

# 194. AuditLog and Rollback Feature

The finalized AuditLog contains:

```text
status = Rolled Back
```

but this does **not** mean the system automatically supports:

```text
Restore this oldData
```

There is no `Restore` action and no automatic reverse-update engine.

So:

```text
Audit history
≠
Automatic undo system
```

---

# 195. If Undo Is Needed Later

A future service could:

```text
Read AuditLog
    ↓
Validate oldData
    ↓
Create reverse update
    ↓
Create new AuditLog
```

But this should be an explicit feature, not assumed from the current schema.

---

# 196. AuditLog and Research/Thesis Value

For Liventra's intelligent feed recommendation component, auditability is particularly useful.

Example:

```text
Animal weight:
180 kg

Recommendation:
Napier 12 kg
Concentrate 3 kg

Rule Version:
v1.4

User:
Manager

Action:
Approve
```

AuditLog can preserve:

```text
Who approved the recommendation
When
```

while FeedRecommendation itself preserves:

```text
Input snapshot
Recommendation output
Rule version
```

Together they provide stronger historical traceability.

---

# 197. AuditLog + FeedRecommendation

This distinction is important:

```text
FeedRecommendation
=
What recommendation was generated?

AuditLog
=
Who approved/changed/rejected it?
```

This is exactly the kind of separation needed for expert validation and historical evaluation.

---

# 198. AuditLog + Finance

Similarly:

```text
Expense
=
What expense exists?

AuditLog
=
Who changed/cancelled it?
```

and:

```text
Income
=
What income exists?

AuditLog
=
Who changed/cancelled it?
```

This gives the finance module accountability.

---

# 199. AuditLog + Inventory

```text
Inventory
=
Current stock

InventoryTransaction
=
Stock movement

AuditLog
=
Who created/modified the stock movement
```

Three different responsibilities.

---

# 200. AuditLog + Health

```text
Vaccination
=
Vaccination event

Treatment
=
Treatment plan/history

AuditLog
=
Who changed the health records
```

This preserves medical accountability without replacing health history.

---

# 201. AuditLog + User/Farm

```text
User
=
Current account

Farm
=
Current farm

AuditLog
=
Account/farm changes and authentication activity
```

---

# 202. Security Event Example

Potential suspicious behavior:

```text
User repeatedly attempts to access
another farm's Animal records.
```

Each failed operation can be recorded as:

```text
status = Failed
```

with:

```text
userId
farmId
entityType
entityId
requestId
ipAddress
deviceType
createdAt
```

This creates an investigation trail.

---

# 203. AuditLog Does Not Automatically Detect Attacks

Important:

```text
AuditLog
=
records events
```

It does not itself detect:

```text
brute force
anomalous behavior
account takeover
```

A future security/rule engine can analyze audit events.

---

# 204. Security Analytics

Derived metrics can include:

```text
Failed logins/user
Failed requests/farm
Financial changes/user
Exports/user
Deletes/user
```

These can help identify suspicious activity.

---

# 205. AuditLog + Notification Security Alert

Future architecture:

```text
AuditLog
    ↓
Security Rule Engine
    ↓
Suspicious pattern?
    ↓
Notification
```

Example:

```text
10 failed login attempts
```

could trigger:

```text
Critical security alert
```

But this is future application logic, not a direct current-schema relationship.

---

# 206. Final Business Rules Checklist

```text
[✓] AuditLog stores historical accountability events
[✓] AuditLog is separate from operational data
[✓] AuditLog is separate from application error logging
[✓] AuditLog is separate from realtime transport
[✓] logNo should be backend-generated
[✓] farmId provides farm scope
[✓] userId identifies the actor for user-driven actions
[✓] Cross-farm actions must be prevented and important failures can be audited
[✓] entityType uses the finalized enum
[✓] entityId identifies the affected record
[✓] entityType + entityId must be consistent
[✓] action uses finalized enum
[✓] module uses finalized enum
[✓] oldData stores relevant previous state
[✓] newData stores relevant resulting state
[✓] changedFields identifies exact modified fields
[✓] Backend should calculate audit diffs
[✓] Frontend must not be trusted to construct audit history
[✓] Passwords/tokens/secrets must never enter audit snapshots
[✓] IP metadata can support security investigation
[✓] deviceType uses finalized enum
[✓] location is optional contextual metadata
[✓] requestId correlates audit with API/application logs
[✓] status distinguishes Success/Failed/Rolled Back
[✓] createdAt defines audit timeline
[✓] AuditLog should generally be append-only
[✓] Normal CRUD editing of AuditLog should not be allowed
[✓] Normal GET requests should not generate audit records
[✓] Financial changes should be strongly audited
[✓] Health changes should be strongly audited
[✓] Feed recommendation approval/rejection should be audited
[✓] Login/logout should be audited
[✓] Export/import should be audited
[✓] AuditLog can power an audit report
[✓] Realtime audit viewing should use Socket.IO/WebSocket on top of DB persistence
[✓] Save audit event before realtime emission
[✓] Realtime events must respect farm/role authorization
[✓] Audit history should remain available if user was offline
[✓] Pagination is required for large audit history
[✓] Date filtering is essential
[✓] Compound indexes are recommended
[✓] oldData/newData should avoid huge unnecessary snapshots
[✓] Sensor telemetry should not be audited one reading at a time
[✓] MongoDB transactions should be considered for critical multi-model operations
[✓] Rolled Back status alone does not automatically provide durable rollback history
[✓] AuditLog is not an automatic undo system
[✓] Current entityType does not cover every finalized collection
[✓] Breed is not currently covered
[✓] FeedIngredient is not currently covered
[✓] Vaccine is not currently covered
[✓] Disease is not currently covered
[✓] Medicine is not currently covered
[✓] Breeding is not currently covered
[✓] Pregnancy is not currently covered
[✓] FarmMember is not currently covered
[✓] Future Device/DeviceAssignment/SensorData are not currently covered
```

---

# 207. Current Schema Coverage — Important Review

The finalized model is strong for the main operational modules:

```text
Animal
Feed
Production
Health
Inventory
Finance
Notification
Farm/User
```

But there is a clear coverage gap.

### Directly auditable:

```text
Animal
FeedFormula
FeedRecommendation
FeedRecord
WeightRecord
MilkRecord
Vaccination
Treatment
Inventory
InventoryTransaction
Expense
Income
Notification
Farm
User
```

### Not directly represented in `entityType`:

```text
FarmMember
Breed
FeedIngredient
Vaccine
Disease
Medicine
Breeding
Pregnancy
```

### Future models not currently represented:

```text
Device
DeviceAssignment
SensorData
```

This is the biggest architectural point to review before implementing AuditLog.

The finalized DB Models explicitly list these collections separately. fileciteturn23file0

---

# 208. Current Schema Coverage — Do Not Silently Change

For this documentation, the finalized schema is preserved exactly.

Therefore, this document does **not** add:

```text
Breed
Vaccine
Medicine
Pregnancy
...
```

to the current enum.

Instead, it flags the coverage gap.

If the team later decides:

> "Every mutable collection must have audit coverage."

then `entityType` should be revisited before implementation.

---

# 209. Recommended Audit Architecture for Liventra

The overall recommended architecture is:

```text
                         User Action
                              │
                              ▼
                    Authentication / RBAC
                              │
                              ▼
                     Business Controller
                              │
                              ▼
                      Domain Service
                              │
              ┌───────────────┴───────────────┐
              ▼                               ▼
       Main DB Operation                Audit Context
              │                               │
              └───────────────┬───────────────┘
                              ▼
                         Transaction
                              │
                    ┌─────────┴─────────┐
                    ▼                   ▼
               Main Record          AuditLog
                    │                   │
                    └─────────┬─────────┘
                              ▼
                           COMMIT
                              │
                              ▼
                     Realtime Event
                              │
                              ▼
                    Audit Dashboard
```

---

# 210. The "Best Way" for Realtime Audit

For the earlier requirement:

> "changes gula audit logs e rakhbo, jeta ekdom realtime-o dekha jabe"

the recommended approach is:

```text
MongoDB
+
AuditLog collection
+
Socket.IO
+
Audit Dashboard
```

### Why?

MongoDB provides:

```text
Permanent history
```

Socket.IO provides:

```text
Instant UI updates
```

Together:

```text
Persistent + Realtime
```

---

# 211. Realtime Audit Flow

Example:

```text
Manager changes Animal A-001
            ↓
Backend validates
            ↓
Animal updated
            ↓
AuditLog created
            ↓
Transaction committed
            ↓
socket.emit("audit.created")
            ↓
Farm Owner's audit dashboard
updates instantly
```

---

# 212. Realtime Audit Security

Never do:

```text
io.emit("audit.created", auditLog)
```

to everyone.

Instead use authorized rooms:

```text
farm:FARM-001:audit
```

Only authorized viewers join that room.

For sensitive metadata:

```text
oldData
newData
IP
location
```

consider sending only to users who have sufficient permission.

---

# 213. Audit Dashboard Filters

Recommended UI:

```text
Farm
User
Module
Entity
Action
Status
Date Range
```

Example:

```text
Module:
Finance

Action:
Update

Date:
August 2026
```

---

# 214. Audit Dashboard Live Mode

A useful UI could have:

```text
🔴 LIVE

10:42:01
Manager updated Animal A-001

10:41:47
Worker created MilkRecord

10:41:20
Owner exported Finance Report
```

A small:

```text
LIVE
```

indicator tells the user the dashboard is receiving realtime audit events.

---

# 215. Final Model Summary

```text
                         AuditLog
                            │
       ┌────────────────────┼─────────────────────┐
       │                    │                     │
       ▼                    ▼                     ▼
     WHO                   WHAT                 WHEN
   userId              entityType            createdAt
                         entityId
       │                    │                     │
       └────────────────────┼─────────────────────┘
                            ▼
                          ACTION
                            │
                 Create / Update / Delete
                 Archive / Login / Logout
                 Export / Import
                 Approve / Reject
                            │
                            ▼
                     CHANGE DETAILS
                            │
              ┌─────────────┴─────────────┐
              ▼                           ▼
           oldData                     newData
              │                           │
              └─────────────┬─────────────┘
                            ▼
                     changedFields
                            │
                            ▼
                     CONTEXT / TRACE
                            │
          IP + Device + Browser + OS
          Location + Request ID
                            │
                            ▼
                         STATUS
                  Success / Failed /
                     Rolled Back
```

The main responsibility of `AuditLog` is:

> **Maintain a trustworthy, append-oriented historical record of important user/system actions across Liventra, including who performed the action, which supported entity was affected, what changed, the request/device context, and whether the operation succeeded, failed, or was rolled back.**

---

# 216. Final Status

**Model:** `AuditLog`  
**Status:** ✅ Finalized  
**Business Logic:** ✅ Documented  
**Change Tracking:** ✅  
**Old/New Snapshot:** ✅  
**Field-Level Diff:** ✅  
**User Accountability:** ✅  
**Farm Isolation:** ✅  
**Security Context:** ✅  
**Financial Audit:** ✅  
**Health Audit:** ✅  
**Feed Audit:** ✅  
**Inventory Audit:** ✅  
**Authentication Audit:** ✅  
**Import/Export Audit:** ✅  
**Approval/Rejection Audit:** ✅  
**Realtime Architecture:** ✅  
**Socket.IO Integration Strategy:** ✅  
**Transaction Strategy:** ✅  
**Report Integration:** ✅  
**Performance / Indexing:** ✅  
**Privacy Considerations:** ✅  
**Current Schema Limitations:** ✅ Documented

### Most Important Review Point Before Implementation

The finalized `AuditLog.entityType` currently does **not** cover every finalized collection. In particular:

```text
FarmMember
Breed
FeedIngredient
Vaccine
Disease
Medicine
Breeding
Pregnancy
```

and future:

```text
Device
DeviceAssignment
SensorData
```

are outside the current `entityType` enum. fileciteturn23file0

So **the model itself is documented as finalized here, but this coverage gap should be consciously accepted or addressed before coding the audit system**.

---

## Project Documentation Progress

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
24 AuditLog  ← CURRENT
```

**Next → `Device` (upcoming)**
