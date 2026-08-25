# Liventra — Business Logic Documentation
## Model 11: WeightRecord

**Document Version:** Business Logic v1.0  
**Project:** Liventra — Smart Livestock Farm Management System  
**Database:** MongoDB  
**Architecture:** MERN  
**Module:** Production

---

# 1. Source / Finalized Model

The finalized `WeightRecord` model in the current Liventra DB design is:

```text
WeightRecord:
_id
farmId
animalId
recordedDate
weight
weightUnit: kg | lb
measurementMethod: Digital Scale | Manual Scale | Estimated | Smart Device
averageDailyGain
measuredBy
remarks
createdAt
updatedAt
```

The current finalized DB model explicitly defines these fields, including the four measurement methods and `averageDailyGain`. fileciteturn11file1

`WeightRecord` is part of Liventra's **Production Module**, alongside `MilkRecord`. fileciteturn11file14

---

# 2. Purpose

`WeightRecord` stores an animal's **weight measurement at a specific point in time**.

It answers:

> "What was this animal's weight on this date, how was it measured, and who measured/recorded it?"

Example:

```text
Animal:
AN-000125

Date:
2026-08-17

Weight:
180 kg

Measurement Method:
Digital Scale

Measured By:
Farm Member A
```

Later:

```text
2026-09-17
Weight = 205 kg
```

Both records remain in history.

---

# 3. Why WeightRecord Is Necessary

Weight is a changing attribute.

It should not be treated as a permanent property of the Animal document.

Conceptually:

```text
Animal
   |
   +--- WeightRecord 1 → 160 kg
   +--- WeightRecord 2 → 180 kg
   +--- WeightRecord 3 → 205 kg
   +--- WeightRecord 4 → 230 kg
```

The Animal model may keep:

```text
currentWeight
```

for fast access to the latest known weight, but `WeightRecord` is the historical source for weight measurements.

This distinction is extremely important.

---

# 4. Animal.currentWeight vs WeightRecord

The finalized Animal model contains:

```text
currentWeight
```

while the Production Module contains:

```text
WeightRecord
```

The conceptual relationship should be:

```text
WeightRecord
     ↓
Latest valid measurement
     ↓
Animal.currentWeight
```

Therefore:

### `WeightRecord`

Stores:

```text
Historical measurements
```

### `Animal.currentWeight`

Stores:

```text
Current/latest known weight for quick access
```

---

# 5. Why Not Calculate Everything from Animal.currentWeight?

Suppose:

```text
Animal.currentWeight = 230 kg
```

and we want to know:

```text
What was the animal's weight two months ago?
```

The Animal document cannot answer that.

But:

```text
WeightRecord
```

can.

Example:

```text
June 17 → 180 kg
July 17 → 205 kg
August 17 → 230 kg
```

This creates the animal's growth trajectory.

---

# 6. WeightRecord and FeedRecommendation

This is one of the most important relationships in Liventra.

FeedRecommendation can use:

```text
currentWeight
```

as an input.

The finalized FeedRecommendation stores:

```text
inputSnapshot.currentWeight
```

so the exact weight used for a historical recommendation remains preserved. fileciteturn11file1

Therefore:

```text
WeightRecord
      ↓
Current weight
      ↓
FeedRecommendation
      ↓
inputSnapshot.currentWeight
```

Example:

```text
WeightRecord:
180 kg

FeedRecommendation:
inputSnapshot.currentWeight = 180 kg
```

Later:

```text
WeightRecord:
230 kg
```

The old recommendation still says:

```text
180 kg
```

---

# 7. WeightRecord and FeedFormula

The finalized FeedFormula contains:

```text
targetWeightRange
targetADG
```

which can be used when selecting a suitable feeding formula. fileciteturn11file0

Therefore:

```text
WeightRecord
     ↓
Current weight
     ↓
Formula matching
     ↓
FeedRecommendation
```

This creates the connection between actual animal growth and feed planning.

---

# 8. WeightRecord and FeedRecord

WeightRecord tells us:

```text
How much the animal weighs
```

FeedRecord tells us:

```text
What the animal actually consumed
```

Together:

```text
FeedRecord
   ↓
Feed intake
   ↓
WeightRecord
   ↓
Weight change
```

This can later support:

```text
Feed efficiency
Growth performance
ADG analysis
```

---

# 9. WeightRecord and MilkRecord

For dairy animals:

```text
WeightRecord
+
MilkRecord
```

can provide a more complete production profile.

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

This can support production trend analysis.

---

# 10. Field-by-Field Business Logic

# 10.1 `_id`

### Type

MongoDB `ObjectId`

### Required

Yes

### Purpose

Unique identifier of the weight measurement record.

Example:

```text
ObjectId(...)
```

---

# 10.2 `farmId`

### Type

ObjectId / Farm reference

### Required

Yes

### Purpose

Identifies the farm where the measurement was recorded.

### Business Uses

- Farm-level weight reports
- Authorization
- Data isolation
- Analytics
- Audit

---

# 10.3 `animalId`

### Type

ObjectId / Animal reference

### Required

Yes

### Purpose

Identifies the animal whose weight was measured.

One animal can have many WeightRecords.

```text
Animal A
   |
   +--- WR1
   +--- WR2
   +--- WR3
   +--- WR4
```

---

# 10.4 Farm-Animal Integrity

The backend must verify:

```text
WeightRecord.farmId
        =
Animal.farmId
```

A user must not be able to create:

```text
farmId = Farm A
animalId = Animal from Farm B
```

The backend must reject this.

---

# 10.5 `recordedDate`

### Type

Date

### Required

Yes

### Purpose

Stores the date when the weight was actually measured/recorded.

Example:

```text
2026-08-17
```

---

# 10.6 recordedDate vs createdAt

These are different.

### `recordedDate`

```text
When was the animal actually weighed?
```

### `createdAt`

```text
When was the database record created?
```

Example:

```text
Animal weighed:
August 17

Data entered:
August 18
```

Then:

```text
recordedDate = August 17
createdAt = August 18
```

This supports delayed data entry.

---

# 10.7 `weight`

### Type

Number

### Required

Yes

### Example

```text
180
```

with:

```text
weightUnit = kg
```

means:

```text
180 kg
```

---

# 10.8 Weight Validation

Recommended:

```text
weight > 0
```

Negative weight is invalid.

The exact minimum/maximum range should be based on the project's supported livestock scope rather than an arbitrary hard-coded value.

---

# 10.9 `weightUnit`

### Type

Enum

### Allowed Values

```text
kg
lb
```

These are the finalized units in the current model. fileciteturn11file1

### Purpose

Defines the unit of the recorded weight.

---

# 11. Unit Normalization

For analytics, Liventra should preferably use one internal unit.

Recommended:

```text
kg
```

If the user records:

```text
220 lb
```

the system can normalize it internally to kg for:

```text
ADG
Weight trends
Feed recommendation
Formula matching
Reports
```

The original `weightUnit` should still be preserved so the record reflects what was entered/measured.

---

# 12. Why Unit Normalization Matters

Suppose:

```text
Record 1:
180 kg

Record 2:
400 lb
```

Direct subtraction is invalid unless both are converted to a common unit.

Therefore:

```text
Input Unit
    ↓
Normalize
    ↓
Common Internal Unit
    ↓
Calculate
```

---

# 13. `measurementMethod`

### Type

Enum

### Allowed Values

```text
Digital Scale
Manual Scale
Estimated
Smart Device
```

These are explicitly defined in the finalized model. fileciteturn11file1

### Purpose

Records how the weight measurement was obtained.

---

# 14. Why Measurement Method Matters

Not every measurement has the same level of reliability.

Example:

```text
Digital Scale
```

is an instrument-based measurement.

While:

```text
Estimated
```

may be less precise.

Therefore, Liventra can preserve measurement provenance.

---

# 15. Measurement Method — Digital Scale

Meaning:

```text
Weight measured using a digital weighing scale.
```

Example:

```text
weight = 180 kg
measurementMethod = Digital Scale
```

This is a direct measurement.

---

# 16. Measurement Method — Manual Scale

Meaning:

```text
Weight measured using a manual/mechanical scale.
```

Example:

```text
weight = 178 kg
measurementMethod = Manual Scale
```

---

# 17. Measurement Method — Estimated

Meaning:

```text
Weight was estimated rather than directly measured.
```

Example:

```text
weight = 180 kg
measurementMethod = Estimated
```

### Important Business Rule

An estimated measurement should not be treated as equally reliable as a direct scale measurement.

This becomes particularly important when:

```text
FeedRecommendation
```

depends on the weight.

The system can show a warning:

```text
Current weight is estimated.
```

---

# 18. Measurement Method — Smart Device

Meaning:

```text
Weight was obtained through an integrated smart-device system.
```

This is future-ready because Liventra's upcoming SensorData architecture supports smart livestock devices and sensor data. fileciteturn11file14

The current WeightRecord does not specify a particular device, so detailed device provenance should come from the future Device/SensorData architecture if required.

---

# 19. `averageDailyGain`

### Type

Number

### Required

Recommended: derived/optional

### Unit

kg/day after normalization

### Example

```text
0.8
```

meaning:

```text
0.8 kg/day
```

### Purpose

Represents the animal's Average Daily Gain (ADG).

---

# 20. What Is ADG?

Conceptually:

```text
ADG =
(weight difference)
÷
(time difference)
```

Example:

```text
Previous weight = 180 kg
Current weight = 200 kg

Time = 25 days

ADG =
(200 - 180) / 25
=
0.8 kg/day
```

This is an illustrative calculation.

---

# 21. Important Question — Should ADG Be Stored?

The finalized model includes:

```text
averageDailyGain
```

so the project can preserve the calculated/recorded ADG.

However, business-wise, ADG should preferably be **derived from weight history** rather than manually entered arbitrarily.

Recommended logic:

```text
Current WeightRecord
        ↓
Find previous valid WeightRecord
        ↓
Normalize both weights to kg
        ↓
Calculate date difference
        ↓
Calculate ADG
        ↓
Store/display averageDailyGain
```

This reduces human error.

---

# 22. ADG Calculation Rule

Suppose:

```text
Previous:
180 kg
Date:
2026-08-01

Current:
200 kg
Date:
2026-08-26
```

Then:

```text
Weight gain = 20 kg
Days = 25

ADG = 20 / 25
    = 0.8 kg/day
```

---

# 23. Negative ADG

Suppose:

```text
Previous = 200 kg
Current = 190 kg
```

Then:

```text
ADG = negative
```

This is not necessarily a database error.

It may indicate:

```text
Weight loss
```

which can be clinically or operationally important.

Therefore, **do not automatically reject negative ADG** if the project wants to monitor weight loss.

Instead, the system can generate a warning.

---

# 24. Zero ADG

If:

```text
Previous = 180 kg
Current = 180 kg
```

then:

```text
ADG = 0 kg/day
```

This is valid.

It means no net weight change during the selected interval.

---

# 25. First WeightRecord

Suppose this is the animal's first measurement:

```text
Weight = 180 kg
```

There is no previous weight.

Therefore:

```text
averageDailyGain = null
```

or omitted.

Do not calculate ADG using an imaginary previous value.

---

# 26. Choosing the Previous WeightRecord

For ADG calculation, the system should generally use:

```text
Latest valid previous WeightRecord
```

before the current measurement date.

Example:

```text
Aug 1 → 180 kg
Aug 10 → 187 kg
Aug 20 → 195 kg
```

When calculating Aug 20:

```text
Previous = Aug 10
```

not Aug 1.

---

# 27. Should Estimated Weight Be Used for ADG?

The project should distinguish measurement reliability.

A recommended approach:

```text
Digital Scale
Manual Scale
Smart Device
```

can be treated as direct/instrument-based measurements.

```text
Estimated
```

should either:

```text
be excluded from high-confidence ADG calculation
```

or:

```text
calculate ADG but mark it as estimated
```

The exact policy should be chosen by the project's evaluation methodology.

The important point is:

> Estimated measurements should not silently appear as equally reliable measurements.

---

# 28. `measuredBy`

### Type

ObjectId / User or FarmMember reference

### Required

Yes

### Purpose

Identifies who performed the weight measurement.

Example:

```text
measuredBy → FarmMember A
```

---

# 29. measuredBy vs AuditLog User

These can be different.

Example:

```text
Worker A
→ physically measured animal

Manager B
→ entered/corrected the record
```

Then:

```text
measuredBy = Worker A
AuditLog.userId = Manager B
```

This distinction preserves operational accountability.

---

# 30. `remarks`

### Type

String

### Required

Optional

### Example

```text
Animal was weighed before morning feeding.
```

Other examples:

```text
Animal was restless during measurement.
Scale showed unstable reading.
Estimated due to lack of scale.
```

---

# 31. `createdAt`

### Type

Date

### Required

Yes

### Purpose

Stores when the WeightRecord document was created.

---

# 32. `updatedAt`

### Type

Date

### Required

Yes

### Purpose

Stores when the record was last modified.

Historical weight records should not be casually edited without audit tracking.

---

# 33. Weight Recording Workflow

```text
Select Animal
      ↓
Verify Farm
      ↓
Measure Animal
      ↓
Enter Weight
      ↓
Select Unit
      ↓
Select Measurement Method
      ↓
Identify Measured By
      ↓
Load Previous Weight
      ↓
Normalize Units
      ↓
Calculate ADG
      ↓
Check abnormal change
      ↓
Save WeightRecord
      ↓
Update Animal.currentWeight
      ↓
AuditLog
```

---

# 34. Updating Animal.currentWeight

After a valid new WeightRecord:

```text
WeightRecord
      ↓
Is this the latest valid measurement?
      ↓
Yes
      ↓
Update Animal.currentWeight
```

Example:

```text
Old:
Animal.currentWeight = 180 kg

New WeightRecord:
200 kg

Update:
Animal.currentWeight = 200 kg
```

---

# 35. Important Rule — Do Not Update Current Weight with an Old Record

Suppose:

```text
Latest:
Aug 20 = 200 kg
```

Then a user enters a delayed record:

```text
Aug 10 = 190 kg
```

The system must NOT do:

```text
Animal.currentWeight = 190 kg
```

because Aug 20 is newer.

Instead:

```text
Historical WeightRecord:
190 kg

Animal.currentWeight:
200 kg
```

This is a critical business rule.

---

# 36. Backdated Weight Entry

Backdated entries should be allowed if the farm needs delayed data entry.

Example:

```text
Measured:
Aug 15
Entered:
Aug 17
```

Store:

```text
recordedDate = Aug 15
createdAt = Aug 17
```

Then recalculate any affected derived analytics if necessary.

---

# 37. Backdated Record and ADG

Suppose:

```text
Aug 1 = 180 kg
Aug 10 = 190 kg
Aug 20 = 210 kg
```

Later a new record is inserted:

```text
Aug 15 = 200 kg
```

The historical sequence becomes:

```text
Aug 1 → 180
Aug 10 → 190
Aug 15 → 200
Aug 20 → 210
```

ADG calculations based on neighboring measurements may need recalculation.

This is one reason derived values such as ADG should ideally be recalculated by the service layer rather than manually trusted.

---

# 38. WeightRecord and FeedRecommendation Snapshot

Example:

```text
WeightRecord:
180 kg
```

Recommendation:

```text
inputSnapshot.currentWeight = 180 kg
```

Later:

```text
WeightRecord:
230 kg
```

New recommendation:

```text
inputSnapshot.currentWeight = 230 kg
```

Historical recommendation remains unchanged.

This is exactly the snapshot principle used throughout Liventra's recommendation system.

---

# 39. WeightRecord and FeedFormula Matching

Suppose a FeedFormula is designed for:

```text
targetWeightRange:
150–200 kg
```

Current animal weight:

```text
180 kg
```

The formula can qualify based on weight.

Later:

```text
230 kg
```

The old formula may no longer be appropriate.

The recommendation engine can select another suitable formula.

---

# 40. WeightRecord and Target ADG

FeedFormula contains:

```text
targetADG
```

while WeightRecord can contain:

```text
averageDailyGain
```

These represent two different things.

### Target ADG

```text
What growth rate do we want?
```

### Actual ADG

```text
What growth rate did the animal achieve?
```

Example:

```text
Target ADG = 0.8 kg/day

Actual ADG = 0.6 kg/day
```

Then:

```text
Actual is below target
```

This can trigger analysis.

---

# 41. Target vs Actual ADG

Conceptually:

```text
Target ADG
     ↓
FeedFormula / FeedRecommendation

Actual ADG
     ↓
WeightRecord
```

Then:

```text
Performance Gap
=
Actual ADG - Target ADG
```

Example:

```text
0.6 - 0.8
=
-0.2 kg/day
```

This is an important production KPI.

---

# 42. FeedRecord + WeightRecord + ADG

A future analytics workflow:

```text
FeedRecord
     ↓
Actual feed intake
     ↓
WeightRecord
     ↓
Actual ADG
     ↓
Compare against target ADG
```

This can help evaluate:

```text
Was the feeding plan effective?
```

However, causality should not be claimed simply from these two records; other factors such as health, breed, environment and measurement quality matter.

---

# 43. WeightRecord + Health Module

A sudden weight decline may be associated with:

```text
Disease
Treatment
Poor feed intake
Pregnancy
Other health conditions
```

The analytics layer can compare WeightRecord with:

```text
Treatment
Disease
FeedRecord
SensorData
```

and generate warnings.

WeightRecord itself remains a factual measurement.

---

# 44. WeightRecord + Reproduction

Weight is relevant to:

```text
Breeding
Pregnancy
Calving
Growth
```

For example:

```text
Heifer weight trend
```

can be viewed alongside breeding events.

This helps farm managers understand whether an animal is progressing toward intended production/reproductive stages.

---

# 45. WeightRecord + Smart Devices

The finalized model explicitly includes:

```text
measurementMethod = Smart Device
```

The future SensorData model supports metrics from devices and includes:

```text
bodyTemperature
activityScore
ruminationMinutes
feedIntake
milkYield
...
```

and identifies:

```text
deviceId
assignmentId
animalId
recordedAt
```

This architecture is designed for future smart livestock integration. fileciteturn11file14

If a smart device provides weight, the application can create/synchronize a WeightRecord.

---

# 46. Smart Device Provenance

The current WeightRecord does not have:

```text
deviceId
```

That is acceptable under the current finalized model.

If future implementation requires:

```text
Which exact smart device produced this weight?
```

the Device/SensorData architecture should provide that provenance.

Do not duplicate IoT-specific fields into WeightRecord unless the project later determines that it is necessary.

---

# 47. Measurement Quality

A useful conceptual hierarchy is:

```text
Digital Scale
Smart Device
Manual Scale
Estimated
```

However, the exact reliability ranking should not be hard-coded without project validation.

The important business principle is:

```text
Measurement method
=
data provenance
```

and analytics can use it to assess confidence.

---

# 48. Abnormal Weight Change

Suppose:

```text
Previous = 220 kg
Current = 180 kg
```

A huge decrease should trigger:

```text
Warning:
Significant weight change detected.
```

The threshold should be configurable rather than arbitrarily fixed.

Possible checks:

```text
absolute change
percentage change
ADG
```

Example:

```text
Percentage change
=
(current - previous) / previous × 100
```

---

# 49. Data Entry Error Detection

Suppose:

```text
Previous = 180 kg
New = 1800 kg
```

The system should detect a potentially impossible/unusual change and ask for confirmation.

This is especially useful for:

```text
manual entry
```

---

# 50. Duplicate Weight Records

Unlike MilkRecord, multiple WeightRecords on the same day may be valid.

Example:

```text
Morning:
180 kg

Evening:
181 kg
```

However, if the farm's workflow is:

```text
one official weight measurement per animal per day
```

then duplicate prevention can be applied.

The finalized model does not define a unique session field for weight, so the system should not blindly enforce one record/day without considering the farm's operational requirements.

---

# 51. Recommended Weight Recording Policy

For the current project:

```text
Multiple historical WeightRecords:
Allowed

Duplicate accidental submissions:
Detect and warn

Multiple legitimate measurements:
Allowed if farm workflow permits

Latest valid measurement:
Used for Animal.currentWeight
```

---

# 52. Weight Record Correction

Suppose:

```text
Recorded:
180 kg

Correct:
185 kg
```

The system should:

```text
Update WeightRecord
+
AuditLog
+
Recalculate affected ADG
+
Update Animal.currentWeight if this is the latest valid record
```

---

# 53. Why ADG May Need Recalculation After Correction

Suppose:

```text
Previous = 180 kg
Current = 200 kg
```

ADG:

```text
0.8 kg/day
```

If current weight is corrected to:

```text
195 kg
```

then ADG changes.

Therefore:

```text
Weight correction
      ↓
ADG recalculation
```

is necessary.

---

# 54. Historical Integrity

WeightRecord should be treated as historical production data.

Changes should be:

```text
controlled
+
audited
```

Do not silently rewrite old measurements without preserving the change history.

The finalized AuditLog model includes `WeightRecord` as a supported auditable entity. fileciteturn11file14

---

# 55. CRUD Business Rules

## Create

Authorized farm members can create a WeightRecord for an eligible animal.

Required:

```text
farmId
animalId
recordedDate
weight
weightUnit
measurementMethod
measuredBy
```

---

## Read

Authorized users can view:

```text
Animal weight history
Farm weight reports
Growth trends
```

---

## Update

Allowed with permission.

Changes must be audited.

Derived values such as ADG should be recalculated.

---

## Delete

Hard deletion should generally be avoided for historical measurements.

Use controlled correction/void logic if needed.

---

# 56. Validation Rules

| Field | Validation |
|---|---|
| farmId | Required, valid Farm |
| animalId | Required, valid Animal belonging to farm |
| recordedDate | Required, valid date |
| weight | Required, positive numeric value |
| weightUnit | `kg` or `lb` |
| measurementMethod | Defined enum |
| averageDailyGain | Derived/validated numeric value |
| measuredBy | Required, authorized farm member/user |
| remarks | Optional |
| createdAt | Automatic |
| updatedAt | Automatic |

---

# 57. Important Integrity Rules

```text
[✓] Animal must belong to the selected farm
[✓] measuredBy must be authorized
[✓] Weight must be valid and positive
[✓] Unit must be valid
[✓] Internal analytics should normalize weight to kg
[✓] Measurement method must be preserved
[✓] Estimated measurements should be distinguishable
[✓] First measurement has no meaningful ADG
[✓] ADG should preferably be derived from historical measurements
[✓] Negative ADG can represent real weight loss
[✓] Old backdated records must not replace latest currentWeight
[✓] Latest valid WeightRecord should drive Animal.currentWeight
[✓] Corrections should trigger ADG/current-weight reconciliation
[✓] Historical measurements should remain auditable
```

---

# 58. Audit Rules

Important AuditLog actions:

```text
WeightRecord Created
WeightRecord Updated
WeightRecord Corrected
WeightRecord Deleted/Voided
Weight Changed
Measurement Method Changed
MeasuredBy Changed
RecordedDate Changed
```

The audit record should preserve:

```text
oldData
newData
changedFields
user
timestamp
requestId
status
```

The finalized AuditLog structure explicitly supports old/new data, changed fields, user, request ID and success/failure/rollback status. fileciteturn11file14

---

# 59. Notification / Alert Opportunities

WeightRecord can trigger derived alerts such as:

```text
Significant weight loss
Low ADG
ADG below target
Unexpected weight jump
No recent weight measurement
Growth milestone reached
```

These are application/analytics rules, not mandatory fields of WeightRecord.

---

# 60. No Recent Weight Measurement

Suppose:

```text
Last WeightRecord:
90 days ago
```

The dashboard can show:

```text
Warning:
No recent weight measurement available.
```

This can also affect feed recommendation reliability.

If the current weight is stale, the recommendation engine may warn:

```text
Current weight data may be outdated.
```

---

# 61. Weight Freshness

A useful concept:

```text
Current weight
+
measurement date
```

determines whether weight data is fresh enough for a recommendation.

Example:

```text
Current weight = 180 kg
Recorded = 120 days ago
```

The system should be cautious about treating this as current.

The exact freshness threshold should be configurable.

---

# 62. Weight Recommendation Dependency

FeedRecommendation can use:

```text
currentWeight
```

but the system should ideally identify:

```text
Which WeightRecord supplied this current weight?
```

The current FeedRecommendation model does this through the value snapshot, not through a dedicated `weightRecordId`.

Therefore, the historical recommendation preserves:

```text
currentWeight = 180 kg
```

while the broader system can trace the likely source through the animal's weight history.

---

# 63. Example — Growth History

```text
Animal:
AN-000125
```

Weight records:

```text
2026-06-01 → 150 kg
2026-07-01 → 175 kg
2026-08-01 → 198 kg
2026-09-01 → 220 kg
```

Growth chart:

```text
220 |                         ●
    |
200 |                   ●
    |
180 |             ●
    |
160 |       ●
    |
    +----------------------------
       Jun    Jul    Aug    Sep
```

The frontend can visualize this from WeightRecord history.

---

# 64. Example — ADG History

```text
June → July:
25 kg / 30 days = 0.83 kg/day

July → August:
23 kg / 31 days = 0.74 kg/day

August → September:
22 kg / 31 days = 0.71 kg/day
```

This indicates a decreasing growth rate.

The analytics layer could flag:

```text
ADG trend declining
```

and investigate:

```text
Feed
Health
Season
Reproduction
```

---

# 65. Example — Target vs Actual

Formula:

```text
targetADG = 0.8 kg/day
```

Actual:

```text
averageDailyGain = 0.6 kg/day
```

Performance gap:

```text
0.6 - 0.8
=
-0.2 kg/day
```

Potential investigation:

```text
FeedRecord
Health
Disease/Treatment
FeedRecommendation
Season
```

---

# 66. Example — Estimated Weight

```text
weight = 185 kg
weightUnit = kg
measurementMethod = Estimated
```

The dashboard may show:

```text
⚠ Estimated weight
```

The recommendation engine can include:

```text
Warning:
Current weight is estimated; recommendation confidence may be reduced.
```

The exact wording/behavior can be configured.

---

# 67. Example — Smart Device Weight

```text
weight = 205 kg
weightUnit = kg
measurementMethod = Smart Device
```

This creates a bridge to the future IoT architecture.

Possible future flow:

```text
Smart Device
      ↓
Sensor/API
      ↓
SensorData
      ↓
Weight extraction
      ↓
WeightRecord
      ↓
Animal.currentWeight
```

---

# 68. Example — Backdated Record

Current:

```text
Aug 20 = 200 kg
Animal.currentWeight = 200 kg
```

Later entered:

```text
Aug 10 = 185 kg
```

Correct result:

```text
WeightRecord history:
Aug 10 = 185
Aug 20 = 200

Animal.currentWeight:
200 kg
```

The old record must not replace the newer current state.

---

# 69. Example — Correction of Latest Record

Current:

```text
Aug 20 = 200 kg
Animal.currentWeight = 200 kg
```

Correction:

```text
Aug 20 = 195 kg
```

Then:

```text
WeightRecord = 195 kg
Animal.currentWeight = 195 kg
```

and:

```text
AuditLog:
old = 200
new = 195
```

ADG should be recalculated.

---

# 70. Performance / Index Recommendations

Useful indexes:

```javascript
farmId: 1
animalId: 1
recordedDate: -1
measurementMethod: 1
measuredBy: 1
```

Highly useful compound index:

```javascript
{
    farmId: 1,
    animalId: 1,
    recordedDate: -1
}
```

This supports:

```text
Get latest weight records for an animal
```

and:

```text
Get weight history for date range
```

---

# 71. Latest Weight Query

Conceptually:

```javascript
WeightRecord.findOne({
    farmId,
    animalId
}).sort({
    recordedDate: -1,
    createdAt: -1
});
```

The exact query should additionally account for the project's validity/void rules.

---

# 72. Growth Report

A Weight Report can include:

```text
Animal
First Weight
Latest Weight
Total Weight Gain
Average ADG
Target ADG
Performance Gap
Measurement Method
Last Measurement Date
```

This can become one of Liventra's useful production reports.

---

# 73. Production Dashboard

Animal dashboard:

```text
Current Weight
Previous Weight
Weight Gain
Current ADG
Target ADG
Growth Trend
Last Measurement Date
Measurement Method
```

Farm dashboard:

```text
Average Herd Weight
Average ADG
Fastest Growing Animals
Animals With Weight Loss
Animals With Stale Weight Data
```

---

# 74. Weight and Animal Lifecycle

For:

```text
Active
```

animals:

```text
normal weight recording
```

For:

```text
Sold
Dead
Transferred
Archived
```

new measurements should normally be blocked according to lifecycle rules.

Historical WeightRecords remain preserved.

---

# 75. Weight and Reproductive Stage

The Animal model includes:

```text
Calf
Heifer
Open
Pregnant
Lactating
Dry
```

as reproductive-status values.

Weight trends can therefore be analyzed by stage.

Example:

```text
Heifer:
Growth trend

Pregnant:
Weight progression

Lactating:
Weight vs milk production
```

The WeightRecord itself remains a factual measurement.

---

# 76. Weight and Breed

Because Animal references Breed:

```text
Animal
 ↓
Breed
```

and WeightRecord references Animal:

```text
WeightRecord
 ↓
Animal
 ↓
Breed
```

the system can compare:

```text
Average weight by breed
Growth rate by breed
Weight at different stages
```

This can support farm analytics and research evaluation.

---

# 77. WeightRecord and Reports

The finalized Report model supports:

```text
reportType = Weight
```

so WeightRecord can directly power generated weight reports. fileciteturn11file16

Possible report:

```text
Weight Growth Report
```

containing:

```text
Date
Weight
ADG
Measurement Method
Measured By
```

---

# 78. Historical Data Strategy

WeightRecord should primarily be treated as an append-oriented historical dataset.

Normal:

```text
New measurement
    ↓
New WeightRecord
```

Correction:

```text
Controlled update
+
AuditLog
```

Avoid storing only the latest value and deleting previous measurements.

---

# 79. Defense Questions

### Q: Why do you need WeightRecord if Animal already has currentWeight?

**Answer:**

> "Animal.currentWeight stores the latest known value for fast access, while WeightRecord preserves the complete historical weight measurements needed for growth analysis, ADG calculation and historical recommendations."

---

### Q: Why store measurementMethod?

**Answer:**

> "Because weight can come from a digital scale, manual scale, estimate or future smart device. Recording the measurement method preserves data provenance and helps assess measurement reliability."

---

### Q: Why do you have averageDailyGain?

**Answer:**

> "ADG represents the animal's actual growth rate between weight measurements. It can be compared with the target ADG defined in the feeding formula or recommendation."

---

### Q: How do you calculate ADG?

**Answer:**

> "ADG is calculated as weight difference divided by the number of days between two valid weight measurements, after normalizing the weight units."

---

### Q: What happens for the first weight record?

**Answer:**

> "There is no previous measurement, so a meaningful ADG cannot be calculated. It should remain null or unavailable."

---

### Q: Can ADG be negative?

**Answer:**

> "Yes. A negative ADG can represent actual weight loss. It should generally trigger investigation or a warning rather than being automatically rejected."

---

### Q: What if an old weight record is entered later?

**Answer:**

> "The historical WeightRecord is inserted with its actual recordedDate, but it must not replace Animal.currentWeight if a newer valid measurement already exists."

---

### Q: Why not calculate weight history from Animal.currentWeight?

**Answer:**

> "Because currentWeight contains only the latest state. Historical WeightRecords are necessary to reconstruct the animal's growth trajectory."

---

### Q: How does WeightRecord support feed recommendation?

**Answer:**

> "The latest valid weight can be used to select an appropriate FeedFormula and generate a FeedRecommendation. The recommendation then stores that weight in its inputSnapshot so the historical recommendation remains reproducible."

---

### Q: What is the difference between targetADG and averageDailyGain?

**Answer:**

> "targetADG is the desired growth rate defined by the feeding rule, while averageDailyGain is the animal's observed growth rate calculated from its weight history."

---

# 80. Final Business Rules Checklist

```text
[✓] WeightRecord belongs to a Farm
[✓] WeightRecord belongs to an Animal
[✓] WeightRecord stores historical measurements
[✓] recordedDate is measurement date
[✓] createdAt is database creation time
[✓] weight must be valid and positive
[✓] kg/lb are supported input units
[✓] Analytics should normalize weight to kg
[✓] Measurement method is preserved
[✓] Estimated weight is distinguishable
[✓] Smart Device measurement is future-ready
[✓] ADG represents actual growth rate
[✓] ADG should preferably be derived from weight history
[✓] First measurement has no ADG
[✓] Negative ADG can represent real weight loss
[✓] Latest valid WeightRecord can update Animal.currentWeight
[✓] Backdated records must not overwrite newer currentWeight
[✓] Weight corrections require ADG reconciliation
[✓] Weight corrections should be audited
[✓] Historical records should not be silently deleted
[✓] WeightRecord supports FeedRecommendation
[✓] WeightRecord supports FeedFormula matching
[✓] WeightRecord supports FeedRecord analysis
[✓] WeightRecord supports MilkRecord analysis
[✓] WeightRecord supports health/reproduction analytics
[✓] WeightRecord supports future Smart Device integration
```

---

# 81. Final Status

**Model:** `WeightRecord`  
**Status:** ✅ Finalized  
**Business Logic:** ✅ Documented  
**Historical Weight Tracking:** ✅  
**ADG Tracking:** ✅  
**Measurement Provenance:** ✅  
**Feed Recommendation Integration:** ✅  
**Smart Device Readiness:** ✅  
**Auditability:** ✅  

**Next Model:** `Vaccine`
