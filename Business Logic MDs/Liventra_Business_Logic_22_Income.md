# Liventra — Business Logic Documentation
## Model 22: `Income`

**Document Version:** Business Logic v1.0  
**Project:** Liventra — Smart Livestock Farm Management System  
**Database:** MongoDB  
**Architecture:** MERN  
**Module:** Finance & Income Management

---

# 1. Source / Finalized Model

The latest finalized Liventra DB Model defines `Income` as:

```text
Income Model:
_id
incomeNo
farmId
incomeCategory:
    Milk Sale
    Animal Sale
    Calf Sale
    Manure Sale
    Breeding Service
    Insurance Claim
    Government Subsidy
    Other

amount
incomeDate
paymentMethod:
    Cash
    Bank
    Mobile Banking
    Cheque
    Card

referenceType:
    MilkRecord
    Animal
    Breeding
    Manual

referenceId
customerName
customerPhone
invoiceNumber
quantity
unit:
    Liter
    Kg
    Piece
    Service

unitPrice
description
attachment
status:
    Pending
    Received
    Cancelled

createdBy
updatedBy
createdAt
updatedAt
```

These fields and enum values are defined in the latest finalized DB Models document. fileciteturn21file1

The SRS places Income inside the Finance module together with Expense and Profit/Loss, and requires financial reporting. fileciteturn21file11

---

# 2. Purpose

`Income` records **money earned or expected to be received by a farm**.

It answers:

> "How much money came into the farm, where did it come from, when did it happen, how was it received, who paid it, what business event generated it, and whether the money has actually been received?"

Example:

```text
Income No:
INC-000125

Category:
Milk Sale

Amount:
৳6,000

Date:
2026-08-20

Customer:
ABC Dairy Buyer

Status:
Received
```

---

# 3. Income's Position in Liventra

Income is one of the two main financial flows.

```text
                    Finance
                       │
          ┌────────────┴────────────┐
          ▼                         ▼
       Expense                    Income
          │                         │
          └────────────┬────────────┘
                       ▼
                  Profit / Loss
```

The SRS explicitly identifies:

```text
Expense
Income
Profit / Loss
Monthly Report
```

as Finance module functionality. fileciteturn21file11

---

# 4. Income vs Expense

This distinction is fundamental.

## Expense

Money going out:

```text
Farm → Supplier
Farm → Worker
Farm → Service Provider
```

## Income

Money coming in:

```text
Customer → Farm
Government → Farm
Insurance Provider → Farm
```

Therefore:

```text
Income
↑
Money enters farm

Expense
↓
Money leaves farm
```

---

# 5. Income vs Operational Records

An operational record does not automatically mean money was received.

Example:

```text
MilkRecord
```

means:

```text
Milk was produced.
```

It does not necessarily mean:

```text
Milk was sold and payment received.
```

Therefore:

```text
MilkRecord
    ↓
Milk production event

Income
    ↓
Milk sale financial event
```

This is why the finalized Income model supports:

```text
referenceType = MilkRecord
```

fileciteturn21file0

---

# 6. Most Important Business Rule

## Do not confuse production/sale event with financial receipt.

Example:

```text
100 Liter milk produced
```

does not automatically mean:

```text
৳8,000 income received
```

A sale transaction needs its own Income record.

This preserves accurate production-vs-finance history.

---

# 7. Field-by-Field Details

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

Used by references such as:

```text
referenceId
```

---

# 8. `incomeNo`

### Type

String

### Required

Yes

### Example

```text
INC-000001
```

### Purpose

Human-readable income identifier.

Useful for:

```text
Income history
Search
Reports
Invoices
Audit
Customer transactions
```

---

# 9. Income Number Generation

Recommended format:

```text
INC-{SEQUENCE}
```

Examples:

```text
INC-000001
INC-000002
INC-000003
```

The backend should generate it.

The frontend should not be responsible for uniqueness.

---

# 10. `farmId`

### Type

ObjectId / Farm reference

### Required

Yes

### Purpose

Identifies the farm that earned the income.

Because Liventra supports multiple farms, every Income record must be farm-scoped.

---

# 11. Farm Isolation Rule

Before creating or viewing Income:

```text
Income.farmId
```

must belong to a farm that the authenticated user is authorized to access.

The SRS explicitly requires prevention of cross-farm data access and role-based protected routes. fileciteturn21file11

---

# 12. `incomeCategory`

### Type

Enum

### Required

Yes

### Finalized Values

```text
Milk Sale
Animal Sale
Calf Sale
Manure Sale
Breeding Service
Insurance Claim
Government Subsidy
Other
```

These values come directly from the finalized model. fileciteturn21file0

---

# 13. Why Income Category Matters

It answers:

> "Where is the farm earning its money?"

Example:

```text
Milk Sale          ৳100,000
Animal Sale         ৳50,000
Calf Sale           ৳20,000
Manure Sale          ৳5,000
Breeding Service     ৳8,000
Government Subsidy  ৳10,000
```

This enables:

```text
Revenue analysis
Category-wise income
Monthly reports
Profit/Loss
Dashboard charts
Business decisions
```

---

# 14. Category → Business Source

Recommended interpretation:

```text
Milk Sale
    ↓
MilkRecord

Animal Sale
    ↓
Animal

Calf Sale
    ↓
Animal / relevant sale event

Manure Sale
    ↓
Manual

Breeding Service
    ↓
Breeding

Insurance Claim
    ↓
Manual

Government Subsidy
    ↓
Manual

Other
    ↓
Manual
```

Important:

The finalized `referenceType` does not contain:

```text
Manure
Insurance
GovernmentSubsidy
```

Therefore those categories cannot claim a dedicated reference collection under the current schema.

They can use:

```text
referenceType = Manual
```

where appropriate.

---

# 15. `amount`

### Type

Number

### Required

Yes

### Example

```text
8000
```

Meaning:

```text
৳8,000
```

### Purpose

Stores the monetary value of the income record.

---

# 16. Amount Rule

Recommended:

```text
amount > 0
```

An income record should normally represent a positive financial amount.

Do not use negative Income amounts to represent refunds or losses.

If money must flow back out of the farm, that is conceptually an Expense/correction workflow.

---

# 17. Currency

The project context is Bangladesh, so financial UI/reporting can display:

```text
৳ / BDT
```

The finalized Income model has no `currency` field.

Therefore, do not add per-record currency logic without updating the schema.

---

# 18. `incomeDate`

### Type

Date

### Required

Yes

### Purpose

Records when the income event occurred.

Example:

```text
2026-08-20
```

---

# 19. `incomeDate` vs `createdAt`

These are different.

```text
incomeDate
=
When the sale/earning event occurred

createdAt
=
When the Income document was created
```

Example:

```text
Milk sold:
Aug 18

Entered into system:
Aug 20
```

Then:

```text
incomeDate = Aug 18
createdAt = Aug 20
```

Financial reports should normally use:

```text
incomeDate
```

rather than:

```text
createdAt
```

---

# 20. Backdated Income

Authorized users may need to record a previous sale.

Example:

```text
Sale happened:
Aug 10

Recorded:
Aug 17
```

This is why:

```text
incomeDate
```

must remain separate from:

```text
createdAt
```

Such backdated entries should remain auditable.

---

# 21. `paymentMethod`

### Type

Enum

### Required

Yes / according to workflow

### Finalized Values

```text
Cash
Bank
Mobile Banking
Cheque
Card
```

These values are defined in the finalized model. fileciteturn21file0

---

# 22. Payment Method Meaning

Records how the customer/payer provided the money.

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

---

# 23. `referenceType`

### Type

Enum

### Required

Conditional

### Finalized Values

```text
MilkRecord
Animal
Breeding
Manual
```

These are the exact finalized values. fileciteturn21file0

---

# 24. Why `referenceType` Exists

It answers:

> "Which operational/business record generated this income?"

Examples:

```text
MilkRecord
→ milk sale related to milk production record

Animal
→ animal sale

Breeding
→ breeding service income

Manual
→ subsidy, insurance, manure sale, or other manually recorded income where no supported reference exists
```

---

# 25. `referenceId`

### Type

ObjectId

### Required

Conditional

### Purpose

Points to the source record.

Examples:

```text
referenceType = MilkRecord
referenceId = MilkRecord._id
```

or:

```text
referenceType = Animal
referenceId = Animal._id
```

---

# 26. Reference Integrity

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
referenceId = MilkRecord._id
```

The backend must reject inconsistent references.

---

# 27. Reference Mapping

Recommended mapping:

| Income Category | Reference Type | Reference |
|---|---|---|
| Milk Sale | MilkRecord | MilkRecord._id |
| Animal Sale | Animal | Animal._id |
| Calf Sale | Animal | Animal._id where applicable |
| Breeding Service | Breeding | Breeding._id |
| Manure Sale | Manual | No dedicated source |
| Insurance Claim | Manual | No dedicated source |
| Government Subsidy | Manual | No dedicated source |
| Other | Manual | Depends on event |

This follows the finalized schema without inventing unsupported reference types. fileciteturn21file0

---

# 28. `customerName`

### Type

String

### Required

Optional / conditional

### Example

```text
Rahim Dairy Farm
```

### Purpose

Stores the buyer/customer/payer's name.

---

# 29. Why `customerName` Is Important

Example:

```text
Milk Sale
Amount = ৳6,000
customerName = Rahim Dairy Farm
```

The system can display:

```text
Sold to Rahim Dairy Farm
```

without requiring another customer collection.

---

# 30. `customerPhone`

### Type

String

### Required

Optional

### Example

```text
017XXXXXXXX
```

### Purpose

Stores the customer's contact number.

Useful for:

```text
Customer identification
Sale history
Follow-up
Invoice
```

---

# 31. Customer Information Snapshot

`customerName` and `customerPhone` are useful as historical snapshots.

Example:

```text
2026:
Rahim Dairy Farm
01711XXXXXX

2027:
Customer changed phone
```

Old Income records should still preserve the information captured at the time of sale.

---

# 32. `invoiceNumber`

### Type

String

### Required

Optional

### Example

```text
INC-INV-2026-0051
```

### Purpose

Stores invoice/sale reference number.

Useful for:

```text
Sales records
Customer billing
Financial audit
Report generation
```

---

# 33. Invoice Number Uniqueness

Do not blindly make:

```text
invoiceNumber
```

globally unique.

Different business contexts may use overlapping invoice numbers.

If uniqueness is needed, define the appropriate scope during implementation.

---

# 34. `quantity`

### Type

Number

### Required

Conditional

### Purpose

Stores how much was sold or what quantity generated the income.

Examples:

```text
100
```

with:

```text
unit = Liter
```

means:

```text
100 liters
```

---

# 35. Quantity Examples

Milk:

```text
quantity = 100
unit = Liter
```

Animal:

```text
quantity = 1
unit = Piece
```

Breeding Service:

```text
quantity = 1
unit = Service
```

---

# 36. `unit`

### Type

Enum

### Required

Conditional

### Finalized Values

```text
Liter
Kg
Piece
Service
```

These are the exact finalized values. fileciteturn21file0

---

# 37. Unit → Category Mapping

Recommended:

```text
Milk Sale
→ Liter

Animal Sale
→ Piece

Calf Sale
→ Piece

Manure Sale
→ Kg

Breeding Service
→ Service
```

The finalized schema supports these units.

---

# 38. Unit Validation

The service layer should validate logical combinations.

Examples:

```text
Milk Sale + Liter
```

valid.

```text
Animal Sale + Piece
```

valid.

But:

```text
Animal Sale + Liter
```

should normally be rejected.

Similarly:

```text
Breeding Service + Kg
```

does not make normal business sense.

---

# 39. `unitPrice`

### Type

Number

### Required

Conditional

### Purpose

Stores price per unit.

Example:

```text
100 Liter
৳80/Liter
```

Therefore:

```text
unitPrice = 80
```

---

# 40. Amount Calculation

For quantity-based sales:

```text
amount
=
quantity × unitPrice
```

Example:

```text
100 Liter × ৳80
=
৳8,000
```

The backend should calculate or validate this.

Do not blindly trust frontend totals.

---

# 41. Service-Based Income

For:

```text
Breeding Service
```

the income may be:

```text
quantity = 1
unit = Service
unitPrice = 2000
amount = 2000
```

This keeps service income consistent with the same model.

---

# 42. Non-Quantity Income

Some categories may not naturally have:

```text
quantity
unit
unitPrice
```

Examples:

```text
Government Subsidy
Insurance Claim
```

For these:

```text
quantity = null
unit = null
unitPrice = null
```

may be appropriate, while:

```text
amount
```

remains required.

The finalized schema supports these fields but does not explicitly mark them as conditional, so this behavior belongs in service-level validation.

---

# 43. `description`

### Type

String

### Required

Optional

### Purpose

Provides additional income context.

Examples:

```text
Sale of 100 liters of milk.
```

```text
Sale of adult cattle A-001.
```

```text
Government livestock development subsidy.
```

```text
Breeding service provided to external farm.
```

---

# 44. `attachment`

### Type

String / file URL/path

### Required

Optional

### Purpose

Stores supporting financial documents.

Examples:

```text
Sales invoice
Receipt
Payment proof
Insurance document
Government subsidy document
```

---

# 45. Attachment Storage Rule

Recommended:

```text
Upload file
    ↓
Object/file storage
    ↓
Save URL/path
    ↓
Income.attachment
```

Do not put large binary files directly into the Income document unless a deliberate storage architecture is used.

---

# 46. `status`

### Type

Enum

### Required

Yes

### Finalized Values

```text
Pending
Received
Cancelled
```

These are defined in the finalized model. fileciteturn21file0

---

# 47. Status: `Pending`

Meaning:

```text
Income has been recorded/expected, but payment has not yet been received.
```

Example:

```text
Milk sold:
৳8,000

Payment:
not received yet

status:
Pending
```

This is useful for:

```text
Credit sales
Outstanding customer payments
Receivables
```

---

# 48. Status: `Received`

Meaning:

```text
The farm has actually received the money.
```

Example:

```text
amount = ৳8,000
paymentMethod = Cash
status = Received
```

---

# 49. Status: `Cancelled`

Meaning:

```text
The Income record is no longer considered an active valid income event.
```

Cancelled income should normally be excluded from active revenue totals.

---

# 50. Pending vs Received

This distinction is critical.

Example:

```text
Milk Sale:
৳10,000
```

If customer has not paid:

```text
Pending
```

If customer has paid:

```text
Received
```

Therefore:

```text
Recorded Revenue Event
≠
Cash Actually Received
```

depending on the reporting policy.

---

# 51. Profit/Loss Policy

The finalized model supports:

```text
Pending
Received
Cancelled
```

but does not explicitly define the accounting treatment of Pending income.

Therefore, the application should clearly distinguish:

```text
Recorded Income
Received Income
Outstanding Income
Cancelled Income
```

A practical operational policy is:

```text
Received
→ cash received

Pending
→ outstanding receivable

Cancelled
→ excluded
```

For a simple project dashboard, Profit/Loss can use only `Received` amounts if the team chooses a cash-based interpretation. If an accrual-style interpretation is desired, Pending may be included in revenue. The chosen policy must be consistent across reports.

---

# 52. `createdBy`

### Type

ObjectId / User reference

### Required

Yes

### Purpose

Identifies the user who created the Income record.

---

# 53. `updatedBy`

### Type

ObjectId / User reference

### Required

Conditional / automatic on update

### Purpose

Identifies the user who last modified the record.

---

# 54. `createdAt`

### Type

Date

### Required

Automatic

### Purpose

Records when the document was created.

---

# 55. `updatedAt`

### Type

Date

### Required

Automatic

### Purpose

Records the last update time.

---

# 56. Milk Sale — Complete Workflow

```text
MilkRecord
    ↓
Milk produced
    ↓
Customer buys milk
    ↓
Calculate quantity × unitPrice
    ↓
Create Income
    ↓
referenceType = MilkRecord
    ↓
referenceId = MilkRecord._id
    ↓
status = Pending / Received
    ↓
AuditLog
```

The SRS requires daily milk recording and Income/financial management as separate functions. fileciteturn21file14

---

# 57. Milk Sale Example

Suppose:

```text
MilkRecord:
100 Liter

Sale price:
৳80/Liter
```

Income:

```text
incomeCategory = Milk Sale
quantity = 100
unit = Liter
unitPrice = 80
amount = 8000
referenceType = MilkRecord
referenceId = MilkRecord._id
```

If paid immediately:

```text
status = Received
```

If payment is due later:

```text
status = Pending
```

---

# 58. Important Milk Business Rule

Do not automatically create Income merely because:

```text
MilkRecord
```

was created.

Correct:

```text
Milk Production
    ↓
MilkRecord

Milk Sale
    ↓
Income
```

One is production; the other is financial.

---

# 59. Animal Sale — Complete Workflow

```text
Animal
    ↓
Animal selected for sale
    ↓
Sale amount determined
    ↓
Create Income
    ↓
incomeCategory = Animal Sale
    ↓
referenceType = Animal
    ↓
referenceId = Animal._id
```

The Animal's own life status should be updated according to the animal-sale workflow.

---

# 60. Animal Sale Example

```text
Animal:
A-001

Sale Price:
৳180,000
```

Income:

```text
incomeCategory = Animal Sale
amount = 180000
quantity = 1
unit = Piece
unitPrice = 180000
referenceType = Animal
referenceId = Animal._id
```

---

# 61. Animal Sale + Animal Status

When an animal is successfully sold, the Animal model has:

```text
lifeStatus = Sold
```

and:

```text
isActive
```

can be handled according to the finalized Animal lifecycle policy.

The Income record should not itself change the Animal unless the sale workflow explicitly performs both operations.

---

# 62. Sale Transaction Atomicity

Animal sale is a multi-model business event:

```text
Animal
+
Income
```

Therefore, when the system treats both as one confirmed sale, the backend should ideally perform the critical database changes atomically.

Conceptually:

```text
BEGIN

Create Income
Update Animal → Sold

COMMIT
```

If a critical step fails:

```text
ROLLBACK
```

This prevents:

```text
Income says sold
but Animal still Active
```

or the opposite.

---

# 63. Calf Sale

For:

```text
Calf Sale
```

the likely source is an Animal record representing the calf.

Therefore:

```text
incomeCategory = Calf Sale
referenceType = Animal
referenceId = calf Animal._id
```

This allows the financial record to be linked to the actual calf.

---

# 64. Manure Sale

The finalized model does not provide a dedicated Manure collection.

Therefore:

```text
incomeCategory = Manure Sale
referenceType = Manual
```

can be used.

Example:

```text
quantity = 500
unit = Kg
unitPrice = 5
amount = 2500
```

---

# 65. Breeding Service Income

The finalized `Breeding` model records breeding activity. fileciteturn21file5

If the farm provides breeding services to another party:

```text
incomeCategory = Breeding Service
referenceType = Breeding
referenceId = Breeding._id
```

Example:

```text
quantity = 1
unit = Service
unitPrice = 2000
amount = 2000
```

---

# 66. Insurance Claim

The finalized reference enum does not include:

```text
Insurance
```

Therefore:

```text
incomeCategory = Insurance Claim
referenceType = Manual
```

is appropriate under the current schema.

Supporting documentation can be stored using:

```text
attachment
```

---

# 67. Government Subsidy

Similarly, the finalized reference enum does not include:

```text
GovernmentSubsidy
```

Therefore:

```text
incomeCategory = Government Subsidy
referenceType = Manual
```

Supporting documents can be stored in:

```text
attachment
```

---

# 68. Other Income

Use:

```text
incomeCategory = Other
referenceType = Manual
```

when the source does not match one of the supported categories.

Do not use `Other` as the default for normal sales.

---

# 69. Income + Customer

For sales:

```text
customerName
customerPhone
```

should generally be captured when available.

For:

```text
Government Subsidy
Insurance Claim
```

there may not be a conventional customer, so these fields may remain empty or represent the payer organization if the application chooses.

---

# 70. Income + Invoice

For sale-based income:

```text
invoiceNumber
```

can identify the corresponding invoice.

Example:

```text
Milk Sale
Invoice:
MILK-2026-0015
```

This is useful for:

```text
Customer history
Sales reports
Audit
```

---

# 71. Income + Attachment

Typical attachments:

```text
Milk sale receipt
Animal sale agreement
Customer payment proof
Insurance approval document
Government subsidy document
```

---

# 72. Income Editing

Before the income is financially settled, normal descriptive fields can be edited according to permission:

```text
customerName
customerPhone
description
attachment
invoiceNumber
```

Changes to:

```text
amount
incomeCategory
referenceType
referenceId
status
```

should be more strictly controlled because they affect financial reporting.

---

# 73. Received Income Editing

Once:

```text
status = Received
```

the record represents actual received money.

Therefore, avoid casual edits to:

```text
amount
incomeDate
paymentMethod
referenceId
```

Use an authorized correction process and AuditLog.

---

# 74. Cancelled Income

Do not hard-delete cancelled income.

Use:

```text
status = Cancelled
```

so the history remains available.

Cancelled records should normally be excluded from active income totals.

---

# 75. Income Cancellation vs Operational Event

Cancelling Income does not automatically mean:

```text
Milk was never produced
```

or:

```text
Animal was never sold
```

For example:

```text
Income cancelled
```

might mean:

```text
sale/payment record was invalid
```

while the underlying business event may require a separate correction.

Therefore, Income status and operational lifecycle should not be silently coupled.

---

# 76. Income + AuditLog

The finalized AuditLog explicitly supports:

```text
entityType = Income
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

along with:

```text
oldData
newData
changedFields
requestId
status
```

fileciteturn21file7

Therefore, financial income changes should be audited.

---

# 77. Income vs AuditLog

### Income

Answers:

```text
How much was earned?
From what?
When?
From whom?
```

### AuditLog

Answers:

```text
Who created/modified the income?
What changed?
When?
Was the operation successful?
```

They are not interchangeable.

---

# 78. Profit/Loss Integration

Basic financial calculation:

```text
Profit/Loss
=
Total Income
-
Total Expense
```

The SRS explicitly requires Profit/Loss as part of Finance. fileciteturn21file11

---

# 79. Income Status in Profit/Loss

The project should choose and consistently document a policy.

For a simple operational/cash-oriented dashboard:

```text
Received Income
-
Paid Expense
=
Cash-oriented Profit/Loss
```

Pending income can separately appear as:

```text
Outstanding Receivable
```

This is easy to explain to farm users.

If the team later adopts accrual accounting, Pending income can be treated differently. The current DB model itself does not decide this.

---

# 80. Monthly Income Report

For August:

```text
Milk Sale             ৳100,000
Animal Sale            ৳50,000
Calf Sale              ৳20,000
Manure Sale             ৳5,000
Breeding Service        ৳8,000
Government Subsidy     ৳10,000
--------------------------------
Total                  ৳193,000
```

Cancelled income should normally be excluded.

Whether Pending income is included depends on the selected reporting policy.

---

# 81. Category-Wise Income Analysis

Example:

```text
Milk Sale          52%
Animal Sale        26%
Calf Sale          10%
Breeding Service    4%
Other               8%
```

Useful visualizations:

```text
Pie Chart
Bar Chart
Monthly Trend
```

---

# 82. Income Trend

Example:

```text
May      ৳120k
June     ৳135k
July     ৳150k
August   ৳193k
```

This can show:

```text
Revenue growth
Seasonality
Sales performance
```

---

# 83. Milk Revenue Analysis

Because Income can reference:

```text
MilkRecord
```

the system can analyze:

```text
Milk produced
vs
Milk sold
vs
Milk sale revenue
```

This is valuable because production and revenue remain separate.

---

# 84. Milk Production vs Milk Revenue

Example:

```text
Production:
1,000 Liter

Sold:
800 Liter

Average sale price:
৳80/Liter

Revenue:
৳64,000
```

The remaining:

```text
200 Liter
```

may have another business use or remain unsold.

The Income model should not assume all production becomes revenue.

---

# 85. Animal Sale Revenue

The system can analyze:

```text
Animals Sold:
5

Total Revenue:
৳900,000

Average Sale Price:
৳180,000
```

This can be linked through:

```text
referenceType = Animal
```

---

# 86. Animal Sale + Purchase Price

Animal has:

```text
purchasePrice
```

and Income records:

```text
sale amount
```

Therefore, the system can derive:

```text
Sale Price
-
Purchase Price
=
Gross price difference
```

However, this is **not automatically true profit** because the animal may have incurred:

```text
Feed cost
Medicine cost
Vaccination cost
Labor cost
Other operating costs
```

So do not label this simple difference as final profit.

---

# 87. Breeding Service Revenue

Example:

```text
10 breeding services
× ৳2,000
=
৳20,000
```

Income records can reference the corresponding Breeding records.

This can support:

```text
Breeding service revenue report
```

---

# 88. Income + Expense Dashboard

A finance dashboard can show:

```text
Total Income
Total Expense
Net Profit/Loss
Pending Income
Pending Expense
Top Income Source
Top Expense Category
```

These are calculated metrics, not extra DB fields.

---

# 89. Income Reconciliation

Periodically compare:

```text
Milk Sale Income
vs
Milk sale records

Animal Sale Income
vs
Animals marked Sold

Breeding Service Income
vs
Breeding service records
```

This can identify:

```text
missing income
duplicate income
incorrect reference
```

---

# 90. Animal Sale Reconciliation

Example:

```text
Animals marked Sold:
10

Animal Sale Income records:
9
```

Potential issue:

```text
1 sold animal has no income record
```

This should be flagged for review.

However, not every sold animal necessarily means a recorded sale in every business scenario, so this should be a validation/reporting rule rather than an unconditional database constraint.

---

# 91. Milk Sale Reconciliation

Example:

```text
Milk Sale records:
20

Income records linked to MilkRecord:
19
```

Possible:

```text
1 sale missing financial record
```

Again, this should be treated as a reconciliation warning rather than assuming every MilkRecord is sold.

---

# 92. Duplicate Income Prevention

For reference-based income:

```text
referenceType
+
referenceId
```

can help prevent duplicate automatic income records.

Example:

```text
MilkRecord M-001
```

should not accidentally generate:

```text
INC-001
INC-002
INC-003
```

for the same sale event.

But a single MilkRecord could theoretically be associated with multiple sales in a more advanced partial-sale workflow, so uniqueness should be applied only where the business workflow guarantees one-to-one linkage.

---

# 93. Partial Milk Sale Consideration

Suppose:

```text
MilkRecord = 100 Liter
```

and the farm sells:

```text
60 Liter
```

then later:

```text
40 Liter
```

If the current business workflow allows this, blindly enforcing:

```text
One MilkRecord = One Income
```

would be wrong.

The current Income model supports:

```text
quantity
```

but does not include a dedicated:

```text
remainingQuantity
```

field.

Therefore, if partial sales are required, the service layer must track the sold quantity through Income records and prevent:

```text
total sold > produced
```

---

# 94. Partial Animal Sale

Usually:

```text
Animal
→ one sale
```

because an animal is a single unit.

Therefore:

```text
quantity = 1
unit = Piece
```

is the natural representation.

---

# 95. Income Quantity Validation

For quantity-based categories:

```text
quantity > 0
```

should be required.

For example:

```text
Milk Sale
quantity = 0
```

should normally be rejected.

---

# 96. Income Unit Validation

Recommended combinations:

```text
Milk Sale
→ Liter

Animal Sale
→ Piece

Calf Sale
→ Piece

Manure Sale
→ Kg

Breeding Service
→ Service
```

This prevents meaningless combinations.

---

# 97. Amount Validation

For quantity-based sales:

```text
amount
=
quantity × unitPrice
```

Example:

```text
500 kg manure
× ৳5/kg
=
৳2,500
```

Backend should calculate/validate the amount.

---

# 98. Service Income Validation

For:

```text
Breeding Service
```

use:

```text
unit = Service
```

and:

```text
quantity = 1
```

unless the business explicitly records multiple service units.

---

# 99. Insurance Claim Validation

Insurance claims may not have:

```text
quantity
unit
unitPrice
```

Therefore:

```text
amount
```

is the key required financial field.

The supporting claim document can be stored in:

```text
attachment
```

---

# 100. Government Subsidy Validation

Similarly:

```text
amount
```

is the primary financial value.

Supporting documentation:

```text
attachment
```

can contain the subsidy document.

---

# 101. Income Service Layer

Recommended methods:

```text
incomeService.createIncome()
incomeService.updateIncome()
incomeService.markReceived()
incomeService.cancelIncome()
incomeService.getIncomeById()
incomeService.getIncomeList()
incomeService.getMonthlySummary()
incomeService.getCategorySummary()
incomeService.getPendingIncome()
incomeService.getReceivedIncome()
incomeService.getIncomeReport()
incomeService.reconcileSales()
```

These are service responsibilities, not additional DB fields.

---

# 102. Milk Sale Service

Conceptually:

```text
createMilkSale({
    milkRecordId,
    quantity,
    unitPrice,
    customer,
    payment
})
```

Service should:

```text
1. Validate farm
2. Validate MilkRecord
3. Validate sale quantity
4. Check already-sold quantity
5. Calculate amount
6. Create Income
7. Update related sale tracking if implemented
8. Audit
9. Emit realtime event
```

---

# 103. Animal Sale Service

Conceptually:

```text
sellAnimal({
    animalId,
    salePrice,
    customer,
    payment
})
```

Service should:

```text
1. Validate farm
2. Validate animal
3. Ensure animal is sale-eligible
4. Create Income
5. Update animal lifecycle
6. Audit
7. Emit realtime event
```

Critical steps should ideally be atomic.

---

# 104. Breeding Service Income

Conceptually:

```text
recordBreedingServiceIncome({
    breedingId,
    amount,
    customer
})
```

Service should:

```text
1. Validate breeding record
2. Validate farm
3. Create Income
4. Link reference
5. Audit
```

---

# 105. Manual Income Workflow

For:

```text
Government Subsidy
Insurance Claim
Manure Sale
Other
```

workflow:

```text
User selects category
        ↓
Enter amount
        ↓
Enter date
        ↓
Payment details
        ↓
Customer/payer if applicable
        ↓
Attachment if available
        ↓
Status
        ↓
Create Income
        ↓
AuditLog
```

---

# 106. Mark Income as Received

Typical flow:

```text
Pending Income
      ↓
Customer pays
      ↓
User confirms receipt
      ↓
Validate payment method
      ↓
status = Received
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

# 107. Cancel Income

Possible flow:

```text
Income
   ↓
Authorized user
   ↓
Validate cancellation
   ↓
status = Cancelled
   ↓
AuditLog
   ↓
Dashboard/report recalculation
```

The finalized model has no dedicated cancellation-reason field, so the reason must be captured through the available description/audit workflow if needed.

---

# 108. Income + Notification

The finalized Notification model supports references to:

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

but does **not** include Income as a notification reference type. fileciteturn21file6

Therefore, do not claim that the current Notification schema directly references an Income record.

A general `Manual` notification or another application-level mechanism could be used if income alerts are required, but that is outside the current finalized reference enum.

---

# 109. Realtime Income Updates

The Income collection itself does not provide realtime communication.

For a realtime finance dashboard:

```text
Income Created
      ↓
Database commit
      ↓
Realtime Event
      ↓
Finance Dashboard
      ↓
Total Income updates instantly
```

Possible infrastructure:

```text
Socket.IO
WebSocket
SSE
```

The DB model remains independent of the transport layer.

---

# 110. Income + Report

The finalized Report model supports:

```text
reportType = Income
```

and formats:

```text
PDF
Excel
CSV
JSON
```

fileciteturn21file6

Therefore Income can directly feed:

```text
Income Report
Financial Report
Monthly Report
Profit/Loss Report
```

---

# 111. Income Report Date Range

The Report model supports:

```text
dateRange:
    from
    to
```

fileciteturn21file6

Therefore an Income report can preserve:

```text
From:
2026-08-01

To:
2026-08-31
```

as report metadata.

---

# 112. Income Search

Useful fields:

```text
incomeNo
incomeCategory
customerName
customerPhone
invoiceNumber
description
```

Example:

```text
Search:
Rahim
```

returns matching customer-related income records.

---

# 113. Income Filters

Recommended:

```text
incomeCategory
status
paymentMethod
incomeDate range
referenceType
customerName
```

Example:

```text
Category:
Milk Sale

Status:
Received

Month:
August 2026
```

---

# 114. Income Sorting

Recommended default:

```text
incomeDate DESC
```

so recent income appears first.

Secondary:

```text
createdAt DESC
```

can resolve ties.

---

# 115. Pagination

Long-term financial history should use:

```text
pagination
```

or:

```text
cursor-based loading
```

Example:

```text
20 income records per request
```

This is especially useful as the farm accumulates years of sales data.

---

# 116. Index Recommendations

Recommended:

```javascript
{
    farmId: 1,
    incomeDate: -1
}
```

Useful for:

```text
Recent income
Monthly reports
Date filtering
```

Category analysis:

```javascript
{
    farmId: 1,
    incomeCategory: 1,
    incomeDate: -1
}
```

Reference lookup:

```javascript
{
    farmId: 1,
    referenceType: 1,
    referenceId: 1
}
```

Status dashboard:

```javascript
{
    farmId: 1,
    status: 1,
    incomeDate: -1
}
```

---

# 117. Income Number Uniqueness

Recommended:

```text
incomeNo
```

should be unique according to the selected numbering strategy.

Possible:

```text
INC-000001
```

globally unique or farm-scoped.

The finalized model does not specify which numbering scope must be used.

---

# 118. Example — Milk Sale Income

```javascript
{
    incomeNo: "INC-000001",

    farmId: "FARM-001",

    incomeCategory: "Milk Sale",

    amount: 8000,

    incomeDate: "2026-08-20",

    paymentMethod: "Cash",

    referenceType: "MilkRecord",

    referenceId: "MILK-000125",

    customerName: "Rahim Dairy Farm",

    customerPhone: "017XXXXXXXX",

    invoiceNumber: "MILK-2026-0015",

    quantity: 100,

    unit: "Liter",

    unitPrice: 80,

    description: "Sale of 100 liters of milk",

    status: "Received",

    createdBy: "USER-001"
}
```

Values are illustrative.

---

# 119. Example — Animal Sale

```javascript
{
    incomeNo: "INC-000002",

    farmId: "FARM-001",

    incomeCategory: "Animal Sale",

    amount: 180000,

    incomeDate: "2026-08-21",

    paymentMethod: "Bank",

    referenceType: "Animal",

    referenceId: "ANIMAL-000015",

    customerName: "Karim Traders",

    customerPhone: "018XXXXXXXX",

    invoiceNumber: "SALE-2026-0021",

    quantity: 1,

    unit: "Piece",

    unitPrice: 180000,

    description: "Sale of animal A-015",

    status: "Received",

    createdBy: "USER-001"
}
```

---

# 120. Example — Manure Sale

```javascript
{
    incomeNo: "INC-000003",

    farmId: "FARM-001",

    incomeCategory: "Manure Sale",

    amount: 2500,

    incomeDate: "2026-08-22",

    paymentMethod: "Cash",

    referenceType: "Manual",

    customerName: "Local Buyer",

    customerPhone: "019XXXXXXXX",

    quantity: 500,

    unit: "Kg",

    unitPrice: 5,

    description: "Sale of 500 kg farm manure",

    status: "Received",

    createdBy: "USER-002"
}
```

---

# 121. Example — Breeding Service

```javascript
{
    incomeNo: "INC-000004",

    farmId: "FARM-001",

    incomeCategory: "Breeding Service",

    amount: 2000,

    incomeDate: "2026-08-23",

    paymentMethod: "Mobile Banking",

    referenceType: "Breeding",

    referenceId: "BREED-000010",

    customerName: "XYZ Farm",

    customerPhone: "016XXXXXXXX",

    quantity: 1,

    unit: "Service",

    unitPrice: 2000,

    description: "Breeding service",

    status: "Received",

    createdBy: "USER-001"
}
```

---

# 122. Example — Government Subsidy

```javascript
{
    incomeNo: "INC-000005",

    farmId: "FARM-001",

    incomeCategory: "Government Subsidy",

    amount: 10000,

    incomeDate: "2026-08-25",

    paymentMethod: "Bank",

    referenceType: "Manual",

    description: "Livestock development subsidy",

    attachment: "/uploads/subsidy/subsidy-2026.pdf",

    status: "Received",

    createdBy: "USER-001"
}
```

---

# 123. Example — Pending Milk Sale

```javascript
{
    incomeNo: "INC-000006",

    farmId: "FARM-001",

    incomeCategory: "Milk Sale",

    amount: 12000,

    incomeDate: "2026-08-26",

    paymentMethod: "Bank",

    referenceType: "MilkRecord",

    referenceId: "MILK-000140",

    customerName: "ABC Dairy",

    quantity: 150,

    unit: "Liter",

    unitPrice: 80,

    status: "Pending",

    createdBy: "USER-001"
}
```

---

# 124. Error Scenario — Wrong Farm Reference

```text
Income.farmId = FARM-A

MilkRecord.farmId = FARM-B
```

Result:

```text
Reject
```

Cross-farm references must never be allowed.

---

# 125. Error Scenario — Invalid Reference

```text
referenceType = MilkRecord
referenceId = nonexistent
```

Result:

```text
Reject
```

---

# 126. Error Scenario — Wrong Reference Type

```text
referenceType = Animal
referenceId = MilkRecord._id
```

Result:

```text
Reject
```

---

# 127. Error Scenario — Negative Income

```text
amount = -5000
```

Result:

```text
Reject
```

---

# 128. Error Scenario — Wrong Unit

```text
incomeCategory = Animal Sale
unit = Liter
```

Result:

```text
Reject
```

under the recommended category/unit validation.

---

# 129. Error Scenario — Zero Quantity

For quantity-based income:

```text
quantity = 0
```

Result:

```text
Reject
```

unless the specific income category does not require quantity.

---

# 130. Error Scenario — Wrong Amount

```text
quantity = 100
unitPrice = 80
amount = 5000
```

Expected:

```text
100 × 80 = 8000
```

Result:

```text
Reject or recalculate
```

Recommended:

```text
Backend calculates amount.
```

---

# 131. Error Scenario — Duplicate Animal Sale

If:

```text
Animal A-001
```

already has a confirmed sale Income and the business workflow guarantees one sale per animal:

```text
Create another sale
→ Reject
```

This protects against double selling/double revenue.

---

# 132. Animal Sale Eligibility

Before recording:

```text
Animal Sale
```

the service should verify the animal's current lifecycle.

For example, an already:

```text
Sold
Dead
Archived
```

animal should normally not be sold again.

The exact eligibility policy follows the finalized Animal lifecycle.

---

# 133. Income Creation Authorization

Only authorized farm members should create financial income records.

A reasonable policy:

```text
Worker:
Data entry where permitted

Farm Manager:
Create/manage normal income

Farm Owner:
Full financial management

Admin:
Full system access
```

The SRS defines these broad role responsibilities. fileciteturn21file11

---

# 134. Financial Security

Income contains sensitive financial information:

```text
Amount
Customer
Phone
Invoice
Payment method
Attachments
```

Therefore access should enforce:

```text
Authentication
Farm authorization
Role-based access
Audit logging
```

---

# 135. Income + Realtime Dashboard

After a successful Income creation:

```text
Database commit
      ↓
Income saved
      ↓
Emit:
income.created
      ↓
Dashboard
      ↓
Total Income updates
      ↓
Profit/Loss updates
```

The realtime mechanism is an application/infrastructure concern, not a MongoDB field.

---

# 136. Finance Dashboard Example

```text
This Month

Income:
৳193,000

Expense:
৳120,000

Net:
৳73,000

Pending Income:
৳12,000

Pending Expense:
৳8,000
```

Whether Net uses only Received/Paid or includes Pending depends on the selected finance policy.

---

# 137. Profit Is Not the Same as Animal Sale Margin

Example:

```text
Animal Purchase:
৳120,000

Animal Sale:
৳180,000
```

Difference:

```text
৳60,000
```

But actual profitability may be lower because:

```text
Feed
Medicine
Vaccination
Labor
Other costs
```

must be considered.

Therefore, Liventra should not automatically label:

```text
salePrice - purchasePrice
```

as final profit.

---

# 138. Income + Expense + Profit

The finance layer should conceptually calculate:

```text
Total Income
      -
Total Expense
      =
Net Profit/Loss
```

The exact inclusion policy for:

```text
Pending
```

and:

```text
Received/Paid
```

must remain consistent.

---

# 139. Income + Report Generation

The Report module can generate:

```text
Income Report
Finance Report
```

in:

```text
PDF
Excel
CSV
JSON
```

because the finalized Report model supports these report types/formats. fileciteturn21file6

---

# 140. Current Schema Limitation — No Dedicated Customer Model

The Income model stores:

```text
customerName
customerPhone
```

but does not contain:

```text
customerId
```

Therefore, Liventra currently does not have a structured Customer master relationship in this model.

This means customer history is stored as transaction-level snapshots rather than a normalized customer entity.

Do not claim a full Customer Management module from the current schema.

---

# 141. Current Schema Limitation — No Sale Transaction Collection

There is no separate:

```text
Sale
```

collection in the finalized core collections.

Therefore:

```text
Income
```

acts as the financial sale record for supported sale categories.

Operational source records such as:

```text
MilkRecord
Animal
Breeding
```

provide the underlying business context.

---

# 142. Current Schema Limitation — No Dedicated Manure Record

There is no finalized:

```text
ManureSale
```

collection.

Therefore:

```text
Manure Sale
→ referenceType = Manual
```

under the current model.

---

# 143. Current Schema Limitation — No Insurance Claim Collection

There is no finalized:

```text
InsuranceClaim
```

collection.

Therefore:

```text
Insurance Claim
→ referenceType = Manual
```

under the current model.

---

# 144. Current Schema Limitation — No Subsidy Collection

There is no finalized:

```text
GovernmentSubsidy
```

collection.

Therefore:

```text
Government Subsidy
→ referenceType = Manual
```

under the current model.

---

# 145. Current Schema Limitation — No Receivable Collection

`Pending` Income can represent outstanding money, but there is no separate:

```text
Receivable
```

collection.

Therefore, receivable reporting must currently be derived from:

```text
Income.status = Pending
```

---

# 146. Current Schema Limitation — No Payment History

The current Income model stores:

```text
paymentMethod
status
```

but does not store a separate payment-history array.

Therefore, if a customer makes multiple partial payments against one sale, the current model does not natively represent a detailed payment schedule.

A more advanced receivables/payment system would need additional modeling.

---

# 147. Current Schema Limitation — No Partial Payment Field

There is no:

```text
paidAmount
remainingAmount
```

field.

Therefore, partial payment handling is not fully represented by the finalized schema.

If partial payments are required, this should be considered a future schema enhancement rather than silently implemented as a hidden rule.

---

# 148. Current Schema Limitation — No Explicit Sale Date Separate From Income Date

For the current model:

```text
incomeDate
```

is the financial event date.

If a sale occurs on one date and payment is received on another, the model uses:

```text
incomeDate
status
```

but does not have:

```text
saleDate
receivedDate
```

as separate fields.

This is acceptable for a simple finance workflow but limits advanced accounting analysis.

---

# 149. Current Schema Limitation — No Approval Fields

Income does not contain:

```text
approvedBy
approvalDate
approvalStatus
```

Therefore, do not claim a dedicated approval workflow from the current schema.

The finalized AuditLog supports `Approve` and `Reject` actions generally, but that does not automatically create an Income approval state. fileciteturn21file7

---

# 150. Recommended API Endpoints

Conceptually:

```text
POST   /api/income
GET    /api/income
GET    /api/income/:id
PATCH  /api/income/:id
PATCH  /api/income/:id/receive
PATCH  /api/income/:id/cancel

GET    /api/income/summary
GET    /api/income/monthly
GET    /api/income/category-summary
GET    /api/income/pending
GET    /api/income/report
```

These are application-level recommendations.

---

# 151. Category-Specific APIs

Optional service-specific endpoints:

```text
POST /api/income/milk-sale
POST /api/income/animal-sale
POST /api/income/breeding-service
POST /api/income/manual
```

This can make validation clearer than using one completely generic endpoint.

---

# 152. Generic Income Service

A central internal service can conceptually be:

```text
createIncome({
    farmId,
    incomeCategory,
    amount,
    incomeDate,
    paymentMethod,
    referenceType,
    referenceId,
    customerName,
    customerPhone,
    quantity,
    unit,
    unitPrice,
    ...
})
```

Then category-specific validation can determine:

```text
which fields are required
which references are valid
which unit is allowed
```

---

# 153. Income Transaction Atomicity

For business events that modify multiple collections:

### Animal Sale

```text
Income
+
Animal
```

### Milk Sale

Potentially:

```text
Income
+
sale tracking
```

### Breeding Service

```text
Income
+
Breeding context
```

Where multiple critical state changes are performed together, MongoDB transactions should be considered.

---

# 154. Realtime + Audit Architecture

For Income:

```text
Business Event
      ↓
Income Service
      ↓
MongoDB Transaction
 ┌────┼──────────────┐
 ↓    ↓              ↓
Income  Operational  AuditLog
        record
      ↓
Commit
      ↓
Realtime event
      ↓
Finance Dashboard
```

This gives:

```text
Current financial state
+
Historical business linkage
+
Audit trail
+
Realtime visibility
```

---

# 155. Final Business Rules Checklist

```text
[✓] Income represents money earned/expected by a farm
[✓] Expense represents money spent
[✓] Income and Expense are separate financial flows
[✓] Income must be farm-scoped
[✓] Cross-farm access must be rejected
[✓] incomeNo should be human-readable and unique by chosen scope
[✓] incomeCategory uses finalized enum
[✓] amount should normally be positive
[✓] incomeDate is the business event date
[✓] createdAt is document creation time
[✓] paymentMethod uses finalized enum
[✓] referenceType uses finalized enum
[✓] referenceType/referenceId must be validated together
[✓] Milk Sale can reference MilkRecord
[✓] Animal Sale can reference Animal
[✓] Calf Sale can reference Animal
[✓] Breeding Service can reference Breeding
[✓] Manure Sale uses Manual reference under current schema
[✓] Insurance Claim uses Manual reference under current schema
[✓] Government Subsidy uses Manual reference under current schema
[✓] Other normally uses Manual reference
[✓] customerName stores buyer/payer snapshot
[✓] customerPhone stores customer contact snapshot
[✓] invoiceNumber stores invoice/sale reference
[✓] quantity is used for quantity-based income
[✓] unit uses finalized enum
[✓] unitPrice represents price per unit
[✓] amount should be validated against quantity × unitPrice where applicable
[✓] service income can use unit = Service
[✓] non-quantity income may omit quantity/unit/unitPrice
[✓] description stores additional context
[✓] attachment stores supporting document reference
[✓] status uses Pending/Received/Cancelled
[✓] Pending means outstanding under the operational policy
[✓] Received means money has been received
[✓] Cancelled should normally be excluded from active income totals
[✓] createdBy identifies creator
[✓] updatedBy identifies last modifier
[✓] financial changes should be audited
[✓] Milk production is not automatically the same as milk sale
[✓] Animal sale should update animal lifecycle through sale workflow
[✓] Animal sale and Income should be synchronized
[✓] Duplicate sale income should be prevented where one-to-one business rules apply
[✓] Partial milk sales require service-level tracking
[✓] Current schema has no dedicated Customer collection
[✓] Current schema has no dedicated Sale collection
[✓] Current schema has no dedicated Manure/Insurance/Subsidy collections
[✓] Current schema has no dedicated Receivable collection
[✓] Current schema has no partial-payment fields
[✓] Current schema has no approval fields
[✓] Profit/Loss is derived from Income and Expense
[✓] The project must choose a consistent Pending-vs-Received finance policy
[✓] Realtime delivery belongs to application infrastructure
[✓] Income can feed Income and Finance reports
```

---

# 156. Final Model Summary

```text
                     Income
                        │
       ┌────────────────┼─────────────────┐
       │                │                 │
       ▼                ▼                 ▼
   Milk Sale       Animal Sale      Breeding Service
       │                │                 │
       ▼                ▼                 ▼
 MilkRecord           Animal           Breeding
       │                │                 │
       └────────────────┼─────────────────┘
                        ▼
                 Financial History
                        │
                        ▼
                  Profit / Loss
```

The main responsibility of `Income` is:

> **Maintain a traceable financial record of money earned or expected by each farm, while linking supported income events to their operational source records and providing the foundation for revenue analysis, financial reporting, and Profit/Loss calculation.**

---

# 157. Final Status

**Model:** `Income`  
**Status:** ✅ Finalized  
**Business Logic:** ✅ Documented  
**Finance Integration:** ✅  
**Milk Sale Integration:** ✅  
**Animal Sale Integration:** ✅  
**Breeding Service Integration:** ✅  
**Customer Information:** ✅  
**Payment Tracking:** ✅  
**Invoice Support:** ✅  
**Attachment Support:** ✅  
**Auditability:** ✅  
**Profit/Loss Integration:** ✅  
**Reporting:** ✅  
**Realtime Architecture:** ✅  
**Historical Integrity:** ✅  
**Current Schema Limitations:** ✅ Documented

**Next Model:** `Notification`
