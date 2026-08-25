# Liventra — Business Logic Documentation
## Model 03: FarmMember

**Document Version:** Business Logic v1.0  
**Project:** Liventra — A Smart Livestock Farm Management System  
**Database:** MongoDB  
**Architecture:** MERN

---

# 1. Source Model

The finalized `FarmMember` model contains:

```text
_id
userId
farmId
role: Owner | Manager | Worker | Veterinarian | Accountant
nid
joiningDate
salary
isActive
assignedBy (User)
createdAt
updateAt
```

The finalized DB document explicitly defines FarmMember as the relationship between a `User` and a `Farm`, with a farm-specific enum role. fileciteturn6file1

> **Important:** This document explains the business behavior of the finalized model. It does not add a new `Role` collection. Role remains an enum in `FarmMember`, as decided for the project.

---

# 2. Purpose

`FarmMember` represents a user's membership inside a particular farm.

It answers:

> **"What is this user allowed to do in this particular farm?"**

The `User` model represents global identity.

The `FarmMember` model represents the **User ↔ Farm relationship**.

```text
User
  |
  | membership
  ↓
FarmMember
  |
  | role
  ↓
Farm
```

This is one of the most important authorization models in Liventra.

---

# 3. Why FarmMember Exists

A single user may work with multiple farms.

Example:

```text
Rahim
  |
  +---- Farm A → Owner
  |
  +---- Farm B → Manager
  |
  +---- Farm C → Worker
```

Therefore, storing farm role directly in `User` would be incorrect.

The role belongs to the **relationship between the user and the farm**.

---

# 4. Core Business Principle

The system must never assume:

```text
User = Farm Member
```

Instead:

```text
Authenticated User
        ↓
Requested Farm
        ↓
FarmMember lookup
        ↓
Membership exists?
        ↓
isActive?
        ↓
Role?
        ↓
Permission?
        ↓
Allow / Reject
```

This is the foundation of Liventra's farm-level RBAC.

---

# 5. FarmMember Lifecycle

```text
User Selected
     ↓
Invitation / Assignment
     ↓
FarmMember Created
     ↓
Active Member
     ↓
Role-Based Farm Access
     ↓
Role Change / Salary Update / Other Update
     ↓
Membership Deactivated
     ↓
Historical Membership Preserved
```

The record should normally remain even after the person stops working at the farm.

---

# 6. Field-by-Field Business Logic

## 6.1 `_id`

### Type
MongoDB `ObjectId`

### Required
Yes

### Purpose

Internal primary identifier.

### Example

```text
66f8a29c...
```

---

# 6.2 `userId`

### Type
Reference to User

### Required
Yes

### Logical Reference

```text
User.userId
```

### Example

```text
USR-2026-00015
```

### Purpose

Identifies the person who belongs to the farm.

### Business Rules

- Referenced user must exist.
- A deleted/inactive User should not be treated as an active farm operator.
- User identity comes from `User`; FarmMember must not duplicate profile information such as full name or email.

---

# 6.3 `farmId`

### Type
Reference to Farm

### Required
Yes

### Example

```text
FARM-000001
```

### Purpose

Identifies the farm to which the user belongs.

### Business Rules

- Referenced farm must exist.
- Farm membership is always scoped to one farm.
- A user may have multiple FarmMember records for different farms.

---

# 6.4 `role`

### Type
Enum

### Required
Yes

### Allowed Values

```text
Owner
Manager
Worker
Veterinarian
Accountant
```

### Purpose

Defines the member's responsibility and access level within the specific farm.

---

# 7. Role Business Logic

## 7.1 Owner

The primary decision-maker for the farm.

Typical responsibilities:

- Farm configuration
- Animal management
- Feed management
- Health management
- Inventory
- Finance
- Reports
- Member management
- Device management

The exact API permission matrix will be implemented in the RBAC layer.

### Important

`Farm.ownerId` and the Owner FarmMember record should remain consistent.

---

# 7.2 Manager

Responsible for day-to-day farm operations.

Typical responsibilities:

- Animal operations
- Feed operations
- Health records
- Production records
- Inventory
- Operational reports
- Worker coordination

Depending on the final RBAC policy, a Manager may not be allowed to:

- Transfer ownership
- Change platform/system roles
- Perform sensitive administrative actions

---

# 7.3 Worker

Primarily responsible for daily data-entry and operational activities.

Examples:

- Record feeding
- Record milk
- Record weight
- Update assigned operational data
- View permitted animal information

A Worker should not automatically have access to:

- Farm financial management
- Member administration
- Sensitive audit logs
- Ownership operations

---

# 7.4 Veterinarian

Responsible for animal health-related activities.

Typical responsibilities:

- View animal health history
- Vaccination
- Treatment
- Disease records
- Health observations
- Follow-up information

A Veterinarian should not automatically gain access to:

- Farm finance
- Payroll
- Member management

unless explicitly granted by the RBAC policy.

---

# 7.5 Accountant

Responsible for financial operations.

Typical responsibilities:

- Expenses
- Income
- Financial reports
- Relevant inventory financial records

An Accountant should not automatically receive permission to modify:

- Animal health
- Breeding
- Farm ownership
- User system roles

---

# 8. Why No Separate Role Model?

Liventra intentionally does **not** create:

```text
Role
```

as a separate collection.

Instead:

```text
FarmMember.role
```

uses an enum.

### Reason

The project's role set is small and predefined:

```text
Owner
Manager
Worker
Veterinarian
Accountant
```

There is no current requirement for:

- Dynamic role creation
- Custom permission sets
- User-created roles
- Role templates

Therefore:

```text
FarmMember.role = Enum
```

is simpler and sufficient.

If a future SaaS version needs customizable roles, a dedicated Role/Permission system can be introduced later.

---

# 9. `nid`

### Type
String

### Required
Optional

### Example

```text
19901234567890123
```

### Purpose

Stores the member's National ID information when required by the farm.

### Business Logic

- Store as String, not Number.
- Do not expose it to users who do not need it.
- It is sensitive personal information and should have restricted access.
- Duplicate handling should follow the farm's policy.

### Security

NID should not be shown in:

- Public member lists
- Normal dashboard cards
- General notifications
- Unrestricted audit views

Only authorized roles should access it.

---

# 10. `joiningDate`

### Type
Date

### Required
Yes/Optional depending on onboarding policy

### Example

```text
2026-08-17
```

### Purpose

Stores when the user joined the farm.

### Used For

- Employee history
- Membership duration
- Salary/payroll context
- Farm staffing reports

### Business Rule

It should normally not be later than the current date when recording a real historical joining event, unless the system explicitly supports future-dated employment.

---

# 11. `salary`

### Type
Number

### Required
Optional

### Example

```text
18000
```

### Unit

Bangladeshi Taka (BDT), based on the project's Bangladesh context.

### Purpose

Stores the member's current/assigned salary information.

### Business Logic

Primarily relevant to:

```text
Worker
Manager
Accountant
Other paid farm staff
```

It may be irrelevant for:

```text
Owner
External Veterinarian
```

depending on farm policy.

### Validation

- Cannot be negative.
- Currency should be consistent across the system.

---

# 12. `isActive`

### Type
Boolean

### Required
Yes

### Default
Recommended:

```text
true
```

### Purpose

Determines whether this User currently has active membership in this particular Farm.

### Important Distinction

This is **not the same as `User.isActive`.**

Example:

```text
User.isActive = true

Farm A membership = true
Farm B membership = false
```

The user can still use Farm A.

---

# 13. Membership Deactivation

When a worker leaves a farm:

```text
FarmMember.isActive = false
```

Do not delete the FarmMember record.

Example:

```text
Rahim
   ↓
Farm A
   ↓
Worker
   ↓
Leaves Farm
   ↓
isActive = false
```

Historical information remains available.

---

# 14. `assignedBy`

### Type
Reference to User

### Required
Yes/Optional depending on workflow

### Purpose

Identifies who assigned/invited/added the user to the farm.

### Example

```text
assignedBy = USR-2026-00001
```

### Business Logic

If Owner adds Rahim:

```text
Owner
  ↓
Add Member
  ↓
assignedBy = Owner.userId
```

This supports accountability.

---

# 15. `createdAt`

### Type
Date

### Required
Yes

### Generated
System

### Purpose

Stores when the membership record was created.

---

# 16. `updateAt`

### Type
Date

### Required
Yes

### Generated
System

### Purpose

Stores the latest modification time.

### Implementation Note

The field name in the finalized document is `updateAt`. Keep the database naming consistent during implementation unless the team formally decides to standardize it before coding.

---

# 17. Member Creation Workflow

There are two practical workflows.

## Workflow A — Owner Adds Existing User

```text
Owner
  ↓
Enter User Information / Select User
  ↓
Find User
  ↓
Check existing membership
  ↓
Assign role
  ↓
Set joiningDate
  ↓
Set salary if applicable
  ↓
assignedBy = Owner
  ↓
Create FarmMember
  ↓
AuditLog
  ↓
Notification to User
```

---

## Workflow B — Invitation-Based Membership

```text
Owner/Manager
      ↓
Invite User
      ↓
User accepts
      ↓
FarmMember created
      ↓
Role assigned
      ↓
Notification
      ↓
AuditLog
```

The exact invitation mechanism can be implemented later.

---

# 18. Duplicate Membership Rule

A user should not have multiple active membership records for the same farm.

Bad:

```text
User A
Farm X
Manager
Active

User A
Farm X
Worker
Active
```

Correct:

```text
User A
Farm X
Manager
Active
```

If the role changes:

```text
Manager
   ↓
Worker
```

update the existing membership rather than creating a duplicate active membership.

### Recommended Constraint

Conceptually enforce:

```text
Unique(userId + farmId)
```

for the active/current membership design.

---

# 19. Role Change Workflow

Role changes are sensitive.

Example:

```text
Worker
   ↓
Manager
```

Workflow:

```text
Authorized Owner/Admin
       ↓
Check target member
       ↓
Validate new role
       ↓
Update role
       ↓
AuditLog
       ↓
Notification
```

The member's existing history should not disappear.

---

# 20. Owner Role Special Rule

A farm should normally have exactly one primary owner represented by:

```text
Farm.ownerId
```

and a corresponding:

```text
FarmMember.role = Owner
```

### Ownership Transfer

Do not simply change the FarmMember role.

Use a controlled workflow:

```text
Current Owner
      ↓
Select New Owner
      ↓
Validate New Owner Membership
      ↓
Change Farm.ownerId
      ↓
Old Owner → Manager/appropriate role
      ↓
New Owner → Owner
      ↓
AuditLog
```

This prevents inconsistent ownership.

---

# 21. FarmMember Authorization Workflow

For every farm-scoped request:

```text
Request
  ↓
JWT / Authentication
  ↓
Authenticated User
  ↓
Requested farmId
  ↓
Find FarmMember
  ↓
Does membership exist?
  ↓
Is isActive = true?
  ↓
Get role
  ↓
Check permission
  ↓
Allow / Deny
```

### Example

Worker requests:

```text
GET /farms/FARM-000001/animals
```

Backend:

```text
User = USR-00015
Farm = FARM-000001
       ↓
FarmMember exists?
       ↓
Yes
       ↓
isActive?
       ↓
Yes
       ↓
role = Worker
       ↓
Does Worker have animal-read permission?
       ↓
Yes
       ↓
Return data
```

---

# 22. Data Isolation

FarmMember is the main authorization boundary for multi-farm data.

Suppose:

```text
User A
   ↓
Farm A → Owner

User A
   ↓
Farm B → no membership
```

User A must not be able to access:

```text
Farm B animals
Farm B inventory
Farm B finance
Farm B health records
```

just by changing:

```text
farmId
```

in the request.

---

# 23. Permission Matrix — Recommended Baseline

This is a **business-policy recommendation** for implementation. The finalized model itself only defines the role enum; exact API permissions should be encoded in the backend RBAC layer.

| Module | Owner | Manager | Worker | Veterinarian | Accountant |
|---|---:|---:|---:|---:|---:|
| Farm Settings | Full | Limited | ❌ | ❌ | ❌ |
| Members | Full | Limited | ❌ | ❌ | ❌ |
| Animals | Full | Full | Data Entry/View | Health View | View |
| Breeds | Full | Full | View | View | ❌ |
| Feed | Full | Full | Data Entry | View | Cost View |
| Health | Full | Full | Limited | Full | ❌ |
| Milk | Full | Full | Data Entry | View | View |
| Weight | Full | Full | Data Entry | View | ❌ |
| Breeding | Full | Full | Limited | Full | ❌ |
| Inventory | Full | Full | Data Entry | View | Cost View |
| Expense | Full | Full | ❌ | ❌ | Full |
| Income | Full | Full | ❌ | ❌ | Full |
| Reports | Full | Full | Limited | Health | Finance |
| Audit Logs | Full | Limited | ❌ | ❌ | Limited |
| Devices | Full | Full | View | View | ❌ |

### Important

"Full" and "Limited" are application-level permissions, not additional database fields.

---

# 24. Member Deactivation Workflow

When an employee leaves:

```text
Owner/Authorized Manager
       ↓
Select Member
       ↓
Deactivate Membership
       ↓
isActive = false
       ↓
AuditLog
       ↓
Optional notification
```

The User account itself remains active if the person has other farms.

Example:

```text
User
  |
  +── Farm A → isActive=false
  |
  +── Farm B → isActive=true
```

---

# 25. Salary Update Workflow

Salary changes should be treated as a sensitive financial operation.

```text
Authorized User
       ↓
Change Salary
       ↓
Validate amount
       ↓
Update FarmMember.salary
       ↓
AuditLog
```

Audit should capture:

```text
old salary
new salary
changed field
who changed it
when
```

Access to salary information should be restricted.

---

# 26. NID Handling

NID is sensitive.

Recommended rules:

```text
Create → Authorized roles only
Read → Restricted
Update → Restricted
Audit → Avoid exposing full NID unnecessarily
Reports → Mask unless required
```

Example masked display:

```text
********6789
```

for UI views where full NID is unnecessary.

---

# 27. Notification Rules

FarmMember changes can trigger notifications.

Examples:

### New Member

```text
"Rahim has been added to Green Valley Farm as Worker."
```

### Role Change

```text
"Your role in Green Valley Farm has been changed to Manager."
```

### Membership Deactivation

```text
"Your membership in Green Valley Farm has been deactivated."
```

Notifications belong in the `Notification` model.

---

# 28. Audit Rules

Important FarmMember events:

```text
Member Added
Member Role Changed
Member Salary Changed
Member NID Updated
Member Deactivated
Member Reactivated
Member Removed
Ownership Transferred
```

AuditLog should capture:

```text
userId
farmId
entityType
entityId
action
oldData
newData
changedFields
createdAt
```

### Sensitive Data Rule

Do not unnecessarily expose full NID or other sensitive information in general audit views.

---

# 29. CRUD Business Rules

## Create

Allowed for:

```text
Owner
Authorized Manager
System Admin
```

depending on final RBAC.

Creation must validate:

- User exists
- Farm exists
- No duplicate active membership
- Valid role
- Joining date
- Salary if applicable

---

## Read

A user should generally be able to view members of farms they are authorized to manage/view.

Sensitive fields:

```text
NID
Salary
```

should have restricted visibility.

---

## Update

Allowed fields depend on role.

Normal editable fields:

```text
role
salary
isActive
```

Sensitive operations:

```text
ownership changes
```

need a dedicated workflow.

---

## Delete

Hard delete is not recommended.

Use:

```text
isActive = false
```

to preserve membership history.

---

# 30. Validation Rules

| Field | Validation |
|---|---|
| userId | Required, valid User |
| farmId | Required, valid Farm |
| role | Required, enum |
| nid | Optional, string |
| joiningDate | Valid date |
| salary | Non-negative if provided |
| isActive | Boolean |
| assignedBy | Valid User |
| createdAt | System generated |
| updateAt | System generated |

---

# 31. Business Constraints

### Constraint 1

User must exist.

### Constraint 2

Farm must exist.

### Constraint 3

A user should not have duplicate active membership in the same farm.

### Constraint 4

Only authorized users can create or modify membership.

### Constraint 5

Owner relationship must remain synchronized with `Farm.ownerId`.

### Constraint 6

Inactive members cannot perform normal farm operations.

### Constraint 7

Historical membership should remain preserved.

### Constraint 8

Sensitive fields require restricted access.

---

# 32. Dashboard Usage

Farm Owner/Manager can see:

```text
Farm Members

Name        Role           Status
-------------------------------------
Rahim       Manager        Active
Karim       Worker         Active
Dr. Hasan   Veterinarian   Active
Nabil       Accountant     Active
```

A worker should not necessarily see salary/NID information.

---

# 33. Farm Staffing Summary

Farm dashboard can calculate:

```text
Total Members
Active Members
Workers
Managers
Veterinarians
Accountants
```

These should preferably be calculated from FarmMember rather than stored as duplicate counters in Farm.

---

# 34. API-Level Business Flow

Logical API operations include:

```text
Farm Members
├── Add Member
├── List Members
├── Get Member
├── Update Member Role
├── Update Member Salary
├── Update Member Information
├── Activate Membership
└── Deactivate Membership
```

Ownership workflow:

```text
Transfer Ownership
```

should be a dedicated business operation rather than a generic update.

---

# 35. Edge Cases

## Case 1 — Same User Added Twice

```text
User A
Farm A
```

already exists.

Second add request:

```text
Reject duplicate membership
```

---

## Case 2 — Inactive Member Tries to Access Farm

```text
isActive = false
```

↓

Reject farm operation.

---

## Case 3 — User Leaves One Farm

Only that FarmMember becomes inactive.

User account remains active.

---

## Case 4 — User Leaves All Farms

User can still have a valid Liventra account.

They simply have no active farm membership.

---

## Case 5 — Manager Tries to Remove Owner

Reject unless the ownership-transfer policy explicitly permits it.

---

## Case 6 — Owner Leaves Farm

Cannot simply deactivate the Owner membership.

First:

```text
Transfer Ownership
```

then deactivate/change the previous owner's membership if appropriate.

---

## Case 7 — User is Deleted Globally

The User record may be soft-deleted, but historical FarmMember records remain for audit/history.

Active farm access must be blocked.

---

## Case 8 — Salary = 0

Could be valid for unpaid/owner roles depending on business policy.

Therefore, do not automatically reject zero unless the farm's HR policy requires a positive salary.

---

# 36. Example Scenario — Adding a Worker

Owner:

```text
Rahim
```

adds:

```text
Karim
```

as Worker.

System:

```text
Find User Karim
      ↓
Check Farm Membership
      ↓
No existing membership
      ↓
role = Worker
      ↓
joiningDate = 2026-08-17
      ↓
salary = 15000
      ↓
assignedBy = Rahim
      ↓
isActive = true
      ↓
Create FarmMember
      ↓
AuditLog
      ↓
Notification to Karim
```

---

# 37. Example Scenario — Worker Promoted to Manager

Current:

```text
Karim
Worker
```

Owner changes role:

```text
Worker
   ↓
Manager
```

System:

```text
Authorization
     ↓
Validate target role
     ↓
Update FarmMember
     ↓
Audit old role/new role
     ↓
Notification
```

Audit example:

```text
Old:
Worker

New:
Manager

Changed Field:
role
```

---

# 38. Example Scenario — User Works in Two Farms

```text
User: Karim

Farm A
   ↓
Worker

Farm B
   ↓
Accountant
```

When Karim logs into Farm A:

```text
Permissions = Worker
```

When Karim switches to Farm B:

```text
Permissions = Accountant
```

The User account itself has not changed.

Only the active FarmMember relationship determines farm-level permissions.

---

# 39. Example Scenario — Employee Leaves

Karim leaves Farm A.

System:

```text
FarmMember.isActive = false
```

but keeps:

```text
userId
farmId
role
joiningDate
salary
createdAt
```

This allows the farm to retain historical staffing information.

---

# 40. Defense Questions

### Q: Why is role inside FarmMember instead of User?

**Answer:**

> "Because a user can belong to multiple farms and can have different roles in different farms. Therefore, role belongs to the User–Farm relationship rather than the global User identity."

---

### Q: Why don't you create a separate Role collection?

**Answer:**

> "The current system has a small, fixed set of predefined roles—Owner, Manager, Worker, Veterinarian and Accountant. Since roles are not dynamically created or customized, an enum is simpler and sufficient. A separate Role collection can be introduced if future SaaS requirements demand custom permissions."

---

### Q: What happens if a worker leaves the farm?

**Answer:**

> "We deactivate the FarmMember relationship instead of deleting it. This prevents the worker from accessing the farm while preserving historical membership and audit information."

---

### Q: Can the same user work in multiple farms?

**Answer:**

> "Yes. A User is a global identity, while FarmMember stores each farm-specific relationship. Therefore, the same user can be Owner in one farm and Worker or Accountant in another."

---

### Q: How do you prevent an inactive member from accessing the farm?

**Answer:**

> "Every farm-scoped request first resolves the user's FarmMember record and checks `isActive`. If the membership is inactive, the request is rejected before the business operation is executed."

---

# 41. Final Business Rules Checklist

```text
[✓] FarmMember represents User ↔ Farm relationship
[✓] Role is an enum
[✓] No separate Role collection is required
[✓] User can belong to multiple farms
[✓] User can have different roles in different farms
[✓] Active membership controls farm access
[✓] Duplicate active membership is prevented
[✓] Owner relationship must remain consistent with Farm.ownerId
[✓] Membership is soft-deactivated, not hard-deleted
[✓] NID is treated as sensitive information
[✓] Salary is treated as sensitive financial information
[✓] Role changes are audited
[✓] Membership changes can generate notifications
[✓] Farm-scoped authorization depends on FarmMember
[✓] Historical membership remains preserved
```

---

# 42. Final Status

**Model:** `FarmMember`  
**Status:** ✅ Finalized  
**Business Logic:** ✅ Documented  
**Database Structure:** 🔒 Frozen  
**Next Model:** `Breed`

