# Liventra — Business Logic Documentation
## Model 17: Breeding

**Document Version:** Business Logic v1.0  
**Project:** Liventra — Smart Livestock Farm Management System  
**Database:** MongoDB  
**Architecture:** MERN  
**Module:** Reproduction Management

---

# 1. Source / Finalized Model

The current finalized Liventra DB model defines `Breeding` as:

```text
Breeding Model:
_id
breedingNo
farmId
animalId
heatDetectedDate
inseminationDate
breedingMethod:
    Artificial Insemination
    Natural Mating
    Embryo Transfer
bullId (AnimalID)
semenBatchNo
technicianName
expectedPregnancyCheckDate
pregnancyResult:
    Pending
    Confirmed
    Failed
remarks
createdBy
createdAt
updatedAt
```

This is the current finalized Breeding structure in the latest DB-model source. fileciteturn15file16turn15file17

> **Important:** The finalized model uses `bullId (AnimalID)` rather than a separate Bull/Doctor model. It also uses `pregnancyResult` here, while the separate `Pregnancy` model manages the actual pregnancy record/state. This document follows that structure.

---

# 2. Purpose

`Breeding` records an **animal-specific breeding/reproductive event**.

It answers:

> "When was heat detected, when was the animal inseminated/mated, what breeding method was used, which bull was involved if applicable, which semen batch was used, who handled the process, when should pregnancy be checked, and what was the pregnancy-check result?"

Example:

```text
Animal:
AN-000125

Heat detected:
2026-08-17

Insemination:
2026-08-18

Method:
Artificial Insemination

Semen batch:
SEM-2026-001

Pregnancy check:
2026-09-18

Result:
Pending
```

---

# 3. Breeding vs Pregnancy

This distinction is extremely important.

## Breeding

```text
What reproductive event happened?
```

Example:

```text
Heat detected
Insemination
Breeding method
Semen batch
Bull
Technician
Pregnancy-check date
```

## Pregnancy

```text
What was the outcome after breeding?
```

The finalized Pregnancy model references:

```text
breedingId
```

and contains:

```text
conceptionDate
expectedCalvingDate
actualCalvingDate
calfCount
calfGender
pregnancyStatus
```

fileciteturn15file17

Therefore:

```text
Breeding
   ↓
Pregnancy
```

is the correct lifecycle.

---

# 4. Breeding vs Animal

Animal stores the current reproductive state:

```text
reproductiveStatus:
Calf
Heifer
Open
Pregnant
Lactating
Dry
```

Breeding stores the actual historical reproductive event.

Therefore:

```text
Animal.reproductiveStatus
=
CURRENT STATE

Breeding
=
HISTORICAL EVENT
```

Example:

```text
Animal:
reproductiveStatus = Pregnant

Breeding:
inseminationDate = Aug 18
```

---

# 5. Breeding vs Pregnancy vs Animal State

The three layers should be kept separate:

```text
Breeding
   ↓
Event:
"Insemination happened."

Pregnancy
   ↓
Outcome/state:
"Pregnancy confirmed."

Animal
   ↓
Current state:
"Pregnant."
```

This separation is important for historical tracking.

---

# 6. Overall Reproductive Workflow

The core workflow is:

```text
Female Animal
      ↓
Heat Detected
      ↓
Breeding Record
      ↓
Insemination / Mating
      ↓
Expected Pregnancy Check
      ↓
Pregnancy Check
      ↓
Pending / Confirmed / Failed
      ↓
If Confirmed
      ↓
Pregnancy Record
      ↓
Expected Calving
      ↓
Delivery
```

---

# 7. Field-by-Field Business Logic

# 7.1 `_id`

### Type

MongoDB `ObjectId`

### Required

Yes

### Purpose

Unique database identifier of the breeding record.

Example:

```text
ObjectId(...)
```

---

# 7.2 `breedingNo`

### Type

String

### Required

Yes

### Example

```text
BRD-000001
```

### Purpose

Human-readable unique identifier for a breeding event.

Used in:

```text
Breeding history
Pregnancy reference
Reports
Search
Audit
```

---

# 8. Breeding Number Generation

Recommended format:

```text
BRD-{SEQUENCE}
```

Examples:

```text
BRD-000001
BRD-000002
BRD-000003
```

The backend should generate and validate uniqueness.

The frontend should not be trusted to generate the unique number.

---

# 9. `farmId`

### Type

ObjectId / Farm reference

### Required

Yes

### Purpose

Identifies the farm where the breeding event belongs.

This is necessary for multi-farm isolation.

---

# 10. Farm Isolation Rule

Before creating a Breeding record:

```text
Breeding.farmId
       ↓
must match
       ↓
Animal.farmId
```

Example:

```text
Breeding.farmId = FARM-A

Animal.farmId = FARM-B
```

must be rejected.

The SRS requires users to be restricted to the farms they are authorized to access. fileciteturn14file19

---

# 11. `animalId`

### Type

ObjectId / Animal reference

### Required

Yes

### Purpose

Identifies the animal involved in the breeding event.

---

# 12. Animal Eligibility

Breeding should normally be available only for an eligible reproductive animal.

At minimum, the backend should validate:

```text
Animal exists
+
Animal belongs to farm
+
Animal.gender = Female
```

The current Animal model defines:

```text
gender: Male | Female
```

and:

```text
purpose: Dairy | Beef | Breeding | Dual Purpose
```

fileciteturn15file8

A male animal should not normally be the `animalId` of a female insemination/breeding event.

---

# 13. Reproductive Status Validation

The system should consider the current:

```text
Animal.reproductiveStatus
```

before allowing breeding.

Normally eligible states may include:

```text
Heifer
Open
```

depending on the animal's reproductive history and farm rules.

The following should normally be blocked:

```text
Pregnant
Calf
```

and potentially:

```text
Lactating
Dry
```

depending on the farm's breeding policy.

The exact eligibility rule should be configurable rather than hard-coded as a universal veterinary rule.

---

# 14. `heatDetectedDate`

### Type

Date

### Required

Yes

### Purpose

Records when the animal's heat/estrus was detected.

Example:

```text
2026-08-17
```

---

# 15. Why Heat Date Matters

Heat detection is the starting point of the breeding event.

Therefore:

```text
heatDetectedDate
```

helps determine:

```text
Breeding timing
Insemination timing
Pregnancy-check scheduling
Reproductive history
```

---

# 16. Date Validation

Normally:

```text
heatDetectedDate <= inseminationDate
```

If:

```text
heatDetectedDate > inseminationDate
```

the record should be rejected unless it is a deliberate historical correction.

---

# 17. Future Heat Date

A normal historical breeding event should not generally have:

```text
heatDetectedDate > current date
```

unless the application explicitly supports planned/future reproductive events.

The current model is event-oriented, so future dates should normally be restricted.

---

# 18. `inseminationDate`

### Type

Date

### Required

Conditional

### Purpose

Records when insemination/mating actually occurred.

Example:

```text
2026-08-18
```

---

# 19. Is Insemination Date Always Required?

Not necessarily.

The finalized model supports multiple methods:

```text
Artificial Insemination
Natural Mating
Embryo Transfer
```

For a recorded breeding event, the actual reproductive procedure date should normally be present.

If the system allows creating a preliminary heat-detection record before the breeding event happens, then `inseminationDate` can remain temporarily null.

However, the current schema has no separate `Scheduled` status.

Therefore, the cleanest implementation is:

```text
Create Breeding
→ actual breeding/insemination event
```

rather than treating Breeding as a mere heat-alert record.

---

# 20. `breedingMethod`

### Type

Enum

### Allowed Values

```text
Artificial Insemination
Natural Mating
Embryo Transfer
```

These are the finalized values. fileciteturn15file16

---

# 21. Artificial Insemination

When:

```text
breedingMethod = Artificial Insemination
```

the system should expect information such as:

```text
inseminationDate
semenBatchNo
technicianName
```

`bullId` may identify the genetic sire/bull associated with the semen where the farm tracks that animal.

---

# 22. Natural Mating

When:

```text
breedingMethod = Natural Mating
```

the system should identify the bull where possible:

```text
bullId
```

The bull must be a valid Animal belonging to the relevant farm.

---

# 23. Embryo Transfer

When:

```text
breedingMethod = Embryo Transfer
```

the event is different from normal insemination.

The current schema still provides:

```text
inseminationDate
bullId
semenBatchNo
```

but not dedicated:

```text
embryoId
donorAnimalId
embryoBatchNo
```

Therefore, do not invent those fields in the current implementation.

Relevant additional information can be recorded in:

```text
remarks
```

unless a future reproductive enhancement adds dedicated embryo-transfer fields.

---

# 24. Method-Specific Validation

Recommended:

### Artificial Insemination

```text
inseminationDate required
semenBatchNo recommended/required
technicianName recommended/required
```

### Natural Mating

```text
inseminationDate / mating date required
bullId required
semenBatchNo should normally be null
```

### Embryo Transfer

```text
transfer date required
```

Additional embryo details can currently be captured in `remarks`.

---

# 25. `bullId`

### Type

ObjectId / Animal reference

### Required

Conditional

### Comment in finalized model

```text
bullId (AnimalID)
```

fileciteturn15file16

### Purpose

Identifies the bull involved in the breeding event.

---

# 26. Bull Validation

If `bullId` is provided:

```text
Bull exists
+
Bull belongs to farm
+
Bull.gender = Male
```

should be validated.

A female Animal should not be accepted as:

```text
bullId
```

---

# 27. Bull and Artificial Insemination

For Artificial Insemination:

```text
bullId
```

can represent the sire/genetic source associated with the semen.

However, the current model does not explicitly distinguish:

```text
physical bull used
```

from:

```text
semen genetic source
```

If precise semen genealogy is required later, a dedicated breeding-material model could be introduced.

For the current model, `bullId` should be used consistently according to the project's agreed interpretation.

---

# 28. `semenBatchNo`

### Type

String

### Required

Conditional

### Example

```text
SEM-2026-00125
```

### Purpose

Identifies the semen batch used in Artificial Insemination.

This provides traceability.

---

# 29. Why Semen Batch Tracking Matters

It can help answer:

```text
Which semen batch was used?
Which animals received it?
What was the pregnancy outcome?
```

This can support:

```text
Breeding analysis
Traceability
Supplier/batch analysis
Research
```

---

# 30. Method-Specific Semen Rule

Normally:

```text
Artificial Insemination
→ semenBatchNo applicable
```

while:

```text
Natural Mating
→ semenBatchNo not applicable
```

If the current system does not enforce method-specific nullability, the service layer should still validate it.

---

# 31. `technicianName`

### Type

String

### Required

Conditional/recommended

### Example

```text
Md. Rahim
```

### Purpose

Stores the name of the technician/person who performed the breeding procedure.

---

# 32. Why `technicianName` Is Not `technicianId`

The finalized model specifically contains:

```text
technicianName
```

rather than a dedicated technician reference.

Therefore, do not introduce a Technician model just for this field.

If later the farm wants staff-level traceability, a future schema could replace/augment this with an authorized user/farm-member reference.

For the current finalized model:

```text
technicianName
```

should be retained.

---

# 33. `expectedPregnancyCheckDate`

### Type

Date

### Required

Recommended/Conditional

### Purpose

Stores the planned date when pregnancy status should be checked.

Example:

```text
2026-09-18
```

---

# 34. Pregnancy Check Reminder

This field can trigger:

```text
Notification
```

Example:

```text
Pregnancy check due for Animal AN-000125.
```

The Notification model supports:

```text
referenceType = Pregnancy
```

but the current Notification enum does not include `Breeding`. Therefore, if the reminder is specifically about the breeding event, the implementation should use the available supported reference convention consistently rather than inventing a new enum value. fileciteturn15file7

---

# 35. Expected Check Date vs Actual Result

Breeding stores:

```text
expectedPregnancyCheckDate
pregnancyResult
```

It does not store a separate:

```text
actualPregnancyCheckDate
```

Therefore, the current model can record:

```text
when the check is expected
+
what result was obtained
```

but not precisely:

```text
when the actual check occurred
```

unless that date is captured elsewhere or included in remarks.

This is a current-model limitation, not a reason to silently add a field.

---

# 36. `pregnancyResult`

### Type

Enum

### Allowed Values

```text
Pending
Confirmed
Failed
```

These are the finalized values. fileciteturn15file16

---

# 37. Pregnancy Result: Pending

Meaning:

```text
Breeding occurred
but pregnancy has not yet been confirmed.
```

Example:

```text
Breeding:
Aug 18

Pregnancy check:
Sep 18

Current result:
Pending
```

---

# 38. Pregnancy Result: Confirmed

Meaning:

```text
Pregnancy has been confirmed after the breeding event.
```

At this point, Liventra should create or update the related:

```text
Pregnancy
```

record.

---

# 39. Pregnancy Result: Failed

Meaning:

```text
Pregnancy was not confirmed / breeding did not result in confirmed pregnancy.
```

The animal may become:

```text
Open
```

depending on the farm's reproductive workflow.

A new heat/breeding cycle can then be recorded later.

---

# 40. Breeding → Pregnancy Workflow

When:

```text
pregnancyResult = Confirmed
```

the system should:

```text
Find/Create Pregnancy
        ↓
breedingId = current Breeding._id
        ↓
animalId = current Breeding.animalId
        ↓
conceptionDate = according to approved rule/record
        ↓
expectedCalvingDate = calculated/recorded
        ↓
pregnancyStatus = Pregnant
```

The finalized Pregnancy model explicitly contains `breedingId`. fileciteturn15file17

---

# 41. One Breeding → Pregnancy Relationship

The intended relationship is:

```text
Breeding
   ↓
Pregnancy
```

A confirmed breeding event should normally produce one corresponding pregnancy episode.

A failed breeding does not create a confirmed pregnancy record.

---

# 42. Important Distinction: `pregnancyResult` vs `pregnancyStatus`

Breeding:

```text
pregnancyResult
=
Outcome of this breeding attempt
```

Pregnancy:

```text
pregnancyStatus
=
Current lifecycle of the resulting pregnancy
```

For example:

```text
Breeding:
pregnancyResult = Confirmed

Pregnancy:
pregnancyStatus = Pregnant
```

Later:

```text
Pregnancy:
pregnancyStatus = Delivered
```

The original Breeding record remains:

```text
Confirmed
```

---

# 43. Animal Reproductive Status Update

When pregnancy is confirmed:

```text
Breeding.pregnancyResult = Confirmed
        ↓
Animal.reproductiveStatus = Pregnant
```

When breeding fails:

```text
Breeding.pregnancyResult = Failed
        ↓
Animal.reproductiveStatus = Open
```

This should only occur if no newer reproductive event overrides the state.

---

# 44. Avoiding State Overwrite

Suppose:

```text
Breeding A = Confirmed
```

then later:

```text
Breeding B = Failed
```

The system must determine which is the latest valid reproductive event before changing:

```text
Animal.reproductiveStatus
```

Never blindly update the animal based on an old Breeding record.

---

# 45. Breeding and Pregnancy Record Creation

Recommended logic:

```text
Breeding confirmed
        ↓
Check existing Pregnancy linked by breedingId
        ↓
If none → create Pregnancy
If exists → prevent duplicate
```

This avoids:

```text
One breeding
→ multiple accidental pregnancy records
```

---

# 46. `remarks`

### Type

String

### Required

Optional

### Purpose

Stores additional reproductive information not represented by dedicated fields.

Examples:

```text
Heat signs were clearly observed.
```

```text
Natural mating occurred overnight.
```

```text
Embryo transfer details recorded here under current schema.
```

```text
Pregnancy check was performed later than scheduled.
```

---

# 47. `createdBy`

### Type

ObjectId / User or FarmMember reference

### Required

Yes

### Purpose

Identifies who created the Breeding record.

---

# 48. `createdAt`

### Type

Date

### Required

Automatic

### Purpose

Stores when the Breeding record was created.

---

# 49. `updatedAt`

### Type

Date

### Required

Automatic

### Purpose

Stores when the Breeding record was last updated.

---

# 50. Breeding Creation Workflow

Recommended:

```text
Select Farm
      ↓
Select Female Animal
      ↓
Validate farm ownership
      ↓
Validate reproductive eligibility
      ↓
Record heat detection
      ↓
Select breeding method
      ↓
Record insemination/mating date
      ↓
Record bull/semen/technician as applicable
      ↓
Set expected pregnancy-check date
      ↓
pregnancyResult = Pending
      ↓
Create Breeding
      ↓
AuditLog
```

---

# 51. Pregnancy Check Workflow

```text
Breeding
   ↓
expectedPregnancyCheckDate reached
   ↓
Notification
   ↓
Pregnancy check
   ↓
Result
   ├── Confirmed
   │      ↓
   │   Create Pregnancy
   │      ↓
   │   Animal = Pregnant
   │
   └── Failed
          ↓
       Animal = Open
```

---

# 52. Breeding + Notification

Useful notification:

```text
Pregnancy Check Due
```

Example:

```text
Animal:
AN-000125

Breeding:
BRD-000015

Pregnancy check:
Tomorrow
```

Notification can be associated with the reproductive workflow.

The current Notification model includes `Pregnancy` as a supported reference type. fileciteturn15file7

---

# 53. Breeding + Pregnancy Notification

After pregnancy confirmation:

```text
Pregnancy created
        ↓
expectedCalvingDate
        ↓
future calving reminder
```

The Pregnancy model therefore becomes the source for later pregnancy/calving notifications.

---

# 54. Breeding + Animal Eligibility

Recommended checks before breeding:

```text
Animal exists
Animal belongs to farm
Animal.gender = Female
Animal.lifeStatus = Active
Animal.isActive = true
```

Also check:

```text
Current reproductive state
```

to avoid breeding an already pregnant animal.

---

# 55. Breeding + Life Status

If:

```text
Animal.lifeStatus = Dead
```

block new breeding.

If:

```text
Animal.lifeStatus = Sold
```

block new breeding.

If:

```text
Animal.lifeStatus = Transferred
```

farm ownership/context must be checked.

Historical breeding records should remain.

---

# 56. Breeding + Gender

For the breeding `animalId`:

```text
gender = Female
```

should normally be required.

For:

```text
bullId
```

if provided:

```text
gender = Male
```

should be required.

This is one of the most important validation rules.

---

# 57. Breeding + Purpose

Animal purpose may be:

```text
Dairy
Beef
Breeding
Dual Purpose
```

Breeding is most naturally associated with animals whose farm purpose supports reproduction.

However, purpose alone should not be the only eligibility criterion.

Use:

```text
gender
+
life status
+
reproductive status
+
farm breeding policy
```

---

# 58. Breeding + Reproductive Status

Possible workflow:

```text
Heifer
   ↓
Eligible breeding
   ↓
Breeding
   ↓
Pregnancy confirmed
   ↓
Pregnant
   ↓
Delivered
   ↓
Lactating / Open
```

The exact transition depends on the farm's reproduction policy.

---

# 59. Multiple Breeding Attempts

An animal can have multiple breeding records.

Example:

```text
BRD-001
Aug 18
AI
Failed

BRD-002
Sep 15
AI
Confirmed
```

This is legitimate.

Therefore:

```text
animalId
```

must not be unique in Breeding.

---

# 60. Failed Breeding and Rebreeding

If:

```text
pregnancyResult = Failed
```

the system can allow a new breeding event once the animal is again eligible.

The old Breeding record remains:

```text
Failed
```

for historical analysis.

---

# 61. Duplicate Breeding Prevention

Potential accidental duplicate:

```text
Same animal
Same heatDetectedDate
Same inseminationDate
Same method
Same semenBatchNo
```

The system should warn.

But do not enforce a permanent uniqueness constraint because legitimate repeated attempts can occur.

---

# 62. Breeding History

Animal profile can show:

```text
Breeding History

BRD-000001
Heat: Aug 17
AI: Aug 18
Result: Failed

BRD-000002
Heat: Sep 14
AI: Sep 15
Result: Confirmed
```

This is valuable for:

```text
Reproductive performance
Farm management
Research
```

---

# 63. Breeding Analytics

Possible metrics:

```text
Total breeding attempts
Successful breeding attempts
Failed breeding attempts
Pregnancy confirmation rate
AI success rate
Natural mating success rate
Breeding attempts per animal
Average attempts before pregnancy
```

---

# 64. Pregnancy Confirmation Rate

Conceptually:

```text
Confirmed Breeding Attempts
---------------------------- × 100
Eligible Evaluated Breeding Attempts
```

Example:

```text
Confirmed = 80
Evaluated = 100

Rate = 80%
```

Only evaluated/closed breeding attempts should be included in the denominator.

Pending records should normally be excluded.

---

# 65. Method-Wise Breeding Analytics

Because:

```text
breedingMethod
```

is stored, Liventra can compare:

```text
Artificial Insemination
Natural Mating
Embryo Transfer
```

by:

```text
attempts
confirmed
failed
pending
```

This can support farm decision-making.

---

# 66. Bull/Semen Analysis

If:

```text
bullId
```

is available:

```text
Breeding
   ↓
bullId
```

can support:

```text
Bull-wise breeding outcomes
```

If:

```text
semenBatchNo
```

is available:

```text
Semen batch
   ↓
Breeding outcomes
```

can support batch-level analysis.

---

# 67. Important Limitation — No Separate Semen Master

The current schema stores:

```text
semenBatchNo
```

as a string.

There is no:

```text
Semen
```

master collection.

Therefore, do not introduce one in the current finalized schema.

If later the project requires:

```text
supplier
storage
expiry
price
bull genetics
batch inventory
```

then a dedicated semen inventory/master model would be a future enhancement.

---

# 68. Important Limitation — Bull Is an Animal

The finalized model explicitly says:

```text
bullId (AnimalID)
```

Therefore the bull is expected to be an existing Animal record.

This gives:

```text
Breeding
   ↓
bullId
   ↓
Animal
```

The backend should ensure that referenced animal is male.

---

# 69. Breeding + Inventory

The current Breeding model does not directly contain:

```text
cost
inventoryId
```

Therefore, breeding itself does not automatically create inventory transactions.

If Artificial Insemination uses semen/material inventory later, the current model would need an explicit integration decision.

Do not silently add inventory behavior without a corresponding inventory reference/source.

---

# 70. Breeding + Expense

The current Breeding model does not contain:

```text
cost
```

Therefore, breeding-related financial records should be handled through:

```text
Expense
```

if the farm actually incurs a breeding service cost.

The finalized Expense model supports:

```text
Breeding-related services through its expense categories/reference structure where applicable.
```

fileciteturn15file11

---

# 71. Breeding + Income

The finalized Income model includes:

```text
Breeding Service
```

as an income category.

Therefore, if the farm provides breeding services to others:

```text
Breeding
   ↓
Income
```

can be linked through the Income module.

The current Breeding model itself does not store the financial transaction.

---

# 72. Breeding + Milk

Breeding has a strong reproductive-production relationship:

```text
Breeding
   ↓
Pregnancy
   ↓
Calving
   ↓
Lactation
   ↓
MilkRecord
```

But Breeding should not directly store milk data.

Milk production remains in:

```text
MilkRecord
```

---

# 73. Breeding + Pregnancy + Milk Timeline

Example:

```text
Aug 18
Breeding

Sep 18
Pregnancy Confirmed

Expected Calving
May/Jun

Actual Calving
Recorded in Pregnancy

After calving
Lactation
↓
MilkRecord
```

This gives Liventra a complete reproductive-to-production timeline.

---

# 74. Breeding + Weight

Breeding itself does not store weight.

Animal weight history remains in:

```text
WeightRecord
```

However, breeding eligibility may use current animal information such as age/weight in the application layer.

The current Breeding model does not store a weight snapshot.

---

# 75. Breeding + Health

Before breeding, the system may consider:

```text
Animal healthStatus
```

A severely sick or quarantined animal may not be eligible for breeding according to farm/veterinary policy.

The current schema does not store a dedicated:

```text
breedingEligibilityStatus
```

Therefore, this remains business/service-layer logic.

---

# 76. Breeding + Treatment

Treatment and Breeding are separate events.

Example:

```text
Animal
   ├── Treatment
   └── Breeding
```

A reproductive condition may lead to Treatment, but Treatment should not be embedded inside Breeding.

Both appear in the animal's overall timeline.

---

# 77. Breeding + AuditLog

Important audit events:

```text
Breeding Created
Breeding Updated
Method Changed
Bull Changed
Semen Batch Changed
Pregnancy Check Date Changed
Pregnancy Result Changed
Breeding Corrected
```

This is especially important because reproductive history can affect:

```text
Pregnancy
Calving
Milk production
Animal breeding decisions
```

---

# 78. Breeding + Notification

Useful notifications:

```text
Pregnancy Check Due
Pregnancy Check Overdue
Pregnancy Confirmed
Breeding Failed
Expected Calving Reminder
```

The first two are directly associated with the Breeding workflow.

Later calving reminders should originate from Pregnancy because the finalized Pregnancy model stores `expectedCalvingDate`.

---

# 79. Breeding + Reports

Possible reports:

```text
Breeding History Report
Breeding Performance Report
Pregnancy Success Report
AI Success Report
Bull-wise Breeding Report
Semen Batch Analysis
Animal Reproductive History
```

The Report module supports broader report categories such as Animal and Health; detailed reproductive report naming can be handled through report filters/implementation conventions without changing the current Breeding schema. fileciteturn15file7

---

# 80. Breeding + Animal Health Timeline

An animal's timeline can combine:

```text
Weight
Milk
Vaccination
Treatment
Breeding
Pregnancy
```

Example:

```text
Aug 10 → Weight
Aug 15 → Heat detected
Aug 18 → Breeding
Sep 18 → Pregnancy confirmed
```

This is one of the main benefits of keeping event collections separate.

---

# 81. Authorization

Recommended:

```text
Farm Owner
Farm Manager
Authorized Worker
Authorized reproductive technician
```

depending on the final role-permission matrix.

The user must belong to the relevant farm.

---

# 82. `createdBy` Validation

Before creating:

```text
createdBy
```

must be:

```text
valid user
+
authorized farm member
+
permission to create breeding records
```

This prevents unauthorized reproductive data entry.

---

# 83. Technician Traceability

The current field:

```text
technicianName
```

provides basic traceability.

However, because it is a string, the system cannot guarantee that the technician is an actual registered farm member.

Therefore:

```text
technicianName
=
display/recorded name
```

not a strong identity reference.

This is a current model limitation.

---

# 84. Recommended Date Consistency

The following should normally hold:

```text
heatDetectedDate
        <=
inseminationDate
        <=
expectedPregnancyCheckDate
```

Example:

```text
Heat:
Aug 17

Insemination:
Aug 18

Pregnancy check:
Sep 18
```

If these relationships are violated, the system should reject or flag the record.

---

# 85. Pregnancy Result Timing

A Breeding record should normally remain:

```text
Pending
```

until the pregnancy check.

Then:

```text
Confirmed
```

or:

```text
Failed
```

This prevents premature outcome recording.

---

# 86. Cannot Confirm Too Early

The system should not allow:

```text
Breeding:
Aug 18

Expected pregnancy check:
Sep 18

pregnancyResult:
Confirmed
```

on Aug 18 unless the application explicitly records an externally verified result.

The normal workflow is:

```text
Breeding
→ wait/check
→ result
```

---

# 87. Pending Overdue Logic

If:

```text
pregnancyResult = Pending
```

and:

```text
expectedPregnancyCheckDate < today
```

then the system can mark the breeding event as:

```text
Pregnancy check overdue
```

through Notification/alert logic.

Do not automatically change:

```text
Pending → Failed
```

without an actual result.

---

# 88. Confirmed Workflow

When:

```text
pregnancyResult = Confirmed
```

perform:

```text
Validate no conflicting active pregnancy
       ↓
Create/update Pregnancy
       ↓
Update Animal.reproductiveStatus = Pregnant
       ↓
AuditLog
       ↓
Schedule pregnancy/calving reminders
```

---

# 89. Failed Workflow

When:

```text
pregnancyResult = Failed
```

perform:

```text
No Pregnancy record created
       ↓
Animal may become Open
       ↓
Record failed attempt
       ↓
Allow future breeding when eligible
       ↓
AuditLog
```

Do not delete the failed breeding record.

It is valuable historical data.

---

# 90. Duplicate Pregnancy Prevention

When confirmed:

```text
breedingId
```

should be unique in the Pregnancy relationship.

Meaning:

```text
One Breeding
→ at most one Pregnancy record
```

This is a strong integrity rule.

---

# 91. Pregnancy Model Integration

The finalized Pregnancy model contains:

```text
pregnancyNo
farmId
animalId
breedingId
conceptionDate
expectedCalvingDate
actualCalvingDate
calfCount
calfGender
pregnancyStatus
remarks
createdBy
createdAt
updatedAt
```

fileciteturn15file17

Therefore, Breeding is the upstream event and Pregnancy is the downstream reproductive lifecycle.

---

# 92. Conception Date

The Pregnancy model contains:

```text
conceptionDate
```

while Breeding contains:

```text
inseminationDate
```

These should not automatically be assumed to be exactly the same.

The system should use the project's approved conception-date rule/recorded data.

---

# 93. Expected Calving

Breeding itself stores:

```text
expectedPregnancyCheckDate
```

Pregnancy stores:

```text
expectedCalvingDate
```

Therefore:

```text
Breeding
→ pregnancy check

Pregnancy
→ calving prediction
```

This is a clean separation.

---

# 94. Reproductive State Example

```text
Animal:
reproductiveStatus = Open

       ↓

Heat detected

       ↓

Breeding:
pregnancyResult = Pending

       ↓

Pregnancy check

       ↓

Confirmed

       ↓

Pregnancy created:
pregnancyStatus = Pregnant

       ↓

Animal:
reproductiveStatus = Pregnant
```

---

# 95. Delivery Flow

Later:

```text
Pregnancy
   ↓
actualCalvingDate recorded
   ↓
pregnancyStatus = Delivered
   ↓
Animal reproductive status updated
   ↓
Calf records created
```

Breeding remains historical:

```text
pregnancyResult = Confirmed
```

It should not be changed to:

```text
Delivered
```

because Delivered belongs to Pregnancy.

---

# 96. Failed Breeding Flow

```text
Breeding
   ↓
Pregnancy check
   ↓
Failed
   ↓
Animal = Open
   ↓
Future heat
   ↓
New Breeding
```

This preserves every attempt.

---

# 97. Breeding Attempt Count

For an animal:

```text
Count Breeding records
where animalId = A001
```

This provides:

```text
Number of breeding attempts
```

Useful for reproductive analytics.

---

# 98. Attempts Before Pregnancy

Example:

```text
Attempt 1 → Failed
Attempt 2 → Failed
Attempt 3 → Confirmed
```

Then:

```text
Attempts before successful pregnancy = 3
```

This is useful for farm performance analysis.

---

# 99. Method Success Rate

For AI:

```text
Confirmed AI attempts
/
Evaluated AI attempts
× 100
```

For Natural Mating:

```text
Confirmed natural attempts
/
Evaluated natural attempts
× 100
```

Pending attempts should normally be excluded.

---

# 100. Bull Success Analysis

If:

```text
bullId
```

is consistently recorded:

```text
Bull
   ↓
Breeding attempts
   ↓
Confirmed / Failed
```

can support:

```text
Bull-wise pregnancy success
```

This should be treated as an analytics feature, not as proof of genetic/clinical superiority.

---

# 101. Semen Batch Analysis

If:

```text
semenBatchNo
```

is consistently recorded:

```text
Batch
   ↓
Breeding attempts
   ↓
Pregnancy results
```

can support:

```text
Batch-wise outcome analysis
```

Again, interpretation should account for farm, animal, timing, and other confounding factors.

---

# 102. Current Model Limitation — No Cost

The current Breeding model does not have:

```text
cost
```

Therefore:

```text
Breeding
```

cannot directly calculate breeding expense from its own fields.

If breeding costs are needed:

```text
Expense
```

should be used.

---

# 103. Current Model Limitation — No Actual Pregnancy Check Date

The current model contains:

```text
expectedPregnancyCheckDate
```

but not:

```text
actualPregnancyCheckDate
```

Therefore, exact audit of when the pregnancy examination occurred is limited.

For the current finalized model:

```text
pregnancyResult
+
remarks
```

can capture the result and contextual information.

A future schema revision could add an actual check date if required.

---

# 104. Current Model Limitation — No Embryo Metadata

For:

```text
Embryo Transfer
```

the model does not have:

```text
donorId
embryoBatchNo
embryoCode
```

Therefore, those details should not be invented.

Use:

```text
remarks
```

for the current implementation if needed.

---

# 105. Current Model Limitation — Technician Is Free Text

Because:

```text
technicianName
```

is a string:

```text
"Rahim"
"Md Rahim"
"Rahim Ahmed"
```

could represent the same person.

This limits identity-level analytics.

For the current finalized schema, keep the field as-is and use consistent entry rules.

---

# 106. Current Model Limitation — No Breeding Status

Unlike some reproductive systems, the current Breeding model does not contain:

```text
Scheduled
Completed
Cancelled
```

status.

Instead, the main outcome field is:

```text
pregnancyResult
```

Therefore, do not invent a separate Breeding status.

---

# 107. CRUD Business Rules

## Create

Requires:

```text
breedingNo
farmId
animalId
heatDetectedDate
breedingMethod
```

and applicable method-specific information.

---

## Read

Authorized farm users can view:

```text
Current breeding
Breeding history
Pregnancy outcomes
```

---

## Update

Authorized users can update:

```text
breeding dates
method
bull
semen batch
technician
pregnancy result
remarks
```

Important changes must be audited.

---

## Delete

Avoid hard deletion after:

```text
Pregnancy
```

has been linked to the Breeding record.

Prefer controlled correction/archive logic if required.

---

# 108. Validation Table

| Field | Validation |
|---|---|
| `_id` | MongoDB ObjectId |
| `breedingNo` | Required, unique |
| `farmId` | Required, valid Farm |
| `animalId` | Required, valid Female Animal |
| `heatDetectedDate` | Required, valid date |
| `inseminationDate` | Conditional/required for completed breeding event |
| `breedingMethod` | Enum |
| `bullId` | Conditional, valid male Animal |
| `semenBatchNo` | Conditional, mainly AI |
| `technicianName` | Conditional/recommended |
| `expectedPregnancyCheckDate` | Conditional/recommended |
| `pregnancyResult` | Enum |
| `remarks` | Optional |
| `createdBy` | Required, authorized user |
| `createdAt` | Automatic |
| `updatedAt` | Automatic |

---

# 109. Recommended Indexes

Useful indexes:

```javascript
breedingNo: 1
farmId: 1
animalId: 1
heatDetectedDate: -1
inseminationDate: -1
pregnancyResult: 1
expectedPregnancyCheckDate: 1
bullId: 1
semenBatchNo: 1
```

Potential compound index:

```javascript
{
    farmId: 1,
    animalId: 1,
    inseminationDate: -1
}
```

Useful for:

```text
Animal breeding history
```

---

# 110. Search and Filtering

Breeding list can filter by:

```text
farm
animal
breeding method
pregnancy result
date range
bull
semen batch
```

Example:

```text
pregnancyResult = Pending
```

shows breeding events awaiting pregnancy confirmation.

---

# 111. Pending Breeding Dashboard

A dashboard can show:

```text
Pending Pregnancy Checks
```

Example:

```text
Animal A
Check: Today

Animal B
Check: Tomorrow

Animal C
Check: Overdue
```

This is operationally useful.

---

# 112. Breeding Calendar

Using:

```text
heatDetectedDate
inseminationDate
expectedPregnancyCheckDate
```

the frontend can provide a reproductive calendar.

This can show:

```text
Heat events
Breeding events
Pregnancy checks
```

Later:

```text
Expected calving
```

comes from Pregnancy.

---

# 113. Audit Strategy

For reproductive records, recommended audit actions include:

```text
Create
Update
Pregnancy Result Change
Bull Change
Semen Batch Change
Date Change
Delete/Archive Attempt
```

If:

```text
pregnancyResult:
Pending → Confirmed
```

the audit should preserve:

```text
oldData
newData
changedFields
```

---

# 114. Transaction Strategy for Confirmation

When pregnancy is confirmed, multiple records may change:

```text
Breeding
Pregnancy
Animal
Notification
AuditLog
```

Recommended:

```text
BEGIN TRANSACTION

Update Breeding
Create Pregnancy
Update Animal reproductiveStatus
Create notification if needed
Create AuditLog

COMMIT
```

If a critical operation fails:

```text
ROLLBACK
```

This prevents:

```text
Breeding = Confirmed
but Pregnancy record missing
```

or:

```text
Pregnancy exists
but Animal still shows Open
```

---

# 115. Important Duplicate Confirmation Rule

If:

```text
Breeding.pregnancyResult = Confirmed
```

and Pregnancy already exists for:

```text
breedingId = same breeding
```

do not create another Pregnancy.

Instead:

```text
Return existing record
```

or:

```text
Reject duplicate creation
```

---

# 116. Confirmation Consistency Rule

If:

```text
Breeding.pregnancyResult = Confirmed
```

then there should normally be:

```text
one corresponding Pregnancy record
```

If:

```text
Breeding.pregnancyResult = Failed
```

there should normally be:

```text
no confirmed Pregnancy linked to that breeding
```

This is an important integrity rule.

---

# 117. Reproductive History Integrity

Never overwrite:

```text
Failed
```

breeding attempts with:

```text
Confirmed
```

just because a later breeding succeeded.

Instead:

```text
BRD-001 → Failed
BRD-002 → Confirmed
```

Both remain.

---

# 118. Example — Artificial Insemination

```javascript
{
    breedingNo: "BRD-000001",

    farmId: "FARM-001",

    animalId: "AN-000125",

    heatDetectedDate: "2026-08-17",

    inseminationDate: "2026-08-18",

    breedingMethod: "Artificial Insemination",

    bullId: "AN-BULL-001",

    semenBatchNo: "SEM-2026-001",

    technicianName: "Example Technician",

    expectedPregnancyCheckDate: "2026-09-18",

    pregnancyResult: "Pending",

    remarks: "Routine breeding record.",

    createdBy: "USER-001"
}
```

Values are illustrative.

---

# 119. Example — Natural Mating

```javascript
{
    breedingNo: "BRD-000002",

    farmId: "FARM-001",

    animalId: "AN-000130",

    heatDetectedDate: "2026-08-20",

    inseminationDate: "2026-08-21",

    breedingMethod: "Natural Mating",

    bullId: "AN-BULL-001",

    semenBatchNo: null,

    technicianName: null,

    expectedPregnancyCheckDate: "2026-09-21",

    pregnancyResult: "Pending"
}
```

---

# 120. Example — Confirmed Breeding

```javascript
{
    breedingNo: "BRD-000001",

    pregnancyResult: "Confirmed"
}
```

Then:

```text
Create Pregnancy
```

with:

```text
pregnancy.breedingId = BRD-000001
```

---

# 121. Example — Failed Breeding

```javascript
{
    breedingNo: "BRD-000002",

    pregnancyResult: "Failed",

    remarks:
        "Pregnancy not confirmed during follow-up."
}
```

The breeding remains in history.

---

# 122. Error Scenario — Male Animal as Breeding Animal

Input:

```text
animalId = male animal
```

Reject.

---

# 123. Error Scenario — Female Animal as Bull

Input:

```text
bullId = female animal
```

Reject.

---

# 124. Error Scenario — Cross-Farm Bull

```text
Breeding.farmId = Farm A
bullId = Animal belonging to Farm B
```

Reject unless the system explicitly supports external bull/genetic-source references, which the current schema does not.

---

# 125. Error Scenario — Natural Mating With Semen Batch

Input:

```text
breedingMethod = Natural Mating
semenBatchNo = SEM-001
```

Recommended:

```text
Reject or clear semenBatchNo
```

because the field is not applicable to natural mating.

---

# 126. Error Scenario — AI Without Required Traceability

Input:

```text
breedingMethod = Artificial Insemination
semenBatchNo = null
```

Recommended:

```text
Warning or validation error
```

depending on the project's policy.

If semen traceability is a required project feature, make it required for AI.

---

# 127. Error Scenario — Pregnancy Confirmed Without Breeding

A Pregnancy record should not be created without:

```text
breedingId
```

because the finalized Pregnancy model explicitly references the Breeding event.

fileciteturn15file17

---

# 128. Error Scenario — Duplicate Pregnancy

If:

```text
Pregnancy exists for breedingId = BRD-001
```

and another confirmation attempts to create another:

```text
Pregnancy
```

reject the duplicate.

---

# 129. Error Scenario — Confirmed While Check Is Not Due

If the system uses scheduled pregnancy checks, a premature confirmation should normally be blocked unless an authorized user records an externally verified result.

Do not automatically mark:

```text
Pending → Confirmed
```

just because a user opened the record.

---

# 130. Error Scenario — Pending but Overdue

If:

```text
pregnancyResult = Pending
```

and:

```text
expectedPregnancyCheckDate < today
```

then:

```text
Notification = Overdue
```

but:

```text
pregnancyResult remains Pending
```

until an actual result is recorded.

---

# 131. Breeding Correction Workflow

If a wrong breeding method was entered:

```text
Wrong record
   ↓
Authorized correction
   ↓
Update Breeding
   ↓
AuditLog
```

If a Pregnancy record has already been created, correction becomes more sensitive:

```text
Breeding
+
Pregnancy
+
Animal state
```

must be reconciled together.

Use a transaction.

---

# 132. Breeding Cancellation

There is no:

```text
status = Cancelled
```

in the finalized Breeding model.

Therefore, do not add a cancellation status.

If an event was entered incorrectly:

```text
Controlled correction
+
remarks
+
AuditLog
```

should be used under the current model.

If cancellation becomes a major requirement, it would require a schema decision.

---

# 133. Breeding and Audit-Friendly History

Breeding records should be treated as historical events.

Therefore:

```text
Do not silently overwrite history.
```

For important corrections:

```text
Old value
+
New value
+
Who changed it
+
When
```

should be available through AuditLog.

---

# 134. Reproductive Timeline

The full reproductive timeline becomes:

```text
Heat Detection
      ↓
Breeding
      ↓
Pregnancy Check
      ↓
Pregnancy
      ↓
Expected Calving
      ↓
Actual Calving
      ↓
Lactation
      ↓
Milk Records
```

This is one of the strongest business workflows in Liventra.

---

# 135. Final Architecture

```text
                         ┌──────────────┐
                         │    Animal    │
                         └──────┬───────┘
                                │
                         heat detected
                                │
                                ▼
                         ┌──────────────┐
                         │   Breeding   │
                         └──────┬───────┘
                                │
                     pregnancyResult
                         ┌──────┴───────┐
                         │              │
                    Confirmed        Failed
                         │              │
                         ▼              ▼
                  ┌─────────────┐    Animal
                  │  Pregnancy  │    = Open
                  └──────┬──────┘
                         │
                   expectedCalving
                         │
                         ▼
                     Delivery
                         │
                         ▼
                      Lactation
                         │
                         ▼
                     MilkRecord
```

---

# 136. Final Business Rules Checklist

```text
[✓] Breeding is an animal-specific reproductive event
[✓] breedingNo should be unique
[✓] farmId is required
[✓] animalId is required
[✓] breeding animal should normally be Female
[✓] animal must belong to farmId
[✓] animal lifeStatus should allow active breeding only
[✓] reproductive eligibility should be validated
[✓] heatDetectedDate is required
[✓] heatDetectedDate should normally precede inseminationDate
[✓] breedingMethod is controlled by enum
[✓] Artificial Insemination can use semenBatchNo
[✓] Natural Mating should identify bullId
[✓] Embryo Transfer is supported by the enum
[✓] bullId references Animal
[✓] bullId should reference a Male Animal
[✓] bull must belong to the relevant farm under current model
[✓] semenBatchNo provides AI traceability
[✓] technicianName is currently free-text
[✓] expectedPregnancyCheckDate supports reminders
[✓] pregnancyResult values are Pending, Confirmed, Failed
[✓] Pending should not automatically become Failed when overdue
[✓] Confirmed should create/link Pregnancy
[✓] Failed should preserve the breeding attempt
[✓] One breeding should have at most one linked Pregnancy
[✓] Breeding.pregnancyResult and Pregnancy.pregnancyStatus are different concepts
[✓] Animal.reproductiveStatus should reflect the latest valid reproductive event
[✓] Old breeding records must not overwrite newer reproductive state
[✓] Multiple breeding attempts are legitimate
[✓] Duplicate accidental records should be detected/warned
[✓] Breeding itself does not directly store cost
[✓] Breeding itself does not directly consume inventory in the current model
[✓] Breeding-related financial activity belongs to Expense/Income
[✓] Pregnancy handles expected/actual calving
[✓] MilkRecord handles later milk production
[✓] Important reproductive changes should be audited
[✓] Confirmation + Pregnancy + Animal updates should be transaction-safe
[✓] No separate Bull model is required
[✓] No separate Technician model is required
[✓] No separate Semen model is required in the current schema
[✓] Current model has no cancellation status
[✓] Current model has no actualPregnancyCheckDate
[✓] Current model has no dedicated embryo metadata
```

---

# 137. Final Status

**Model:** `Breeding`  
**Status:** ✅ Finalized  
**Business Logic:** ✅ Documented  
**Animal Integration:** ✅  
**Pregnancy Integration:** ✅  
**Reproductive State Integration:** ✅  
**Notification Integration:** ✅  
**Analytics:** ✅  
**Reporting:** ✅  
**Auditability:** ✅  
**Historical Integrity:** ✅  

**Next Model:** `Pregnancy`
