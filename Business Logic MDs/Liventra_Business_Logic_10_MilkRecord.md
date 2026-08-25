# Liventra — Business Logic Documentation
## Model 10: MilkRecord

**Document Version:** Business Logic v1.0  
**Project:** Liventra — Smart Livestock Farm Management System  
**Database:** MongoDB  
**Architecture:** MERN  
**Module:** Production

---

# 1. Source / Finalized Model

The finalized `MilkRecord` model is:

```text
_id
farmId
animalId
recordDate
milkingSession: Morning | Afternoon | Evening | Night
quantity
unit: liter | ml
fatPercentage
proteinPercentage
snfPercentage
quality: Excellent | Good | Average | Poor | Rejected
milkingMethod: Manual | Milking Machine | Automatic
recordedBy
remarks
createdAt
updatedAt
```

The current finalized DB model document defines `MilkRecord` with these fields and places it in the **Production Module** alongside `WeightRecord`. fileciteturn9file8

---

# 2. Purpose

`MilkRecord` stores the **actual milk production event** for an individual animal.

It answers:

> **"How much milk did this animal produce during this particular milking session, and what was the recorded milk quality?"**

Example:

```text
Animal:
AN-000125

Date:
2026-08-17

Session:
Morning

Quantity:
8 liter

Fat:
3.8%

Protein:
3.2%

SNF:
8.5%

Quality:
Good

Method:
Milking Machine
```

This becomes part of the animal's historical production record.

---

# 3. Why MilkRecord Is Necessary

Milk production is not a static property of an animal.

It changes over time because of factors such as:

```text
Lactation stage
Feed
Health
Pregnancy
Season
Breed
Milking practice
Animal condition
```

Therefore, storing only:

```text
animal.currentMilk = 15 liter
```

would lose historical information.

Instead:

```text
Animal
   ↓
MilkRecord 1
MilkRecord 2
MilkRecord 3
MilkRecord 4
...
```

creates a production history.

---

# 4. Production Module Architecture

The current DB architecture places:

```text
WeightRecord
MilkRecord
```

under the Production Module. fileciteturn9file8

Conceptually:

```text
Animal
   |
   +------ WeightRecord
   |
   +------ MilkRecord
              |
              +---- Production Analytics
```

Milk production can also interact with:

```text
FeedRecommendation
FeedRecord
Pregnancy
Breeding
SensorData (future)
Income
Report
```

---

# 5. Important Difference: MilkRecord vs Animal

The Animal model stores the animal's identity and current state.

MilkRecord stores:

```text
production event/history
```

Therefore:

```text
Animal
=
Who is this animal?

MilkRecord
=
What did this animal produce at a particular time?
```

---

# 6. Important Difference: MilkRecord vs FeedRecord

These are complementary.

```text
FeedRecord
=
What was fed?

MilkRecord
=
What milk was produced?
```

Together:

```text
Feed
 ↓
Milk Production
```

can later support productivity analysis.

---

# 7. Field-by-Field Business Logic

# 7.1 `_id`

### Type

MongoDB `ObjectId`

### Required

Yes

### Purpose

Unique identifier of the milk-production record.

Example:

```text
MongoDB ObjectId
```

---

# 7.2 `farmId`

### Type

ObjectId / Farm reference

### Required

Yes

### Purpose

Identifies the farm where the milk production was recorded.

### Why Required?

Liventra is multi-farm capable.

Therefore, the backend must know:

```text
Which farm owns this production record?
```

### Used For

- Farm-level milk dashboard
- Milk production reports
- Revenue calculations
- Authorization
- Data isolation
- Analytics

---

# 7.3 `animalId`

### Type

ObjectId / Animal reference

### Required

Yes

### Purpose

Identifies the animal that produced the milk.

Example:

```text
animalId → AN-000125
```

### Relationship

One animal can have many MilkRecords:

```text
Animal A
   |
   +--- MilkRecord 1
   +--- MilkRecord 2
   +--- MilkRecord 3
   +--- MilkRecord 4
```

---

# 7.4 Farm-Animal Integrity Rule

The backend must verify:

```text
MilkRecord.farmId
        =
Animal.farmId
```

If:

```text
Animal belongs to Farm B
```

but a user submits:

```text
farmId = Farm A
```

the request must be rejected.

Never depend only on frontend filtering for this.

---

# 7.5 `recordDate`

### Type

Date

### Required

Yes

### Purpose

Stores the date on which the milk was actually produced/recorded for the specified milking session.

Example:

```text
2026-08-17
```

### Business Uses

- Daily milk production
- Weekly/monthly reports
- Lactation history
- Milk trend charts
- Income calculations

---

# 7.6 `recordDate` vs `createdAt`

These represent different events.

### `recordDate`

```text
When did the milk production event occur?
```

### `createdAt`

```text
When was the database record created?
```

Example:

```text
Milk produced:
August 17, 6:00 AM

Record entered:
August 17, 10:00 AM
```

Then:

```text
recordDate = August 17
createdAt = August 17, 10:00 AM
```

This distinction is important when records are entered later.

---

# 7.7 `milkingSession`

### Type

Enum

### Required

Yes

### Allowed Values

```text
Morning
Afternoon
Evening
Night
```

These are the finalized session values in the current DB model. fileciteturn9file8

### Purpose

Identifies which milking session the record represents.

---

# 8. Why Milking Session Is Necessary

Milk production can vary significantly across sessions.

Example:

```text
Morning:
8 L

Afternoon:
5 L

Evening:
7 L
```

Without `milkingSession`, the system could not distinguish these production events.

---

# 9. Duplicate Session Rule

For a normal operational setup, the system should prevent accidental duplicate records for:

```text
same farm
+
same animal
+
same recordDate
+
same milkingSession
```

unless the farm explicitly allows multiple records per session.

### Recommended Rule

One official MilkRecord per:

```text
Animal + Date + Session
```

This prevents accidental duplication.

If a correction is needed, update/correct the existing record with audit tracking rather than creating an accidental duplicate.

---

# 10. `quantity`

### Type

Number

### Required

Yes

### Example

```text
8
```

### Meaning

The amount of milk produced during the specified session.

If:

```text
unit = liter
```

then:

```text
8 liter
```

---

# 11. Quantity Validation

Recommended:

```text
quantity > 0
```

A normal milk production record should not contain:

```text
-5
```

or an invalid negative quantity.

For zero production, the implementation needs a deliberate policy.

Possible meaning:

```text
0 = animal produced no milk
```

but this should be distinguished from:

```text
record not taken
```

Therefore, the UI should avoid silently treating missing records as zero production.

---

# 12. `unit`

### Type

Enum

### Required

Yes

### Allowed Values

```text
liter
ml
```

The current finalized model specifies these units. fileciteturn9file8

### Purpose

Defines the measurement unit of the recorded milk quantity.

---

# 13. Unit Normalization

For analytics, Liventra should use one internal standard unit.

Recommended:

```text
liter
```

Example:

```text
5000 ml
```

can be normalized internally to:

```text
5 liter
```

This prevents problems when calculating:

```text
Daily milk
Monthly milk
Average milk
Milk price
Milk revenue
```

The conversion should be centralized.

---

# 14. `fatPercentage`

### Type

Number

### Required

Optional depending on available testing data

### Example

```text
3.8
```

meaning:

```text
3.8%
```

### Purpose

Stores the recorded milk fat percentage.

### Business Uses

- Milk quality analysis
- Production quality monitoring
- Farmer reports
- Potential pricing/quality analysis
- Historical comparison

---

# 15. Fat Percentage Validation

Recommended:

```text
fatPercentage >= 0
```

and the value should be within a scientifically reasonable range for the project's supported cattle/milk context.

The exact acceptable range should come from the project's approved dairy/lab data standard rather than being arbitrarily hard-coded.

---

# 16. `proteinPercentage`

### Type

Number

### Required

Optional depending on testing availability

### Example

```text
3.2
```

meaning:

```text
3.2%
```

### Purpose

Stores recorded milk protein percentage.

### Business Uses

- Milk quality analysis
- Production comparison
- Nutrition/productivity analysis
- Historical trend

---

# 17. Protein Percentage Validation

Recommended:

```text
proteinPercentage >= 0
```

The final scientifically valid range should be determined from the project's approved dairy-quality reference.

---

# 18. `snfPercentage`

### Type

Number

### Required

Optional depending on measurement availability

### Example

```text
8.5
```

meaning:

```text
8.5%
```

### SNF Meaning

SNF = **Solids-Not-Fat**

It represents milk solids other than fat.

### Purpose

Stores the recorded SNF percentage for milk quality assessment.

---

# 19. Why SNF Is Useful

Together:

```text
fatPercentage
proteinPercentage
snfPercentage
```

give more information than milk volume alone.

Therefore:

```text
Milk Production
+
Milk Composition
```

can be analyzed separately.

---

# 20. `quality`

### Type

Enum

### Required

Yes/according to business policy

### Allowed Values

```text
Excellent
Good
Average
Poor
Rejected
```

These are the finalized quality values. fileciteturn9file8

### Purpose

Provides a human-readable classification of milk quality.

---

# 21. Quality Is Not the Same as Quantity

Example:

```text
Quantity = 10 L
Quality = Poor
```

is valid.

Likewise:

```text
Quantity = 5 L
Quality = Excellent
```

is possible.

Milk volume and milk quality represent different dimensions.

---

# 22. Quality Determination

The quality value can be:

```text
Manually selected
```

or, in a future system:

```text
Automatically determined from quality measurements
```

The current DB model does not define a specific quality-scoring algorithm.

Therefore, the application should not claim that `quality` is automatically calculated unless such a rule is explicitly implemented.

---

# 23. `milkingMethod`

### Type

Enum

### Required

Yes/according to business policy

### Allowed Values

```text
Manual
Milking Machine
Automatic
```

These are the finalized values. fileciteturn9file8

### Purpose

Records how the milk was obtained.

---

# 24. Why Milking Method Matters

Different milking methods can affect operational analysis.

For example:

```text
Manual
```

versus:

```text
Milking Machine
```

can be compared for:

- Production consistency
- Labor requirements
- Operational efficiency
- Quality trends

This becomes useful for farm-level analytics.

---

# 25. `recordedBy`

### Type

ObjectId / User or FarmMember reference

### Required

Yes

### Purpose

Identifies who recorded the milk-production data.

Example:

```text
recordedBy → FarmMember
```

### Business Uses

- Accountability
- Data traceability
- Staff activity
- Audit
- Record verification

---

# 26. `recordedBy` vs AuditLog User

These can be different.

Example:

```text
Worker A
→ measured milk

Worker B
→ entered data later
```

Then:

```text
recordedBy = Worker A
```

while:

```text
AuditLog.userId = Worker B
```

This distinction is useful.

---

# 27. `remarks`

### Type

String

### Required

Optional

### Example

```text
Milk quantity was lower than usual.
Animal appeared stressed during milking.
```

### Purpose

Stores operational observations that are not represented by structured fields.

Possible notes:

```text
Animal refused milking
Machine issue
Spillage observed
Unusual milk appearance
Measurement repeated
```

---

# 28. `createdAt`

### Type

Date

### Required

Yes

### Purpose

Stores when the MilkRecord was created.

---

# 29. `updatedAt`

### Type

Date

### Required

Yes

### Purpose

Stores when the MilkRecord was last modified.

### Important

Historical milk records should not be casually changed without audit tracking.

---

# 30. Milk Recording Workflow

```text
Milk Production Occurs
        ↓
Select Animal
        ↓
Select Date
        ↓
Select Milking Session
        ↓
Enter Quantity
        ↓
Select Unit
        ↓
Enter Quality Metrics
        ↓
Select Quality
        ↓
Select Milking Method
        ↓
Select Recorded By
        ↓
Add Remarks if needed
        ↓
Validate
        ↓
Save MilkRecord
        ↓
AuditLog
        ↓
Optional Income/Report Update
```

---

# 31. MilkRecord and Animal Lifecycle

Not every animal should have a MilkRecord.

For example:

```text
Male
```

normally should not be treated as a milk-producing animal.

Similarly:

```text
Calf
```

should normally not have routine milk-production records.

The application should validate animal eligibility according to the project's livestock business rules.

---

# 32. Lactation Validation

The Animal model contains:

```text
reproductiveStatus
```

with:

```text
Lactating
```

as one of its values.

Therefore, the system can use animal reproductive state as part of validation.

Recommended:

```text
If animal is not lactating:
    warn or restrict normal milk recording
```

However, the exact exception rules should be defined by the project.

The DB model itself does not state that every MilkRecord must have a lactating animal, so this is an application-level business rule rather than a schema fact.

---

# 33. MilkRecord and Pregnancy

Pregnancy can affect milk production and lactation stage.

Therefore, MilkRecord can later be analyzed together with:

```text
Pregnancy
+
Breeding
+
MilkRecord
```

Example:

```text
Pregnancy confirmed
      ↓
Milk yield trend
      ↓
Drying-off period
```

This is useful for reproductive and production analytics.

---

# 34. MilkRecord and FeedRecord

One of the strongest Liventra analytics relationships is:

```text
FeedRecord
      ↓
Feed quantity/cost
      ↓
MilkRecord
      ↓
Milk production
```

This allows analysis such as:

```text
How much feed was actually consumed?
How much milk was produced?
What was the feed cost?
```

---

# 35. MilkRecord and FeedRecommendation

FeedRecommendation may use:

```text
milkYield
```

as an input.

MilkRecord provides the historical source data.

Example:

```text
MilkRecord:
18 L/day

       ↓

FeedRecommendation inputSnapshot:
milkYield = 18
```

The recommendation stores the value used at that time as a snapshot.

Therefore, later milk changes do not alter historical recommendations.

---

# 36. MilkRecord and WeightRecord

Together:

```text
MilkRecord
+
WeightRecord
```

can provide a richer production profile.

Example:

```text
Weight = 520 kg
Milk = 18 L/day
```

Later:

```text
Weight = 530 kg
Milk = 20 L/day
```

This allows trend analysis.

---

# 37. Daily Milk Production

Because each MilkRecord represents a session:

```text
Morning
Afternoon
Evening
Night
```

daily production can be calculated by aggregation.

Example:

```text
Morning = 8 L
Afternoon = 5 L
Evening = 7 L

Daily Total = 20 L
```

Conceptually:

```text
Σ quantity for same animal + same date
```

after normalizing units.

---

# 38. Do We Need `totalMilk` in MilkRecord?

No, not in the current finalized model.

The current design stores:

```text
quantity
+
milkingSession
```

Therefore daily total can be derived.

This is preferable because:

```text
Morning + Afternoon + Evening + Night
```

can be aggregated without storing a potentially duplicated daily total in every session record.

---

# 39. Monthly Milk Production

Conceptually:

```text
All MilkRecords
for animal/farm
within month
        ↓
Normalize units
        ↓
Sum quantity
        ↓
Monthly production
```

Example:

```text
January:
1,250 L

February:
1,310 L
```

---

# 40. Average Daily Milk

For a selected period:

```text
Total milk production
÷
Number of production days
```

The exact definition of "production days" should be selected carefully.

For example:

```text
Calendar days
```

and:

```text
Days with recorded production
```

produce different results.

The report should clearly state which one is used.

---

# 41. Average Session Yield

Example:

```text
Total morning milk
÷
Number of morning records
```

This can support:

```text
Morning vs Evening
```

comparison.

---

# 42. Milk Quality Trends

Because MilkRecord stores:

```text
fatPercentage
proteinPercentage
snfPercentage
quality
```

the system can generate:

```text
Fat trend
Protein trend
SNF trend
Quality trend
```

over time.

Example:

```text
Date      Fat
Aug 1     3.7%
Aug 8     3.8%
Aug 15    3.9%
```

---

# 43. Quality Rejected

If:

```text
quality = Rejected
```

the system should preserve the record rather than deleting it.

Possible business actions:

```text
Warning
Quality issue report
Milk sale exclusion
Manual review
```

The exact financial treatment belongs to the Income/Sales workflow.

---

# 44. Milk Sale Integration

The current Income model supports:

```text
incomeCategory = Milk Sale
```

and can reference:

```text
referenceType = MilkRecord
```

as shown in the finalized DB model. fileciteturn9file5

Therefore, a future flow can be:

```text
MilkRecord
    ↓
Milk sold
    ↓
Income
    ↓
Milk Sale Revenue
```

Important:

> Recording milk production does not automatically mean the milk was sold.

A MilkRecord and Income record should remain separate.

---

# 45. MilkRecord vs Income

### MilkRecord

```text
How much milk was produced?
```

### Income

```text
How much money was received from selling milk?
```

Example:

```text
Production:
20 L

Sold:
18 L

Consumed by farm:
2 L
```

Then:

```text
MilkRecord = 20 L
Income = sale of 18 L
```

This separation is very important.

---

# 46. Multiple Milk Records and Sales

A farm may aggregate milk from several animals before selling.

Therefore, one Income record may represent:

```text
Milk from multiple MilkRecords
```

The system should not assume:

```text
One MilkRecord = One Sale
```

unless the business workflow explicitly enforces it.

---

# 47. Milk Quantity and Income Calculation

If milk is sold by liter:

```text
Quantity sold × unit price
=
Income amount
```

Example:

```text
18 L × ৳70/L
=
৳1,260
```

These are illustrative values only.

The actual price should come from the farm's sales data.

---

# 48. Data Entry Validation

Before saving:

```text
Farm exists
Animal exists
Animal belongs to farm
Record date valid
Session valid
Quantity valid
Unit valid
Quality metrics valid
Milking method valid
RecordedBy authorized
```

Then:

```text
Save
+
Audit
```

---

# 49. Duplicate Record Prevention

Recommended uniqueness logic:

```text
farmId
+
animalId
+
recordDate
+
milkingSession
```

should identify one normal official session record.

If the same session is submitted twice:

```text
Reject duplicate
```

or:

```text
Ask user to edit existing record
```

rather than creating duplicate production.

---

# 50. Correcting MilkRecord

Example:

Original:

```text
Morning
8 L
```

Correction:

```text
Morning
8.5 L
```

The system should:

```text
Update record
+
Create AuditLog
```

Audit should preserve:

```text
oldData
newData
changedFields
```

according to the project's AuditLog design.

---

# 51. Backdated Milk Entry

A worker may record:

```text
August 15 milk
```

on:

```text
August 16
```

This can be allowed.

Store:

```text
recordDate = August 15
createdAt = August 16
```

AuditLog captures who entered it and when.

---

# 52. Future-Dated Milk Entry

Normally:

```text
recordDate > current date
```

should be rejected unless the system explicitly supports scheduled/planned production records.

MilkRecord is an actual production record, so future-dated entries generally do not make business sense.

---

# 53. Animal Status Validation

Recommended:

```text
Animal exists
      ↓
Animal is active
      ↓
Animal is not sold
      ↓
Animal is not dead
      ↓
Animal eligible for milk production
```

Then save.

---

# 54. Animal Transfer

If an animal is transferred:

```text
lifeStatus = Transferred
```

the old farm should normally stop receiving new MilkRecords after the transfer date.

Historical records remain associated with the farm where production was recorded.

The exact ownership-transfer workflow should determine how the new farm begins recording production.

---

# 55. Manual vs Machine Milking

`milkingMethod` supports operational analytics.

Example:

```text
Manual:
Average = 17.5 L/day

Milking Machine:
Average = 19.2 L/day
```

The system can compare production patterns by method.

However, correlation should not automatically be interpreted as causation.

---

# 56. Smart Device Future Integration

The finalized SensorData model is designed to support future metrics including:

```text
milkYield
```

among other sensor metrics. fileciteturn9file3

Future architecture could therefore support:

```text
Smart Device
      ↓
SensorData
      ↓
Milk Yield Data
      ↓
MilkRecord
```

However, the current MilkRecord should remain the authoritative production record unless the project explicitly defines automated synchronization.

---

# 57. Automated Sensor Data

If future smart devices provide:

```text
milkYield = 8.2 L
```

the system could create/import a MilkRecord with:

```text
milkingMethod = Automatic
```

or another appropriate representation.

But the current schema alone does not define the automatic synchronization algorithm.

---

# 58. Data Source / Provenance

The current MilkRecord model does not contain a dedicated:

```text
source
```

field.

Therefore, the current system can distinguish the milking method:

```text
Manual
Milking Machine
Automatic
```

but not necessarily the exact external device/source.

If future IoT integration requires detailed provenance, that should be handled through the SensorData/Device architecture rather than silently changing MilkRecord logic.

---

# 59. Audit Rules

Important MilkRecord audit events:

```text
MilkRecord Created
MilkRecord Updated
MilkRecord Corrected
MilkRecord Deleted/Voided
Quantity Changed
Quality Changed
Fat Changed
Protein Changed
SNF Changed
Milking Session Changed
Milking Method Changed
RecordedBy Changed
```

Because production records affect reports and potentially revenue, corrections should be auditable.

---

# 60. Notification Rules

Possible notifications:

```text
Milk production significantly below normal
Milk quality = Rejected
Missing expected milking record
Sudden production drop
Unusual production trend
```

These are future/derived notification rules, not mandatory schema behavior.

---

# 61. Sudden Milk Drop Detection

A future analytics service can compare:

```text
Today's milk
vs
Historical baseline
```

Example:

```text
Normal:
18–20 L/day

Today:
11 L/day
```

The system could generate:

```text
Warning:
Milk production is significantly below recent baseline.
```

This can be correlated with:

```text
HealthStatus
FeedRecord
WeightRecord
Pregnancy
SensorData
```

---

# 62. Feed-to-Milk Analysis

Suppose:

```text
Feed cost = ৳300/day
Milk production = 20 L/day
```

Then a report can show:

```text
Feed cost per liter
=
300 / 20
=
৳15/L
```

This is an analytical metric and should normally be calculated from historical records rather than stored directly in MilkRecord.

---

# 63. Production Dashboard

MilkRecord can power:

### Farm Dashboard

```text
Today's Total Milk
Average Daily Milk
Morning Milk
Evening Milk
Milk Quality
Top Producing Animals
```

### Animal Dashboard

```text
Today's Milk
7-Day Trend
30-Day Trend
Average Yield
Fat Trend
Protein Trend
SNF Trend
```

---

# 64. Production Report

A report can include:

```text
Animal
Date
Session
Quantity
Fat
Protein
SNF
Quality
Milking Method
Recorded By
```

This gives a complete production history.

---

# 65. Daily Production Aggregation

Conceptually:

```javascript
MilkRecord.aggregate([
    {
        $match: {
            farmId,
            recordDate: {
                $gte: startDate,
                $lte: endDate
            }
        }
    },
    {
        $group: {
            _id: "$recordDate",
            totalMilk: {
                $sum: "$quantity"
            }
        }
    }
])
```

If mixed units are allowed, quantities should first be normalized to a common unit.

---

# 66. Animal-wise Production Aggregation

Conceptually:

```text
Match:
farmId + animalId + date range

Group:
animalId

Sum:
quantity
```

This supports:

```text
Animal A:
520 L/month

Animal B:
610 L/month
```

---

# 67. Session-wise Production Aggregation

Group by:

```text
milkingSession
```

Example:

```text
Morning:
55%

Afternoon:
20%

Evening:
25%
```

This can reveal farm milking patterns.

---

# 68. Quality Report

The system can aggregate:

```text
Excellent
Good
Average
Poor
Rejected
```

Example:

```text
Excellent = 60%
Good = 30%
Average = 8%
Poor = 2%
Rejected = 0%
```

This supports farm quality monitoring.

---

# 69. Index Recommendations

These are implementation recommendations.

Useful indexes:

```javascript
farmId: 1
animalId: 1
recordDate: -1
milkingSession: 1
quality: 1
recordedBy: 1
```

Highly useful compound index:

```javascript
{
    farmId: 1,
    animalId: 1,
    recordDate: -1
}
```

For duplicate-session prevention, a unique compound index may be appropriate:

```javascript
{
    farmId: 1,
    animalId: 1,
    recordDate: 1,
    milkingSession: 1
}
```

provided the project does not intentionally allow multiple official records per session.

---

# 70. Important Note About Optional Quality Data

A farm may not have a milk analyzer every day.

Therefore, fields such as:

```text
fatPercentage
proteinPercentage
snfPercentage
```

may reasonably be optional if the current operational workflow does not always have laboratory/analyzer measurements.

The finalized schema includes them, but the project should not force users to enter fabricated values.

---

# 71. Missing Quality Data

Correct behavior:

```text
fatPercentage = null / absent
```

if not measured.

Incorrect behavior:

```text
fatPercentage = 0
```

just because no test was performed.

Zero means an actual measured value of zero, while missing means:

```text
not available / not measured
```

These are different states.

---

# 72. Missing Milk Record vs Zero Milk

Similarly:

```text
No MilkRecord
```

does not automatically mean:

```text
0 liter
```

For example:

```text
No entry
```

could mean:

```text
worker forgot to record
```

while:

```text
quantity = 0
```

could mean:

```text
animal produced no milk
```

The reporting layer should distinguish these cases.

---

# 73. Data Correction and Audit

Suppose:

```text
Original:
8 L

Corrected:
8.5 L
```

AuditLog should capture:

```text
oldData:
8 L

newData:
8.5 L

changedFields:
["quantity"]

user:
who made correction

createdAt:
when correction occurred
```

The current AuditLog architecture explicitly includes oldData, newData, changedFields, user, entity type/entity ID, status and timestamp. fileciteturn9file3

---

# 74. MilkRecord and Income Integrity

If a MilkRecord is used as the basis for a milk-sale Income record:

```text
MilkRecord
      ↓
Income.referenceType = MilkRecord
Income.referenceId = MilkRecord._id
```

The Income model in the finalized DB design supports this relationship. fileciteturn9file5

If the MilkRecord is later corrected, the system should check whether an associated sale/revenue record needs reconciliation.

It should not silently change financial history.

---

# 75. Example — One Day

Animal:

```text
AN-000125
```

### Morning

```text
recordDate = 2026-08-17
session = Morning
quantity = 8 L
fat = 3.8%
protein = 3.2%
SNF = 8.5%
quality = Good
method = Milking Machine
```

### Evening

```text
recordDate = 2026-08-17
session = Evening
quantity = 7 L
fat = 3.9%
protein = 3.3%
SNF = 8.6%
quality = Good
method = Milking Machine
```

Daily production:

```text
8 + 7 = 15 L
```

---

# 76. Example — Production Drop

Previous average:

```text
18 L/day
```

Current day:

```text
12 L/day
```

The production analytics layer detects:

```text
Significant drop
```

Then it may cross-check:

```text
FeedRecord
WeightRecord
Treatment
Vaccination
Pregnancy
SensorData
```

and generate a warning.

MilkRecord itself remains a factual production record; the interpretation belongs to analytics/alert logic.

---

# 77. Example — Rejected Milk

```text
quantity = 10 L
quality = Rejected
```

The record remains stored.

Possible workflow:

```text
MilkRecord
      ↓
Quality = Rejected
      ↓
Warning/Alert
      ↓
Exclude from sale calculation if business rules require
```

Do not automatically delete the production record.

---

# 78. Example — Machine Milking

```text
milkingMethod = Milking Machine
```

Later the farm changes to:

```text
Automatic
```

New records use:

```text
milkingMethod = Automatic
```

Old records retain:

```text
Milking Machine
```

This preserves operational history.

---

# 79. Example — Animal No Longer Lactating

Animal changes:

```text
reproductiveStatus = Dry
```

Then routine milk-record creation should normally be restricted or warned.

Historical MilkRecords remain unchanged.

When animal becomes:

```text
Lactating
```

new MilkRecords can resume.

---

# 80. CRUD Business Rules

## Create

Authorized farm users can create a milk-production record for an eligible animal.

Required:

```text
farmId
animalId
recordDate
milkingSession
quantity
unit
recordedBy
```

Quality information may depend on availability.

---

## Read

Authorized farm members can view milk history for animals belonging to their farm.

---

## Update

Corrections are allowed with appropriate permission and audit.

---

## Delete

Hard deletion should generally be avoided after the record has contributed to reports or revenue.

Use controlled correction/voiding logic where needed.

---

# 81. Validation Rules

| Field | Validation |
|---|---|
| farmId | Required, valid Farm |
| animalId | Required, valid Animal belonging to farm |
| recordDate | Required, valid date |
| milkingSession | Enum |
| quantity | Valid non-negative/positive production quantity |
| unit | Enum |
| fatPercentage | Non-negative if provided |
| proteinPercentage | Non-negative if provided |
| snfPercentage | Non-negative if provided |
| quality | Enum |
| milkingMethod | Enum |
| recordedBy | Required, authorized farm member/user |
| remarks | Optional |

---

# 82. Business Integrity Rules

```text
[✓] Animal must belong to the selected farm
[✓] RecordedBy must have permission for the farm
[✓] Milking session must be valid
[✓] Quantity must be valid
[✓] Unit must be valid
[✓] Quality must use the defined enum
[✓] Milking method must use the defined enum
[✓] Quality measurements must not be fabricated
[✓] Missing quality data ≠ zero
[✓] Missing MilkRecord ≠ zero milk
[✓] Duplicate animal/date/session records should normally be prevented
[✓] Historical records should remain auditable
[✓] Sold/dead animals should not normally receive new milk records
[✓] Historical records remain after animal lifecycle changes
[✓] Milk production and milk sales are separate concepts
```

---

# 83. Audit Events

Recommended AuditLog actions:

```text
Create
Update
Delete/Controlled Void
Export
```

Important changed fields:

```text
quantity
fatPercentage
proteinPercentage
snfPercentage
quality
milkingSession
milkingMethod
recordedBy
recordDate
```

---

# 84. Notification / Alert Opportunities

Future derived alerts:

```text
Sudden milk production drop
Poor milk quality
Rejected milk
Missing expected session
Unusual production trend
```

These can generate:

```text
Notification
```

with appropriate priority.

The current Notification model supports types such as:

```text
Reminder
Alert
Warning
Information
Success
```

and references such as Animal/Milk-related workflows can be extended as the final implementation requires. The exact current notification reference enum should remain the source of truth when implementing this integration.

---

# 85. Performance Considerations

Milk data can become large because:

```text
1 animal
×
multiple sessions/day
×
365 days
×
many animals
```

Therefore:

- Index by `farmId + animalId + recordDate`
- Use date-range queries
- Aggregate in MongoDB for dashboards
- Paginate historical lists
- Avoid loading an animal's complete milk history unnecessarily

---

# 86. Historical Data Strategy

MilkRecord should remain append-oriented.

Normal pattern:

```text
New production event
      ↓
Create new MilkRecord
```

Correction:

```text
Controlled update
+
AuditLog
```

Do not continuously rewrite old records just to maintain a current total.

Daily/monthly totals should be derived through aggregation.

---

# 87. Defense Questions

### Q: Why do you need MilkRecord if Animal already exists?

**Answer:**

> "Animal stores the animal's identity and current state, while MilkRecord stores time-specific milk production history. Since milk yield changes over time, production must be represented as historical records."

---

### Q: Why store milkingSession?

**Answer:**

> "Milk production can occur in multiple sessions per day. The session allows us to distinguish morning, afternoon, evening and night production and calculate daily and session-wise production accurately."

---

### Q: Why not store totalMilk directly?

**Answer:**

> "The finalized model stores session-level quantity. Daily total can be derived by aggregating the session records, which avoids duplicated totals and keeps the event data normalized."

---

### Q: Why store fat, protein and SNF?

**Answer:**

> "Milk volume alone does not describe milk quality. Fat, protein and SNF allow Liventra to monitor milk composition and quality trends."

---

### Q: What if the farmer cannot measure fat/protein every day?

**Answer:**

> "Those values should remain missing/null when not measured rather than being replaced with fabricated or zero values. The schema supports these quality measurements while allowing the operational workflow to determine when they are available."

---

### Q: What is the difference between MilkRecord and Income?

**Answer:**

> "MilkRecord represents milk production, while Income represents money received from selling milk. Produced milk does not necessarily equal sold milk."

---

### Q: Can one MilkRecord be linked to one Income?

**Answer:**

> "It can be referenced by an Income record when appropriate, but the system should not assume one-to-one mapping because milk from multiple animals may be aggregated before sale."

---

### Q: What happens if the animal is sold?

**Answer:**

> "Historical MilkRecords remain available, but new production records should normally be blocked after the animal's sale/transfer date according to the lifecycle rules."

---

### Q: How can MilkRecord help the feed recommendation system?

**Answer:**

> "MilkRecord provides historical production data. The relevant milk yield can be used as an input to generate a FeedRecommendation, and that value is then stored in the recommendation's inputSnapshot so the historical recommendation remains reproducible."

---

### Q: How can MilkRecord help the research/analytics part of Liventra?

**Answer:**

> "It allows us to analyze milk production trends and relate actual milk output to feeding, animal weight, health, reproduction and potentially smart-device data."

---

# 88. Final Business Rules Checklist

```text
[✓] MilkRecord belongs to a Farm
[✓] MilkRecord belongs to an Animal
[✓] MilkRecord represents an actual production event
[✓] recordDate represents production date
[✓] createdAt represents database creation time
[✓] milkingSession distinguishes production sessions
[✓] One normal official record per animal/date/session is recommended
[✓] quantity stores actual milk production
[✓] unit is liter/ml
[✓] Internal analytics should normalize units
[✓] fatPercentage stores milk fat when measured
[✓] proteinPercentage stores milk protein when measured
[✓] snfPercentage stores SNF when measured
[✓] quality stores production quality classification
[✓] milkingMethod stores operational milking method
[✓] recordedBy provides accountability
[✓] remarks stores additional observations
[✓] Daily milk can be derived from session records
[✓] Monthly milk can be derived through aggregation
[✓] Milk production is separate from milk sales
[✓] MilkRecord can support Income through reference
[✓] MilkRecord can support FeedRecommendation inputs
[✓] MilkRecord can be analyzed with FeedRecord
[✓] MilkRecord can be analyzed with WeightRecord
[✓] Historical records remain after animal lifecycle changes
[✓] Missing quality data must not be replaced by fake values
[✓] Missing record must not automatically mean zero production
[✓] Corrections should be audited
[✓] Historical records should not be casually deleted
```

---

# 89. Final Status

**Model:** `MilkRecord`  
**Status:** ✅ Finalized  
**Business Logic:** ✅ Documented  
**Production History:** ✅  
**Session-wise Tracking:** ✅  
**Quality Tracking:** ✅  
**Feed/Production Analytics Integration:** ✅  
**Historical Integrity:** ✅  

**Next Model:** `WeightRecord`
