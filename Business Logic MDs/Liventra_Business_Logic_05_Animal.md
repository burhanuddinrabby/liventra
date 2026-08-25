# Liventra — Business Logic Documentation
## Model 05: Animal

**Document Version:** Business Logic v1.0  
**Project:** Liventra — Smart Livestock Farm Management System  
**Database:** MongoDB  
**Architecture:** MERN

---

# 1. Source Model

The finalized `Animal` model contains:

```text
_id
farmId
breedId
animalCode (auto-generated)
rfid
qrCode
tagNumber
name
gender: Male | Female
purpose: Dairy | Beef | Breeding | Dual Purpose
color
birthDate
purchaseDate
purchasePrice
supplierId
currentWeight
profilePhoto
gallery: [{
    url
    caption
    uploadedAt
}]
location: {
    shed
    pen
    stall
}
remarks
lifeStatus: Active | Sold | Dead | Transferred | Archived
healthStatus: Healthy | Sick | Under Treatment | Quarantine
reproductiveStatus: Calf | Heifer | Open | Pregnant | Lactating | Dry
createdBy
updatedBy
isActive
createdAt
updatedAt
```

This structure is supported by the finalized DB model document. fileciteturn6file0L35-L65

---

# 2. Purpose

`Animal` is the **central operational entity** of Liventra.

Almost every livestock-related module ultimately connects to an individual animal.

```text
                    Animal
                      |
       +--------------+--------------+
       |              |              |
       ↓              ↓              ↓
     Feed           Health        Production
       |              |              |
       ↓              ↓              ↓
FeedRecord       Vaccination      MilkRecord
FeedRecommendation Treatment       WeightRecord
FeedFormula      Disease
                 Pregnancy
                 Breeding
                      |
                      ↓
                    IoT
                      |
                      ↓
                 SensorData
```

The Animal model represents the **current master profile/state** of an animal.

Historical events should be stored in their respective transaction/history collections.

---

# 3. Why Animal Is the Core Model

An individual cattle animal has a lifecycle.

Example:

```text
Birth / Purchase
       ↓
Registration
       ↓
Growing
       ↓
Breeding / Dairy / Beef
       ↓
Health & Feeding Management
       ↓
Production Monitoring
       ↓
Possible Transfer / Sale / Death
       ↓
Historical Record
```

Liventra needs to know both:

1. **What is this animal now?**
2. **What happened to this animal over time?**

Therefore:

```text
Animal
   ↓
Current state/profile

Other Records
   ↓
Historical events
```

---

# 4. Core Business Principle

The `Animal` document should contain the animal's **current master state**.

Do not use Animal to store every historical event.

For example:

```text
Animal.currentWeight
```

represents the latest known/current weight.

Historical weights belong to:

```text
WeightRecord
```

Similarly:

```text
Animal.healthStatus
```

represents current health state.

Historical health events belong to:

```text
Vaccination
Treatment
Disease
```

---

# 5. Animal Lifecycle

The primary lifecycle is represented by:

```text
lifeStatus
```

Conceptually:

```text
Animal Registered
       ↓
Active
       ↓
+------+--------+-------------+
|      |        |             |
↓      ↓        ↓             ↓
Sold  Dead  Transferred   Archived
```

The exact transitions must be controlled by business rules.

---

# 6. Field-by-Field Business Logic

## 6.1 `_id`

### Type
MongoDB `ObjectId`

### Required
Yes

### Purpose

Internal primary identifier.

### Used By

Many operational collections reference the Animal using this ID.

---

# 6.2 `farmId`

### Type
ObjectId / Farm reference

### Required
Yes

### Purpose

Identifies the farm currently responsible for the animal.

### Example

```text
FARM-000001
```

### Critical Rule

Every Animal belongs to exactly one current farm.

This field is also essential for multi-farm data isolation.

---

# 6.3 `breedId`

### Type
ObjectId / Breed reference

### Required
Yes

### Purpose

Connects the animal to the standardized Breed master data.

```text
Animal
   ↓
breedId
   ↓
Breed
```

### Business Uses

- Breed display
- Breed-based analytics
- Feed formula selection
- Recommendation generation
- Production comparison

---

# 6.4 `animalCode`

### Type
String

### Required
Yes

### Unique
Yes within the required system/farm scope

### Generated
System

### Example

```text
AN-000125
```

### Purpose

Human-readable permanent identifier for the animal.

### Used In

- Animal profile
- Search
- Reports
- QR code
- Printed records
- Farmer communication

### Important Rule

Once assigned, the Animal Code should normally never change, even if:

```text
Animal is transferred
Animal is sold
Animal changes location
```

Historical references depend on stable identification.

---

# 6.5 `rfid`

### Type
String

### Required
Optional

### Example

```text
RFID-8800123456
```

### Purpose

Electronic identification tag associated with the animal.

### Business Logic

RFID enables:

```text
RFID Scan
   ↓
Find Animal
   ↓
Open Animal Profile
```

### Important Rule

RFID should be unique within the system/farm according to the project's identification policy.

### Optional Nature

Not every farm may have RFID infrastructure.

Therefore:

```text
rfid = null
```

is valid.

### Future Integration

Supports future:

- RFID readers
- Automated animal identification
- Smart weighing
- Feeding station integration

The project's SRS identifies RFID integration as a future scope item. fileciteturn6file17

---

# 6.6 `qrCode`

### Type
String

### Required
Optional

### Example

```text
https://liventra.app/animals/AN-000125
```

or a generated QR identifier.

### Purpose

Provides low-cost visual identification.

Typical workflow:

```text
Scan QR
   ↓
Authenticate
   ↓
Open Animal Profile
```

### Important Security Rule

The QR code should not expose private farm information directly.

It should point to a protected application route or safe public identifier.

---

# 6.7 `tagNumber`

### Type
String

### Required
Optional

### Example

```text
TAG-125
```

### Purpose

Physical/manual identification number attached to the animal.

### Why RFID + QR + Tag?

They solve different problems:

```text
Tag
 ↓
Human/manual identification

QR
 ↓
Phone-camera identification

RFID
 ↓
Electronic/automated identification
```

A farm may use one, two, or all three.

---

# 6.8 `name`

### Type
String

### Required
Optional

### Example

```text
Maya
```

### Purpose

Farmer-friendly animal name.

### Business Logic

An animal can be identified by:

```text
animalCode
RFID
QR
tagNumber
name
```

`animalCode` remains the primary system identity; name is a human-friendly label.

---

# 6.9 `gender`

### Type
Enum

### Required
Yes

### Allowed Values

```text
Male
Female
```

### Business Logic

Gender affects:

- Breeding eligibility
- Reproductive management
- Milk-production applicability
- Animal purpose
- Feed/management decisions

Example:

```text
Male + Lactating
```

would be an invalid business state.

---

# 6.10 `purpose`

### Type
Enum

### Required
Yes

### Allowed Values

```text
Dairy
Beef
Breeding
Dual Purpose
```

### Purpose

Defines the operational purpose of this individual animal.

### Important Distinction

This is different from:

```text
Breed.purpose
```

Breed purpose is a breed-level reference characteristic.

Animal purpose is the farm's operational classification for this particular animal.

Example:

```text
Breed:
Holstein Friesian
Purpose:
Dairy

Animal:
Maya
Purpose:
Dairy
```

But a breed generally associated with dual-purpose use may be assigned differently for a specific animal depending on farm management goals.

---

# 6.11 `color`

### Type
String

### Required
Optional

### Example

```text
Black and White
```

### Purpose

Physical identification and profile information.

### Used For

- Animal search
- Profile
- Visual identification
- Reports

---

# 6.12 `birthDate`

### Type
Date

### Required
Optional depending on whether known

### Example

```text
2024-03-15
```

### Purpose

Stores date of birth.

### Used For

- Age calculation
- Growth stage
- Feed recommendation
- Breeding readiness
- Vaccination eligibility
- Sale-readiness analysis

### Important Rule

Do not permanently store `age` as a separate field.

Calculate:

```text
Current Date - birthDate
```

when needed.

This avoids stale age values.

---

# 6.13 `purchaseDate`

### Type
Date

### Required
Optional

### Example

```text
2026-02-10
```

### Purpose

Stores when the farm acquired the animal.

### Business Uses

- Purchase history
- Financial analysis
- Animal ownership history
- ROI calculations
- Reports

---

# 6.14 `purchasePrice`

### Type
Number

### Required
Optional

### Example

```text
185000
```

### Unit

BDT.

### Purpose

Stores the purchase price paid by the farm for the animal.

### Business Uses

- Investment tracking
- Financial reports
- Animal profitability
- Sale profit calculation

### Validation

Should not be negative.

---

# 6.15 `supplierId`

### Type
Reference/ObjectId

### Required
Optional

### Purpose

Identifies the supplier/source from whom the animal was purchased.

### Business Logic

If supplier information is available:

```text
Animal
   ↓
supplierId
   ↓
Supplier
```

The current finalized model contains the reference, while a separate Supplier master is not part of the finalized collection list. Therefore the backend should keep this reference design consistent with whatever supplier-management implementation is chosen.

---

# 6.16 `currentWeight`

### Type
Number

### Required
Optional

### Unit

kg, according to the project's normal livestock weight convention.

### Example

```text
230
```

### Purpose

Stores the latest/current known body weight.

### Critical Business Rule

This is a **snapshot**, not the historical weight database.

Historical measurements belong to:

```text
WeightRecord
```

Workflow:

```text
Weight measured
      ↓
WeightRecord created
      ↓
Animal.currentWeight updated
```

This gives:

```text
Fast current lookup
+
Complete historical history
```

---

# 6.17 `profilePhoto`

### Type
String

### Required
Optional

### Example

```text
/uploads/animals/AN-000125/profile.jpg
```

### Purpose

Primary profile image.

### Used In

- Animal card
- Animal profile
- Search results
- Reports

---

# 6.18 `gallery`

### Type
Array of Embedded Objects

### Structure

```javascript
gallery: [
    {
        url,
        caption,
        uploadedAt
    }
]
```

### Purpose

Stores additional animal photographs.

### Example

```javascript
[
    {
        url: "/uploads/animals/AN-000125/front.jpg",
        caption: "Front view",
        uploadedAt: "2026-08-17"
    },
    {
        url: "/uploads/animals/AN-000125/side.jpg",
        caption: "Side view",
        uploadedAt: "2026-08-18"
    }
]
```

### Business Logic

Gallery can be used to visually track:

- Growth
- Body condition
- Injury
- Appearance
- Identification

### Important

`profilePhoto` = primary image.

`gallery` = supporting image history.

---

# 6.19 `location`

### Type
Embedded Object

### Structure

```javascript
location: {
    shed,
    pen,
    stall
}
```

### Example

```javascript
{
    shed: "Shed-A",
    pen: "Pen-03",
    stall: "Stall-12"
}
```

### Purpose

Stores the animal's current physical location inside the farm.

### Business Uses

- Animal finding
- Worker task assignment
- Feeding operations
- Health visits
- Farm layout
- Dashboard filtering

### Critical Rule

This is **current location**, not location history.

If historical movement tracking is later required, a separate movement/history mechanism should be introduced rather than overwriting historical location data.

---

# 6.20 `remarks`

### Type
String

### Required
Optional

### Example

```text
Recently transferred to Shed B for observation.
```

### Purpose

General notes that do not fit into structured fields.

### Business Rule

Do not use remarks as a replacement for structured records.

For example, do not write:

```text
"Vaccinated today"
```

instead of creating a Vaccination record.

---

# 6.21 `lifeStatus`

### Type
Enum

### Required
Yes

### Allowed Values

```text
Active
Sold
Dead
Transferred
Archived
```

This represents the animal's lifecycle state.

---

## Active

Animal currently belongs to and is operationally managed by the farm.

```text
lifeStatus = Active
```

Normal operations are allowed.

---

## Sold

Animal has been sold.

```text
lifeStatus = Sold
```

New routine feeding/health/production transactions should generally be blocked.

Historical data remains.

---

## Dead

Animal has died.

```text
lifeStatus = Dead
```

New routine operational transactions should be blocked.

Historical health/production information remains available.

---

## Transferred

Animal has been transferred away from the current farm.

```text
lifeStatus = Transferred
```

The animal should no longer appear as an active farm animal.

If the system later supports inter-farm transfer, the receiving farm's ownership/reference workflow must be handled carefully.

---

## Archived

Animal is retained for historical/reference purposes but is no longer part of normal operations.

---

# 6.22 `healthStatus`

### Type
Enum

### Required
Yes

### Allowed Values

```text
Healthy
Sick
Under Treatment
Quarantine
```

### Purpose

Represents the animal's **current health state**.

### Business Logic

Example:

```text
Healthy
   ↓
Sick
   ↓
Under Treatment
   ↓
Healthy
```

or:

```text
Healthy
   ↓
Quarantine
   ↓
Healthy
```

### Important

HealthStatus is current state.

Historical medical events belong to:

```text
Disease
Vaccination
Treatment
```

---

# 6.23 `reproductiveStatus`

### Type
Enum

### Required
Yes

### Allowed Values

```text
Calf
Heifer
Open
Pregnant
Lactating
Dry
```

### Purpose

Represents the animal's current reproductive/physiological state as defined by the project.

### Business Logic

This status affects:

- Feed recommendation
- Breeding management
- Pregnancy monitoring
- Milk-production management

### Important Consistency Rules

Some states have biological/business relationships that should be validated.

Examples:

```text
Pregnant
→ Female

Lactating
→ Normally Female

Calf
→ Young animal stage
```

The exact transition rules should be implemented in the service layer.

---

# 6.24 `createdBy`

### Type
Reference to User

### Required
Yes

### Purpose

Identifies who created the Animal record.

### Example

```text
USR-2026-00005
```

### Used For

- Accountability
- Audit
- Record attribution

---

# 6.25 `updatedBy`

### Type
Reference to User

### Required
Optional

### Purpose

Identifies who last modified the Animal record.

### Example

```text
USR-2026-00003
```

---

# 6.26 `isActive`

### Type
Boolean

### Required
Yes

### Default

```text
true
```

### Purpose

Controls whether the Animal record is currently active in normal application queries.

### Important Distinction

`isActive` and `lifeStatus` are not duplicates.

Example:

```text
lifeStatus = Sold
isActive = false
```

means the animal is historically retained but not active in routine operations.

---

# 6.27 `createdAt`

### Type
Date

### Required
Yes

### Generated
System

---

# 6.28 `updatedAt`

### Type
Date

### Required
Yes

### Generated
System

---

# 7. Animal Registration Workflow

```text
Authorized Farm Member
        ↓
Open Add Animal
        ↓
Select Farm
        ↓
Select Breed
        ↓
Enter Animal Information
        ↓
Validate identification
        ↓
Generate animalCode
        ↓
Optional RFID / QR / tag
        ↓
Set initial status
        ↓
Create Animal
        ↓
AuditLog
        ↓
Animal available in dashboard
```

Recommended initial state:

```text
lifeStatus = Active
healthStatus = Healthy
isActive = true
```

unless the actual animal condition requires a different valid state.

---

# 8. Identification Logic

Liventra supports multiple identification mechanisms:

```text
Animal Code
      +
Tag Number
      +
QR Code
      +
RFID
```

### Animal Code

System identity.

### Tag Number

Physical/manual identifier.

### QR Code

Camera-readable identifier.

### RFID

Electronic identification.

---

# 9. Identification Uniqueness

Recommended constraints:

```text
animalCode → Unique
rfid       → Unique when provided
qrCode     → Unique when provided
tagNumber  → Unique according to farm policy
```

At minimum, the system must prevent ambiguous identification within the same farm.

---

# 10. Current Weight Workflow

When a weight is measured:

```text
Measure Animal
      ↓
Create WeightRecord
      ↓
Update Animal.currentWeight
      ↓
Update updatedBy / updatedAt
      ↓
AuditLog
```

Example:

```text
Old currentWeight = 180 kg

New measurement = 230 kg

WeightRecord = 230 kg
Animal.currentWeight = 230 kg
```

The old 180 kg measurement remains in WeightRecord history.

---

# 11. Feed Recommendation Relationship

Animal is one of the main inputs for FeedRecommendation.

Conceptually:

```text
Animal
  |
  +-- Breed
  +-- Weight
  +-- Age
  +-- Purpose
  +-- Health Status
  +-- Reproductive Status
  +-- Milk Yield
  +-- Target ADG
       |
       ↓
Feed Recommendation
```

Example:

```text
Animal A

Weight: 180 kg
Stage: Growing
Target ADG: 0.8 kg/day
Season: Summer

        ↓

System generates recommendation
```

The resulting recommendation stores an input snapshot so that future changes to the Animal do not rewrite the historical recommendation.

---

# 12. Feed Record Relationship

After a farmer actually provides feed:

```text
FeedRecommendation
       ↓
Farmer confirms/uses recommendation
       ↓
FeedRecord
       ↓
Actual feeding event
```

This distinguishes:

```text
Recommended
vs
Actually Fed
```

which is important for analytics.

---

# 13. Milk Production Relationship

For dairy/lactating animals:

```text
Animal
   ↓
MilkRecord
   ↓
Daily/session milk production
```

Animal itself should not permanently store total historical milk.

Current dashboard values should be calculated from MilkRecord.

---

# 14. Weight/Growth Relationship

```text
Animal
   ↓
WeightRecord
   ↓
Growth History
```

This enables:

- Growth chart
- ADG analysis
- Feed effectiveness
- Sale readiness
- Breed comparison

---

# 15. Health Relationship

Animal connects to:

```text
Disease
Vaccination
Treatment
```

Example:

```text
Animal
  ↓
Sick
  ↓
Disease identified
  ↓
Treatment
  ↓
HealthStatus = Under Treatment
  ↓
Treatment completed
  ↓
HealthStatus = Healthy
```

---

# 16. Reproductive Relationship

Animal connects to:

```text
Breeding
Pregnancy
```

Example:

```text
Female Animal
      ↓
Heat detected
      ↓
Breeding record
      ↓
Pregnancy confirmation
      ↓
Pregnancy record
      ↓
Pregnant
      ↓
Expected delivery
      ↓
Lactation
```

---

# 17. IoT Relationship

Future smart-device architecture:

```text
Animal
   ↓
DeviceAssignment
   ↓
Device
   ↓
SensorData
```

Example:

```text
Cow A
  ↓
smaXtec Bolus
  ↓
SensorData
  ↓
Temperature / Rumination / Activity
```

The project documentation identifies IoT/Bolus and RFID integration as future scope. fileciteturn6file17

---

# 18. Animal Location Logic

Current location:

```text
Shed
 ↓
Pen
 ↓
Stall
```

Example:

```text
Shed-A
Pen-03
Stall-12
```

When the animal moves:

```text
Old:
Shed-A / Pen-03 / Stall-12

New:
Shed-B / Pen-01 / Stall-04
```

update the current `location`.

If the project later needs historical movement tracking, a dedicated movement-history model/event mechanism should be introduced rather than trying to reconstruct history from the current location field.

---

# 19. Animal Sale Workflow

When an animal is sold:

```text
Active
  ↓
Sale Transaction
  ↓
lifeStatus = Sold
  ↓
isActive = false
  ↓
Stop normal operational transactions
  ↓
Historical records preserved
  ↓
AuditLog
  ↓
Income record where applicable
```

The finalized `Income` model supports an `Animal Sale` category/reference concept. fileciteturn6file19

### Important

Selling the animal should not delete:

- Weight history
- Milk history
- Feed history
- Vaccination history
- Treatment history
- Breeding history
- Audit history

---

# 20. Animal Death Workflow

If an animal dies:

```text
Active
  ↓
Death recorded
  ↓
lifeStatus = Dead
  ↓
isActive = false
  ↓
Normal operational transactions blocked
  ↓
Historical data preserved
  ↓
AuditLog
```

The system should not silently change the animal's historical records.

---

# 21. Animal Transfer Workflow

If an animal leaves the current farm:

```text
Active
  ↓
Transfer Process
  ↓
lifeStatus = Transferred
  ↓
isActive = false
```

If future multi-farm transfer is implemented, ownership/current-farm assignment must be handled through a controlled transaction so that the old and new farm histories remain consistent.

---

# 22. Animal Archive Workflow

Archiving is for historical/non-operational records.

```text
Animal
  ↓
Archived
  ↓
No routine operations
  ↓
Historical visibility retained
```

---

# 23. Health Status Rules

Recommended baseline:

```text
Healthy
Sick
Under Treatment
Quarantine
```

Example:

```text
Healthy
   ↓
Disease detected
   ↓
Sick
   ↓
Treatment begins
   ↓
Under Treatment
   ↓
Treatment completed
   ↓
Healthy
```

Quarantine:

```text
Potential infection / isolation requirement
        ↓
Quarantine
        ↓
Monitoring
        ↓
Healthy / Sick / Under Treatment
```

The final transition should be performed by authorized users based on the farm's health workflow.

---

# 24. Reproductive Status Rules

Recommended conceptual flow:

```text
Calf
  ↓
Heifer
  ↓
Open
  ↓
Pregnant
  ↓
Lactating
  ↓
Dry
  ↓
Open
```

This is a conceptual lifecycle, not a mandatory sequence for every animal.

### Examples

A male animal should not be set to:

```text
Pregnant
Lactating
```

A young calf should not automatically be treated as a lactating adult.

These validations should be enforced in the service/business layer.

---

# 25. Gender + Reproductive Status Validation

Examples of invalid combinations:

```text
Male + Pregnant
Male + Lactating
```

These should be rejected.

Examples of valid combinations:

```text
Female + Pregnant
Female + Lactating
Female + Open
Female + Dry
```

`Calf` and `Heifer` should also be consistent with age/stage rules established by the project's livestock domain rules.

---

# 26. Purpose + Operational Logic

Purpose can influence which modules are relevant.

### Dairy

Prioritize:

```text
Milk
Feed
Lactation
Health
```

### Beef

Prioritize:

```text
Weight
ADG
Feed
Sale readiness
```

### Breeding

Prioritize:

```text
Heat
Breeding
Pregnancy
Health
```

### Dual Purpose

Monitor multiple production objectives.

This does not mean the system should block unrelated records; it mainly influences recommendations, dashboards and workflows.

---

# 27. CRUD Business Rules

## Create

Allowed to authorized farm members.

Must validate:

- Farm access
- Breed
- Identification
- Gender
- Purpose
- Initial statuses
- Purchase information if applicable

---

## Read

Any role with animal-view permission may read the permitted animal information.

Sensitive information should follow role permissions.

---

## Update

Normal updates include:

```text
name
color
location
remarks
profilePhoto
gallery
current status
```

Sensitive/lifecycle changes require stronger authorization:

```text
lifeStatus
farm transfer
sale/death
```

---

## Delete

Hard delete is not recommended.

Use lifecycle/status handling:

```text
Sold
Dead
Transferred
Archived
```

and:

```text
isActive = false
```

where appropriate.

---

# 28. Validation Rules

| Field | Validation |
|---|---|
| farmId | Required, valid Farm |
| breedId | Required, valid Breed |
| animalCode | Unique |
| rfid | Unique when provided |
| qrCode | Unique when provided |
| tagNumber | Unique according to farm policy |
| gender | Enum |
| purpose | Enum |
| birthDate | Valid date |
| purchaseDate | Valid date |
| purchasePrice | Non-negative |
| currentWeight | Non-negative |
| lifeStatus | Enum |
| healthStatus | Enum |
| reproductiveStatus | Enum |
| createdBy | Valid User |
| updatedBy | Valid User |
| isActive | Boolean |

---

# 29. Business Constraints

### Constraint 1

Animal must belong to a valid Farm.

### Constraint 2

Breed must exist.

### Constraint 3

Animal code must be unique.

### Constraint 4

Identification values must not create ambiguity.

### Constraint 5

Inactive/non-operational animals cannot receive normal new operational records.

### Constraint 6

Historical records must remain after sale/death/transfer/archive.

### Constraint 7

Gender and reproductive state must be logically consistent.

### Constraint 8

Current snapshot fields should be synchronized with their source records where applicable.

---

# 30. Notification Rules

Animal-related events may trigger notifications.

Examples:

```text
Vaccination due
Treatment follow-up
Pregnancy due
Expected delivery
Abnormal health status
Low activity
High temperature
Feed recommendation available
Animal sale/transfer
```

The Notification model supports references to Animal and several animal-related entities. fileciteturn6file13

---

# 31. Audit Rules

Important Animal events:

```text
Animal Created
Animal Updated
Identification Changed
RFID Assigned/Changed
QR Updated
Location Changed
Health Status Changed
Reproductive Status Changed
Life Status Changed
Weight Updated
Profile Photo Updated
Animal Sold
Animal Transferred
Animal Archived
Animal Marked Dead
```

For structured updates:

```text
oldData
newData
changedFields
```

should be captured in AuditLog.

---

# 32. Dashboard Usage

The Animal model powers the main livestock dashboard.

Example:

```text
Total Animals: 125

Active: 110
Sold: 8
Dead: 3
Transferred: 2
Archived: 2
```

Health:

```text
Healthy: 96
Sick: 7
Under Treatment: 5
Quarantine: 2
```

Reproductive:

```text
Pregnant: 22
Lactating: 38
Dry: 14
Open: 18
```

These summaries should be calculated from current Animal state.

---

# 33. Search & Filtering

Animal search can use:

```text
animalCode
name
RFID
QR
tagNumber
breed
gender
purpose
lifeStatus
healthStatus
reproductiveStatus
location
```

Example:

```text
Search:
RFID-8800123456

↓

Animal AN-000125
```

Or:

```text
Filter:
Purpose = Dairy
Health = Healthy
Location = Shed-A
```

---

# 34. Reporting Usage

Animal data supports:

```text
Animal Report
Health Report
Feed Report
Weight Report
Milk Report
Breeding Report
Financial Report
```

Examples:

```text
All active dairy cows

All animals under treatment

All pregnant animals

Animals above 400 kg

Animals ready for sale
```

---

# 35. Performance / Index Recommendations

These are implementation recommendations and do not alter the finalized model.

Useful indexes:

```javascript
farmId: 1
animalCode: 1
rfid: 1
qrCode: 1
tagNumber: 1
breedId: 1
lifeStatus: 1
healthStatus: 1
reproductiveStatus: 1
```

Useful compound indexes:

```javascript
{
    farmId: 1,
    lifeStatus: 1
}
```

and:

```javascript
{
    farmId: 1,
    healthStatus: 1
}
```

For farm-level search/filtering, these can significantly improve common queries.

---

# 36. Example Scenario — Registering a Dairy Cow

Input:

```text
Farm:
Green Valley Dairy Farm

Breed:
Holstein Friesian

Animal Code:
AN-000125

Gender:
Female

Purpose:
Dairy

Birth Date:
2024-03-15

Purchase Date:
2026-02-10

Purchase Price:
185000

Current Weight:
480 kg

Location:
Shed-A / Pen-03 / Stall-12
```

Initial state:

```text
lifeStatus = Active
healthStatus = Healthy
reproductiveStatus = Heifer
isActive = true
```

System then:

```text
Create Animal
      ↓
AuditLog
      ↓
Available in Dashboard
      ↓
Can receive Feed / Health / Production records
```

---

# 37. Example Scenario — Weight Update

Current:

```text
Animal.currentWeight = 180 kg
```

New measurement:

```text
230 kg
```

System:

```text
Create WeightRecord
        ↓
Animal.currentWeight = 230
        ↓
AuditLog
```

Historical:

```text
180 kg → previous WeightRecord
230 kg → latest WeightRecord
```

---

# 38. Example Scenario — Health Event

Current:

```text
healthStatus = Healthy
```

Disease detected:

```text
Disease Record
      ↓
healthStatus = Sick
      ↓
Treatment created
      ↓
healthStatus = Under Treatment
      ↓
Treatment completed
      ↓
healthStatus = Healthy
```

---

# 39. Example Scenario — Feed Recommendation

```text
Animal
Weight = 180 kg
Breed = Crossbred
Stage = Growing
Purpose = Beef
Season = Summer

        ↓

FeedFormula matching

        ↓

FeedRecommendation

        ↓

Farmer confirms

        ↓

FeedRecord
```

The recommendation should preserve its input snapshot so that if the Animal later becomes:

```text
230 kg
```

the old recommendation still represents what the system recommended at 180 kg.

---

# 40. Example Scenario — Sale

```text
Animal
AN-000125
Active
   ↓
Farmer sells animal
   ↓
Income record created
   ↓
Animal.lifeStatus = Sold
   ↓
Animal.isActive = false
   ↓
Normal feed/health operations blocked
   ↓
Historical records retained
   ↓
AuditLog
```

---

# 41. Example Scenario — RFID Integration

Future device integration:

```text
RFID Reader
    ↓
RFID = RFID-8800123456
    ↓
Find Animal
    ↓
AN-000125
    ↓
Open profile / record event
```

This supports future automated farm workflows.

---

# 42. Example Scenario — Smart Device

Future:

```text
Animal AN-000125
      ↓
DeviceAssignment
      ↓
smaXtec Bolus
      ↓
SensorData
      ↓
Temperature = 40.2°C
Rumination = Reduced
Activity = Reduced
      ↓
Alert
      ↓
Notification
```

This connects the core Animal model with the future IoT architecture.

---

# 43. Important Data Integrity Rules

The Animal model has several snapshot/current-state fields.

These must remain synchronized.

### Current Weight

```text
WeightRecord
      ↓
Animal.currentWeight
```

### Current Health

```text
Health event / treatment workflow
      ↓
Animal.healthStatus
```

### Reproductive Status

```text
Breeding/Pregnancy workflow
      ↓
Animal.reproductiveStatus
```

### Life Status

```text
Sale/Death/Transfer/Archive workflow
      ↓
Animal.lifeStatus
```

The system should avoid allowing arbitrary frontend edits that can create contradictory states.

---

# 44. What Should NOT Be Stored in Animal

Do not use Animal to store:

```text
All historical weights
All milk records
All vaccination events
All treatments
All breeding attempts
All pregnancy events
All feed records
All sensor readings
```

Those belong to their respective models.

Animal is the **current master profile/state**.

---

# 45. Defense Questions

### Q: Why is Animal the central model?

**Answer:**

> "Animal represents the individual livestock unit being managed by Liventra. Feed, health, production, reproduction and future IoT data are all associated with individual animals, while historical events are maintained in specialized records."

---

### Q: Why do you store currentWeight in Animal if WeightRecord exists?

**Answer:**

> "Animal.currentWeight is a current snapshot for fast access, while WeightRecord stores the complete historical measurements. When a new weight is recorded, the system creates a WeightRecord and updates currentWeight."

---

### Q: Why do you have both lifeStatus and isActive?

**Answer:**

> "`lifeStatus` describes the animal's business lifecycle, such as Active, Sold, Dead or Transferred. `isActive` is an operational flag used to control whether the record participates in normal active queries. For example, a sold animal can have lifeStatus=Sold and isActive=false while its historical data remains available."

---

### Q: Why store RFID, QR and tagNumber separately?

**Answer:**

> "They represent different identification mechanisms. TagNumber supports manual physical identification, QR supports smartphone-based identification, and RFID supports electronic/automated identification. AnimalCode remains the system's primary human-readable identifier."

---

### Q: Why don't you store age?

**Answer:**

> "Age changes continuously, so storing it creates stale data. We store birthDate and calculate age dynamically when required."

---

### Q: Why not delete an animal after it is sold?

**Answer:**

> "Because the animal's historical feed, health, milk, weight, breeding and financial information is valuable for farm analytics and auditing. We change the lifecycle status and retain the historical records."

---

### Q: How does Animal connect with the Feed Recommendation system?

**Answer:**

> "Animal provides contextual inputs such as breed, age, current weight, purpose, reproductive state and production information. These are combined with FeedFormula rules and seasonal information to generate a recommendation. The recommendation stores an input snapshot so historical recommendations remain immutable."

---

# 46. Final Business Rules Checklist

```text
[✓] Animal belongs to one current farm
[✓] Animal references standardized Breed
[✓] Animal has a stable animalCode
[✓] RFID is optional
[✓] QR is optional
[✓] Tag number is optional
[✓] Identification values should not be ambiguous
[✓] Animal purpose is farm-specific operational classification
[✓] Birth date is used to calculate age
[✓] Current weight is a snapshot
[✓] Historical weight is stored separately
[✓] Health status represents current state
[✓] Historical health events are stored separately
[✓] Reproductive status represents current state
[✓] Breeding/Pregnancy preserve reproductive history
[✓] Current location is stored separately from future movement history
[✓] Life status controls animal lifecycle
[✓] Sold/Dead/Transferred/Archived animals remain historically available
[✓] Hard deletion is discouraged
[✓] Animal changes generate AuditLog
[✓] Relevant events can generate Notifications
[✓] Animal is a major input to FeedRecommendation
[✓] Animal supports future RFID/IoT integration
[✓] Farm-level authorization is required for every animal operation
```

---

# 47. Final Status

**Model:** `Animal`  
**Status:** ✅ Finalized  
**Business Logic:** ✅ Documented  
**Database Structure:** 🔒 Frozen  
**Next Model:** `FeedIngredient`
