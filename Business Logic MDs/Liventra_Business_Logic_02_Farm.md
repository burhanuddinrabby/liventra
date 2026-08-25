# Liventra — Business Logic Documentation
## Model 02: Farm

**Document Version:** Business Logic v1.0  
**Project:** Liventra — Smart Livestock Farm Management System  
**Database:** MongoDB  
**Architecture:** MERN

---

# 1. Source Model

The finalized `Farm` model contains:

```text
_id
farmName
farmCode (auto)
ownerId (userId)
address:
    village
    postOffice
    upazila
    district
    country
latitude
longitude
phone
email
establishedDate
farmType: Dairy | Beef | Breeding | Mixed
totalArea
description
logo
status: Active | Inactive | Suspended
createdAt
updatedAt
```

This document explains the business behavior around that finalized structure. It does not replace the database model.

---

# 2. Purpose

The `Farm` model represents a physical livestock farm/business managed through Liventra.

It is the **primary organizational boundary** for farm-related data.

Most operational records in Liventra are associated with a farm either directly or indirectly.

Examples:

```text
Farm
 ├── FarmMember
 ├── Animal
 ├── Inventory
 ├── FeedRecord
 ├── MilkRecord
 ├── Expense
 ├── Income
 ├── Device
 ├── Notification
 ├── AuditLog
 └── Report
```

The Farm therefore acts as the central tenant/organization entity of the system.

---

# 3. Why This Model Exists

Liventra is designed to support more than one farm and potentially more than one farm per user.

Therefore, farm data must be isolated.

Example:

```text
User
 |
 +---- Farm A
 |
 +---- Farm B
```

A user may have:

```text
Farm A → Owner
Farm B → Manager
Farm C → Worker
```

The `Farm` model identifies the farm itself, while `FarmMember` determines who can operate inside it.

---

# 4. Core Business Principle

The most important Farm-level rule is:

> **Every farm-scoped operation must be associated with the correct `farmId` and authorized through Farm membership.**

Example:

```text
Authenticated User
        ↓
Requested Farm
        ↓
FarmMember lookup
        ↓
Is user an active member?
        ↓
Check role/permission
        ↓
Allow operation
```

The frontend must never be trusted to decide whether a user belongs to a farm.

The backend must enforce this.

---

# 5. Farm Lifecycle

```text
Farm Creation
      ↓
Active
      ↓
Normal Farm Operations
      ↓
Possible Suspension / Inactivation
      ↓
Inactive / Suspended
```

A farm should normally not be physically deleted because many historical records depend on it.

---

# 6. Field-by-Field Business Logic

## 6.1 `_id`

### Type
MongoDB `ObjectId`

### Required
Yes

### Example

```text
66d8c3a91...
```

### Purpose

MongoDB's internal primary identifier.

### Used By

Many collections reference the Farm through this identifier.

---

# 6.2 `farmName`

### Type
String

### Required
Yes

### Example

```text
Green Valley Dairy Farm
```

### Purpose

Human-readable farm name shown throughout the application.

### Used In

- Dashboard
- Farm selector
- Reports
- Notifications
- Animal management
- Inventory
- Financial pages

### Validation

- Cannot be empty.
- Trim unnecessary whitespace.
- Reasonable maximum length should be enforced at API validation level.

### Business Rule

Changing the farm name does not change the `farmCode`.

---

# 6.3 `farmCode`

### Type
String

### Required
Yes

### Unique
Yes

### Generated
System

### Example

```text
FARM-000001
```

### Purpose

Unique human-readable farm identifier.

### Why It Exists

MongoDB `_id` is suitable for internal references, while `farmCode` is useful for:

- UI
- Reports
- QR/printed documents
- Support
- Audit
- Human communication

### Business Rule

Once assigned, the farm code should not normally change.

---

# 6.4 `ownerId`

### Type
String / User `userId`

### Required
Yes

### Reference

`User.userId`

### Example

```text
USR-2026-00001
```

### Purpose

Identifies the primary owner of the farm.

### Important Design Rule

This is different from FarmMember.

```text
Farm.ownerId
      ↓
Primary farm owner

FarmMember
      ↓
All members and their farm-level roles
```

### Business Logic

When a farm is created by a user:

```text
User
  ↓
Create Farm
  ↓
ownerId = creator.userId
```

A corresponding FarmMember record should be created with:

```text
role = Owner
```

This keeps ownership and membership synchronized.

---

# 6.5 `address`

### Type
Embedded Object

### Required
Yes/Partially Required depending on creation flow

### Structure

```javascript
address: {
    village,
    postOffice,
    upazila,
    district,
    country
}
```

### Example

```javascript
{
    village: "Kashipur",
    postOffice: "Mirpur",
    upazila: "Mirpur",
    district: "Kushtia",
    country: "Bangladesh"
}
```

### Purpose

Stores the physical location/address of the farm.

### Important Distinction

This is the **farm address**, not the owner's residential address.

---

# 6.6 `latitude`

### Type
Number

### Required
No

### Example

```text
23.8103
```

### Purpose

Geographic latitude of the farm.

### Used For

- Map display
- GIS integration
- Farm location visualization
- Future environmental/weather integration

---

# 6.7 `longitude`

### Type
Number

### Required
No

### Example

```text
90.4125
```

### Purpose

Geographic longitude of the farm.

### Business Logic

`latitude` and `longitude` should normally be treated as a pair.

Do not accept one without the other unless the product explicitly allows incomplete location information.

---

# 6.8 `phone`

### Type
String

### Required
Yes/No depending on finalized onboarding requirement

### Example

```text
+8801712345678
```

### Purpose

Primary contact number for the farm.

### Used For

- Farm contact
- Future SMS
- Emergency communication
- Supplier/customer communication
- Farm profile

### Business Rule

Store as String, not Number.

---

# 6.9 `email`

### Type
String

### Required
No

### Example

```text
contact@greenvalleyfarm.com
```

### Purpose

Official farm contact email.

### Validation

If provided:

- Validate email format.
- Normalize to lowercase.

---

# 6.10 `establishedDate`

### Type
Date

### Required
No

### Example

```text
2018-03-15
```

### Purpose

Stores when the farm was established.

### Used For

- Farm profile
- Business history
- Reports
- Farm age calculations

The application should calculate farm age dynamically rather than storing a separate age field.

---

# 6.11 `farmType`

### Type
Enum

### Required
Yes

### Allowed Values

```text
Dairy
Beef
Breeding
Mixed
```

### Purpose

Defines the farm's primary operational purpose.

### Business Logic

Farm type can influence:

- Dashboard emphasis
- Animal purpose
- Feed recommendations
- Production reports
- Breeding management
- Financial analytics

### Example

```text
farmType = Dairy
```

means the dashboard can prioritize:

```text
Milk Production
Lactating Animals
Feed Cost
Milk Revenue
```

A Beef farm can prioritize:

```text
Weight Gain
ADG
Feed Cost
Sale Readiness
```

---

# 6.12 `totalArea`

### Type
Number

### Required
No

### Example

```text
2.5
```

### Unit

The final model does not store a separate area unit field.

Therefore, the application should establish one consistent unit in its business/configuration layer, for example:

```text
hectare
```

or another project-approved unit.

### Purpose

Stores total farm land/operational area.

### Used For

- Farm profile
- Capacity analysis
- Future stocking-density calculations
- Feed/forage planning

---

# 6.13 `description`

### Type
String

### Required
No

### Example

```text
Mixed livestock farm specializing in dairy cattle and breeding.
```

### Purpose

Additional descriptive information about the farm.

---

# 6.14 `logo`

### Type
String

### Required
No

### Example

```text
/uploads/farms/FARM-000001/logo.png
```

### Purpose

Stores the farm logo URL/path.

### Business Logic

Used in:

- Dashboard
- Farm profile
- Reports
- PDF exports

---

# 6.15 `status`

### Type
Enum

### Required
Yes

### Allowed Values

```text
Active
Inactive
Suspended
```

### Default

```text
Active
```

---

## Active

Normal operations are allowed.

```text
Active
 ↓
Farm operations
```

---

## Inactive

Farm is not currently operating/being used.

Normal operational actions should generally be blocked or restricted.

Historical data remains available.

---

## Suspended

Farm access has been restricted by authorized system administration.

This is stronger than ordinary inactivity.

Potential reasons:

- Administrative action
- Serious policy violation
- Platform issue
- Temporary system restriction

The exact administrative policy can be defined later.

---

# 6.16 `createdAt`

### Type
Date

### Required
Yes

### Generated
System

### Purpose

Farm creation timestamp.

---

# 6.17 `updatedAt`

### Type
Date

### Required
Yes

### Generated
System

### Purpose

Latest Farm document update timestamp.

---

# 7. Farm Creation Workflow

When a user creates a farm:

```text
Authenticated User
        ↓
Submit Farm Information
        ↓
Validate Data
        ↓
Generate farmCode
        ↓
Create Farm
        ↓
Create FarmMember
        ↓
Role = Owner
        ↓
AuditLog
        ↓
Farm Dashboard Available
```

This should ideally be handled as one consistent onboarding transaction/workflow.

---

# 8. FarmMember Synchronization

This is a critical rule.

When:

```text
Farm.ownerId = USR-2026-00001
```

the system should have a corresponding:

```text
FarmMember
{
    userId: USR-2026-00001,
    farmId: FARM-000001,
    role: Owner
}
```

### Why?

Because the owner also needs farm-level authorization.

---

# 9. Farm Access Workflow

For any farm-specific API:

```text
Request
   ↓
Authenticate User
   ↓
Get farmId
   ↓
Find Farm
   ↓
Check Farm.status
   ↓
Find FarmMember
   ↓
Check membership
   ↓
Check role
   ↓
Perform operation
```

---

# 10. Farm Status Business Logic

## If Active

```text
Create Animal       → Allowed
Create Feed Record  → Allowed
Record Milk         → Allowed
Create Treatment    → Allowed
Inventory Operation → Allowed
Financial Operation → Allowed
```

---

## If Inactive

The system should restrict new operational transactions according to the application's policy.

Historical records remain readable to authorized users.

---

## If Suspended

Normal farm operations should be blocked.

Authorized administrators can still inspect the farm and resolve the suspension.

---

# 11. Farm Update Workflow

Example:

```text
Owner edits farm phone
        ↓
Authenticate
        ↓
Check FarmMember role
        ↓
Validate new phone
        ↓
Update Farm
        ↓
AuditLog
```

Not every FarmMember should be allowed to change every field.

---

# 12. Ownership Change

Changing `ownerId` is a **sensitive operation**.

It should never be treated like changing:

```text
farmName
description
phone
```

Instead:

```text
Current Owner/Admin
        ↓
Ownership Transfer Request
        ↓
Validate New Owner
        ↓
Update Farm.ownerId
        ↓
Update corresponding FarmMember roles
        ↓
AuditLog
```

Example:

```text
Old Owner
    ↓
Owner → Manager/Member

New Owner
    ↓
Member → Owner
```

This keeps the ownership relationship consistent.

---

# 13. Farm Deactivation

A farm should normally be deactivated rather than deleted.

```text
Active
  ↓
Deactivate
  ↓
Inactive
```

Historical records remain:

```text
Animals
Treatments
Milk Records
Expenses
Income
Audit Logs
Reports
```

This is important for financial and research history.

---

# 14. Farm Suspension

Suspension should be treated as an administrative state.

```text
Active
  ↓
Suspended
  ↓
Operations Restricted
  ↓
Admin Resolution
  ↓
Active / Inactive
```

A suspension action should generate an AuditLog.

---

# 15. Farm-Level Data Isolation

This is one of the most important backend rules in Liventra.

Suppose:

```text
Farm A = FARM-000001
Farm B = FARM-000002
```

A user who is only a member of Farm A must not be able to access:

```text
Animal belonging to Farm B
Inventory of Farm B
Milk records of Farm B
Expenses of Farm B
Reports of Farm B
```

even if they manually change the URL/API request.

Every service must enforce farm scope.

---

# 16. Relationships

## Farm → FarmMember

```text
Farm (1)
   ↓
FarmMember (Many)
```

A farm can have many members.

---

## Farm → Animal

```text
Farm (1)
   ↓
Animal (Many)
```

Every animal belongs to a farm.

---

## Farm → Inventory

```text
Farm (1)
   ↓
Inventory (Many)
```

---

## Farm → Finance

```text
Farm
 ├── Expense
 └── Income
```

---

## Farm → Device

```text
Farm (1)
   ↓
Device (Many)
```

---

## Farm → AuditLog

```text
Farm (1)
   ↓
AuditLog (Many)
```

---

# 17. CRUD Business Rules

## Create

### Who?

Normally:

```text
Authenticated User
```

### Requirements

- Valid user
- Valid farm information
- Unique farmCode
- Valid address/contact information
- Creator becomes owner
- FarmMember Owner record created

---

## Read

A user may read a farm only if:

```text
User is an authorized FarmMember
```

System admins may have broader platform-level access.

---

## Update

Allowed according to FarmMember role.

Typical policy:

```text
Owner    → Full farm configuration
Manager  → Operational configuration
Worker   → Limited/no farm configuration
```

Exact permission mapping belongs to the RBAC layer.

---

## Delete

Hard deletion is not recommended.

Use:

```text
status = Inactive
```

or administrative suspension.

---

# 18. Validation Rules

| Field | Validation |
|---|---|
| farmName | Required, non-empty |
| farmCode | Unique |
| ownerId | Valid User |
| address | Valid structured object |
| latitude | Valid geographic range if provided |
| longitude | Valid geographic range if provided |
| phone | Valid normalized phone format |
| email | Valid email format if provided |
| farmType | Enum |
| totalArea | Positive if provided |
| status | Enum |

---

# 19. Business Constraints

### Constraint 1

`farmCode` cannot be duplicated.

### Constraint 2

`ownerId` must refer to an existing User.

### Constraint 3

The owner must have a corresponding active FarmMember relationship.

### Constraint 4

A suspended/inactive farm should not accept normal operational transactions.

### Constraint 5

Farm historical records must remain available after deactivation.

### Constraint 6

Farm-scoped queries must always respect farm authorization.

---

# 20. Notification Rules

Farm-level events may trigger notifications.

Examples:

```text
Farm invitation
Farm member added
Farm member removed
Farm suspended
Farm reactivated
Farm ownership transferred
Critical farm-level alert
```

The actual notification record belongs to the `Notification` model.

---

# 21. Audit Rules

Important Farm events:

```text
Farm Created
Farm Updated
Farm Deactivated
Farm Reactivated
Farm Suspended
Farm Ownership Changed
Farm Contact Changed
Farm Type Changed
Farm Location Changed
```

For an update:

```text
oldData
    ↓
newData
    ↓
changedFields
```

should be captured by AuditLog where applicable.

---

# 22. Dashboard Usage

The Farm model is the root of the Farm Dashboard.

Example:

```text
Green Valley Dairy Farm
FARM-000001
Status: Active
Type: Dairy
Location: Kushtia

Animals: 125
Lactating: 67
Milk Today: 842 L
Feed Cost Today: ৳18,500
Low Stock Items: 4
Vaccinations Due: 7
```

Most of these dashboard values are calculated from other collections, not stored directly in Farm.

---

# 23. Important Non-Duplication Rule

Do not store calculated values inside Farm unless there is a strong performance reason.

For example, avoid storing:

```text
totalAnimals
totalMilk
totalExpense
totalIncome
```

as ordinary Farm fields if they can be calculated from source collections.

Instead:

```text
Farm
  +
Animal
  +
MilkRecord
  +
Expense
  +
Income
       ↓
Dashboard Aggregation
```

This prevents stale values.

---

# 24. Reporting Usage

Farm is the primary filter for most reports:

```text
Farm
 ↓
Date Range
 ↓
Report Type
 ↓
Aggregation
 ↓
PDF/Excel
```

Examples:

- Farm Summary
- Milk Report
- Animal Report
- Feed Cost Report
- Inventory Report
- Finance Report
- Health Report

---

# 25. API-Level Business Flow

The exact API list will be documented separately, but logically the Farm module needs:

```text
Farm
├── Create Farm
├── Get Farm
├── Update Farm
├── Get My Farms
├── Switch Active Farm
├── Deactivate Farm
└── Reactivate Farm
```

Administrative workflows:

```text
Suspend Farm
Reactivate Farm
Transfer Ownership
```

These require stronger authorization.

---

# 26. Edge Cases

## Case 1 — User creates a farm but FarmMember creation fails

Do not leave an ownerless operational farm.

Use a transaction or compensating workflow.

---

## Case 2 — Owner account becomes inactive

The farm should not silently lose ownership.

The system needs a defined administrative workflow for ownership reassignment.

---

## Case 3 — Farm has no active members

This should be detectable and treated as an administrative issue.

---

## Case 4 — Farm is suspended while operations are being performed

Backend must check current farm status before accepting sensitive/new transactions.

---

## Case 5 — Farm is inactive but user requests historical report

Authorized users should generally still be able to view historical data.

---

## Case 6 — Farm ownership transfer

Both:

```text
Farm.ownerId
```

and:

```text
FarmMember.role
```

must remain consistent.

---

# 27. Example Scenario — Creating a Dairy Farm

Rahim is authenticated.

He submits:

```text
farmName:
Green Valley Dairy Farm

farmType:
Dairy

district:
Kushtia

upazila:
Mirpur

totalArea:
2.5
```

System:

```text
Validate User
      ↓
Validate Farm Data
      ↓
Generate FARM-000001
      ↓
Create Farm
      ↓
ownerId = Rahim.userId
      ↓
Create FarmMember
      ↓
role = Owner
      ↓
AuditLog
      ↓
Dashboard
```

---

# 28. Example Scenario — User Has Two Farms

```text
Rahim
systemRole = user

        |
        +------------------------+
        |                        |
        ▼                        ▼
Green Valley Farm          Rahim Dairy Farm
FARM-000001                FARM-000002

FarmMember                 FarmMember
Owner                      Manager
```

When Rahim requests:

```text
GET /farms/FARM-000002/animals
```

the backend checks the second FarmMember record before returning data.

---

# 29. Defense Questions

### Q: Why is Farm a separate model?

**Answer:**

> Farm is the primary organizational boundary of Liventra. Operational data such as animals, inventory, finance, health and production records belongs to a specific farm, so Farm provides the foundation for multi-farm data isolation and management.

---

### Q: Why is ownerId in Farm if FarmMember already contains the owner?

**Answer:**

> `ownerId` directly identifies the primary owner of the farm, while `FarmMember` represents the complete membership relationship and farm-level roles. Both are useful, but the system must keep them consistent.

---

### Q: Why don't you delete a farm?

**Answer:**

> Because the farm has historical animals, financial transactions, health records, audit logs and reports. Hard deletion could destroy historical integrity, so the system uses lifecycle states such as Active, Inactive and Suspended.

---

### Q: How do you prevent one farm's data from being accessed by another farm's user?

**Answer:**

> Every farm-scoped request is authenticated and then authorized through FarmMember. After verifying membership and role, the backend scopes the database query to the requested farm. The frontend's farmId is never trusted by itself.

---

# 30. Final Business Rules Checklist

```text
[✓] Farm has unique farmCode
[✓] Farm has a primary owner
[✓] Owner is also represented through FarmMember
[✓] Farm supports multiple members
[✓] Farm supports multiple users
[✓] Farm supports multiple farms per user
[✓] Farm-level access is membership-based
[✓] Farm status controls operational availability
[✓] Historical records remain after deactivation
[✓] Ownership transfer requires controlled workflow
[✓] Farm-scoped APIs enforce authorization
[✓] Farm changes generate AuditLog
[✓] Farm-level events can generate Notifications
[✓] Calculated dashboard metrics come from source collections
```

---

# 31. Final Status

**Model:** `Farm`  
**Status:** ✅ Finalized  
**Business Logic:** ✅ Documented  
**Database Structure:** 🔒 Frozen  
**Next Model:** `FarmMember`
