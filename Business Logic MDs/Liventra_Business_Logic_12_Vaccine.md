# Liventra — Business Logic Documentation
## Model 12: Vaccine

**Document Version:** Business Logic v1.0  
**Project:** Liventra — Smart Livestock Farm Management System  
**Database:** MongoDB  
**Architecture:** MERN  
**Module:** Health & Vaccination

---

# 1. Source / Finalized Model

The finalized `Vaccine` model is:

```text
Vaccine Model:
_id
vaccineCode
vaccineName
shortCode
targetDiseaseIds[]
manufacturer
category: Viral | Bacterial | Parasitic | Combined
dosage
dosageUnit: ml | mg | tablet | dose
administrationRoute: IM | SC | IV | Oral | Nasal
minimumAge
boosterRequired
boosterAfterDays
repeatIntervalDays
storageTemperature
description
isMandatory
isActive
createdAt
updatedAt
```

This is the Vaccine structure in the current finalized Liventra DB model. The same source also defines `Vaccination` separately, with `vaccineId` referencing this master vaccine record. 

---

# 2. Very Important: Vaccine vs Vaccination

These two models must not be confused.

## Vaccine

```text
What is the vaccine?
```

It is a **master/reference definition**.

Example:

```text
FMD Vaccine
Code: VAC-FMD-001
Manufacturer: Company X
Target Disease: Foot and Mouth Disease
Dose: 2 ml
Route: IM
Booster: Yes
```

## Vaccination

```text
What happened to this particular animal?
```

Example:

```text
Animal A
received FMD Vaccine
on 2026-08-17
next due = 2026-09-16
```

Therefore:

```text
Vaccine
   ↓
Master definition

Vaccination
   ↓
Actual vaccination event
```

The finalized DB model explicitly separates these two entities and connects `Vaccination.vaccineId` to the Vaccine master record.

---

# 3. Vaccine's Role in Liventra

The health architecture becomes:

```text
Disease
   ↓
recommendedVaccineIds[]
   ↓
Vaccine
   ↓
Vaccination
   ↓
Animal
```

And operationally:

```text
Vaccine
   ↓
Inventory
   ↓
Vaccination
   ↓
Inventory_Transaction
```

This means Vaccine is not merely a list of vaccine names.

It acts as the **master configuration used to plan, validate, record and manage vaccination**.

---

# 4. Why Vaccine Needs Its Own Collection

Suppose Liventra only had:

```text
Vaccination
```

with:

```text
vaccineId
```

but no Vaccine collection.

Then:

```text
vaccineId → ???
```

would have no proper master reference.

The Vaccine collection solves this.

```text
Vaccine
_id = V001

        ↓

Vaccination
vaccineId = V001
```

---

# 5. Master Data Concept

`Vaccine` is master/reference data.

It should generally be maintained centrally and reused.

Example:

```text
Vaccine:
FMD Vaccine
```

can be used for:

```text
Animal A
Animal B
Animal C
Animal D
...
```

without creating duplicate vaccine definitions.

---

# 6. Field-by-Field Business Logic

# 6.1 `_id`

### Type

MongoDB `ObjectId`

### Required

Yes

### Purpose

Unique database identifier of the vaccine master record.

Example:

```text
ObjectId(...)
```

---

# 6.2 `vaccineCode`

### Type

String

### Required

Yes

### Example

```text
VAC-FMD-001
```

### Purpose

Human-readable unique code for identifying the vaccine.

### Business Uses

- Search
- Inventory reference
- Reports
- Vaccination records
- Administration
- API responses
- Auditing

---

# 7. Vaccine Code Generation

Recommended pattern:

```text
VAC-{SHORTCODE}-{SEQUENCE}
```

Example:

```text
VAC-FMD-001
VAC-BRUC-001
VAC-HS-001
```

The exact format can be adjusted, but it should be:

```text
Unique
Stable
Human-readable
```

The backend should generate/validate uniqueness rather than relying on the frontend.

---

# 8. `vaccineName`

### Type

String

### Required

Yes

### Example

```text
FMD Vaccine
```

### Purpose

Official/common name of the vaccine.

This is the primary display name in:

```text
Vaccine List
Vaccination Form
Vaccination History
Reports
Notifications
```

---

# 9. `shortCode`

### Type

String

### Required

Recommended

### Example

```text
FMD
```

### Purpose

Short identifier used for:

```text
UI
Search
Reports
Notifications
Labels
```

Example:

```text
FMD
```

is easier to display than:

```text
Foot and Mouth Disease Vaccine
```

---

# 10. `targetDiseaseIds[]`

### Type

Array of ObjectId references to `Disease`

### Required

Yes / at least one target disease recommended

### Example

```text
[
    Disease(Foot and Mouth Disease)
]
```

### Purpose

Defines which disease(s) the vaccine targets.

---

# 11. Why `targetDiseaseIds[]` Is an Array

One vaccine can potentially be associated with more than one disease.

Therefore:

```text
targetDiseaseIds[]
```

is more flexible than:

```text
targetDiseaseId
```

The relationship becomes:

```text
Vaccine
   |
   +---- Disease A
   +---- Disease B
```

If a particular vaccine targets only one disease:

```text
targetDiseaseIds = [Disease A]
```

---

# 12. Vaccine ↔ Disease Relationship

The finalized Disease model also contains:

```text
recommendedVaccineIds[]
```

Therefore, the domain has a bidirectional conceptual relationship:

```text
Disease
   ↓
recommendedVaccineIds[]

Vaccine
   ↓
targetDiseaseIds[]
```

This allows the system to answer:

### From Disease

```text
What vaccines are recommended for this disease?
```

### From Vaccine

```text
Which disease(s) does this vaccine target?
```

---

# 13. Important Reference Integrity

Every value inside:

```text
targetDiseaseIds[]
```

should refer to an existing Disease document.

The backend should reject:

```text
Invalid Disease ObjectId
```

or a reference to a non-existing disease.

---

# 14. `manufacturer`

### Type

String

### Required

Yes

### Example

```text
Company X
```

### Purpose

Stores the vaccine manufacturer.

### Business Uses

- Vaccine identification
- Inventory
- Procurement
- Reports
- Product differentiation

---

# 15. Manufacturer Changes

Manufacturer is part of the vaccine master definition.

If a new manufacturer supplies a clinically different/product-specific vaccine, the safer business approach may be to create a separate Vaccine master record rather than silently rewriting historical meaning.

Historical `Vaccination` records should not be changed simply because the current master data changes.

---

# 16. `category`

### Type

Enum

### Allowed Values

```text
Viral
Bacterial
Parasitic
Combined
```

These are the finalized categories in the DB model.

### Purpose

Classifies the vaccine according to its target disease category.

---

# 17. Category Uses

Can support:

```text
Filtering
Searching
Reports
Vaccination planning
Dashboard statistics
```

Example:

```text
Category = Viral
```

shows relevant vaccines.

---

# 18. `dosage`

### Type

Number

### Required

Yes

### Example

```text
2
```

with:

```text
dosageUnit = ml
```

means:

```text
2 ml
```

### Purpose

Stores the standard/reference dose defined for the vaccine.

---

# 19. Important Dosage Principle

`Vaccine.dosage` is the **master/reference dosage**.

It is not necessarily the final administered amount for every animal under every circumstance.

The actual administration event belongs to:

```text
Vaccination
```

and should follow the applicable veterinary/protocol rules.

Do not design the Vaccine master as if it stores individual animal administration history.

---

# 20. `dosageUnit`

### Type

Enum

### Allowed Values

```text
ml
mg
tablet
dose
```

### Purpose

Defines the unit used by the standard dosage.

Example:

```text
dosage = 2
dosageUnit = ml
```

---

# 21. Dosage Validation

Recommended:

```text
dosage > 0
```

The unit must be compatible with the vaccine's dosage definition.

The system should not allow:

```text
dosage = -2
```

---

# 22. `administrationRoute`

### Type

Enum

### Allowed Values

```text
IM
SC
IV
Oral
Nasal
```

These are the finalized values in the DB model.

### Meaning

Defines how the vaccine is administered.

Examples:

```text
IM = Intramuscular
SC = Subcutaneous
IV = Intravenous
Oral = Oral administration
Nasal = Nasal administration
```

---

# 23. Why Administration Route Is Important

The route is part of the vaccine administration protocol.

It affects:

```text
Vaccination instructions
Validation
UI guidance
Reports
Veterinary workflow
```

The system should not let a user arbitrarily change the route during routine vaccination entry if the selected Vaccine defines a fixed route.

---

# 24. `minimumAge`

### Type

Recommended representation: numeric value + standardized age unit at implementation level

### Current Model

The finalized model defines:

```text
minimumAge
```

with an example:

```text
4 Months
```

### Purpose

Defines the minimum age at which the vaccine is intended to be applicable according to the configured vaccine protocol.

---

# 25. Minimum Age Business Logic

When preparing vaccination for an animal:

```text
Animal age
       ↓
Compare with Vaccine.minimumAge
       ↓
Eligible / Warning
```

Example:

```text
Animal age = 2 months
Minimum age = 4 months
```

The system can show:

```text
Warning:
Animal has not reached the configured minimum age.
```

The exact veterinary eligibility decision should remain subject to the applicable vaccination protocol.

---

# 26. Important Design Note About `minimumAge`

The current DB model expresses the value conceptually as:

```text
4 Months
```

For implementation, avoid storing arbitrary text such as:

```text
"4 Months"
```

if the system needs mathematical comparison.

A structured representation is more reliable, for example:

```text
minimumAgeValue = 4
minimumAgeUnit = Month
```

or a normalized number of days/months.

This is an implementation recommendation; the finalized source only specifies the `minimumAge` field.

---

# 27. `boosterRequired`

### Type

Boolean

### Required

Yes

### Example

```text
true
```

### Purpose

Defines whether a booster dose is required according to the configured vaccine protocol.

---

# 28. Booster Logic

If:

```text
boosterRequired = true
```

then the vaccination workflow should use:

```text
boosterAfterDays
```

to determine the next expected booster date when applicable.

Conceptually:

```text
Vaccination Date
      +
boosterAfterDays
      ↓
Next Booster Due Date
```

---

# 29. `boosterAfterDays`

### Type

Number

### Required

Conditional

### Example

```text
30
```

### Purpose

Defines the number of days after the relevant vaccination when the booster is expected.

### Business Rule

If:

```text
boosterRequired = false
```

then:

```text
boosterAfterDays
```

should normally be:

```text
null / absent
```

or ignored.

Do not create a fake booster schedule when no booster is required.

---

# 30. `repeatIntervalDays`

### Type

Number

### Required

Conditional/according to vaccine protocol

### Example

```text
180
```

### Purpose

Defines the repeat interval for subsequent vaccination when applicable.

Conceptually:

```text
Previous applicable vaccination date
+
repeatIntervalDays
=
next repeat due date
```

---

# 31. Booster vs Repeat Interval

These are not necessarily the same thing.

### Booster

```text
A specifically required follow-up dose
```

### Repeat interval

```text
The longer-term interval for repeating vaccination
```

Example:

```text
First dose
    ↓ 30 days
Booster
    ↓ 180 days
Repeat vaccination
```

The exact schedule depends on the vaccine protocol.

---

# 32. Next Due Date Logic

The Vaccine master provides scheduling rules.

The actual:

```text
nextDueDate
```

belongs to the `Vaccination` record.

Therefore:

```text
Vaccine
   ↓
Scheduling rule

Vaccination
   ↓
Actual due date for this animal/event
```

This separation is important.

---

# 33. `storageTemperature`

### Type

String

### Required

Yes

### Example

```text
2-8°C
```

### Purpose

Stores the configured storage-temperature requirement for the vaccine.

This is useful for:

```text
Inventory
Storage guidance
Warnings
Procurement
Farm management
```

---

# 34. Storage Temperature and Inventory

When a vaccine exists in:

```text
Inventory
```

its Vaccine master can provide the storage requirement.

Conceptually:

```text
Vaccine
   ↓
storageTemperature = 2-8°C
   ↓
Inventory / Storage guidance
```

The current Inventory model supports vaccine inventory through:

```text
itemType = Vaccine
referenceId
```

and includes storage-related fields. 

---

# 35. Cold-Chain Warning

If the application later integrates environmental sensors, it could compare:

```text
Required:
2-8°C

Actual storage:
10°C
```

and generate:

```text
Cold-chain warning
```

This is a future integration opportunity.

The current Vaccine model itself does not measure actual temperature.

---

# 36. `description`

### Type

String

### Required

Optional

### Purpose

Stores additional vaccine information.

Example:

```text
Used for prevention of FMD according to farm vaccination protocol.
```

Possible information:

```text
General instructions
Product notes
Precautions
Usage information
```

The system should avoid turning this field into a substitute for formal veterinary guidance.

---

# 37. `isMandatory`

### Type

Boolean

### Required

Yes

### Example

```text
true
```

### Purpose

Indicates whether the vaccine is considered mandatory within Liventra's configured vaccination program.

---

# 38. Mandatory Vaccine Logic

If:

```text
isMandatory = true
```

the system can use it for:

```text
Vaccination schedule
Reminder
Compliance dashboard
Overdue alert
Farm health plan
```

This does not automatically mean every animal must receive it at every time.

Animal eligibility and the configured schedule still matter.

---

# 39. `isActive`

### Type

Boolean

### Required

Yes

### Example

```text
true
```

### Purpose

Controls whether the Vaccine master is currently available for normal use.

---

# 40. Active vs Inactive

If:

```text
isActive = true
```

the vaccine can normally be:

```text
Selected
Scheduled
Used for new vaccination records
```

If:

```text
isActive = false
```

it should normally be hidden/blocked from new routine selection.

But historical Vaccination records referencing it must remain valid.

---

# 41. Why Not Delete Old Vaccines?

Suppose:

```text
Vaccine V001
```

was used for 500 historical vaccinations.

If it is deleted:

```text
Vaccination.vaccineId → broken reference
```

Therefore:

> Deactivation is preferable to hard deletion for master records that already have historical references.

---

# 42. Historical Integrity

If a Vaccine definition changes:

```text
storageTemperature
manufacturer
description
```

historical Vaccination records should not be silently rewritten.

The master data represents the current definition.

Historical event data should remain traceable.

For especially important product details, the implementation may later consider snapshotting relevant fields in Vaccination, but that is not part of the current finalized model.

---

# 43. Vaccine Lifecycle

Recommended lifecycle:

```text
Create Vaccine
      ↓
Active
      ↓
Used in Vaccination
      ↓
May become Inactive
      ↓
Historical records remain
```

Avoid:

```text
Create
 ↓
Delete permanently
```

when historical references exist.

---

# 44. Vaccine Creation Workflow

```text
Admin / Authorized User
        ↓
Create Vaccine
        ↓
Enter vaccine identity
        ↓
Select target disease(s)
        ↓
Set manufacturer
        ↓
Set category
        ↓
Set dosage
        ↓
Set administration route
        ↓
Configure minimum age
        ↓
Configure booster/repeat rules
        ↓
Set storage requirement
        ↓
Set mandatory status
        ↓
Activate
```

---

# 45. Vaccine Usage Workflow

The actual operational workflow is:

```text
Disease / Vaccination Plan
        ↓
Find applicable Vaccine
        ↓
Check isActive
        ↓
Check animal eligibility
        ↓
Check minimum age
        ↓
Check previous vaccination history
        ↓
Determine due date
        ↓
Create/complete Vaccination
        ↓
Update inventory
        ↓
Create Inventory_Transaction
        ↓
Create reminder/notification
        ↓
AuditLog
```

---

# 46. Vaccine + Vaccination

The finalized Vaccination model contains:

```text
vaccineId
```

along with:

```text
vaccinationNo
farmId
animalId
vaccinationDate
nextDueDate
administeredBy
batchNumber
status
remarks
createdBy
updatedAt
```

Therefore:

```text
Vaccine
    ↓
vaccineId
    ↓
Vaccination
```

This is the central relationship.

---

# 47. Vaccine + Disease

The relationship is:

```text
Disease
    ↓
recommendedVaccineIds[]

Vaccine
    ↓
targetDiseaseIds[]
```

Example:

```text
Disease:
Foot and Mouth Disease

recommendedVaccineIds:
[FMD Vaccine]
```

and:

```text
FMD Vaccine

targetDiseaseIds:
[Foot and Mouth Disease]
```

---

# 48. Vaccine + Inventory

The finalized Inventory model supports:

```text
itemType = Vaccine
referenceId = Vaccine._id
```

Therefore:

```text
Vaccine
   ↓
Inventory item
   ↓
Stock
```

This enables:

```text
Current stock
Low stock
Expiry
Purchase
Consumption
```

for vaccines.

---

# 49. Vaccine + Inventory_Transaction

The finalized Inventory_Transaction model supports:

```text
transactionType = Vaccination Usage
referenceType = Vaccination
referenceId = Vaccination._id
```

Therefore, after a completed vaccination:

```text
Vaccination
      ↓
Vaccine inventory consumption
      ↓
Inventory_Transaction
```

This is important for stock traceability.

---

# 50. Example — Vaccination Consumption

Suppose:

```text
Vaccine stock = 100 doses
```

One vaccination uses:

```text
1 dose
```

After completion:

```text
Stock = 99 doses
```

and an Inventory_Transaction is created:

```text
transactionType = Vaccination Usage
referenceType = Vaccination
quantity = 1
```

---

# 51. Important Transaction Rule

Inventory should normally be consumed when the vaccination is actually:

```text
Completed
```

not merely when it is:

```text
Scheduled
```

Otherwise, a scheduled-but-cancelled vaccination could incorrectly reduce stock.

---

# 52. Cancelled Vaccination

If:

```text
status = Cancelled
```

then normally:

```text
No vaccination consumption
```

should be recorded unless the workflow explicitly documents wastage.

If a dose was opened/used and then vaccination was cancelled, that is a different inventory/wastage event and should be recorded accordingly.

---

# 53. Missed Vaccination

If:

```text
status = Missed
```

the vaccine has not necessarily been consumed.

Therefore:

```text
No automatic stock deduction
```

unless there is evidence that the vaccine was actually used.

A new due date may need to be scheduled according to the vaccination protocol.

---

# 54. Scheduled Vaccination

If:

```text
status = Scheduled
```

the system can create:

```text
Notification
```

for:

```text
Upcoming vaccination
```

but should not treat the vaccine as administered.

---

# 55. Mandatory Vaccine Reminder

If:

```text
isMandatory = true
```

and:

```text
nextDueDate is approaching
```

the system can generate:

```text
Vaccination Reminder
```

The SRS explicitly includes **Vaccine Reminder** under the Notification Module. 

---

# 56. Overdue Vaccine

If:

```text
today > nextDueDate
```

and vaccination is not completed:

```text
Overdue
```

can be detected by the scheduling/notification service.

The current Vaccine model itself does not need an `overdue` field.

---

# 57. Why Overdue Should Not Be Stored in Vaccine

Because:

```text
Vaccine
=
Master definition
```

while:

```text
Vaccination
=
Animal-specific schedule
```

Whether a vaccine is overdue depends on:

```text
Animal
+
Vaccination history
+
nextDueDate
+
current date
```

Therefore overdue status belongs to the vaccination scheduling layer.

---

# 58. Minimum Age + Animal Age

Example:

```text
Vaccine.minimumAge = 4 months
Animal age = 3 months
```

System:

```text
Not yet eligible
```

If:

```text
Animal age = 5 months
```

then:

```text
Age requirement satisfied
```

The exact final clinical decision should remain aligned with the configured veterinary vaccination program.

---

# 59. Booster Schedule Example

Configured:

```text
boosterRequired = true
boosterAfterDays = 30
```

Actual vaccination:

```text
2026-08-17
```

Then:

```text
Booster due ≈ 2026-09-16
```

The actual `nextDueDate` is stored in the Vaccination record.

---

# 60. Repeat Schedule Example

Configured:

```text
repeatIntervalDays = 180
```

Then the system can use the applicable completed vaccination date to determine a future repeat date.

Again:

```text
Vaccine
=
rule

Vaccination
=
actual schedule for the animal
```

---

# 61. Booster and Repeat Together

A vaccine may have:

```text
boosterRequired = true
boosterAfterDays = 30
repeatIntervalDays = 180
```

The application must define which interval applies to which vaccination stage.

Do not simply add both numbers together.

The schedule should follow the vaccine's configured protocol.

---

# 62. Vaccine Editing Rules

Master data can be edited by authorized users.

However:

```text
Used vaccine
```

should not be casually modified in ways that destroy historical meaning.

Examples requiring care:

```text
Changing manufacturer
Changing dosage
Changing route
Changing disease association
Changing schedule
```

Such changes should be:

```text
Audited
```

and, where they represent a genuinely different product/protocol, creating a new Vaccine record may be safer than overwriting the old master.

---

# 63. Vaccine Deactivation

Recommended:

```text
isActive = false
```

instead of deletion.

Example reasons:

```text
No longer used
Product discontinued
Farm changed vaccination program
Outdated master entry
```

Historical Vaccination records remain intact.

---

# 64. Authorization

Recommended access:

### Admin

Can:

```text
Create
Update
Deactivate
View
```

### Farm Owner / Manager

Can:

```text
View
Use active vaccines
```

depending on the final permission matrix.

### Worker

Can:

```text
View/select applicable vaccine
Record vaccination
```

only if their role allows health data entry.

The current SRS defines role-based access with Admin, Farm Owner, Farm Manager and Worker responsibilities. 

---

# 65. CRUD Business Rules

## Create

Requires:

```text
vaccineCode
vaccineName
target disease
manufacturer
category
dosage
dosageUnit
administrationRoute
minimumAge
booster configuration
storageTemperature
```

according to the project's validation policy.

---

## Read

Authorized users can view active and historical vaccine definitions.

---

## Update

Allowed to authorized users.

Important changes should be audited.

---

## Delete

Prefer:

```text
Deactivate
```

instead of hard delete if referenced by Vaccination.

---

# 66. Validation Rules

| Field | Validation |
|---|---|
| _id | MongoDB ObjectId |
| vaccineCode | Required, unique |
| vaccineName | Required |
| shortCode | Required/recommended, unique within policy |
| targetDiseaseIds | Valid Disease references |
| manufacturer | Required |
| category | Enum |
| dosage | Positive number |
| dosageUnit | Enum |
| administrationRoute | Enum |
| minimumAge | Valid age representation |
| boosterRequired | Boolean |
| boosterAfterDays | Positive when boosterRequired=true |
| repeatIntervalDays | Positive when applicable |
| storageTemperature | Required/valid configured string |
| description | Optional |
| isMandatory | Boolean |
| isActive | Boolean |
| createdAt | Automatic |
| updatedAt | Automatic |

---

# 67. Conditional Validation

Important:

### If:

```text
boosterRequired = true
```

then:

```text
boosterAfterDays
```

should normally be provided.

### If:

```text
boosterRequired = false
```

then:

```text
boosterAfterDays
```

should normally be null/unused.

### If:

```text
repeatIntervalDays
```

is configured, it should be positive.

---

# 68. Vaccine Code Uniqueness

Recommended:

```text
vaccineCode unique
```

Example:

```text
VAC-FMD-001
```

cannot be duplicated.

This makes references and reports safer.

---

# 69. Short Code Uniqueness

Recommended:

```text
shortCode unique
```

or at least unique within the relevant vaccine catalog.

Example:

```text
FMD
```

should not represent two unrelated vaccines.

---

# 70. Index Recommendations

Useful indexes:

```javascript
vaccineCode: 1
shortCode: 1
vaccineName: 1
category: 1
isActive: 1
isMandatory: 1
targetDiseaseIds: 1
```

Useful compound index:

```javascript
{
    isActive: 1,
    category: 1
}
```

for active-vaccine filtering.

---

# 71. Search

The UI should support searching by:

```text
vaccineCode
vaccineName
shortCode
manufacturer
```

Example:

```text
Search:
FMD
```

returns:

```text
FMD Vaccine
```

---

# 72. Vaccination Form Behavior

When a user selects:

```text
Vaccine = FMD Vaccine
```

the UI can automatically display:

```text
Target Disease
Dose
Route
Minimum Age
Booster
Storage Requirement
```

The user should not have to manually re-enter master information.

---

# 73. Why Vaccine Fields Should Not Be Duplicated in Vaccination

Example:

```text
Vaccination:
vaccineId = V001
```

The UI can populate:

```text
vaccineName
dose
route
```

from Vaccine.

This avoids unnecessary duplication.

However, if the project later requires legally/audit-critical historical snapshots, a controlled snapshot strategy can be considered.

---

# 74. Historical Master-Data Problem

Suppose:

```text
V001:
dosage = 2 ml
```

Later master data is changed to:

```text
dosage = 3 ml
```

An old Vaccination record references:

```text
V001
```

If the UI dynamically displays the current Vaccine dosage, it may misleadingly show:

```text
3 ml
```

for a historical event that occurred under the previous configuration.

Therefore, this is an important architectural consideration.

### Current model

The finalized model does not include a dosage snapshot in Vaccination.

### Recommendation

For the current project, preserve master-data changes through:

```text
AuditLog
```

and controlled master versioning/change history if needed.

If later the thesis requires exact historical reproducibility of administered dosage/route, consider adding snapshots to `Vaccination` in a future model revision rather than silently assuming current Vaccine data is historical truth.

---

# 75. Vaccine + AuditLog

Important auditable events:

```text
Vaccine Created
Vaccine Updated
Vaccine Activated
Vaccine Deactivated
Manufacturer Changed
Dosage Changed
Route Changed
Target Disease Changed
Booster Rule Changed
Repeat Interval Changed
Mandatory Status Changed
```

This is particularly important because Vaccine is master data used by many vaccination records.

---

# 76. Vaccine + Notification

The notification workflow can use:

```text
Vaccination.nextDueDate
```

and Vaccine properties such as:

```text
isMandatory
vaccineName
```

to produce:

```text
Upcoming Vaccine Reminder
Overdue Vaccine Alert
Mandatory Vaccine Warning
```

The SRS specifically identifies **Vaccine Reminder** as a notification requirement. 

---

# 77. Vaccine + Report

The Vaccine master can support reports such as:

```text
Active Vaccine List
Mandatory Vaccine List
Vaccine Usage Report
Vaccination Coverage
Vaccine Stock Report
```

The actual animal-level vaccination statistics should come from `Vaccination`, not Vaccine alone.

---

# 78. Vaccine + Farm Scope

The finalized Vaccine model does **not** contain `farmId`.

That means the current design treats Vaccine as shared/global master data rather than farm-specific data.

This is reasonable if:

```text
All farms use the same central vaccine catalog.
```

Then:

```text
Farm A
   ↓
Vaccination
   ↓
Vaccine V001

Farm B
   ↓
Vaccination
   ↓
Vaccine V001
```

If farms later need custom private vaccine catalogs, then a farm-scoping strategy would need to be introduced.

That is **not part of the current finalized model**.

---

# 79. Global Master vs Farm-Specific Usage

Important distinction:

```text
Vaccine
=
Global master definition

Vaccination
=
Farm + Animal-specific usage
```

This keeps the master data reusable.

---

# 80. Vaccine + Inventory Scope

Inventory is farm-specific:

```text
Inventory.farmId
```

while Vaccine is global.

Therefore:

```text
Global Vaccine V001
       ↓
Farm A Inventory item
Farm B Inventory item
Farm C Inventory item
```

Each farm maintains its own stock.

---

# 81. Example — Complete Vaccine

```javascript
{
    vaccineCode: "VAC-FMD-001",

    vaccineName: "FMD Vaccine",

    shortCode: "FMD",

    targetDiseaseIds: [
        "DISEASE-FMD-001"
    ],

    manufacturer: "Example Manufacturer",

    category: "Viral",

    dosage: 2,

    dosageUnit: "ml",

    administrationRoute: "IM",

    minimumAge: "4 Months",

    boosterRequired: true,

    boosterAfterDays: 30,

    repeatIntervalDays: 180,

    storageTemperature: "2-8°C",

    description:
        "FMD vaccine maintained according to the farm's configured vaccination protocol.",

    isMandatory: true,

    isActive: true
}
```

Values are illustrative and should not be interpreted as veterinary instructions.

---

# 82. Example — Actual Vaccination

The master:

```text
Vaccine:
VAC-FMD-001
```

is then used by:

```text
Vaccination:
vaccinationNo = VACC-000001
farmId = FARM-001
animalId = AN-000125
vaccineId = VAC-FMD-001
vaccinationDate = 2026-08-17
nextDueDate = ...
status = Completed
```

The exact due date should come from the configured vaccination protocol.

---

# 83. Example — Disease to Vaccine

```text
Disease:
Foot and Mouth Disease

recommendedVaccineIds:
[
    VAC-FMD-001
]
```

Then:

```text
Vaccine:
VAC-FMD-001

targetDiseaseIds:
[
    DISEASE-FMD-001
]
```

This makes the relationship discoverable from both directions.

---

# 84. Example — Inventory to Vaccination

Before:

```text
FMD Vaccine stock:
50 doses
```

Vaccination completed:

```text
1 dose
```

Then:

```text
Inventory:
49 doses
```

Transaction:

```text
transactionType = Vaccination Usage
referenceType = Vaccination
quantity = 1
```

This provides traceability from:

```text
Animal
→ Vaccination
→ Vaccine
→ Inventory
→ Stock movement
```

---

# 85. Example — Mandatory Vaccine Reminder

Configuration:

```text
isMandatory = true
```

Animal vaccination:

```text
nextDueDate = 2026-09-16
```

Before due date:

```text
Notification:
FMD vaccination due soon.
```

After due date without completion:

```text
Notification:
Mandatory vaccination overdue.
```

The exact notification timing is an application-level rule.

---

# 86. Error Scenario — Inactive Vaccine Selected

User tries to create a new vaccination with:

```text
isActive = false
```

Recommended behavior:

```text
Reject normal new selection
```

unless an authorized historical/correction workflow explicitly permits it.

---

# 87. Error Scenario — Invalid Disease Reference

User submits:

```text
targetDiseaseIds = [nonexistentId]
```

Backend:

```text
Reject
```

because Vaccine → Disease reference integrity is broken.

---

# 88. Error Scenario — Booster Enabled but No Interval

Input:

```text
boosterRequired = true
boosterAfterDays = null
```

Recommended:

```text
Validation error
```

unless the business protocol intentionally defines the booster schedule elsewhere.

---

# 89. Error Scenario — Negative Dosage

Input:

```text
dosage = -2
```

Reject.

---

# 90. Error Scenario — Vaccine Deactivated but Historical Vaccination Exists

Do:

```text
isActive = false
```

Do not:

```text
delete Vaccine
```

Historical Vaccination records remain intact.

---

# 91. Error Scenario — Vaccine Master Changed After Historical Use

If:

```text
dosage changes
```

do not rewrite old Vaccination events automatically.

Use:

```text
AuditLog
+
controlled master-data change
```

and consider a new master/version strategy if the change represents a materially different vaccine definition.

---

# 92. Performance Considerations

Vaccine is generally a relatively small master collection compared with:

```text
SensorData
MilkRecord
FeedRecord
AuditLog
```

Therefore:

```text
Indexes
Caching
Active-only queries
```

can make vaccine selection very fast.

A common UI query can be:

```text
Find active vaccines
sorted by vaccineName
```

---

# 93. Security / Permissions

Only authorized roles should modify master Vaccine data.

Workers should generally not be allowed to:

```text
Change dosage
Change administration route
Change booster interval
Deactivate vaccines
```

unless explicitly granted that permission.

Otherwise, a simple data-entry user could alter the vaccination program for the entire system.

---

# 94. Business Logic Summary

The Vaccine model is responsible for:

```text
Identity
Disease association
Manufacturer
Classification
Standard dosage
Administration route
Eligibility rule
Booster rule
Repeat rule
Storage requirement
Mandatory status
Active/inactive lifecycle
```

It is NOT responsible for:

```text
Which animal received it
When an animal received it
Whether the animal actually received it
Who administered it
Batch used for that event
Actual next due date
Vaccination status
```

Those belong to:

```text
Vaccination
```

---

# 95. Final Architecture

```text
                 ┌──────────────┐
                 │    Disease   │
                 └──────┬───────┘
                        │
             recommendedVaccineIds[]
                        │
                        ▼
                 ┌──────────────┐
                 │    Vaccine   │
                 │  MASTER DATA │
                 └──────┬───────┘
                        │
                    vaccineId
                        │
                        ▼
                 ┌──────────────┐
                 │ Vaccination  │
                 │ EVENT/HISTORY│
                 └──────┬───────┘
                        │
          ┌─────────────┼──────────────┐
          ▼             ▼              ▼
       Animal        Inventory      Notification
                        │
                        ▼
              Inventory_Transaction
```

---

# 96. Defense Questions

### Q: Why do you need a separate Vaccine model?

**Answer:**

> "Vaccine is master data describing the vaccine itself, while Vaccination records the actual administration of that vaccine to a specific animal. Separating them avoids duplication and allows the same vaccine definition to be reused across many animals and farms."

---

### Q: What does vaccineId in Vaccination reference?

**Answer:**

> "It references the `_id` of the Vaccine master collection."

---

### Q: Why is targetDiseaseIds an array?

**Answer:**

> "Because a vaccine can potentially be associated with multiple target diseases. The array also gives the schema flexibility for combined vaccines."

---

### Q: Why does Disease also have recommendedVaccineIds?

**Answer:**

> "It allows the system to navigate in both directions: from a disease to its recommended vaccines and from a vaccine to its target diseases."

---

### Q: Why is nextDueDate not stored in Vaccine?

**Answer:**

> "Because due date is animal-specific and event-specific. Vaccine stores the scheduling rules, while Vaccination stores the actual next due date for that animal."

---

### Q: Why is isActive necessary?

**Answer:**

> "Because a vaccine may no longer be used for new vaccinations but historical Vaccination records may still reference it. Deactivation preserves historical integrity better than deletion."

---

### Q: What happens when a vaccine is mandatory?

**Answer:**

> "The system can use isMandatory to prioritize scheduling, reminders and compliance monitoring. It does not by itself mean every animal is automatically eligible at every time."

---

### Q: What happens when boosterRequired is true?

**Answer:**

> "The system uses boosterAfterDays as part of the scheduling rule to determine the applicable booster due date for a Vaccination event."

---

### Q: When should inventory be reduced?

**Answer:**

> "Normally when a vaccination is actually completed, not merely scheduled. A scheduled or missed vaccination should not automatically consume stock."

---

### Q: Why should old vaccines not be deleted?

**Answer:**

> "Because historical Vaccination records reference the Vaccine. Deactivating the master preserves those historical references."

---

### Q: Why is Vaccine global instead of farm-specific?

**Answer:**

> "The finalized model does not contain farmId, so the current architecture treats Vaccine as shared master data. Farm-specific usage and stock are represented through Vaccination and Inventory."

---

# 97. Final Business Rules Checklist

```text
[✓] Vaccine is master/reference data
[✓] Vaccination is the actual animal-level event
[✓] Vaccination.vaccineId references Vaccine._id
[✓] vaccineCode should be unique
[✓] vaccineName is required
[✓] targetDiseaseIds references Disease
[✓] targetDiseaseIds is an array
[✓] category is controlled by enum
[✓] dosage must be valid
[✓] dosageUnit is controlled by enum
[✓] administrationRoute is controlled by enum
[✓] minimumAge is used for eligibility checking
[✓] boosterRequired controls booster logic
[✓] boosterAfterDays is conditional
[✓] repeatIntervalDays represents repeat scheduling
[✓] nextDueDate belongs to Vaccination, not Vaccine
[✓] storageTemperature describes required storage condition
[✓] isMandatory supports vaccination-program compliance
[✓] isActive controls current availability
[✓] Historical references must remain valid
[✓] Deactivation is preferred over deletion
[✓] Vaccine can connect to Inventory
[✓] Completed Vaccination can create inventory consumption
[✓] Disease can reference recommended vaccines
[✓] Vaccine can reference target diseases
[✓] Vaccine changes should be auditable
[✓] Master-data changes should not silently rewrite historical events
[✓] Only authorized roles should modify vaccine master data
```

---

# 98. Final Status

**Model:** `Vaccine`  
**Status:** ✅ Finalized  
**Business Logic:** ✅ Documented  
**Master Data:** ✅  
**Disease Relationship:** ✅  
**Vaccination Relationship:** ✅  
**Inventory Integration:** ✅  
**Reminder/Scheduling Support:** ✅  
**Historical Integrity:** ✅  
**Auditability:** ✅  

**Next Model:** `Vaccination`
