# Liventra — Business Logic Documentation
## Model 27: `SensorData` (Upcoming)

**Project:** Liventra — Smart Livestock Farm Management System  
**Module:** IoT / Smart Monitoring  
**Collection Status:** Upcoming

> **Source basis:** This document follows the finalized Liventra DB model. `SensorData` is defined as an upcoming collection containing farm/device/assignment/animal references, a measurement timestamp, a structured `metrics` object, alerts, the original payload, the ingestion source, and a creation timestamp. fileciteturn26file0turn26file1

---

# 1. Finalized SensorData Model

```text
SensorData:
_id,
farmId,
deviceId,
assignmentId,
animalId,
recordedAt,

metrics:{
    bodyTemperature,
    ruminationMinutes,
    activityScore,
    steps,
    waterIntake,
    feedIntake,
    milkYield,
    heartRate,
    respirationRate,
    batteryLevel
},

alerts:[
    {
        type,
        severity,
        message
    }
],

rawPayload,

source:
    API
    MQTT
    Bluetooth
    Manual Import,

createdAt
```

These fields and enum values are directly present in the finalized DB model. fileciteturn26file0

---

# 2. What Is SensorData?

`SensorData` stores the **actual measurements/events received from smart devices**.

The simplest way to understand the IoT collections is:

```text
Device
=
Which physical device exists?

DeviceAssignment
=
Which animal is/was the device assigned to?

SensorData
=
What did the device measure?
```

So:

```text
Device
   ↓
DeviceAssignment
   ↓
SensorData
```

The project documentation describes Liventra as an IoT-ready architecture intended for future integration with smart collars, ear tags, rumen bolus sensors, milk meters, environmental sensors, and related technologies. fileciteturn26file7turn26file18

---

# 3. Why SensorData Must Be a Separate Collection

Sensor measurements can become extremely large.

For example, one device might produce:

```text
Every 1 minute
```

or:

```text
Every 5 minutes
```

Then a single device can generate thousands of records.

Therefore this is better:

```text
Device
   │
   └── SensorData collection
          ├── Record 1
          ├── Record 2
          ├── Record 3
          ├── ...
          └── Record N
```

rather than:

```text
Device
{
    sensorData: [
        thousands of records
    ]
}
```

The finalized model already separates Device and SensorData. fileciteturn26file1

---

# 4. Main Responsibility of SensorData

SensorData answers:

```text
What was measured?
When was it measured?
By which device?
For which assignment?
For which animal?
How did it enter Liventra?
Was there an alert?
What was the original payload?
```

It does **not** represent:

```text
device registration
device ownership
device assignment history
animal master data
```

Those belong to other models.

---

# 5. `_id`

### Type
MongoDB `ObjectId`

### Required
Yes

### Purpose

Unique identifier of one sensor-data event/document.

Example:

```text
ObjectId("...")
```

Each measurement/event gets its own document identity.

---

# 6. `farmId`

### Type
ObjectId → Farm

### Required
Yes

### Purpose

Identifies the farm to which the sensor-data record belongs.

Example:

```text
farmId = FARM-001
```

This is important for:

```text
Multi-farm architecture
Farm-level filtering
Authorization
Reports
Analytics
```

---

# 7. Why SensorData Needs `farmId`

The Device already has:

```text
farmId
```

and the DeviceAssignment also has:

```text
farmId
```

So why duplicate it?

Because SensorData is expected to become a **large collection**.

Having `farmId` directly on every SensorData record makes common queries much easier:

```text
Give me today's sensor data for Farm A.
```

Instead of always resolving:

```text
SensorData
→ Device
→ Farm
```

This is a denormalized query-friendly design recommendation based on the finalized structure.

---

# 8. Farm Isolation

A user belonging to:

```text
Farm A
```

must not be able to query arbitrary SensorData belonging to:

```text
Farm B
```

Backend queries should therefore normally include:

```text
farmId
```

alongside the requested device/animal filters.

---

# 9. `deviceId`

### Type
ObjectId → Device

### Required
Yes

### Purpose

Identifies which physical device produced the sensor data.

Example:

```text
deviceId = DEV-001
```

Relationship:

```text
Device._id
      ↓
SensorData.deviceId
```

The finalized SensorData model explicitly contains `deviceId`. fileciteturn26file0

---

# 10. Why `deviceId` Is Critical

Suppose:

```text
Animal A-001
```

has:

```text
DEV-001
```

and:

```text
DEV-009
```

at different periods.

A sensor record must identify the actual physical source.

Therefore:

```text
deviceId
```

provides device-level provenance.

---

# 11. `assignmentId`

### Type
ObjectId → DeviceAssignment

### Required
Yes for animal-linked telemetry

### Purpose

Identifies the assignment context under which the measurement was produced.

Example:

```text
assignmentId = ASN-001
```

This is extremely important when a device is reassigned.

---

# 12. Why `assignmentId` Is One of the Most Important Fields

Suppose:

```text
DEV-001
```

was assigned to:

```text
Animal A-001
```

for two months.

Then the same:

```text
DEV-001
```

was reassigned to:

```text
Animal A-017
```

If SensorData only had:

```text
deviceId
```

then historical interpretation could become more complicated.

With:

```text
assignmentId
```

we have:

```text
SensorData
   ↓
ASN-001
   ↓
DEV-001 → A-001
```

and later:

```text
SensorData
   ↓
ASN-002
   ↓
DEV-001 → A-017
```

The finalized model intentionally contains `assignmentId` for this relationship. fileciteturn26file0

---

# 13. `animalId`

### Type
ObjectId → Animal

### Required
Yes in the current finalized model

### Purpose

Identifies the animal associated with the sensor-data record.

Example:

```text
animalId = ANM-001
```

---

# 14. Why Both `assignmentId` and `animalId`?

At first this looks redundant:

```text
assignmentId
animalId
```

because Assignment already contains:

```text
animalId
```

But they serve different query/provenance purposes.

```text
assignmentId
=
historical assignment context

animalId
=
direct animal reference
```

This makes animal-level analytics and filtering easier without always joining/resolving the assignment.

---

# 15. Example

```text
SensorData
deviceId = DEV-001
assignmentId = ASN-001
animalId = ANM-001
```

Interpretation:

```text
Device DEV-001
was operating under assignment ASN-001
for Animal ANM-001
when this measurement was recorded.
```

---

# 16. Farm/Device/Assignment/Animal Consistency

For animal-linked sensor data, a strong business consistency rule is:

```text
SensorData.farmId
==
Device.farmId

SensorData.assignmentId
→ same Device

SensorData.animalId
→ same Animal as assignment

SensorData.farmId
==
Animal.farmId
```

These consistency checks are implementation-level recommendations based on the finalized relationships.

---

# 17. `recordedAt`

### Type
Date

### Required
Yes

### Purpose

Stores **when the measurement/event actually occurred or was recorded by the source device/system**.

Example:

```text
recordedAt:
2026-08-17 10:30:00
```

This is the primary timestamp for time-series analysis.

---

# 18. `recordedAt` vs `createdAt`

These are not the same.

### `recordedAt`

```text
When the measurement happened.
```

### `createdAt`

```text
When the SensorData document was created in Liventra.
```

Example:

```text
Device measured:
10:20 AM

Network unavailable

Uploaded to Liventra:
10:35 AM
```

Then:

```text
recordedAt = 10:20
createdAt = 10:35
```

This distinction is extremely important for IoT systems.

---

# 19. Offline Device Example

```text
10:00
Device measures temperature

10:05
Device measures temperature

10:10
Device measures temperature

Network returns at 10:30

10:31
Data uploaded
```

SensorData should preserve:

```text
recordedAt = 10:00
recordedAt = 10:05
recordedAt = 10:10
```

rather than making all records appear to have occurred at:

```text
10:31
```

---

# 20. `metrics`

### Type
Nested Object

### Required
The `metrics` object is part of the finalized model.

### Purpose

Stores the actual measurable values reported by the device.

Finalized metrics:

```text
bodyTemperature
ruminationMinutes
activityScore
steps
waterIntake
feedIntake
milkYield
heartRate
respirationRate
batteryLevel
```

fileciteturn26file0turn26file1

---

# 21. Important Metric Design Concept

Not every device will necessarily provide every metric.

For example:

```text
Neck Collar
```

may provide:

```text
activityScore
steps
ruminationMinutes
```

while:

```text
Milk Meter
```

may provide:

```text
milkYield
```

and:

```text
Temperature Sensor
```

may provide:

```text
bodyTemperature
```

The finalized schema groups these possible measurements under one common `metrics` object.

---

# 22. `bodyTemperature`

### Type
Number

### Required
Conditional / depends on device capability

### Example

```text
bodyTemperature = 38.7
```

Purpose:

Stores a temperature measurement when the connected device supports body-temperature sensing.

The finalized model defines the field but does not specify a unit or valid numeric range. Therefore do **not** silently assume a unit/range as a finalized requirement.

---

# 23. `ruminationMinutes`

### Type
Number

### Required
Conditional

### Example

```text
ruminationMinutes = 420
```

Meaning:

```text
420 minutes of detected rumination
```

The exact measurement window should be interpreted according to the source device/integration.

The finalized model names the field but does not define a fixed aggregation period.

---

# 24. `activityScore`

### Type
Number

### Required
Conditional

### Example

```text
activityScore = 75
```

Purpose:

Stores an activity-related score from a smart device.

Important:

The finalized model does not define:

```text
0–100
```

or another exact scale.

Therefore the application should not pretend that a particular numeric range is part of the finalized DB specification unless the device/integration defines it.

---

# 25. `steps`

### Type
Number

### Required
Conditional

### Example

```text
steps = 1834
```

Purpose:

Stores a step/activity count when supported by the device.

The exact counting period is device/integration dependent unless separately specified.

---

# 26. `waterIntake`

### Type
Number

### Required
Conditional

### Example

```text
waterIntake = 18.5
```

Purpose:

Stores detected/estimated water intake.

The finalized model does not define the unit.

Therefore the unit must be established by the device/integration specification rather than invented in the DB model.

---

# 27. `feedIntake`

### Type
Number

### Required
Conditional

### Example

```text
feedIntake = 8.2
```

Purpose:

Stores detected/estimated feed intake where supported.

Again, the finalized model defines the field but not a unit.

---

# 28. `milkYield`

### Type
Number

### Required
Conditional

### Example

```text
milkYield = 12.5
```

Purpose:

Stores milk production measured by a compatible smart device such as a milk meter.

The project documentation identifies milk meters as part of the future smart-livestock integration direction. fileciteturn26file7

The finalized SensorData model does not define the unit or time period.

---

# 29. `heartRate`

### Type
Number

### Required
Conditional

### Example

```text
heartRate = 72
```

Purpose:

Stores heart-rate measurement when supported by the device.

The finalized model does not define a unit/time window, so those should come from the actual device/integration specification.

---

# 30. `respirationRate`

### Type
Number

### Required
Conditional

### Example

```text
respirationRate = 28
```

Purpose:

Stores respiration-related measurement when supported.

Again, the finalized model does not define the exact unit/measurement period.

---

# 31. `batteryLevel`

### Type
Number

### Required
Conditional

### Example

```text
batteryLevel = 82
```

Purpose:

Stores battery level reported with the sensor data.

This is useful for historical battery monitoring.

---

# 32. Device Battery vs SensorData Battery

There are two battery representations in the finalized design:

```text
Device.batteryLevel
```

and:

```text
SensorData.metrics.batteryLevel
```

They have different purposes.

### Device

```text
Current/latest battery state
```

### SensorData

```text
Historical battery reading at a particular recorded time
```

Example:

```text
08:00 → 90%
10:00 → 82%
12:00 → 75%
```

Device's current state:

```text
batteryLevel = 75
```

---

# 33. Why Historical Battery Data Matters

It allows future analytics such as:

```text
Battery drain over time
```

and:

```text
Estimated battery performance
```

without losing older readings.

The finalized model supports this by putting `batteryLevel` inside SensorData metrics.

---

# 34. Missing Metrics

A device may send only:

```text
bodyTemperature
batteryLevel
```

and not:

```text
milkYield
steps
heartRate
```

Those unsupported metrics should remain absent/null rather than being filled with fake zeros.

Conceptually:

```json
{
  "bodyTemperature": 38.7,
  "batteryLevel": 82
}
```

is different from:

```json
{
  "bodyTemperature": 38.7,
  "milkYield": 0
}
```

because `0` could mean an actual measured zero.

This is an implementation/data-quality recommendation.

---

# 35. `alerts`

### Type
Array of Objects

### Required
Optional / can be empty

### Structure

```text
alerts:[
    {
        type,
        severity,
        message
    }
]
```

This structure is explicitly defined in the finalized model. fileciteturn26file0

---

# 36. Why Alerts Are Embedded in SensorData

A measurement can trigger an alert at the moment it is recorded.

Example:

```text
bodyTemperature = abnormal
```

then:

```text
alerts:[
    {
        type: "Temperature",
        severity: "High",
        message: "Abnormal body temperature detected"
    }
]
```

This preserves the relationship:

```text
measurement
+
alert
```

within the same event.

---

# 37. `alerts.type`

### Type
String

### Required
Within alert object: application-defined

### Example

```text
Temperature
```

or:

```text
Low Battery
```

The finalized model does **not** define an enum for alert `type`.

Therefore alert types should be standardized at the application/integration level rather than treated as a source-defined enum.

---

# 38. `alerts.severity`

### Type
String

### Required
Within alert object: application-defined

### Example

```text
High
```

The finalized model does not provide a severity enum here.

Therefore:

```text
Low
Medium
High
Critical
```

should not be presented as finalized SensorData schema values unless separately defined by the implementation.

---

# 39. `alerts.message`

### Type
String

### Required
Within alert object: yes when an alert exists

### Example

```text
"Abnormal body temperature detected."
```

Purpose:

Human-readable explanation of the alert.

---

# 40. Multiple Alerts in One SensorData Record

Possible:

```text
alerts:[
    {
        type: "Temperature",
        severity: "High",
        message: "Abnormal temperature"
    },
    {
        type: "Battery",
        severity: "Warning",
        message: "Battery level is low"
    }
]
```

This is useful if one sensor event triggers multiple conditions.

---

# 41. Empty Alerts

Normal measurement:

```text
alerts: []
```

This means:

```text
No alert was attached to this sensor-data event.
```

---

# 42. Alert vs Notification

These are not identical.

### SensorData.alerts

Stores:

```text
Alert information attached to a measurement/event.
```

### Notification

Stores:

```text
A user-facing notification.
```

The finalized Notification model explicitly allows:

```text
referenceType = SensorData
```

so a SensorData event can be referenced by a notification. fileciteturn26file0

---

# 43. Sensor Alert → Notification Flow

Conceptually:

```text
Device
   ↓
SensorData
   ↓
Alert detected
   ↓
Create Notification
   ↓
User sees alert
```

Example:

```text
SensorData:
bodyTemperature = abnormal

alerts:
High temperature

↓

Notification:
"Animal A-001 has abnormal temperature."
```

The exact automatic notification trigger is application logic, not explicitly defined by the DB model.

---

# 44. `rawPayload`

### Type
Object / JSON-like data

### Required
Optional / depends on ingestion

### Purpose

Stores the original payload received from the device/integration source.

Example:

```json
{
  "device_serial": "SN-9988",
  "temp": 38.7,
  "rumination": 420,
  "battery": 82
}
```

The finalized model includes `rawPayload` without defining its exact internal structure. fileciteturn26file0

---

# 45. Why Store `rawPayload`?

Suppose vendor sends:

```json
{
  "temp": 38.7,
  "rum": 420
}
```

Liventra normalizes it into:

```text
metrics.bodyTemperature
metrics.ruminationMinutes
```

If the normalized transformation later has a bug, the original payload helps with:

```text
Debugging
Reprocessing
Vendor integration troubleshooting
Data provenance
```

---

# 46. Raw Payload vs Metrics

These serve different purposes.

```text
rawPayload
=
original source format
```

```text
metrics
=
Liventra's normalized structure
```

Example:

```text
Vendor payload:
temp = 38.7
rum = 420
```

Normalized:

```text
bodyTemperature = 38.7
ruminationMinutes = 420
```

---

# 47. Raw Payload Security

Raw payloads may contain:

```text
device identifiers
vendor metadata
technical information
```

so access should be restricted to appropriate backend/admin/integration contexts.

Do not expose raw vendor payloads unnecessarily to ordinary farm users.

This is a security recommendation.

---

# 48. Raw Payload Size

Because `rawPayload` can potentially be large, ingestion should avoid storing unnecessarily huge payloads.

The finalized model does not specify a maximum size.

A practical implementation should define limits to prevent oversized documents.

---

# 49. `source`

### Type
Enum

### Required
Yes

### Finalized Values

```text
API
MQTT
Bluetooth
Manual Import
```

fileciteturn26file0turn26file1

---

# 50. `API`

Meaning:

Data entered Liventra through an API-based integration.

Example:

```text
Commercial vendor platform
        ↓
REST API
        ↓
Liventra
        ↓
SensorData
```

---

# 51. `MQTT`

MQTT represents an IoT messaging-based ingestion path.

Conceptually:

```text
Device/Gateway
      ↓
MQTT Broker
      ↓
Liventra MQTT Consumer
      ↓
SensorData
```

The finalized model defines MQTT as a source value; the exact broker architecture is implementation-level.

---

# 52. `Bluetooth`

Data received through Bluetooth-based communication.

Example:

```text
Bluetooth Device
      ↓
Mobile/Gateway
      ↓
Liventra
      ↓
SensorData
```

The Device model also allows Bluetooth as a communication protocol. fileciteturn26file0

---

# 53. `Manual Import`

Useful when historical sensor/device data is imported from:

```text
CSV
Excel
JSON
Vendor export
```

rather than arriving through a live integration.

This is important for onboarding farms that already have historical smart-device data.

---

# 54. `source` vs `communicationProtocol`

These are different.

### Device.communicationProtocol

```text
What communication technology does the device use?
```

### SensorData.source

```text
How did this particular data record enter Liventra?
```

Example:

```text
Device:
communicationProtocol = Bluetooth

SensorData:
source = API
```

This could happen if a vendor platform receives Bluetooth data and Liventra later retrieves it through an API.

---

# 55. `createdAt`

### Type
Date

### Required
Yes

### Purpose

Stores when the SensorData document was created in Liventra.

Example:

```text
createdAt = 2026-08-17 10:35
```

Again:

```text
recordedAt
≠
createdAt
```

---

# 56. Complete SensorData Example

```json
{
  "farmId": "FARM-001",
  "deviceId": "DEV-001",
  "assignmentId": "ASN-001",
  "animalId": "ANM-001",
  "recordedAt": "2026-08-17T10:30:00Z",

  "metrics": {
    "bodyTemperature": 38.7,
    "ruminationMinutes": 420,
    "activityScore": 75,
    "steps": 1834,
    "waterIntake": 18.5,
    "feedIntake": 8.2,
    "milkYield": 12.5,
    "heartRate": 72,
    "respirationRate": 28,
    "batteryLevel": 82
  },

  "alerts": [
    {
      "type": "Temperature",
      "severity": "High",
      "message": "Abnormal body temperature detected"
    }
  ],

  "rawPayload": {
    "temp": 38.7,
    "rum": 420,
    "activity": 75,
    "battery": 82
  },

  "source": "API",
  "createdAt": "2026-08-17T10:32:00Z"
}
```

This is an illustrative example. The finalized schema defines the fields but does not define units, alert enums, or vendor-specific payload formats. fileciteturn26file0

---

# 57. Complete Historical Example

Suppose:

```text
DEV-001
```

was assigned to:

```text
ANM-001
```

through:

```text
ASN-001
```

SensorData:

```text
10:00
DEV-001
ASN-001
ANM-001
Temperature = 38.5

10:10
DEV-001
ASN-001
ANM-001
Temperature = 38.7

10:20
DEV-001
ASN-001
ANM-001
Temperature = 39.0
```

Later the device is reassigned:

```text
ASN-001 → Removed
ASN-002 → Active
DEV-001 → ANM-017
```

New SensorData:

```text
11:00
DEV-001
ASN-002
ANM-017
Temperature = 38.4
```

The historical records remain correctly attributable.

---

# 58. SensorData and DeviceAssignment

This is the most important relationship:

```text
DeviceAssignment
      │
      ├── deviceId
      ├── animalId
      ├── assignedDate
      └── removedDate
               │
               ▼
          SensorData
          assignmentId
```

This gives the application a historical timeline.

---

# 59. SensorData and Animal

Direct:

```text
animalId
```

makes queries easy:

```text
Get all sensor data for Animal A-001
```

without having to search every DeviceAssignment first.

---

# 60. SensorData and Device

Direct:

```text
deviceId
```

allows:

```text
Get all data generated by Device DEV-001
```

Useful for:

```text
Device diagnostics
Battery analysis
Device performance
Data-quality checks
```

---

# 61. SensorData and Farm

Direct:

```text
farmId
```

allows:

```text
Get today's telemetry for Farm A
```

efficiently.

This is especially important because SensorData is expected to become one of the highest-volume collections.

---

# 62. Time-Series Nature of SensorData

SensorData is naturally time-oriented:

```text
recordedAt
```

is the core analytical dimension.

Typical queries:

```text
Last 1 hour
Today
Last 7 days
Last 30 days
Custom date range
```

---

# 63. Time-Based Query Example

Conceptually:

```text
farmId = FARM-001

recordedAt:
>= startTime
<= endTime
```

Then optionally:

```text
animalId = ANM-001
```

or:

```text
deviceId = DEV-001
```

---

# 64. Recommended Indexes

For expected query patterns, useful indexes would include:

```javascript
{ farmId: 1, recordedAt: -1 }
```

for:

```text
farm sensor history
```

```javascript
{ deviceId: 1, recordedAt: -1 }
```

for:

```text
device telemetry
```

```javascript
{ animalId: 1, recordedAt: -1 }
```

for:

```text
animal monitoring
```

and potentially:

```javascript
{ assignmentId: 1, recordedAt: -1 }
```

for historical assignment analysis.

These are implementation recommendations; the finalized DB model does not specify indexes.

---

# 65. Most Important Index

For the dashboard:

```text
Show latest sensor data for this farm
```

a compound index such as:

```javascript
{ farmId: 1, recordedAt: -1 }
```

is a strong candidate.

---

# 66. Latest Sensor Data

The dashboard often needs:

```text
Latest reading per animal
```

rather than every historical reading.

That should be calculated efficiently from:

```text
animalId
+
recordedAt
```

or maintained through a separate current-state mechanism if scale later requires it.

Do not duplicate entire sensor histories inside Animal.

---

# 67. SensorData and Dashboard

Possible dashboard cards:

```text
Active Smart Devices
Online Devices
Animals Monitored
Recent Alerts
Average Activity
Average Temperature
Recent Milk Yield
Battery Warnings
```

These are derived analytics, not SensorData fields.

---

# 68. SensorData and Animal Health Monitoring

The project documentation states that Liventra aims to integrate health monitoring and future smart-device capabilities. fileciteturn26file7turn26file18

SensorData can provide observations that support:

```text
Animal health alerts
Behavior monitoring
Abnormal condition detection
```

But SensorData should not automatically overwrite:

```text
Animal.healthStatus
```

without a clearly defined business rule.

---

# 69. SensorData Does Not Equal Diagnosis

Example:

```text
bodyTemperature = abnormal
```

means:

```text
Observed sensor measurement
```

It does **not automatically mean**:

```text
Disease = X
```

or:

```text
Animal.healthStatus = Sick
```

A diagnostic workflow should interpret sensor evidence.

---

# 70. SensorData Alert vs Treatment

Example:

```text
SensorData
temperature abnormal
```

can generate:

```text
Notification
```

Then a human/authorized workflow can create:

```text
Treatment
```

if clinically appropriate.

This keeps:

```text
Observation
```

separate from:

```text
Medical decision
```

---

# 71. SensorData and Notification

The finalized Notification model explicitly supports:

```text
referenceType = SensorData
referenceId
```

fileciteturn26file0

Therefore:

```text
SensorData
    ↓
Notification
```

is directly supported by the finalized schema.

---

# 72. SensorData and Report

The finalized Report model supports:

```text
reportType = Sensor
```

fileciteturn26file0

Therefore SensorData can feed:

```text
Sensor reports
Device reports
Health analytics
```

through the reporting layer.

---

# 73. SensorData and AuditLog

Current finalized AuditLog `entityType` does **not include `SensorData`**. fileciteturn26file1

This is an important schema consideration.

Because SensorData can arrive automatically at high frequency, auditing every single sensor reading through the normal AuditLog may also create huge volumes.

The current finalized design therefore does not directly make SensorData an AuditLog entity.

Do not change this here; just remember it for implementation.

---

# 74. Why We Should Not Audit Every SensorData Record

Suppose:

```text
100 devices
```

each send:

```text
1 record / minute
```

That becomes:

```text
144,000 records/day
```

If every reading also generated a full AuditLog entry, the audit collection could become unnecessarily huge.

A better conceptual distinction is:

```text
SensorData
=
telemetry history

AuditLog
=
user/system administrative actions
```

This is architectural reasoning, not a source-defined rule.

---

# 75. Raw Payload and Reprocessing

Suppose the vendor sends:

```json
{
  "temp": 38.7
}
```

and Liventra initially maps:

```text
temp → bodyTemperature
```

Later the integration logic changes.

Raw payload allows future reprocessing:

```text
rawPayload
   ↓
New parser
   ↓
Normalized metrics
```

This can be valuable for future integration maturity.

---

# 76. Data Validation Before Save

Recommended ingestion flow:

```text
Incoming Data
      ↓
Authenticate Source
      ↓
Identify Device
      ↓
Validate Device/Farm
      ↓
Resolve Assignment
      ↓
Validate Timestamp
      ↓
Normalize Metrics
      ↓
Generate Alerts
      ↓
Save SensorData
```

This is implementation architecture.

---

# 77. Device Validation

Incoming data should not blindly create a SensorData record.

Backend should verify:

```text
deviceId exists
device belongs to expected farm
device is recognized
```

before accepting telemetry.

---

# 78. Assignment Validation

For animal-linked telemetry:

```text
assignmentId exists
assignment belongs to deviceId
assignment relates to animalId
```

should be validated.

This prevents:

```text
deviceId = DEV-001
assignmentId = ASN-999
```

where ASN-999 belongs to another device.

---

# 79. Farm Validation

Recommended:

```text
SensorData.farmId
==
Device.farmId
```

and:

```text
SensorData.assignmentId.farmId
==
SensorData.farmId
```

and:

```text
Animal.farmId
==
SensorData.farmId
```

for animal-linked data.

---

# 80. Timestamp Validation

The system should protect against obviously invalid timestamps.

Examples:

```text
recordedAt far in the future
```

or:

```text
malformed timestamp
```

The finalized model does not define an allowed clock-skew window, so the exact rule should be configured at implementation time.

---

# 81. Duplicate Data

IoT systems may resend the same event.

Example:

```text
Device sends record
Network fails
Device retries
```

The same measurement may arrive twice.

The finalized SensorData model does not include a dedicated:

```text
eventId
```

or:

```text
messageId
```

field.

Therefore duplicate detection is a future integration concern.

Possible strategies include using vendor-provided IDs inside `rawPayload` or a derived uniqueness strategy, but this should be designed according to actual device protocols rather than invented now.

---

# 82. Out-of-Order Data

Data can arrive:

```text
10:20
10:10
10:15
```

because of network delays.

Do not assume insertion order equals measurement order.

Always use:

```text
recordedAt
```

for chronological analysis.

---

# 83. Late Data

A device may upload old measurements after reconnecting.

Example:

```text
createdAt = 12:00
recordedAt = 10:30
```

This is not necessarily an error.

It can represent:

```text
delayed synchronization
```

---

# 84. Manual Import

Suppose a farm previously used another system.

They export:

```text
CSV
```

and import historical sensor data.

Then:

```text
source = Manual Import
```

This preserves the provenance of those records.

---

# 85. API Ingestion

Commercial integration:

```text
Vendor Cloud
      ↓
API
      ↓
Liventra
      ↓
SensorData
```

Store:

```text
source = API
```

---

# 86. MQTT Ingestion

IoT messaging:

```text
Device
 ↓
MQTT
 ↓
Broker
 ↓
Liventra
 ↓
SensorData
```

Store:

```text
source = MQTT
```

---

# 87. Bluetooth Ingestion

Example:

```text
Smart device
 ↓
Bluetooth
 ↓
Gateway/mobile
 ↓
Liventra
```

Store:

```text
source = Bluetooth
```

---

# 88. SensorData and Device Connection Status

Device has:

```text
connectionStatus
```

SensorData has:

```text
recordedAt
createdAt
```

These work together.

Example:

```text
No SensorData for a long period
+
Device.lastSyncAt old
```

can indicate a possible connectivity problem.

However, absence of data alone does not prove hardware failure.

---

# 89. SensorData and Device Health

Similarly:

```text
No new data
```

does not automatically mean:

```text
healthStatus = Faulty
```

Possible reasons:

```text
Network issue
Gateway issue
Battery depletion
Maintenance
Device failure
```

These should be distinguished by the monitoring layer.

---

# 90. Battery Alert Example

SensorData:

```text
metrics.batteryLevel = 15
```

Alert:

```text
type = Battery
severity = Warning
message = "Low device battery"
```

Then:

```text
Notification
referenceType = SensorData
referenceId = SENSOR-DATA-ID
```

This creates a complete alert chain.

---

# 91. Temperature Alert Example

SensorData:

```text
bodyTemperature = abnormal
```

Alert:

```text
type = Temperature
severity = High
message = "Abnormal temperature detected."
```

Then a user-facing notification can be generated.

The actual threshold is **not defined in the finalized DB model**.

---

# 92. Activity Alert Example

If a device detects an unusual activity pattern:

```text
activityScore
```

can be stored in metrics and an alert can be attached.

The exact detection algorithm is not defined by the current DB model.

---

# 93. Rumination Alert Example

If:

```text
ruminationMinutes
```

falls outside an application-defined expected range:

```text
SensorData
+
alert
```

can be generated.

Again, the finalized model defines the storage fields, not the scientific threshold.

---

# 94. Water Intake Monitoring

SensorData can store:

```text
waterIntake
```

and potentially produce an alert if an application-level rule detects unusual intake.

The finalized DB model does not define:

```text
normal water intake
```

or threshold values.

---

# 95. Feed Intake Monitoring

Likewise:

```text
feedIntake
```

can support future analytics/alerts.

But:

```text
feedIntake < X
```

is not a finalized rule.

---

# 96. Milk Yield Monitoring

For compatible devices:

```text
milkYield
```

can be stored in SensorData.

This can later support:

```text
production trends
animal monitoring
farm analytics
```

The project's broader system already contains `MilkRecord` for production records, so the IoT data and manually/operationally recorded milk data should be kept conceptually distinct unless a synchronization rule is explicitly defined.

---

# 97. SensorData vs MilkRecord

Important distinction:

```text
SensorData
=
raw/telemetry observation from smart device
```

```text
MilkRecord
=
farm production record
```

A future integration may use SensorData to help populate or validate MilkRecord, but the finalized DB model does not define automatic synchronization between them.

---

# 98. SensorData vs WeightRecord

Similarly:

```text
SensorData
```

may eventually provide smart-device observations related to animal activity/health.

`WeightRecord` explicitly supports:

```text
measurementMethod = Smart Device
```

in the finalized model. fileciteturn26file19

However, the current SensorData model does not explicitly contain:

```text
weight
```

so no direct weight metric is currently defined inside SensorData.

---

# 99. Important Current Schema Limitation: No `weight`

The finalized SensorData metrics contain:

```text
bodyTemperature
ruminationMinutes
activityScore
steps
waterIntake
feedIntake
milkYield
heartRate
respirationRate
batteryLevel
```

but **not `weight`**. fileciteturn26file0

Therefore do not add:

```text
metrics.weight
```

during implementation unless the schema is intentionally revised.

---

# 100. Important Current Schema Limitation: No `humidity`

The Device model includes:

```text
Environment Sensor
```

but SensorData does not define:

```text
humidity
```

or other environmental metrics.

Therefore an Environment Sensor may be registered, but the finalized SensorData structure does not currently define a dedicated humidity field.

This is a design consideration for future IoT expansion.

---

# 101. Important Current Schema Limitation: No Location Field

SensorData currently has:

```text
farmId
deviceId
assignmentId
animalId
```

but no:

```text
shedId
penId
stallId
location
```

So location-specific sensor data is not directly represented by the finalized SensorData model.

---

# 102. Important Current Schema Limitation: No Explicit Unit

Several metrics have no dedicated unit field:

```text
bodyTemperature
waterIntake
feedIntake
milkYield
heartRate
respirationRate
```

Therefore the actual units need to be standardized by the integration/application layer.

Do not silently add a `unit` field to the finalized model here.

---

# 103. Important Current Schema Limitation: No Metric Timestamp

There is only:

```text
recordedAt
```

for the whole SensorData record.

So the current model assumes the metrics inside one record share the same measurement timestamp.

If a future device sends:

```text
temperature at 10:00
milk yield at 10:02
battery at 10:03
```

inside one document, the current schema does not provide separate timestamps for each metric.

---

# 104. Important Current Schema Limitation: No Event ID

There is no explicit:

```text
eventId
messageId
sequenceNumber
```

in the finalized model.

This can matter for:

```text
Duplicate detection
Retry handling
Exactly-once ingestion
Device sequence tracking
```

The raw payload may contain vendor-specific identifiers, but the finalized model does not require them.

---

# 105. Important Current Schema Limitation: Alert Enum

The structure:

```text
type
severity
message
```

is defined, but:

```text
type
severity
```

do not have finalized enums.

Therefore the application should establish a controlled vocabulary before production to avoid:

```text
"Temp"
"Temperature"
"Body Temp"
```

being treated as unrelated types.

---

# 106. Important Current Schema Limitation: Raw Payload Schema

`rawPayload` is defined, but its internal schema is not.

This is good for flexibility, but implementation should still establish:

```text
maximum size
allowed format
security policy
retention policy
```

according to actual integrations.

---

# 107. SensorData Data Lifecycle

Conceptually:

```text
Device produces measurement
        ↓
Integration receives data
        ↓
Validate
        ↓
Normalize
        ↓
Attach device/assignment/animal
        ↓
Generate alerts if applicable
        ↓
Store SensorData
        ↓
Update current Device state if applicable
        ↓
Trigger Notification if applicable
        ↓
Analytics / Reports
```

---

# 108. Current State vs Historical Data

A very important separation:

```text
Device
=
current state
```

```text
SensorData
=
historical observations
```

Example:

```text
Device.batteryLevel = 75
```

while:

```text
SensorData:
90%
85%
82%
75%
```

preserves history.

---

# 109. SensorData Retention

The finalized DB model does not specify how long SensorData must be retained.

Because this is potentially high-volume data, retention policy should be considered before production.

Possible future policy:

```text
Raw data:
longer retention

Aggregated data:
long-term retention
```

But this is not part of the finalized schema and should not be assumed as an existing requirement.

---

# 110. SensorData Aggregation

For dashboards, raw data may be aggregated:

```text
minute-level
→ hourly average

hourly
→ daily summary

daily
→ monthly trend
```

The aggregation result should not overwrite the original SensorData.

This preserves historical raw telemetry.

---

# 111. Example Daily Analytics

From:

```text
SensorData
```

the system could calculate:

```text
Average body temperature
Total activity
Average rumination
Total water intake
Total feed intake
Total milk yield
Average heart rate
Average respiration
Battery trend
```

These are derived values.

They should not be stored as raw SensorData fields unless explicitly required.

---

# 112. Real-Time Dashboard

A future real-time flow could be:

```text
Device
 ↓
MQTT/API
 ↓
SensorData Service
 ↓
Database
 ↓
Socket.IO
 ↓
Frontend Dashboard
```

For example:

```text
New sensor reading
→ save SensorData
→ emit sensor.data.created
→ dashboard updates
```

The finalized DB model itself does not specify Socket.IO; this is application architecture.

---

# 113. Why Database First, Realtime Second?

Realtime delivery should not be the only storage mechanism.

Correct:

```text
Incoming data
 ↓
Validate
 ↓
Persist SensorData
 ↓
Emit realtime event
```

Not:

```text
Incoming data
 ↓
Emit to browser
 ↓
Maybe save later
```

because users could miss data if the browser is disconnected.

---

# 114. Realtime Alert Flow

```text
Sensor reading
      ↓
Store SensorData
      ↓
Alert generated
      ↓
Create Notification
      ↓
Emit realtime event
      ↓
Frontend shows alert
```

This combines:

```text
persistent history
+
user notification
+
realtime UX
```

---

# 115. SensorData Query Examples

### By farm

```text
farmId = FARM-001
```

### By animal

```text
animalId = ANM-001
```

### By device

```text
deviceId = DEV-001
```

### By assignment

```text
assignmentId = ASN-001
```

### By time

```text
recordedAt:
start → end
```

These can be combined.

---

# 116. Example Animal Monitoring Query

Conceptually:

```text
animalId = ANM-001
recordedAt >= startOfDay
recordedAt <= endOfDay
```

Sort:

```text
recordedAt descending
```

This gives the latest readings first.

---

# 117. Example Device Monitoring Query

```text
deviceId = DEV-001
recordedAt >= last24Hours
```

Useful for:

```text
Device diagnostic page
Battery graph
Data availability
Telemetry history
```

---

# 118. Example Assignment History Query

```text
assignmentId = ASN-001
sort by recordedAt ascending
```

This gives the sensor history for the exact period that the device was assigned under that assignment.

---

# 119. SensorData and Reassignment

This is the key historical scenario:

```text
Before:
DEV-001 → ANM-001
Assignment ASN-001

After:
DEV-001 → ANM-017
Assignment ASN-002
```

SensorData:

```text
Old:
assignmentId = ASN-001
animalId = ANM-001

New:
assignmentId = ASN-002
animalId = ANM-017
```

No historical record needs to be rewritten.

---

# 120. SensorData and Animal Sold

Suppose:

```text
ANM-001
lifeStatus = Sold
```

The current assignment is closed.

Historical SensorData remains available.

This allows:

```text
Pre-sale monitoring history
```

to remain intact.

---

# 121. SensorData and Animal Dead

Similarly:

```text
Animal dead
```

does not mean:

```text
Delete sensor data
```

Historical telemetry remains valuable for:

```text
medical history
research
post-analysis
```

subject to the project's retention/privacy policy.

---

# 122. SensorData and Device Replacement

Old device:

```text
DEV-001
```

new device:

```text
DEV-009
```

SensorData remains split by:

```text
deviceId
```

This allows device-performance comparison.

---

# 123. SensorData and Device Fault

If:

```text
DEV-001
healthStatus = Faulty
```

there may be no new SensorData.

But old SensorData remains.

This is useful for diagnosing:

```text
when did the device stop producing valid data?
```

---

# 124. SensorData and Data Quality

A SensorData record can technically exist while containing:

```text
missing metrics
```

or:

```text
unexpected values
```

Therefore ingestion validation is important.

Possible validation levels:

```text
Schema validation
Type validation
Range validation
Timestamp validation
Device validation
Assignment validation
Duplicate detection
```

Exact scientific ranges should come from device/domain specifications, not be invented from the DB schema.

---

# 125. SensorData Type Validation

For metrics:

```text
bodyTemperature
ruminationMinutes
activityScore
steps
waterIntake
feedIntake
milkYield
heartRate
respirationRate
batteryLevel
```

numeric values are logically expected because they represent measurements.

The finalized model, however, lists field names but does not explicitly provide Mongoose/TypeScript primitive types.

Therefore the exact implementation types should be documented in the schema code.

---

# 126. Null vs Zero

This is important.

Suppose a milk meter does not provide:

```text
heartRate
```

Prefer:

```text
heartRate = null/absent
```

rather than:

```text
heartRate = 0
```

because:

```text
0
```

is a value, while:

```text
missing
```

means the metric was not provided.

---

# 127. SensorData and Manual Import

When importing historical data:

```text
source = Manual Import
```

The importer should still preserve:

```text
recordedAt
deviceId
animalId
```

where available.

Do not replace historical measurement dates with import time.

---

# 128. Import Validation

For manual imports:

```text
Validate file
      ↓
Validate columns
      ↓
Resolve Device
      ↓
Resolve Animal/Assignment
      ↓
Normalize metrics
      ↓
Create SensorData
      ↓
source = Manual Import
```

The finalized model defines the source value but not the import workflow.

---

# 129. API Ingestion Security

For API integrations, the system should authenticate the source.

Possible mechanisms:

```text
API key
OAuth
signed requests
vendor credentials
```

The finalized Device/SensorData models do not define credential storage.

Do not put secrets into:

```text
rawPayload
```

or:

```text
notes
```

as a substitute for secure credential management.

---

# 130. MQTT Security

For MQTT, the integration layer should handle:

```text
broker authentication
topic authorization
message validation
```

These are outside the SensorData schema.

---

# 131. SensorData Immutability

A strong recommendation:

Once a sensor measurement is stored:

```text
Do not casually edit historical metrics.
```

If a correction is needed, use a controlled correction process rather than silently changing historical telemetry.

Why?

Because SensorData is historical evidence of:

```text
what the system received
when it received/measured it
```

---

# 132. Raw Payload Helps Preserve Provenance

If normalized data is corrected later:

```text
rawPayload
```

can preserve what the source originally sent.

This is useful for debugging and research reproducibility.

---

# 133. SensorData and Research

Because Liventra is designed as a research-oriented/intelligent platform with future smart-device integration, historical SensorData can potentially support:

```text
behavior analysis
health monitoring
prediction
anomaly detection
production analysis
```

The project guide explicitly positions the system as future-ready for smart livestock technologies. fileciteturn26file7

These are future uses, not current finalized SensorData algorithms.

---

# 134. SensorData and Machine Learning

Future ML pipelines may consume:

```text
SensorData
+
Animal
+
WeightRecord
+
MilkRecord
+
Treatment
+
Pregnancy
```

to build features.

But the database should store source observations rather than precompute every ML feature inside SensorData.

---

# 135. SensorData Feature Engineering

Example derived features:

```text
7-day average activity
24-hour average temperature
rumination trend
water-intake trend
milk-yield trend
battery drain rate
```

These are analytical features.

They are not fields in the finalized SensorData model.

---

# 136. SensorData and Alert Thresholds

The DB stores:

```text
alerts
```

but does not store:

```text
threshold configuration
```

Therefore threshold logic must exist elsewhere.

For example:

```text
bodyTemperature > configured threshold
→ alert
```

The threshold should be maintained by a rule/configuration layer rather than hidden inside the SensorData schema.

---

# 137. SensorData and Rule Versioning

Unlike FeedRecommendation, the finalized SensorData model does **not** contain:

```text
ruleVersion
```

Therefore alert-generation rule version is not currently persisted directly in SensorData.

This is a schema consideration if research/auditability of IoT alert rules becomes important.

Do not add it here unless the project requirement changes.

---

# 138. SensorData and Notification Lifecycle

SensorData:

```text
created
```

may trigger:

```text
Notification
```

Notification then has its own lifecycle:

```text
Active
Archived
Expired
```

The finalized Notification model supports this. fileciteturn26file0

SensorData itself has no status field.

---

# 139. SensorData Has No `isActive`

Correct.

SensorData represents historical events, not an active/inactive asset.

Therefore:

```text
isActive
```

belongs to Device, not SensorData.

---

# 140. SensorData Has No `updatedAt`

The finalized model contains:

```text
createdAt
```

but does not define:

```text
updatedAt
```

This is meaningful because telemetry records should generally behave like historical event records rather than editable master-data documents.

Do not add `updatedAt` to the finalized model here.

---

# 141. SensorData Has No `status`

Correct.

There is no:

```text
status
```

field in the finalized SensorData model.

Operational state belongs to:

```text
Device
```

and assignment state belongs to:

```text
DeviceAssignment
```

---

# 142. SensorData Has No `userId`

Correct.

SensorData can be generated automatically by:

```text
API
MQTT
Bluetooth
```

so a human user does not necessarily exist for every record.

Manual imports are identified through:

```text
source = Manual Import
```

rather than requiring every telemetry record to have a `userId`.

---

# 143. SensorData and `createdBy`

Also not present.

Again, automatic IoT ingestion does not necessarily have a human creator.

The system can still determine ingestion provenance through:

```text
source
deviceId
rawPayload
createdAt
```

and integration logs where applicable.

---

# 144. SensorData Volume

Potentially:

```text
Device
→ many SensorData
```

Therefore pagination should be used for UI history pages.

Do not load:

```text
all SensorData
```

for an animal/device into the browser at once.

---

# 145. SensorData Pagination

Example conceptual API:

```text
GET /api/sensor-data?animalId=ANM-001&page=1&limit=50
```

Sort:

```text
recordedAt DESC
```

This is an implementation example.

---

# 146. SensorData Aggregation

For charts:

```text
raw SensorData
      ↓
Aggregation pipeline
      ↓
Hourly/Daily summaries
      ↓
Chart
```

Do not necessarily create a new database record for every UI aggregation unless caching is required.

---

# 147. Data Archiving

At large scale:

```text
Recent data
→ hot storage

Older data
→ archive strategy
```

The finalized model does not define an archival mechanism.

This is a future scalability concern.

---

# 148. Complete IoT Data Flow

```text
                 PHYSICAL DEVICE
                        │
                        ▼
              Communication Layer
          ┌─────────┬────────┬────────┐
          │         │        │        │
       Bluetooth   MQTT     API     Import
          │         │        │        │
          └─────────┴────────┴────────┘
                        │
                        ▼
                Ingestion Service
                        │
                        ▼
                 Validate Device
                        │
                        ▼
              Resolve Assignment
                        │
                        ▼
                Normalize Metrics
                        │
                        ▼
                 Generate Alerts
                        │
                        ▼
                   SensorData
                   ┌────┴────┐
                   │         │
                   ▼         ▼
              Notification  Analytics
                   │         │
                   ▼         ▼
                 User      Reports
```

---

# 149. Complete Relationship Graph

```text
Farm
 │
 ├───────────────┐
 │               │
 ▼               ▼
Device        Animal
 │               │
 │               │
 └──────┬────────┘
        ▼
DeviceAssignment
        │
        ▼
   SensorData
        │
   ┌────┴─────┐
   ▼          ▼
Alerts     Raw Payload
   │
   ▼
Notification
```

The finalized DB models directly support:

```text
Device → SensorData
DeviceAssignment → SensorData
Animal → SensorData
SensorData → Notification
SensorData → Report
```

while direct SensorData support in AuditLog is currently absent. fileciteturn26file0turn26file1

---

# 150. Example: Full Animal Monitoring Scenario

Animal:

```text
ANM-001
```

Device:

```text
DEV-001
Neck Collar
```

Assignment:

```text
ASN-001
DEV-001 → ANM-001
Active
```

SensorData:

```text
10:00
temperature = 38.4
activity = 72
rumination = 410
battery = 90

10:10
temperature = 38.6
activity = 75
rumination = 420
battery = 89

10:20
temperature = 39.4
activity = 42
rumination = 350
battery = 88
```

The 10:20 event might generate an application-defined alert.

Then:

```text
SensorData
   ↓
Notification
   ↓
Farm user
```

---

# 151. Example: Reassignment Scenario

Before:

```text
DEV-001
ASN-001
ANM-001
```

SensorData:

```text
assignmentId = ASN-001
animalId = ANM-001
```

After reassignment:

```text
DEV-001
ASN-002
ANM-017
```

New SensorData:

```text
assignmentId = ASN-002
animalId = ANM-017
```

No old SensorData is changed.

---

# 152. Example: Offline Synchronization

Device records:

```text
10:00
10:05
10:10
```

Network unavailable.

At:

```text
10:30
```

data arrives.

SensorData:

```text
recordedAt = 10:00
createdAt = 10:30

recordedAt = 10:05
createdAt = 10:30

recordedAt = 10:10
createdAt = 10:30
```

This preserves measurement chronology.

---

# 153. Example: Manual Import

Historical vendor CSV:

```text
Date, Device, Animal, Temperature
2026-08-01, DEV-001, ANM-001, 38.5
2026-08-02, DEV-001, ANM-001, 38.7
```

After import:

```text
source = Manual Import
```

while:

```text
recordedAt
```

preserves the original measurement dates.

---

# 154. Example: Raw Payload + Normalization

Vendor sends:

```json
{
  "temp": 38.7,
  "rum": 420,
  "act": 75,
  "bat": 82
}
```

Liventra stores:

```text
rawPayload:
{
    "temp": 38.7,
    "rum": 420,
    "act": 75,
    "bat": 82
}
```

and normalized:

```text
metrics:
{
    bodyTemperature: 38.7,
    ruminationMinutes: 420,
    activityScore: 75,
    batteryLevel: 82
}
```

This is a strong integration pattern because the original payload is preserved while the application gets a standardized structure.

---

# 155. Common Mistakes to Avoid

### Mistake 1
Embedding all sensor records inside Device.

**Problem:** massive document growth.

### Mistake 2
Using `createdAt` instead of `recordedAt`.

**Problem:** delayed/offline data gets the wrong measurement time.

### Mistake 3
Dropping `assignmentId`.

**Problem:** historical device reassignment becomes harder to interpret.

### Mistake 4
Treating missing metric as zero.

**Problem:** missing data becomes fake measurement.

### Mistake 5
Treating sensor alert as diagnosis.

**Problem:** observation ≠ medical diagnosis.

### Mistake 6
Trusting incoming `farmId`.

**Problem:** possible cross-farm data injection.

### Mistake 7
Deleting historical SensorData when an animal/device is retired.

**Problem:** destroys historical evidence.

### Mistake 8
Editing old sensor readings casually.

**Problem:** destroys telemetry provenance.

---

# 156. Business Rules Summary

```text
1. SensorData stores historical telemetry/events.
2. Every record belongs to a farm.
3. Every record references a device.
4. Every animal-linked record references an assignment.
5. The finalized model also directly stores animalId.
6. Device, assignment, animal, and farm relationships must be consistent.
7. recordedAt represents measurement/event time.
8. createdAt represents SensorData document creation time.
9. Metrics are stored under the metrics object.
10. Not every device must provide every metric.
11. Missing metrics should not automatically become zero.
12. Alerts are stored with the sensor event.
13. Alert type/severity are not finalized enums in the current model.
14. rawPayload preserves the source payload when available.
15. source identifies how data entered Liventra.
16. Device.communicationProtocol and SensorData.source are different concepts.
17. SensorData should remain separate from Device.
18. Historical sensor records should be retained according to future retention policy.
19. Reassignment must not rewrite historical SensorData.
20. assignmentId preserves assignment context.
21. SensorData can generate user Notifications.
22. SensorData can feed Sensor reports.
23. Current AuditLog does not directly include SensorData.
24. SensorData should not automatically diagnose disease.
25. SensorData should not automatically overwrite Animal health status without a defined rule.
26. Incoming telemetry should be validated before persistence.
27. Farm isolation must be enforced server-side.
28. Duplicate/retry handling needs integration-level design.
29. Out-of-order data should be ordered by recordedAt, not insertion order.
30. The current model does not define metric units.
31. The current model does not define metric-specific timestamps.
32. The current model does not define event/message IDs.
33. The current model does not define weight or humidity metrics.
34. The current model does not define location fields for sensor data.
35. SensorData has no isActive/status/updatedAt fields in the finalized schema.
```

The structure/enums in rules 1–16 and 21–35 are grounded in the finalized DB model; the validation, retention, indexing, and ingestion recommendations are implementation-level guidance rather than source-defined requirements. fileciteturn26file0turn26file1

---

# 157. Current Schema Review — Important Gaps

## Gap 1 — Metric Units

Current:

```text
waterIntake
feedIntake
milkYield
```

have no explicit units.

The DB model defines the fields but not units.

**Implementation decision needed before real IoT integration.**

---

## Gap 2 — Alert Enums

Current:

```text
type
severity
```

have no enum.

**Recommendation:** standardize these before multiple device vendors are integrated.

---

## Gap 3 — Event/Message ID

No:

```text
eventId
messageId
sequenceNumber
```

exists.

This may become important for duplicate detection.

---

## Gap 4 — Environmental Metrics

Device supports:

```text
Environment Sensor
```

but SensorData does not explicitly contain:

```text
humidity
airTemperature
airQuality
```

So environmental monitoring is only partially represented by the current schema.

---

## Gap 5 — Location

No:

```text
shed
pen
stall
location
```

exists inside SensorData.

Therefore farm/shed-level sensor data is not fully modeled.

---

## Gap 6 — Weight

No:

```text
weight
```

metric exists.

The finalized `WeightRecord` separately supports:

```text
measurementMethod = Smart Device
```

so weight integration is conceptually possible elsewhere, but not directly through the current SensorData metrics. fileciteturn26file19

---

## Gap 7 — AuditLog

Current AuditLog entityType does not include:

```text
SensorData
```

This may be intentional because high-frequency telemetry should not necessarily be audited like user CRUD actions.

---

# 158. Recommended Implementation Philosophy

Keep the database responsibilities clean:

```text
Device
→ master/current state

DeviceAssignment
→ relationship/history

SensorData
→ raw historical telemetry

Notification
→ user-facing alert

Report
→ generated output

AuditLog
→ administrative/system actions
```

This prevents one collection from becoming responsible for everything.

---

# 159. Final SensorData Architecture

```text
                     FARM
                       │
                       ▼
                    DEVICE
                       │
                       ▼
               DEVICE ASSIGNMENT
                       │
                       ▼
                  SENSOR DATA
                ┌──────┼──────┐
                │      │      │
                ▼      ▼      ▼
             Metrics Alerts RawPayload
                │      │
                │      ▼
                │  Notification
                │
                ▼
             Analytics
                │
          ┌─────┴─────┐
          ▼           ▼
       Dashboard     Report
```

---

# 160. Final Status

**Model:** `SensorData`  
**Number:** 27  
**Status:** ✅ Upcoming / fully documented

```text
Identity                    ✓
Farm relationship           ✓
Device relationship        ✓
Assignment relationship    ✓
Animal relationship        ✓
Measurement timestamp      ✓
All finalized metrics      ✓
Alerts structure            ✓
Raw payload                 ✓
Source enum                 ✓
Creation timestamp          ✓
Historical telemetry        ✓
Device reassignment logic   ✓
Offline synchronization     ✓
Data provenance             ✓
Notification relationship   ✓
Report relationship         ✓
Security/farm isolation     ✓
Validation logic            ✓
Index recommendations      ✓
Current schema gaps         ✓
IoT ingestion architecture  ✓
Realtime architecture       ✓
```

## Most Important Concept

```text
Device
   ↓
"Who/what is the physical device?"

DeviceAssignment
   ↓
"Which animal was this device assigned to at this time?"

SensorData
   ↓
"What did that device measure at that time?"
```

And the historical chain is:

```text
SensorData
    ↓
assignmentId
    ↓
DeviceAssignment
    ↓
deviceId + animalId
    ↓
Device + Animal
```

That chain is what makes Liventra's future IoT data **traceable, farm-scoped, and historically interpretable**. The finalized DB model explicitly provides the necessary `farmId`, `deviceId`, `assignmentId`, `animalId`, `recordedAt`, metrics, alerts, raw payload, and source fields. fileciteturn26file0

---

# 161. Model Progress

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
26 DeviceAssignment
27 SensorData ← CURRENT
28 Report
```

**Next → `Report`**
