# Liventra — Business Logic Documentation
## Model 23: `Notification`

**Document Version:** Business Logic v1.0  
**Project:** Liventra — Smart Livestock Farm Management System  
**Database:** MongoDB  
**Architecture:** MERN  
**Module:** System / Notification Management

---

# 1. Source / Finalized Model

The latest finalized Liventra DB Model defines `Notification` as:

```text
Notification Model:
_id
notificationNo
farmId
userId
title
message
type:
    Reminder
    Alert
    Warning
    Information
    Success

priority:
    Low
    Medium
    High
    Critical

referenceType:
    Animal
    FeedRecommendation
    Vaccination
    Treatment
    Inventory
    Pregnancy
    SensorData
    Manual

referenceId
isRead
readAt
expiresAt
status:
    Active
    Archived
    Expired

createdAt
updatedAt
```

These fields and enum values are explicitly present in the latest finalized DB Models document. fileciteturn21file19

The SRS identifies the Notification module with these notification categories:

```text
Vaccine Reminder
Feed Reminder
Medicine Reminder
Low Stock
Doctor Visit
Birth
Pregnancy
```

fileciteturn21file3turn21file4

---

# 2. Purpose

`Notification` is the system's **persistent user-facing notification record**.

It answers:

> "What does the system want this particular user to know, why is it important, what business record caused it, whether the user has seen it, and whether the notification is still active?"

Example:

```text
Title:
Vaccination Due

Message:
FMD vaccination for Animal A-001 is due tomorrow.

Type:
Reminder

Priority:
High

isRead:
false
```

---

# 3. Notification's Position in Liventra

Notification sits above multiple business modules.

```text
                    Notification
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
      Animal           Health           Feed
        │                │                │
        ▼                ▼                ▼
   Pregnancy        Vaccination       FeedRecommendation
        │                │                │
        └────────────────┼────────────────┘
                         ▼
                    User Dashboard
```

It is therefore a **cross-module system component**, not a health-only or farm-only collection.

---

# 4. Notification vs AuditLog

These two must remain separate.

## Notification

Tells the user:

```text
"You need to do something / know something."
```

## AuditLog

Records:

```text
"Something happened in the system."
```

Example:

```text
Notification:
"FMD vaccination is due tomorrow."

AuditLog:
"Vaccination record created by User X."
```

The finalized AuditLog explicitly supports `Notification` as an entity type. fileciteturn21file0turn21file1

---

# 5. Notification vs Realtime Event

A realtime event and a Notification are also different.

```text
Database Notification
=
Persistent record

Socket/WebSocket event
=
Realtime delivery mechanism
```

Example:

```text
Low stock detected
       ↓
Create Notification
       ↓
Emit realtime event
       ↓
User sees notification instantly
```

If the user was offline, the persistent Notification remains available.

---

# 6. Field-by-Field Business Logic

# 6.1 `_id`

### Type

MongoDB `ObjectId`

### Required

Yes

### Purpose

Unique internal identifier.

Example:

```text
ObjectId(...)
```

Used internally by the application and by references from other records where applicable.

---

# 7. `notificationNo`

### Type

String

### Required

Yes

### Example

```text
NOT-000001
```

### Purpose

Human-readable notification identifier.

Useful for:

```text
Debugging
Audit
Support
Logs
Administrative inspection
```

Normal users generally do not need to see this number.

---

# 8. Notification Number Generation

Recommended:

```text
NOT-{SEQUENCE}
```

Examples:

```text
NOT-000001
NOT-000002
NOT-000003
```

The backend should generate it.

Do not depend on the frontend to generate a unique notification number.

---

# 9. `farmId`

### Type

ObjectId / Farm reference

### Required

Yes

### Purpose

Identifies the farm to which the notification belongs.

This is important because Liventra is multi-farm capable.

---

# 10. Farm Isolation

A notification must never leak across farms.

Example:

```text
Notification.farmId = Farm A
Notification.userId = User X
```

The user must only receive this notification if:

```text
User X
```

has valid access to:

```text
Farm A
```

The SRS explicitly requires the system to prevent users from accessing another farm's data. fileciteturn21file3

---

# 11. `userId`

### Type

ObjectId / User reference

### Required

Yes

### Purpose

Identifies **which user should receive the notification**.

This is extremely important.

The same farm can have:

```text
Owner
Manager
Worker
```

but not every notification should necessarily go to every user.

---

# 12. Why `farmId` + `userId` Both Exist

They answer two different questions:

```text
farmId
→ Which farm is this notification about?

userId
→ Which user should see it?
```

Example:

```text
farmId = Farm A
userId = Manager X
```

means:

```text
This notification belongs to Farm A
and is intended for Manager X.
```

---

# 13. Notification Audience

The current model is **user-specific** because it has:

```text
userId
```

It does not have:

```text
role
recipientType
recipientUserIds[]
```

Therefore, do not treat one Notification document as inherently broadcast to an entire farm.

If multiple users need the same notification, the application can create separate Notification documents:

```text
Notification → Owner
Notification → Manager
Notification → Worker
```

where appropriate.

---

# 14. Role-Based Notification Delivery

The SRS defines:

```text
Admin
Farm Owner
Farm Manager
Worker
```

with different access levels. fileciteturn21file3

Therefore notification generation should consider the user's role.

Example:

```text
Critical animal health alert
→ Farm Owner
→ Farm Manager

Routine feed reminder
→ Worker
→ Farm Manager
```

The exact recipient policy should be defined in the notification service.

---

# 15. `title`

### Type

String

### Required

Yes

### Example

```text
Vaccination Due
```

### Purpose

Short human-readable heading.

Good:

```text
FMD Vaccination Due
```

Bad:

```text
Notification regarding something related to vaccination
```

Keep title concise.

---

# 16. `message`

### Type

String

### Required

Yes

### Purpose

Contains the actual notification explanation.

Example:

```text
FMD vaccination for Animal A-001 is due tomorrow.
```

The message should answer:

```text
What happened?
What is due?
Which animal/item is affected?
What should the user do?
```

where applicable.

---

# 17. Notification Message Example

```text
Title:
Low Stock Alert

Message:
Napier grass stock has fallen below the configured minimum level.
```

This is more useful than:

```text
Title:
Inventory

Message:
Inventory issue.
```

---

# 18. `type`

### Type

Enum

### Required

Yes

### Finalized Values

```text
Reminder
Alert
Warning
Information
Success
```

These values are defined in the finalized model. fileciteturn21file19

---

# 19. `type = Reminder`

Use when:

```text
Something is due or approaching.
```

Examples:

```text
Vaccination due
Pregnancy check due
Follow-up due
Feed reminder
```

SRS examples include:

```text
Vaccine Reminder
Feed Reminder
Medicine Reminder
Doctor Visit
Pregnancy
```

fileciteturn21file3

---

# 20. `type = Alert`

Use when the system wants the user to pay immediate attention to a condition/event.

Example:

```text
Critical animal health event
```

or:

```text
Sensor detected abnormal temperature
```

when IoT is enabled.

---

# 21. `type = Warning`

Use when something is potentially problematic but not necessarily an immediate critical event.

Example:

```text
Inventory is approaching reorder level.
```

or:

```text
Animal health condition requires attention.
```

---

# 22. `type = Information`

Use for useful system information that does not require urgent action.

Example:

```text
Monthly farm report has been generated.
```

---

# 23. `type = Success`

Use when a meaningful system/business action completed successfully.

Example:

```text
Report generated successfully.
```

or:

```text
Feed recommendation confirmed successfully.
```

Do not create Success notifications for every normal CRUD operation.

---

# 24. `priority`

### Type

Enum

### Required

Yes

### Finalized Values

```text
Low
Medium
High
Critical
```

These values are defined in the finalized model. fileciteturn21file19

---

# 25. Type vs Priority

These are different.

### Type

Describes **what kind of message it is**.

```text
Reminder
Alert
Warning
Information
Success
```

### Priority

Describes **how important/urgent it is**.

```text
Low
Medium
High
Critical
```

Example:

```text
type = Reminder
priority = High
```

means:

```text
It is a reminder,
but the user should pay attention soon.
```

---

# 26. Example Type + Priority Matrix

```text
Vaccination due in 30 days
→ Reminder + Low

Vaccination due tomorrow
→ Reminder + High

Vaccination overdue
→ Warning + High

Critical sensor abnormality
→ Alert + Critical

Report generated
→ Success + Low
```

These are recommended business interpretations.

---

# 27. `referenceType`

### Type

Enum

### Required

Conditional

### Finalized Values

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

These exact values are in the finalized model. fileciteturn21file19

---

# 28. Why `referenceType` Exists

It answers:

> "Which business record caused this notification?"

Example:

```text
referenceType = Vaccination
```

means:

```text
This notification was generated because of a vaccination record/event.
```

---

# 29. `referenceId`

### Type

ObjectId

### Required

Conditional

### Purpose

Identifies the actual source record.

Example:

```text
referenceType = Vaccination
referenceId = Vaccination._id
```

This allows the frontend to provide:

```text
View Vaccination
```

or navigate to the relevant record.

---

# 30. Reference Integrity

The pair:

```text
referenceType
+
referenceId
```

must be validated together.

Valid:

```text
referenceType = Animal
referenceId = Animal._id
```

Invalid:

```text
referenceType = Animal
referenceId = Treatment._id
```

The backend must reject invalid combinations.

---

# 31. Reference Mapping

Recommended:

| Notification Scenario | referenceType | referenceId |
|---|---|---|
| Animal-related alert | Animal | Animal._id |
| Feed recommendation reminder | FeedRecommendation | FeedRecommendation._id |
| Vaccine due | Vaccination | Vaccination._id |
| Treatment follow-up | Treatment | Treatment._id |
| Low stock | Inventory | Inventory._id |
| Pregnancy reminder | Pregnancy | Pregnancy._id |
| IoT alert | SensorData | SensorData._id |
| General system notification | Manual | optional |

This follows the finalized reference enum. fileciteturn21file19

---

# 32. Notification Reference vs Notification Content

The message is human-readable:

```text
"FMD vaccination for Animal A-001 is due tomorrow."
```

The reference is machine-readable:

```text
referenceType = Vaccination
referenceId = ...
```

Together they provide:

```text
Human explanation
+
Structured navigation
```

---

# 33. `isRead`

### Type

Boolean

### Required

Yes

### Example

```text
false
```

### Purpose

Tracks whether the intended user has read/seen the notification.

Typical lifecycle:

```text
isRead = false
```

when created.

After user opens/marks it:

```text
isRead = true
```

---

# 34. Default `isRead`

Recommended default:

```text
false
```

because a newly created notification has not yet been read.

---

# 35. `readAt`

### Type

Date / nullable

### Required

Optional

### Purpose

Stores the exact time when the notification was marked as read.

Example:

```text
readAt = 2026-08-20T09:30:00
```

---

# 36. `isRead` + `readAt` Consistency

Recommended rules:

### Unread

```text
isRead = false
readAt = null
```

### Read

```text
isRead = true
readAt = valid Date
```

Do not normally allow:

```text
isRead = false
readAt = Date
```

because that is logically inconsistent.

---

# 37. Mark as Read Workflow

```text
User opens notification
       ↓
PATCH /notifications/:id/read
       ↓
Validate notification.userId
       ↓
isRead = true
       ↓
readAt = current time
       ↓
updatedAt = current time
```

An AuditLog may be created if the application considers read-state changes auditable.

---

# 38. Mark All as Read

The frontend may provide:

```text
Mark all as read
```

Backend logic:

```text
Update all active unread notifications
for current user/farm
```

Conceptually:

```text
isRead = true
readAt = now
```

This should be scoped to the authenticated user.

---

# 39. `expiresAt`

### Type

Date / nullable

### Required

Optional

### Purpose

Defines when a notification should no longer be considered valid/current.

Example:

```text
expiresAt = 2026-08-30
```

After that point, the notification may transition to:

```text
status = Expired
```

---

# 40. Why `expiresAt` Matters

Some notifications have a useful lifetime.

Example:

```text
Vaccination due tomorrow
```

After the vaccination date has passed, the reminder should not remain permanently active.

Similarly:

```text
Low Stock Alert
```

may no longer be useful after stock is replenished.

---

# 41. `expiresAt` Is Not the Same as `createdAt`

```text
createdAt
=
When notification was created

expiresAt
=
When notification stops being valid/current
```

Example:

```text
createdAt:
Aug 20

expiresAt:
Aug 25
```

---

# 42. `status`

### Type

Enum

### Required

Yes

### Finalized Values

```text
Active
Archived
Expired
```

These values are defined in the finalized model. fileciteturn21file19

---

# 43. `status = Active`

Meaning:

```text
Notification is currently valid/available.
```

It can appear in:

```text
Notification Center
Unread list
Dashboard
```

depending on read state and priority.

---

# 44. `status = Archived`

Meaning:

```text
Notification has been intentionally removed from the active notification view
but remains stored for history.
```

Do not hard-delete simply because the notification is no longer useful.

---

# 45. `status = Expired`

Meaning:

```text
Notification passed its useful lifetime.
```

Example:

```text
expiresAt < now
```

and:

```text
status = Expired
```

---

# 46. `status` vs `isRead`

These represent different dimensions.

Example:

```text
status = Active
isRead = true
```

means:

```text
The notification is still valid,
but the user has already read it.
```

Another:

```text
status = Active
isRead = false
```

means:

```text
Valid + unread.
```

Another:

```text
status = Expired
isRead = false
```

means:

```text
The user never read it before it expired.
```

---

# 47. Notification Lifecycle

Typical lifecycle:

```text
Created
  ↓
Active + Unread
  ↓
Active + Read
  ↓
Archived
```

or:

```text
Created
  ↓
Active
  ↓
Expired
```

depending on the notification.

---

# 48. Expiration Logic

Recommended scheduled/background logic:

```text
Find notifications where:
status = Active
AND expiresAt <= now

→ status = Expired
```

This can run periodically.

The system does not need to wait for a user to open the notification before marking it expired.

---

# 49. Notification Creation — Event Driven

Notifications should generally be generated because of meaningful business events.

Examples:

```text
Vaccination due
Feed recommendation reminder
Treatment follow-up
Low stock
Pregnancy milestone
Sensor alert
```

Avoid creating notifications for every database change.

---

# 50. Vaccine Reminder

SRS explicitly requires:

```text
Vaccine Reminder
```

fileciteturn21file3

Example:

```text
title:
Vaccination Due

message:
FMD vaccination for Animal A-001 is due tomorrow.

type:
Reminder

priority:
High

referenceType:
Vaccination

referenceId:
Vaccination._id

isRead:
false

status:
Active
```

---

# 51. Feed Reminder

SRS includes:

```text
Feed Reminder
```

fileciteturn21file3

A notification can reference:

```text
FeedRecommendation
```

Example:

```text
title:
Feed Recommendation Ready

message:
Today's recommended feeding plan for Animal A-001 is available.

type:
Reminder

priority:
Medium

referenceType:
FeedRecommendation

referenceId:
FeedRecommendation._id
```

---

# 52. Medicine Reminder

The SRS requires:

```text
Medicine Reminder
```

fileciteturn21file3

The finalized Notification `referenceType` does **not** include:

```text
Medicine
```

Therefore, under the current schema, a medicine reminder cannot directly reference a Medicine record.

This is an important schema limitation.

A treatment-related medicine reminder could potentially use:

```text
referenceType = Treatment
```

if the reminder is about a treatment event, but the current model does not provide a direct Medicine reference type.

Do not silently invent:

```text
referenceType = Medicine
```

because it is not in the finalized enum. fileciteturn21file19

---

# 53. Low Stock Alert

SRS explicitly requires:

```text
Low Stock
```

fileciteturn21file3

Notification:

```text
type = Alert
priority = High
referenceType = Inventory
referenceId = Inventory._id
```

Example:

```text
title:
Low Stock Alert

message:
Concentrate feed stock has fallen below the reorder level.
```

---

# 54. Low Stock Trigger

Inventory contains:

```text
currentStock
minimumStock
reorderLevel
status
```

fileciteturn21file15

Therefore notification logic can use those values.

Example:

```text
currentStock <= reorderLevel
```

then:

```text
Create Low Stock Notification
```

The exact trigger threshold should be consistently chosen.

---

# 55. Low Stock Duplicate Prevention

Potential problem:

```text
Every inventory query
→ creates another notification
```

Result:

```text
100 duplicate low-stock notifications
```

Do not do this.

Recommended:

```text
One active low-stock notification
per inventory item
```

while the condition remains unresolved.

When stock returns above the threshold:

```text
Archive/resolve the active notification
```

---

# 56. Treatment Follow-Up

The finalized Treatment model contains:

```text
followUpDate
status
```

fileciteturn21file15

Therefore the system can generate:

```text
Treatment follow-up reminder
```

Example:

```text
title:
Treatment Follow-up Due

message:
Follow-up for Animal A-001 is scheduled today.

type:
Reminder

priority:
High

referenceType:
Treatment

referenceId:
Treatment._id
```

---

# 57. Pregnancy Reminder

SRS includes:

```text
Pregnancy
```

as a notification category. fileciteturn21file3

The finalized Pregnancy model includes:

```text
expectedCalvingDate
pregnancyStatus
```

fileciteturn21file15

Therefore:

```text
expectedCalvingDate approaching
```

can generate:

```text
referenceType = Pregnancy
referenceId = Pregnancy._id
```

---

# 58. Birth Notification

The SRS includes:

```text
Birth
```

fileciteturn21file3

The finalized Pregnancy model contains:

```text
actualCalvingDate
calfCount
calfGender
```

fileciteturn21file15

Therefore after delivery is recorded:

```text
Pregnancy
    ↓
actualCalvingDate set
    ↓
Birth notification
    ↓
referenceType = Pregnancy
```

This is consistent with the current schema.

---

# 59. SensorData Alert

The finalized Notification model includes:

```text
referenceType = SensorData
```

fileciteturn21file19

The upcoming SensorData model includes:

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
alerts[]
```

fileciteturn21file19

Therefore future IoT alerts can create notifications such as:

```text
type = Alert
priority = Critical
referenceType = SensorData
referenceId = SensorData._id
```

---

# 60. Sensor Alert Example

```text
title:
Abnormal Body Temperature

message:
Animal A-001 has an abnormal body temperature reading.

type:
Alert

priority:
Critical

referenceType:
SensorData

referenceId:
SensorData._id
```

This is future-ready because SensorData is marked as upcoming.

---

# 61. Manual Notification

The finalized model includes:

```text
referenceType = Manual
```

This is useful for notifications that do not originate from a supported business collection.

Example:

```text
title:
Farm Inspection Scheduled

message:
Farm inspection is scheduled for tomorrow at 10:00 AM.

type:
Information

priority:
Medium

referenceType:
Manual
```

---

# 62. Notification Generation Architecture

Recommended:

```text
Business Event
      ↓
Event / Service Layer
      ↓
Notification Service
      ↓
Create Notification
      ↓
Database
      ↓
Realtime Delivery
```

Examples:

```text
Vaccination due
→ NotificationService

Inventory low
→ NotificationService

Pregnancy due
→ NotificationService
```

---

# 63. Notification Service

Recommended internal methods:

```text
notificationService.create()
notificationService.markAsRead()
notificationService.markAllAsRead()
notificationService.archive()
notificationService.expire()
notificationService.getUserNotifications()
notificationService.getUnreadCount()
notificationService.getByReference()
notificationService.cleanupExpired()
```

These are service-level responsibilities.

---

# 64. Notification Should Not Own Business Logic

Bad architecture:

```text
Notification model itself calculates
vaccination dates
feed recommendations
inventory stock
pregnancy dates
```

Better:

```text
Vaccination Service
        ↓
detect due event
        ↓
Notification Service
        ↓
create Notification
```

Notification stores the result/context.

---

# 65. Avoid Duplicate Notifications

This is one of the most important notification rules.

Suppose:

```text
Vaccination due tomorrow
```

A scheduled job runs every hour.

Without duplicate prevention:

```text
24 notifications
```

could be generated.

Instead:

```text
Find existing active notification
for same reference/event
```

before creating a new one.

---

# 66. Notification Deduplication

A practical deduplication key can conceptually be:

```text
farmId
+
userId
+
referenceType
+
referenceId
+
notification event type
```

The exact implementation can use a dedicated event identifier or carefully designed index.

The current schema does not have an explicit:

```text
eventKey
```

field, so deduplication is a service/index design concern.

---

# 67. Example — Vaccine Reminder Deduplication

First scheduler run:

```text
Vaccination V-001
→ notification created
```

Second scheduler run:

```text
Vaccination V-001
→ active notification already exists
→ do not create duplicate
```

---

# 68. Repeated Reminder Strategy

Sometimes reminders legitimately need to repeat.

Example:

```text
Vaccination due in 7 days
Vaccination due tomorrow
Vaccination overdue
```

These are different notification states/events.

The service should distinguish them rather than blindly deduplicating every notification referencing the same record.

---

# 69. Priority Escalation

Example:

```text
7 days before:
Reminder + Medium

1 day before:
Reminder + High

Overdue:
Warning + Critical
```

This is a strong practical notification strategy.

The exact escalation policy is application logic and can be adjusted per notification type.

---

# 70. Notification Query — User Inbox

Typical query:

```text
farmId = currentFarm
userId = currentUser
status = Active
```

Then sort by:

```text
priority
createdAt
```

with high-priority recent notifications first.

---

# 71. Unread Notification Query

Conceptually:

```javascript
{
    farmId,
    userId,
    status: "Active",
    isRead: false
}
```

This powers:

```text
notification badge
```

Example:

```text
🔔 5
```

---

# 72. Unread Count

The frontend should not download every notification just to calculate:

```text
Unread = 5
```

Use a count query:

```text
countDocuments({
    farmId,
    userId,
    status: Active,
    isRead: false
})
```

This is more efficient.

---

# 73. Notification Pagination

Notification history can become large.

Use:

```text
pagination
```

or:

```text
cursor-based loading
```

Example:

```text
First request:
20 notifications

Scroll:
next 20
```

---

# 74. Notification Sorting

Recommended:

```text
Critical
High
Medium
Low
```

combined with:

```text
newest first
```

However, exact UI sorting can vary.

A simple default is:

```text
createdAt DESC
```

while visually highlighting priority.

---

# 75. Notification Indexes

Recommended:

```javascript
{
    farmId: 1,
    userId: 1,
    status: 1,
    isRead: 1,
    createdAt: -1
}
```

This supports:

```text
Unread count
Notification inbox
Active notification list
```

Reference lookup:

```javascript
{
    farmId: 1,
    referenceType: 1,
    referenceId: 1
}
```

Expiration processing:

```javascript
{
    status: 1,
    expiresAt: 1
}
```

---

# 76. Why Farm ID Should Be in Notification Queries

Even though:

```text
userId
```

identifies the recipient, `farmId` is still important because the same user can potentially belong to multiple farms.

Example:

```text
User X
 ├── Farm A
 └── Farm B
```

Notification:

```text
Farm A → User X
```

must not appear in the Farm B context.

---

# 77. Notification Realtime Architecture

Recommended:

```text
Business Event
      ↓
Notification Service
      ↓
MongoDB save
      ↓
Commit successful
      ↓
Socket.IO / WebSocket
      ↓
user-specific channel
      ↓
Frontend notification badge updates
```

Important:

**Save to DB first, then emit realtime event.**

This prevents a UI event from being sent for a database operation that later fails.

---

# 78. Realtime Event Example

```text
Event:
notification.created

Payload:
{
    notificationId,
    farmId,
    userId,
    type,
    priority
}
```

The frontend can then fetch/render the actual notification.

Do not put the entire database document into every realtime event unless necessary.

---

# 79. Offline User Behavior

Suppose:

```text
Notification created at 10:00
```

but the user is offline.

Because the notification is persisted:

```text
User logs in at 14:00
        ↓
GET active notifications
        ↓
Notification appears
```

This is one of the main reasons Notification should be stored in MongoDB instead of relying only on WebSocket messages.

---

# 80. Notification Delivery vs Persistence

```text
Persistence:
MongoDB Notification

Realtime:
Socket/WebSocket

Push:
Future mobile/web push service
```

These can coexist.

---

# 81. Future Push Notification

The current schema does not contain:

```text
pushToken
deviceToken
channel
deliveryStatus
```

Therefore, do not claim that the current model fully supports:

```text
FCM push notifications
email
SMS
WhatsApp
```

It can be extended later.

---

# 82. Notification Channels

Current model effectively supports:

```text
In-app notification
```

because it stores:

```text
title
message
isRead
readAt
status
```

A future multi-channel notification system would need additional delivery metadata.

---

# 83. Notification + SRS Requirements

The SRS specifically lists:

```text
Vaccine Reminder
Feed Reminder
Medicine Reminder
Low Stock
Doctor Visit
Birth
Pregnancy
```

fileciteturn21file3

Current schema supports direct references for:

```text
Vaccination
FeedRecommendation
Inventory
Pregnancy
Treatment
```

and future:

```text
SensorData
```

But two SRS notification concepts need special handling:

### Medicine Reminder

No `Medicine` reference type exists.

### Doctor Visit

No `Doctor` reference type exists.

Therefore, under the current finalized schema, these should be implemented using the closest supported business reference (for example `Treatment`) or as `Manual`, rather than inventing a new enum value.

---

# 84. Doctor Visit Limitation

The latest finalized core model does not contain a dedicated `Doctor` collection in the current finalized collection list.

The SRS nevertheless mentions:

```text
Doctor Visit
```

as a notification requirement. fileciteturn21file3

Therefore:

```text
referenceType = Treatment
```

could be used if the doctor visit is represented through Treatment, otherwise:

```text
referenceType = Manual
```

is the schema-compatible fallback.

---

# 85. Medicine Reminder Limitation

The finalized Notification enum does not include:

```text
Medicine
```

and the current finalized schema uses Treatment's medicine information within its medicine array. fileciteturn21file15

Therefore a medicine reminder can be associated with:

```text
Treatment
```

when the reminder belongs to a treatment plan.

---

# 86. Notification + Treatment

Treatment includes:

```text
followUpDate
status
medicines[]
```

fileciteturn21file15

Therefore Treatment can act as a parent context for:

```text
Treatment Follow-up
Medicine Reminder
Doctor Visit Reminder
```

where appropriate.

---

# 87. Notification + FeedRecommendation

FeedRecommendation is a historical snapshot of a recommendation.

Therefore a notification can reference:

```text
FeedRecommendation._id
```

Example:

```text
title:
Today's Feed Recommendation

message:
A new feeding recommendation is available for Animal A-001.

type:
Reminder

referenceType:
FeedRecommendation
```

This is especially useful because the recommendation model preserves historical recommendation versions.

---

# 88. Notification + Inventory

Inventory contains:

```text
currentStock
minimumStock
reorderLevel
status
```

fileciteturn21file15

Therefore notifications can be generated when:

```text
currentStock <= reorderLevel
```

or when:

```text
status = Expired
```

depending on the business rule.

---

# 89. Expired Inventory Notification

Example:

```text
title:
Inventory Item Expired

message:
A vaccine/medicine inventory item has reached its expiry date.

type:
Warning

priority:
High

referenceType:
Inventory

referenceId:
Inventory._id
```

This is a useful extension of the Inventory module.

---

# 90. Pregnancy Notification

Possible events:

```text
Pregnancy confirmed
Expected calving approaching
Calving overdue
Delivery recorded
```

Examples:

```text
Pregnancy confirmed
→ Success + Medium

Expected calving soon
→ Reminder + High

Overdue calving
→ Warning + Critical
```

The exact threshold should be defined by business rules.

---

# 91. Notification Expiry Example

For:

```text
Vaccination due tomorrow
```

the notification might have:

```text
createdAt = Aug 20
expiresAt = Aug 23
```

After expiry:

```text
status = Expired
```

This prevents stale reminders from filling the inbox.

---

# 92. Archive vs Expire

Use:

```text
Archived
```

when the system/user intentionally removes it from active view.

Use:

```text
Expired
```

when its defined useful lifetime has passed.

Example:

```text
User manually archives:
→ Archived

expiresAt reached:
→ Expired
```

---

# 93. Read Does Not Mean Archived

A common mistake is:

```text
User reads notification
→ delete/archive immediately
```

Do not do this unless explicitly desired.

Correct:

```text
isRead = true
status = Active
```

can remain valid.

Example:

```text
Read vaccination reminder
```

may still be useful in notification history.

---

# 94. Notification History

Historical notifications can help users understand:

```text
What the system alerted me about
When it happened
Whether I read it
```

Therefore avoid hard deletion for routine cleanup.

Use:

```text
Archived
Expired
```

instead.

---

# 95. Notification Cleanup

A background job can periodically:

```text
1. Find expired active notifications
2. Set status = Expired
3. Optionally archive old expired notifications
```

Do not permanently delete recent history immediately.

If long-term storage becomes large, an archival/retention strategy can be introduced later.

---

# 96. Notification Security

A user should only be able to:

```text
Read
Mark as read
Archive
```

their own notifications, unless an administrator has an explicit administrative capability.

Authorization condition:

```text
notification.userId == authenticatedUser._id
```

plus:

```text
notification.farmId
```

must belong to the user's accessible farm context.

---

# 97. Notification Creation Security

Users should not normally be allowed to arbitrarily create system notifications for other users through a generic public endpoint.

Prefer:

```text
Business Service
      ↓
Notification Service
```

rather than:

```text
Any frontend user
      ↓
Create arbitrary notification for another user
```

This prevents notification abuse.

---

# 98. Notification API Design

Recommended endpoints:

```text
GET    /api/notifications
GET    /api/notifications/unread-count
GET    /api/notifications/:id
PATCH  /api/notifications/:id/read
PATCH  /api/notifications/read-all
PATCH  /api/notifications/:id/archive
```

System-generated creation can remain internal:

```text
notificationService.create()
```

rather than being exposed as a general-purpose user endpoint.

---

# 99. Notification Query

Typical:

```text
GET /api/notifications
```

should automatically scope:

```text
farmId = currentFarm
userId = currentUser
```

rather than trusting the frontend to supply arbitrary user IDs.

---

# 100. Query Filters

Useful filters:

```text
isRead
type
priority
status
referenceType
date range
```

Example:

```text
Unread + High priority + Active
```

---

# 101. Unread Badge

Frontend:

```text
🔔 4
```

Backend:

```text
count Active + unread notifications
```

After user marks one as read:

```text
🔔 3
```

Realtime update can make this happen instantly.

---

# 102. Notification Center UI

Recommended structure:

```text
Notifications
────────────────────

🔴 Critical
Animal A-001 abnormal temperature

🟠 High
FMD vaccination due tomorrow

🟡 Medium
Pregnancy check due

🔵 Low
Monthly report generated
```

This is derived from:

```text
type
priority
isRead
status
```

---

# 103. Notification Detail

Clicking a notification can show:

```text
Title
Message
Created At
Priority
Type
Read At
```

and, when `referenceId` is available:

```text
View Related Record
```

---

# 104. Reference Navigation

Example:

```text
Notification:
"FMD vaccination due"

referenceType:
Vaccination

referenceId:
VACC-001
```

Frontend can route:

```text
/animals/A-001/health/vaccinations/VACC-001
```

The exact URL is application-specific.

---

# 105. Notification + AuditLog

Potential AuditLog events:

```text
Notification Created
Notification Updated
Notification Archived
```

Whether `isRead` changes are audited should be decided based on audit volume.

For normal systems, every read event may create a huge number of AuditLogs.

Therefore:

```text
Business/administrative changes
→ Audit

Simple read state
→ optionally not audited
```

This is an implementation recommendation.

---

# 106. Notification Audit Volume

Suppose:

```text
10,000 notifications
```

and every read generates an AuditLog.

Then:

```text
10,000 additional logs
```

could be created.

Because the project wants detailed AuditLog capability, the system should distinguish:

```text
important state change
```

from:

```text
high-frequency UI interaction
```

to avoid unnecessary log growth.

---

# 107. Notification Realtime + Audit

For a generated notification:

```text
Business Event
      ↓
Notification created
      ↓
AuditLog for creation if required
      ↓
Realtime event
```

For marking read:

```text
Read state update
      ↓
Realtime badge update
```

Audit can be optional depending on policy.

---

# 108. Critical Notification Handling

For:

```text
priority = Critical
```

the UI should make it highly visible.

Example:

```text
Critical Alert
Animal A-001 has abnormal temperature.
```

The database does not itself enforce UI behavior.

The frontend should interpret the priority.

---

# 109. Priority Does Not Automatically Mean Push

Do not assume:

```text
Critical
→ SMS
```

or:

```text
Critical
→ Mobile push
```

The current schema does not define notification channels.

Priority is only an importance level.

---

# 110. Notification Deduplication by Condition

For low stock:

```text
Condition:
currentStock <= reorderLevel
```

Do:

```text
if active notification exists
    don't create another
else
    create notification
```

For resolved condition:

```text
currentStock > reorderLevel
```

then:

```text
archive/resolve active notification
```

This prevents notification spam.

---

# 111. Condition-Based Notification Architecture

```text
Business State
     ↓
Rule Evaluation
     ↓
Condition True?
   /       \
 No         Yes
 |           |
Stop    Existing notification?
            /       \
          Yes        No
           |          |
          Stop      Create
```

This architecture works well for:

```text
Low stock
Vaccination due
Pregnancy due
Treatment follow-up
Sensor alerts
```

---

# 112. Scheduled Notification Jobs

Some reminders depend on time.

Examples:

```text
Vaccination due date
Pregnancy expected calving date
Treatment followUpDate
```

A scheduler/cron/queue worker can periodically check them.

Example:

```text
Every hour
   ↓
Find upcoming due events
   ↓
Generate missing notifications
```

The highest-frequency automation constraints apply to ChatGPT automations, not the Liventra application's own backend scheduler; the project's server can use its own scheduling infrastructure.

---

# 113. Important: Notification Is Not Scheduler

The Notification model stores:

```text
the result
```

It does not schedule jobs.

For example:

```text
Vaccination.nextDueDate
```

is business data.

A scheduler checks it.

Then:

```text
Notification
```

is created.

---

# 114. Example — Vaccination Scheduler

```text
Vaccination.nextDueDate
        ↓
Scheduler checks date
        ↓
Due soon?
        ↓
Create Notification
        ↓
referenceType = Vaccination
        ↓
referenceId = vaccination._id
```

---

# 115. Example — Pregnancy Scheduler

```text
Pregnancy.expectedCalvingDate
        ↓
Scheduler
        ↓
Approaching?
        ↓
Notification
        ↓
referenceType = Pregnancy
        ↓
referenceId = pregnancy._id
```

---

# 116. Example — Treatment Scheduler

```text
Treatment.followUpDate
        ↓
Scheduler
        ↓
Due
        ↓
Notification
        ↓
referenceType = Treatment
```

---

# 117. Notification Failure Handling

If Notification creation fails:

```text
Business event
```

should not necessarily fail if the notification is only an auxiliary action.

Example:

```text
Vaccination record successfully created
Notification creation failed
```

The core vaccination record should remain successful.

The system can:

```text
log error
retry notification
```

depending on architecture.

For critical workflows, notification creation can be queued for reliable asynchronous delivery.

---

# 118. Queue-Based Architecture

For scalable notification processing:

```text
Business Event
      ↓
Message Queue
      ↓
Notification Worker
      ↓
Create Notification
      ↓
Realtime Event
```

Possible technologies include:

```text
BullMQ
Redis
RabbitMQ
Kafka
```

The current project does not require a specific queue technology.

---

# 119. Notification Reliability

Because notifications are reminders/alerts, missing one can be problematic.

A reliable architecture should avoid:

```text
business event
→ notification request lost
```

Therefore future production architecture can use:

```text
queue + retry
```

for important notification generation.

---

# 120. Notification + Reports

Notification can inform the user that a Report is ready:

```text
title:
Monthly Finance Report Ready

message:
Your August financial report has been generated.

type:
Success

priority:
Low
```

However, the finalized Notification reference enum does not include:

```text
Report
```

Therefore this notification should use:

```text
referenceType = Manual
```

unless the schema is later expanded.

---

# 121. Notification + Income/Expense

Similarly, the finalized Notification reference enum does not include:

```text
Income
Expense
```

So do not claim direct notification references to those models.

A finance-related notification can currently use:

```text
Manual
```

if needed.

---

# 122. Current Schema Limitation — No `Income` Reference

Even though the Finance module has:

```text
Income
```

the Notification model does not include:

```text
Income
```

in `referenceType`.

Therefore:

```text
Income received
```

cannot directly reference an Income document under the current finalized enum.

---

# 123. Current Schema Limitation — No `Expense` Reference

Likewise:

```text
Expense
```

is not a Notification reference type.

Therefore:

```text
Expense pending
```

cannot directly reference an Expense record.

If finance notifications are required, use:

```text
Manual
```

or consider a future schema enhancement.

---

# 124. Current Schema Limitation — No `Medicine` Reference

SRS requires:

```text
Medicine Reminder
```

but Notification reference types do not include:

```text
Medicine
```

fileciteturn21file3turn21file19

Therefore the current design must use:

```text
Treatment
```

or:

```text
Manual
```

depending on context.

---

# 125. Current Schema Limitation — No `Doctor` Reference

SRS requires:

```text
Doctor Visit
```

but Notification reference types do not include:

```text
Doctor
```

and the finalized core collection list does not contain a Doctor collection. fileciteturn21file0

Therefore the current schema supports this only indirectly, for example through:

```text
Treatment
```

or:

```text
Manual
```

---

# 126. Current Schema Limitation — No Recipient Group

The model has:

```text
userId
```

but not:

```text
recipientUsers[]
role
```

Therefore group notification is implemented by creating separate notification documents for each intended recipient.

---

# 127. Current Schema Limitation — No Notification Channel

No field exists for:

```text
In-App
Push
Email
SMS
```

Therefore current model should be treated primarily as:

```text
in-app notification persistence
```

with realtime transport handled separately.

---

# 128. Current Schema Limitation — No Delivery Status

There is no:

```text
sentAt
deliveredAt
failedAt
deliveryStatus
```

Therefore the model cannot provide detailed push/email delivery tracking.

---

# 129. Current Schema Limitation — No Explicit Event Key

There is no:

```text
eventKey
```

or:

```text
deduplicationKey
```

Therefore duplicate prevention must be handled through:

```text
reference fields
+
type/priority/business condition
+
service logic
```

---

# 130. Current Schema Limitation — No Action Field

The model does not contain:

```text
actionUrl
actionType
actionLabel
```

Therefore the frontend must infer navigation from:

```text
referenceType
referenceId
```

or use message-based/general routing.

---

# 131. Current Schema Limitation — No Notification Grouping

There is no:

```text
groupId
threadId
```

Therefore multiple notifications from the same event family are separate documents.

---

# 132. Recommended API Endpoints

```text
GET    /api/notifications
GET    /api/notifications/unread-count
GET    /api/notifications/:id
PATCH  /api/notifications/:id/read
PATCH  /api/notifications/read-all
PATCH  /api/notifications/:id/archive
```

System generation should preferably happen internally through services/jobs.

---

# 133. Recommended Service Methods

```text
createNotification()
createForUser()
createForFarmUsers()
markAsRead()
markAllAsRead()
archiveNotification()
expireNotification()
getUserNotifications()
getUnreadCount()
getNotificationsByReference()
findExistingActiveNotification()
cleanupExpiredNotifications()
```

---

# 134. Notification Controller Responsibility

Controller should handle:

```text
Authentication
Request validation
Calling notification service
Response formatting
```

It should not contain complicated business rules such as:

```text
vaccination due calculation
pregnancy due calculation
low-stock detection
```

Those belong in domain/service logic.

---

# 135. Notification Repository Responsibility

Repository/data-access layer can handle:

```text
findByUser
findUnread
countUnread
findActiveByReference
markRead
archive
expire
```

This keeps database logic separate from notification-generation rules.

---

# 136. Notification Event Examples

Recommended application events:

```text
vaccination.due
treatment.followup_due
inventory.low_stock
pregnancy.due
pregnancy.delivered
feed.recommendation_ready
sensor.alert
```

Then:

```text
Event
→ Notification handler
```

This is cleaner than tightly coupling every module to Notification.

---

# 137. Example Event Flow — Low Stock

```text
Inventory Transaction
      ↓
Stock changes
      ↓
Inventory Service recalculates status
      ↓
currentStock <= reorderLevel
      ↓
emit inventory.low_stock
      ↓
Notification Service
      ↓
create Notification
      ↓
emit notification.created
      ↓
Frontend updates
```

---

# 138. Example Event Flow — Vaccination

```text
Vaccination.nextDueDate
      ↓
Scheduler
      ↓
Due condition detected
      ↓
vaccination.due event
      ↓
Notification Service
      ↓
Notification created
      ↓
Realtime event
```

---

# 139. Example Event Flow — Pregnancy

```text
Pregnancy.expectedCalvingDate
      ↓
Scheduler
      ↓
Approaching date
      ↓
pregnancy.due
      ↓
Notification
      ↓
User sees reminder
```

---

# 140. Notification Priority Strategy

A practical default:

```text
Low:
Information

Medium:
Routine Reminder

High:
Warning / important Reminder

Critical:
Critical Alert
```

This is a recommendation; the finalized schema only defines the available enum values.

---

# 141. Notification Retention

Recommended concept:

```text
Active:
current notifications

Archived:
historical notifications

Expired:
time-based stale notifications
```

Do not delete notifications immediately after reading.

This preserves useful history.

---

# 142. Notification Performance

The notification inbox is read frequently.

Therefore indexes are important.

Primary access pattern:

```text
farmId
+
userId
+
status
+
isRead
+
createdAt
```

This should be optimized with a compound index.

---

# 143. Notification Data Volume

Notifications can grow quickly because:

```text
daily reminders
low stock alerts
health alerts
pregnancy alerts
IoT alerts
```

can generate many records.

Therefore:

```text
deduplication
expiration
archiving
indexing
pagination
```

are essential.

---

# 144. Avoid Notification Spam

Bad:

```text
Stock low
Stock low
Stock low
Stock low
Stock low
```

Good:

```text
One active low-stock notification
```

until the condition is resolved.

Then, if the stock becomes low again later:

```text
Create a new notification event
```

---

# 145. Notification State Machine

A useful conceptual model:

```text
                 ┌──────────────┐
                 │    Active    │
                 └──────┬───────┘
                        │
              ┌─────────┴─────────┐
              ▼                   ▼
          User reads          expiresAt reached
              │                   │
              ▼                   ▼
        Active + Read          Expired
              │
              │
         User archives
              │
              ▼
           Archived
```

Note:

```text
isRead
```

is separate from:

```text
status
```

---

# 146. Example Complete Notification

```javascript
{
    notificationNo: "NOT-000125",

    farmId: "FARM-001",

    userId: "USER-001",

    title: "FMD Vaccination Due",

    message: "FMD vaccination for Animal A-001 is due tomorrow.",

    type: "Reminder",

    priority: "High",

    referenceType: "Vaccination",

    referenceId: "VACC-000125",

    isRead: false,

    readAt: null,

    expiresAt: "2026-08-23T23:59:59Z",

    status: "Active",

    createdAt: "2026-08-22T09:00:00Z",

    updatedAt: "2026-08-22T09:00:00Z"
}
```

Values are illustrative.

---

# 147. Example Low Stock Notification

```javascript
{
    notificationNo: "NOT-000126",

    farmId: "FARM-001",

    userId: "USER-002",

    title: "Low Stock Alert",

    message: "Concentrate feed stock has fallen below the reorder level.",

    type: "Alert",

    priority: "High",

    referenceType: "Inventory",

    referenceId: "INV-00015",

    isRead: false,

    readAt: null,

    expiresAt: null,

    status: "Active"
}
```

---

# 148. Example Pregnancy Notification

```javascript
{
    notificationNo: "NOT-000127",

    farmId: "FARM-001",

    userId: "USER-001",

    title: "Expected Calving Soon",

    message: "Animal A-005 is approaching its expected calving date.",

    type: "Reminder",

    priority: "High",

    referenceType: "Pregnancy",

    referenceId: "PREG-00005",

    isRead: false,

    readAt: null,

    expiresAt: "2026-09-01T23:59:59Z",

    status: "Active"
}
```

---

# 149. Example Critical Sensor Notification

Future IoT use:

```javascript
{
    notificationNo: "NOT-000128",

    farmId: "FARM-001",

    userId: "USER-001",

    title: "Critical Temperature Alert",

    message: "Animal A-001 has an abnormal body temperature reading.",

    type: "Alert",

    priority: "Critical",

    referenceType: "SensorData",

    referenceId: "SENSOR-DATA-0001",

    isRead: false,

    readAt: null,

    expiresAt: null,

    status: "Active"
}
```

The SensorData module is currently marked as upcoming. fileciteturn21file19

---

# 150. Error Scenario — Cross-Farm Reference

```text
Notification.farmId = FARM-A

referenceId belongs to FARM-B
```

Result:

```text
Reject
```

---

# 151. Error Scenario — Wrong User

User X attempts:

```text
PATCH /notifications/NOT-001/read
```

but:

```text
notification.userId = User Y
```

Result:

```text
Reject / Forbidden
```

A user must not mark another user's notification as read.

---

# 152. Error Scenario — Invalid Reference

```text
referenceType = Vaccination
referenceId = nonexistent
```

Result:

```text
Reject
```

for a notification that claims to be linked to a specific vaccination.

---

# 153. Error Scenario — Inconsistent Read State

```text
isRead = false
readAt = some date
```

Result:

```text
Reject / normalize
```

Recommended:

```text
Unread → readAt = null
```

---

# 154. Error Scenario — Expired but Active

```text
status = Active
expiresAt < now
```

This is a stale state.

Background cleanup should transition:

```text
Active → Expired
```

---

# 155. Error Scenario — Notification Spam

If the scheduler generates:

```text
same reference
same event
same user
```

repeatedly:

```text
Find active existing notification
→ do not create duplicate
```

---

# 156. Notification + Farm Deletion

If a farm is deleted/archived, its notifications should not become accessible to users.

Depending on the farm lifecycle policy:

```text
Farm archived
    ↓
Notifications archived/hidden
```

Do not leave orphaned notifications visible in active farm contexts.

---

# 157. Notification + User Deactivation

If a user becomes inactive:

```text
User.isActive = false
```

their notifications should not be delivered to an active session.

Historical notification records can remain for audit/history depending on retention policy.

---

# 158. Notification + FarmMember

Because user membership determines farm access, notification queries should effectively respect:

```text
FarmMember
```

authorization.

Example:

```text
User
 ↓
FarmMember
 ↓
Farm access
 ↓
Notification.farmId
```

This prevents cross-farm access.

---

# 159. Notification + User Role

Role itself can remain an enum in FarmMember, consistent with the project's finalized design.

Notification service can check:

```text
FarmMember.role
```

to determine intended recipients.

No separate Notification Role model is needed.

---

# 160. Notification + Dashboard

Dashboard can show:

```text
Unread notifications
Critical alerts
Recent reminders
Low stock warnings
Upcoming pregnancy events
Vaccination reminders
```

The dashboard should query only the user's authorized farm context.

---

# 161. Notification + Mobile Future

The project future scope includes a mobile application. fileciteturn21file3

The persistent Notification model is useful for this future because:

```text
Web
+
Mobile
```

can both read the same notification history.

However, actual push delivery requires additional infrastructure not represented in the current model.

---

# 162. Notification + Multi-Language Future

The SRS future scope includes:

```text
Bangla & English
```

fileciteturn21file3

The current model stores:

```text
title
message
```

as one language's rendered text.

Therefore, the current schema does not directly support multilingual notification content.

A future multilingual design could store localized content, but that would require a schema change.

---

# 163. Notification + Report

The finalized Report model supports:

```text
Audit Log
```

as a report type. fileciteturn21file19

A report generation completion notification could therefore inform the user that a report is ready.

But because Notification has no `Report` reference type, the current implementation should use:

```text
referenceType = Manual
```

if a direct reference is needed.

---

# 164. Notification Creation Timing

For time-based reminders, create notifications at an appropriate lead time.

Example:

```text
Vaccination:
7 days before → Reminder

1 day before → High priority Reminder

Overdue → Warning
```

Do not create all future reminders months in advance unless the application has a clear reason.

This keeps the active notification list manageable.

---

# 165. Notification Expiry Strategy

Example:

```text
Reminder created:
Aug 20

Due:
Aug 27

expiresAt:
Aug 28
```

After the event becomes overdue, a new notification may replace/escalate the previous one rather than leaving multiple stale reminders active.

---

# 166. Notification Replacement / Escalation

For the same business event:

```text
Medium reminder
        ↓
High reminder
        ↓
Critical/overdue warning
```

The service can:

```text
archive/supersede old notification
+
create new notification
```

The current schema has no explicit:

```text
supersededBy
```

field, so this relationship is managed at the service level if needed.

---

# 167. Current Schema Limitation — No `supersededBy`

FeedRecommendation has a dedicated:

```text
status = superseded
```

concept, but Notification does not.

Therefore, notification escalation is represented through:

```text
status
priority
new notification
```

rather than an explicit supersession relationship.

---

# 168. Notification Reporting

The current Report model does not list:

```text
Notification
```

as a report type. fileciteturn21file19

Therefore, do not claim a dedicated Notification Report in the finalized reporting schema.

Notification data can still be inspected through the application/admin interface.

---

# 169. Notification Analytics

Even without a dedicated report type, the system can calculate operational metrics:

```text
Unread count
Notifications by type
Notifications by priority
Expired notifications
Average time to read
```

These are derived analytics, not stored fields.

---

# 170. Notification Performance Goal

The SRS specifies:

```text
95% of normal requests shall respond within 2 seconds
```

fileciteturn21file3

Therefore notification APIs should use:

```text
proper indexes
pagination
count queries
farm/user scoping
```

to keep notification requests fast.

---

# 171. Final Business Rules Checklist

```text
[✓] Notification is a persistent user-facing system record
[✓] Notification is separate from realtime transport
[✓] Notification is separate from AuditLog
[✓] farmId identifies the farm context
[✓] userId identifies the intended recipient
[✓] Cross-farm notification access must be prevented
[✓] notificationNo should be backend-generated
[✓] title is a concise heading
[✓] message contains the user-facing explanation
[✓] type uses finalized enum
[✓] priority uses finalized enum
[✓] type and priority are different concepts
[✓] referenceType uses finalized enum
[✓] referenceId points to the source business record
[✓] referenceType/referenceId must be validated together
[✓] isRead tracks read state
[✓] readAt stores read timestamp
[✓] unread notification should normally have readAt = null
[✓] expiresAt controls useful lifetime
[✓] status uses Active/Archived/Expired
[✓] read state and lifecycle status are separate
[✓] vaccine reminders are supported
[✓] feed reminders are supported
[✓] low stock alerts are supported
[✓] treatment follow-up reminders are supported
[✓] pregnancy reminders are supported
[✓] future SensorData alerts are supported
[✓] manual notifications are supported
[✓] medicine reminder has no direct Medicine reference in current schema
[✓] doctor visit has no direct Doctor reference in current schema
[✓] Income has no direct Notification reference in current schema
[✓] Expense has no direct Notification reference in current schema
[✓] Report has no direct Notification reference in current schema
[✓] duplicate notification prevention is necessary
[✓] condition-based notifications should not spam users
[✓] scheduled reminders should be generated by jobs/services
[✓] Notification model should not calculate business rules itself
[✓] save notification before realtime emission
[✓] offline users can still see persisted notifications later
[✓] pagination is required for long notification history
[✓] unread count should use count query
[✓] compound indexes are recommended
[✓] notification creation should be authorization-controlled
[✓] users should only modify their own notifications
[✓] hard deletion should generally be avoided
[✓] future push/email/SMS requires additional delivery infrastructure
[✓] current schema does not support multilingual content directly
[✓] current schema does not support notification grouping
[✓] current schema does not support explicit notification actions
[✓] current schema does not contain a deduplication event key
```

---

# 172. Final Model Summary

```text
                         Notification
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
     Recipient             Message             Business Source
     userId            title + message          referenceType
                                                referenceId
        │                     │                     │
        └─────────────────────┼─────────────────────┘
                              ▼
                         User Inbox
                              │
                    ┌─────────┴─────────┐
                    ▼                   ▼
                 Unread               Read
                    │                   │
                    └─────────┬─────────┘
                              ▼
                    Active / Archived / Expired
```

The main responsibility of `Notification` is:

> **Persist and deliver meaningful, user-specific alerts, reminders, warnings, information, and success messages generated by Liventra's business modules, while preserving their source reference, priority, read state, and lifecycle.**

---

# 173. Final Status

**Model:** `Notification`  
**Status:** ✅ Finalized  
**Business Logic:** ✅ Documented  
**User Targeting:** ✅  
**Farm Isolation:** ✅  
**Notification Types:** ✅  
**Priority System:** ✅  
**Read Tracking:** ✅  
**Expiration:** ✅  
**Lifecycle:** ✅  
**Animal Integration:** ✅  
**Feed Integration:** ✅  
**Vaccination Integration:** ✅  
**Treatment Integration:** ✅  
**Inventory Integration:** ✅  
**Pregnancy Integration:** ✅  
**Future IoT Integration:** ✅  
**Realtime Architecture:** ✅  
**Deduplication Strategy:** ✅  
**Audit Integration:** ✅  
**Performance / Indexing:** ✅  
**Current Schema Limitations:** ✅ Documented

**Next Model:** `AuditLog`
