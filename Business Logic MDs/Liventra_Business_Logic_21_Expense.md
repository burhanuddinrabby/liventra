# Liventra — Business Logic Documentation
## Model 21: `Expense`

**Document Version:** Business Logic v1.0  
**Project:** Liventra — Smart Livestock Farm Management System  
**Database:** MongoDB  
**Architecture:** MERN  
**Module:** Finance & Expense Management

---

# 1. Source / Finalized Model

The latest finalized Liventra DB Model defines `Expense` as:

```text
Expense Model:
_id
expenseNo
farmId
expenseCategory:
    Feed Purchase
    Medicine Purchase
    Vaccine Purchase
    Salary
    Veterinary Service
    Electricity
    Water
    Transportation
    Equipment
    Repair
    Maintenance
    Miscellaneous

amount
expenseDate
paymentMethod:
    Cash
    Bank
    Mobile Banking
    Cheque
    Card

referenceType:
    InventoryTransaction
    Treatment
    Manual
    Salary
    Other

referenceId
paidTo
invoiceNumber
description
attachment
status:
    Pending
    Paid
    Cancelled

createdBy
updatedBy
createdAt
updatedAt
```

These fields and enum values come from the latest finalized `Liventra - DB Models.docx`. fileciteturn20file5turn20file6

The SRS defines Finance as including:

```text
Expense
Income
Profit / Loss
Monthly Report
```

and Inventory as including purchase history, while reports include financial and inventory reports. fileciteturn20file7

---

# 2. Purpose

The `Expense` model records **money spent by a farm**.

It answers:

> "How much money did the farm spend, when was it spent, what was it spent on, how was it paid, who received it, what business event caused it, and what is its payment status?"

Example:

```text
Expense No:
EXP-000125

Category:
Feed Purchase

Amount:
৳4,000

Date:
2026-08-20

Payment:
Cash

Paid To:
ABC Agro Supplier

Status:
Paid
```

---

# 3. Expense's Position in Liventra

Expense belongs to the **Finance Module**.

The overall finance relationship is:

```text
                Finance
                   │
          ┌────────┴────────┐
          ▼                 ▼
       Expense            Income
          │                 │
          └────────┬────────┘
                   ▼
              Profit / Loss
```

The SRS explicitly requires Expense, Income, Profit/Loss, and Monthly Report functionality. fileciteturn20file7

---

# 4. Expense vs Inventory_Transaction

This distinction is extremely important.

## Inventory_Transaction

Represents:

```text
STOCK EVENT
```

Example:

```text
100 kg feed purchased
```

## Expense

Represents:

```text
FINANCIAL EVENT
```

Example:

```text
৳4,000 spent on that purchase
```

Therefore:

```text
Purchase
   ├── Inventory_Transaction
   │      → stock +100 kg
   │
   └── Expense
          → money spent ৳4,000
```

They describe the same real-world purchase from two different business perspectives.

---

# 5. Why Both Are Necessary

Suppose:

```text
100 kg feed purchased
৳4,000
```

Inventory needs to know:

```text
Stock increased by 100 kg
```

Finance needs to know:

```text
Farm spent ৳4,000
```

Therefore:

```text
Inventory_Transaction
+
Expense
```

should be linked rather than replacing one another.

---

# 6. Most Important Business Rule

## One real purchase should not become two financial expenses.

Correct:

```text
Purchase
   ↓
Inventory_Transaction
   ↓
Expense linked by reference
```

The Inventory_Transaction is the operational stock event.

The Expense is the financial record.

When calculating total expenses, count the Expense record—not both records separately.

---

# 7. Field-by-Field Business Logic

# 7.1 `_id`

### Type

MongoDB `ObjectId`

### Required

Yes

### Purpose

Unique internal database identifier.

Example:

```text
ObjectId(...)
```

Used internally by relationships such as:

```text
Inventory_Transaction.referenceId
Expense.referenceId
```

depending on the reference type.

---

# 8. `expenseNo`

### Type

String

### Required

Yes

### Example

```text
EXP-000001
```

### Purpose

Human-readable unique expense identifier.

Useful for:

```text
Expense search
Reports
Invoices
Audit
Financial history
```

---

# 9. Expense Number Generation

Recommended:

```text
EXP-{SEQUENCE}
```

Examples:

```text
EXP-000001
EXP-000002
EXP-000003
```

The backend should generate this value.

The frontend should not be trusted to generate unique expense numbers.

---

# 10. `farmId`

### Type

ObjectId / Farm reference

### Required

Yes

### Purpose

Identifies which farm incurred the expense.

This is essential because Liventra supports multiple farms.

---

# 11. Farm Isolation

Every Expense query must be scoped to the authorized farm.

Conceptually:

```javascript
Expense.find({
    farmId: authorizedFarmId
});
```

A user belonging to Farm A must not be able to access Farm B's expenses.

The SRS explicitly requires prevention of cross-farm data access. fileciteturn20file0

---

# 12. `expenseCategory`

### Type

Enum

### Required

Yes

### Finalized Values

```text
Feed Purchase
Medicine Purchase
Vaccine Purchase
Salary
Veterinary Service
Electricity
Water
Transportation
Equipment
Repair
Maintenance
Miscellaneous
```

These are the finalized categories in the DB model. fileciteturn20file5

---

# 13. Why Expense Category Matters

Category allows Liventra to answer:

```text
Where is the farm spending the most money?
```

Example:

```text
Feed Purchase        ৳50,000
Salary               ৳30,000
Electricity          ৳8,000
Veterinary Service   ৳6,000
Transportation       ৳5,000
```

This supports:

```text
Cost analysis
Budgeting
Profit/Loss
Dashboard charts
Monthly reports
```

---

# 14. Category → Operational Module Mapping

Recommended relationship:

```text
Feed Purchase
    ↓
Inventory / Feed

Medicine Purchase
    ↓
Inventory / Medicine

Vaccine Purchase
    ↓
Inventory / Vaccine

Salary
    ↓
Employee / Farm workforce

Veterinary Service
    ↓
Health / Treatment

Electricity
    ↓
Farm operations

Water
    ↓
Farm operations

Transportation
    ↓
Farm operations

Equipment
    ↓
Farm assets

Repair
    ↓
Farm maintenance

Maintenance
    ↓
Farm maintenance

Miscellaneous
    ↓
Other legitimate farm expense
```

This mapping is business logic; it does not add new fields to the finalized model.

---

# 15. `amount`

### Type

Number

### Required

Yes

### Example

```text
4000
```

Meaning:

```text
৳4,000
```

### Purpose

Stores the monetary amount of the expense.

---

# 16. Amount Must Be Positive

Recommended rule:

```text
amount > 0
```

An Expense represents money spent.

Therefore:

```text
amount = 0
```

or:

```text
amount < 0
```

should normally be rejected.

Corrections should be handled through an appropriate business workflow rather than negative expenses.

---

# 17. Currency

The project context is Bangladesh, so financial UI/reporting can represent amounts in:

```text
৳ / BDT
```

The DB model itself does not include a `currency` field, so do not treat currency as a stored per-record attribute.

---

# 18. `expenseDate`

### Type

Date

### Required

Yes

### Purpose

Stores the date when the expense actually occurred.

Example:

```text
2026-08-20
```

---

# 19. `expenseDate` vs `createdAt`

These have different meanings.

```text
expenseDate
=
When money was actually spent

createdAt
=
When the Expense document was created
```

Example:

```text
Actual purchase:
Aug 18

Entered into Liventra:
Aug 20
```

Then:

```text
expenseDate = Aug 18
createdAt = Aug 20
```

Financial reports should normally use:

```text
expenseDate
```

rather than:

```text
createdAt
```

---

# 20. Historical Expense Entry

Authorized users may need to enter an old expense.

Example:

```text
Expense happened:
Aug 10

Entered:
Aug 17
```

This is why keeping:

```text
expenseDate
```

separate from:

```text
createdAt
```

is useful.

Such backdated entry should be auditable.

---

# 21. `paymentMethod`

### Type

Enum

### Required

Yes / according to payment workflow

### Allowed Values

```text
Cash
Bank
Mobile Banking
Cheque
Card
```

These are the finalized values. fileciteturn20file5

---

# 22. Payment Method Purpose

Records how the expense was paid.

Examples:

```text
Cash
```

```text
Bank
```

```text
Mobile Banking
```

```text
Cheque
```

```text
Card
```

This supports financial analysis and payment history.

---

# 23. Payment Method Example

```text
Expense:
৳5,000

paymentMethod:
Mobile Banking

status:
Paid
```

This tells the system:

```text
৳5,000 has been paid using mobile banking.
```

---

# 24. `referenceType`

### Type

Enum

### Required

Conditional

### Allowed Values

```text
InventoryTransaction
Treatment
Manual
Salary
Other
```

These are the finalized values. fileciteturn20file5

---

# 25. Why `referenceType` Exists

It answers:

> "What business event caused this expense?"

Examples:

```text
InventoryTransaction
→ feed/medicine/vaccine purchase

Treatment
→ veterinary/medical expense

Salary
→ employee salary

Manual
→ manually entered general expense

Other
→ another supported business source
```

---

# 26. `referenceId`

### Type

ObjectId

### Required

Conditional

### Purpose

Links the expense to the source business record.

Example:

```text
referenceType = InventoryTransaction
referenceId = ITX-00015
```

This means:

```text
This expense came from Inventory Transaction ITX-00015.
```

---

# 27. Reference Integrity

The pair:

```text
referenceType
+
referenceId
```

must be validated together.

Examples:

```text
referenceType = Treatment
referenceId = Treatment._id
```

Valid.

But:

```text
referenceType = Treatment
referenceId = Vaccination._id
```

Invalid.

---

# 28. Reference Type → Reference ID Mapping

Recommended:

```text
InventoryTransaction
    ↓
Inventory_Transaction._id

Treatment
    ↓
Treatment._id

Salary
    ↓
Salary-related record

Manual
    ↓
No referenceId required

Other
    ↓
Depends on the specific supported workflow
```

The current finalized schema does not contain a dedicated Salary model in the listed core collections, so Salary references should only be implemented if the corresponding employee/salary record exists in the application.

---

# 29. `paidTo`

### Type

String

### Required

Optional / conditional

### Example

```text
ABC Agro Supplier
```

Other examples:

```text
Rahim Electric
Dr. Karim
Farm Worker
```

### Purpose

Stores who received the payment.

---

# 30. Why `paidTo` Matters

It provides human-readable payment context.

Example:

```text
expenseCategory = Veterinary Service
paidTo = Dr. Karim
amount = ৳2,000
```

The system can then show:

```text
Paid to Dr. Karim
```

without requiring the user to inspect another collection.

---

# 31. `invoiceNumber`

### Type

String

### Required

Optional

### Example

```text
INV-2026-4582
```

### Purpose

Stores supplier/vendor invoice reference.

Useful for:

```text
Purchase verification
Financial records
Audit
Invoice search
```

---

# 32. Invoice Number Uniqueness

Do not necessarily make:

```text
invoiceNumber
```

globally unique.

Different suppliers may generate the same invoice number.

If the application later needs uniqueness, it should be considered within an appropriate business scope such as:

```text
farm + supplier + invoice
```

The current schema does not include a supplier ID, so do not impose an unsupported uniqueness rule blindly.

---

# 33. `description`

### Type

String

### Required

Optional

### Purpose

Provides additional explanation of the expense.

Examples:

```text
Purchased 100 kg Napier grass.
```

```text
Electricity bill for August.
```

```text
Veterinary visit for animal A-001.
```

```text
Repair of milking machine.
```

---

# 34. `attachment`

### Type

String / file URL/path

### Required

Optional

### Purpose

Stores a reference to supporting documentation.

Examples:

```text
Invoice image
PDF receipt
Payment proof
Bill copy
```

The model stores the attachment reference rather than the raw file content.

---

# 35. Attachment Business Logic

Recommended:

```text
Upload file
    ↓
Store in file/object storage
    ↓
Save resulting URL/path
    ↓
Expense.attachment
```

Do not store large binary files directly inside the Expense document unless there is a deliberate architecture for it.

---

# 36. `status`

### Type

Enum

### Required

Yes

### Allowed Values

```text
Pending
Paid
Cancelled
```

These are the finalized values. fileciteturn20file5

---

# 37. Status: `Pending`

Meaning:

```text
Expense has been recorded but payment has not yet been completed.
```

Example:

```text
Supplier bill:
৳10,000

status:
Pending
```

This is especially useful for:

```text
Credit purchases
Unpaid invoices
Outstanding bills
```

---

# 38. Status: `Paid`

Meaning:

```text
The expense has been financially settled/paid.
```

Example:

```text
amount = ৳4,000
paymentMethod = Cash
status = Paid
```

---

# 39. Status: `Cancelled`

Meaning:

```text
The expense record is no longer considered a valid active expense.
```

Important:

A cancelled expense should normally **not be included in active expense totals**.

---

# 40. Cancelled Expense Rule

Suppose:

```text
Expense:
৳5,000
status = Cancelled
```

Then:

```text
Total Expenses
```

should normally exclude it.

Otherwise:

```text
Profit/Loss
```

will be incorrect.

---

# 41. Pending Expense and Profit/Loss

This is an important business decision.

The finalized model supports:

```text
Pending
Paid
Cancelled
```

but does not explicitly define accounting policy for whether Pending expenses count in profit/loss.

Therefore, the application should define a consistent finance policy.

A practical operational policy is:

```text
Pending
→ outstanding expense

Paid
→ settled expense

Cancelled
→ excluded
```

Reports should clearly distinguish:

```text
Total Recorded Expenses
vs
Total Paid Expenses
vs
Outstanding Expenses
```

rather than silently mixing them.

---

# 42. `createdBy`

### Type

ObjectId / User reference

### Required

Yes

### Purpose

Identifies who created the expense record.

---

# 43. `updatedBy`

### Type

ObjectId / User reference

### Required

Conditional / automatic on update

### Purpose

Identifies who last modified the expense.

Important for:

```text
Accountability
Audit
Finance security
```

---

# 44. `createdAt`

### Type

Date

### Required

Automatic

### Purpose

Stores when the expense document was created.

---

# 45. `updatedAt`

### Type

Date

### Required

Automatic

### Purpose

Stores when the expense document was last updated.

---

# 46. Expense Creation Workflow

Recommended:

```text
User chooses expense category
        ↓
Enter amount
        ↓
Enter expense date
        ↓
Select payment method
        ↓
Select reference if applicable
        ↓
Enter paidTo
        ↓
Enter invoice number
        ↓
Upload attachment if needed
        ↓
Set status
        ↓
Create Expense
        ↓
AuditLog
```

---

# 47. Purchase Expense Workflow

For a feed purchase:

```text
Purchase
   ↓
Inventory_Transaction
   ↓
Stock increases
   ↓
Expense created
   ↓
referenceType = InventoryTransaction
   ↓
referenceId = transaction._id
```

This is the preferred relationship between Inventory and Finance.

---

# 48. Feed Purchase Example

Farm buys:

```text
100 kg feed
৳40/kg
```

Inventory:

```text
+100 kg
```

Inventory transaction:

```text
transactionType = Purchase
quantity = 100
unitPrice = 40
totalAmount = 4000
```

Expense:

```text
expenseCategory = Feed Purchase
amount = 4000
referenceType = InventoryTransaction
referenceId = ITX-0001
```

---

# 49. Medicine Purchase Example

```text
Medicine:
৳6,000

Inventory:
+stock

Expense:
expenseCategory = Medicine Purchase
amount = 6000
referenceType = InventoryTransaction
```

---

# 50. Vaccine Purchase Example

```text
Vaccine purchase:
৳8,500

Inventory:
+stock

Expense:
expenseCategory = Vaccine Purchase
amount = 8500
referenceType = InventoryTransaction
```

---

# 51. Veterinary Service Expense

Suppose a veterinarian visits the farm.

```text
expenseCategory = Veterinary Service
amount = 2000
referenceType = Treatment
referenceId = Treatment._id
paidTo = Doctor/Service Provider
```

This creates a link between:

```text
Health
    ↓
Treatment
    ↓
Finance
    ↓
Expense
```

---

# 52. Treatment Cost vs Expense

The finalized Treatment model contains:

```text
estimatedCost
```

while Expense contains:

```text
amount
```

These should not automatically be treated as the same thing.

### Treatment.estimatedCost

Means:

```text
Estimated treatment cost
```

### Expense.amount

Means:

```text
Actual financial expense recorded
```

Therefore:

```text
Estimated:
৳2,500

Actual:
৳2,200
```

can legitimately occur.

---

# 53. Important Rule — Do Not Double Count Treatment Cost

Do not calculate:

```text
Treatment.estimatedCost
+
Expense.amount
```

as total expense.

For actual financial reporting, the Expense record should represent the recorded financial event.

Treatment cost is operational/estimated information.

---

# 54. Salary Expense

The finalized Expense enum includes:

```text
Salary
```

Therefore:

```text
expenseCategory = Salary
```

can record salary payments.

The finalized DB collection list does not show a separate Salary collection, although the SRS mentions worker information, attendance, salary, and tasks. fileciteturn20file7

Therefore, Salary Expense should not reference a nonexistent collection.

If salary management is implemented later, the reference can point to that salary record using:

```text
referenceType = Salary
```

---

# 55. Electricity Expense

Example:

```text
expenseCategory = Electricity
amount = 7500
paymentMethod = Bank
paidTo = Electricity Provider
invoiceNumber = ELEC-2026-08
status = Paid
```

---

# 56. Water Expense

Example:

```text
expenseCategory = Water
amount = 2500
paymentMethod = Cash
status = Paid
```

---

# 57. Transportation Expense

Example:

```text
expenseCategory = Transportation
amount = 1500
description = "Transport for feed delivery"
status = Paid
```

---

# 58. Equipment Expense

Example:

```text
expenseCategory = Equipment
amount = 15000
paidTo = Farm Equipment Supplier
invoiceNumber = EQ-0091
```

Whether this is also represented in Inventory depends on the equipment workflow.

If the equipment is stock-tracked, an Inventory_Transaction may also exist.

---

# 59. Repair vs Maintenance

The finalized model contains both:

```text
Repair
Maintenance
```

Use them consistently.

Example:

```text
Repair:
Broken milking machine fixed
```

```text
Maintenance:
Routine machine servicing
```

This improves category-wise financial analysis.

---

# 60. Miscellaneous

Use:

```text
Miscellaneous
```

only when the expense does not fit an existing finalized category.

Do not use it as a default for everything.

---

# 61. Payment Status Workflow

Typical lifecycle:

```text
Expense Created
      ↓
Pending
      ↓
Payment Made
      ↓
Paid
```

If the expense is invalid before settlement:

```text
Pending
   ↓
Cancelled
```

A paid expense should not normally be casually changed to Cancelled without an authorized correction process.

---

# 62. Paid Expense Editing

Once:

```text
status = Paid
```

financial history becomes sensitive.

Avoid casually editing:

```text
amount
expenseDate
paymentMethod
paidTo
referenceId
```

after downstream reports depend on them.

If a correction is necessary:

```text
AuditLog
+
controlled correction
```

should preserve the history.

---

# 63. Cancelled Expense History

Do not hard-delete cancelled expenses.

Keep:

```text
status = Cancelled
```

so that:

```text
Audit
History
```

remain available.

---

# 64. Expense + AuditLog

The finalized AuditLog explicitly supports:

```text
entityType = Expense
```

and actions such as:

```text
Create
Update
Delete
Archive
Approve
Reject
```

with:

```text
oldData
newData
changedFields
requestId
status
createdAt
```

fileciteturn20file3turn20file4

Therefore, financial changes should be auditable.

---

# 65. What AuditLog Answers

Expense:

```text
How much?
Why?
When?
Who received it?
```

AuditLog:

```text
Who changed the Expense record?
What fields changed?
When?
From what value to what value?
Was the operation successful?
```

These are complementary.

---

# 66. Expense + Report

The finalized Report model supports:

```text
Expense
Income
Finance
```

report types. fileciteturn20file2

Therefore Expense is a primary source for:

```text
Expense Report
Financial Report
Profit/Loss Report
Monthly Report
```

---

# 67. Monthly Expense Report

For a month:

```text
Total Expense
```

can be calculated by:

```text
expenseDate
```

Example:

```text
August 2026

Feed Purchase       ৳50,000
Salary              ৳30,000
Electricity          ৳8,000
Medicine             ৳6,000
Transportation       ৳5,000
--------------------------------
Total                ৳99,000
```

Cancelled expenses should normally be excluded.

---

# 68. Category-Wise Expense Analysis

Example:

```text
Feed Purchase        50%
Salary               30%
Electricity           8%
Medicine              6%
Others                6%
```

This can be visualized as:

```text
Pie Chart
Bar Chart
Monthly Trend
```

---

# 69. Date Filtering

Expense listing should support:

```text
Today
Last 7 days
Last 30 days
This month
Previous month
Custom date range
```

The SRS requires monthly financial reporting, and date-based filtering is useful for long-term history. fileciteturn20file7

---

# 70. Expense Search

Useful search fields:

```text
expenseNo
expenseCategory
paidTo
invoiceNumber
description
```

Example:

```text
Search:
ABC Agro
```

returns expenses where:

```text
paidTo = ABC Agro
```

or other supported searchable text fields.

---

# 71. Expense Pagination

Do not load thousands of historical expenses in one request.

Use:

```text
pagination
```

or:

```text
cursor-based loading
```

for large farms.

This is especially important because the system is designed for long-term farm history.

---

# 72. Expense Indexes

Recommended:

```javascript
{
    farmId: 1,
    expenseDate: -1
}
```

Useful for:

```text
Monthly reports
Recent expenses
Date range filtering
```

Another useful index:

```javascript
{
    farmId: 1,
    expenseCategory: 1,
    expenseDate: -1
}
```

for category-wise reports.

Another:

```javascript
{
    farmId: 1,
    referenceType: 1,
    referenceId: 1
}
```

for linked business events.

---

# 73. Expense Number Index

Recommended unique index:

```text
expenseNo
```

For multi-farm deployments, an application may choose:

```text
farmId + expenseNo
```

if numbering is only guaranteed within a farm.

The finalized schema does not specify the uniqueness strategy, so this should be decided during implementation.

---

# 74. Expense + Inventory Purchase Integrity

For an inventory purchase:

```text
Inventory_Transaction
        ↓
Expense
```

The system should validate:

```text
Expense.amount
=
Inventory_Transaction.totalAmount
```

for the linked purchase when they represent the same financial event.

Example:

```text
Inventory Transaction:
৳4,000

Expense:
৳4,000
```

---

# 75. Avoid Duplicate Expense

Potential bug:

```text
User purchases feed
        ↓
Inventory automatically creates Expense
        ↓
User manually creates another Feed Purchase Expense
```

Result:

```text
Actual cost = ৳4,000
System total = ৳8,000
```

This is a serious finance integrity problem.

Therefore, purchase workflows should detect whether an Expense already exists for the corresponding:

```text
InventoryTransaction
```

---

# 76. Reference-Based Duplicate Prevention

For automatically generated purchase expenses:

```text
referenceType = InventoryTransaction
referenceId = transaction._id
```

The application can enforce:

```text
One active Expense per InventoryTransaction
```

when appropriate.

This is a strong candidate for a unique/partial index strategy.

---

# 77. Expense Creation Modes

There are two broad ways to create Expense:

### Automatic

Example:

```text
Inventory Purchase
→ Expense automatically generated
```

### Manual

Example:

```text
Electricity bill
→ User creates Expense manually
```

Both are supported by the finalized `referenceType` design.

---

# 78. Manual Expense

Example:

```text
expenseCategory = Electricity
amount = 5000
expenseDate = 2026-08-25
paymentMethod = Bank
referenceType = Manual
paidTo = Electricity Provider
status = Paid
```

No external business document is required.

---

# 79. Treatment Expense

Example:

```text
Treatment
      ↓
Actual veterinary/service cost
      ↓
Expense
```

Use:

```text
referenceType = Treatment
referenceId = Treatment._id
```

This allows health-cost analysis.

---

# 80. Salary Expense

If a salary record exists:

```text
referenceType = Salary
referenceId = Salary._id
```

Otherwise:

```text
referenceType = Manual
```

can be used for a manually entered salary expense.

Do not create a fake Salary reference.

---

# 81. `paidTo` and Reference ID

Both can be useful.

Example:

```text
referenceId:
Treatment._id

paidTo:
Dr. Rahman
```

The reference provides structured linkage.

`paidTo` provides a convenient human-readable snapshot.

---

# 82. Why Store `paidTo` Instead of Only User/Entity ID?

Because the receiver may be:

```text
Supplier
Doctor
Utility provider
Transport provider
Individual worker
```

and the current finalized schema does not define one universal `partyId` model.

Therefore:

```text
paidTo
```

provides flexible real-world support.

---

# 83. Expense Attachments

Typical attachments:

```text
Feed invoice
Medicine bill
Electricity bill
Repair receipt
Salary proof
Payment screenshot
```

The attachment should remain linked to the Expense for audit/documentation.

---

# 84. Attachment Security

Because financial documents may be sensitive, access should be protected by:

```text
authentication
farm authorization
role permissions
```

A user from another farm must not be able to retrieve another farm's attachment.

This follows the SRS's farm data isolation requirement. fileciteturn20file0

---

# 85. Role-Based Expense Permissions

The SRS defines:

```text
Admin → Full Access
Farm Owner → Manage Everything
Farm Manager → Regular Management
Worker → Data Entry
```

fileciteturn20file7

A reasonable policy is:

```text
Worker:
Create/view permitted operational expenses

Farm Manager:
Create/update normal expenses

Farm Owner:
Full finance management

Admin:
Full system access
```

The exact permission matrix should be implemented at the authorization layer.

---

# 86. Expense Approval Limitation

The current Expense model has:

```text
Pending | Paid | Cancelled
```

but no:

```text
approvedBy
approvalDate
approvalStatus
```

Therefore, do not claim a separate expense approval workflow from this schema alone.

`Pending` should not automatically mean:

```text
Awaiting approval
```

unless the application explicitly defines that meaning.

---

# 87. Pending ≠ Approval

Important distinction:

```text
Pending
```

can mean:

```text
Payment not yet completed
```

It does not necessarily mean:

```text
Manager has not approved it
```

because there is no explicit approval field in the finalized model.

---

# 88. Expense + Profit/Loss

The basic financial formula is:

```text
Profit/Loss
=
Total Income
-
Total Expense
```

Expense contributes:

```text
Total Expense
```

Income is handled separately by the Income model.

---

# 89. Expense Status in Profit/Loss

A consistent policy should be applied.

Recommended operational reporting:

```text
Paid
→ actual paid expense

Pending
→ outstanding expense

Cancelled
→ excluded
```

Then dashboards can show:

```text
Paid Expenses
Outstanding Expenses
Total Recorded Expenses
```

This avoids ambiguity.

---

# 90. Expense + Income

The finance architecture becomes:

```text
Income
   +
Expense
   ↓
Profit / Loss
```

Example:

```text
Income = ৳150,000
Expense = ৳100,000

Profit = ৳50,000
```

---

# 91. Monthly Profit/Loss

For August:

```text
Total Income:
৳180,000

Total Expense:
৳120,000

Net Profit:
৳60,000
```

The date dimension should use:

```text
incomeDate
expenseDate
```

respectively.

---

# 92. Expense Trend

Monthly aggregation can show:

```text
May     ৳80k
June    ৳95k
July    ৳110k
August  ৳120k
```

This can help identify increasing operational costs.

---

# 93. Category Trend

Example:

```text
Feed Purchase

May      ৳40k
June     ৳42k
July     ৳48k
August   ৳50k
```

This can support feed cost analysis.

---

# 94. Expense Per Animal

The current Expense model does not contain:

```text
animalId
```

Therefore, general expenses cannot always be directly assigned to one animal.

However, treatment-related expenses can be connected indirectly:

```text
Expense
   ↓
referenceType = Treatment
   ↓
Treatment.animalId
```

This enables animal-specific treatment cost analysis.

---

# 95. Expense Per Farm

Because:

```text
farmId
```

is required, the system can calculate:

```text
Farm A:
৳100,000

Farm B:
৳150,000
```

This is important for multi-farm support.

---

# 96. Expense + Inventory Cost

For feed/medicine/vaccine purchases:

```text
Expense
      ↕
Inventory_Transaction
```

can provide:

```text
Purchase Cost
+
Stock Quantity
```

Together:

```text
Cost per kg
Cost per bottle
Cost per unit
```

can be analyzed.

---

# 97. Expense + Feed Cost Analysis

Example:

```text
Feed Purchase:
৳50,000

Feed consumption:
1,250 kg
```

A simple historical cost metric:

```text
৳50,000 / 1,250
=
৳40/kg
```

The exact costing method should be defined if stock from multiple purchase batches is involved.

---

# 98. Expense + Health Cost Analysis

Example:

```text
Veterinary Service:
৳10,000

Medicine Purchase:
৳15,000

Total Health-related Expense:
৳25,000
```

This can support:

```text
Health Cost Report
```

even though the current Report enum calls the broader report type:

```text
Health
```

or:

```text
Expense
```

---

# 99. Expense + Inventory Transaction Atomicity

For automatic purchase expense creation:

```text
BEGIN DATABASE TRANSACTION

1. Create Inventory_Transaction
2. Update Inventory stock
3. Create Expense

COMMIT
```

If step 3 fails:

```text
ROLLBACK
```

This prevents stock and finance from becoming inconsistent.

---

# 100. Example of Inconsistency Without Atomicity

Bad scenario:

```text
Inventory stock:
+100 kg
```

but:

```text
Expense:
not created
```

Now:

```text
Stock says purchased
Finance says no expense
```

This makes:

```text
Profit/Loss
Inventory valuation
Purchase history
```

inconsistent.

---

# 101. Expense Update Rules

Normal editable fields may include:

```text
expenseCategory
expenseDate
paymentMethod
paidTo
invoiceNumber
description
attachment
```

But changes to:

```text
amount
referenceType
referenceId
status
```

should be more strictly controlled because they affect financial integrity.

---

# 102. Expense Delete Rules

Avoid hard deletion once the expense has been:

```text
Paid
included in reports
linked to InventoryTransaction
```

Prefer:

```text
status = Cancelled
```

with:

```text
AuditLog
```

when appropriate.

---

# 103. Cancelled Expense and Inventory

If an expense is linked to an Inventory purchase:

```text
Expense → InventoryTransaction
```

cancelling the Expense does **not automatically mean** the stock purchase should disappear.

The business meaning must be evaluated.

For example:

### Payment record cancelled

Stock may still exist.

### Entire purchase cancelled/returned

Inventory may need:

```text
Return transaction
```

Therefore, Finance cancellation and Inventory reversal are not automatically identical.

---

# 104. Important Separation

```text
Expense.status = Cancelled
```

means:

```text
This financial Expense record is no longer active.
```

It does not necessarily mean:

```text
The underlying stock event never happened.
```

This distinction is critical.

---

# 105. Purchase Cancellation Example

Suppose:

```text
Feed purchased:
100 kg
```

Inventory:

```text
+100 kg
```

Expense:

```text
৳4,000
```

If the purchase is later returned:

```text
Inventory_Transaction:
Return
100 kg
```

and the related financial records should be corrected through the finance workflow.

Do not simply delete the original Expense and transaction.

---

# 106. Expense Reconciliation

Periodic finance reconciliation can compare:

```text
Inventory Purchase Transactions
vs
Feed/Medicine/Vaccine Purchase Expenses
```

For automatically linked purchases:

```text
Every qualifying InventoryTransaction
should have the corresponding Expense.
```

This can detect missing financial records.

---

# 107. Finance Reconciliation Rule

Example:

```text
Purchase Transactions:
20

Linked Expenses:
19
```

System should flag:

```text
1 purchase transaction without Expense
```

for investigation.

Similarly:

```text
Expense referenceType = InventoryTransaction
```

should point to a valid transaction.

---

# 108. Duplicate Expense Detection

Potential duplicate signals:

```text
same farmId
same referenceType
same referenceId
same amount
```

especially:

```text
referenceType = InventoryTransaction
```

A duplicate should normally be rejected or require explicit override.

---

# 109. Manual Expense Duplicate Risk

Manual expenses have:

```text
referenceType = Manual
```

so duplicate detection cannot rely only on referenceId.

For manual records, useful warning signals can include:

```text
same date
same amount
same category
same paidTo
same invoiceNumber
```

but these should be treated as warnings rather than absolute proof of duplication.

---

# 110. Expense Invoice Verification

For purchase expenses with:

```text
invoiceNumber
```

the UI can show:

```text
Invoice
Amount
Supplier
Date
Attachment
```

This helps farm managers verify purchases.

---

# 111. Expense Attachment + Audit

If an attachment is replaced:

```text
old attachment
→ new attachment
```

this change should be captured in AuditLog because the attachment can be part of financial evidence.

---

# 112. Financial Security

Expenses should be protected by:

```text
Authentication
Farm authorization
Role-based access
Audit logging
```

The SRS requires password security, farm data isolation, and role-based access on protected routes. fileciteturn20file0

---

# 113. Expense Data Privacy

Financial information can include:

```text
supplier
doctor
salary
invoice
payment details
```

Therefore, normal farm users should only access data belonging to farms where they have appropriate membership/permission.

---

# 114. Realtime Expense Updates

The Expense collection itself does not provide realtime delivery.

For realtime dashboard updates:

```text
Expense Created
      ↓
Finance Service
      ↓
Database commit
      ↓
Emit realtime event
      ↓
Dashboard updates
```

Possible infrastructure:

```text
Socket.IO
WebSocket
SSE
```

The DB model remains independent from the realtime transport.

---

# 115. Realtime Finance Dashboard

Example:

```text
New Expense:
৳4,000

Database:
Expense saved

Realtime event:
expense.created

Dashboard:
Total expense increases instantly
```

---

# 116. Finance Dashboard Cards

Useful calculated metrics:

```text
Today's Expenses
This Month's Expenses
Pending Expenses
Paid Expenses
Top Expense Category
Total Income
Net Profit/Loss
```

These are derived values, not stored fields.

---

# 117. Monthly Summary

Example:

```text
August 2026

Expenses:
৳120,000

Income:
৳180,000

Net:
৳60,000
```

Category breakdown:

```text
Feed:
৳50,000

Salary:
৳30,000

Health:
৳21,000

Utilities:
৳8,000

Other:
৳11,000
```

---

# 118. Expense Report

The Report model supports:

```text
reportType = Expense
```

and:

```text
reportFormat =
PDF | Excel | CSV | JSON
```

fileciteturn20file2

Therefore Expense data can be exported/generated in those report formats.

---

# 119. Financial Report

For broader reporting:

```text
reportType = Finance
```

can combine:

```text
Income
+
Expense
+
Profit/Loss
```

---

# 120. Report Date Range

The finalized Report model contains:

```text
dateRange:
    from
    to
```

fileciteturn20file2

Therefore an Expense Report can preserve:

```text
From:
2026-08-01

To:
2026-08-31
```

as part of the generated report metadata.

---

# 121. Expense Search and Filtering

Recommended filters:

```text
expenseCategory
status
paymentMethod
expenseDate range
paidTo
referenceType
```

Example:

```text
Category:
Feed Purchase

Status:
Paid

Month:
August 2026
```

---

# 122. Expense Sorting

Useful default:

```text
expenseDate DESC
```

so recent expenses appear first.

Secondary sorting:

```text
createdAt DESC
```

can resolve ties.

---

# 123. Pagination

For long-term history:

```text
page
limit
```

or cursor pagination should be used.

Example:

```text
20 expenses per request
```

This keeps normal requests fast.

The SRS expects normal requests to be responsive, and the system is intended for long-term farm data. fileciteturn20file0

---

# 124. Index Recommendations

Recommended:

```javascript
{
    farmId: 1,
    expenseDate: -1
}
```

For category reports:

```javascript
{
    farmId: 1,
    expenseCategory: 1,
    expenseDate: -1
}
```

For linked records:

```javascript
{
    farmId: 1,
    referenceType: 1,
    referenceId: 1
}
```

For status dashboards:

```javascript
{
    farmId: 1,
    status: 1,
    expenseDate: -1
}
```

---

# 125. Expense Number Uniqueness

Recommended:

```text
expenseNo
```

should be unique according to the chosen numbering scope.

Possible approach:

```text
EXP-000001
```

globally unique, or:

```text
Farm-specific sequence
```

The finalized schema does not dictate which strategy must be used.

---

# 126. Example — Feed Purchase Expense

```javascript
{
    expenseNo: "EXP-000001",

    farmId: "FARM-001",

    expenseCategory: "Feed Purchase",

    amount: 4000,

    expenseDate: "2026-08-20",

    paymentMethod: "Cash",

    referenceType: "InventoryTransaction",

    referenceId: "ITX-000001",

    paidTo: "ABC Agro Supplier",

    invoiceNumber: "ABC-2026-458",

    description: "100 kg feed purchase",

    attachment: "/uploads/invoices/abc-458.pdf",

    status: "Paid",

    createdBy: "USER-001"
}
```

Values are illustrative.

---

# 127. Example — Electricity Expense

```javascript
{
    expenseNo: "EXP-000002",

    farmId: "FARM-001",

    expenseCategory: "Electricity",

    amount: 7500,

    expenseDate: "2026-08-25",

    paymentMethod: "Bank",

    referenceType: "Manual",

    paidTo: "Electricity Provider",

    invoiceNumber: "ELEC-2026-08",

    description: "August electricity bill",

    status: "Paid",

    createdBy: "USER-001"
}
```

---

# 128. Example — Veterinary Expense

```javascript
{
    expenseNo: "EXP-000003",

    farmId: "FARM-001",

    expenseCategory: "Veterinary Service",

    amount: 2000,

    expenseDate: "2026-08-26",

    paymentMethod: "Cash",

    referenceType: "Treatment",

    referenceId: "TR-000025",

    paidTo: "Veterinary Service Provider",

    description: "Veterinary service for animal treatment",

    status: "Paid",

    createdBy: "USER-002"
}
```

---

# 129. Example — Pending Expense

```javascript
{
    expenseNo: "EXP-000004",

    farmId: "FARM-001",

    expenseCategory: "Feed Purchase",

    amount: 10000,

    expenseDate: "2026-08-27",

    paymentMethod: "Bank",

    referenceType: "InventoryTransaction",

    referenceId: "ITX-000020",

    paidTo: "Feed Supplier",

    status: "Pending",

    createdBy: "USER-001"
}
```

This represents an outstanding payment under the application's finance policy.

---

# 130. Example — Cancelled Expense

```javascript
{
    expenseNo: "EXP-000005",

    farmId: "FARM-001",

    expenseCategory: "Equipment",

    amount: 15000,

    expenseDate: "2026-08-28",

    paymentMethod: "Cash",

    referenceType: "Manual",

    description: "Cancelled equipment order",

    status: "Cancelled",

    createdBy: "USER-001"
}
```

This record remains in history but should normally be excluded from active expense totals.

---

# 131. Error Scenario — Wrong Farm

```text
Expense.farmId = FARM-A
Reference transaction.farmId = FARM-B
```

Result:

```text
Reject
```

Cross-farm references must not be allowed.

---

# 132. Error Scenario — Invalid Reference

```text
referenceType = InventoryTransaction
referenceId = nonexistent
```

Result:

```text
Reject
```

unless the reference is explicitly optional for the chosen workflow.

---

# 133. Error Scenario — Wrong Reference Type

```text
referenceType = Treatment
referenceId = InventoryTransaction._id
```

Result:

```text
Reject
```

---

# 134. Error Scenario — Negative Amount

```text
amount = -500
```

Result:

```text
Reject
```

---

# 135. Error Scenario — Invalid Payment Method

```text
paymentMethod = Bitcoin
```

Result:

```text
Reject
```

because it is not among the finalized enum values.

---

# 136. Error Scenario — Invalid Expense Category

```text
expenseCategory = Food
```

Result:

```text
Reject
```

unless the schema is intentionally updated later.

The current finalized model does not include `Food`. fileciteturn20file5

---

# 137. Error Scenario — Paid Without Payment Information

If:

```text
status = Paid
```

the application should normally ensure the payment workflow has enough information to explain the payment.

For example:

```text
paymentMethod
```

should normally be present.

The exact validation policy should be enforced in the service layer.

---

# 138. Error Scenario — Duplicate Purchase Expense

If:

```text
referenceType = InventoryTransaction
referenceId = ITX-001
```

already has an active Expense, creating another automatic Expense should normally be rejected or flagged.

This prevents double counting.

---

# 139. Expense Service Layer

Recommended service methods:

```text
expenseService.createExpense()
expenseService.updateExpense()
expenseService.cancelExpense()
expenseService.markPaid()
expenseService.getExpenses()
expenseService.getExpenseById()
expenseService.getMonthlySummary()
expenseService.getCategorySummary()
expenseService.getPendingExpenses()
expenseService.getPaidExpenses()
expenseService.getExpenseReport()
expenseService.reconcileInventoryPurchases()
```

These are service-level responsibilities, not new DB fields.

---

# 140. Automatic Purchase Expense Service

Recommended:

```text
createInventoryPurchase()
        ↓
Create Inventory_Transaction
        ↓
Update Inventory
        ↓
Create Expense
        ↓
Link Expense.referenceId
        ↓
AuditLog
```

For critical operations, use a database transaction.

---

# 141. Marking an Expense as Paid

Possible flow:

```text
Pending Expense
      ↓
User confirms payment
      ↓
Validate payment method
      ↓
status = Paid
      ↓
updatedBy = current user
      ↓
updatedAt = now
      ↓
AuditLog
      ↓
Realtime finance update
```

---

# 142. Cancelling an Expense

Possible flow:

```text
Expense
      ↓
Authorized user
      ↓
Cancellation reason in audit/description workflow
      ↓
status = Cancelled
      ↓
AuditLog
      ↓
Dashboard/report recalculation
```

The finalized schema does not contain a dedicated cancellation-reason field.

---

# 143. Expense Audit Event

Example:

```text
entityType = Expense
entityId = EXP-001
action = Update
changedFields = ["amount", "status"]
oldData = ...
newData = ...
status = Success
```

The finalized AuditLog supports Expense as an entity type and stores old/new data plus changed fields. fileciteturn20file3

---

# 144. Financial Integrity Architecture

The recommended architecture is:

```text
                 Finance
                    │
          ┌─────────┴─────────┐
          ▼                   ▼
       Expense              Income
          │                   │
          └─────────┬─────────┘
                    ▼
               Profit/Loss
```

For inventory purchases:

```text
Inventory_Transaction
          │
          ▼
       Expense
```

For treatment:

```text
Treatment
    │
    ▼
Expense
```

For salary:

```text
Salary workflow
    │
    ▼
Expense
```

---

# 145. Realtime Finance Architecture

```text
User Action
    ↓
Expense Service
    ↓
MongoDB Transaction
    ├── Expense
    ├── Related records if needed
    └── AuditLog
    ↓
Commit
    ↓
Realtime Event
    ↓
Finance Dashboard
```

Possible events:

```text
expense.created
expense.updated
expense.paid
expense.cancelled
```

---

# 146. Final Business Rules Checklist

```text
[✓] Expense represents a financial outflow
[✓] Inventory_Transaction represents stock movement
[✓] One purchase can create both records
[✓] They must not be double-counted financially
[✓] expenseNo should be human-readable and unique by chosen scope
[✓] farmId is required
[✓] all queries must be farm-scoped
[✓] expenseCategory is controlled by finalized enum
[✓] amount should normally be positive
[✓] expenseDate represents business date
[✓] createdAt represents document creation time
[✓] paymentMethod uses finalized enum
[✓] referenceType identifies the source business event
[✓] referenceId links to the source record
[✓] referenceType/referenceId must be validated together
[✓] paidTo stores human-readable payment recipient
[✓] invoiceNumber stores supplier/vendor invoice reference
[✓] description stores additional context
[✓] attachment stores supporting file reference
[✓] status uses Pending/Paid/Cancelled
[✓] Pending and Paid should be clearly distinguished in reports
[✓] Cancelled expenses should normally be excluded from active totals
[✓] createdBy identifies creator
[✓] updatedBy identifies last modifier
[✓] financial changes should be audited
[✓] purchase expenses should link to InventoryTransaction when applicable
[✓] duplicate purchase expenses should be prevented
[✓] Treatment estimatedCost is not automatically actual Expense
[✓] Inventory purchase and Expense should be synchronized
[✓] critical purchase flows should be atomic
[✓] cancellation of Expense does not automatically reverse stock
[✓] stock reversal requires Inventory_Transaction logic
[✓] historical financial records should not be casually deleted
[✓] reports should normally use expenseDate
[✓] date/category/status filtering should be supported
[✓] monthly expense reporting is required
[✓] Expense contributes to Profit/Loss
[✓] Income is handled separately
[✓] realtime delivery belongs to application infrastructure
[✓] current schema has no explicit approval fields
[✓] current schema has no dedicated cancellation-reason field
[✓] current finalized model has no dedicated Supplier model
[✓] current finalized model has no dedicated Salary collection in the core list
```

---

# 147. Final Model Summary

```text
                  Expense
                     │
       ┌─────────────┼─────────────┐
       │             │             │
       ▼             ▼             ▼
   Inventory      Treatment      Manual
  Transaction
       │             │             │
       └─────────────┼─────────────┘
                     ▼
              Financial History
                     │
                     ▼
              Profit / Loss
                     ▲
                     │
                  Income
```

The main responsibility of `Expense` is:

> **Maintain a traceable financial record of money spent by each farm and provide the financial foundation for expense analysis, monthly reporting, and profit/loss calculation.**

---

# 148. Final Status

**Model:** `Expense`  
**Status:** ✅ Finalized  
**Business Logic:** ✅ Documented  
**Finance Integration:** ✅  
**Inventory Integration:** ✅  
**Treatment Integration:** ✅  
**Salary Reference Support:** ✅  
**Payment Tracking:** ✅  
**Invoice Support:** ✅  
**Attachment Support:** ✅  
**Auditability:** ✅  
**Profit/Loss Integration:** ✅  
**Reporting:** ✅  
**Realtime Architecture:** ✅  
**Historical Integrity:** ✅  

**Next Model:** `Income`
