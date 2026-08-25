# Liventra — Business Logic Documentation
## Model 25: `Device` (Upcoming)

**Project:** Liventra — Smart Livestock Farm Management System  
**Module:** IoT / Smart Device Management  
**Status:** Upcoming / Future Integration  
**Database:** MongoDB  
**Architecture:** MERN

> **Source basis:** This document follows the finalized Liventra DB Models. The finalized model marks `Device` as an **upcoming** collection and defines the fields/enums below. fileciteturn24file1

---

# 1. Finalized Device Model

```text
Device:
_id,
deviceCode,
farmId,
company,
model,
serialNumber,

deviceType:
    Bolus
    Neck Collar
    Ear Tag
    Milk Meter
    Temperature Sensor
    Environment Sensor
    Water Meter
    Other,

communicationProtocol:
    Bluetooth
    WiFi
    LoRaWAN
    NB-IoT
    LTE
    RFID
    API,

firmwareVersion,
installationDate,
purchaseDate,
warrantyExpiry,

batteryType:
    Rechargeable
    Replaceable
    Internal
    Not Applicable,

batteryLevel,

lastSyncAt,

connectionStatus:
    Online
    Offline
    Disconnected
    Maintenance,

healthStatus:
    Healthy
    Warning
    Critical
    Faulty,

notes,
isActive,
createdBy,
updatedBy,
createdAt,
updatedAt
```

The finalized DB document contains this exact Device structure and places Device under the upcoming IoT-related collections. fileciteturn24file1

---

# 2. What Is a Device?

In Liventra, a `Device` represents a **physical smart livestock device registered with a farm**.

Examples:

```text
Smart rumen bolus
Neck collar
Smart ear tag
Milk meter
Temperature sensor
Environment sensor
Water meter
```

The project's architecture is intentionally IoT-ready so future smart livestock technologies can be integrated without redesigning the entire system. The project documentation specifically discusses smart collars, RFID tags, rumen bolus sensors, milk meters, and environmental sensors as future smart-farming technologies. fileciteturn24file2turn24file6

---

# 3. Important Concept: Device ≠ SensorData

These two collections have different responsibilities.

```text
Device
=
What physical device exists?

SensorData
=
What did that device measure?
```

Example:

```text
Device
deviceCode = DEV-001
deviceType = Temperature Sensor
serialNumber = TS-98372
```

Then:

```text
SensorData
bodyTemperature = 38.7
recordedAt = 2026-08-17 10:30
```

So:

```text
Device
     ↓
SensorData
```

---

# 4. Device ≠ DeviceAssignment

Similarly:

```text
Device
=
The physical device itself

DeviceAssignment
=
Which animal/farm context the device is currently assigned to
```

Example:

```text
Device:
DEV-001
Neck Collar
```

Assignment:

```text
DEV-001
→ Animal A-001
```

If later moved:

```text
DEV-001
→ Animal A-017
```

the Device itself remains the same.

The assignment history belongs to `DeviceAssignment`, whose finalized model contains `deviceId`, `animalId`, dates, reasons, assigned/removed by, and status. fileciteturn24file1

---

# 5. Device Lifecycle

The overall lifecycle should conceptually be:

```text
Device Purchased
       ↓
Device Registered
       ↓
Device Installed
       ↓
Device Assigned
       ↓
Device Communicates
       ↓
SensorData Received
       ↓
Monitoring
       ↓
Maintenance / Fault
       ↓
Replacement / Removal
       ↓
Inactive
```

Important:

`Device` stores the **device lifecycle/state**, while `DeviceAssignment` stores the **assignment lifecycle**.

---

# 6. `_id`

### Type
MongoDB `ObjectId`

### Required
Yes

### Purpose
Internal unique identifier for the Device document.

Example:

```text
ObjectId(...)
```

This is the database identity and should not be used as the human-facing device number.

---

# 7. `deviceCode`

### Type
String

### Required
Yes

### Example

```text
DEV-000001
```

### Purpose

Human-readable identifier for the device.

Useful for:

```text
Device list
Device dashboard
Maintenance
Search
Reports
Support
```

---

# 8. Device Code Generation

Recommended:

```text
DEV-000001
DEV-000002
DEV-000003
```

The backend should generate this.

Do not let the frontend freely generate device codes because duplicate codes can create serious confusion when devices are physically installed.

---

# 9. Device Code vs Serial Number

These are different.

### `deviceCode`

Liventra's internal identifier:

```text
DEV-000001
```

### `serialNumber`

Manufacturer/device-specific physical identifier:

```text
SN-AX92-7731
```

Therefore:

```text
deviceCode
=
Liventra identity

serialNumber
=
physical/manufacturer identity
```

---

# 10. `farmId`

### Type
ObjectId → Farm

### Required
Yes

### Purpose

Identifies which farm owns/registers the device.

Example:

```text
farmId = FARM-001
```

This is important for multi-farm isolation.

---

# 11. Device Farm Ownership Logic

A device belongs to one farm context:

```text
Farm A
   ↓
Device DEV-001
Device DEV-002
Device DEV-003
```

Another farm should not be able to access:

```text
DEV-001
```

through normal API requests.

The project requirements emphasize preventing cross-farm data access. fileciteturn24file2

---

# 12. Can One Device Belong to Multiple Farms?

Under the current model:

```text
Device.farmId
```

contains a single farm reference.

Therefore the current design treats the device as belonging to one farm at a time.

If a device is transferred between farms in the future, the application would need a defined transfer workflow.

Do not assume multi-farm ownership from the current schema.

---

# 13. `company`

### Type
String

### Required
Yes

### Example

```text
smaXtec
```

or:

```text
Afimilk
```

or:

```text
Local Manufacturer
```

### Purpose

Stores the device manufacturer/company.

The project's background research specifically discusses commercial smart livestock technologies and companies such as Afimilk, smaXtec, Allflex SenseHub, CowManager, and Connecterra. fileciteturn24file2

---

# 14. Why `company` Is Needed

Two devices can have:

```text
deviceType = Neck Collar
```

but come from different manufacturers.

Example:

```text
Company A
Neck Collar
```

and:

```text
Company B
Neck Collar
```

Their:

```text
API
communication protocol
firmware
data format
```

may differ.

---

# 15. `model`

### Type
String

### Required
Yes

### Example

```text
NC-Pro-2
```

Purpose:

Identifies the manufacturer's specific device model.

Useful for:

```text
Firmware compatibility
API integration
Maintenance
Device documentation
Replacement
```

---

# 16. `serialNumber`

### Type
String

### Required
Yes

### Example

```text
SN-2026-00881
```

### Purpose

Unique physical/manufacturer identifier.

Recommended business rule:

```text
serialNumber should normally be unique within the system
```

because two registered physical devices should not accidentally share the same serial number.

---

# 17. `deviceType`

### Type
Enum

### Required
Yes

### Finalized Values

```text
Bolus
Neck Collar
Ear Tag
Milk Meter
Temperature Sensor
Environment Sensor
Water Meter
Other
```

These are defined in the finalized DB model. fileciteturn24file1

---

# 18. `Bolus`

A rumen bolus-type device is intended to be associated with an animal and can provide physiological/behavior-related measurements depending on the actual hardware.

Example:

```text
deviceType = Bolus
```

Possible future SensorData:

```text
bodyTemperature
ruminationMinutes
activityScore
```

The finalized SensorData model explicitly contains these metrics. fileciteturn24file1

---

# 19. `Neck Collar`

Example:

```text
deviceType = Neck Collar
```

Possible use:

```text
Activity monitoring
Rumination monitoring
Animal behavior
```

The exact measurements depend on the device manufacturer and integration.

Do not assume every collar provides every metric.

---

# 20. `Ear Tag`

Example:

```text
deviceType = Ear Tag
```

Potential role:

```text
Animal identification
RFID-based identification
Smart monitoring
```

The actual data capability depends on the device.

---

# 21. `Milk Meter`

Example:

```text
deviceType = Milk Meter
```

Potential purpose:

```text
Milk production measurement
```

This aligns with Liventra's production-management objectives and future smart-device integration.

---

# 22. `Temperature Sensor`

Example:

```text
deviceType = Temperature Sensor
```

Potential measurement:

```text
bodyTemperature
```

or an environmental temperature measurement depending on installation.

Important:

The Device record itself does not store the measurements.

Measurements belong in:

```text
SensorData
```

---

# 23. `Environment Sensor`

Potential use:

```text
Temperature
Humidity
Environmental conditions
```

The current SensorData schema does not explicitly define a humidity field, so humidity should **not** be assumed to be directly supported by the finalized SensorData structure.

This is an example of why the Device type and SensorData metric definitions must be treated separately.

---

# 24. `Water Meter`

Potential use:

```text
Water consumption
```

The finalized SensorData model contains:

```text
waterIntake
```

so a compatible water-monitoring device can contribute to that metric. fileciteturn24file1

---

# 25. `Other`

Use when:

```text
A future device type
```

does not fit the predefined categories.

Example:

```text
deviceType = Other
```

and:

```text
notes = "Smart environmental gateway"
```

Do not create arbitrary new enum values in production without updating the schema.

---

# 26. `communicationProtocol`

### Type
Enum

### Required
Yes

### Finalized Values

```text
Bluetooth
WiFi
LoRaWAN
NB-IoT
LTE
RFID
API
```

These are explicitly defined in the finalized model. fileciteturn24file1

---

# 27. Why Communication Protocol Matters

The protocol determines how Liventra can receive or communicate with the device.

Conceptually:

```text
Device
   ↓
Communication Protocol
   ↓
Gateway/API/Middleware
   ↓
Liventra
   ↓
SensorData
```

---

# 28. `Bluetooth`

Useful for:

```text
Short-range communication
Local gateway/mobile integration
```

Example:

```text
deviceType = Temperature Sensor
communicationProtocol = Bluetooth
```

---

# 29. `WiFi`

Useful when the device can directly communicate over a local wireless network.

Example:

```text
Environment Sensor
→ WiFi
→ Farm gateway/server
→ Liventra
```

---

# 30. `LoRaWAN`

Useful for:

```text
Long-range, low-power IoT communication
```

Potentially suitable for distributed farm environments.

The exact implementation would require compatible gateway/infrastructure.

---

# 31. `NB-IoT`

Cellular IoT communication.

Useful for devices that communicate through supported cellular infrastructure.

---

# 32. `LTE`

Cellular communication option for devices/gateways with mobile-network connectivity.

---

# 33. `RFID`

Useful primarily for identification/tracking workflows.

Example:

```text
RFID Ear Tag
```

Important distinction:

```text
RFID identification
≠
continuous sensor telemetry
```

An RFID-enabled device does not automatically mean it produces the full SensorData metric set.

---

# 34. `API`

Use when Liventra communicates with an external device platform through an API.

Architecture:

```text
Commercial Device Platform
          ↓
       REST/API
          ↓
      Liventra
          ↓
      SensorData
```

This is particularly relevant to the project's goal of future integration with commercial smart livestock systems. fileciteturn24file2

---

# 35. `firmwareVersion`

### Type
String

### Required
Conditional

### Example

```text
v2.4.1
```

### Purpose

Tracks the software/firmware version running on the physical device.

Useful for:

```text
Compatibility
Bug investigation
Maintenance
Device updates
Vendor support
```

---

# 36. Firmware Business Logic

Suppose:

```text
100 devices
```

are registered.

Some run:

```text
v2.1
```

others:

```text
v2.4
```

If SensorData behavior changes after a firmware update, the firmware version helps identify whether the issue is device-version-specific.

---

# 37. Firmware Version Is Not Device Model

Example:

```text
model:
NC-Pro-2

firmwareVersion:
v3.1.0
```

Model identifies hardware.

Firmware identifies the software version running on that hardware.

---

# 38. `installationDate`

### Type
Date

### Required
Conditional

### Example

```text
2026-08-10
```

### Purpose

Records when the device was physically installed/deployed.

This is different from:

```text
purchaseDate
```

because a device can be purchased before installation.

---

# 39. Purchase vs Installation

Example:

```text
purchaseDate:
2026-07-01

installationDate:
2026-08-10
```

Meaning:

```text
Device bought in July
Device installed in August
```

---

# 40. `purchaseDate`

### Type
Date

### Required
Optional

### Purpose

Records when the farm purchased/acquired the device.

Useful for:

```text
Cost tracking
Warranty
Asset management
Replacement planning
```

---

# 41. `warrantyExpiry`

### Type
Date

### Required
Optional

### Example

```text
2028-07-01
```

Purpose:

Tracks the warranty period.

Business logic:

```text
today < warrantyExpiry
→ warranty active

today > warrantyExpiry
→ warranty expired
```

---

# 42. Warranty Logic

The system can use:

```text
warrantyExpiry
```

to display:

```text
Warranty Active
Warranty Expiring Soon
Warranty Expired
```

The exact "expiring soon" threshold is not defined by the source, so implementation should choose and document one rather than assuming a specific number of days.

---

# 43. `batteryType`

### Type
Enum

### Required
Yes

### Values

```text
Rechargeable
Replaceable
Internal
Not Applicable
```

fileciteturn24file1

---

# 44. `Rechargeable`

Example:

```text
batteryType = Rechargeable
```

Business meaning:

```text
Device has a rechargeable battery.
```

---

# 45. `Replaceable`

Example:

```text
batteryType = Replaceable
```

Means the battery can be physically replaced.

Useful for maintenance planning.

---

# 46. `Internal`

Example:

```text
batteryType = Internal
```

Means the device uses an internal battery configuration.

The exact replacement/service behavior depends on the manufacturer.

---

# 47. `Not Applicable`

Use for devices that do not operate on a battery.

Example:

```text
Milk Meter
```

if powered through another source.

Do not assume every milk meter is non-battery-powered; this is device-specific.

---

# 48. `batteryLevel`

### Type
Number

### Required
Conditional

### Example

```text
78
```

Meaning:

```text
78%
```

Recommended logical range:

```text
0–100
```

The finalized model defines the field but does not explicitly state a numeric range. Therefore the `0–100` constraint is an implementation recommendation based on the field's meaning, not a source-defined enum.

---

# 49. Battery Logic

For battery-powered devices:

```text
batteryType != Not Applicable
```

then:

```text
batteryLevel
```

can be populated when the device supports battery telemetry.

For:

```text
Not Applicable
```

battery level should normally be:

```text
null
```

rather than:

```text
0
```

because:

```text
0%
```

means "battery empty", while:

```text
null
```

means "not applicable/not provided."

---

# 50. Battery Warning

The current Device model does not define a battery threshold field.

Therefore the backend can implement a warning rule using a documented application-level threshold.

Example:

```text
batteryLevel < configured threshold
→ low-battery warning
```

Do not hard-code an undocumented threshold into the database model.

---

# 51. `lastSyncAt`

### Type
Date

### Required
Optional

### Purpose

Records the most recent time Liventra successfully synchronized/received data from the device or its integration layer.

Example:

```text
lastSyncAt:
2026-08-17 10:30
```

---

# 52. `lastSyncAt` vs `recordedAt`

These are different.

### SensorData

```text
recordedAt
```

means:

> When was the measurement recorded?

### Device

```text
lastSyncAt
```

means:

> When did the system last synchronize/receive data from the device/integration?

Example:

```text
Sensor measurement:
10:20

Uploaded to Liventra:
10:25
```

Then:

```text
SensorData.recordedAt = 10:20
Device.lastSyncAt = 10:25
```

This distinction is important.

---

# 53. `connectionStatus`

### Type
Enum

### Required
Yes

### Values

```text
Online
Offline
Disconnected
Maintenance
```

fileciteturn24file1

---

# 54. `Online`

Means the system currently considers the device connected/reachable according to the integration mechanism.

Important:

```text
Online
```

does not necessarily mean:

```text
the device is producing valid measurements.
```

Data validity is a separate concern.

---

# 55. `Offline`

Means the device is not currently communicating according to the application's connectivity logic.

Possible causes:

```text
Temporary network issue
Battery issue
Gateway unavailable
Device out of range
```

---

# 56. `Disconnected`

Represents a more explicit loss/removal of connection.

The exact distinction between:

```text
Offline
Disconnected
```

should be defined in the integration layer because the finalized model provides both values but does not prescribe the exact transition rules.

A consistent rule is essential.

---

# 57. `Maintenance`

Means the device is intentionally taken out of normal operation for:

```text
Repair
Inspection
Battery replacement
Firmware update
Testing
```

This is different from:

```text
Offline
```

because:

```text
Offline
=
communication problem/unavailable

Maintenance
=
intentional operational state
```

---

# 58. `healthStatus`

### Type
Enum

### Required
Yes

### Values

```text
Healthy
Warning
Critical
Faulty
```

fileciteturn24file1

---

# 59. `healthStatus` vs `connectionStatus`

These should never be confused.

### Connection status

```text
Can the system communicate with it?
```

### Health status

```text
Does the device appear to be functioning normally?
```

Example:

```text
connectionStatus = Online
healthStatus = Faulty
```

This is possible.

A device can be connected but malfunctioning.

---

# 60. Example State Combinations

### Healthy + Online

```text
Normal
```

### Warning + Online

```text
Connected but something needs attention
```

### Critical + Online

```text
Connected but serious issue detected
```

### Healthy + Offline

```text
No current communication,
but no known device health issue
```

### Faulty + Disconnected

```text
Device has a known fault and is not communicating
```

---

# 61. `notes`

### Type
String

### Required
Optional

### Example

```text
Installed near cattle shed 2.
Vendor support contact available.
```

Purpose:

Free-form operational notes.

Use for:

```text
Installation notes
Maintenance comments
Special device information
```

Do not use `notes` as a replacement for structured fields.

---

# 62. `isActive`

### Type
Boolean

### Required
Yes

### Example

```text
true
```

Purpose:

Indicates whether the device record is currently active in Liventra.

---

# 63. `isActive` vs `connectionStatus`

Again, different responsibilities.

```text
isActive = true
```

means:

```text
Device is an active registered asset.
```

while:

```text
connectionStatus = Offline
```

means:

```text
It is not currently connected.
```

Therefore:

```text
isActive = true
connectionStatus = Offline
```

is completely valid.

---

# 64. Deactivating a Device

Suppose a device is permanently retired.

The application can set:

```text
isActive = false
```

instead of deleting the document.

This preserves:

```text
device identity
history
assignments
sensor relationships
auditability
```

---

# 65. Device Deletion

Physical device history is valuable.

Therefore the recommended business approach is:

```text
Deactivate
```

rather than:

```text
Hard delete
```

for normal retirement.

The exact deletion policy is not specified in the source, so this is an implementation recommendation.

---

# 66. `createdBy`

### Type
ObjectId → User

### Required
Yes

### Purpose

Identifies who registered the device in Liventra.

Example:

```text
createdBy = USER-001
```

---

# 67. `updatedBy`

### Type
ObjectId → User

### Required
Optional/conditional

### Purpose

Identifies the user responsible for the latest modification.

Example:

```text
updatedBy = USER-002
```

---

# 68. Device Audit

The finalized AuditLog `entityType` does **not currently include `Device`**. fileciteturn24file15

Therefore:

```text
Device created
Device updated
Device deactivated
Device maintenance state changed
```

cannot currently be represented as a direct `AuditLog.entityType = Device` event under the finalized AuditLog enum.

This is an important schema consideration.

Do not silently modify the AuditLog enum here.

---

# 69. `createdAt`

### Type
Date

### Required
Yes

Purpose:

Stores when the Device record was created in Liventra.

---

# 70. `updatedAt`

### Type
Date

### Required
Yes

Purpose:

Stores the most recent modification time of the Device document.

---

# 71. Device Lifecycle Example

```text
2026-07-01
Device purchased

        ↓

2026-07-05
Registered in Liventra

        ↓

2026-07-10
Installed

        ↓

2026-07-11
Assigned to Animal A-001

        ↓

2026-07-11 → 2026-08-17
Sensor data received

        ↓

2026-08-17
Battery warning

        ↓

2026-08-18
Maintenance

        ↓

2026-08-20
Returned to service
```

The Device record stores the device's current state, while assignment/history and sensor measurements belong to their respective collections.

---

# 72. Device Registration Workflow

Recommended:

```text
Admin/Farm Owner
      ↓
Add Device
      ↓
Enter:
Company
Model
Serial Number
Device Type
Protocol
Firmware
Purchase Info
      ↓
Validate
      ↓
Generate deviceCode
      ↓
Save Device
      ↓
Device becomes available for assignment
```

---

# 73. Device Registration Validation

Backend should validate:

```text
farmId belongs to authorized user
serialNumber is valid
deviceType is valid
communicationProtocol is valid
batteryType is valid
dates are valid
```

---

# 74. Serial Number Uniqueness

Recommended business rule:

```text
serialNumber unique
```

This prevents:

```text
Two physical devices
→ same serial number
```

from being registered accidentally.

If vendors have overlapping serial-number formats, a composite uniqueness rule may be considered:

```text
company + serialNumber
```

This is an implementation consideration; the finalized source does not explicitly define the unique index.

---

# 75. Device Code Uniqueness

Recommended:

```text
deviceCode unique
```

because it is the system's human-readable device identity.

---

# 76. Device Assignment Dependency

A Device should generally be registered before assignment.

```text
Device
   ↓
DeviceAssignment
```

not:

```text
DeviceAssignment
   ↓
unknown Device
```

The DeviceAssignment model requires:

```text
deviceId
```

and:

```text
animalId
```

which links the physical device to the animal context. fileciteturn24file15

---

# 77. Device Can Exist Without Animal Assignment

Yes.

Example:

```text
New device purchased
```

but not installed yet.

Device:

```text
isActive = true
```

but no active DeviceAssignment exists.

This is useful for:

```text
Inventory/pre-installation
Testing
Spare devices
```

---

# 78. Device Assignment Example

```text
Device:
DEV-001

Type:
Neck Collar

Animal:
ANM-001
```

DeviceAssignment:

```text
deviceId = DEV-001
animalId = ANM-001
status = Active
```

SensorData can then reference:

```text
deviceId
assignmentId
animalId
```

The finalized SensorData model explicitly contains all three relationships. fileciteturn24file1

---

# 79. Why `assignmentId` Is Important Later

Suppose:

```text
DEV-001
```

was assigned to:

```text
Animal A
```

then later:

```text
Animal B
```

Sensor history needs to know the assignment context.

Therefore:

```text
SensorData.assignmentId
```

helps distinguish:

```text
DEV-001 → Animal A
```

from:

```text
DEV-001 → Animal B
```

The finalized SensorData model includes `assignmentId` for this purpose. fileciteturn24file1

---

# 80. Device Transfer

Suppose:

```text
DEV-001
```

moves from:

```text
Animal A
```

to:

```text
Animal B
```

Do not overwrite the old assignment.

Instead:

```text
Old assignment
status = Removed
removedDate = ...
```

then create:

```text
New assignment
status = Active
assignedDate = ...
```

This preserves assignment history.

---

# 81. Device Removal

The finalized DeviceAssignment model defines removal reasons:

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

and status:

```text
Active
Removed
Cancelled
```

fileciteturn24file1

Therefore Device lifecycle and DeviceAssignment lifecycle should work together.

---

# 82. Device Maintenance Workflow

Example:

```text
Device status:
Maintenance
```

Then:

```text
connectionStatus = Maintenance
```

During maintenance:

```text
No normal assignment activity
```

depending on the operational policy.

After repair:

```text
connectionStatus = Online
```

if successfully connected.

---

# 83. Device Fault Workflow

Suppose device reports a hardware problem.

Possible state:

```text
healthStatus = Faulty
connectionStatus = Disconnected
```

Then:

```text
DeviceAssignment
status = Removed
removalReason = Device Fault
```

and a replacement device may be assigned.

---

# 84. Device Replacement Workflow

```text
Old Device
    ↓
Fault detected
    ↓
Old DeviceAssignment removed
    ↓
New Device registered
    ↓
New DeviceAssignment created
    ↓
New SensorData starts
```

Do not replace:

```text
deviceId
```

inside old SensorData.

Historical data must remain associated with the original physical device.

---

# 85. Device + SensorData Architecture

The finalized SensorData model is:

```text
SensorData:
_id,
farmId,
deviceId,
assignmentId,
animalId,
recordedAt,
metrics:{...},
alerts:[...],
rawPayload,
source,
createdAt
```

fileciteturn24file1

So the relationship is:

```text
Farm
 ↓
Device
 ↓
DeviceAssignment
 ↓
Animal
 ↓
SensorData
```

with the exact relationships represented through IDs rather than embedded documents.

---

# 86. Device + SensorData: One-to-Many

One Device can generate many SensorData records.

Example:

```text
DEV-001
   ↓
SensorData 10:00
SensorData 10:05
SensorData 10:10
SensorData 10:15
...
```

Therefore:

```text
Device 1 → SensorData many
```

---

# 87. Device + DeviceAssignment: One-to-Many Over Time

One device can have multiple assignments over its lifetime:

```text
DEV-001
   ↓
Assignment #1 → Animal A
Assignment #2 → Animal B
Assignment #3 → Animal C
```

But normally only one should be:

```text
status = Active
```

at a time for a device that supports single-animal assignment.

The exact restriction depends on device type.

---

# 88. Device Type and Assignment Semantics

Not every device necessarily belongs to one animal.

Example:

```text
Neck Collar
→ animal-specific
```

while:

```text
Environment Sensor
→ farm/shed-level
```

This is important.

The current Device model only has:

```text
farmId
```

and DeviceAssignment provides:

```text
animalId
```

So application logic must determine which device types are animal-assignable.

---

# 89. Animal-Specific vs Farm-Level Devices

### Animal-specific

Examples:

```text
Bolus
Neck Collar
Ear Tag
```

Typically associated with:

```text
animalId
```

### Farm/environment-level

Examples:

```text
Environment Sensor
Water Meter
```

may operate at:

```text
farm/shed/area
```

rather than one animal.

The current DeviceAssignment model only has `animalId`, not a location/shed reference. Therefore farm-level assignment is a design consideration for future implementation.

---

# 90. Important Schema Consideration: Environment Devices

The finalized model includes:

```text
Environment Sensor
Water Meter
```

but `DeviceAssignment` is centered around:

```text
animalId
```

Therefore the current schema naturally fits animal-linked devices better than farm-location devices.

This is **not being silently changed** here; it should simply be reviewed before implementing farm-level IoT devices.

---

# 91. `lastSyncAt` and Connection Status

The system can derive useful operational information from:

```text
lastSyncAt
+
connectionStatus
```

Example:

```text
connectionStatus = Offline
lastSyncAt = 2 hours ago
```

Dashboard:

```text
⚠ Device has not synchronized recently
```

The exact timeout threshold is not defined in the finalized model.

---

# 92. Device Health vs Sensor Health

Device:

```text
healthStatus = Healthy
```

means the device itself is considered healthy.

SensorData:

```text
metrics
alerts
```

represents observed measurements/alerts.

One should not automatically change Device health to `Faulty` merely because one sensor reading is abnormal.

Example:

```text
Body temperature high
```

may indicate:

```text
Animal issue
```

rather than:

```text
Device fault
```

This distinction is critical.

---

# 93. Example: Animal Alert vs Device Fault

```text
bodyTemperature = high
```

could produce:

```text
SensorData alert
```

while:

```text
device communication failure
```

could produce:

```text
Device health/connection problem
```

These are different domains.

---

# 94. SensorData Alerts

The finalized SensorData model includes:

```text
alerts:[
    {
        type,
        severity,
        message
    }
]
```

fileciteturn24file1

Therefore device-generated health/animal alerts should generally be stored with the SensorData event, while Device stores the device's operational health state.

---

# 95. Device Data Ingestion

Future architecture:

```text
Physical Device
      ↓
Protocol
      ↓
Gateway / Vendor API / Middleware
      ↓
Liventra Integration Layer
      ↓
Validate / Normalize
      ↓
SensorData
      ↓
Alert Processing
      ↓
Notification
```

This fits the project's IoT-ready architecture goal. fileciteturn24file2

---

# 96. Why Middleware Is Useful

Different vendors may send different formats:

```text
Vendor A
{
  temp: 38.5
}
```

Vendor B:

```text
{
  body_temperature: 38.5
}
```

Liventra should normalize these into:

```text
SensorData.metrics.bodyTemperature
```

The Device model identifies:

```text
company
model
communicationProtocol
```

which can help the integration layer determine how to process incoming data.

---

# 97. `source` Belongs to SensorData, Not Device

The finalized SensorData source enum is:

```text
API
MQTT
Bluetooth
Manual Import
```

fileciteturn24file1

Do not put this in Device.

Why?

Because the same physical device may eventually send data through different integration paths.

Device stores:

```text
communicationProtocol
```

SensorData stores:

```text
actual source of that data ingestion
```

---

# 98. Device Firmware Update

Possible future workflow:

```text
Current:
firmwareVersion = v2.1

Update:
v2.2

Device:
firmwareVersion = v2.2
updatedAt = ...
```

This should be audited if/when Device becomes an auditable entity.

Currently, direct Device auditing is not supported by the finalized AuditLog enum. fileciteturn24file15

---

# 99. Device Synchronization

A successful synchronization can update:

```text
lastSyncAt
connectionStatus
batteryLevel
```

Example:

```text
Device sends data
      ↓
Data validated
      ↓
SensorData created
      ↓
Device.lastSyncAt updated
      ↓
Device.connectionStatus = Online
```

---

# 100. Failed Synchronization

If communication fails:

```text
SensorData not created
```

and the device may become:

```text
connectionStatus = Offline
```

or:

```text
Disconnected
```

depending on the application's defined connectivity state rules.

The exact timeout/transition logic is not specified by the source.

---

# 101. Battery Update

A device may send:

```text
batteryLevel = 20
```

Then:

```text
Device.batteryLevel = 20
```

This can support dashboard warnings.

If battery information is included in SensorData metrics:

```text
metrics.batteryLevel
```

then the latest value can be synchronized to Device for quick dashboard access.

---

# 102. Why Store Battery Level in Both?

The finalized design has:

```text
Device.batteryLevel
```

and:

```text
SensorData.metrics.batteryLevel
```

This is reasonable because:

```text
SensorData
=
historical measurement

Device.batteryLevel
=
latest/current state
```

Example:

```text
SensorData:
08:00 → 80%
09:00 → 75%
10:00 → 70%

Device:
batteryLevel = 70%
```

---

# 103. Why Store `lastSyncAt` in Device?

SensorData can tell:

```text
latest recordedAt
```

but Device-level:

```text
lastSyncAt
```

provides a direct operational status field.

This avoids repeatedly querying SensorData just to render:

```text
Last synchronized: 10:30 AM
```

---

# 104. Device Dashboard

A useful Device dashboard could show:

```text
Device Code
Company
Model
Type
Serial Number
Protocol
Firmware
Battery
Connection
Health
Last Sync
Warranty
Assigned Animal
```

Example:

```text
DEV-001
Neck Collar
Healthy
Online
Battery: 82%
Last Sync: 2 min ago
Animal: A-001
```

---

# 105. Device Status Colors in UI

The database does not define UI colors.

But semantically:

```text
Online
Healthy
```

can be shown as normal/positive states.

```text
Warning
Offline
```

as attention states.

```text
Critical
Faulty
```

as urgent states.

Actual UI styling is outside the DB model.

---

# 106. Device Search

Useful search fields:

```text
deviceCode
serialNumber
company
model
deviceType
connectionStatus
healthStatus
```

Example:

```text
Search:
SN-2026
```

finds the physical device.

---

# 107. Device Filtering

Useful filters:

```text
Company
Device Type
Communication Protocol
Connection Status
Health Status
Active/Inactive
Warranty Status
```

---

# 108. Device Maintenance List

Example:

```text
Filter:
connectionStatus = Maintenance
```

shows:

```text
DEV-001
DEV-008
DEV-019
```

Then the manager can inspect:

```text
notes
healthStatus
lastSyncAt
warrantyExpiry
```

---

# 109. Warranty Dashboard

Query:

```text
warrantyExpiry
```

can generate:

```text
Active Warranty
Expired Warranty
Expiring Soon
```

Again, the exact "soon" period should be a configurable/application-level rule rather than an undocumented fixed database rule.

---

# 110. Device Inventory Relationship

The Device model has:

```text
purchaseDate
```

but it is **not an Inventory record**.

Therefore:

```text
Inventory
=
general farm stock/items

Device
=
registered smart device asset
```

If device purchases need financial tracking:

```text
Expense
```

can record the purchase cost, while:

```text
Device
```

stores the asset identity.

The current Device model itself does not contain:

```text
purchasePrice
```

so do not assume that field exists.

---

# 111. Device + Expense

Possible relationship:

```text
Expense
expenseCategory = Equipment
```

and:

```text
referenceType = Other
```

under the finalized Expense enum if the exact device-specific reference is not represented.

The current Expense model does not define `Device` as a `referenceType`. Therefore direct Device-to-Expense reference is not currently supported by that enum.

---

# 112. Device + Report

The finalized Report model explicitly supports:

```text
reportType = Device
```

and:

```text
reportType = Sensor
```

fileciteturn24file1

Therefore the Device module can eventually generate reports such as:

```text
Device Inventory Report
Device Status Report
Device Health Report
```

and Sensor reports.

---

# 113. Device Report Workflow

```text
User selects:
Device Report

Filters:
Farm
Device Type
Connection Status
Date Range

        ↓

Report Generator
        ↓
PDF / Excel / CSV / JSON
```

The Report model supports these formats. fileciteturn24file1

---

# 114. Device Notification

The finalized Notification model supports:

```text
referenceType = SensorData
```

but does not include:

```text
Device
```

as a reference type. fileciteturn24file1

Therefore current notifications for device-related problems would most naturally be tied to:

```text
SensorData
```

or:

```text
Manual
```

depending on the event.

This is a schema consideration.

---

# 115. Device Alert Example

Suppose:

```text
DEV-001
```

detects abnormal animal temperature.

Flow:

```text
Device
   ↓
SensorData
   ↓
Alert
   ↓
Notification
```

Notification can use:

```text
referenceType = SensorData
referenceId = SENSOR-DATA-ID
```

because that is supported by the finalized Notification enum. fileciteturn24file1

---

# 116. Device Fault Notification

Suppose:

```text
Device itself is faulty
```

The current Notification enum has no:

```text
Device
```

reference type.

Therefore a direct:

```text
referenceType = Device
```

is not valid under the current finalized model.

This should be reviewed before implementing device-specific notification workflows.

---

# 117. Device Health Monitoring

Possible automatic rules:

```text
healthStatus = Warning
→ notification

healthStatus = Critical
→ high-priority notification

healthStatus = Faulty
→ maintenance workflow
```

But the finalized model does not itself specify the exact notification rules or thresholds.

These belong to application business logic.

---

# 118. Device Connection Monitoring

Similarly:

```text
connectionStatus = Offline
```

could trigger:

```text
Warning
```

after a configured inactivity period.

But:

```text
Offline
```

should not immediately mean:

```text
Faulty
```

because network interruption can be temporary.

---

# 119. Connection Monitoring Logic

Conceptually:

```text
lastSyncAt
      ↓
Compare with current time
      ↓
No recent synchronization?
      ↓
connectionStatus = Offline
```

If the device is intentionally under maintenance:

```text
connectionStatus = Maintenance
```

should take precedence over an automatic offline classification.

---

# 120. Device Health Monitoring Logic

Health status can be updated based on:

```text
Device diagnostics
Battery
Communication health
Vendor/device error codes
Manual inspection
```

But animal health should not automatically be equated with device health.

---

# 121. Critical Distinction

```text
Device.healthStatus
```

answers:

> Is the physical smart device functioning properly?

Whereas:

```text
Animal.healthStatus
```

answers:

> What is the health condition of the animal?

These are completely different concepts.

---

# 122. Device + Animal Health Example

```text
Device:
Healthy

SensorData:
bodyTemperature = 40.2°C

Animal:
healthStatus = Sick
```

This is perfectly possible.

The sensor can be working correctly while reporting an unhealthy animal.

---

# 123. Device Failure Example

```text
Device:
healthStatus = Faulty

SensorData:
No new data
```

This does not automatically mean:

```text
Animal.healthStatus = Sick
```

The system should not make that assumption.

---

# 124. Device Registration Security

Only authorized users should be able to:

```text
Create Device
Update Device
Deactivate Device
Change maintenance state
```

because device configuration affects the farm's IoT infrastructure.

The project requirements include role-based access and farm-level isolation. fileciteturn24file2

---

# 125. Device Access by Role

A reasonable mapping:

```text
Admin
→ system-level device management

Farm Owner
→ farm device management

Farm Manager
→ operational device management

Worker
→ limited device visibility/use
```

Exact permissions should follow the project's final RBAC implementation.

---

# 126. Cross-Farm Device Security

Every Device query should be scoped by:

```text
farmId
```

Example:

```text
User belongs to Farm A

Request:
GET /devices/DEV-001
```

Backend must verify:

```text
DEV-001.farmId == authorized farm
```

before returning the device.

---

# 127. Device API Security

Do not trust:

```text
farmId
```

sent by the frontend.

Instead derive authorized farm access from:

```text
authenticated user
+
FarmMember
+
role
```

and then validate the requested Device.

---

# 128. Device Creation Security

When a Farm Manager creates a device:

```text
farmId
```

should come from the authorized farm context rather than being freely chosen to another farm.

This prevents cross-farm registration.

---

# 129. Device Deactivation Security

Deactivating a device can affect:

```text
SensorData ingestion
DeviceAssignment
Notifications
Monitoring
```

Therefore it should be a protected operation.

---

# 130. Device Deactivation Workflow

Recommended:

```text
Check authorization
      ↓
Check active assignment
      ↓
If active:
    remove/close assignment according to workflow
      ↓
Set isActive = false
      ↓
Stop normal ingestion/monitoring
      ↓
Preserve historical data
```

The exact removal workflow belongs to DeviceAssignment.

---

# 131. Historical Data Preservation

Never delete historical:

```text
SensorData
```

just because:

```text
Device.isActive = false
```

Historical sensor measurements remain valuable.

Example:

```text
DEV-001 retired in 2028
```

but its:

```text
2026–2028 SensorData
```

should remain available according to the project's data-retention policy.

---

# 132. Device Retirement

A device can be:

```text
isActive = false
```

while historical records remain.

This allows:

```text
Past device reports
Historical animal monitoring
Maintenance analysis
Research evaluation
```

---

# 133. Device Replacement and Historical Continuity

Suppose:

```text
DEV-001
```

fails and is replaced by:

```text
DEV-009
```

Do not merge their histories.

Instead:

```text
DEV-001
→ old SensorData

DEV-009
→ new SensorData
```

and DeviceAssignment records show the transition.

---

# 134. Device Performance Analysis

Using:

```text
Device
+
SensorData
```

the system can later calculate:

```text
Data availability
Sync reliability
Battery trends
Connectivity problems
Fault frequency
```

This is a future analytics capability.

---

# 135. Device Data Quality

A connected device may send:

```text
invalid values
duplicate readings
missing metrics
out-of-order timestamps
```

The finalized Device model does not contain data-quality fields.

Therefore validation should happen in the ingestion layer before creating SensorData.

---

# 136. Duplicate Device Data

Example:

```text
same device
same timestamp
same measurement
```

The ingestion layer should detect duplicates where possible.

The exact uniqueness rule belongs to SensorData design rather than Device.

---

# 137. Device Clock vs Server Clock

A physical device may have:

```text
device timestamp
```

while Liventra has:

```text
server receipt time
```

The current Device model has:

```text
lastSyncAt
```

but SensorData has:

```text
recordedAt
```

The integration layer should preserve the intended measurement time correctly.

---

# 138. Device Offline Buffering

Some IoT devices may continue collecting data while offline.

Then:

```text
Device offline
      ↓
Measurements buffered
      ↓
Device reconnects
      ↓
Historical SensorData uploaded
```

This is why:

```text
recordedAt
```

must represent the measurement time rather than blindly using upload time.

---

# 139. Device Data Backfill

Example:

```text
Device offline:
10:00–12:00

Reconnect:
12:05

Uploads:
10:00
10:15
10:30
...
11:45
12:00
```

SensorData should preserve those historical timestamps.

Device:

```text
lastSyncAt = 12:05
```

This gives correct operational history.

---

# 140. Device Communication Gateway

A scalable architecture can use:

```text
Device
 ↓
Gateway
 ↓
Protocol Adapter
 ↓
Normalizer
 ↓
SensorData Service
```

Example:

```text
Bluetooth Adapter
MQTT Adapter
API Adapter
```

all produce the same normalized SensorData structure.

---

# 141. Protocol Adapter Concept

```text
if protocol == Bluetooth
    use Bluetooth adapter

if protocol == API
    use vendor API adapter

if protocol == MQTT
    use MQTT integration
```

The Device document provides the metadata required to select/understand the integration path.

---

# 142. Device Integration Is Future Work

The project documentation explicitly positions smart-device integration as future-ready/extensible rather than saying every commercial device will be integrated in the first release. fileciteturn24file2

Therefore the current Device model should be treated as:

```text
architecture foundation
```

rather than proof that all listed devices are already connected.

---

# 143. Device + RFID

Animal Model already has:

```text
rfid
```

The Device model also supports:

```text
deviceType = Ear Tag
communicationProtocol = RFID
```

This creates an important distinction:

```text
Animal.rfid
=
animal's registered RFID identifier

Device
=
physical smart RFID device/tag
```

The application should define how the two are linked.

For an RFID tag used as a physical device, DeviceAssignment can provide the animal association.

---

# 144. Device + Animal `rfid`

Potential flow:

```text
Device:
DEV-001
Ear Tag
RFID

        ↓

DeviceAssignment
animalId = ANM-001

        ↓

Animal
rfid = RFID-ABC123
```

This creates a traceable physical tag-to-animal relationship.

---

# 145. Important RFID Consideration

Do not duplicate the same identifier inconsistently.

Example bad state:

```text
Animal.rfid = RFID-001

Device.serialNumber = RFID-002
```

if the physical tag actually corresponds to RFID-001.

Registration/assignment workflow should validate identifiers according to the actual device design.

---

# 146. Device + QR Code

Animal Model has:

```text
qrCode
```

but Device does not have a dedicated QR field.

Therefore:

```text
Animal QR
```

and:

```text
Device QR
```

are separate concepts.

Do not assume Device can be identified by the Animal QR field.

---

# 147. Device Search by Serial

For maintenance:

```text
Search serialNumber
```

should locate the exact physical device.

Then display:

```text
Device Code
Model
Firmware
Health
Connection
Assignment
Warranty
```

---

# 148. Device Maintenance History

The current Device model has:

```text
notes
connectionStatus
healthStatus
updatedAt
```

but there is no dedicated:

```text
DeviceMaintenance
```

collection.

Therefore detailed maintenance history is not explicitly represented in the finalized schema.

If detailed maintenance records become a requirement, that would be a future schema consideration.

---

# 149. Device Fault History

Similarly, current Device only stores:

```text
healthStatus
```

not a historical list of fault events.

Therefore:

```text
Current state
```

is supported, but:

```text
complete fault history
```

would require either application event logging or future schema design.

---

# 150. Device State Transition

Conceptually:

```text
Healthy
   ↓
Warning
   ↓
Critical
   ↓
Faulty
```

or:

```text
Healthy
   ↓
Maintenance
   ↓
Healthy
```

The actual transition rules are application logic.

---

# 151. Device Status Validation

The backend should reject invalid states such as:

```text
connectionStatus = "Working"
```

because:

```text
Working
```

is not a finalized enum value.

Allowed:

```text
Online
Offline
Disconnected
Maintenance
```

---

# 152. Device Health Validation

Similarly:

```text
healthStatus = "Bad"
```

is invalid.

Allowed:

```text
Healthy
Warning
Critical
Faulty
```

---

# 153. Device Type Validation

Allowed only:

```text
Bolus
Neck Collar
Ear Tag
Milk Meter
Temperature Sensor
Environment Sensor
Water Meter
Other
```

Do not use:

```text
Collar
Smart Collar
Thermometer
Water Sensor
```

unless the schema is intentionally updated.

---

# 154. Communication Protocol Validation

Allowed:

```text
Bluetooth
WiFi
LoRaWAN
NB-IoT
LTE
RFID
API
```

The actual protocol adapter must support the selected protocol before the device is marked operational.

---

# 155. Firmware Update Logic

If:

```text
firmwareVersion
```

changes:

```text
updatedAt
```

should also change.

If the update causes temporary downtime:

```text
connectionStatus = Maintenance
```

may be used during the operation.

---

# 156. Battery Maintenance Logic

For:

```text
Replaceable
```

battery devices:

```text
battery low
→ maintenance
→ battery replacement
→ battery level refreshed
```

For:

```text
Rechargeable
```

the device may require:

```text
charging
```

rather than replacement.

The exact maintenance workflow is device-dependent.

---

# 157. Warranty Logic with Maintenance

If:

```text
healthStatus = Faulty
```

and:

```text
warrantyExpiry > current date
```

the system can display:

```text
Potential warranty service
```

This can help the farm manager decide whether to contact the vendor.

---

# 158. Device Reporting

Possible reports:

```text
All Devices
Active Devices
Offline Devices
Faulty Devices
Maintenance Devices
Warranty Expiry
Device Assignment
Device Synchronization
```

The finalized Report model supports a generic `Device` report type. fileciteturn24file1

---

# 159. Device Analytics

Future analytics may include:

```text
Online percentage
Average battery level
Sync frequency
Fault rate
Maintenance frequency
Device uptime
```

These metrics would be derived from Device + SensorData/integration history.

They are not fields in the Device model.

---

# 160. Device Uptime

Do not add:

```text
uptime
```

to Device.

It can be calculated from:

```text
connectionStatus history
lastSyncAt
```

and/or integration events.

The finalized model does not contain a dedicated uptime field.

---

# 161. Device Health Dashboard

Example:

```text
Total Devices: 25

Healthy:      20
Warning:       3
Critical:      1
Faulty:        1
```

These are aggregations over:

```text
healthStatus
```

---

# 162. Connection Dashboard

Example:

```text
Online:        18
Offline:        4
Disconnected:   2
Maintenance:    1
```

Derived from:

```text
connectionStatus
```

---

# 163. Battery Dashboard

For battery-enabled devices:

```text
Critical:
2

Low:
4

Normal:
15
```

Exact thresholds are application-level configuration.

---

# 164. Device Security

Device credentials/API keys should **not** be stored in the current Device model.

The finalized model contains:

```text
company
model
serialNumber
protocol
firmware
```

but no:

```text
apiKey
secret
password
token
```

This is a good separation.

If vendor credentials are needed later, they should be handled through a secure secrets/configuration mechanism rather than plain Device documents.

---

# 165. Device API Authentication

A future integration may require:

```text
API key
OAuth
MQTT credentials
```

These should not be placed into:

```text
notes
```

or:

```text
raw Device fields
```

because that creates a security risk.

---

# 166. Device Data Ownership

SensorData should be scoped by:

```text
farmId
deviceId
```

so queries can remain farm-isolated.

This is especially important because device telemetry can contain:

```text
animal health
production
behavior
```

data.

---

# 167. Device Query Security

When fetching:

```text
GET /devices/:deviceId
```

backend should:

```text
Authenticate
    ↓
Find Device
    ↓
Check Device.farmId
    ↓
Check user's FarmMember authorization
    ↓
Return device
```

not simply:

```text
Device.findById(deviceId)
```

without farm validation.

---

# 168. Device Deactivation and SensorData

If:

```text
isActive = false
```

then normal new telemetry should generally stop being accepted from that device.

However, delayed historical data may still arrive after deactivation.

Therefore ingestion logic should decide whether such data is:

```text
accepted as historical
```

or:

```text
rejected
```

based on the actual retirement workflow.

---

# 169. Device Assignment and Deactivation

Before deactivating:

```text
Check active assignment
```

If assigned:

```text
close DeviceAssignment
```

with appropriate:

```text
removedDate
removalReason
removedBy
status = Removed
```

The finalized DeviceAssignment model explicitly supports these fields. fileciteturn24file1

---

# 170. Device Registration + Audit Gap

A key architecture issue:

```text
Device
```

is not included in:

```text
AuditLog.entityType
```

So if the team wants:

```text
Who registered Device DEV-001?
Who changed firmwareVersion?
Who deactivated it?
```

the current AuditLog schema cannot represent this directly.

This should be consciously reviewed before implementing Device management.

---

# 171. Device + Notification Gap

Similarly:

```text
Notification.referenceType
```

does not include:

```text
Device
```

under the finalized enum. fileciteturn24file1

So:

```text
Device offline
Device faulty
Warranty expiring
```

cannot currently be directly referenced as:

```text
referenceType = Device
```

without a schema change.

---

# 172. Device + Report Is Supported

Unlike AuditLog/Notification, the Report model **does** explicitly support:

```text
reportType = Device
```

so device reporting is already represented in the finalized design. fileciteturn24file1

---

# 173. Device + SensorData Is Supported

SensorData explicitly contains:

```text
deviceId
```

so the main IoT data relationship is already represented.

This is the strongest direct relationship:

```text
Device._id
    ↓
SensorData.deviceId
```

---

# 174. Device + DeviceAssignment Is Supported

DeviceAssignment explicitly contains:

```text
deviceId
```

so assignment history is also directly represented.

Architecture:

```text
Device
  ├── DeviceAssignment
  │       └── Animal
  │
  └── SensorData
```

---

# 175. Complete IoT Relationship

```text
                    Farm
                     │
                     ▼
                  Device
             ┌───────┴────────┐
             │                │
             ▼                ▼
      DeviceAssignment     SensorData
             │                │
             ▼                ▼
           Animal          Metrics
                              │
                              ▼
                           Alerts
                              │
                              ▼
                        Notification
```

This is the conceptual architecture of the upcoming IoT module under the finalized models.

---

# 176. Example Complete Scenario

Suppose:

```text
DEV-001
Company: ExampleTech
Model: Collar-X
Serial: SN-9988
Type: Neck Collar
Protocol: Bluetooth
```

Device is assigned:

```text
Animal:
ANM-001
```

Then device sends:

```text
Body temperature:
38.7°C

Rumination:
420 min

Activity:
75
```

SensorData stores:

```text
deviceId = DEV-001
assignmentId = ASSIGN-001
animalId = ANM-001
```

Device updates:

```text
batteryLevel = 82
lastSyncAt = now
connectionStatus = Online
```

This is the intended separation of current state vs historical telemetry.

---

# 177. Example Device Failure Scenario

```text
DEV-001
```

stops communicating.

System detects:

```text
No recent sync
```

Then:

```text
connectionStatus = Offline
```

If diagnostics confirm hardware failure:

```text
healthStatus = Faulty
```

Then:

```text
DeviceAssignment
→ Removed
→ removalReason = Device Fault
```

Replacement:

```text
DEV-009
→ Active Assignment
→ Animal A-001
```

Historical SensorData remains associated with:

```text
DEV-001
```

---

# 178. Example Device Maintenance Scenario

```text
DEV-002
```

needs battery replacement.

Before maintenance:

```text
connectionStatus = Online
healthStatus = Healthy
```

During maintenance:

```text
connectionStatus = Maintenance
```

After replacement:

```text
batteryLevel = 100
connectionStatus = Online
healthStatus = Healthy
```

---

# 179. Example Warranty Scenario

```text
purchaseDate:
2026-07-01

warrantyExpiry:
2028-07-01
```

If device becomes faulty in:

```text
2027
```

the system can show:

```text
Warranty Active
```

and help the manager initiate vendor support.

---

# 180. Example Multi-Device Farm

```text
Farm A
│
├── DEV-001
│   └── Neck Collar
│
├── DEV-002
│   └── Bolus
│
├── DEV-003
│   └── Milk Meter
│
├── DEV-004
│   └── Environment Sensor
│
└── DEV-005
    └── Water Meter
```

Each Device has:

```text
farmId = Farm A
```

but only animal-specific devices need animal assignments.

---

# 181. Device Registration API Concept

Example:

```text
POST /api/devices
```

Request:

```json
{
  "company": "ExampleTech",
  "model": "Collar-X",
  "serialNumber": "SN-9988",
  "deviceType": "Neck Collar",
  "communicationProtocol": "Bluetooth",
  "firmwareVersion": "v2.1.0",
  "batteryType": "Rechargeable"
}
```

Backend should determine:

```text
farmId
createdBy
deviceCode
createdAt
```

from authorized context/system logic rather than blindly trusting the client.

---

# 182. Device Update API Concept

Example:

```text
PATCH /api/devices/:id
```

Possible updates:

```text
firmwareVersion
batteryLevel
connectionStatus
healthStatus
notes
isActive
```

Sensitive changes should require proper role authorization.

---

# 183. Device List API Concept

Example:

```text
GET /api/devices
```

Filters:

```text
deviceType
company
connectionStatus
healthStatus
isActive
```

always scoped to the authorized farm.

---

# 184. Device Detail API

Example:

```text
GET /api/devices/:id
```

UI can combine:

```text
Device
+
Current Assignment
+
Latest SensorData
+
Latest Alerts
```

without embedding all of these into the Device document.

This keeps the Device collection lightweight.

---

# 185. Why Not Embed SensorData in Device?

Bad design:

```text
Device
{
   ...
   sensorData: [
      thousands of records
   ]
}
```

This causes:

```text
Huge documents
Slow updates
MongoDB document growth
Poor historical querying
```

Better:

```text
Device
```

stores metadata/current state.

```text
SensorData
```

stores time-series-like measurements separately.

---

# 186. Why Not Embed Assignments in Device?

Similarly:

```text
Device
{
   assignments: [...]
}
```

would grow over time.

The finalized design correctly separates:

```text
DeviceAssignment
```

as its own collection.

---

# 187. Device Collection Should Stay Relatively Small

A Device document should mainly represent:

```text
Identity
Manufacturer
Hardware
Connectivity
Firmware
Installation
Warranty
Current state
Ownership
Timestamps
```

Historical events belong elsewhere.

---

# 188. Device Index Recommendations

Based on expected usage, useful indexes would include:

```javascript
{ farmId: 1, deviceCode: 1 }
{ farmId: 1, serialNumber: 1 }
{ farmId: 1, connectionStatus: 1 }
{ farmId: 1, healthStatus: 1 }
{ farmId: 1, deviceType: 1 }
```

If serial numbers are globally unique:

```javascript
{ serialNumber: 1 }
```

could be unique.

These are implementation recommendations; the finalized model itself does not specify indexes.

---

# 189. Most Important Device Index

For farm-scoped device listing:

```javascript
{
    farmId: 1,
    isActive: 1
}
```

can help queries such as:

```text
Show active devices in this farm
```

---

# 190. Device Search Index

If users frequently search by:

```text
deviceCode
serialNumber
```

indexes should be added accordingly.

---

# 191. Device Data Consistency Rules

Recommended:

```text
deviceCode unique
serialNumber appropriately unique
farmId valid
deviceType valid
communicationProtocol valid
batteryType valid
dates logically consistent
batteryLevel within valid range when provided
createdBy authorized
updatedBy authorized
```

---

# 192. Date Consistency

Examples:

Invalid:

```text
installationDate
<
purchaseDate
```

may be suspicious depending on the business context.

Generally:

```text
purchaseDate <= installationDate
```

when both are known.

Similarly:

```text
warrantyExpiry
```

would normally be after purchase date.

These are business validation recommendations, not explicit source rules.

---

# 193. Firmware Validation

`firmwareVersion` is a string because vendors may use different formats:

```text
v1.2.3
2.4
2026.08
R12
```

Do not force numeric storage.

---

# 194. Battery Validation

For:

```text
batteryType = Not Applicable
```

recommended:

```text
batteryLevel = null
```

For battery-powered devices:

```text
batteryLevel ∈ [0,100]
```

when available.

---

# 195. Device State Validation

Potential invalid combination:

```text
isActive = false
connectionStatus = Online
```

This may indicate stale state.

The application should define what deactivation does to connection state.

A practical rule could be:

```text
isActive = false
→ device should not remain operationally Online
```

but the exact enum transition is not defined in the source.

---

# 196. Device Maintenance vs Inactive

Do not confuse:

```text
connectionStatus = Maintenance
```

with:

```text
isActive = false
```

Maintenance:

```text
Temporarily not in normal operation
```

Inactive:

```text
No longer active in the system
```

Therefore:

```text
Maintenance
→ isActive can remain true
```

while:

```text
Retired
→ isActive = false
```

---

# 197. Device Ownership and Farm Transfer

Current model:

```text
farmId
```

does not provide a transfer history.

If a device moves to another farm, simply overwriting:

```text
farmId
```

would lose historical ownership context.

Therefore future cross-farm transfer requirements should be carefully designed before implementation.

---

# 198. Device Transfer Consideration

If cross-farm transfer becomes required, possible future architecture:

```text
Device
→ current farmId

DeviceTransferHistory
→ previous/current farms
```

But **this is not part of the finalized model** and should not be implemented unless the requirement is confirmed.

---

# 199. Device + Report + Audit Gap Summary

Current finalized architecture:

```text
Device
   ├── SensorData       ✓
   ├── DeviceAssignment ✓
   ├── Report           ✓
   ├── AuditLog         ✗ direct entityType support
   └── Notification     ✗ direct referenceType support
```

This is the key Device-related schema review point.

---

# 200. Final Business Rules Checklist

```text
[✓] Device represents a registered physical smart device
[✓] Device is an upcoming collection
[✓] deviceCode is the Liventra-facing identifier
[✓] serialNumber represents physical/manufacturer identity
[✓] farmId provides farm ownership/scope
[✓] company identifies manufacturer
[✓] model identifies device model
[✓] deviceType uses finalized enum
[✓] communicationProtocol uses finalized enum
[✓] firmwareVersion tracks device firmware
[✓] installationDate tracks deployment
[✓] purchaseDate tracks acquisition
[✓] warrantyExpiry tracks warranty
[✓] batteryType uses finalized enum
[✓] batteryLevel represents current battery state
[✓] lastSyncAt represents latest synchronization
[✓] connectionStatus tracks connectivity state
[✓] healthStatus tracks device operational health
[✓] connectionStatus and healthStatus are different concepts
[✓] notes stores optional operational information
[✓] isActive tracks current registration/activity state
[✓] createdBy identifies device registrar
[✓] updatedBy identifies latest modifier
[✓] createdAt/updatedAt track document lifecycle
[✓] Device should generally be registered before assignment
[✓] Device can exist without an active animal assignment
[✓] DeviceAssignment stores assignment history
[✓] SensorData stores historical measurements
[✓] One Device can have many SensorData records
[✓] One Device can have multiple assignments over time
[✓] Historical SensorData should not be deleted simply because Device is inactive
[✓] Device replacement should preserve old device history
[✓] Device health is not Animal health
[✓] Device connection is not Animal health
[✓] Device data should be farm-scoped
[✓] Frontend must not be trusted for farm authorization
[✓] Device credentials/secrets should not be stored in the current Device model
[✓] Device reports are supported through Report.reportType = Device
[✓] Device is currently not supported directly by AuditLog.entityType
[✓] Device is currently not supported directly by Notification.referenceType
[✓] Environment/farm-level device assignment needs future design consideration
[✓] Device maintenance history is not separately modeled
[✓] Device fault history is not separately modeled
[✓] Device uptime is not directly stored
[✓] Device analytics should be derived rather than embedded
[✓] Sensor telemetry should remain outside Device
```

---

# 201. Final Architecture

The upcoming IoT module should conceptually work like this:

```text
                    ┌──────────────┐
                    │     Farm     │
                    └──────┬───────┘
                           │
                           ▼
                    ┌──────────────┐
                    │    Device    │
                    │              │
                    │ Identity     │
                    │ Hardware     │
                    │ Protocol     │
                    │ Firmware     │
                    │ Battery      │
                    │ Connection   │
                    │ Health       │
                    └──────┬───────┘
                           │
              ┌────────────┴────────────┐
              ▼                         ▼
      ┌────────────────┐       ┌────────────────┐
      │DeviceAssignment│       │   SensorData   │
      │                │       │                │
      │deviceId        │       │deviceId        │
      │animalId        │       │assignmentId    │
      │assignedDate    │       │animalId        │
      │removedDate     │       │recordedAt      │
      │status          │       │metrics         │
      └───────┬────────┘       │alerts          │
              │                │source          │
              ▼                └───────┬────────┘
           Animal                       │
                                        ▼
                                  Notifications
```

This design keeps the three responsibilities clean:

```text
Device
→ physical device + current state

DeviceAssignment
→ where/with whom the device is assigned

SensorData
→ measurements/events produced by the device
```

---

# 202. Final Status

**Model:** `Device`  
**Number:** 25  
**Status:** ✅ Upcoming / documented  
**Identity:** ✅  
**Farm ownership:** ✅  
**Device classification:** ✅  
**Communication:** ✅  
**Firmware:** ✅  
**Installation/purchase/warranty:** ✅  
**Battery management:** ✅  
**Connection state:** ✅  
**Device health:** ✅  
**Assignment relationship:** ✅  
**SensorData relationship:** ✅  
**Security/farm isolation:** ✅  
**Realtime integration architecture:** ✅  
**Reporting integration:** ✅  
**Current schema limitations:** ✅ Documented

## Most Important Review Points

Before actual IoT implementation, keep these in mind:

### 1. Device is intentionally future-ready

The project documentation describes the system as modular and IoT-ready, with future integration of smart collars, RFID, rumen bolus sensors, milk meters, and environmental sensors. fileciteturn24file2

### 2. `DeviceAssignment` is essential

Do not put `animalId` directly into Device. The finalized architecture intentionally separates assignment history into `DeviceAssignment`. fileciteturn24file1

### 3. `SensorData` is separate

Do not embed telemetry inside Device. SensorData already provides:

```text
deviceId
assignmentId
animalId
recordedAt
metrics
alerts
rawPayload
source
```

fileciteturn24file1

### 4. There are two current schema gaps

```text
Device → AuditLog
Device → Notification
```

are not directly represented by the finalized enums.

This should be consciously reviewed before the IoT module is implemented.

### 5. Farm-level devices need attention

`Environment Sensor` and `Water Meter` exist in `deviceType`, but `DeviceAssignment` is currently centered around `animalId`. This is a design consideration if farm/shed-level IoT devices are actually implemented.

---

# 203. Model Progress

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
25 Device ← CURRENT
26 DeviceAssignment
27 SensorData
28 Report
```

**Next → `DeviceAssignment`**
