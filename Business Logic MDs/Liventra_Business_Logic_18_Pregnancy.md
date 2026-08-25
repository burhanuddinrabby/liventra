# Liventra — Business Logic Documentation
## Model 18: Pregnancy

**Document Version:** Business Logic v1.0  
**Project:** Liventra — Smart Livestock Farm Management System  
**Database:** MongoDB  
**Architecture:** MERN  
**Module:** Reproduction Management

---

# 1. Source / Finalized Model

The current finalized Liventra DB model defines `Pregnancy` as:

```text
Pregnancy Model:
_id
pregnancyNo
farmId
animalId
breedingId
conceptionDate
expectedCalvingDate
actualCalvingDate
calfCount
calfGender: Male | Female | Twin
pregnancyStatus: Pregnant | Delivered | Aborted | Failed
remarks
createdBy
createdAt
updatedAt
```

This is the Pregnancy structure in the finalized DB model source. fileciteturn16file0turn16file1

> **Important:** This document follows the finalized model exactly. It does not silently add fields such as `pregnancyCheckDate`, `abortionDate`, `deliveryType`, or `calfIds[]`, because those are not present in the current model.

---

# 2. Purpose

`Pregnancy` represents the **pregnancy lifecycle of a specific animal after a breeding event**.

It answers:

> "Which animal became pregnant from which breeding event, when was conception recorded, when is calving expected, when did calving actually happen, how many calves resulted, what was their gender classification, and what is the current pregnancy status?"

Example:

```text
Animal:
AN-000125

Breeding:
BRD-000015

Conception:
2026-08-18

Expected Calving:
2027-05-17

Status:
Pregnant
```

After delivery:

```text
Actual Calving:
2027-05-16

Calf Count:
1

Calf Gender:
Female

Status:
Delivered
```

---

# 3. Pregnancy vs Breeding

This distinction is fundamental.

## Breeding

Records the **attempt/event**:

```text
Heat
Insemination / Mating
Method
Bull
Semen Batch
Technician
Pregnancy Check
```

## Pregnancy

Records the **resulting pregnancy lifecycle**:

```text
Conception
Expected Calving
Actual Calving
Calf Outcome
Pregnancy Status
```

Therefore:

```text
Breeding
    ↓
Pregnancy
```

The finalized `Breeding` model contains `pregnancyResult`, while `Pregnancy` contains `breedingId`. fileciteturn16file0

---

# 4. Overall Reproductive Architecture

The complete flow is:

```text
Female Animal
      ↓
Heat Detection
      ↓
Breeding
      ↓
Pregnancy Check
      ↓
Breeding.pregnancyResult
      ↓
Confirmed
      ↓
Pregnancy
      ↓
Pregnant
      ↓
Expected Calving
      ↓
Actual Calving
      ↓
Delivered
      ↓
Calf / New Animal Records
      ↓
Lactation
      ↓
MilkRecord
```

If the breeding fails:

```text
Breeding
   ↓
Failed
   ↓
No confirmed Pregnancy
   ↓
Animal may return to Open
```

---

# 5. Pregnancy Is an Event + Lifecycle Record

Unlike a simple boolean:

```text
isPregnant = true
```

the Pregnancy collection preserves historical reproductive information.

For example:

```text
Pregnancy #1
2026
Delivered

Pregnancy #2
2028
Aborted
```

Both events can remain in history.

This is much more useful than keeping only the current state on Animal.

---

# 6. Pregnancy vs Animal.reproductiveStatus

The Animal model contains:

```text
reproductiveStatus:
Calf | Heifer | Open | Pregnant | Lactating | Dry
```

That represents the **current state**.

Pregnancy represents a **specific pregnancy episode**.

Therefore:

```text
Animal.reproductiveStatus
=
Current state

Pregnancy
=
Historical/active pregnancy episode
```

Example:

```text
Animal:
reproductiveStatus = Pregnant

Pregnancy:
pregnancyNo = PREG-000015
pregnancyStatus = Pregnant
```

---

# 7. Field-by-Field Business Logic

# 7.1 `_id`

### Type

MongoDB `ObjectId`

### Required

Yes

### Purpose

Unique database identifier for the Pregnancy record.

Example:

```text
ObjectId(...)
```

---

# 7.2 `pregnancyNo`

### Type

String

### Required

Yes

### Example

```text
PREG-000001
```

### Purpose

Human-readable unique identifier for a pregnancy episode.

Used in:

```text
Pregnancy History
Reports
Notifications
AuditLog
Search
Animal reproductive history
```

---

# 8. Pregnancy Number Generation

Recommended format:

```text
PREG-{SEQUENCE}
```

Examples:

```text
PREG-000001
PREG-000002
PREG-000003
```

The backend should generate and validate uniqueness.

The frontend should not be trusted to generate unique identifiers.

---

# 9. `farmId`

### Type

ObjectId / Farm reference

### Required

Yes

### Purpose

Identifies the farm to which the Pregnancy record belongs.

This is required for multi-farm data isolation.

---

# 10. Farm Isolation Rule

Before creating Pregnancy:

```text
Pregnancy.farmId
       ↓
must match
       ↓
Animal.farmId
```

Also:

```text
Breeding.farmId
       ↓
must match
       ↓
Pregnancy.farmId
```

Example:

```text
Pregnancy.farmId = Farm A
Animal.farmId = Farm B
```

must be rejected.

The SRS requires that users cannot access another farm's data. fileciteturn16file18

---

# 11. `animalId`

### Type

ObjectId / Animal reference

### Required

Yes

### Purpose

Identifies the pregnant animal.

The animal should normally be:

```text
Female
```

because the current Pregnancy model represents the pregnancy carried by the animal.

---

# 12. Animal Validation

Before creating Pregnancy:

```text
Animal exists
+
Animal belongs to farmId
+
Animal.gender = Female
```

should be verified.

The animal should also normally have:

```text
lifeStatus = Active
isActive = true
```

for an active pregnancy.

---

# 13. `breedingId`

### Type

ObjectId / Breeding reference

### Required

Yes

### Purpose

Identifies the Breeding event from which this pregnancy originated.

This is the most important relationship in the Pregnancy model.

```text
Breeding._id
       ↓
Pregnancy.breedingId
```

The finalized model explicitly contains this relationship. fileciteturn16file0

---

# 14. Why `breedingId` Is Important

Without `breedingId`, Liventra could know:

```text
Animal is pregnant
```

but not:

```text
Which breeding attempt caused this pregnancy?
```

With `breedingId`, the system can trace:

```text
Pregnancy
   ↓
Breeding
   ↓
Method
   ↓
Bull
   ↓
Semen Batch
   ↓
Technician
```

This provides reproductive traceability.

---

# 15. Breeding Integrity Rule

Before creating Pregnancy:

```text
Breeding exists
```

and:

```text
Breeding.animalId = Pregnancy.animalId
```

and:

```text
Breeding.farmId = Pregnancy.farmId
```

must all match.

Otherwise:

```text
Reject
```

---

# 16. Confirmed Breeding Requirement

A normal Pregnancy should only be created when:

```text
Breeding.pregnancyResult = Confirmed
```

The intended flow is:

```text
Breeding
   ↓
Pregnancy Check
   ↓
Confirmed
   ↓
Create Pregnancy
```

If:

```text
Breeding.pregnancyResult = Failed
```

a confirmed Pregnancy should not be created from that breeding.

---

# 17. One Breeding → One Pregnancy

A single breeding event should normally produce:

```text
0 or 1 Pregnancy
```

Therefore:

```text
breedingId
```

should be unique in the Pregnancy collection.

Conceptually:

```text
One Breeding
      ↓
at most one Pregnancy
```

This prevents accidental duplicate pregnancy records.

---

# 18. `conceptionDate`

### Type

Date

### Required

Yes / recommended

### Purpose

Records the date associated with conception.

Example:

```text
2026-08-18
```

---

# 19. Conception Date vs Insemination Date

Do not automatically assume these are always identical.

Breeding has:

```text
inseminationDate
```

Pregnancy has:

```text
conceptionDate
```

They represent related but conceptually different information.

For Liventra:

```text
inseminationDate
=
breeding procedure date

conceptionDate
=
recorded/estimated conception date for the pregnancy
```

The exact business rule should be consistent throughout the system.

---

# 20. Conception Date Validation

Normally:

```text
conceptionDate >= breeding.heatDetectedDate
```

and should generally be related to:

```text
breeding.inseminationDate
```

A conception date before heat detection should be rejected unless correcting historical data.

---

# 21. `expectedCalvingDate`

### Type

Date

### Required

Yes / recommended

### Purpose

Stores the expected date of calving/delivery.

Example:

```text
2027-05-17
```

This is used for:

```text
Pregnancy monitoring
Calving reminders
Farm planning
Reports
```

---

# 22. Expected Calving Calculation

The system can calculate an initial expected calving date from:

```text
conceptionDate
+
approved gestation-period rule
```

However, the exact gestation period should be configurable according to the project's livestock domain rules rather than hard-coded blindly.

Once calculated:

```text
expectedCalvingDate
```

should be stored with the Pregnancy record.

---

# 23. Snapshot Principle for Expected Calving

Suppose the system's calculation rule later changes.

The old Pregnancy should retain its previously recorded:

```text
expectedCalvingDate
```

rather than being silently recalculated every time the UI opens.

This is important for:

```text
Historical accuracy
Reports
Notifications
Evaluation
```

---

# 24. Expected Date vs Actual Date

These are different.

```text
expectedCalvingDate
=
prediction/planning date

actualCalvingDate
=
what actually happened
```

Example:

```text
Expected:
May 17

Actual:
May 16
```

This allows the system to measure timing differences.

---

# 25. `actualCalvingDate`

### Type

Date

### Required

Conditional

### Purpose

Records when the animal actually delivered/calved.

Normally:

```text
pregnancyStatus = Delivered
```

requires:

```text
actualCalvingDate
```

---

# 26. Actual Calving Date Rules

For:

```text
pregnancyStatus = Pregnant
```

normally:

```text
actualCalvingDate = null
```

For:

```text
pregnancyStatus = Delivered
```

normally:

```text
actualCalvingDate != null
```

This prevents inconsistent records.

---

# 27. Actual Date Cannot Normally Be Before Conception

Normally:

```text
actualCalvingDate > conceptionDate
```

A delivery date before conception should be rejected.

---

# 28. `calfCount`

### Type

Number / Integer

### Required

Conditional

### Purpose

Stores the number of calves resulting from the pregnancy.

Examples:

```text
1
2
```

---

# 29. Calf Count Validation

For:

```text
pregnancyStatus = Delivered
```

normally:

```text
calfCount >= 1
```

For:

```text
pregnancyStatus = Pregnant
```

the field may remain:

```text
null
```

because delivery has not happened yet.

For:

```text
pregnancyStatus = Failed
```

or:

```text
Aborted
```

normally:

```text
calfCount = null
```

or no delivered calf count.

---

# 30. `calfGender`

### Type

Enum

### Allowed Values

```text
Male
Female
Twin
```

These are the finalized values. fileciteturn16file0turn16file1

---

# 31. Important Interpretation of `calfGender`

The current model does **not** have:

```text
calves: []
```

Therefore, it cannot store the individual gender of each calf when:

```text
calfCount = 2
```

Instead, it has:

```text
calfGender = Twin
```

So `Twin` should be interpreted as:

```text
The pregnancy resulted in twins.
```

rather than as an individual calf gender.

---

# 32. Current Model Limitation — Twin Gender

Example:

```text
calfCount = 2
```

but:

```text
calfGender = Twin
```

does not tell us:

```text
Male + Female
Male + Male
Female + Female
```

The current finalized model does not represent that level of detail.

Do not silently add:

```text
calfGender[]
```

to the finalized schema.

If detailed calf-level tracking is later required, a dedicated calf/offspring relationship can be considered.

---

# 33. `pregnancyStatus`

### Type

Enum

### Allowed Values

```text
Pregnant
Delivered
Aborted
Failed
```

These are the finalized values. fileciteturn16file0turn16file1

---

# 34. Status: Pregnant

Meaning:

```text
Pregnancy is confirmed and currently ongoing.
```

Typical state:

```text
conceptionDate exists
expectedCalvingDate exists
actualCalvingDate = null
```

Example:

```text
Status:
Pregnant

Expected calving:
May 17
```

---

# 35. Status: Delivered

Meaning:

```text
Pregnancy ended with successful calving/delivery.
```

Typical required fields:

```text
actualCalvingDate
calfCount
calfGender
```

The Animal's reproductive state can then transition toward the appropriate post-calving state.

---

# 36. Status: Aborted

Meaning:

```text
Pregnancy ended before successful delivery due to abortion.
```

Current model does not contain:

```text
abortionDate
```

Therefore, if the date is important, it must currently be captured through available date/remarks conventions rather than inventing a new field.

The status itself is sufficient to indicate the pregnancy outcome.

---

# 37. Status: Failed

Meaning:

```text
The pregnancy record was considered unsuccessful/failed.
```

Important distinction:

```text
Breeding.pregnancyResult = Failed
```

normally means pregnancy was not confirmed from that breeding.

Whereas:

```text
Pregnancy.pregnancyStatus = Failed
```

means a Pregnancy record exists but the pregnancy episode is considered failed.

The exact application workflow should avoid creating a Pregnancy record for a breeding that was never confirmed.

---

# 38. Pregnancy Status Lifecycle

Normal successful lifecycle:

```text
Pregnant
   ↓
Delivered
```

Unsuccessful pregnancy lifecycle:

```text
Pregnant
   ↓
Aborted
```

or:

```text
Pregnant
   ↓
Failed
```

depending on the farm's recorded outcome.

Do not allow:

```text
Delivered → Pregnant
```

as a normal state transition.

---

# 39. Status Transition Table

| Current | Next | Normal? |
|---|---|---|
| Pregnant | Delivered | Yes |
| Pregnant | Aborted | Yes |
| Pregnant | Failed | Conditional |
| Delivered | Pregnant | No |
| Delivered | Aborted | No |
| Aborted | Pregnant | No |
| Failed | Pregnant | No |

If a correction is necessary, it should be controlled and audited.

---

# 40. `remarks`

### Type

String

### Required

Optional

### Purpose

Stores additional pregnancy/delivery information not represented by dedicated fields.

Examples:

```text
Normal pregnancy progression.
```

```text
Delivery occurred earlier than expected.
```

```text
Abortion recorded after veterinary confirmation.
```

```text
Twin delivery; individual calf records not yet created.
```

---

# 41. `createdBy`

### Type

ObjectId / User or FarmMember reference

### Required

Yes

### Purpose

Identifies who created the Pregnancy record.

---

# 42. `createdAt`

### Type

Date

### Required

Automatic

### Purpose

Stores when the Pregnancy record was created.

---

# 43. `updatedAt`

### Type

Date

### Required

Automatic

### Purpose

Stores when the Pregnancy record was last updated.

---

# 44. Pregnancy Creation Workflow

Recommended:

```text
Breeding
   ↓
pregnancyResult = Confirmed
   ↓
Validate Breeding
   ↓
Validate Animal
   ↓
Check no Pregnancy already exists
   ↓
Create Pregnancy
   ↓
Set pregnancyStatus = Pregnant
   ↓
Update Animal.reproductiveStatus = Pregnant
   ↓
Generate calving reminder
   ↓
AuditLog
```

---

# 45. Why Pregnancy Should Not Be Created Too Early

Do not create:

```text
Pregnancy
```

merely because:

```text
Breeding exists
```

because a breeding attempt can fail.

Correct:

```text
Breeding
   ↓
Pending
   ↓
Pregnancy Check
   ↓
Confirmed
   ↓
Pregnancy
```

Incorrect:

```text
Breeding
   ↓
Immediately create Pregnancy
```

---

# 46. Pregnancy Confirmation Transaction

When pregnancy is confirmed:

```text
BEGIN TRANSACTION

Update Breeding:
pregnancyResult = Confirmed

Create Pregnancy

Update Animal:
reproductiveStatus = Pregnant

Create Notification

Create AuditLog

COMMIT
```

If any critical operation fails:

```text
ROLLBACK
```

This prevents inconsistent states.

---

# 47. Duplicate Pregnancy Prevention

Before creating:

```text
Pregnancy
```

check:

```text
breedingId
```

for an existing record.

If one exists:

```text
Reject duplicate
```

or:

```text
Return existing Pregnancy
```

depending on API design.

---

# 48. Animal Pregnancy State

When:

```text
Pregnancy.pregnancyStatus = Pregnant
```

the system should normally set:

```text
Animal.reproductiveStatus = Pregnant
```

But this must be based on the latest active pregnancy.

---

# 49. Avoiding Old Pregnancy State Overwrite

Suppose an animal has:

```text
Pregnancy A → Delivered
Pregnancy B → Pregnant
```

The current Animal state should be:

```text
Pregnant
```

based on Pregnancy B.

Therefore, updating Animal state must consider:

```text
latest valid pregnancy
```

not simply whichever record was last edited.

---

# 50. Delivered Workflow

When delivery occurs:

```text
Pregnancy:
status = Delivered

actualCalvingDate = actual date
calfCount = number
calfGender = appropriate value
```

Then:

```text
Animal reproductive state
        ↓
post-calving state
```

The exact state may depend on lactation.

For example:

```text
Lactating
```

if the animal has entered milk production.

---

# 51. Delivery Validation

For:

```text
status = Delivered
```

recommended validation:

```text
actualCalvingDate required
calfCount >= 1
calfGender required
```

Also:

```text
actualCalvingDate > conceptionDate
```

---

# 52. Expected vs Actual Calving Analytics

The system can calculate:

```text
Calving Variance
=
actualCalvingDate - expectedCalvingDate
```

Example:

```text
Expected:
May 17

Actual:
May 16

Variance:
-1 day
```

This can support reproductive analytics.

---

# 53. Abortion Workflow

```text
Pregnant
   ↓
Abortion confirmed
   ↓
status = Aborted
   ↓
Animal.reproductiveStatus = Open
   ↓
AuditLog
```

The exact date of abortion is not represented by a dedicated field in the current model.

Therefore, do not invent one.

---

# 54. Failed Pregnancy Workflow

If a Pregnancy record is considered failed:

```text
status = Failed
```

Then:

```text
Animal.reproductiveStatus
→ Open
```

provided there is no newer pregnancy.

The historical Pregnancy remains.

---

# 55. Pregnancy + Breeding History

Animal profile can display:

```text
Breeding:
BRD-001
Result:
Confirmed

Pregnancy:
PREG-001
Status:
Delivered
```

This creates a traceable reproductive chain.

---

# 56. Pregnancy + Breeding Analytics

From:

```text
Pregnancy.breedingId
```

the system can calculate:

```text
Successful pregnancies
Breeding success rate
Pregnancy outcomes
Pregnancy duration
Calving timing
```

---

# 57. Pregnancy Duration

Conceptually:

```text
Pregnancy Duration
=
actualCalvingDate - conceptionDate
```

Only calculate this when:

```text
status = Delivered
```

and both dates exist.

Example:

```text
Conception:
Aug 18

Actual calving:
May 16

Duration:
calculated in days
```

---

# 58. Current Pregnancy Dashboard

Farm dashboard can show:

```text
Currently Pregnant
Expected Calving Soon
Delivered This Month
Aborted
Failed
```

Current pregnant animals can be found using:

```text
pregnancyStatus = Pregnant
```

---

# 59. Expected Calving Reminder

If:

```text
pregnancyStatus = Pregnant
```

and:

```text
expectedCalvingDate
```

is approaching:

```text
Create Notification
```

The finalized Notification model explicitly supports:

```text
referenceType = Pregnancy
```

and therefore Pregnancy is directly supported as a notification reference. fileciteturn16file15

---

# 60. Calving Notification

Example:

```text
Expected Calving Reminder

Animal:
AN-000125

Expected calving:
2027-05-17
```

The system can generate reminders before the expected date according to configured rules.

---

# 61. Overdue Pregnancy

If:

```text
pregnancyStatus = Pregnant
```

and:

```text
expectedCalvingDate < today
```

the system can raise:

```text
Warning / Alert
```

However:

```text
Do NOT automatically set status = Delivered
```

because actual delivery must be recorded.

---

# 62. Delivered Confirmation

When actual delivery is recorded:

```text
actualCalvingDate
+
calfCount
+
calfGender
+
status = Delivered
```

then:

```text
Expected-calving notification
→ resolved/archived
```

and relevant animal state can be updated.

---

# 63. Pregnancy + Calf Creation

After delivery, the farm may create Animal records for the calves.

The current Pregnancy model stores:

```text
calfCount
calfGender
```

but does not contain:

```text
calfIds[]
```

Therefore, calf creation is a separate Animal workflow.

Conceptually:

```text
Pregnancy
   ↓
Delivered
   ↓
Calf count/gender
   ↓
Create Animal record(s)
```

---

# 64. Calf Parentage

Because Pregnancy has:

```text
animalId
breedingId
```

and Breeding has:

```text
bullId
```

the system can infer:

```text
Mother = Pregnancy.animalId
Father = Breeding.bullId
```

where `bullId` is available.

This is useful for lineage/animal history.

---

# 65. Current Model Limitation — No `calfIds[]`

The current model cannot directly store:

```text
PREG-001
   ↓
calfIds:
[
   CALF-001,
   CALF-002
]
```

Therefore, if calves are created as Animals, parent-child relationships are not directly represented by the Pregnancy collection unless another mechanism is added later.

Do not silently add `calfIds[]` to the finalized schema.

This is a useful future enhancement if genealogy becomes a major feature.

---

# 66. Twin Logic

If:

```text
calfCount = 2
```

then:

```text
calfGender = Twin
```

is valid under the current enum.

But:

```text
calfCount = 1
calfGender = Twin
```

would be inconsistent and should be rejected.

---

# 67. Calf Count + Gender Validation

Recommended:

```text
calfCount = 1
→ calfGender = Male or Female

calfCount = 2
→ calfGender = Twin
```

The current enum does not support:

```text
Male + Female
```

as a separate value.

Therefore, exact twin gender combinations cannot currently be represented.

---

# 68. Pregnancy + Milk Production

After:

```text
Delivered
```

the animal may become:

```text
Lactating
```

Then MilkRecord becomes relevant:

```text
Pregnancy
   ↓
Delivered
   ↓
Lactating
   ↓
MilkRecord
```

The finalized SRS includes daily milk entry and production reporting. fileciteturn16file18

---

# 69. Pregnancy + Weight

Pregnancy itself does not store weight.

Weight history remains in:

```text
WeightRecord
```

The system can display pregnancy alongside weight trends:

```text
Pregnancy timeline
+
Weight timeline
```

without mixing the data models.

---

# 70. Pregnancy + Treatment

A pregnant animal may also receive Treatment.

These are separate records:

```text
Animal
 ├── Pregnancy
 └── Treatment
```

Treatment-specific health events should remain in Treatment.

Pregnancy remains focused on reproductive state.

---

# 71. Pregnancy + Vaccination

Vaccination remains a separate preventive-health record:

```text
Animal
 ├── Pregnancy
 └── Vaccination
```

The application may apply farm/veterinary rules for vaccination during pregnancy, but the Pregnancy model itself does not store vaccination data.

---

# 72. Pregnancy + Feed Recommendation

Pregnancy status can be relevant to the FeedRecommendation input snapshot.

For example:

```text
pregnancyStatus = Pregnant
```

can influence feed recommendation rules.

The important architectural point is:

```text
Pregnancy
   ↓
current reproductive information
   ↓
FeedRecommendation input snapshot
```

The recommendation should store its own snapshot rather than dynamically rewriting historical recommendations.

---

# 73. Pregnancy + Notification

Notification can reference Pregnancy directly:

```text
referenceType = Pregnancy
referenceId = Pregnancy._id
```

This supports:

```text
Expected Calving Reminder
Pregnancy Monitoring Alert
```

The finalized Notification model explicitly supports Pregnancy as a reference type. fileciteturn16file15

---

# 74. Pregnancy + Report

The finalized Report model includes health/animal/report infrastructure but does not explicitly list a dedicated `Pregnancy` report type. fileciteturn16file15

Therefore, the current implementation should not invent a new `reportType` enum value.

Pregnancy information can be included through appropriate:

```text
Animal
```

or:

```text
Health
```

reports and filters, depending on the final reporting implementation.

---

# 75. Pregnancy + AuditLog

Important audit events:

```text
Pregnancy Created
Pregnancy Updated
Expected Calving Changed
Actual Calving Added
Status Changed
Calf Count Changed
Calf Gender Changed
Pregnancy Aborted
Pregnancy Failed
```

Especially important:

```text
Pregnant → Delivered
Pregnant → Aborted
Pregnant → Failed
```

These should be auditable.

---

# 76. Historical Integrity

Pregnancy records should be treated as historical reproductive records.

Avoid:

```text
Hard delete
```

after they have been used for:

```text
Animal state
Notifications
Reports
Calf creation
Analytics
```

If correction is required:

```text
Authorized update
+
AuditLog
```

should be used.

---

# 77. Pregnancy Correction Example

Suppose:

```text
expectedCalvingDate:
May 20
```

was entered incorrectly and corrected to:

```text
May 17
```

Audit should preserve:

```text
oldData:
May 20

newData:
May 17

changedFields:
expectedCalvingDate
```

---

# 78. Transaction Strategy — Delivery

Delivery can affect multiple parts of the system:

```text
Pregnancy
Animal
Notification
Calf Animal records
AuditLog
```

Recommended:

```text
BEGIN TRANSACTION

Update Pregnancy
Create/Update calf records
Update mother Animal reproductive status
Resolve relevant notifications
Create AuditLog

COMMIT
```

If the operation fails:

```text
ROLLBACK
```

This prevents partial delivery records.

---

# 79. Delivery and Mother Animal State

When:

```text
Pregnancy.status = Delivered
```

the mother is no longer:

```text
Pregnant
```

The system should transition her to the appropriate post-calving state, commonly:

```text
Lactating
```

when lactation has started.

The exact state should reflect actual farm data rather than blindly assuming every delivered animal is lactating.

---

# 80. Multiple Pregnancies Over Lifetime

An animal can have multiple Pregnancy records over time.

Example:

```text
PREG-001
2026
Delivered

PREG-002
2028
Delivered

PREG-003
2030
Aborted
```

Therefore:

```text
animalId
```

must not be unique in Pregnancy.

---

# 81. Active Pregnancy Uniqueness

Although multiple historical pregnancies are valid, an animal should normally have only:

```text
one active Pregnant episode
```

at a time.

Therefore, before creating a new Pregnancy:

```text
Check existing pregnancyStatus = Pregnant
for same animal
```

If one exists:

```text
Reject new Pregnancy
```

unless the operation is an authorized correction.

---

# 82. Pregnancy After Failed Breeding

If:

```text
Breeding A → Failed
```

and later:

```text
Breeding B → Confirmed
```

then:

```text
Pregnancy B
```

can be created.

The failed breeding remains historical.

---

# 83. Pregnancy After Abortion

If:

```text
Pregnancy A → Aborted
```

then later the animal may be:

```text
Open
```

and can enter another breeding cycle.

A future Breeding event creates a future Pregnancy.

The old aborted Pregnancy remains.

---

# 84. Pregnancy Status Must Match Animal State

Recommended consistency:

### Pregnancy status = Pregnant

```text
Animal.reproductiveStatus = Pregnant
```

### Pregnancy status = Delivered

```text
Animal.reproductiveStatus ≠ Pregnant
```

and should move to appropriate post-calving state.

### Pregnancy status = Aborted/Failed

```text
Animal.reproductiveStatus ≠ Pregnant
```

and may become:

```text
Open
```

provided no newer reproductive event exists.

---

# 85. Latest-Event Rule

Animal reproductive state should always be determined from the latest valid reproductive history.

Conceptually:

```text
Get latest relevant Breeding/Pregnancy event
             ↓
Determine current state
             ↓
Update Animal.reproductiveStatus
```

Do not allow an older record edit to incorrectly overwrite a newer state.

---

# 86. Pregnancy Dashboard Metrics

Useful metrics:

```text
Currently Pregnant
Expected to Calve This Month
Delivered This Month
Aborted Pregnancies
Failed Pregnancies
Average Pregnancy Duration
Average Calving Variance
Twin Deliveries
```

---

# 87. Pregnancy Analytics

Possible analytics:

```text
Pregnancy success rate
Abortion rate
Calving rate
Average gestation duration
Expected vs actual calving variance
Average calves per delivery
Twin rate
Animal-wise pregnancy history
```

---

# 88. Pregnancy Success Rate

This metric should be based on confirmed/eligible breeding events.

Conceptually:

```text
Confirmed Pregnancies
--------------------- × 100
Evaluated Breeding Attempts
```

Do not include:

```text
Pending
```

in the denominator if the attempt has not yet been evaluated.

---

# 89. Delivery Rate

Conceptually:

```text
Delivered Pregnancies
--------------------- × 100
Confirmed Pregnancies
```

This can help monitor pregnancy outcomes.

---

# 90. Abortion Rate

Conceptually:

```text
Aborted Pregnancies
------------------- × 100
Pregnancy Episodes
```

The exact denominator should be defined consistently in reports.

---

# 91. Twin Rate

Conceptually:

```text
Pregnancies with calfCount = 2
-------------------------------- × 100
Delivered Pregnancies
```

This is possible because the current model stores `calfCount`.

---

# 92. Expected vs Actual Calving Report

The system can generate:

```text
Animal
Pregnancy
Expected Calving
Actual Calving
Variance
```

Example:

```text
Animal      Expected       Actual       Variance
AN-001      May 17         May 16       -1 day
AN-002      May 20         May 23       +3 days
```

---

# 93. Search and Filtering

Pregnancy list should support:

```text
pregnancyNo
farm
animal
breeding
pregnancyStatus
expectedCalvingDate
actualCalvingDate
date range
```

Example:

```text
pregnancyStatus = Pregnant
```

shows active pregnancy records.

---

# 94. Expected Calving Search

Farm manager can filter:

```text
Expected calving:
Next 7 days
Next 30 days
This month
```

This is useful for operational planning.

---

# 95. Pregnancy Creation Authorization

Recommended based on the SRS role model:

```text
Admin
Farm Owner
Farm Manager
Authorized Worker
```

depending on final permission matrix.

The SRS defines:

```text
Admin = Full Access
Farm Owner = Manage Everything
Farm Manager = Regular Management
Worker = Data Entry
```

fileciteturn16file18

---

# 96. `createdBy` Validation

The creator should:

```text
exist
+
belong to the farm
+
have permission to create/update pregnancy records
```

This protects reproductive data.

---

# 97. Error Scenario — Breeding Does Not Exist

Input:

```text
breedingId = invalid
```

Result:

```text
Reject
```

---

# 98. Error Scenario — Cross-Farm Breeding

```text
Pregnancy.farmId = FARM-A

Breeding.farmId = FARM-B
```

Result:

```text
Reject
```

---

# 99. Error Scenario — Animal Mismatch

```text
Pregnancy.animalId = Animal A

Breeding.animalId = Animal B
```

Result:

```text
Reject
```

The breeding and pregnancy must refer to the same animal.

---

# 100. Error Scenario — Failed Breeding Creates Pregnancy

```text
Breeding.pregnancyResult = Failed
```

Attempt:

```text
Create Pregnancy
```

Result:

```text
Reject
```

unless there is an explicit historical correction scenario.

---

# 101. Error Scenario — Duplicate Pregnancy

```text
breedingId = BRD-001
```

already has:

```text
PREG-001
```

Attempting:

```text
PREG-002
breedingId = BRD-001
```

should be rejected.

---

# 102. Error Scenario — Delivered Without Actual Date

```text
pregnancyStatus = Delivered
actualCalvingDate = null
```

Result:

```text
Validation error
```

---

# 103. Error Scenario — Delivered Without Calf Count

```text
pregnancyStatus = Delivered
calfCount = null
```

Recommended:

```text
Validation error
```

---

# 104. Error Scenario — Invalid Twin Data

```text
calfCount = 1
calfGender = Twin
```

Result:

```text
Reject
```

under the current model interpretation.

---

# 105. Error Scenario — Pregnant With Actual Calving Date

```text
pregnancyStatus = Pregnant
actualCalvingDate = 2027-05-16
```

Result:

```text
Reject or require status transition to Delivered
```

because the record is internally inconsistent.

---

# 106. Error Scenario — Calving Before Conception

```text
conceptionDate = Aug 18
actualCalvingDate = Aug 10
```

Result:

```text
Reject
```

---

# 107. Error Scenario — Two Active Pregnancies

Animal:

```text
AN-001
```

already has:

```text
PREG-001
status = Pregnant
```

Attempt:

```text
PREG-002
status = Pregnant
animalId = AN-001
```

Result:

```text
Reject
```

unless explicitly handling a correction.

---

# 108. Current Model Limitation — No Pregnancy Check Date

Breeding stores:

```text
expectedPregnancyCheckDate
```

but Pregnancy does not store:

```text
actualPregnancyCheckDate
```

Therefore:

```text
When exactly was pregnancy confirmed?
```

is not directly represented inside Pregnancy.

The confirmation event can be traced through:

```text
Breeding.pregnancyResult
```

and timestamps/audit records.

---

# 109. Current Model Limitation — No Abortion Date

The current Pregnancy model has:

```text
pregnancyStatus = Aborted
```

but no:

```text
abortionDate
```

Therefore, exact abortion timing is not a dedicated field.

Do not silently add it to the finalized model.

---

# 110. Current Model Limitation — No Calf IDs

The model stores:

```text
calfCount
calfGender
```

but not:

```text
calfIds[]
```

Therefore, the direct parent → calf database relationship is not represented here.

If lineage is later required, this should be considered as a future model enhancement.

---

# 111. Current Model Limitation — Limited Twin Information

The enum:

```text
Male | Female | Twin
```

does not encode:

```text
Twin A = Male
Twin B = Female
```

Therefore, detailed twin sex information is outside the current finalized model.

---

# 112. Current Model Limitation — No Delivery Type

The model does not include:

```text
Normal
Assisted
Cesarean
Difficult Birth
```

Therefore, delivery type should not be inferred from the current data.

If required later, it can be considered as a future enhancement.

---

# 113. Current Model Limitation — No Birth Complication Field

Similarly, there is no dedicated:

```text
birthComplication
```

field.

Additional information can currently go into:

```text
remarks
```

but structured analytics on complications would require future schema support.

---

# 114. Recommended Service-Layer Methods

A clean backend can use:

```text
pregnancyService.createFromConfirmedBreeding()
pregnancyService.update()
pregnancyService.confirmDelivery()
pregnancyService.recordAbortion()
pregnancyService.markFailed()
pregnancyService.calculateExpectedCalving()
pregnancyService.getActivePregnancies()
pregnancyService.getUpcomingCalvings()
pregnancyService.createCalfRecords()
```

These are service-layer responsibilities, not extra DB fields.

---

# 115. Delivery Service Logic

Conceptually:

```text
confirmDelivery(pregnancyId)

1. Load Pregnancy
2. Verify status = Pregnant
3. Validate actualCalvingDate
4. Validate calfCount
5. Validate calfGender
6. Update Pregnancy → Delivered
7. Create/update calf Animal records if required
8. Update mother Animal reproductive state
9. Resolve calving notification
10. Create AuditLog
11. Commit transaction
```

---

# 116. Abortion Service Logic

Conceptually:

```text
recordAbortion(pregnancyId)

1. Load Pregnancy
2. Verify status = Pregnant
3. Update status → Aborted
4. Update Animal reproductive state
5. Resolve pregnancy/calving reminders
6. Create AuditLog
7. Commit
```

Because there is no `abortionDate` field, do not invent one in the current schema.

---

# 117. Calving Reminder Logic

Query:

```text
pregnancyStatus = Pregnant
AND
expectedCalvingDate <= reminderThreshold
```

Then:

```text
Create Notification
referenceType = Pregnancy
referenceId = Pregnancy._id
```

The Notification model explicitly supports Pregnancy references. fileciteturn16file15

---

# 118. Overdue Calving Logic

Query:

```text
pregnancyStatus = Pregnant
AND
expectedCalvingDate < today
```

Then:

```text
Notification:
type = Warning
or
Alert
```

But:

```text
Do not automatically change status.
```

The farmer/user must record the actual outcome.

---

# 119. Pregnancy + Notification Lifecycle

```text
Pregnancy Created
      ↓
Expected Calving Reminder
      ↓
Notification Active
      ↓
Delivery Recorded
      ↓
Pregnancy = Delivered
      ↓
Notification Archived/Resolved
```

This avoids stale reminders.

---

# 120. Pregnancy + Audit Example

Before:

```text
status = Pregnant
```

After:

```text
status = Delivered
actualCalvingDate = 2027-05-16
calfCount = 1
calfGender = Female
```

Audit should capture:

```text
oldData
newData
changedFields
```

and identify:

```text
who
when
```

---

# 121. Pregnancy History Query

For an animal:

```text
find Pregnancy
where animalId = selectedAnimal
sort by conceptionDate descending
```

This provides:

```text
Latest pregnancy first
```

---

# 122. Active Pregnancy Query

```text
find Pregnancy
where:
animalId = selectedAnimal
AND
pregnancyStatus = Pregnant
```

Normally:

```text
0 or 1 result
```

per animal.

---

# 123. Upcoming Calving Query

```text
pregnancyStatus = Pregnant
AND
expectedCalvingDate between today and selected date
```

Example:

```text
Next 30 days
```

This powers:

```text
Dashboard
Calendar
Notification
```

---

# 124. Delivered This Month Query

```text
pregnancyStatus = Delivered
AND
actualCalvingDate within month
```

Useful for:

```text
Production planning
Calf statistics
Farm reports
```

---

# 125. Pregnancy Analytics by Farm

Because `farmId` is stored:

```text
Farm
   ↓
Pregnancies
```

can calculate:

```text
Active pregnancies
Deliveries
Abortion count
Pregnancy failures
Average gestation
Twin deliveries
```

---

# 126. Reproductive Performance Flow

```text
Breeding Attempts
        ↓
Pregnancy Confirmation
        ↓
Pregnancy Outcomes
        ↓
Calving Outcomes
        ↓
Calf Records
        ↓
Milk Production
```

This allows Liventra to connect reproduction with production.

---

# 127. Example — Complete Pregnancy

```javascript
{
    pregnancyNo: "PREG-000001",

    farmId: "FARM-001",

    animalId: "AN-000125",

    breedingId: "BRD-000015",

    conceptionDate: "2026-08-18",

    expectedCalvingDate: "2027-05-17",

    actualCalvingDate: null,

    calfCount: null,

    calfGender: null,

    pregnancyStatus: "Pregnant",

    remarks: "Pregnancy confirmed.",

    createdBy: "USER-001",

    createdAt: "2026-09-18T10:00:00Z",

    updatedAt: "2026-09-18T10:00:00Z"
}
```

Values are illustrative.

---

# 128. Example — Delivered Pregnancy

```javascript
{
    pregnancyNo: "PREG-000001",

    pregnancyStatus: "Delivered",

    actualCalvingDate: "2027-05-16",

    calfCount: 1,

    calfGender: "Female",

    remarks: "Successful delivery."
}
```

---

# 129. Example — Twin Delivery

```javascript
{
    pregnancyNo: "PREG-000002",

    pregnancyStatus: "Delivered",

    actualCalvingDate: "2027-06-02",

    calfCount: 2,

    calfGender: "Twin",

    remarks: "Twin delivery recorded."
}
```

Remember:

```text
Twin
```

does not encode the individual sex of each calf.

---

# 130. Example — Aborted Pregnancy

```javascript
{
    pregnancyNo: "PREG-000003",

    pregnancyStatus: "Aborted",

    actualCalvingDate: null,

    calfCount: null,

    calfGender: null,

    remarks: "Pregnancy loss recorded."
}
```

---

# 131. Final Pregnancy Lifecycle

```text
                 Breeding
                    │
             Result = Confirmed
                    │
                    ▼
              ┌─────────────┐
              │   Pregnant  │
              └──────┬──────┘
                     │
              ┌──────┴───────┐
              │              │
              ▼              ▼
         ┌──────────┐   ┌──────────┐
         │ Delivered│   │  Aborted │
         └──────────┘   └──────────┘
              │
              ▼
         Calf Records
              │
              ▼
          Lactation
              │
              ▼
          MilkRecord
```

`Failed` is also available as a terminal outcome under the finalized enum.

---

# 132. Final Business Rules Checklist

```text
[✓] Pregnancy represents a specific pregnancy episode
[✓] Breeding represents the upstream reproductive event
[✓] pregnancyNo should be unique
[✓] farmId is required
[✓] animalId is required
[✓] animal should normally be Female
[✓] animal must belong to farmId
[✓] breedingId is required
[✓] breeding must exist
[✓] breeding and pregnancy must reference the same animal
[✓] breeding and pregnancy must belong to the same farm
[✓] Pregnancy should normally originate from Confirmed breeding
[✓] One breeding should have at most one Pregnancy
[✓] conceptionDate records pregnancy conception information
[✓] expectedCalvingDate supports planning and reminders
[✓] actualCalvingDate is required for Delivered
[✓] calfCount is required for Delivered
[✓] calfGender is required for Delivered
[✓] calfCount and calfGender must be internally consistent
[✓] Twin means twin outcome under the current enum
[✓] Pregnancy status values are Pregnant, Delivered, Aborted, Failed
[✓] Pregnant → Delivered is the normal successful lifecycle
[✓] Pregnant → Aborted is a valid unsuccessful outcome
[✓] Failed breeding should not normally create Pregnancy
[✓] Multiple historical pregnancies per animal are valid
[✓] One animal should normally have only one active Pregnant episode
[✓] Old pregnancy records must not overwrite newer reproductive state
[✓] Delivered pregnancy should update mother animal state
[✓] Pregnancy can lead to calf Animal records
[✓] Current model does not store calfIds[]
[✓] Current model does not store actual pregnancy-check date
[✓] Current model does not store abortion date
[✓] Current model does not store delivery type
[✓] Current model does not store individual twin genders
[✓] Expected calving notifications can reference Pregnancy
[✓] Important pregnancy changes should be audited
[✓] Delivery operations affecting multiple collections should be transaction-safe
[✓] Pregnancy history should not be casually hard-deleted
[✓] Pregnancy supports reproductive analytics
[✓] Pregnancy supports calving planning
[✓] Pregnancy connects breeding with later production
```

---

# 133. Final Status

**Model:** `Pregnancy`  
**Status:** ✅ Finalized  
**Business Logic:** ✅ Documented  
**Breeding Integration:** ✅  
**Animal Integration:** ✅  
**Calving Workflow:** ✅  
**Calf Outcome:** ✅  
**Notification Integration:** ✅  
**Auditability:** ✅  
**Analytics:** ✅  
**Historical Integrity:** ✅  

**Next Model:** `Inventory`
