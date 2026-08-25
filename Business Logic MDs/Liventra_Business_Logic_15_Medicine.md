# Liventra — Business Logic Documentation
## Model 15: Medicine

**Document Version:** Business Logic v1.0  
**Project:** Liventra — Smart Livestock Farm Management System  
**Database:** MongoDB  
**Architecture:** MERN  
**Module:** Health & Treatment

---

# 1. Source / Finalized Model

The current finalized DB model defines `Medicine` as:

```text
Medicine Model:
_id,
medicineCode,
medicineName,
genericName,
brandName,
manufacturer,
category:
    Antibiotic
    Anthelmintic
    Antipyretic
    Vitamin
    Mineral
    Pain Killer
    Hormone
    Supplement
    Other,
dosageForm:
    Injection
    Tablet
    Bolus
    Powder
    Liquid
    Spray
    Ointment,
strength,
administrationRoute:
    IM
    IV
    SC
    Oral
    Topical
    Intranasal,
description,
storageCondition,
withdrawalPeriodDays,
prescriptionRequired,
isActive,
createdAt,
updatedAt
```

This is the Medicine structure in the current finalized DB-model source. fileciteturn14file0

---

# 2. Purpose

`Medicine` is the **master/reference model for medicines that can be used in animal treatment**.

It answers:

> "What medicine/product is this, what is its generic/brand identity, what category and dosage form does it have, how is it administered, how should it be stored, and what withdrawal period applies?"

It does **not** record:

- which animal received it,
- when it was administered,
- how many times it was given,
- who administered it,
- or which treatment event used it.

Those belong to `Treatment`.

Therefore:

```text
Medicine
   ↓
MASTER / REFERENCE DATA

Treatment
   ↓
ANIMAL-SPECIFIC TREATMENT EVENT
```

---

# 3. Medicine vs Treatment

This distinction is fundamental.

## Medicine

```text
What medicine is available/recognized in Liventra?
```

Example:

```text
Medicine:
Example Antibiotic

Generic:
Amoxicillin

Form:
Injection

Route:
IM

Strength:
100 mg/ml
```

## Treatment

```text
How was this medicine used for Animal A?
```

Example:

```text
Animal:
AN-000125

Medicine:
M001

Dosage:
5 ml

Frequency:
Once daily

Duration:
5 days
```

Therefore:

```text
Medicine
   ↓
Definition

Treatment
   ↓
Actual use
```

---

# 4. Medicine's Role in the Health Architecture

The treatment flow becomes:

```text
Disease
   ↓
Treatment
   ↓
Medicines[]
   ↓
Medicine
```

And operationally:

```text
Treatment
    ↓
Medicine selected
    ↓
Inventory availability
    ↓
Medicine consumption
    ↓
Inventory_Transaction
    ↓
Withdrawal-period tracking
    ↓
Animal health timeline
    ↓
AuditLog
```

The finalized Treatment model contains a nested `medicines[]` array with `medicineId`, dosage, dosage unit, frequency, duration, administration route and instructions. fileciteturn14file0

---

# 5. Why Medicine Needs Its Own Collection

Without a Medicine master collection, Treatment records would repeatedly store medicine names as free text.

Bad:

```text
Treatment 1:
medicineName = Amoxicillin

Treatment 2:
medicineName = amoxycillin

Treatment 3:
medicineName = Amox 100
```

This causes:

```text
Duplicate data
Inconsistent naming
Poor reporting
Difficult inventory integration
```

With a Medicine master:

```text
Medicine
_id = MED-001
medicineName = Example Antibiotic
```

Treatment stores:

```text
medicineId = MED-001
```

---

# 6. Field-by-Field Business Logic

# 6.1 `_id`

### Type

MongoDB `ObjectId`

### Required

Yes

### Purpose

Unique database identifier for the medicine master record.

Example:

```text
ObjectId(...)
```

This is different from:

```text
medicineCode
```

because `_id` is the database identifier while `medicineCode` is the human-readable business identifier.

---

# 6.2 `medicineCode`

### Type

String

### Required

Yes

### Example

```text
MED-ANT-001
```

### Purpose

Human-readable unique code for identifying the medicine.

Used in:

```text
Medicine search
Treatment
Inventory
Reports
Audit
Administration
```

---

# 7. Medicine Code Generation

Recommended pattern:

```text
MED-{CATEGORY/PREFIX}-{SEQUENCE}
```

Examples:

```text
MED-ANT-001
MED-VIT-001
MED-ANH-001
```

The exact prefix can be decided during implementation.

The backend should generate/validate uniqueness.

---

# 8. `medicineName`

### Type

String

### Required

Yes

### Example

```text
Example Antibiotic
```

### Purpose

Primary medicine/product name displayed in the UI.

Used in:

```text
Medicine List
Treatment Form
Inventory
Reports
Animal Health History
```

---

# 9. `genericName`

### Type

String

### Required

Recommended

### Example

```text
Amoxicillin
```

### Purpose

Stores the generic/active-ingredient name.

This is useful because multiple brands may contain the same active ingredient.

Example:

```text
Brand A
Generic: Amoxicillin

Brand B
Generic: Amoxicillin
```

This supports better searching and reporting.

---

# 10. Generic Name vs Medicine Name

Do not confuse:

```text
medicineName
```

with:

```text
genericName
```

Example:

```text
medicineName:
Product X Injection

genericName:
Amoxicillin
```

The medicine name identifies the product/catalog entry.

The generic name identifies the active medicinal substance.

---

# 11. `brandName`

### Type

String

### Required

Optional

### Example

```text
Brand X
```

### Purpose

Stores the commercial brand/product name where applicable.

This is useful when:

```text
Different manufacturers
Different branded products
Procurement
Inventory
```

are involved.

---

# 12. Why Generic + Brand Are Separate

Example:

```text
Generic:
Amoxicillin

Brand:
Brand X
```

Another product may be:

```text
Generic:
Amoxicillin

Brand:
Brand Y
```

Liventra can therefore distinguish products while still identifying their generic ingredient.

---

# 13. `manufacturer`

### Type

String

### Required

Yes

### Example

```text
Example Pharmaceutical Company
```

### Purpose

Identifies the manufacturer/company associated with the medicine.

Uses:

```text
Procurement
Inventory
Product identification
Reports
Traceability
```

---

# 14. Manufacturer Changes

If a materially different product is introduced by another manufacturer, it is generally safer to create a separate Medicine master entry rather than silently changing the manufacturer of an already-used product.

Why?

Because historical Treatment/Inventory records may rely on the old product identity.

Master-data changes should be audited.

---

# 15. `category`

### Type

Enum

### Allowed Values

```text
Antibiotic
Anthelmintic
Antipyretic
Vitamin
Mineral
Pain Killer
Hormone
Supplement
Other
```

These are the finalized categories in the current DB model. fileciteturn14file0

### Purpose

Classifies the medicine.

---

# 16. Category Uses

Medicine category supports:

```text
Search
Filtering
Inventory reports
Treatment analytics
Medicine statistics
```

Example:

```text
Show all Antibiotics.
```

---

# 17. Important Category Principle

The category is a classification of the medicine master.

It does not determine:

```text
whether the medicine is appropriate for a particular animal
```

That decision belongs to the treatment/veterinary workflow.

---

# 18. `dosageForm`

### Type

Enum

### Allowed Values

```text
Injection
Tablet
Bolus
Powder
Liquid
Spray
Ointment
```

### Purpose

Defines the physical/pharmaceutical form in which the medicine is supplied/administered.

---

# 19. Why Dosage Form Matters

The dosage form affects:

```text
Administration
UI
Inventory unit
Treatment instructions
Storage
```

Example:

```text
Injection
```

may be administered through:

```text
IM / IV / SC
```

while:

```text
Ointment
```

uses:

```text
Topical
```

The system should validate route/form compatibility where appropriate.

---

# 20. `strength`

### Type

String

### Required

Recommended

### Example

```text
100 mg/ml
```

### Purpose

Stores the concentration/strength of the medicine product.

This is important because:

```text
5 ml of 100 mg/ml
```

is not equivalent to:

```text
5 ml of 20 mg/ml
```

Therefore strength must be associated with the medicine product.

---

# 21. Why Strength Is String

The current finalized model defines:

```text
strength
```

without a separate numeric strength value/unit pair.

Therefore, examples such as:

```text
100 mg/ml
500 mg
10%
```

can be represented.

If the system later needs mathematical dose calculations from strength, a structured strength representation may be considered in a future schema revision.

---

# 22. `administrationRoute`

### Type

Enum

### Allowed Values

```text
IM
IV
SC
Oral
Topical
Intranasal
```

These are the finalized values. fileciteturn14file0

### Meaning

Defines the route through which the medicine is administered.

Examples:

```text
IM = Intramuscular
IV = Intravenous
SC = Subcutaneous
Oral = Oral
Topical = Topical application
Intranasal = Nasal administration
```

---

# 23. Route Validation

The route should be compatible with:

```text
dosageForm
```

Example:

```text
dosageForm = Ointment
administrationRoute = IV
```

is logically invalid and should be rejected.

Similarly:

```text
dosageForm = Injection
administrationRoute = Topical
```

would normally require validation.

The exact compatibility matrix should be defined by the project's approved medicine catalog.

---

# 24. Treatment Route vs Medicine Route

The Medicine master stores:

```text
standard/default administrationRoute
```

The finalized Treatment model also stores:

```text
medicines[].administrationRoute
```

This is important.

Medicine says:

```text
What is the configured/default route for this product?
```

Treatment says:

```text
What route was selected/recorded for this particular treatment?
```

The Treatment route should normally match the medicine's allowed route unless an authorized clinical workflow explicitly permits another configured route.

---

# 25. `description`

### Type

String

### Required

Optional

### Purpose

Stores general information about the medicine.

Example:

```text
General product description and usage information.
```

It can be shown on:

```text
Medicine details
Treatment selection
Inventory details
```

---

# 26. `storageCondition`

### Type

String

### Required

Recommended

### Example

```text
Store in a cool, dry place
```

or:

```text
2-8°C
```

### Purpose

Stores the required storage condition for the medicine.

This is useful for:

```text
Inventory
Storage management
Warnings
Procurement
```

---

# 27. Storage and Inventory

The Medicine master provides the expected storage condition.

Farm-specific inventory stores the actual stock.

Therefore:

```text
Medicine
   ↓
storageCondition
   ↓
Inventory
   ↓
Farm stock
```

If future IoT sensors are integrated, actual storage conditions can be compared with the medicine's configured requirement.

---

# 28. `withdrawalPeriodDays`

### Type

Number

### Required

Recommended / Conditional

### Example

```text
7
```

### Purpose

Stores the configured withdrawal period associated with the medicine.

This is especially important for food-producing livestock.

---

# 29. What Is Withdrawal Period?

Withdrawal period is the period after a medicine is administered during which relevant animal products may require restriction according to the applicable veterinary/product protocol.

For example:

```text
Treatment completed
       ↓
Withdrawal period
       ↓
Milk/meat restriction
       ↓
Eligible according to applicable protocol
```

The actual period and applicable products must come from the approved veterinary/product information.

---

# 30. Why Withdrawal Period Is Important in Liventra

Liventra manages:

```text
Milk production
Animal health
Treatment
```

Therefore, medicine use can affect production decisions.

Example:

```text
Animal treated
+
Withdrawal period active
```

The system can warn:

```text
Milk/product may be under withdrawal restriction.
```

This is an important smart-farm business rule.

---

# 31. Withdrawal Period Is Not a Treatment Date

Medicine stores:

```text
withdrawalPeriodDays
```

as a master rule.

Treatment stores:

```text
actual treatment dates
```

Therefore:

```text
Medicine
   ↓
Rule: 7 days

Treatment
   ↓
Actual last administration date
   ↓
Withdrawal end date
```

The actual animal-specific withdrawal end date should be derived from the treatment event.

---

# 32. Withdrawal Calculation

Conceptually:

```text
Last applicable treatment date
+
withdrawalPeriodDays
=
withdrawal end date
```

Example:

```text
Last treatment:
2026-08-17

Withdrawal:
7 days

Withdrawal end:
approximately 2026-08-24
```

Exact date semantics should be defined consistently by the application.

---

# 33. Important Milk Integration

Because Liventra contains `MilkRecord`, the system can use withdrawal information to warn before recording/using milk for eligible production workflows.

Conceptually:

```text
Treatment
   ↓
Medicine
   ↓
Withdrawal period
   ↓
Animal under withdrawal
   ↓
Milk/product warning
```

The current Medicine model itself does not store an animal-specific withdrawal state.

---

# 34. Important Meat Integration

If the animal is sold/slaughtered while under a relevant withdrawal period, the system can generate an alert.

Conceptually:

```text
Animal
   ↓
Treatment
   ↓
Withdrawal active
   ↓
Sale/market readiness warning
```

The exact business rules should be aligned with applicable veterinary/regulatory requirements.

---

# 35. `prescriptionRequired`

### Type

Boolean

### Required

Yes

### Example

```text
true
```

### Purpose

Indicates whether the medicine requires authorized/prescription-controlled handling in Liventra's workflow.

---

# 36. Prescription Logic

If:

```text
prescriptionRequired = true
```

the system can require:

```text
Authorized user
Veterinary approval
Appropriate treatment documentation
```

before allowing treatment completion/use according to the project's permission rules.

This flag is a workflow control, not a substitute for actual legal/regulatory classification.

---

# 37. `isActive`

### Type

Boolean

### Required

Yes

### Example

```text
true
```

### Purpose

Controls whether the medicine is currently available for normal new treatment selection.

---

# 38. Active vs Inactive

If:

```text
isActive = true
```

medicine can normally be:

```text
Selected
Added to Treatment
Displayed in active medicine list
```

If:

```text
isActive = false
```

it should normally:

```text
Not appear in normal new-treatment selection
```

while historical Treatment and Inventory references remain valid.

---

# 39. Why Medicine Should Not Be Hard-Deleted

Suppose:

```text
Medicine M001
```

has been used in:

```text
100 Treatment records
```

and exists in:

```text
Inventory
```

Deleting it could break historical references.

Therefore:

```text
isActive = false
```

is preferable to hard deletion.

---

# 40. `createdAt`

### Type

Date

### Required

Automatic

### Purpose

Stores when the Medicine master record was created.

---

# 41. `updatedAt`

### Type

Date

### Required

Automatic

### Purpose

Stores when the Medicine master record was last modified.

---

# 42. Medicine Lifecycle

Recommended:

```text
Create Medicine
      ↓
Active
      ↓
Used in Treatment / Inventory
      ↓
May be updated
      ↓
May become Inactive
      ↓
Historical references remain
```

---

# 43. Medicine Creation Workflow

```text
Admin / Authorized User
        ↓
Create Medicine
        ↓
Medicine Code
        ↓
Name
        ↓
Generic / Brand
        ↓
Manufacturer
        ↓
Category
        ↓
Dosage Form
        ↓
Strength
        ↓
Administration Route
        ↓
Storage Condition
        ↓
Withdrawal Period
        ↓
Prescription Requirement
        ↓
Activate
```

---

# 44. Medicine + Treatment Relationship

The finalized Treatment model contains:

```text
medicines:[
    {
        medicineId,
        dosage,
        dosageUnit,
        frequency,
        duration,
        administrationRoute,
        instructions
    }
]
```

Therefore:

```text
Treatment
   ↓
medicines[].medicineId
   ↓
Medicine._id
```

This is a one-to-many relationship from Treatment to medicine references.

---

# 45. Why Treatment Stores `medicines[]`

One treatment event can use multiple medicines.

Example:

```text
Treatment T001

medicines:
[
    Antibiotic,
    Antipyretic,
    Supplement
]
```

This is cleaner than creating a separate Treatment document for every medicine.

---

# 46. Medicine Master vs Treatment Dose

This is one of the most important design decisions.

Medicine stores:

```text
strength
```

Treatment stores:

```text
dosage
dosageUnit
frequency
duration
```

Why?

Because:

```text
Medicine
=
What product is it?

Treatment
=
How was it prescribed/used for this animal?
```

Therefore, the animal-specific dosage must not be stored only in Medicine.

---

# 47. Example

Medicine:

```text
strength = 100 mg/ml
```

Treatment:

```text
dosage = 5
dosageUnit = ml
frequency = Once daily
duration = 5 days
```

This means the treatment event contains the actual configured administration schedule for that animal.

---

# 48. Treatment Administration Route

Treatment also stores:

```text
administrationRoute
```

Example:

```text
Medicine:
Injection
Default route:
IM

Treatment:
administrationRoute = IM
```

The backend should validate that the treatment route is allowed for the selected medicine.

---

# 49. Medicine + Disease

Medicine does not directly need:

```text
diseaseIds[]
```

in the current finalized model.

Why?

Because medicine selection is contextual to a Treatment.

Relationship:

```text
Disease
   ↓
Treatment
   ↓
Medicine
```

This avoids storing potentially misleading global claims such as:

```text
Medicine M001 treats Disease X
```

unless the project later explicitly introduces a validated medicine-disease knowledge base.

---

# 50. Medicine + Treatment Example

```text
Disease:
D001

Treatment:
T001

medicines:
[
    {
        medicineId: M001,
        dosage: 5,
        dosageUnit: "ml",
        frequency: "Once daily",
        duration: "5 days",
        administrationRoute: "IM",
        instructions: "Use according to treatment plan."
    }
]
```

This means:

```text
Animal-specific treatment
using Medicine M001.
```

---

# 51. Medicine + Inventory

Medicine is master data.

Inventory is farm-specific stock.

Conceptually:

```text
Medicine
   ↓
Inventory
   ↓
Farm stock
```

The finalized Inventory model uses:

```text
itemType
referenceId
```

to represent items such as:

```text
Medicine
```

and other inventory categories. fileciteturn14file1

---

# 52. Medicine + Inventory Transaction

When medicine is consumed through treatment:

```text
Treatment
   ↓
Medicine
   ↓
Inventory consumption
   ↓
Inventory_Transaction
```

The finalized Inventory_Transaction model supports transaction references and types including treatment usage. fileciteturn14file1

---

# 53. Important Stock Rule

Do not reduce medicine inventory simply because:

```text
Treatment is scheduled/drafted
```

Normally, consumption should happen when the medicine is actually administered/consumed according to the treatment workflow.

This avoids:

```text
Cancelled treatment
→ Incorrect stock reduction
```

---

# 54. Treatment + Inventory Atomicity

When treatment administration causes medicine consumption:

```text
BEGIN TRANSACTION

Validate Treatment
Validate Medicine
Validate Stock
Update Treatment
Reduce Inventory
Create Inventory_Transaction
Create AuditLog

COMMIT
```

If a critical operation fails:

```text
ROLLBACK
```

This prevents treatment and stock from becoming inconsistent.

---

# 55. Insufficient Medicine Stock

Example:

```text
Required:
5 ml

Available:
0 ml
```

The system should normally:

```text
Prevent completion/consumption
```

unless an authorized override/manual adjustment process exists.

---

# 56. Medicine Expiry

The finalized Inventory model contains expiry-related information.

Therefore, before medicine consumption, the system should ideally ensure the selected stock is not:

```text
Expired
```

Medicine master:

```text
What product is it?
```

Inventory:

```text
Which stock is currently available and valid?
```

These responsibilities remain separate.

---

# 57. Prescription Workflow

If:

```text
prescriptionRequired = true
```

then Treatment creation/completion may require:

```text
Authorized prescriber / veterinarian
or
approved treatment record
```

depending on the final role system.

Since the current architecture does not require a separate Doctor collection, `treatedBy` should use the project's authorized user/farm-member identity rather than inventing a Doctor reference.

---

# 58. No Separate Doctor Model

For the current Liventra architecture:

```text
Medicine
   ↓
Treatment
   ↓
treatedBy
```

The system should not assume:

```text
doctorId
```

unless a Doctor model is actually introduced later.

This is consistent with the current finalized Treatment structure, which uses `treatedBy` rather than the old `doctorId` design. fileciteturn14file0

---

# 59. Medicine and Farm Scope

The current Medicine model does not contain:

```text
farmId
```

Therefore, Medicine is treated as shared/global master data.

Farm-specific stock is represented through:

```text
Inventory.farmId
```

Conceptually:

```text
Global Medicine M001
       ↓
Farm A Inventory
Farm B Inventory
Farm C Inventory
```

---

# 60. Global Medicine vs Farm Stock

This separation is useful:

```text
Medicine
=
Product definition

Inventory
=
Farm-specific quantity
```

Example:

```text
Medicine M001:
Amoxicillin Injection

Farm A:
20 units

Farm B:
5 units

Farm C:
0 units
```

One master medicine can therefore be reused across farms.

---

# 61. Medicine + Withdrawal + Milk

Example workflow:

```text
Treatment completed
        ↓
Medicine used
        ↓
withdrawalPeriodDays = 7
        ↓
Calculate withdrawal end
        ↓
Animal flagged for product restriction
        ↓
Milk-related warning
```

The actual withdrawal calculation should be based on the last applicable administration date.

---

# 62. Withdrawal Status Should Not Be Stored in Medicine

Medicine only stores:

```text
withdrawalPeriodDays
```

It should not store:

```text
Animal A is under withdrawal
```

because that is animal-specific.

The status is derived from:

```text
Treatment
+
Medicine
+
treatment date
```

---

# 63. Medicine + Animal Sale

Before marking an animal as sold/market-ready, Liventra can check whether a relevant withdrawal period remains active.

Conceptually:

```text
Animal
   ↓
Recent Treatment
   ↓
Medicine withdrawal period
   ↓
Still active?
   ↓
Warning
```

The exact enforcement policy should be configurable and aligned with applicable regulations.

---

# 64. Medicine + Health Timeline

Animal health timeline can display:

```text
Disease
Treatment
Medicine
Vaccination
Weight
Milk
Breeding
Pregnancy
```

Example:

```text
Aug 10 → Disease/Treatment
Aug 10 → Medicine A
Aug 17 → Vaccination
Aug 20 → Weight
```

This gives the farmer a chronological health history.

---

# 65. Medicine + Reports

Possible reports:

```text
Medicine Master List
Medicine Usage Report
Treatment Medicine Usage
Medicine Stock Report
Medicine Cost Report
Withdrawal Alert Report
Prescription Medicine Report
```

The Report module can be used to generate health/treatment-related reports.

---

# 66. Medicine Analytics

Possible metrics:

```text
Most used medicines
Medicine usage by category
Medicine consumption by farm
Treatment cost by medicine
Antibiotic usage trend
Withdrawal-period animals
```

These are derived from:

```text
Medicine
+
Treatment
+
Inventory
```

rather than from Medicine master records alone.

---

# 67. Important Antibiotic Tracking

Because the Medicine category includes:

```text
Antibiotic
```

the system can later produce:

```text
Antibiotic usage report
```

by querying Treatment records whose medicine references point to medicines with:

```text
category = Antibiotic
```

This is a useful management/analytics feature.

It should not be treated as a clinical prescription engine.

---

# 68. Medicine Search

Search should support:

```text
medicineCode
medicineName
genericName
brandName
manufacturer
category
```

Example:

```text
Search:
Amoxicillin
```

can return products whose:

```text
genericName = Amoxicillin
```

---

# 69. Medicine Detail Page

A medicine detail page can display:

```text
Medicine Code
Medicine Name
Generic Name
Brand Name
Manufacturer
Category
Dosage Form
Strength
Administration Route
Storage Condition
Withdrawal Period
Prescription Required
Active Status
```

---

# 70. Medicine Editing Rules

Authorized users may update master information.

However, be careful with:

```text
genericName
brandName
manufacturer
strength
dosageForm
administrationRoute
withdrawalPeriodDays
```

because changing these fields can affect how the product is interpreted.

Important changes should be audited.

If a change represents a genuinely different product, creating a new Medicine master entry is safer than overwriting the old product identity.

---

# 71. Medicine Deactivation

If a product is no longer used:

```text
isActive = false
```

rather than:

```text
DELETE
```

Historical Treatment records remain valid.

Inventory history also remains traceable.

---

# 72. CRUD Business Rules

## Create

Requires the core medicine identity:

```text
medicineCode
medicineName
manufacturer
category
dosageForm
administrationRoute
```

with other fields according to the project's validation policy.

---

## Read

Authorized users can view:

```text
Active medicines
Historical/inactive medicines
```

depending on UI context.

---

## Update

Authorized users can update master data.

Important changes should create AuditLog records.

---

## Delete

Prefer:

```text
isActive = false
```

if referenced by Treatment or Inventory.

---

# 73. Validation Rules

| Field | Validation |
|---|---|
| `_id` | MongoDB ObjectId |
| `medicineCode` | Required, unique |
| `medicineName` | Required |
| `genericName` | Recommended string |
| `brandName` | Optional string |
| `manufacturer` | Required |
| `category` | Enum |
| `dosageForm` | Enum |
| `strength` | String, valid format according to catalog |
| `administrationRoute` | Enum |
| `description` | Optional |
| `storageCondition` | String |
| `withdrawalPeriodDays` | Non-negative number |
| `prescriptionRequired` | Boolean |
| `isActive` | Boolean |
| `createdAt` | Automatic |
| `updatedAt` | Automatic |

---

# 74. Withdrawal Validation

Recommended:

```text
withdrawalPeriodDays >= 0
```

A negative withdrawal period should never be accepted.

For medicines with no applicable withdrawal requirement:

```text
0
```

may represent no configured waiting period, if that matches the project's data convention.

---

# 75. Route/Form Compatibility

Recommended application-level validation:

```text
Injection
→ IM / IV / SC

Tablet
→ Oral

Bolus
→ Oral

Ointment
→ Topical

Spray
→ context-dependent
```

The final compatibility matrix should be based on the actual medicine catalog.

Do not hard-code a medically universal matrix without validating the project requirements.

---

# 76. Strength Validation

Avoid accepting meaningless values such as:

```text
strength = "abc"
```

If strength is stored as a string, the system can still validate a reasonable format where practical.

Examples:

```text
100 mg/ml
500 mg
10%
```

For advanced mathematical dose calculations, a structured strength representation would be more suitable, but that is outside the current finalized model.

---

# 77. Duplicate Medicine Prevention

Potential duplicates:

```text
Medicine A:
genericName = Amoxicillin
brandName = Brand X
manufacturer = Company A
```

and:

```text
Medicine B:
genericName = Amoxicillin
brandName = Brand X
manufacturer = Company A
```

The backend should detect possible duplicates.

However, `genericName` alone should not necessarily be unique because multiple brands/products can share the same generic ingredient.

---

# 78. Recommended Uniqueness Strategy

Strong candidate:

```text
medicineCode → unique
```

For product identity, a normalized combination may be used:

```text
genericName
+
brandName
+
manufacturer
+
strength
+
dosageForm
```

This can reduce accidental duplicate master entries.

---

# 79. Index Recommendations

Useful indexes:

```javascript
medicineCode: 1
medicineName: 1
genericName: 1
brandName: 1
manufacturer: 1
category: 1
isActive: 1
prescriptionRequired: 1
```

Potential compound index:

```javascript
{
    isActive: 1,
    category: 1
}
```

Useful for active medicine filtering.

---

# 80. Medicine + AuditLog

Important audit events:

```text
Medicine Created
Medicine Updated
Medicine Activated
Medicine Deactivated
Strength Changed
Route Changed
Withdrawal Period Changed
Prescription Requirement Changed
Manufacturer Changed
Category Changed
```

This is important because Medicine master data influences Treatment and Inventory workflows.

---

# 81. Medicine + Notification

Medicine can indirectly trigger notifications.

Examples:

```text
Low medicine stock
Medicine expired
Medicine near expiry
Withdrawal period active
Prescription-required medicine used
```

The notification itself belongs to:

```text
Notification
```

not Medicine.

---

# 82. Medicine + Inventory Notification

Example:

```text
Medicine:
Antibiotic A

Farm stock:
2 units

minimumStock:
10 units
```

Inventory service:

```text
2 < 10
```

creates:

```text
Low Stock Warning
```

Medicine provides identity; Inventory provides the actual stock state.

---

# 83. Medicine + Expiry Notification

If inventory contains:

```text
expiryDate = soon
```

the system can notify:

```text
Medicine nearing expiry
```

Again:

```text
Medicine
=
Product definition

Inventory
=
Specific farm stock with expiry
```

---

# 84. Prescription-Required Medicine Workflow

Example:

```text
Medicine.prescriptionRequired = true
```

User attempts treatment entry.

System checks:

```text
User role
Treatment authorization
Required approval
```

Then:

```text
Allow
```

or:

```text
Require authorization
```

The exact permission rule belongs to the application's role/authorization layer.

---

# 85. Completed Treatment and Medicine Consumption

Conceptually:

```text
Treatment
   ↓
Status/administration event
   ↓
Medicine consumption
   ↓
Inventory_Transaction
```

The exact point at which consumption occurs depends on how the Treatment workflow records individual doses.

Because the current Treatment model stores treatment-level medicine schedules rather than a separate per-dose administration collection, the implementation must define whether inventory is consumed:

```text
per completed treatment
```

or:

```text
per recorded administration/session
```

For precise stock accounting, per-administration tracking would be more exact, but that is not part of the current finalized model.

---

# 86. Important Current-Model Limitation

The current `Medicine` model is a master catalog.

It does not contain:

```text
price
stock
supplier
batch
expiry
```

Those belong to Inventory/procurement.

This separation is good.

However, exact medicine cost at the treatment-event level should come from:

```text
Inventory / transaction data
```

or a treatment-cost field, rather than Medicine master data.

---

# 87. Important Current-Model Limitation — No Dose History in Medicine

Medicine does not store:

```text
animal-specific dosage
frequency
duration
```

This is intentional.

Those belong to:

```text
Treatment.medicines[]
```

Therefore:

```text
Medicine
=
Reference product

Treatment.medicines[]
=
actual treatment instruction
```

---

# 88. Example — Medicine Master

```javascript
{
    medicineCode: "MED-ANT-001",

    medicineName: "Example Antibiotic Injection",

    genericName: "Example Active Ingredient",

    brandName: "Example Brand",

    manufacturer: "Example Pharmaceutical Company",

    category: "Antibiotic",

    dosageForm: "Injection",

    strength: "100 mg/ml",

    administrationRoute: "IM",

    description:
        "Example medicine master record.",

    storageCondition: "Store according to product label.",

    withdrawalPeriodDays: 7,

    prescriptionRequired: true,

    isActive: true
}
```

Values are illustrative only and should not be interpreted as veterinary prescribing instructions.

---

# 89. Example — Treatment Using Medicine

```javascript
{
    treatmentNo: "TRT-000001",

    farmId: "FARM-001",

    animalId: "AN-000125",

    diseaseId: "DIS-001",

    medicines: [
        {
            medicineId: "MED-ANT-001",
            dosage: 5,
            dosageUnit: "ml",
            frequency: "Once daily",
            duration: "5 days",
            administrationRoute: "IM",
            instructions: "Follow the authorized treatment plan."
        }
    ],

    status: "Ongoing"
}
```

The exact dose/frequency must come from the authorized veterinary treatment plan.

---

# 90. Example — Withdrawal Logic

Suppose:

```text
Medicine.withdrawalPeriodDays = 7
```

Treatment:

```text
Last applicable administration:
2026-08-17
```

Then the system can derive:

```text
Withdrawal end:
2026-08-24
```

and show:

```text
Withdrawal active
```

until the configured end condition is reached.

---

# 91. Example — Inactive Medicine

```text
Medicine M001
isActive = false
```

Result:

```text
Cannot normally select for new treatment
```

but:

```text
Historical Treatment → M001
```

remains readable.

---

# 92. Error Scenario — Invalid Route

Input:

```text
dosageForm = Ointment
administrationRoute = IV
```

Recommended:

```text
Validation error
```

---

# 93. Error Scenario — Negative Withdrawal Period

Input:

```text
withdrawalPeriodDays = -5
```

Reject.

---

# 94. Error Scenario — Inactive Medicine in New Treatment

Input:

```text
medicineId = M001
```

but:

```text
isActive = false
```

Recommended:

```text
Reject normal new-treatment selection
```

Historical records remain valid.

---

# 95. Error Scenario — Prescription Required

Medicine:

```text
prescriptionRequired = true
```

User without required authorization attempts to use it.

Recommended:

```text
Block or require authorized approval
```

according to the final role/permission system.

---

# 96. Error Scenario — Insufficient Inventory

Treatment requires medicine but:

```text
Inventory available quantity < required quantity
```

Recommended:

```text
Prevent administration/completion
```

unless an authorized exception is recorded.

---

# 97. Error Scenario — Expired Inventory

Medicine master is active:

```text
isActive = true
```

but selected farm stock is:

```text
Expired
```

The system should:

```text
Prevent use of expired stock
```

The product itself is not necessarily inactive; only that inventory stock is invalid.

---

# 98. Medicine + Animal Health Safety

Medicine data should support informed workflows, but the system should not automatically prescribe medication solely from:

```text
Disease
+
Symptoms
```

without an explicitly validated clinical decision-support system.

The current model supports:

```text
Recorded treatment management
```

rather than autonomous diagnosis/prescription.

---

# 99. Medicine + Disease Recommendation

Do not automatically infer:

```text
Disease A
→ Medicine B
```

from the Medicine model.

The current architecture intentionally keeps:

```text
Disease
Treatment
Medicine
```

separate.

The authorized user records the actual treatment relationship through:

```text
Treatment.medicines[]
```

---

# 100. Service-Layer Responsibilities

Recommended services:

```text
medicineService.create()
medicineService.update()
medicineService.deactivate()
medicineService.search()
medicineService.validateRoute()
medicineService.validateStock()
medicineService.getActiveMedicines()
```

Treatment service handles:

```text
Treatment + Medicine
```

relationships and administration logic.

---

# 101. Transaction Boundary for Treatment + Inventory

When a treatment event consumes medicine:

```text
BEGIN TRANSACTION

Validate Treatment
Validate Medicine
Validate Inventory
Record/Update Treatment
Reduce Inventory
Create Inventory_Transaction
Create AuditLog

COMMIT
```

If a critical operation fails:

```text
ROLLBACK
```

This avoids:

```text
Treatment says medicine used
but inventory still shows unused stock
```

or the reverse.

---

# 102. Final Architecture

```text
                    ┌──────────────┐
                    │   Disease    │
                    └──────┬───────┘
                           │
                           ▼
                    ┌──────────────┐
                    │  Treatment   │
                    └──────┬───────┘
                           │
                       medicines[]
                           │
                           ▼
                    ┌──────────────┐
                    │   Medicine   │
                    │ MASTER DATA  │
                    └──────┬───────┘
                           │
                           ▼
                    ┌──────────────┐
                    │  Inventory   │
                    │ Farm-specific│
                    └──────┬───────┘
                           │
                           ▼
                Inventory_Transaction
```

And for food-safety logic:

```text
Medicine
   ↓
withdrawalPeriodDays
   ↓
Treatment
   ↓
Last applicable administration
   ↓
Withdrawal end
   ↓
Milk / Animal sale warning
```

---

# 103. Final Business Rules Checklist

```text
[✓] Medicine is master/reference data
[✓] medicineCode should be unique
[✓] medicineName is required
[✓] genericName identifies the generic/active ingredient
[✓] brandName identifies commercial product identity
[✓] manufacturer identifies the product manufacturer
[✓] category is controlled by enum
[✓] dosageForm is controlled by enum
[✓] strength represents product concentration/strength
[✓] administrationRoute is controlled by enum
[✓] Route should be compatible with dosage form
[✓] Treatment stores animal-specific dosage
[✓] Treatment stores frequency and duration
[✓] Treatment stores actual treatment route
[✓] description is general medicine information
[✓] storageCondition defines required storage conditions
[✓] withdrawalPeriodDays is a master rule
[✓] Actual withdrawal end is animal/treatment-specific
[✓] prescriptionRequired controls treatment workflow authorization
[✓] isActive controls current availability
[✓] Deactivation is preferred over hard deletion
[✓] Medicine does not store farm-specific stock
[✓] Inventory stores farm-specific medicine stock
[✓] Treatment references Medicine through medicines[].medicineId
[✓] One Treatment can contain multiple medicines
[✓] Inventory consumption should follow actual treatment use
[✓] Insufficient stock should normally block consumption
[✓] Expired stock should not be used
[✓] Important master-data changes should be audited
[✓] Historical Treatment references must remain valid
[✓] Medicine should not automatically diagnose or prescribe
[✓] Disease → Treatment → Medicine is the current health flow
[✓] Withdrawal information can support milk/product warnings
[✓] Medicine usage can support treatment/health reports
[✓] Antibiotic usage can be analyzed from Treatment + Medicine category
[✓] Multi-collection treatment/inventory updates should use service/transaction logic
```

---

# 104. Final Status

**Model:** `Medicine`  
**Status:** ✅ Finalized  
**Business Logic:** ✅ Documented  
**Master Data:** ✅  
**Treatment Integration:** ✅  
**Inventory Integration:** ✅  
**Withdrawal Support:** ✅  
**Prescription Workflow:** ✅  
**Health Timeline Integration:** ✅  
**Reporting:** ✅  
**Auditability:** ✅  
**Historical Integrity:** ✅  

**Next Model:** `Treatment`
