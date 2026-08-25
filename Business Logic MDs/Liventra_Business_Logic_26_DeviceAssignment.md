# Liventra — Business Logic Documentation
## Model 26: `DeviceAssignment` (Upcoming)

**Project:** Liventra — Smart Livestock Farm Management System  
**Module:** IoT / Smart Device Management  
**Collection Status:** Upcoming

> This document follows the finalized Liventra DB model. The source defines `DeviceAssignment` as an upcoming collection with `assignmentNo`, `farmId`, `deviceId`, `animalId`, assignment/removal dates, assignment/removal reasons, assigned/removed users, status, remarks, and timestamps. fileciteturn25file0turn25file1

---

# 1. Finalized Model

```text
DeviceAssignment:
_id,
assignmentNo,
farmId,
deviceId,
animalId,
assignedDate,
removedDate,

assignmentReason:
    New Installation
    Replacement
    Device Upgrade
    Reassignment
    Testing,

removalReason:
    Maintenance
    Battery Failure
    Device Fault
    Animal Sold
    Animal Dead
    Replacement
    Lost
    Manual Removal,

assignedBy,
removedBy,

status:
    Active
    Removed
    Cancelled,

remarks,
createdAt,
updatedAt
```

The exact finalized enum values above are present in the current DB-model source. fileciteturn25file0turn25file14

---

# 2. What Is `DeviceAssignment`?

`DeviceAssignment` is the collection that records:

> **Which device was assigned to which animal, when the assignment started, why it started, when it ended, why it ended, and who performed the operation.**

It is deliberately separate from `Device`.

Conceptually:

```text
Device
=
physical smart device

DeviceAssignment
=
relationship/history between Device and Animal

SensorData
=
measurements produced by the device
```

The finalized architecture separately defines `Device`, `DeviceAssignment`, and `SensorData`. fileciteturn25file0

---

# 3. Why Do We Need a Separate Collection?

Suppose:

```text
DEV-001
```

is first assigned to:

```text
Animal A-001
```

and later moved to:

```text
Animal A-017
```

If `animalId` were stored directly inside Device:

```text
Device.animalId = A-017
```

the old relationship:

```text
DEV-001 → A-001
```

would disappear.

With `DeviceAssignment`:

```text
Assignment #1
DEV-001 → A-001
```

then:

```text
Assignment #2
DEV-001 → A-017
```

The historical relationship remains preserved.

---

# 4. Main Responsibility

The responsibility of this collection is **assignment history**, not device telemetry.

It answers:

```text
Which device?
Which animal?
Which farm?
When assigned?
Why assigned?
When removed?
Why removed?
Who assigned it?
Who removed it?
Current status?
```

It does **not** store:

```text
temperature
rumination
activity
battery measurements over time
```

Those belong to `SensorData`. fileciteturn25file0

---

# 5. Relationship Overview

```text
Farm
  │
  ├───────────────┐
  ▼               ▼
Device          Animal
  │               │
  └───────┬───────┘
          ▼
   DeviceAssignment
          │
          ▼
      SensorData
```

More precisely:

```text
Device 1 ──────< DeviceAssignment >────── 1 Animal
                    │
                    └────< SensorData
```

One device can have multiple assignment records over time.

---

# 6. `_id`

### Type
MongoDB `ObjectId`

### Required
Yes

### Purpose

Internal unique identifier of the assignment record.

Example:

```text
ObjectId("...")
```

It identifies the assignment event/history record, not the physical device.

---

# 7. `assignmentNo`

### Type
String

### Required
Yes

### Example

```text
ASN-000001
```

### Purpose

Human-readable assignment identifier.

Useful for:

```text
Assignment history
Reports
Search
Audit/reference
Support
```

---

# 8. `assignmentNo` vs `deviceId`

These are different:

```text
assignmentNo
=
identity of this assignment record
```

```text
deviceId
=
identity of the physical device
```

Example:

```text
assignmentNo = ASN-000021
deviceId = DEV-001
```

If DEV-001 is reassigned later:

```text
assignmentNo = ASN-000038
deviceId = DEV-001
```

Same device, different assignment record.

---

# 9. Assignment Number Generation

Recommended application behavior:

```text
ASN-000001
ASN-000002
ASN-000003
...
```

The backend should generate it.

The frontend should not be trusted to generate globally unique assignment numbers.

The finalized model specifies `assignmentNo`, but does not prescribe the exact numbering algorithm; the format above is an implementation recommendation.

---

# 10. `farmId`

### Type
ObjectId → Farm

### Required
Yes

### Purpose

Identifies the farm under which this assignment exists.

Example:

```text
farmId = FARM-001
```

This is important because Liventra supports multiple farms and farm-level access control.

---

# 11. Why `farmId` Exists Even Though Device Has `farmId`

The Device model already contains:

```text
farmId
```

but DeviceAssignment also contains:

```text
farmId
```

This duplication can be useful for:

```text
fast farm-scoped queries
historical context
security filtering
reporting
```

It also provides a denormalized farm context for the assignment record.

---

# 12. Farm Consistency Rule

When creating an assignment:

```text
Device.farmId
==
DeviceAssignment.farmId
```

should normally be true.

And:

```text
Animal.farmId
==
DeviceAssignment.farmId
```

should also normally be true.

This is a business validation recommendation based on the multi-farm architecture.

The finalized model itself does not explicitly state these validation rules.

---

# 13. Cross-Farm Assignment

Under the current model, an assignment should not casually connect:

```text
Device from Farm A
```

to:

```text
Animal from Farm B
```

because that would break farm isolation.

Recommended validation:

```text
Device.farmId == Animal.farmId == Assignment.farmId
```

before creating the assignment.

---

# 14. `deviceId`

### Type
ObjectId → Device

### Required
Yes

### Purpose

References the physical device being assigned.

Example:

```text
deviceId = DEV-001
```

Relationship:

```text
Device._id
      ↓
DeviceAssignment.deviceId
```

The finalized SensorData model also references the Device through `deviceId`. fileciteturn25file0

---

# 15. `animalId`

### Type
ObjectId → Animal

### Required
Yes

### Purpose

Identifies the animal receiving/using the device.

Example:

```text
animalId = ANM-001
```

Relationship:

```text
Animal._id
      ↓
DeviceAssignment.animalId
```

---

# 16. Why `animalId` Is Needed

For animal-specific devices such as:

```text
Bolus
Neck Collar
Ear Tag
```

the system needs to know:

```text
Which animal is this device monitoring?
```

The project's smart-livestock architecture discusses smart collars, ear tags, rumen bolus sensors and related monitoring technologies. fileciteturn25file9turn25file11

---

# 17. Important Limitation: Not Every Device Is Necessarily Animal-Specific

The Device model also contains:

```text
Milk Meter
Environment Sensor
Water Meter
```

These may operate at:

```text
farm level
shed level
equipment level
```

rather than being attached to one animal.

The current `DeviceAssignment` model requires `animalId`, so the current finalized schema is naturally optimized for **animal-linked device assignments**.

This is a design consideration, not a change being made here.

---

# 18. `assignedDate`

### Type
Date

### Required
Yes

### Purpose

Records when the assignment became effective.

Example:

```text
2026-08-17
```

Meaning:

```text
DEV-001 started being assigned to ANM-001 on 17 Aug 2026.
```

---

# 19. Assignment Date vs Device Installation Date

These are different.

### Device

```text
installationDate
```

means:

> When the physical device was installed/deployed.

### DeviceAssignment

```text
assignedDate
```

means:

> When this particular device-to-animal assignment began.

Example:

```text
Device installationDate:
2026-08-01

Assignment assignedDate:
2026-08-05
```

The device could have been installed/tested first and assigned to an animal later.

---

# 20. `removedDate`

### Type
Date

### Required
Optional

### Example

```text
2026-09-15
```

Purpose:

Records when the assignment ended.

For:

```text
status = Active
```

`removedDate` should normally be:

```text
null
```

For:

```text
status = Removed
```

it should normally contain the removal date.

This is an implementation consistency recommendation.

---

# 21. Assignment Timeline

Example:

```text
2026-08-01
assignedDate

        ↓

Device actively monitors Animal A

        ↓

2026-09-15
removedDate

        ↓

Assignment closed
```

The record remains in the database.

It is not replaced by a new assignment record.

---

# 22. `assignmentReason`

### Type
Enum

### Required
Yes

### Finalized Values

```text
New Installation
Replacement
Device Upgrade
Reassignment
Testing
```

fileciteturn25file0turn25file1

---

# 23. `New Installation`

Example:

```text
assignmentReason = New Installation
```

Meaning:

A newly deployed device is being assigned to the animal for the first time.

Example:

```text
New smart collar
+
Animal A-001
```

---

# 24. `Replacement`

Example:

```text
assignmentReason = Replacement
```

Meaning:

An older device was replaced by another device.

Example:

```text
DEV-001
Faulty

↓

DEV-009
Replacement

↓

Animal A-001
```

The old assignment should be closed and a new assignment created.

---

# 25. `Device Upgrade`

Example:

```text
assignmentReason = Device Upgrade
```

Meaning:

The farm is replacing the previous device with a newer/better device as an upgrade.

This is different from:

```text
Replacement
```

because replacement can happen due to failure, while upgrade is an intentional improvement.

The source defines both values but does not prescribe the exact operational distinction; this interpretation is business logic.

---

# 26. `Reassignment`

Example:

```text
DEV-001
Animal A-001
```

becomes:

```text
DEV-001
Animal A-017
```

Then the new assignment:

```text
assignmentReason = Reassignment
```

---

# 27. `Testing`

Example:

```text
DEV-005
→ Animal A-010
```

temporarily for:

```text
Device testing
```

This should not automatically imply that the device is part of the animal's permanent monitoring history.

The application's testing workflow should distinguish temporary testing from operational assignments.

---

# 28. `removalReason`

### Type
Enum

### Required
Required when assignment is removed

### Finalized Values

```text
Maintenance
Battery Failure
Device Fault
Animal Sold
Animal Dead
Replacement
Lost
Manual Removal
```

fileciteturn25file0turn25file14

---

# 29. `Maintenance`

Example:

```text
removalReason = Maintenance
```

Meaning:

The device is being removed from the animal for maintenance.

Possible scenario:

```text
Battery service
Physical inspection
Firmware service
Cleaning/repair
```

---

# 30. `Battery Failure`

Example:

```text
removalReason = Battery Failure
```

Meaning:

The device is removed because its battery has failed.

This is particularly relevant for:

```text
Bolus
Neck Collar
Ear Tag
```

depending on the actual device.

---

# 31. `Device Fault`

Example:

```text
removalReason = Device Fault
```

Meaning:

The device itself has a functional/hardware problem.

This can lead to:

```text
new DeviceAssignment
```

with:

```text
assignmentReason = Replacement
```

---

# 32. `Animal Sold`

Example:

```text
Animal A-001
lifeStatus = Sold
```

Then:

```text
DeviceAssignment
status = Removed
removalReason = Animal Sold
```

The device may then become available for reassignment.

---

# 33. `Animal Dead`

If the animal's lifecycle changes to:

```text
lifeStatus = Dead
```

the device assignment should normally be closed.

Example:

```text
removalReason = Animal Dead
```

This prevents the device from remaining falsely attached to a non-active animal.

---

# 34. `Replacement`

Example:

```text
Old Device:
DEV-001

New Device:
DEV-009
```

Old assignment:

```text
status = Removed
removalReason = Replacement
```

New assignment:

```text
status = Active
assignmentReason = Replacement
```

---

# 35. `Lost`

Example:

```text
Smart collar cannot be located.
```

Then:

```text
status = Removed
removalReason = Lost
```

The physical Device record should not necessarily be deleted because its historical existence remains important.

---

# 36. `Manual Removal`

Used when an authorized user intentionally removes the device and none of the predefined operational reasons is appropriate.

Example:

```text
removalReason = Manual Removal
remarks = "Device removed for farm restructuring."
```

---

# 37. `assignedBy`

### Type
ObjectId → User

### Required
Yes

### Purpose

Identifies the user who created/performed the assignment.

Example:

```text
assignedBy = USER-001
```

This provides accountability.

---

# 38. `removedBy`

### Type
ObjectId → User

### Required
Optional

### Purpose

Identifies who removed/closed the assignment.

Example:

```text
removedBy = USER-002
```

For an active assignment:

```text
removedBy = null
```

would normally be appropriate.

---

# 39. `assignedBy` vs `removedBy`

Example:

```text
assignedBy = Farm Manager
removedBy = Farm Manager
```

or:

```text
assignedBy = Admin
removedBy = Technician
```

This makes the assignment lifecycle traceable.

---

# 40. `status`

### Type
Enum

### Required
Yes

### Values

```text
Active
Removed
Cancelled
```

fileciteturn25file0turn25file1

---

# 41. `Active`

Meaning:

```text
This assignment is currently in effect.
```

Example:

```text
DEV-001
→ Animal A-001
```

currently active.

Recommended state:

```text
removedDate = null
removedBy = null
removalReason = null
```

---

# 42. `Removed`

Meaning:

```text
The assignment existed,
but it has ended.
```

Example:

```text
DEV-001
→ Animal A-001

removedDate = 2026-09-15
status = Removed
```

The historical record remains.

---

# 43. `Cancelled`

Meaning:

```text
The assignment process was cancelled
rather than becoming a completed active-to-removed lifecycle.
```

Example:

```text
Device was scheduled for testing
but testing was cancelled before the assignment became operational.
```

The source defines `Cancelled` but does not specify the exact workflow distinction from `Removed`; this interpretation is an implementation recommendation.

---

# 44. Status Transition

A normal lifecycle is:

```text
Active
   ↓
Removed
```

A cancelled workflow may be:

```text
Pending/created
   ↓
Cancelled
```

The finalized model does not contain a separate `Pending` status, so the exact point at which an assignment document is created should be defined by the backend workflow.

---

# 45. Important Rule: Do Not Reuse an Old Assignment

Suppose:

```text
DEV-001 → Animal A-001
```

was removed.

Later:

```text
DEV-001 → Animal A-017
```

Do **not** update the old record from:

```text
Animal A-001
```

to:

```text
Animal A-017
```

Instead create a new assignment.

This preserves history.

---

# 46. Why Historical Assignment Matters

Suppose SensorData from 10 AM says:

```text
deviceId = DEV-001
assignmentId = ASN-001
animalId = A-001
```

Later the same device is assigned to:

```text
A-017
```

Without historical assignment records, it becomes difficult to understand which animal the old measurements belonged to.

The finalized SensorData model explicitly includes:

```text
deviceId
assignmentId
animalId
```

which supports this historical relationship. fileciteturn25file0

---

# 47. `remarks`

### Type
String

### Required
Optional

### Example

```text
"Collar fitted after replacing damaged strap."
```

Purpose:

Stores additional context that does not fit structured fields.

---

# 48. What Should NOT Go in `remarks`

Do not use remarks for important structured information such as:

```text
device ID
animal ID
assignment date
removal reason
user ID
```

Those already have dedicated fields.

---

# 49. `createdAt`

### Type
Date

### Required
Yes

Purpose:

Records when the assignment document was created.

---

# 50. `updatedAt`

### Type
Date

### Required
Yes

Purpose:

Records the most recent modification to the assignment document.

Example:

```text
removedDate added
→ updatedAt changes
```

---

# 51. Assignment Creation Workflow

Recommended:

```text
User selects Device
        ↓
User selects Animal
        ↓
Backend validates farm ownership
        ↓
Backend checks Device status
        ↓
Backend checks existing active assignment
        ↓
Create DeviceAssignment
        ↓
status = Active
        ↓
assignedDate = current/effective date
        ↓
assignedBy = authenticated user
```

The exact workflow is implementation logic; the source only defines the fields/enums.

---

# 52. Assignment Validation — Device

Before assignment:

```text
Device exists
```

and:

```text
Device.farmId == assignment.farmId
```

and ideally:

```text
Device.isActive == true
```

This is recommended business validation.

---

# 53. Assignment Validation — Animal

Before assignment:

```text
Animal exists
```

and:

```text
Animal.farmId == assignment.farmId
```

and the animal should normally be eligible for device assignment.

For example, an animal with:

```text
lifeStatus = Sold
```

should not receive a new active device assignment.

This is an application-level recommendation based on the finalized Animal lifecycle enum. fileciteturn25file17

---

# 54. Assignment Validation — Device Status

A device in:

```text
connectionStatus = Maintenance
```

may not be appropriate for a normal active assignment.

Likewise:

```text
isActive = false
```

should normally prevent new active assignments.

The finalized Device model contains these states, but it does not explicitly prescribe assignment validation.

---

# 55. Duplicate Active Assignment Problem

Potential bad state:

```text
DEV-001
→ Animal A-001
status = Active

DEV-001
→ Animal A-017
status = Active
```

For a device intended for one animal at a time, this creates ambiguity.

Therefore the backend should enforce:

```text
one active assignment per device
```

for device types that are animal-specific.

This is a business recommendation, not an explicit source constraint.

---

# 56. Duplicate Animal Assignment Problem

Potential bad state:

```text
Animal A-001
← DEV-001
Active

Animal A-001
← DEV-009
Active
```

Whether this is allowed depends on the farm's device policy.

Some animals may legitimately have:

```text
Neck Collar
+
Bolus
+
Ear Tag
```

simultaneously.

Therefore **do not enforce one active device per animal globally**.

Instead, restrictions should depend on device type/use case.

---

# 57. Very Important: Device-to-Animal Cardinality

Potentially:

```text
One Device → One Animal at a time
```

for:

```text
Neck Collar
Bolus
Ear Tag
```

while:

```text
One Animal → Multiple Devices
```

may be valid.

Example:

```text
Animal A-001
├── Ear Tag
├── Neck Collar
└── Bolus
```

This is a much better business interpretation than forcing a single device per animal.

---

# 58. Device Type-Specific Assignment

Conceptually:

```text
Neck Collar
→ animal-specific

Bolus
→ animal-specific

Ear Tag
→ animal-specific
```

while:

```text
Environment Sensor
→ potentially location/farm-level

Water Meter
→ potentially location/farm-level

Milk Meter
→ potentially equipment/milking-point-level
```

The current finalized `DeviceAssignment` requires `animalId`, so only the animal-specific side is fully represented.

---

# 59. New Installation Example

```text
Device:
DEV-001

Animal:
ANM-001

assignmentReason:
New Installation

assignedDate:
2026-08-17

status:
Active
```

This means the device has been newly installed/assigned to the animal.

---

# 60. Reassignment Example

Initial:

```text
ASN-001
DEV-001 → ANM-001
Active
```

Later:

```text
ASN-001
DEV-001 → ANM-001
Removed
removalReason = Manual Removal
```

Then:

```text
ASN-002
DEV-001 → ANM-017
Active
assignmentReason = Reassignment
```

The old record remains untouched.

---

# 61. Replacement Example

Old:

```text
ASN-001
DEV-001 → ANM-001
status = Removed
removalReason = Device Fault
```

New:

```text
ASN-002
DEV-009 → ANM-001
status = Active
assignmentReason = Replacement
```

This is the cleanest historical representation.

---

# 62. Animal Sold Example

Animal:

```text
lifeStatus = Sold
```

Assignment:

```text
status = Removed
removalReason = Animal Sold
removedDate = sale/assignment closure date
removedBy = authorized user
```

The device can then become available for another animal if physically reusable.

---

# 63. Animal Dead Example

Animal:

```text
lifeStatus = Dead
```

Then:

```text
DeviceAssignment
status = Removed
removalReason = Animal Dead
```

This avoids future telemetry being interpreted as belonging to the dead animal through an active assignment.

---

# 64. Device Lost Example

```text
DEV-001
→ Animal A-001
```

If the physical device is lost:

```text
status = Removed
removalReason = Lost
```

The Device record remains historically available.

---

# 65. Testing Example

A technician wants to test a collar:

```text
assignmentReason = Testing
status = Active
```

After testing:

```text
status = Removed
removalReason = Manual Removal
```

or another appropriate removal reason.

The application should make clear that the assignment was temporary/testing.

---

# 66. Maintenance Example

```text
DEV-001
→ ANM-001
```

Maintenance needed:

```text
status = Removed
removalReason = Maintenance
```

After maintenance:

```text
new assignment record
assignmentReason = Reassignment
```

or another appropriate reason depending on the workflow.

The source does not define which assignment reason must be used after maintenance; the application should standardize this.

---

# 67. Device Upgrade Example

Old:

```text
DEV-001
→ ANM-001
```

New device:

```text
DEV-009
```

Old assignment:

```text
status = Removed
removalReason = Replacement
```

New assignment:

```text
status = Active
assignmentReason = Device Upgrade
```

This makes the reason for the new assignment explicit.

---

# 68. Assignment and SensorData

The finalized SensorData model contains:

```text
deviceId
assignmentId
animalId
recordedAt
metrics
```

fileciteturn25file0

Therefore every animal-linked telemetry record can preserve:

```text
physical device
+
assignment context
+
animal
```

---

# 69. Why `assignmentId` in SensorData Is Powerful

Suppose:

```text
DEV-001
```

was assigned to:

```text
Animal A-001
```

for three months.

Then:

```text
DEV-001
```

was assigned to:

```text
Animal A-017
```

SensorData can preserve:

```text
Old data → assignment ASN-001
New data → assignment ASN-002
```

This prevents historical ambiguity.

---

# 70. Assignment History Query

A device detail page could show:

```text
DEV-001

Assignment History

ASN-001
Animal A-001
Aug 1 – Sep 15
Removed: Replacement

ASN-002
Animal A-017
Sep 15 – Present
Active
```

This is one of the main reasons this collection exists.

---

# 71. Animal Detail Query

Animal A-001 can show:

```text
Current Devices
```

and:

```text
Previous Devices
```

using DeviceAssignment.

Example:

```text
Current:
DEV-009 Neck Collar

Previous:
DEV-001 Neck Collar
```

---

# 72. Device Detail Query

Device DEV-001 can show:

```text
Current Assignment:
None

Previous Assignment:
ANM-001
```

This can be derived from:

```text
DeviceAssignment
```

without modifying Device.

---

# 73. Current Assignment Query

Conceptually:

```text
find assignment where:

deviceId = DEV-001
status = Active
```

This gives the current assignment.

For an animal:

```text
find assignments where:

animalId = ANM-001
status = Active
```

This gives currently assigned devices.

---

# 74. Historical Assignment Query

For complete history:

```text
deviceId = DEV-001
sort by assignedDate descending
```

This returns:

```text
current
+
previous
```

assignments.

---

# 75. Assignment Overlap

Potential bad history:

```text
ASN-001
Aug 1 → Sep 15

ASN-002
Sep 10 → Oct 1
```

for the same device.

This creates a 5-day overlap.

For a single-animal device, backend should normally prevent overlapping active/effective assignment periods.

The finalized schema does not explicitly define this rule; it is a recommended integrity rule.

---

# 76. Assignment Date Validation

Recommended:

```text
removedDate >= assignedDate
```

If:

```text
removedDate < assignedDate
```

the assignment timeline is invalid.

---

# 77. Active Assignment Date Logic

For:

```text
status = Active
```

recommended:

```text
assignedDate != null
removedDate = null
```

For:

```text
status = Removed
```

recommended:

```text
assignedDate != null
removedDate != null
removalReason != null
removedBy != null
```

These are implementation consistency recommendations.

---

# 78. Cancelled Assignment Logic

For:

```text
status = Cancelled
```

the application should define whether:

```text
removedDate
```

is used.

Because the source only defines the enum and fields, it does not prescribe exact cancelled-record rules.

A consistent convention should be selected during backend implementation.

---

# 79. `assignedBy` Security

The frontend should not freely send:

```text
assignedBy = another user
```

The backend should derive it from:

```text
authenticated user
```

Example:

```text
req.user.userId
```

or the corresponding internal user reference.

---

# 80. `removedBy` Security

Same principle:

```text
removedBy
```

should represent the authenticated user performing the removal.

It should not be trusted from arbitrary client input.

---

# 81. Farm Authorization

Before creating assignment:

```text
User
 ↓
FarmMember
 ↓
Role/permission
 ↓
Farm
 ↓
Device
 ↓
Animal
 ↓
Create Assignment
```

The project architecture uses farm membership and role-based access control for farm isolation. fileciteturn25file9

---

# 82. Cross-Farm Security Example

User belongs to:

```text
Farm A
```

tries:

```text
deviceId = Device from Farm B
animalId = Animal from Farm A
```

Backend must reject.

Similarly:

```text
device from Farm A
animal from Farm B
```

must be rejected.

---

# 83. Assignment Cancellation

A user may start a testing/assignment workflow and then cancel it.

Conceptually:

```text
Assignment created
      ↓
Cancelled
```

This can preserve the fact that an assignment attempt existed without pretending that it became a normal operational assignment.

Again, the exact frontend workflow is not defined in the source.

---

# 84. Assignment History Must Be Append-Oriented

For important historical changes:

```text
Do not overwrite old assignment
Create a new assignment
```

This is especially important for:

```text
Reassignment
Replacement
Device Upgrade
```

because historical data must remain interpretable.

---

# 85. DeviceAssignment and AuditLog

Current finalized AuditLog `entityType` does **not include `DeviceAssignment`**. fileciteturn25file0turn25file18

Therefore direct:

```text
entityType = DeviceAssignment
```

is not currently represented by the finalized AuditLog enum.

This is a schema consideration to remember during implementation.

Do not silently change the finalized model here.

---

# 86. DeviceAssignment and Notification

Current Notification `referenceType` includes:

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

but does not include:

```text
DeviceAssignment
```

or:

```text
Device
```

fileciteturn25file0

Therefore assignment-specific notifications are not directly represented by the current enum.

---

# 87. DeviceAssignment and Report

Report supports:

```text
reportType = Device
```

but does not have a dedicated:

```text
DeviceAssignment
```

report type. fileciteturn25file0

Assignment history can still be included inside a Device report through filtering/query logic.

---

# 88. DeviceAssignment and Animal Lifecycle

Animal has:

```text
lifeStatus:
Active
Sold
Dead
Transferred
Archived
```

fileciteturn25file17

Therefore assignment management should react to lifecycle changes.

Examples:

```text
Animal Sold
→ close active assignment

Animal Dead
→ close active assignment

Animal Archived
→ normally no new active assignment
```

The exact automatic transition behavior should be implemented as service-level business logic.

---

# 89. DeviceAssignment and Animal Transfer

If:

```text
Animal.lifeStatus = Transferred
```

the device may need to be:

```text
removed
```

or:

```text
transferred with the animal
```

depending on the business workflow.

The finalized model does not specify which behavior should always occur.

Therefore this should be explicitly decided before implementation.

---

# 90. DeviceAssignment and Device Lifecycle

Device has:

```text
isActive
connectionStatus
healthStatus
```

The assignment service should validate these before assignment.

Example:

```text
Device.isActive = false
```

→ should not create a new Active assignment.

---

# 91. DeviceAssignment and Maintenance

If Device:

```text
connectionStatus = Maintenance
```

the active assignment may need to be temporarily closed depending on the farm's operational workflow.

The finalized schema provides:

```text
removalReason = Maintenance
```

which supports this use case.

---

# 92. Device Replacement Business Flow

Full flow:

```text
Device Fault
      ↓
Old Device healthStatus = Faulty
      ↓
Old Assignment
status = Removed
removalReason = Device Fault
      ↓
New Device Registered
      ↓
New Assignment
assignmentReason = Replacement
status = Active
```

This creates a complete chain.

---

# 93. Device Upgrade Business Flow

```text
Old Device
      ↓
Old Assignment Removed
      ↓
New Device
      ↓
New Assignment
assignmentReason = Device Upgrade
      ↓
Active
```

The old device remains available historically.

---

# 94. Animal Sold Business Flow

```text
Animal lifeStatus = Sold
      ↓
Find Active DeviceAssignments
      ↓
Close assignment
      ↓
removalReason = Animal Sold
      ↓
Device becomes available
```

This is a recommended service-level automation.

---

# 95. Animal Dead Business Flow

```text
Animal lifeStatus = Dead
      ↓
Find Active DeviceAssignments
      ↓
Close assignment
      ↓
removalReason = Animal Dead
```

This prevents stale active relationships.

---

# 96. Device Lost Business Flow

```text
Device reported lost
      ↓
Assignment removed
      ↓
removalReason = Lost
      ↓
Device remains historical
```

The Device record may later be marked inactive according to the farm's asset policy.

---

# 97. Device Assignment and Data Integrity

The most important consistency rule is:

```text
Assignment.farmId
==
Device.farmId
==
Animal.farmId
```

when all three belong to the same farm context.

---

# 98. Recommended Indexes

Useful indexes:

```javascript
{ farmId: 1, status: 1 }
```

for:

```text
active assignments in a farm
```

```javascript
{ deviceId: 1, status: 1 }
```

for:

```text
current assignment of a device
```

```javascript
{ animalId: 1, status: 1 }
```

for:

```text
current devices of an animal
```

```javascript
{ deviceId: 1, assignedDate: -1 }
```

for:

```text
assignment history of a device
```

These are implementation recommendations; the source defines fields but not indexes.

---

# 99. Most Important Index

For the common query:

```text
Is this device currently assigned?
```

a useful index is:

```javascript
{ deviceId: 1, status: 1 }
```

Potentially with a partial index for:

```text
status = Active
```

if the application's query patterns justify it.

---

# 100. Assignment History UI

A good UI could show:

| Assignment No | Device | Animal | Reason | Assigned | Removed | Status |
|---|---|---|---|---|---|---|
| ASN-001 | DEV-001 | ANM-001 | New Installation | Aug 1 | Sep 15 | Removed |
| ASN-002 | DEV-009 | ANM-001 | Replacement | Sep 15 | — | Active |

This is an application/UI concept derived from the finalized fields.

---

# 101. Device Page

On Device details:

```text
Device Information
------------------
Device Code
Company
Model
Serial Number
Device Type
Protocol
Firmware
Battery
Connection
Health

Assignment
----------
Current Animal
Assigned Date
Assignment Reason

History
-------
Previous Assignments
```

---

# 102. Animal Page

On Animal details:

```text
Animal Information
------------------

Current Devices
---------------
DEV-001
Neck Collar
Online

DEV-002
Ear Tag
Online

Previous Devices
----------------
DEV-005
Removed
Device Fault
```

This information is derived through DeviceAssignment.

---

# 103. Reassignment API Concept

Example:

```text
POST /api/device-assignments/reassign
```

Conceptually:

```text
Find current active assignment
        ↓
Close old assignment
        ↓
Create new assignment
        ↓
assignmentReason = Reassignment
        ↓
New status = Active
```

The exact API route is not defined by the source; this is an implementation example.

---

# 104. Replacement API Concept

Conceptually:

```text
POST /api/device-assignments/replace
```

Flow:

```text
Close old assignment
        ↓
Register/select replacement device
        ↓
Create new assignment
        ↓
assignmentReason = Replacement
```

Again, route naming is an implementation recommendation.

---

# 105. Transaction Safety

Replacement/reassignment often changes multiple records:

```text
Old Assignment
+
New Assignment
+
possibly Device status
```

These operations should ideally be handled atomically using a database transaction where appropriate.

This is implementation-level architecture, not a rule explicitly stated in the source.

---

# 106. Example Reassignment Transaction

```text
BEGIN TRANSACTION

1. Find active assignment
2. Validate device and animal
3. Set old assignment:
   status = Removed
   removedDate = now
   removalReason = Reassignment
   removedBy = current user

4. Create new assignment:
   assignmentReason = Reassignment
   status = Active
   assignedDate = now
   assignedBy = current user

COMMIT
```

If something fails:

```text
ROLLBACK
```

so the system does not end up with:

```text
no assignment
```

or:

```text
two active assignments
```

accidentally.

---

# 107. Why Not Just Update `animalId`?

Bad:

```text
ASN-001
animalId = A-001
```

later:

```text
ASN-001
animalId = A-017
```

This destroys historical information.

Correct:

```text
ASN-001
A-001
Removed

ASN-002
A-017
Active
```

---

# 108. Historical SensorData Example

Before reassignment:

```text
SensorData
deviceId = DEV-001
assignmentId = ASN-001
animalId = A-001
recordedAt = Aug 10
```

After reassignment:

```text
SensorData
deviceId = DEV-001
assignmentId = ASN-002
animalId = A-017
recordedAt = Sep 20
```

The same physical device is preserved while the animal context changes correctly.

---

# 109. Assignment and Research/Evaluation

Historical assignment records can be valuable for:

```text
animal monitoring analysis
device performance
sensor-data interpretation
IoT evaluation
research datasets
```

because the system can reconstruct:

```text
Which device monitored which animal
during which period.
```

This is especially important if Liventra later evaluates smart-device data.

---

# 110. Assignment and Data Provenance

A SensorData record can be traced:

```text
SensorData
    ↓
assignmentId
    ↓
DeviceAssignment
    ↓
deviceId
    ↓
Device

and

DeviceAssignment
    ↓
animalId
    ↓
Animal
```

This gives a clear data-provenance chain.

---

# 111. Complete Example

### Device

```text
deviceCode:
DEV-001

deviceType:
Neck Collar

farmId:
FARM-001
```

### Animal

```text
animalCode:
ANM-001

farmId:
FARM-001
```

### Assignment

```text
assignmentNo:
ASN-000001

farmId:
FARM-001

deviceId:
DEV-001

animalId:
ANM-001

assignedDate:
2026-08-17

assignmentReason:
New Installation

assignedBy:
USER-001

status:
Active
```

---

# 112. Complete Replacement Example

Old:

```text
ASN-000001
DEV-001
ANM-001
New Installation
2026-08-17
2026-09-20
removalReason = Device Fault
status = Removed
```

New:

```text
ASN-000002
DEV-009
ANM-001
Replacement
2026-09-20
status = Active
```

History is preserved.

---

# 113. Complete Reassignment Example

Old:

```text
ASN-000001
DEV-001
ANM-001
status = Removed
removalReason = Manual Removal
```

New:

```text
ASN-000002
DEV-001
ANM-017
assignmentReason = Reassignment
status = Active
```

Same device, different animal, two assignment records.

---

# 114. Complete Animal-Sold Example

Before:

```text
ASN-000002
DEV-009
ANM-001
status = Active
```

Animal sold:

```text
ANM-001.lifeStatus = Sold
```

Assignment becomes:

```text
status = Removed
removalReason = Animal Sold
removedDate = ...
```

---

# 115. Complete Testing Example

```text
ASN-000010
DEV-020
ANM-005
assignmentReason = Testing
status = Active
```

Testing complete:

```text
status = Removed
removalReason = Manual Removal
```

The testing history remains available.

---

# 116. Common Mistakes to Avoid

### Mistake 1

Putting `animalId` directly inside Device.

**Why bad:** loses assignment history.

### Mistake 2

Updating old assignment during reassignment.

**Why bad:** destroys historical relationship.

### Mistake 3

Allowing cross-farm Device/Animal assignment.

**Why bad:** violates farm isolation.

### Mistake 4

Allowing multiple active assignments for a single-animal device.

**Why bad:** ambiguous ownership.

### Mistake 5

Deleting old assignment records.

**Why bad:** historical data becomes incomplete.

### Mistake 6

Using `remarks` instead of structured reasons.

**Why bad:** difficult to query/report.

---

# 117. Business Rules Summary

```text
1. Every assignment belongs to a farm.
2. Every assignment references a Device.
3. Every assignment references an Animal under the current schema.
4. Device, Animal, and Assignment should belong to the same farm.
5. assignmentNo should be unique.
6. DeviceAssignment represents a historical relationship.
7. Reassignment creates a new record.
8. Replacement creates a new record.
9. Old assignments are not overwritten.
10. Active means the assignment is currently in effect.
11. Removed means the assignment has ended.
12. Cancelled represents a cancelled assignment workflow.
13. removedDate should be populated when an assignment ends.
14. removalReason should be populated when an assignment is removed.
15. removedBy should identify the user who removed it.
16. assignedBy should identify the user who created the assignment.
17. A single-animal device should normally have at most one Active assignment.
18. An animal may have multiple devices of different types.
19. Animal lifecycle changes should trigger assignment review.
20. Historical assignments should be retained.
21. SensorData should reference assignmentId to preserve context.
22. Cross-farm assignment should be rejected.
23. Deactivated devices should normally not receive new Active assignments.
24. DeviceAssignment should not store sensor telemetry.
25. DeviceAssignment should not replace Device's hardware metadata.
```

Rules 5–24 above are primarily implementation/business recommendations derived from the finalized model relationships; the source explicitly defines the fields/enums but does not spell out every validation rule. The source-supported structure is the basis. fileciteturn25file0turn25file1

---

# 118. Current Schema Considerations

## Consideration 1 — AuditLog

Current finalized `AuditLog.entityType` does not include:

```text
Device
DeviceAssignment
SensorData
Report
```

although these are now collections in the overall model. fileciteturn25file12

Therefore assignment operations cannot currently be represented as:

```text
entityType = DeviceAssignment
```

without changing that enum.

**Do not change it here; just remember this before implementation.**

---

## Consideration 2 — Notification

Current Notification `referenceType` does not include:

```text
Device
DeviceAssignment
```

but does include:

```text
SensorData
```

fileciteturn25file0

Therefore sensor-related notifications can be directly referenced, while assignment-specific notifications are not directly represented.

---

## Consideration 3 — Farm-Level Devices

The Device model includes:

```text
Environment Sensor
Water Meter
```

but the current DeviceAssignment model requires:

```text
animalId
```

This means the finalized structure is much more naturally suited to animal-linked devices.

If Liventra later needs:

```text
Environment Sensor → Shed
Water Meter → Water Point
Milk Meter → Milking Station
```

a separate location/equipment assignment strategy may be needed.

No schema change is being made in this document.

---

# 119. Final Architecture

```text
                         FARM
                          │
              ┌───────────┴───────────┐
              ▼                       ▼
           DEVICE                  ANIMAL
              │                       │
              └──────────┬────────────┘
                         ▼
                DEVICE ASSIGNMENT
                         │
          ┌──────────────┼──────────────┐
          │              │              │
          ▼              ▼              ▼
     assignmentNo    assignedDate    status
          │
          ├── assignmentReason
          ├── assignedBy
          ├── removedDate
          ├── removalReason
          ├── removedBy
          └── remarks
                         │
                         ▼
                    SENSOR DATA
                         │
                         ▼
                  Animal Monitoring
```

---

# 120. Final Model Responsibility

The clean separation is:

```text
Device
────────────────────────
Physical smart device
Manufacturer
Model
Serial
Protocol
Firmware
Battery
Connection
Device health
Current device state
```

```text
DeviceAssignment
────────────────────────
Device ↔ Animal relationship
Assignment start
Assignment end
Assignment reason
Removal reason
Assigned user
Removed user
Assignment status
Assignment history
```

```text
SensorData
────────────────────────
Measurements
Recorded time
Metrics
Alerts
Raw payload
Data source
```

This separation is consistent with the finalized three-collection IoT structure. fileciteturn25file0

---

# 121. Final Status

**Model:** `DeviceAssignment`  
**Number:** 26  
**Status:** ✅ Upcoming / documented

```text
Identity                     ✓
Farm relationship            ✓
Device relationship         ✓
Animal relationship         ✓
Assignment lifecycle        ✓
Assignment reasons          ✓
Removal reasons             ✓
User accountability         ✓
Status management           ✓
Historical reassignment     ✓
Replacement workflow        ✓
Animal lifecycle handling   ✓
SensorData relationship     ✓
Farm isolation              ✓
Validation logic            ✓
Security considerations     ✓
Index recommendations      ✓
Schema limitations          ✓
```

## Most Important Concept

```text
NEVER do this:

Device
animalId = newAnimal
```

to represent reassignment.

Instead:

```text
Old DeviceAssignment
→ Removed

New DeviceAssignment
→ Active
```

That is what allows Liventra to preserve the complete history of:

```text
Device → Animal → Time Period
```

and allows SensorData to remain historically interpretable through `assignmentId`. fileciteturn25file0

---

# 122. Model Progress

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
26 DeviceAssignment ← CURRENT
27 SensorData
28 Report
```

**Next → `SensorData`**
