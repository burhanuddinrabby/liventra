# Liventra — Business Logic Documentation
## Model 09: FeedRecord

**Document Version:** Business Logic v1.0  
**Project:** Liventra — Smart Livestock Farm Management System  
**Database:** MongoDB  
**Architecture:** MERN

---

# 1. Source / Finalized Model

The finalized `FeedRecord` model is:

```text
_id
farmId
animalId
recommendationId (optional)

feedingDate

items:
[
    {
        ingredientId,
        ingredientName,
        quantity,
        unit,
        unitPrice,
        totalCost
    }
]

totalQuantity
totalCost

feedingTime
fedBy

notes
createdAt
updatedAt
```

The current finalized DB model document defines these FeedRecord fields and explicitly links FeedRecord to the Feed Management module. fileciteturn8file5

The finalized model also shows the relationship between FeedRecommendation and FeedRecord: a recommendation can exist first, while FeedRecord stores what was actually fed. fileciteturn8file2

---

# 2. Purpose

`FeedRecord` stores the **actual feeding event** that occurred on the farm.

This is different from:

```text
FeedIngredient
→ What is the ingredient?

FeedFormula
→ What should the formula contain?

FeedRecommendation
→ What does the system recommend for this animal?

FeedRecord
→ What was actually fed?
```

Therefore:

```text
FeedRecommendation
        ↓
       Plan

FeedRecord
        ↓
      Reality
```

This distinction is essential for Liventra.

---

# 3. Why FeedRecord Is Necessary

Suppose the system recommends:

```text
Napier Grass = 12 kg
Concentrate = 3 kg
```

But the farmer actually feeds:

```text
Napier Grass = 10 kg
Concentrate = 2.5 kg
```

If Liventra stores only the recommendation, it cannot know what actually happened.

With FeedRecord:

```text
Recommended
    ↓
12 kg + 3 kg

Actual
    ↓
10 kg + 2.5 kg
```

The system can later calculate:

```text
Recommended vs Actual
```

This enables meaningful farm analytics.

---

# 4. Feed Module Architecture

The complete flow is:

```text
FeedIngredient
      ↓
FeedFormula
      ↓
FeedRecommendation
      ↓
FeedRecord
      ↓
Analytics / Reports
```

Each layer has a different responsibility.

---

# 5. FeedRecord Is an Event/Transaction Record

Unlike FeedIngredient and FeedFormula, FeedRecord represents something that **actually happened**.

Example:

```text
Date:
2026-08-17

Animal:
AN-000125

Time:
08:00 AM

Feed:
Napier Grass = 10 kg
Concentrate = 2.5 kg

Fed By:
Farm Member A
```

That event should remain in history.

---

# 6. Field-by-Field Business Logic

# 6.1 `_id`

### Type

MongoDB `ObjectId`

### Required

Yes

### Purpose

Unique identifier of the feeding event.

Example:

```text
FEED-000001
```

---

# 6.2 `farmId`

### Type

ObjectId / Farm reference

### Required

Yes

### Purpose

Identifies the farm where the feeding event occurred.

### Why Required?

Liventra supports farm-level data isolation.

Therefore:

```text
FeedRecord
   ↓
Farm
```

must be traceable.

### Used For

- Farm feed history
- Farm cost reports
- Authorization
- Inventory consumption
- Analytics
- Audit

---

# 6.3 `animalId`

### Type

ObjectId / Animal reference

### Required

Yes

### Purpose

Identifies the animal that received the feed.

Example:

```text
animalId → AN-000125
```

### Business Logic

One animal can have many FeedRecords.

```text
Animal A
   |
   +--- FeedRecord 1
   +--- FeedRecord 2
   +--- FeedRecord 3
   +--- FeedRecord 4
```

This creates the animal's feeding history.

---

# 6.4 `recommendationId`

### Type

ObjectId / FeedRecommendation reference

### Required

Optional

### Purpose

Connects the actual feeding event to the recommendation that motivated it.

Example:

```text
recommendationId → R-000100
```

### Why Optional?

Not every actual feeding event must originate from Liventra's recommendation system.

For example:

```text
Farmer manually feeds animal
```

without first generating a recommendation.

Then:

```text
recommendationId = null
```

is valid.

---

# 7. Recommendation → Actual Feeding Relationship

When recommendation exists:

```text
FeedRecommendation R1
        ↓
FeedRecord F1
```

This allows:

```text
What was recommended?
        ↓
What was actually fed?
```

If recommendation does not exist:

```text
FeedRecord F2
recommendationId = null
```

The event is still valid as an actual feeding record.

---

# 8. `feedingDate`

### Type

Date

### Required

Yes

### Purpose

Stores the date on which the feeding event occurred.

Example:

```text
2026-08-17
```

### Business Uses

- Daily feeding history
- Weekly/monthly reports
- Cost analysis
- Animal feeding timeline
- Feed consumption trends

---

# 9. Why Feeding Date Is Separate from createdAt

These represent different concepts.

### `feedingDate`

```text
When did the actual feeding happen?
```

### `createdAt`

```text
When was this database record created?
```

Example:

```text
Actual feeding:
August 17, 8:00 AM

Record entered:
August 17, 2:00 PM
```

Then:

```text
feedingDate = August 17
createdAt = August 17, 2:00 PM
```

This distinction is important.

---

# 10. `items[]`

This is the core of the FeedRecord.

Structure:

```javascript
items: [
    {
        ingredientId,
        ingredientName,
        quantity,
        unit,
        unitPrice,
        totalCost
    }
]
```

The finalized model uses this embedded item structure. fileciteturn8file2

---

# 11. Why `items[]` Is an Array

One feeding event can contain multiple ingredients.

Example:

```text
Morning Feeding

Napier Grass = 10 kg
Concentrate = 2.5 kg
Mineral Mixture = 0.1 kg
```

Instead of creating:

```text
3 separate FeedRecord documents
```

we can represent one feeding event:

```text
FeedRecord
    |
    +-- Napier
    +-- Concentrate
    +-- Mineral
```

This makes the event easier to understand and query.

---

# 12. `items[].ingredientId`

### Type

ObjectId / FeedIngredient reference

### Required

Yes

### Purpose

Identifies the ingredient actually fed.

Example:

```text
ingredientId → Napier Grass
```

### Business Rule

The ingredient should normally exist in FeedIngredient.

---

# 13. `items[].ingredientName`

### Type

String

### Required

Yes

### Purpose

Stores the human-readable ingredient name.

Example:

```text
Napier Grass
```

### Why Store Name if ID Exists?

This is a historical snapshot.

Suppose:

```text
Current FeedIngredient name:
Rice Bran
```

Later:

```text
Rice Bran Premium
```

Historical FeedRecord should still show the name used at the time of feeding.

Therefore:

```text
ingredientId
+
ingredientName snapshot
```

is intentional denormalization.

---

# 14. `items[].quantity`

### Type

Number

### Required

Yes

### Example

```text
10
```

with:

```text
unit = kg
```

means:

```text
10 kg
```

### Business Meaning

This is the **actual quantity fed**, not the recommended quantity.

That distinction is critical.

---

# 15. Recommended Quantity vs Actual Quantity

Example:

### Recommendation

```text
Napier = 12 kg
```

### FeedRecord

```text
Napier = 10 kg
```

Then:

```text
variance = actual - recommended

10 - 12 = -2 kg
```

This can later support adherence analysis.

---

# 16. `items[].unit`

### Type

Enum/string aligned with FeedIngredient units

### Required

Yes

### Examples

```text
kg
g
liter
ml
piece
```

### Purpose

Defines how the actual quantity should be interpreted.

Example:

```text
quantity = 10
unit = kg
```

---

# 17. Unit Consistency

The unit should be compatible with the referenced ingredient's standard unit.

Example:

```text
Ingredient:
Napier Grass
standard unit = kg

Record:
quantity = 10
unit = kg
```

If a different compatible unit is used:

```text
10000 g
```

the backend should use a centralized conversion mechanism where appropriate.

Avoid implementing conversion separately in multiple controllers.

---

# 18. `items[].unitPrice`

### Type

Number

### Required

Yes

### Example

```text
5
```

meaning:

```text
৳5/kg
```

if:

```text
unit = kg
```

### Purpose

Stores the actual/reference price used for this feeding event.

This is important for historical cost tracking.

---

# 19. Why Unit Price Is Stored in FeedRecord

Current FeedIngredient price can change.

Example:

```text
August:
Napier = ৳5/kg

September:
Napier = ৳7/kg
```

If an August FeedRecord only referenced FeedIngredient, its historical cost could become ambiguous.

By storing:

```text
unitPrice = 5
```

the August event remains historically accurate.

---

# 20. `items[].totalCost`

### Type

Number

### Required

Yes

### Example

```text
50
```

### Calculation

Conceptually:

```text
quantity × unitPrice = totalCost
```

Example:

```text
10 kg × ৳5/kg
=
৳50
```

### Business Rule

Backend should calculate and validate this value rather than blindly trusting client-provided totals.

---

# 21. `totalQuantity`

### Type

Number

### Required

Yes

### Purpose

Stores the total quantity of the feeding event according to the project's chosen aggregation basis.

Example:

```text
Napier = 10 kg
Concentrate = 2.5 kg

totalQuantity = 12.5 kg
```

### Important

This field becomes ambiguous if different units are mixed.

For example:

```text
10 kg
+
500 ml
```

cannot meaningfully become:

```text
10.5
```

without a common conversion basis.

Therefore, totalQuantity should be used only when all item quantities can be aggregated under a common unit/basis.

The final implementation should define this clearly.

---

# 22. `totalCost`

### Type

Number

### Required

Yes

### Purpose

Stores the total actual cost of the feeding event.

Conceptually:

```text
Sum(item.totalCost)
=
FeedRecord.totalCost
```

Example:

```text
Napier = ৳50
Concentrate = ৳175
Mineral = ৳10

Total = ৳235
```

---

# 23. Cost Integrity Rule

The server should calculate:

```text
totalCost = Σ item.totalCost
```

rather than allowing arbitrary client values.

Similarly:

```text
item.totalCost = item.quantity × item.unitPrice
```

should be validated/recalculated by the backend.

This prevents accidental or malicious cost manipulation.

---

# 24. `feedingTime`

### Type

Time/string or Date-derived time representation

### Required

Yes

### Example

```text
08:00
```

### Purpose

Stores when during the day the feeding event occurred.

### Business Uses

- Morning vs evening feeding
- Feeding schedule analysis
- Daily routine
- Comparison of feeding sessions

---

# 25. Feeding Date + Feeding Time

Together:

```text
feedingDate
+
feedingTime
```

identify the feeding event's actual occurrence.

Example:

```text
2026-08-17
08:00 AM
```

---

# 26. `fedBy`

### Type

ObjectId / FarmMember/User reference

### Required

Yes

### Purpose

Identifies who performed or recorded the feeding event.

Example:

```text
fedBy → FarmMember/User
```

### Business Uses

- Accountability
- Staff activity
- Audit
- Feeding responsibility
- Farm operation tracking

---

# 27. `fedBy` and AuditLog

These have different purposes.

### `fedBy`

```text
Who actually performed the feeding?
```

### `AuditLog.userId`

```text
Who performed the database/system action?
```

These may be the same person, but they do not have to be.

Example:

```text
Worker A fed the animal.

Worker B entered the record later.
```

Then:

```text
fedBy = Worker A
AuditLog.userId = Worker B
```

This is a valuable distinction.

---

# 28. `notes`

### Type

String

### Required

Optional

### Example

```text
Animal consumed only part of the concentrate.
```

### Purpose

Stores operational observations.

Examples:

```text
Feed spilled
Animal refused feed
Extra feed given
Feeding delayed
Animal showed low appetite
```

This can be useful for later analysis.

---

# 29. `createdAt`

### Type

Date

### Required

Yes

### Purpose

Stores when the FeedRecord document was created.

---

# 30. `updatedAt`

### Type

Date

### Required

Yes

### Purpose

Stores when the FeedRecord document was last updated.

### Important

Actual feeding history should not be casually edited after finalization.

If correction is necessary, the change should be auditable.

---

# 31. FeedRecord Creation Workflow

```text
Farmer/Farm Member
       ↓
Select Animal
       ↓
Select feeding date/time
       ↓
Select ingredients
       ↓
Enter actual quantities
       ↓
Load/enter applicable prices
       ↓
Calculate item costs
       ↓
Calculate total cost
       ↓
Optionally link recommendation
       ↓
Select fedBy
       ↓
Add notes
       ↓
Validate
       ↓
Save FeedRecord
       ↓
Update Inventory
       ↓
Create AuditLog
       ↓
Optional Notification
```

---

# 32. FeedRecord + Inventory

This is one of the most important integrations.

When feed is actually consumed:

```text
FeedRecord created
       ↓
Ingredient quantity consumed
       ↓
Inventory decreases
```

Example:

```text
Before:
Napier stock = 500 kg

FeedRecord:
Napier = 10 kg

After:
Napier stock = 490 kg
```

---

# 33. Inventory Transaction

The consumption should be represented in the inventory transaction/history system.

Conceptually:

```text
Inventory_Transaction
    ↓
itemId
consumption = 10 kg
```

The exact finalized Inventory transaction structure should remain the source of truth for its own implementation.

The important business rule is:

> Actual feeding consumption must be reflected in farm inventory.

---

# 34. Inventory Update Must Be Atomic

A dangerous situation is:

```text
FeedRecord saved
BUT
Inventory not decreased
```

or:

```text
Inventory decreased
BUT
FeedRecord failed
```

These create inconsistent data.

Therefore, where MongoDB transaction support is available, the operation should ideally be atomic:

```text
Create FeedRecord
+
Create inventory consumption transaction
+
Update inventory balance
```

all within one transaction.

If one fails:

```text
Rollback
```

---

# 35. Insufficient Stock

Suppose:

```text
Napier stock = 5 kg
```

but user enters:

```text
Feed quantity = 10 kg
```

The system should not silently allow the inventory to become negative unless the project explicitly supports negative stock.

Recommended behavior:

```text
Check stock
      ↓
Insufficient
      ↓
Show warning/error
      ↓
Do not finalize consumption
```

Possible exception:

```text
Authorized override
```

if the business decides to support manual adjustments.

---

# 36. FeedRecord + FeedRecommendation

If:

```text
recommendationId = R1
```

then the system can compare:

```text
Recommendation
      ↓
Actual
```

Example:

```text
Recommended:
Napier = 12 kg
Concentrate = 3 kg

Actual:
Napier = 10 kg
Concentrate = 2.5 kg
```

---

# 37. Recommended vs Actual Variance

For each ingredient:

```text
varianceQuantity
=
actualQuantity - recommendedQuantity
```

Example:

```text
Napier:
10 - 12 = -2 kg
```

Percentage variance:

```text
(actual - recommended) / recommended × 100
```

Example:

```text
(10 - 12) / 12 × 100
=
-16.67%
```

This can later be used in reports.

These calculated metrics do not need to be stored in FeedRecord unless the project later needs materialized analytics.

---

# 38. Why Recommended vs Actual Matters

This enables Liventra to answer:

```text
Did the farmer follow the recommendation?

How much less/more was actually fed?

Did actual feed cost exceed estimated cost?

Did deviation affect weight gain?

Did deviation affect milk production?
```

This is a major advantage over a simple feeding log.

---

# 39. FeedRecord + WeightRecord

Suppose:

```text
FeedRecords:
daily feeding history
```

and:

```text
WeightRecord:
weekly weight
```

Then Liventra can analyze:

```text
Feed intake
     ↓
Weight gain
     ↓
ADG
```

This can support future feed-efficiency analysis.

---

# 40. FeedRecord + MilkRecord

For lactating animals:

```text
FeedRecord
     ↓
Feed consumption
     ↓
MilkRecord
     ↓
Milk production
```

This allows analysis such as:

```text
Feed cost per liter of milk
```

or:

```text
Feed quantity vs milk yield
```

Actual calculations should be based on complete historical data and appropriate nutrition/business definitions.

---

# 41. FeedRecord + Animal Lifecycle

If:

```text
animal.lifeStatus = Sold
```

new feeding events should normally be blocked.

If:

```text
animal.lifeStatus = Dead
```

feeding should be blocked.

If:

```text
animal.lifeStatus = Transferred
```

the farm ownership/location rules should determine whether new FeedRecords can be created under the old farm.

---

# 42. Feeding Validation Based on Animal Status

Recommended validation:

```text
Animal exists
       ↓
Animal belongs to farm
       ↓
Animal is active
       ↓
Animal is not Sold
       ↓
Animal is not Dead
       ↓
Proceed
```

This prevents invalid feeding history.

---

# 43. Feeding During Treatment / Quarantine

If:

```text
healthStatus = Under Treatment
```

or:

```text
healthStatus = Quarantine
```

the system may allow feeding, but the UI can display relevant warnings.

Feeding should not automatically be blocked merely because the animal is under treatment; the exact business rule depends on the project's veterinary/feed policy.

---

# 44. Multiple Feeding Sessions Per Day

The model supports multiple FeedRecords for the same animal and date.

Example:

```text
08:00 AM
FeedRecord 1

01:00 PM
FeedRecord 2

06:00 PM
FeedRecord 3
```

This is preferable to forcing all daily feeding into one record.

Then daily totals can be calculated:

```text
Σ all FeedRecords for animal/date
```

---

# 45. Morning / Evening Example

Morning:

```text
feedingDate = 2026-08-17
feedingTime = 08:00

Napier = 10 kg
Concentrate = 2 kg
```

Evening:

```text
feedingDate = 2026-08-17
feedingTime = 18:00

Napier = 8 kg
Concentrate = 1 kg
```

Daily actual:

```text
Napier = 18 kg
Concentrate = 3 kg
```

The system can aggregate this for daily reporting.

---

# 46. Feeding Date Validation

A normal operational rule:

```text
feedingDate
```

should not be arbitrarily far in the future.

Past-date entry may be allowed because farmers may enter records later.

If backdated entry is allowed:

```text
AuditLog
```

becomes particularly important.

---

# 47. Backdated Feeding

Example:

```text
Actual feeding:
August 15

Record entered:
August 17
```

Valid if the system permits backdated records.

Store:

```text
feedingDate = Aug 15
createdAt = Aug 17
```

This preserves the difference between occurrence and entry.

---

# 48. Editing FeedRecord

Because FeedRecord is historical event data, edits should be controlled.

Example:

```text
Wrong quantity entered:
10 kg
```

Correction:

```text
12 kg
```

The system should:

```text
Update record
+
Audit old/new values
+
Correct inventory transaction if necessary
```

This is important.

Otherwise:

```text
FeedRecord says 12 kg
Inventory says 10 kg consumed
```

could occur.

---

# 49. Correcting Inventory After FeedRecord Edit

If an existing FeedRecord is edited:

```text
Old consumption = 10 kg
New consumption = 12 kg
```

Inventory adjustment should be:

```text
Additional consumption = 2 kg
```

not:

```text
Consume 12 kg again
```

This is why FeedRecord and Inventory updates need carefully designed service-layer transactions.

---

# 50. Deleting FeedRecord

Hard delete is generally discouraged after inventory and reporting effects occur.

Preferred:

```text
Correction workflow
```

rather than deleting historical events.

If deletion is absolutely required:

```text
Reverse inventory transaction
+
Audit deletion
```

must happen together.

---

# 51. Price Source Logic

There are potentially three prices:

```text
FeedIngredient.pricePerUnit
Inventory purchase price
FeedRecord.unitPrice
```

Their meanings should remain distinct.

### FeedIngredient

Current/reference master price.

### Inventory purchase

Historical procurement price.

### FeedRecord

Price used for the actual feeding event/cost calculation.

This separation is important for accurate financial reporting.

---

# 52. Actual Cost vs Estimated Cost

Recommendation:

```text
estimatedDailyCost
```

FeedRecord:

```text
totalCost
```

Therefore:

```text
Estimated:
৳280/day

Actual:
৳235
```

The difference is:

```text
Actual - Estimated
=
-৳45
```

This can be reported.

---

# 53. Cost Variance

Conceptually:

```text
costVariance
=
actualTotalCost - recommendedEstimatedCost
```

This should normally be calculated at report/analytics level rather than permanently stored in FeedRecord.

---

# 54. Authorization

Recommended permissions:

### Farm Owner / Manager

Can:

```text
Create
View
Correct
```

### Farm Member

Can:

```text
Create
View
```

depending on assigned role.

### Unauthorized user

Cannot create records for another farm.

The backend must verify:

```text
fedBy/user
+
farm membership
+
permission
+
animal belongs to farm
```

---

# 55. CRUD Business Rules

## Create

Requires:

```text
Farm
Animal
Feeding date/time
At least one item
Actual quantity
Unit
Price
FedBy
```

---

## Read

Authorized farm members can view records for their farm.

---

## Update

Allowed only with permission.

Changes should be audited.

Inventory must be reconciled.

---

## Delete

Avoid hard deletion.

Use controlled correction/reversal workflow.

---

# 56. Validation Rules

| Field | Validation |
|---|---|
| farmId | Required, valid Farm |
| animalId | Required, valid Animal belonging to farm |
| recommendationId | Optional, valid Recommendation |
| feedingDate | Required, valid date |
| items | At least one item |
| ingredientId | Required, valid FeedIngredient |
| ingredientName | Required snapshot |
| quantity | Positive |
| unit | Valid/compatible unit |
| unitPrice | Non-negative |
| totalCost | Calculated/validated |
| totalQuantity | Valid aggregation basis |
| totalCost | Calculated from items |
| feedingTime | Valid time |
| fedBy | Required, authorized farm member |
| notes | Optional |

---

# 57. Important Data Integrity Rules

### Rule 1

Animal must belong to the same farm as `farmId`.

### Rule 2

`fedBy` must be authorized for that farm.

### Rule 3

Every ingredient must exist.

### Rule 4

Actual quantity must be positive.

### Rule 5

Price cannot be negative.

### Rule 6

Item total cost must match:

```text
quantity × unitPrice
```

### Rule 7

FeedRecord total cost must match the sum of item costs.

### Rule 8

Inventory consumption must correspond to actual FeedRecord quantities.

### Rule 9

Historical records should not be silently altered.

### Rule 10

Official corrections must be auditable.

---

# 58. Audit Rules

Important FeedRecord events:

```text
FeedRecord Created
FeedRecord Updated
FeedRecord Corrected
FeedRecord Deleted/Voided
Quantity Changed
Price Changed
Ingredient Added
Ingredient Removed
Feeding Time Changed
FedBy Changed
Inventory Reconciled
```

AuditLog should preserve:

```text
oldData
newData
changedFields
user
timestamp
```

according to the project's AuditLog architecture.

---

# 59. Notification Rules

Possible notifications:

```text
Low feed stock
Insufficient stock during feeding entry
Unusual feeding deviation
Feeding missed
High cost
Recommendation significantly deviated
```

Not all of these need to be implemented initially.

The important point is that FeedRecord provides the actual event data required for such future features.

---

# 60. Reporting

FeedRecord supports:

```text
Daily Feed Consumption
Weekly Feed Consumption
Monthly Feed Consumption
Animal-wise Feed History
Farm-wise Feed Cost
Ingredient-wise Consumption
Feeding Cost per Animal
Recommended vs Actual
Actual Feed Cost vs Estimated Cost
Feeding Frequency
```

---

# 61. Daily Feed Summary

For:

```text
Animal A
Date = 2026-08-17
```

system can aggregate:

```text
All FeedRecords on that date
```

Example:

```text
Morning:
Napier = 10 kg
Concentrate = 2 kg

Evening:
Napier = 8 kg
Concentrate = 1 kg
```

Daily result:

```text
Napier = 18 kg
Concentrate = 3 kg
```

---

# 62. Recommended vs Actual Report

Example:

| Ingredient | Recommended | Actual | Variance |
|---|---:|---:|---:|
| Napier | 12 kg | 10 kg | -2 kg |
| Concentrate | 3 kg | 2.5 kg | -0.5 kg |

This can be used to understand feeding adherence.

---

# 63. Feed Cost Report

Example:

```text
Animal A
Daily actual feed cost = ৳235

Monthly:
Σ FeedRecord.totalCost
```

This can be compared with:

```text
FeedRecommendation.estimatedDailyCost
```

to evaluate cost deviation.

---

# 64. Feed Efficiency Future Analysis

With:

```text
FeedRecord
+
WeightRecord
```

the system can potentially calculate:

```text
Feed input
vs
Weight gain
```

With:

```text
FeedRecord
+
MilkRecord
```

the system can potentially analyze:

```text
Feed cost
vs
Milk production
```

These should be treated as analytics derived from historical data, not as fields that must be stored directly in FeedRecord.

---

# 65. Example — Full FeedRecord

Illustrative example:

```javascript
{
    farmId: "FARM-001",
    animalId: "AN-000125",
    recommendationId: "REC-000100",

    feedingDate: "2026-08-17",

    items: [
        {
            ingredientId: "ING-001",
            ingredientName: "Napier Grass",
            quantity: 10,
            unit: "kg",
            unitPrice: 5,
            totalCost: 50
        },
        {
            ingredientId: "ING-002",
            ingredientName: "Concentrate",
            quantity: 2.5,
            unit: "kg",
            unitPrice: 70,
            totalCost: 175
        }
    ],

    totalQuantity: 12.5,
    totalCost: 225,

    feedingTime: "08:00",
    fedBy: "FARMMEMBER-001",

    notes: "Animal consumed the full ration.",

    createdAt: "2026-08-17T08:15:00",
    updatedAt: "2026-08-17T08:15:00"
}
```

All numerical values are illustrative only.

---

# 66. Important Distinction: `feedingDate` vs `generatedAt`

Recommendation:

```text
generatedAt
=
when the system generated the recommendation
```

FeedRecord:

```text
feedingDate + feedingTime
=
when the actual feeding occurred
```

These dates may be different.

Example:

```text
Recommendation:
08:00 AM

Actual feeding:
08:30 AM
```

This is normal.

---

# 67. Important Distinction: Recommendation vs Actual Price

Recommendation:

```text
estimatedCost
```

FeedRecord:

```text
unitPrice
totalCost
```

Therefore:

```text
Recommendation
=
expected/estimated economics

FeedRecord
=
actual recorded economics
```

---

# 68. Important Distinction: FeedRecord vs Inventory Transaction

FeedRecord:

```text
What was fed to the animal?
```

Inventory transaction:

```text
How did farm stock change because of that consumption?
```

They are related but not identical.

---

# 69. Error Scenario — FeedRecord Saved but Inventory Failed

Bad state:

```text
FeedRecord:
10 kg consumed

Inventory:
still shows old stock
```

Solution:

Use transaction/compensation logic so that:

```text
FeedRecord
+
Inventory consumption
```

remain consistent.

---

# 70. Error Scenario — Inventory Updated but FeedRecord Failed

Bad state:

```text
Inventory:
-10 kg

FeedRecord:
not created
```

Again:

```text
Atomic transaction
```

or a reliable compensation/reconciliation mechanism is required.

---

# 71. Error Scenario — User Enters Wrong Quantity

Example:

```text
Actual:
10 kg
```

entered as:

```text
100 kg
```

Correction should:

```text
Update FeedRecord
+
Reverse previous inventory effect
+
Apply corrected inventory effect
+
Audit change
```

---

# 72. Error Scenario — Animal Is from Another Farm

User tries:

```text
farmId = FARM-A
animalId = Animal belonging to FARM-B
```

Backend must reject.

Never trust only frontend filtering.

---

# 73. Error Scenario — Unauthorized `fedBy`

User selects another farm's member.

Backend must verify:

```text
fedBy belongs to farmId
```

Otherwise the record should be rejected.

---

# 74. Error Scenario — Negative Quantity

Example:

```text
quantity = -10
```

Reject.

Similarly:

```text
unitPrice = -5
```

must be rejected.

---

# 75. Historical Integrity

FeedRecord is historical operational data.

Therefore:

```text
FeedIngredient changes
        ↓
Do not rewrite FeedRecord

FeedFormula changes
        ↓
Do not rewrite FeedRecord

FeedRecommendation changes
        ↓
Do not rewrite FeedRecord
```

The FeedRecord represents what actually happened.

---

# 76. Recommended vs Actual Is the Key Business Value

The complete flow becomes:

```text
FeedFormula
     ↓
FeedRecommendation
     ↓
Recommended quantity/cost
     ↓
Farmer feeds animal
     ↓
FeedRecord
     ↓
Actual quantity/cost
     ↓
Inventory update
     ↓
Weight/Milk outcome
     ↓
Analytics
```

This is the strongest conceptual relationship in the Feed module.

---

# 77. Defense Questions

### Q: What is the difference between FeedRecommendation and FeedRecord?

**Answer:**

> "FeedRecommendation represents what the system recommends for an animal, while FeedRecord represents what was actually fed. This allows Liventra to compare planned feeding with real farm practice."

---

### Q: Why is recommendationId optional?

**Answer:**

> "Because farmers may record actual feeding manually without first generating a system recommendation. In that case the FeedRecord remains valid but has no recommendationId."

---

### Q: Why store ingredientName when ingredientId already exists?

**Answer:**

> "ingredientId provides the master reference, while ingredientName preserves the human-readable value at the time of the actual feeding event for historical readability."

---

### Q: Why store unitPrice in FeedRecord?

**Answer:**

> "Because ingredient prices change over time. Storing unitPrice preserves the actual price used for that historical feeding event."

---

### Q: Why not calculate totalCost every time from FeedIngredient?

**Answer:**

> "Because the current FeedIngredient price may be different from the price at the time of feeding. FeedRecord stores historical cost information so past financial records remain accurate."

---

### Q: What happens to inventory after FeedRecord is created?

**Answer:**

> "The consumed quantity should be recorded as an inventory consumption transaction and the corresponding stock should be reduced. Ideally the FeedRecord and inventory update are performed atomically."

---

### Q: What if there is insufficient stock?

**Answer:**

> "The system should detect insufficient inventory before finalizing the feeding event and normally reject the transaction or require an authorized override."

---

### Q: Why can an animal have multiple FeedRecords per day?

**Answer:**

> "Because feeding can occur multiple times per day. Each record represents an actual feeding event, and daily totals can be calculated by aggregating the day's records."

---

### Q: Why not store daily feeding in one document?

**Answer:**

> "Event-level records provide better traceability, time-specific history and correction capability. Daily summaries can be derived by aggregation."

---

# 78. Final Business Rules Checklist

```text
[✓] FeedRecord represents actual feeding
[✓] FeedRecord is different from FeedRecommendation
[✓] FeedRecord belongs to a farm
[✓] FeedRecord belongs to an animal
[✓] recommendationId is optional
[✓] Feeding date is separate from record creation time
[✓] Multiple feeding events per day are allowed
[✓] items[] stores actual ingredients
[✓] ingredientId references FeedIngredient
[✓] ingredientName is preserved as a snapshot
[✓] quantity means actual quantity fed
[✓] unit must be valid/compatible
[✓] unitPrice preserves historical price
[✓] totalCost is derived from quantity × unitPrice
[✓] totalCost equals sum of item costs
[✓] feedingTime records event time
[✓] fedBy records who performed the feeding
[✓] fedBy and AuditLog user can be different
[✓] Inventory must reflect actual consumption
[✓] Inventory and FeedRecord should remain consistent
[✓] Insufficient stock should normally block finalization
[✓] Historical feeding records should not be silently rewritten
[✓] Corrections must be auditable
[✓] Recommended vs Actual analysis is supported
[✓] Feed cost variance is supported
[✓] Weight/Milk outcome analysis is supported
[✓] Sold/Dead animals should normally not receive new feeding records
[✓] Hard deletion should generally be avoided
```

---

# 79. Final Status

**Model:** `FeedRecord`  
**Status:** ✅ Finalized  
**Business Logic:** ✅ Documented  
**Actual Feeding Tracking:** ✅  
**Recommendation Comparison:** ✅  
**Inventory Integration:** ✅  
**Historical Cost Tracking:** ✅  
**Next Model:** `MilkRecord`
