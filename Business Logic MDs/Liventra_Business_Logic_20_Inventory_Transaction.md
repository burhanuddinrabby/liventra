# Liventra — Business Logic Documentation
## Model 20: `Inventory_Transaction`

**Document Version:** Business Logic v1.0  
**Project:** Liventra — Smart Livestock Farm Management System  
**Database:** MongoDB  
**Architecture:** MERN  
**Module:** Inventory & Stock Movement

---

# 1. Source / Finalized Model

The latest finalized Liventra DB Model defines `Inventory_Transaction` as:

```text
Inventory_Transaction Model:
_id
transactionNo
farmId
inventoryId
transactionType:
    Purchase
    Consumption
    Vaccination Usage
    Treatment Usage
    Damage
    Expired
    Adjustment
    Return

transactionDate
quantity
unit:
    kg | g | liter | ml | piece | box | packet | bottle

unitPrice
totalAmount

referenceType:
    FeedRecord
    Treatment
    Vaccination
    Expense
    Manual
    Purchase

referenceId
supplierName
remarks
performedBy
createdAt
updatedAt
```

This exact field structure and enum set are present in the latest finalized DB model document. fileciteturn19file1

The `Inventory` model stores current stock, while this model is the historical record of stock movement. fileciteturn19file1

> **Documentation rule:** The schema above is treated as finalized. The business rules below explain how these fields should behave; they do not silently add new database fields.

---

# 2. Purpose

`Inventory_Transaction` records **every meaningful movement or adjustment of inventory stock**.

It answers:

> "When did stock change, what happened, how much changed, at what price, why did it happen, which farm/item was affected, and who performed the operation?"

Example:

```text
Inventory:
Napier Grass

Transaction:
Purchase

Quantity:
100 kg

Unit Price:
৳40/kg

Total:
৳4,000

Date:
2026-08-20
```

The corresponding Inventory record then reflects the resulting current stock.

---

# 3. Inventory vs Inventory_Transaction

This distinction is one of the most important design decisions in Liventra.

## Inventory

Stores:

```text
CURRENT STATE
```

Example:

```text
currentStock = 250 kg
```

## Inventory_Transaction

Stores:

```text
HISTORY OF HOW STOCK CHANGED
```

Example:

```text
Purchase      +100 kg
Consumption    -20 kg
Damage          -5 kg
Adjustment      -3 kg
```

Therefore:

```text
Inventory
    ↓
"What do we have now?"

Inventory_Transaction
    ↓
"How did we get here?"
```

---

# 4. Why Transaction History Is Necessary

Suppose:

```text
Current Stock = 100 kg
```

Without transactions, the system cannot reliably explain:

```text
Why is it 100 kg?
```

With transactions:

```text
Purchase       +150 kg
Consumption     -30 kg
Damage          -10 kg
Adjustment      -10 kg
--------------------------------
Current         100 kg
```

Now Liventra has:

```text
Current stock
+
Historical traceability
+
Operational accountability
```

---

# 5. Core Architecture

```text
                    Inventory
                currentStock = X
                       ▲
                       │
              Stock movement service
                       │
                       ▼
             Inventory_Transaction
                       │
       ┌───────────────┼────────────────┐
       │               │                │
       ▼               ▼                ▼
    Purchase       Consumption       Adjustment
       │               │                │
       ▼               ▼                ▼
    Expense       Feed/Treatment     Physical count
                    /Vaccination
```

---

# 6. The Most Important Business Rule

## Never change `Inventory.currentStock` without a corresponding transaction.

Bad:

```text
Inventory.currentStock -= 10
```

with no history.

Correct:

```text
Stock movement
      ↓
Create Inventory_Transaction
      ↓
Update Inventory.currentStock
```

For critical operations, both should happen atomically.

---

# 7. Field-by-Field Details

---

# 7.1 `_id`

### Type

MongoDB `ObjectId`

### Required

Yes

### Purpose

Unique database identifier of the transaction.

### Example

```text
ObjectId(...)
```

It is used internally for relationships such as:

```text
Expense.referenceId
```

when:

```text
Expense.referenceType = InventoryTransaction
```

The finalized Expense model explicitly supports `InventoryTransaction` as a reference type. fileciteturn19file1

---

# 8. `transactionNo`

### Type

String

### Required

Yes

### Example

```text
ITX-000001
```

### Purpose

Human-readable transaction identifier.

Useful for:

```text
Inventory history
Purchase records
Reports
Audit
Search
Expense references
```

---

# 9. Transaction Number Generation

Recommended format:

```text
ITX-{SEQUENCE}
```

Examples:

```text
ITX-000001
ITX-000002
ITX-000003
```

The backend should generate this value.

The frontend should not be trusted to create unique transaction numbers.

---

# 10. `farmId`

### Type

ObjectId / Farm reference

### Required

Yes

### Purpose

Identifies the farm where the inventory transaction occurred.

Because Liventra supports multiple farms, every transaction must be farm-scoped.

---

# 11. Farm Isolation Rule

Before creating a transaction:

```text
Transaction.farmId
        ↓
must match
        ↓
Inventory.farmId
```

If:

```text
Transaction.farmId = Farm A
Inventory.farmId = Farm B
```

the operation must be rejected.

The SRS requires that users cannot access another farm's data. fileciteturn19file8

---

# 12. `inventoryId`

### Type

ObjectId / Inventory reference

### Required

Yes

### Purpose

Identifies which Inventory item was affected.

Relationship:

```text
Inventory_Transaction.inventoryId
                ↓
          Inventory._id
```

This is the central relationship of this collection.

---

# 13. Inventory Validation

Before creating a transaction:

```text
Inventory exists
+
Inventory belongs to farmId
+
Inventory is eligible for the transaction
```

must be verified.

Do not allow a transaction against a nonexistent or cross-farm inventory item.

---

# 14. `transactionType`

### Type

Enum

### Required

Yes

### Finalized Values

```text
Purchase
Consumption
Vaccination Usage
Treatment Usage
Damage
Expired
Adjustment
Return
```

These values come directly from the finalized model. fileciteturn19file1

---

# 15. Transaction Type: `Purchase`

Meaning:

```text
Stock enters the farm inventory because the farm purchased it.
```

Normal stock effect:

```text
currentStock ↑
```

Example:

```text
100 kg purchased
```

Stock movement:

```text
+100 kg
```

Typical related records:

```text
Inventory_Transaction
+
Expense
```

---

# 16. Purchase Business Flow

```text
Purchase
   ↓
Validate Inventory
   ↓
Validate quantity/unit/price
   ↓
Create Inventory_Transaction
   ↓
Increase Inventory.currentStock
   ↓
Update averagePurchasePrice
   ↓
Create/link Expense
   ↓
Recalculate Inventory.status
   ↓
AuditLog
```

The Expense model supports `InventoryTransaction` through `referenceType` and `referenceId`. fileciteturn19file1

---

# 17. Transaction Type: `Consumption`

Meaning:

```text
Inventory was used during normal farm operation.
```

Normal stock effect:

```text
currentStock ↓
```

Example:

```text
12 kg feed consumed
```

Stock:

```text
250 kg → 238 kg
```

---

# 18. Consumption Sources

Consumption can originate from operational modules.

Examples:

```text
FeedRecord
Treatment
General manual usage
```

The finalized `referenceType` supports:

```text
FeedRecord
Treatment
Manual
```

among others. fileciteturn19file1

---

# 19. Transaction Type: `Vaccination Usage`

Meaning:

```text
Vaccine stock was consumed during vaccination.
```

Normal stock effect:

```text
currentStock ↓
```

Relationship:

```text
Vaccination
      ↓
Vaccine
      ↓
Inventory
      ↓
Inventory_Transaction
```

The finalized `referenceType` explicitly supports:

```text
Vaccination
```

fileciteturn19file1

---

# 20. Vaccination Usage Example

Suppose:

```text
FMD Vaccine
Stock = 20 bottles
```

A vaccination uses:

```text
1 bottle
```

Transaction:

```text
transactionType = Vaccination Usage
quantity = 1
unit = bottle
referenceType = Vaccination
referenceId = VAC-00015
```

Inventory becomes:

```text
19 bottles
```

---

# 21. Transaction Type: `Treatment Usage`

Meaning:

```text
Medicine stock was consumed during animal treatment.
```

Normal stock effect:

```text
currentStock ↓
```

Relationship:

```text
Treatment
     ↓
Medicine
     ↓
Inventory
     ↓
Inventory_Transaction
```

The finalized `referenceType` supports:

```text
Treatment
```

fileciteturn19file1

---

# 22. Treatment Usage Example

Initial:

```text
Oxytetracycline
20 bottles
```

Treatment uses:

```text
2 bottles
```

Transaction:

```text
transactionType = Treatment Usage
quantity = 2
unit = bottle
referenceType = Treatment
referenceId = TR-00020
```

New stock:

```text
18 bottles
```

---

# 23. Transaction Type: `Damage`

Meaning:

```text
Some inventory was physically damaged and is no longer usable.
```

Normal stock effect:

```text
currentStock ↓
```

Example:

```text
100 bottles
3 damaged
```

Result:

```text
97 bottles
```

Transaction:

```text
transactionType = Damage
quantity = 3
```

---

# 24. Damage Remarks

For a damage transaction, `remarks` should ideally explain the reason.

Examples:

```text
Bottle broken during transport.
```

```text
Feed spoiled due to water leakage.
```

This makes stock reconciliation easier.

---

# 25. Transaction Type: `Expired`

Meaning:

```text
Inventory became unusable because it expired.
```

Normal stock effect:

```text
usable stock ↓
```

Example:

```text
10 vaccine bottles expired
```

Transaction:

```text
transactionType = Expired
quantity = 10
```

The Inventory item should then reflect the appropriate expired state according to its current stock/expiry business rules.

---

# 26. Important Expiry Limitation

The current Inventory model has:

```text
expiryDate
```

but no:

```text
batchNo
lotNo
batchQuantity
```

Therefore, transaction-level expiry handling cannot provide true batch-level FEFO/FIFO tracking.

Do not claim that the current schema supports complete batch-level expiry management.

---

# 27. Transaction Type: `Adjustment`

Meaning:

```text
The system stock is corrected to match an authorized physical count or data correction.
```

Example:

```text
System:
100 kg

Physical:
96 kg
```

Adjustment:

```text
-4 kg
```

---

# 28. Adjustment Is Different From Damage

Do not confuse:

```text
Damage
```

with:

```text
Adjustment
```

### Damage

Known operational reason:

```text
5 bottles were broken.
```

### Adjustment

System correction:

```text
System says 100
Physical count says 96
Difference = -4
```

The reason should be recorded in `remarks`.

---

# 29. Transaction Type: `Return`

Meaning:

```text
Inventory movement associated with returned stock.
```

The finalized model includes `Return`, but does not further specify the exact operational direction of every return scenario. fileciteturn19file1

Therefore, the application must explicitly define whether a particular Return means:

```text
stock returned TO the farm
```

or:

```text
stock returned TO supplier
```

before implementing automatic quantity changes.

Do not leave this ambiguous in the service layer.

---

# 30. `transactionDate`

### Type

Date

### Required

Yes

### Purpose

Records when the stock movement actually occurred.

Example:

```text
2026-08-20
```

---

# 31. Transaction Date vs CreatedAt

These are conceptually different.

```text
transactionDate
=
When the business event happened

createdAt
=
When the database record was created
```

Example:

```text
Purchase happened:
Aug 18

User entered it:
Aug 20
```

Then:

```text
transactionDate = Aug 18
createdAt = Aug 20
```

This distinction is important for historical reporting.

---

# 32. Historical Date Integrity

Reports such as:

```text
Monthly Purchase Report
Monthly Consumption Report
```

should generally use:

```text
transactionDate
```

rather than:

```text
createdAt
```

because business reports should reflect when the stock movement happened.

---

# 33. `quantity`

### Type

Number

### Required

Yes

### Purpose

Stores the amount of inventory affected.

Examples:

```text
100
12
3
```

The meaning depends on:

```text
unit
```

---

# 34. Quantity Examples

```text
quantity = 100
unit = kg
```

means:

```text
100 kg
```

```text
quantity = 5
unit = bottle
```

means:

```text
5 bottles
```

---

# 35. Quantity Must Be Positive

Recommended rule:

```text
quantity > 0
```

The direction of movement should come from:

```text
transactionType
```

rather than storing negative quantities.

For example:

```text
Purchase +100
Consumption -100
```

should preferably be represented as:

```text
Purchase
quantity = 100

Consumption
quantity = 100
```

and the service layer determines the stock direction.

This keeps quantity semantics consistent.

---

# 36. `unit`

### Type

Enum

### Required

Yes

### Allowed Values

```text
kg
g
liter
ml
piece
box
packet
bottle
```

These are the finalized values. fileciteturn19file1

---

# 37. Unit Must Match Inventory

If:

```text
Inventory.unit = kg
```

then a normal transaction should use:

```text
unit = kg
```

unless a formal unit conversion rule exists.

Example of invalid operation:

```text
Inventory:
500 kg

Transaction:
20 liter
```

without a defined conversion.

---

# 38. Why Unit Consistency Matters

Otherwise:

```text
currentStock = 500 kg
```

could incorrectly be changed using:

```text
20 liter
```

and the system would produce meaningless stock calculations.

Therefore, unit validation should happen before stock modification.

---

# 39. `unitPrice`

### Type

Number

### Required

Conditional

### Purpose

Stores the price per unit for the transaction.

Most important for:

```text
Purchase
```

Example:

```text
quantity = 100 kg
unitPrice = ৳40
```

---

# 40. Unit Price Example

```text
100 kg × ৳40/kg
=
৳4,000
```

So:

```text
unitPrice = 40
```

and:

```text
totalAmount = 4000
```

---

# 41. When `unitPrice` Is Relevant

Most directly:

```text
Purchase
```

It may also be useful for other financially meaningful stock transactions.

For pure operational events such as:

```text
Damage
Consumption
Adjustment
```

it may be:

```text
null
```

or calculated/recorded only if the application needs cost valuation.

Do not invent a mandatory price for every transaction type if it has no business meaning.

---

# 42. `totalAmount`

### Type

Number

### Required

Conditional

### Purpose

Represents the total monetary value associated with the transaction.

For purchases:

```text
totalAmount
=
quantity × unitPrice
```

Example:

```text
100 kg × ৳40
=
৳4,000
```

---

# 43. Total Amount Validation

For a purchase:

```text
totalAmount = quantity × unitPrice
```

The backend should calculate or verify this.

Do not trust a frontend-supplied total blindly.

---

# 44. Floating-Point / Money Rule

For financial calculations:

```text
৳42.50
```

should be handled carefully.

The backend should avoid accidental floating-point inconsistencies when calculating:

```text
unitPrice
totalAmount
averagePurchasePrice
```

The exact implementation can use appropriate decimal handling.

---

# 45. `referenceType`

### Type

Enum

### Required

Conditional

### Allowed Values

```text
FeedRecord
Treatment
Vaccination
Expense
Manual
Purchase
```

These are the finalized values. fileciteturn19file1

---

# 46. Why `referenceType` Exists

It tells the system:

> "What business event caused this inventory transaction?"

Examples:

```text
FeedRecord
→ feed was given

Treatment
→ medicine was used

Vaccination
→ vaccine was used

Purchase
→ stock was purchased

Manual
→ manually recorded movement
```

---

# 47. `referenceId`

### Type

ObjectId

### Required

Conditional

### Purpose

Stores the ID of the related business record.

Examples:

```text
referenceType = FeedRecord
referenceId = FeedRecord._id
```

or:

```text
referenceType = Treatment
referenceId = Treatment._id
```

---

# 48. Reference Integrity

If:

```text
referenceType = Treatment
```

then:

```text
referenceId
```

must point to a valid Treatment record.

If:

```text
referenceType = Vaccination
```

then:

```text
referenceId
```

must point to a valid Vaccination record.

The backend must validate the pair.

---

# 49. Reference + Transaction Type Mapping

Recommended consistency:

| Transaction Type | Normal Reference Type |
|---|---|
| Purchase | Purchase |
| Consumption | FeedRecord / Manual |
| Vaccination Usage | Vaccination |
| Treatment Usage | Treatment |
| Damage | Manual |
| Expired | Manual |
| Adjustment | Manual |
| Return | Purchase / Manual |

The exact mapping for `Return` should be finalized in the service layer because the schema itself does not specify the direction of return.

---

# 50. `supplierName`

### Type

String

### Required

Optional

### Example

```text
ABC Agro Feed Supplier
```

### Purpose

Stores supplier information associated with the transaction.

Most useful for:

```text
Purchase
Return
```

---

# 51. Why Supplier Is Stored in Transaction

Inventory has:

```text
supplierName
```

but supplier information can change over time.

Example:

```text
2026 purchase:
Supplier A

2027 purchase:
Supplier B
```

Transaction-level supplier data preserves historical information.

The finalized model explicitly includes `supplierName` in the transaction. fileciteturn19file1

---

# 52. `remarks`

### Type

String

### Required

Optional

### Purpose

Stores additional transaction context.

Examples:

```text
Monthly feed purchase.
```

```text
Three vaccine bottles damaged during transport.
```

```text
Physical stock reconciliation.
```

```text
Returned unused medicine.
```

---

# 53. `performedBy`

### Type

ObjectId / User reference

### Required

Yes

### Purpose

Identifies the user who performed or recorded the stock movement.

This is important for:

```text
Accountability
Audit
Security
Farm management
```

---

# 54. `createdAt`

### Type

Date

### Required

Automatic

### Purpose

Records when the transaction document was created.

---

# 55. `updatedAt`

### Type

Date

### Required

Automatic

### Purpose

Records when the transaction document was last updated.

---

# 56. Transaction Creation Rules

Before creating a transaction:

```text
1. Authenticate user
2. Verify farm membership/permission
3. Load Inventory
4. Verify farmId
5. Validate transactionType
6. Validate quantity
7. Validate unit
8. Validate reference if provided
9. Validate price if required
10. Determine stock direction
11. Update Inventory atomically
12. Create transaction record
13. Trigger related financial/notification logic
14. Create AuditLog
```

---

# 57. Purchase Transaction — Complete Flow

```text
User purchases stock
        ↓
Select Inventory item
        ↓
Enter quantity + unit price
        ↓
Validate supplier
        ↓
Calculate totalAmount
        ↓
Create Inventory_Transaction
        ↓
Increase currentStock
        ↓
Update averagePurchasePrice
        ↓
Create Expense
        ↓
Recalculate status
        ↓
AuditLog
```

---

# 58. Purchase Example

Initial:

```text
Inventory:
Napier Grass

Stock:
100 kg

Average Purchase Price:
৳40/kg
```

New purchase:

```text
50 kg
৳45/kg
```

Transaction:

```text
transactionType = Purchase
quantity = 50
unit = kg
unitPrice = 45
totalAmount = 2250
referenceType = Purchase
```

New stock:

```text
150 kg
```

---

# 59. Weighted Average Price

If:

```text
Existing:
100 kg × ৳40 = ৳4,000

New:
50 kg × ৳45 = ৳2,250
```

Then:

```text
Total quantity = 150 kg
Total value = ৳6,250

Average =
6250 / 150
= ৳41.67/kg
```

Inventory's:

```text
averagePurchasePrice
```

can then be updated to the new weighted average.

---

# 60. Consumption Transaction — Complete Flow

```text
Feed/Treatment/Vaccination/Manual Usage
                ↓
          Determine quantity
                ↓
        Check available stock
                ↓
        Validate not expired
                ↓
       Atomically decrease stock
                ↓
      Create Inventory_Transaction
                ↓
        Recalculate status
                ↓
             AuditLog
```

---

# 61. Important Ordering Rule

For stock-consuming operations:

```text
Check stock
+
Reserve/decrease atomically
+
Create transaction
```

should be handled as one protected operation.

Otherwise, two users may consume the same stock simultaneously.

---

# 62. Concurrency Example

Suppose:

```text
Stock = 10
```

Two workers simultaneously request:

```text
Worker A = 7
Worker B = 5
```

Without atomic logic:

```text
Both read 10
```

and both may succeed incorrectly.

With an atomic conditional update:

```text
A consumes 7
Stock = 3

B requests 5
Stock < 5
→ Reject
```

This protects inventory integrity.

---

# 63. Negative Stock Prevention

Normal rule:

```text
currentStock >= requested quantity
```

must be true before consumption.

If:

```text
currentStock = 3
quantity = 5
```

return:

```text
Insufficient stock
```

Do not create the consumption transaction.

---

# 64. Vaccination Usage Workflow

```text
Vaccination record
        ↓
vaccineId
        ↓
Find corresponding Inventory
        ↓
Validate stock
        ↓
Consume quantity
        ↓
Create:
Vaccination Usage transaction
        ↓
Update stock
```

---

# 65. Treatment Usage Workflow

```text
Treatment
   ↓
medicineId
   ↓
Find Inventory
   ↓
Validate stock
   ↓
Consume medicine
   ↓
Treatment Usage transaction
   ↓
Update stock
```

The finalized Treatment model contains a nested medicines array with dosage and other treatment details. fileciteturn19file1

---

# 66. Feed Usage Workflow

```text
FeedRecord
   ↓
Actual quantity fed
   ↓
Identify corresponding feed inventory
   ↓
Consume stock
   ↓
Inventory_Transaction
transactionType = Consumption
referenceType = FeedRecord
   ↓
Update stock
```

Important:

```text
FeedRecommendation
```

should not directly consume inventory.

Actual feeding should be represented by:

```text
FeedRecord
```

and then stock consumption.

---

# 67. Recommendation vs Transaction

Correct:

```text
FeedRecommendation
→ recommends 12 kg

FeedRecord
→ records actual 10 kg fed

Inventory_Transaction
→ records 10 kg consumed
```

This preserves the difference between:

```text
Recommended
vs
Actually used
```

---

# 68. Damage Workflow

```text
Damage detected
       ↓
Enter quantity
       ↓
Create Damage transaction
       ↓
Decrease stock
       ↓
Update status
       ↓
Audit
```

---

# 69. Expiry Workflow

```text
Expired stock identified
        ↓
Create Expired transaction
        ↓
Decrease/removal from usable stock
        ↓
Update Inventory status
        ↓
Audit
```

Because the current Inventory schema is not batch-based, exact expired quantity management may require careful operational handling.

---

# 70. Adjustment Workflow

```text
Physical count
      ↓
Compare with system stock
      ↓
Difference calculated
      ↓
Adjustment transaction
      ↓
Update currentStock
      ↓
Record reason in remarks
      ↓
AuditLog
```

---

# 71. Adjustment Example

```text
System = 500 kg
Physical = 485 kg

Difference = 15 kg
```

Transaction:

```text
transactionType = Adjustment
quantity = 15
unit = kg
referenceType = Manual
remarks = "Monthly physical stock count"
```

Stock becomes:

```text
485 kg
```

---

# 72. Return Workflow

Because `Return` can represent different business directions, define the operation explicitly.

### If returned TO farm:

```text
Stock increases
```

### If returned TO supplier:

```text
Stock decreases
```

The transaction should record:

```text
supplierName
referenceType
referenceId
remarks
```

to preserve the context.

---

# 73. Expense Integration

The finalized Expense model supports:

```text
referenceType = InventoryTransaction
```

and:

```text
referenceId
```

fileciteturn19file1

Therefore, a purchase can be represented as:

```text
Inventory_Transaction
        ↓
Expense
```

Example:

```text
Inventory Transaction:
ITX-00015
৳4,000

Expense:
EXP-00022
referenceType = InventoryTransaction
referenceId = ITX-00015
amount = ৳4,000
```

---

# 74. Avoiding Double Counting

Do not treat:

```text
Inventory_Transaction.totalAmount
```

and:

```text
Expense.amount
```

as two separate expenses.

They represent the same financial event from two different perspectives:

```text
Inventory Transaction
→ operational stock movement

Expense
→ financial accounting record
```

---

# 75. Purchase + Expense Transaction Safety

Recommended:

```text
BEGIN TRANSACTION

Create Inventory_Transaction
Update Inventory
Create Expense

COMMIT
```

If any critical operation fails:

```text
ROLLBACK
```

This prevents:

```text
Stock increased
but Expense missing
```

or:

```text
Expense created
but stock not increased
```

---

# 76. Transaction History

Inventory detail page should show:

```text
Transaction No
Date
Type
Quantity
Unit
Unit Price
Total Amount
Reference
Supplier
Performed By
Remarks
```

Example:

```text
ITX-001
Aug 01
Purchase
100 kg
৳40
৳4,000
ABC Supplier
```

---

# 77. Stock Ledger

A transaction ledger can calculate running balance.

Example:

```text
Date      Type          Qty    Balance
Aug 01    Purchase      +100   100
Aug 03    Consumption    -20    80
Aug 05    Damage          -5    75
Aug 10    Purchase       +50   125
```

The stored transaction quantity remains positive; the display/service layer applies the appropriate direction.

---

# 78. Opening Stock

If the farm already has physical stock when Liventra is first introduced, do not create stock with no explanation.

Recommended:

```text
Inventory created
+
Adjustment transaction
referenceType = Manual
remarks = "Opening stock"
```

Example:

```text
Opening stock:
500 kg
```

This establishes the beginning of the stock ledger.

---

# 79. Why Opening Stock Matters

Otherwise:

```text
Current Stock = 500 kg
```

but transaction history starts from:

```text
Purchase +20
```

and reports become confusing.

With:

```text
Opening Adjustment +500
```

the ledger has a clear starting point.

---

# 80. Historical Backdated Transactions

Because `transactionDate` is separate from `createdAt`, the system can support authorized historical entry.

Example:

```text
Actual purchase:
Aug 10

Entered:
Aug 17
```

The record can retain:

```text
transactionDate = Aug 10
createdAt = Aug 17
```

This should be permission-controlled and auditable.

---

# 81. Editing Transactions

Transactions represent historical business events.

Therefore:

```text
Edit freely
```

is not recommended.

Especially avoid editing:

```text
quantity
transactionType
transactionDate
inventoryId
```

after downstream records depend on them.

---

# 82. Transaction Correction

If an incorrect transaction was recorded:

Preferred approach:

```text
Original transaction
        ↓
Authorized correction
        ↓
Reversal/Adjustment
        ↓
New correct transaction
        ↓
AuditLog
```

This preserves history.

The exact reversal mechanism is not a separate enum in the finalized model, so the implementation should use the existing supported transaction types rather than inventing a new enum value.

---

# 83. Hard Delete

Avoid hard deleting an Inventory_Transaction once it has affected:

```text
Inventory
Expense
Reports
```

because deletion destroys the historical explanation for the stock balance.

---

# 84. Transaction + AuditLog

Important transaction events should be audited:

```text
Transaction Created
Transaction Corrected
Transaction-linked stock changed
Manual Adjustment
Opening Stock
Purchase
Damage
Expiry
```

AuditLog can answer:

```text
Who did it?
When?
What changed?
```

Inventory_Transaction answers:

```text
Why did stock change?
How much?
```

Together they provide strong traceability.

---

# 85. Transaction vs AuditLog

Do not use AuditLog as a replacement for Inventory_Transaction.

### Inventory_Transaction

Business record:

```text
100 kg purchased
```

### AuditLog

Change history:

```text
User X created transaction ITX-001 at 10:35
```

They serve different purposes.

---

# 86. `performedBy` vs AuditLog User

`performedBy` answers:

```text
Who performed the stock operation?
```

AuditLog identifies:

```text
Who created/modified the database record
```

Often these are the same person, but they are conceptually separate.

---

# 87. Farm Permission

Before creating a transaction:

```text
Authenticated User
       ↓
FarmMember
       ↓
Role
       ↓
Permission
```

Only authorized farm members should be able to perform stock operations.

The project uses farm roles rather than requiring a separate Role model.

---

# 88. Read Permissions

Authorized users may view:

```text
Inventory transaction history
```

but the ability to:

```text
Create Purchase
Adjust Stock
Record Damage
```

can be restricted according to role.

The SRS describes:

```text
Admin
Farm Owner
Farm Manager
Worker
```

with different access levels. fileciteturn19file8

---

# 89. Worker vs Manager Example

A reasonable application policy:

```text
Worker:
Record normal operational consumption

Farm Manager:
Purchase
Adjustment
Damage approval
Inventory management

Farm Owner:
Full farm inventory management
```

This is a recommended permission design, not an additional database field.

---

# 90. Status Effects on Transactions

Before consumption:

```text
Inventory.status
```

should be checked.

Do not normally allow:

```text
Expired
Inactive
```

inventory to be consumed.

---

# 91. Low Stock After Transaction

After every stock-changing transaction:

```text
Update currentStock
       ↓
Recalculate status
```

Example:

```text
Before:
85 kg
reorderLevel = 80
status = Available

Consumption:
10 kg

After:
75 kg
status = Low Stock
```

---

# 92. Out-of-Stock After Transaction

Example:

```text
Before:
5 bottles

Consumption:
5 bottles

After:
0 bottles
```

Then:

```text
status = Out of Stock
```

and a notification may be created.

---

# 93. Notification Integration

The finalized Notification model supports:

```text
referenceType = Inventory
referenceId = Inventory._id
```

fileciteturn19file17

Therefore:

```text
Transaction
   ↓
Inventory status changes
   ↓
Low Stock / Out of Stock / Expiry
   ↓
Notification
```

---

# 94. Notification Spam Prevention

If stock remains below reorder level:

```text
Do not create duplicate notification on every transaction/page load.
```

A practical approach:

```text
Threshold crossed
   ↓
Create notification

Still below threshold
   ↓
No duplicate

Stock rises above threshold
   ↓
Reset threshold state

Falls below again
   ↓
New notification
```

---

# 95. Reporting Integration

The finalized Report model supports:

```text
reportType = Inventory
```

fileciteturn19file17

Therefore Inventory_Transaction data can power:

```text
Inventory reports
Purchase reports
Consumption reports
Damage reports
Expiry reports
Stock movement reports
```

---

# 96. Monthly Inventory Report

Possible structure:

```text
Opening Stock
+
Purchases
+
Returns In
-
Consumption
-
Vaccination Usage
-
Treatment Usage
-
Damage
-
Expired
± Adjustments
=
Closing Stock
```

The exact Return direction must follow the application's defined return workflow.

---

# 97. Purchase Report

Filter:

```text
transactionType = Purchase
```

Then calculate:

```text
Total Purchases
Total Quantity Purchased
Purchase Cost
Supplier-wise Purchase
Item-wise Purchase
```

---

# 98. Consumption Report

Possible filters:

```text
transactionType = Consumption
```

and:

```text
referenceType = FeedRecord
```

to analyze feed usage.

Similarly:

```text
Treatment Usage
Vaccination Usage
```

can be analyzed separately.

---

# 99. Feed Consumption Analytics

Example:

```text
Napier Grass
Total consumed this month:
1,250 kg

Average daily:
41.67 kg
```

This can be calculated from transactions referencing:

```text
FeedRecord
```

---

# 100. Medicine Consumption Analytics

Example:

```text
Oxytetracycline
Used:
25 bottles

Treatment-related cost:
৳X
```

The system can connect:

```text
Treatment
+
Inventory_Transaction
+
Expense
```

for health-cost analysis.

---

# 101. Vaccine Consumption Analytics

Example:

```text
FMD Vaccine
Used:
40 bottles

Vaccination events:
40
```

This can support:

```text
Vaccination campaign planning
Vaccine stock planning
```

---

# 102. Inventory Transaction + FeedRecommendation

The recommendation system may say:

```text
Napier Grass:
12 kg/day
```

But the transaction should only happen after actual usage.

Correct:

```text
FeedRecommendation
      ↓
Farmer sees recommendation
      ↓
FeedRecord
      ↓
Inventory_Transaction
```

This preserves historical recommendation snapshots.

---

# 103. Why This Is Important for Liventra

The project has an intelligent feed recommendation component.

Therefore:

```text
Recommended quantity
```

and:

```text
Actual quantity consumed
```

should remain separate.

This allows future analysis such as:

```text
Recommended:
12 kg/day

Actual:
10 kg/day

Difference:
-2 kg/day
```

---

# 104. Transaction Search

Inventory transaction history should support:

```text
transactionNo
inventoryId
transactionType
referenceType
referenceId
supplierName
transactionDate
performedBy
```

---

# 105. Date Filtering

Useful filters:

```text
Today
Last 7 days
Last 30 days
This month
Previous month
Custom date range
```

Use:

```text
transactionDate
```

for business-date filtering.

---

# 106. Index Recommendations

Useful MongoDB indexes:

```javascript
{
    farmId: 1,
    inventoryId: 1,
    transactionDate: -1
}
```

This supports:

```text
Inventory history
```

Another useful index:

```javascript
{
    farmId: 1,
    transactionType: 1,
    transactionDate: -1
}
```

for reports.

Another:

```javascript
{
    farmId: 1,
    referenceType: 1,
    referenceId: 1
}
```

for linked business records.

---

# 107. Unique Transaction Number

Recommended unique constraint:

```text
transactionNo
```

should be unique.

Because it is a farm-facing business identifier, duplicate numbers would make:

```text
reports
search
audit
```

confusing.

---

# 108. Transaction Number vs MongoDB `_id`

Do not use:

```text
_id
```

as the human-facing transaction number.

Use:

```text
_id
→ internal database identity

transactionNo
→ business-facing identity
```

---

# 109. API-Level Business Operations

Recommended service methods:

```text
inventoryTransactionService.createPurchase()
inventoryTransactionService.recordConsumption()
inventoryTransactionService.recordVaccinationUsage()
inventoryTransactionService.recordTreatmentUsage()
inventoryTransactionService.recordDamage()
inventoryTransactionService.recordExpiry()
inventoryTransactionService.recordAdjustment()
inventoryTransactionService.recordReturn()
inventoryTransactionService.getHistory()
inventoryTransactionService.getLedger()
inventoryTransactionService.getSummary()
```

These are service-layer operations, not extra DB fields.

---

# 110. Generic Transaction Service

A generic internal method can conceptually be:

```text
processStockMovement({
    inventoryId,
    transactionType,
    quantity,
    unit,
    referenceType,
    referenceId,
    ...
})
```

It should:

```text
1. Validate user
2. Validate farm
3. Validate inventory
4. Validate transaction type
5. Determine direction
6. Validate stock
7. Update inventory atomically
8. Create transaction
9. Recalculate status
10. Trigger notifications if needed
11. Trigger financial record if needed
12. Audit
```

---

# 111. Purchase + Average Price Logic

Only purchase-type transactions should normally affect:

```text
averagePurchasePrice
```

Example:

```text
Damage
```

should not change the historical purchase price.

Likewise:

```text
Consumption
```

should not redefine the average purchase price.

---

# 112. Purchase Price History

Because each purchase has:

```text
unitPrice
transactionDate
supplierName
```

the system can analyze historical price changes.

Example:

```text
Aug:
৳40/kg

Sep:
৳43/kg

Oct:
৳45/kg
```

This supports purchasing decisions.

---

# 113. Supplier-Wise Purchase Analysis

Using:

```text
supplierName
```

the system can calculate:

```text
Supplier A:
৳50,000

Supplier B:
৳32,000
```

and:

```text
quantity purchased
average unit price
```

---

# 114. Inventory Cost Analysis

The transaction history can support:

```text
Feed cost
Medicine cost
Vaccine cost
Damage cost
Expiry loss
```

For example:

```text
Purchase Cost:
৳100,000

Damage:
৳5,000 estimated value

Expiry:
৳2,000 estimated value
```

The exact loss valuation should be based on the application's chosen costing method.

---

# 115. Important Limitation — No Explicit `transactionDirection`

The finalized model does not contain:

```text
IN / OUT
```

or:

```text
quantityChange
```

Therefore, direction must be derived from:

```text
transactionType
```

This is acceptable, but the service layer must define the direction consistently.

---

# 116. Recommended Direction Map

```text
Purchase
→ IN

Consumption
→ OUT

Vaccination Usage
→ OUT

Treatment Usage
→ OUT

Damage
→ OUT

Expired
→ OUT

Adjustment
→ IN or OUT

Return
→ depends on defined return workflow
```

This mapping should be centralized in one service/helper rather than duplicated throughout controllers.

---

# 117. Adjustment Direction

For Adjustment:

```text
Physical > System
→ IN

Physical < System
→ OUT
```

Example:

```text
System = 100
Physical = 105

Adjustment:
+5
```

or:

```text
System = 100
Physical = 95

Adjustment:
-5
```

Even though the stored `quantity` can remain positive, the service must know the direction.

---

# 118. Important Limitation — Quantity Sign

The finalized schema only has:

```text
quantity
```

and not:

```text
quantityChange
```

Therefore, the implementation should not ambiguously store both:

```text
quantity = -5
```

and:

```text
transactionType = Damage
```

unless the project explicitly defines signed quantities.

A cleaner rule is:

```text
quantity > 0
```

and transaction type determines direction.

---

# 119. Data Consistency Rule

For every stock-changing transaction:

```text
Inventory.currentStock
```

must equal the result of applying valid transactions from the opening balance.

Conceptually:

```text
Closing Stock
=
Opening Stock
+
IN Transactions
-
OUT Transactions
```

This provides a powerful consistency check.

---

# 120. Reconciliation Algorithm

Periodically:

```text
Calculate ledger balance
        ↓
Compare with Inventory.currentStock
        ↓
If equal:
    OK

If different:
    Investigate
    ↓
    Authorized Adjustment
```

This can detect:

```text
manual data corruption
missing transaction
incorrect transaction
concurrency issue
```

---

# 121. Transaction Immutability Principle

Once a transaction is used for:

```text
stock
expense
reports
```

treat it as effectively immutable.

For correction:

```text
new corrective transaction
```

is preferable to silently rewriting history.

This is especially important for auditability.

---

# 122. Realtime Inventory Updates

The database model itself does not provide realtime communication.

Realtime behavior belongs to:

```text
WebSocket / Socket.IO / SSE
```

or another application-layer mechanism.

Possible flow:

```text
Inventory Transaction Created
        ↓
Inventory Updated
        ↓
Emit stock-update event
        ↓
Dashboard receives event
        ↓
Current stock changes instantly
```

This matches the project's broader requirement for modern smart management, while keeping realtime delivery separate from the MongoDB model.

---

# 123. Realtime Low-Stock Event

```text
Consumption
    ↓
Stock = 75
    ↓
Reorder Level = 80
    ↓
Status = Low Stock
    ↓
Create Notification
    ↓
Emit realtime event
    ↓
UI updates
```

The Notification model supports Inventory references. fileciteturn19file17

---

# 124. Example — Full Purchase Transaction

```javascript
{
    transactionNo: "ITX-000001",

    farmId: "FARM-001",

    inventoryId: "INV-000001",

    transactionType: "Purchase",

    transactionDate: "2026-08-20",

    quantity: 100,

    unit: "kg",

    unitPrice: 40,

    totalAmount: 4000,

    referenceType: "Purchase",

    referenceId: "PUR-000001",

    supplierName: "ABC Agro Supplier",

    remarks: "Monthly feed purchase",

    performedBy: "USER-001"
}
```

Values are illustrative.

---

# 125. Example — Feed Consumption Transaction

```javascript
{
    transactionNo: "ITX-000002",

    farmId: "FARM-001",

    inventoryId: "INV-000001",

    transactionType: "Consumption",

    transactionDate: "2026-08-21",

    quantity: 12,

    unit: "kg",

    referenceType: "FeedRecord",

    referenceId: "FEED-000125",

    remarks: "Daily feed consumption",

    performedBy: "USER-002"
}
```

---

# 126. Example — Vaccination Usage

```javascript
{
    transactionNo: "ITX-000003",

    farmId: "FARM-001",

    inventoryId: "INV-000020",

    transactionType: "Vaccination Usage",

    transactionDate: "2026-08-21",

    quantity: 1,

    unit: "bottle",

    referenceType: "Vaccination",

    referenceId: "VAC-000015",

    remarks: "FMD vaccination",

    performedBy: "USER-002"
}
```

---

# 127. Example — Treatment Usage

```javascript
{
    transactionNo: "ITX-000004",

    farmId: "FARM-001",

    inventoryId: "INV-000030",

    transactionType: "Treatment Usage",

    transactionDate: "2026-08-21",

    quantity: 2,

    unit: "bottle",

    referenceType: "Treatment",

    referenceId: "TR-000025",

    remarks: "Medicine used for treatment",

    performedBy: "USER-002"
}
```

---

# 128. Example — Damage

```javascript
{
    transactionNo: "ITX-000005",

    farmId: "FARM-001",

    inventoryId: "INV-000020",

    transactionType: "Damage",

    transactionDate: "2026-08-22",

    quantity: 2,

    unit: "bottle",

    referenceType: "Manual",

    remarks: "Two bottles damaged during transport",

    performedBy: "USER-003"
}
```

---

# 129. Example — Adjustment

```javascript
{
    transactionNo: "ITX-000006",

    farmId: "FARM-001",

    inventoryId: "INV-000001",

    transactionType: "Adjustment",

    transactionDate: "2026-08-31",

    quantity: 4,

    unit: "kg",

    referenceType: "Manual",

    remarks: "Monthly physical stock count: system 100 kg, physical 96 kg",

    performedBy: "USER-001"
}
```

The actual direction is determined by the adjustment logic.

---

# 130. Example — Expired

```javascript
{
    transactionNo: "ITX-000007",

    farmId: "FARM-001",

    inventoryId: "INV-000020",

    transactionType: "Expired",

    transactionDate: "2026-09-01",

    quantity: 5,

    unit: "bottle",

    referenceType: "Manual",

    remarks: "Expired vaccine stock removed",

    performedBy: "USER-001"
}
```

---

# 131. Error Scenario — Wrong Farm

```text
Inventory.farmId = FARM-A
Transaction.farmId = FARM-B
```

Result:

```text
Reject
```

---

# 132. Error Scenario — Invalid Inventory

```text
inventoryId = nonexistent
```

Result:

```text
Reject
```

---

# 133. Error Scenario — Wrong Unit

```text
Inventory:
kg

Transaction:
liter
```

Result:

```text
Reject
```

unless a defined conversion exists.

---

# 134. Error Scenario — Insufficient Stock

```text
Current:
3 bottles

Requested:
5 bottles
```

Result:

```text
Reject
```

No transaction should be created.

---

# 135. Error Scenario — Expired Item Consumption

```text
Inventory.status = Expired
```

Attempt:

```text
Treatment Usage
```

Result:

```text
Reject
```

under the normal safety policy.

---

# 136. Error Scenario — Invalid Reference

```text
referenceType = Treatment
referenceId = Vaccination._id
```

Result:

```text
Reject
```

because the reference type and reference ID do not match.

---

# 137. Error Scenario — Purchase Without Valid Price

If the business rule requires purchase pricing:

```text
transactionType = Purchase
unitPrice = null
```

Result:

```text
Reject
```

or require authorized exception handling.

---

# 138. Error Scenario — Wrong Total Amount

```text
quantity = 100
unitPrice = 40
totalAmount = 3000
```

Result:

```text
Reject or recalculate
```

Recommended:

```text
backend calculates:
100 × 40 = 4000
```

---

# 139. Error Scenario — Zero Quantity

```text
quantity = 0
```

Result:

```text
Reject
```

A transaction should represent an actual movement.

---

# 140. Error Scenario — Negative Quantity

```text
quantity = -5
```

Recommended:

```text
Reject
```

Use transaction type/direction instead.

---

# 141. Transaction Creation Authorization

Only authorized farm members should create:

```text
Purchase
Adjustment
Damage
Expiry
Return
```

Operational workers may be allowed to create:

```text
Consumption
```

depending on the final permission matrix.

This is application-level business logic, not a new schema field.

---

# 142. Transaction Lifecycle

A transaction itself generally has no:

```text
Pending
Completed
Cancelled
```

status field in the finalized schema.

Therefore, the application should not pretend that `Inventory_Transaction` has a stored transaction status.

Instead:

```text
Create
→ Apply
→ Preserve
```

If a correction is needed:

```text
Corrective transaction
+
AuditLog
```

should be preferred.

---

# 143. Why No Transaction Status Is Fine

The model is designed as a historical ledger.

For example:

```text
Purchase happened
```

is an event.

It does not necessarily need:

```text
status = Completed
```

because the existence of the transaction means the event has been recorded/applied.

---

# 144. Inventory Transaction + Expense Status

Expense separately has:

```text
Pending
Paid
Cancelled
```

fileciteturn19file1

Therefore, financial payment status belongs to:

```text
Expense
```

not Inventory_Transaction.

This separation is important.

---

# 145. Purchase Event vs Payment

A purchase can mean:

```text
Stock received
```

while payment can be:

```text
Pending
```

Therefore:

```text
Inventory_Transaction
→ stock event

Expense
→ financial payment/accounting state
```

This allows the two concerns to remain separate.

---

# 146. Transaction + Expense Example

```text
Stock purchased:
100 kg

Inventory Transaction:
৳4,000

Expense:
৳4,000
status = Pending
```

Later:

```text
Expense.status = Paid
```

The stock transaction itself does not need a payment status.

---

# 147. Transaction + Report

The finalized Report model supports:

```text
Inventory
Expense
Finance
Audit Log
```

report types. fileciteturn19file17

Inventory_Transaction provides the raw operational history needed to generate these reports.

---

# 148. Transaction + Financial Analysis

Possible calculations:

```text
Total purchases
Total inventory consumption cost
Damage value
Expired stock value
Item-wise purchase cost
Supplier-wise purchase cost
Monthly stock movement
```

The exact costing method for non-purchase consumption should be defined by the project's accounting policy.

---

# 149. Current Schema Limitation — No Batch Number

The transaction model has:

```text
transactionNo
```

but no:

```text
batchNumber
lotNumber
```

Therefore, medicine/vaccine batch-level traceability is not directly represented.

Do not claim batch-level traceability from this model alone.

---

# 150. Current Schema Limitation — No Expiry Per Transaction

`expiryDate` is stored in Inventory, not Inventory_Transaction.

Therefore, a transaction cannot directly say:

```text
This exact quantity came from batch X
with expiry Y
```

under the current schema.

This is a future enhancement area.

---

# 151. Current Schema Limitation — No Purchase Invoice Field

The finalized Inventory_Transaction model does not contain:

```text
invoiceNumber
```

The Expense model does contain:

```text
invoiceNumber
```

fileciteturn19file1

Therefore, purchase invoice information should currently be associated with the Expense side rather than inventing an invoice field in Inventory_Transaction.

---

# 152. Current Schema Limitation — No Payment Method

Inventory_Transaction does not store:

```text
Cash
Bank
Mobile Banking
Cheque
Card
```

Payment method belongs to Expense in the finalized model. fileciteturn19file1

---

# 153. Current Schema Limitation — No Explicit `approvedBy`

The finalized transaction model contains:

```text
performedBy
```

but not:

```text
approvedBy
```

Therefore, approval workflows should not be claimed as a database feature unless handled externally.

---

# 154. Current Schema Limitation — No Explicit Reversal Type

The enum does not contain:

```text
Reversal
```

Therefore, corrections should use the existing transaction types and AuditLog rather than silently adding a new enum.

---

# 155. Recommended API Endpoints

Conceptually:

```text
POST   /api/inventory-transactions
GET    /api/inventory-transactions
GET    /api/inventory-transactions/:id
GET    /api/inventory/:inventoryId/transactions
POST   /api/inventory/:inventoryId/purchase
POST   /api/inventory/:inventoryId/consume
POST   /api/inventory/:inventoryId/adjust
POST   /api/inventory/:inventoryId/damage
POST   /api/inventory/:inventoryId/expire
POST   /api/inventory/:inventoryId/return
```

These are recommended API designs, not fields from the DB model.

---

# 156. Recommended Query Filters

Example:

```text
farmId
inventoryId
transactionType
referenceType
transactionDate range
supplierName
performedBy
```

Example:

```text
All Treatment Usage
for August 2026
```

can be queried using:

```text
transactionType = Treatment Usage
transactionDate = August 2026
```

---

# 157. Inventory Ledger API

A useful endpoint can return:

```text
Opening Balance
Transactions
Running Balance
Closing Balance
```

Example response concept:

```text
Opening:
100 kg

Transactions:
+50 Purchase
-10 Consumption
-5 Damage

Closing:
135 kg
```

---

# 158. Transaction Summary API

For a date range:

```text
Total Purchases
Total Consumption
Total Damage
Total Expired
Total Adjustments
Total Returns
```

This can feed the dashboard and reports.

---

# 159. Transaction Data Integrity Formula

For each Inventory item:

```text
Expected Current Stock
=
Opening Stock
+
Purchase
+
Positive Return
+
Positive Adjustment
-
Consumption
-
Vaccination Usage
-
Treatment Usage
-
Damage
-
Expired
-
Supplier Return
-
Negative Adjustment
```

The exact treatment of Return depends on the final return direction policy.

---

# 160. Reconciliation Rule

If:

```text
Expected Current Stock
≠
Inventory.currentStock
```

then:

```text
Inventory integrity issue
```

should be investigated.

Do not automatically overwrite the stock without an adjustment transaction.

---

# 161. Audit + Realtime + Transaction Architecture

For the user's requirement of change tracking and realtime visibility, the clean architecture is:

```text
Business Action
      ↓
Service Layer
      ↓
MongoDB Transaction
 ┌────┼───────────────┐
 │    │               │
 ▼    ▼               ▼
Inventory  Inventory_Transaction  AuditLog
 │
 ▼
Notification if needed
 │
 ▼
Realtime Event
 │
 ▼
Frontend Dashboard
```

This separates:

```text
Business history
Audit history
Current state
Realtime delivery
```

instead of trying to make one collection do everything.

---

# 162. Final Business Rules Checklist

```text
[✓] Inventory_Transaction records stock movement history
[✓] Inventory stores current stock state
[✓] transactionNo should be unique
[✓] farmId is required
[✓] transaction must belong to the same farm as Inventory
[✓] inventoryId must reference a valid Inventory item
[✓] transactionType is controlled by finalized enum
[✓] transactionDate records the business event date
[✓] createdAt records database creation time
[✓] quantity should normally be positive
[✓] transactionType determines stock direction
[✓] unit must match Inventory.unit unless conversion exists
[✓] Purchase increases stock
[✓] Consumption decreases stock
[✓] Vaccination Usage decreases vaccine stock
[✓] Treatment Usage decreases medicine stock
[✓] Damage decreases stock
[✓] Expired removes unusable stock
[✓] Adjustment can increase or decrease stock
[✓] Return direction must be explicitly defined
[✓] unitPrice is especially relevant for purchases
[✓] totalAmount should be validated/calculated
[✓] Purchase total = quantity × unitPrice
[✓] referenceType explains the source business event
[✓] referenceId links the transaction to that event
[✓] referenceType/referenceId must be consistent
[✓] supplierName preserves supplier history
[✓] remarks provide operational context
[✓] performedBy provides operational accountability
[✓] normal stock changes should not bypass transactions
[✓] negative stock should normally be prevented
[✓] atomic stock updates are important for concurrency
[✓] purchase can create/link Expense
[✓] InventoryTransaction can be referenced by Expense
[✓] InventoryTransaction should not be double-counted as a separate Expense
[✓] low-stock status should be recalculated after stock changes
[✓] out-of-stock status should be recalculated after stock changes
[✓] expired/inactive stock should normally be protected from consumption
[✓] transaction history should support reports
[✓] transaction history should support stock ledger
[✓] transaction history should support reconciliation
[✓] historical transactions should not be casually hard-deleted
[✓] corrections should preserve history
[✓] AuditLog and Inventory_Transaction serve different purposes
[✓] realtime updates belong to application infrastructure, not the DB model
[✓] current schema does not provide batch-level traceability
[✓] current schema does not provide batch-specific expiry
[✓] current schema does not provide payment method
[✓] current schema does not provide invoiceNumber
[✓] current schema does not provide approval field
[✓] current schema does not provide explicit reversal enum
```

---

# 163. Final Model Summary

```text
Inventory
    │
    │ currentStock
    ▼
Inventory_Transaction
    │
    ├── Purchase
    ├── Consumption
    ├── Vaccination Usage
    ├── Treatment Usage
    ├── Damage
    ├── Expired
    ├── Adjustment
    └── Return
```

The model's main responsibility is:

> **Preserve a reliable, traceable history of every inventory stock movement while `Inventory` maintains the current stock state.**

---

# 164. Final Status

**Model:** `Inventory_Transaction`  
**Status:** ✅ Finalized  
**Business Logic:** ✅ Documented  
**Inventory Integration:** ✅  
**Purchase Flow:** ✅  
**Consumption Flow:** ✅  
**Treatment Integration:** ✅  
**Vaccination Integration:** ✅  
**Feed Integration:** ✅  
**Expense Integration:** ✅  
**Notification Integration:** ✅  
**Auditability:** ✅  
**Stock Reconciliation:** ✅  
**Concurrency Protection:** ✅  
**Reporting:** ✅  
**Realtime Architecture:** ✅  
**Historical Integrity:** ✅  

**Next Model:** `Expense`
