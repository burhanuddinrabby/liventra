# Liventra — Business Logic Documentation
## Model 19: Inventory

**Document Version:** Business Logic v1.0  
**Project:** Liventra — Smart Livestock Farm Management System  
**Database:** MongoDB  
**Architecture:** MERN  
**Module:** Inventory & Stock Management

---

# 1. Source / Finalized Model

The latest finalized Liventra DB model defines `Inventory` as:

```text
Inventory Model:
_id
inventoryCode
farmId
itemName
itemType: Feed | Medicine | Vaccine | Equipment | Cleaning Supply | Other
referenceId (FeedIngredientId/MedicineId etc.)
category: Roughage | Antibiotic | Injection | Cleaning
currentStock
unit: kg | g | liter | ml | piece | packet | box | bottle
minimumStock
maximumStock
reorderLevel
averagePurchasePrice
supplierName
storageLocation
expiryDate
status: Available | Low Stock | Out of Stock | Expired | Inactive
remarks
createdBy
updatedBy
createdAt
updatedAt
```

The finalized source also defines `Inventory_Transaction` separately, which is essential because Inventory stores the **current stock state**, while transactions preserve the **stock movement history**. fileciteturn17file7turn17file8

The SRS explicitly includes Inventory features such as Feed Stock, Medicine Stock, Purchase History, and Low Stock Alert. fileciteturn17file9

---

# 2. Purpose

The `Inventory` model represents a farm's **current stock item/master record**.

It answers:

> "What item does this farm keep in inventory, what type is it, how much is currently available, what are its stock thresholds, where is it stored, what supplier is associated with it, and is the item currently usable?"

Example:

```text
Inventory Code:
INV-000125

Item:
Napier Grass

Type:
Feed

Current Stock:
250 kg

Minimum Stock:
50 kg

Reorder Level:
80 kg

Status:
Available
```

---

# 3. Inventory vs Inventory_Transaction

This distinction is extremely important.

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
WHAT CAUSED THE STOCK TO CHANGE
```

Example:

```text
Purchase
+100 kg

Consumption
-20 kg

Damage
-5 kg
```

Therefore:

```text
Inventory
    ↓
Current stock

Inventory_Transaction
    ↓
Stock movement history
```

The finalized transaction model contains transaction types such as Purchase, Consumption, Vaccination Usage, Treatment Usage, Damage, Expired, Adjustment, and Return. fileciteturn17file7

---

# 4. Why Both Models Are Necessary

Suppose:

```text
Current Stock = 100 kg
```

If we only store `currentStock`, we cannot answer:

```text
Why did stock become 100 kg?
```

With transactions:

```text
Purchase       +150 kg
Consumption     -30 kg
Damage          -10 kg
Return          -10 kg

Final           100 kg
```

Now the system has:

```text
Current State
+
Historical Traceability
```

This is much stronger for reporting and audit.

---

# 5. Overall Inventory Architecture

```text
                    ┌──────────────┐
                    │   Inventory  │
                    │ CurrentStock │
                    └──────┬───────┘
                           │
                    Stock Movement
                           │
                           ▼
              ┌────────────────────────┐
              │ Inventory_Transaction  │
              └────────────┬───────────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
        ▼                  ▼                  ▼
     Purchase          Consumption          Damage
        │                  │                  │
        ▼                  ▼                  ▼
     Expense          Feed/Treatment      Adjustment
```

---

# 6. Field-by-Field Business Logic

# 6.1 `_id`

### Type

MongoDB `ObjectId`

### Required

Yes

### Purpose

Unique database identifier of the Inventory record.

Example:

```text
ObjectId(...)
```

---

# 7. `inventoryCode`

### Type

String

### Required

Yes

### Example

```text
INV-000001
```

### Purpose

Human-readable unique identifier for an inventory item.

Useful for:

```text
Search
Stock reports
Purchase records
Audit
Barcode/QR UI
```

---

# 8. Inventory Code Generation

Recommended:

```text
INV-{SEQUENCE}
```

Examples:

```text
INV-000001
INV-000002
INV-000003
```

The backend should generate the code and enforce uniqueness.

The frontend should not be trusted to generate the unique code.

---

# 9. `farmId`

### Type

ObjectId / Farm reference

### Required

Yes

### Purpose

Identifies the farm that owns the inventory item.

This is essential because Liventra supports multiple farms.

---

# 10. Farm Isolation

Every inventory query should be scoped by:

```text
farmId
```

Example:

```text
GET /api/inventory
```

should internally behave conceptually like:

```javascript
Inventory.find({
    farmId: authorizedFarmId
});
```

A user from Farm A must never see:

```text
Farm B inventory
```

The SRS explicitly requires prevention of access to another farm's data. fileciteturn17file1

---

# 11. `itemName`

### Type

String

### Required

Yes

### Example

```text
Napier Grass
```

Other examples:

```text
Concentrate Feed
Oxytetracycline
FMD Vaccine
Cleaning Solution
Water Bucket
```

### Purpose

Human-readable name of the inventory item.

---

# 12. `itemType`

### Type

Enum

### Allowed Values

```text
Feed
Medicine
Vaccine
Equipment
Cleaning Supply
Other
```

These are the finalized values. fileciteturn17file7turn17file8

---

# 13. Why `itemType` Is Important

`itemType` determines how the item is used.

Example:

```text
Feed
```

may be consumed through:

```text
FeedRecord
```

Medicine may be consumed through:

```text
Treatment
```

Vaccine may be consumed through:

```text
Vaccination
```

Equipment may normally be tracked as stock rather than consumed in the same way.

---

# 14. Item Type → Module Mapping

Recommended integration:

```text
Feed
   ↓
FeedRecord / FeedIngredient

Medicine
   ↓
Treatment

Vaccine
   ↓
Vaccination

Equipment
   ↓
Farm operations

Cleaning Supply
   ↓
General farm consumption
```

The exact stock deduction workflow should be controlled by the service layer.

---

# 15. `referenceId`

### Type

ObjectId

### Required

Conditional

### Purpose

Links the inventory item to its corresponding master record when one exists.

The finalized model describes it as:

```text
referenceId
(FeedIngredientId / MedicineId etc.)
```

fileciteturn17file7turn17file8

---

# 16. Why `referenceId` Exists

Suppose:

```text
itemType = Feed
itemName = Napier Grass
```

and the actual feed master is:

```text
FeedIngredient._id = FI-001
```

Then:

```text
Inventory.referenceId = FI-001
```

This connects:

```text
Inventory
   ↓
FeedIngredient
```

Similarly:

```text
Medicine Inventory
   ↓
Medicine._id
```

or:

```text
Vaccine Inventory
   ↓
Vaccine._id
```

---

# 17. Polymorphic Reference Rule

`referenceId` can refer to different master collections depending on:

```text
itemType
```

Conceptually:

```text
itemType = Feed
→ referenceId = FeedIngredient._id

itemType = Medicine
→ referenceId = Medicine._id

itemType = Vaccine
→ referenceId = Vaccine._id
```

This should be handled carefully in the backend because MongoDB cannot enforce a normal foreign key across different collections.

---

# 18. Reference Validation

When `referenceId` is provided:

```text
Load referenced master
        ↓
Check correct collection/type
        ↓
Check item belongs to same farm if applicable
        ↓
Accept
```

Do not allow:

```text
itemType = Medicine
referenceId = FeedIngredient._id
```

---

# 19. `category`

### Type

String / Enum

### Finalized examples

```text
Roughage
Antibiotic
Injection
Cleaning
```

The current finalized source does not define an exhaustive category enum beyond these listed examples. fileciteturn17file7

Therefore, do not silently assume these are the only possible categories unless the implementation explicitly decides to enforce them.

---

# 20. Category Purpose

`category` provides more detailed classification within `itemType`.

Example:

```text
itemType = Feed
category = Roughage
```

or:

```text
itemType = Medicine
category = Antibiotic
```

or:

```text
itemType = Medicine
category = Injection
```

---

# 21. `currentStock`

### Type

Number

### Required

Yes

### Example

```text
250
```

with:

```text
unit = kg
```

means:

```text
250 kg
```

### Purpose

Stores the current available quantity.

This is the main operational stock value.

---

# 22. Current Stock Rule

Conceptually:

```text
currentStock
=
previousStock
+
stockIn
-
stockOut
+
adjustment
```

Every change should ideally generate an `Inventory_Transaction`.

Do not let arbitrary modules directly modify stock without recording the corresponding transaction.

---

# 23. Stock Must Not Normally Become Negative

Example:

```text
Current Stock = 5 kg
Consumption = 8 kg
```

Normal operation:

```text
Reject
```

unless the system explicitly allows negative stock.

For Liventra, preventing negative stock is strongly recommended because it keeps inventory records consistent.

---

# 24. `unit`

### Type

Enum

### Allowed Values

```text
kg
g
liter
ml
piece
packet
box
bottle
```

These are the finalized values. fileciteturn17file7turn17file8

---

# 25. Unit Consistency

A stock quantity must always use the Inventory item's configured unit.

Example:

```text
currentStock = 500
unit = kg
```

means:

```text
500 kg
```

A transaction using:

```text
ml
```

for a:

```text
kg
```

inventory item should be rejected unless a valid unit-conversion policy exists.

---

# 26. Unit Conversion

Do not automatically assume:

```text
1 liter = 1 kg
```

because this depends on the material.

If conversion is needed, it should be explicitly defined by the relevant business rule.

For the current model, the safest approach is:

```text
Inventory.unit
=
Transaction.unit
```

for normal transactions.

---

# 27. `minimumStock`

### Type

Number

### Required

Yes

### Example

```text
50
```

### Purpose

Defines the minimum acceptable stock threshold.

Example:

```text
minimumStock = 50 kg
```

When stock goes below this threshold, the system can identify a low-stock condition.

---

# 28. `maximumStock`

### Type

Number

### Required

Optional / recommended

### Example

```text
500
```

### Purpose

Defines the desired upper stock capacity.

Useful for:

```text
Overstock prevention
Purchase planning
Storage planning
```

---

# 29. `reorderLevel`

### Type

Number

### Required

Yes / recommended

### Example

```text
80
```

### Purpose

Defines the stock level at which the system should recommend or trigger reordering.

Example:

```text
currentStock = 75
reorderLevel = 80
```

then:

```text
Reorder Alert
```

can be generated.

---

# 30. Minimum Stock vs Reorder Level

They are not necessarily the same.

Example:

```text
minimumStock = 50
reorderLevel = 80
```

Meaning:

```text
80 → start planning/reordering

50 → critical minimum threshold
```

This gives the farm time to purchase before stock reaches a critical level.

---

# 31. Maximum Stock

Example:

```text
maximumStock = 500
```

If:

```text
currentStock = 480
```

and a new purchase of:

```text
200
```

is attempted, the system can warn:

```text
Purchase exceeds maximum stock capacity.
```

Whether to block or merely warn should depend on farm policy.

---

# 32. `averagePurchasePrice`

### Type

Number

### Required

Optional / system-maintained

### Example

```text
42.50
```

with:

```text
unit = kg
```

means average purchase price:

```text
৳42.50 per kg
```

---

# 33. Why Average Purchase Price Exists

Purchase prices can change.

Example:

```text
Purchase 1:
100 kg × ৳40

Purchase 2:
100 kg × ৳45
```

Average price can be maintained as:

```text
(100×40 + 100×45) / 200
= ৳42.50/kg
```

This can support:

```text
Stock valuation
Feed cost analysis
Purchase planning
Financial reporting
```

---

# 34. Average Purchase Price Should Not Be Manually Changed Casually

Recommended:

```text
Purchase Transaction
        ↓
Recalculate weighted average
        ↓
Inventory.averagePurchasePrice
```

If an authorized manual correction occurs:

```text
AuditLog
```

should capture it.

---

# 35. `supplierName`

### Type

String

### Required

Optional

### Example

```text
ABC Agro Feed Supplier
```

### Purpose

Stores the supplier associated with the inventory item.

---

# 36. Supplier History Limitation

Because the current model stores:

```text
supplierName
```

as a string, Inventory itself does not represent a full supplier master.

Historical supplier information should primarily be preserved through:

```text
Inventory_Transaction.supplierName
```

where appropriate.

The transaction model explicitly contains `supplierName`. fileciteturn17file7

---

# 37. `storageLocation`

### Type

String

### Required

Optional

### Example

```text
Feed Store Room A
```

or:

```text
Medicine Cabinet 2
```

### Purpose

Identifies where the stock is physically stored.

---

# 38. Storage Location Business Logic

Useful for:

```text
Finding items
Stock management
Expiry checking
Warehouse organization
```

Example:

```text
FMD Vaccine
→ Medicine/Vaccine Refrigerator 01
```

---

# 39. `expiryDate`

### Type

Date

### Required

Conditional

### Purpose

Stores the expiry date of inventory.

Especially important for:

```text
Medicine
Vaccine
Certain feed/products
```

---

# 40. Expiry Logic

If:

```text
expiryDate < today
```

then:

```text
status = Expired
```

should be considered.

Expired stock should not normally be consumed.

---

# 41. Expiry Warning

Before expiry:

```text
expiryDate - today <= configured threshold
```

the system can generate:

```text
Expiry Warning
```

This is especially useful for:

```text
Vaccines
Medicines
```

---

# 42. Important Limitation — Single Expiry Date

The current Inventory model stores only:

```text
expiryDate
```

at the inventory-item level.

This can become problematic if the same item is purchased in multiple batches with different expiry dates.

Example:

```text
FMD Vaccine
Batch A → expires Jan
Batch B → expires Mar
```

A single inventory record cannot accurately represent both batch expiries.

This is a **current model limitation**.

The current finalized schema should not be silently changed here, but batch-level inventory would be a strong future enhancement.

---

# 43. `status`

### Type

Enum

### Allowed Values

```text
Available
Low Stock
Out of Stock
Expired
Inactive
```

These are the finalized values. fileciteturn17file7turn17file8

---

# 44. Status: Available

Meaning:

```text
Stock is usable and above the relevant threshold.
```

Example:

```text
currentStock = 200
reorderLevel = 80
```

→ `Available`

---

# 45. Status: Low Stock

Meaning:

```text
Stock has reached the low/reorder threshold.
```

Recommended logic:

```text
currentStock <= reorderLevel
AND
currentStock > 0
```

→ `Low Stock`

The exact threshold policy should be consistent.

---

# 46. Status: Out of Stock

Meaning:

```text
currentStock = 0
```

Normally:

```text
status = Out of Stock
```

unless the item is also expired/inactive and the application chooses a different display priority.

---

# 47. Status: Expired

Meaning:

```text
expiryDate < today
```

The item should not normally be used for consumption.

---

# 48. Status: Inactive

Meaning:

```text
The inventory item is no longer actively managed/usable.
```

Examples:

```text
Old discontinued item
Deleted/retired equipment record
No longer used supply
```

Inactive is different from:

```text
Out of Stock
```

because an inactive item may have:

```text
currentStock > 0
```

but is intentionally no longer active.

---

# 49. Status Should Be Derived Carefully

A good approach is to calculate operational status from:

```text
currentStock
+
reorderLevel
+
expiryDate
+
isActive/inactive decision
```

Example priority:

```text
Inactive
   ↓
Expired
   ↓
Out of Stock
   ↓
Low Stock
   ↓
Available
```

The exact priority should be defined in the backend so the UI does not invent conflicting statuses.

---

# 50. `remarks`

### Type

String

### Required

Optional

### Purpose

Additional information.

Examples:

```text
Store away from direct sunlight.
```

```text
Reserved for emergency use.
```

```text
Equipment requires maintenance before use.
```

---

# 51. `createdBy`

### Type

ObjectId / User reference

### Required

Yes

### Purpose

Identifies who created the Inventory record.

---

# 52. `updatedBy`

### Type

ObjectId / User reference

### Required

Conditional / automatic on update

### Purpose

Identifies who last modified the inventory record.

This is important for auditability.

---

# 53. `createdAt`

### Type

Date

### Required

Automatic

### Purpose

Stores creation time.

---

# 54. `updatedAt`

### Type

Date

### Required

Automatic

### Purpose

Stores last update time.

---

# 55. Inventory Creation Workflow

Recommended:

```text
Select Farm
      ↓
Enter Item Name
      ↓
Select Item Type
      ↓
Select/Link Reference Master
      ↓
Set Unit
      ↓
Set Stock Thresholds
      ↓
Set Supplier
      ↓
Set Storage Location
      ↓
Set Expiry if applicable
      ↓
Create Inventory
      ↓
Determine Initial Status
      ↓
AuditLog
```

---

# 56. Initial Stock

A critical business decision:

If an Inventory item is created with:

```text
currentStock > 0
```

the system should preferably create an initial:

```text
Inventory_Transaction
transactionType = Adjustment
```

or:

```text
Purchase
```

depending on the source of the stock.

This preserves the stock's origin.

Avoid creating arbitrary stock with no history.

---

# 57. Purchase Workflow

When the farm buys stock:

```text
Purchase
   ↓
Inventory_Transaction
   ↓
Increase currentStock
   ↓
Update averagePurchasePrice
   ↓
Create/Update Expense
   ↓
Update Inventory status
   ↓
AuditLog
```

---

# 58. Purchase Example

Existing:

```text
Current Stock = 100 kg
Average Price = ৳40/kg
```

Purchase:

```text
50 kg
Price = ৳45/kg
```

New stock:

```text
150 kg
```

Weighted average:

```text
(100×40 + 50×45) / 150
= ৳41.67/kg
```

---

# 59. Inventory Transaction Integration

The finalized transaction model contains:

```text
transactionType
transactionDate
quantity
unit
unitPrice
totalAmount
referenceType
referenceId
supplierName
remarks
performedBy
```

fileciteturn17file7

Therefore, every important stock movement can have a traceable transaction.

---

# 60. Consumption Workflow

When stock is consumed:

```text
Check currentStock
      ↓
Validate quantity
      ↓
Create Consumption Transaction
      ↓
Decrease currentStock
      ↓
Recalculate status
      ↓
AuditLog
```

Do not directly decrease stock without recording the transaction.

---

# 61. Feed Consumption

Example:

```text
Inventory:
Napier Grass
Current Stock = 250 kg
```

FeedRecord consumes:

```text
12 kg
```

Then:

```text
Inventory currentStock = 238 kg
```

and:

```text
Inventory_Transaction:
transactionType = Consumption
referenceType = FeedRecord
```

---

# 62. Treatment Usage

When medicine is used in Treatment:

```text
Treatment
   ↓
Medicine usage
   ↓
Inventory transaction
transactionType = Treatment Usage
referenceType = Treatment
   ↓
Decrease medicine stock
```

This creates a direct health-to-inventory connection.

---

# 63. Vaccination Usage

When vaccine is administered:

```text
Vaccination
   ↓
Vaccine stock usage
   ↓
Inventory_Transaction
transactionType = Vaccination Usage
referenceType = Vaccination
   ↓
Decrease vaccine stock
```

---

# 64. Damage

If stock is damaged:

```text
Inventory_Transaction
transactionType = Damage
```

Then:

```text
currentStock -= quantity
```

Example:

```text
100 bottles
5 damaged
→ 95 bottles
```

---

# 65. Expired Stock

When stock expires:

```text
Inventory_Transaction
transactionType = Expired
```

Then:

```text
currentStock -= expired quantity
```

and:

```text
status = Expired
```

However, because the current Inventory model has only one `expiryDate`, batch-specific expiry handling is limited.

---

# 66. Return

The finalized transaction enum includes:

```text
Return
```

A return may represent stock being returned to the supplier or otherwise returned into inventory, depending on the transaction workflow.

The service layer must clearly define direction:

```text
Stock increases
```

or:

```text
Stock decreases
```

for each return scenario.

Do not leave this ambiguous in implementation.

---

# 67. Adjustment

`Adjustment` is useful when:

```text
Physical stock count
≠
System stock
```

Example:

```text
System:
100 kg

Physical:
97 kg

Adjustment:
-3 kg
```

Create:

```text
Inventory_Transaction
transactionType = Adjustment
quantity = 3
```

with remarks explaining why.

---

# 68. Stock Reconciliation

Recommended periodic process:

```text
System Stock
      ↓
Physical Count
      ↓
Compare
      ↓
Difference
      ↓
Adjustment Transaction
      ↓
Updated Current Stock
```

This is useful for real farms where manual measurement may differ from digital records.

---

# 69. `referenceType`

The finalized Inventory_Transaction model supports:

```text
FeedRecord
Treatment
Vaccination
Expense
Manual
Purchase
```

fileciteturn17file7

This is important for knowing why a transaction occurred.

---

# 70. Reference Examples

### Feed

```text
referenceType = FeedRecord
referenceId = FR-001
```

### Treatment

```text
referenceType = Treatment
referenceId = TR-001
```

### Vaccination

```text
referenceType = Vaccination
referenceId = VAC-001
```

### Purchase

```text
referenceType = Purchase
```

---

# 71. Inventory + Expense

Inventory purchase and financial expense are related but not the same record.

Correct architecture:

```text
Purchase
   ↓
Inventory_Transaction
   ↓
Expense
```

Example:

```text
100 kg feed purchased
```

Inventory transaction:

```text
quantity = 100 kg
unitPrice = ৳40
totalAmount = ৳4000
```

Expense:

```text
expenseCategory = Feed Purchase
amount = ৳4000
referenceType = InventoryTransaction
referenceId = transaction._id
```

The finalized Expense model explicitly supports `InventoryTransaction` as a reference type. fileciteturn17file7

---

# 72. Avoiding Double Expense

Do not create:

```text
Inventory purchase = ৳4000
Expense = ৳4000
```

and then separately count both as expenses.

The relationship should be:

```text
One real purchase
   ↓
One inventory transaction
   ↓
One financial expense reference
```

The dashboard should count the Expense record as the financial transaction.

---

# 73. Inventory + FeedIngredient

FeedIngredient is a feed master.

Inventory represents actual stock.

Therefore:

```text
FeedIngredient
   ↓
defines ingredient
```

while:

```text
Inventory
   ↓
stores available physical quantity
```

Example:

```text
FeedIngredient:
Napier Grass

Inventory:
Napier Grass
250 kg
```

`referenceId` can connect the two.

---

# 74. Inventory + Medicine

Medicine master:

```text
Medicine
```

Inventory:

```text
Medicine stock
```

Relationship:

```text
Medicine._id
      ↓
Inventory.referenceId
```

This avoids duplicating the complete medicine definition inside Inventory.

---

# 75. Inventory + Vaccine

Similarly:

```text
Vaccine
   ↓
Inventory.referenceId
```

allows the stock item to reference the vaccine master.

This is especially useful because vaccines have domain information such as:

```text
dosage
administration route
minimum age
storage temperature
```

while Inventory focuses on:

```text
quantity
storage
expiry
stock thresholds
```

---

# 76. Inventory + Treatment

A Treatment may contain multiple medicines.

When a medicine is administered:

```text
Treatment
   ↓
Medicine usage
   ↓
Inventory
```

Stock should be deducted based on the actual quantity used according to the Treatment workflow.

The finalized Treatment model contains nested medicine details including dosage, dosage unit, frequency, duration, administration route, and instructions. fileciteturn17file7

---

# 77. Inventory + Vaccination

Vaccination records:

```text
animalId
vaccineId
date
nextDueDate
```

and Inventory tracks:

```text
vaccine stock
```

Therefore:

```text
Vaccination
   ↓
Vaccine
   ↓
Inventory
```

can support automatic stock deduction.

---

# 78. Low Stock Alert

When:

```text
currentStock <= reorderLevel
```

the system can create:

```text
Notification
```

with:

```text
referenceType = Inventory
referenceId = Inventory._id
```

The finalized Notification model explicitly supports Inventory as a reference type. fileciteturn17file14

---

# 79. Low Stock Alert Example

```text
Inventory:
Concentrate Feed

Current:
65 kg

Reorder:
80 kg
```

System:

```text
Low Stock
```

Notification:

```text
"Concentrate Feed stock is below reorder level."
```

---

# 80. Avoid Notification Spam

Do not create a new low-stock notification every time a page loads.

Recommended:

```text
Stock crosses threshold
        ↓
Create notification

Still below threshold
        ↓
Do not create duplicate notification
```

A new notification can be created after:

```text
stock returns above threshold
```

and later crosses the threshold again.

---

# 81. Out of Stock Notification

When:

```text
currentStock = 0
```

the system can generate a higher-priority alert:

```text
Out of Stock
```

This is particularly important for:

```text
Medicine
Vaccine
Feed
```

---

# 82. Expiry Notification

If:

```text
expiryDate
```

is approaching:

```text
Expiry Warning
```

can be generated.

If expired:

```text
status = Expired
```

and consumption should normally be blocked.

---

# 83. Expired Medicine Rule

Attempt:

```text
Treatment
→ consume expired medicine
```

should be rejected.

Correct:

```text
Medicine inventory
status = Expired
        ↓
Cannot consume
```

unless an authorized correction process is explicitly implemented.

---

# 84. Expired Vaccine Rule

Same principle:

```text
Expired vaccine
→ should not be used
```

The system should warn/block vaccine usage based on the expiry rule.

---

# 85. Equipment Inventory

For:

```text
itemType = Equipment
```

stock semantics may differ from consumables.

Example:

```text
Water Bucket
currentStock = 10 pieces
```

Consumption may not mean the same thing as feed/medicine consumption.

Therefore, equipment may require:

```text
Purchase
Return
Damage
Adjustment
```

more frequently than:

```text
Consumption
```

---

# 86. Cleaning Supply

Example:

```text
Disinfectant
itemType = Cleaning Supply
unit = liter
```

Usage can be:

```text
Consumption
```

and reduce stock.

---

# 87. Stock Status Calculation

A recommended service function:

```text
calculateInventoryStatus(item)
```

Conceptually:

```text
if inactive:
    Inactive

else if expired:
    Expired

else if currentStock <= 0:
    Out of Stock

else if currentStock <= reorderLevel:
    Low Stock

else:
    Available
```

This makes status behavior consistent across the system.

---

# 88. Current Stock Update Must Be Atomic

Two workers may consume stock at almost the same time.

Example:

```text
Current = 10

Worker A consumes 7
Worker B consumes 5
```

If updates are not atomic, both may read:

```text
10
```

and create an invalid final state.

Therefore, stock deduction should use:

```text
atomic database update
```

or a transaction/conditional update.

---

# 89. Safe Stock Deduction

Conceptually:

```javascript
updateOne(
  {
    _id: inventoryId,
    currentStock: { $gte: quantity }
  },
  {
    $inc: { currentStock: -quantity }
  }
)
```

If no document is updated:

```text
Insufficient stock
```

This prevents negative inventory under concurrent requests.

---

# 90. Transaction + Stock Update

For critical stock operations:

```text
BEGIN TRANSACTION

Update Inventory
Create Inventory_Transaction
Create/Update Expense if purchase
Create AuditLog

COMMIT
```

If a critical step fails:

```text
ROLLBACK
```

This avoids:

```text
Stock changed
but transaction missing
```

---

# 91. Inventory Audit

Important changes:

```text
Inventory Created
Stock Increased
Stock Decreased
Threshold Changed
Supplier Changed
Expiry Changed
Storage Location Changed
Status Changed
Manual Adjustment
Inventory Deactivated
```

should be auditable.

The project already includes an AuditLog model designed to preserve entity changes.

---

# 92. Audit Example

Before:

```text
currentStock = 100
```

After:

```text
currentStock = 80
```

Audit:

```text
entity = Inventory
entityId = INV-001
action = UPDATE
changedFields = ["currentStock"]
```

The actual reason should also be traceable through:

```text
Inventory_Transaction
```

---

# 93. Inventory History

Inventory detail page should show:

```text
Current Stock
Purchase History
Consumption History
Damage
Expired
Adjustment
Returns
```

Example:

```text
Date       Type          Qty
Aug 01     Purchase      +100
Aug 03     Consumption   -20
Aug 05     Damage        -5
Aug 10     Purchase      +50
```

---

# 94. Inventory Valuation

A simple estimated stock value:

```text
currentStock × averagePurchasePrice
```

Example:

```text
250 kg × ৳42.50
= ৳10,625
```

This can support inventory reports.

It should be treated as an estimate when average purchase price is used.

---

# 95. Inventory Report

The SRS explicitly requires:

```text
Inventory Report
```

and Inventory functionality includes:

```text
Feed Stock
Medicine Stock
Purchase History
Low Stock Alert
```

fileciteturn17file9

A report can include:

```text
Item
Type
Current Stock
Unit
Reorder Level
Status
Supplier
Estimated Value
```

---

# 96. Inventory Dashboard

Useful cards:

```text
Total Inventory Items
Low Stock Items
Out of Stock Items
Expired Items
Total Estimated Stock Value
Recent Purchases
Recent Consumption
```

---

# 97. Search and Filtering

Inventory should support:

```text
Search by itemName
Search by inventoryCode
Filter by itemType
Filter by category
Filter by status
Filter by supplier
Filter by storage location
```

Example:

```text
itemType = Medicine
status = Low Stock
```

shows medicines requiring attention.

---

# 98. Expiry Filter

Useful filters:

```text
Expired
Expiring in 7 days
Expiring in 30 days
```

This is particularly valuable for:

```text
Medicine
Vaccine
```

---

# 99. Stock Threshold Configuration

Example:

```text
minimumStock = 50
reorderLevel = 80
maximumStock = 500
```

Then:

```text
current > 80
→ Available

0 < current <= 80
→ Low Stock

current = 0
→ Out of Stock
```

`minimumStock` remains the critical minimum threshold.

---

# 100. Minimum vs Reorder vs Maximum

Think of them as:

```text
Maximum Stock
      ↑
      │
Ideal upper capacity
      │
Reorder Level
      │
Start purchasing
      │
Minimum Stock
      │
Critical minimum
      │
Zero
```

This gives the inventory system a practical decision-support layer.

---

# 101. Purchase Quantity Recommendation

The current model does not have a dedicated:

```text
recommendedPurchaseQuantity
```

field.

But it can be calculated:

```text
maximumStock - currentStock
```

Example:

```text
maximum = 500
current = 200

Suggested purchase:
300
```

This should be a calculated value, not stored unless a future requirement needs a snapshot.

---

# 102. Inventory + Feed Recommendation

FeedRecommendation can recommend:

```text
Napier Grass = 12 kg/day
Concentrate = 3 kg/day
```

Inventory can then answer:

```text
How much of those ingredients are currently available?
```

Conceptually:

```text
FeedRecommendation
       ↓
ingredientId
       ↓
Inventory.referenceId
       ↓
currentStock
```

This creates an important decision-support connection.

---

# 103. Recommendation vs Inventory

Do not make FeedRecommendation itself change inventory.

Correct:

```text
Recommendation
→ tells user what should be fed
```

Actual feeding:

```text
FeedRecord
→ records what was fed
```

Then:

```text
FeedRecord
→ Inventory consumption
```

This separation is important.

---

# 104. Feed Flow

```text
FeedIngredient
      ↓
FeedFormula
      ↓
FeedRecommendation
      ↓
Farmer confirms/feeds
      ↓
FeedRecord
      ↓
Inventory Consumption
      ↓
Inventory currentStock decreases
```

This connects the intelligent module with operational stock management.

---

# 105. Medicine Flow

```text
Medicine
   ↓
Inventory
   ↓
Treatment
   ↓
Medicine Used
   ↓
Inventory_Transaction
   ↓
Stock decreases
```

---

# 106. Vaccine Flow

```text
Vaccine
   ↓
Inventory
   ↓
Vaccination
   ↓
Vaccine Used
   ↓
Inventory_Transaction
   ↓
Stock decreases
```

---

# 107. Important Business Rule — No Direct Stock Mutation

Avoid:

```text
Treatment controller:
Inventory.currentStock -= 2
```

without a transaction.

Better:

```text
Treatment service
      ↓
Inventory service
      ↓
Create Inventory_Transaction
      ↓
Atomically update currentStock
```

This gives a single source of truth for stock movement.

---

# 108. Inventory Service Layer

Recommended service methods:

```text
inventoryService.createItem()
inventoryService.updateItem()
inventoryService.getStock()
inventoryService.increaseStock()
inventoryService.decreaseStock()
inventoryService.recordPurchase()
inventoryService.recordConsumption()
inventoryService.recordDamage()
inventoryService.recordExpiry()
inventoryService.adjustStock()
inventoryService.calculateStatus()
inventoryService.getLowStockItems()
inventoryService.getExpiringItems()
inventoryService.getTransactionHistory()
```

---

# 109. Purchase Service

Conceptually:

```text
recordPurchase()

1. Validate Inventory
2. Validate quantity
3. Validate unit
4. Calculate totalAmount
5. Create Inventory_Transaction
6. Increase currentStock
7. Recalculate weighted average price
8. Create Expense reference
9. Update status
10. AuditLog
```

---

# 110. Consumption Service

Conceptually:

```text
recordConsumption()

1. Validate inventory
2. Validate item not expired/inactive
3. Validate quantity
4. Atomically deduct stock
5. Create Inventory_Transaction
6. Recalculate status
7. Create AuditLog
```

---

# 111. Insufficient Stock

Example:

```text
Current:
3 bottles

Requested:
5 bottles
```

Result:

```text
Reject:
Insufficient stock
```

Do not allow:

```text
currentStock = -2
```

---

# 112. Inactive Inventory

If:

```text
status = Inactive
```

then normal:

```text
Purchase
Consumption
```

should be blocked unless an authorized administrative workflow permits it.

Historical transactions should remain accessible.

---

# 113. Expired Inventory

If:

```text
status = Expired
```

normal consumption should be blocked.

A disposal workflow can record:

```text
transactionType = Expired
```

or an appropriate adjustment/expiry transaction.

---

# 114. Stock Reservation

The current Inventory model does not have:

```text
reservedStock
```

Therefore, Liventra should not pretend to support formal stock reservation.

For example, FeedRecommendation can recommend an amount, but it does not reserve that quantity.

This is a current model limitation.

---

# 115. Batch-Level Inventory Limitation

The current model has:

```text
currentStock
expiryDate
averagePurchasePrice
```

but no:

```text
batchNo
lotNo
batchQuantity
```

Therefore, precise FIFO/FEFO inventory cannot be fully implemented at the batch level.

This is an important future enhancement, especially for medicines and vaccines.

---

# 116. FIFO / FEFO

The current schema is not sufficient for true:

```text
FIFO
```

or:

```text
FEFO
```

because individual stock batches are not represented.

Do not claim that Liventra currently provides batch-level FEFO.

A future inventory-batch model would be needed.

---

# 117. Supplier Limitation

The current Inventory model has:

```text
supplierName
```

and transaction also has:

```text
supplierName
```

but there is no dedicated Supplier master model in the finalized source.

Therefore:

```text
Supplier management
```

is not a separate domain entity in the current schema.

---

# 118. Equipment Limitation

The current Inventory model can store equipment as:

```text
itemType = Equipment
```

but it does not provide:

```text
equipmentSerialNumber
maintenanceSchedule
assetCondition
```

Therefore, detailed asset management is outside the current Inventory model.

---

# 119. Inventory and IoT Future Scope

The SRS identifies future IoT integration including:

```text
IoT / Bolus
RFID
QR Code
Smart Weighing Scale
Milk Analyzer
Automatic Feed Machine
```

fileciteturn17file9turn17file17

Inventory can later integrate with automatic feed equipment, but the current Inventory model should not assume IoT-driven stock movements until those future modules are implemented.

---

# 120. CRUD Rules

## Create

Required:

```text
inventoryCode
farmId
itemName
itemType
currentStock
unit
minimumStock
reorderLevel
```

plus any applicable reference information.

---

## Read

Authorized farm users can view:

```text
Current stock
Stock status
Item details
Transaction history
```

---

## Update

Authorized users can modify:

```text
itemName
category
thresholds
supplier
storageLocation
expiryDate
remarks
status
```

Direct manual modification of:

```text
currentStock
```

should be restricted.

---

## Delete

Avoid hard deletion when transactions already exist.

Prefer:

```text
status = Inactive
```

or controlled archival.

---

# 121. Why Current Stock Should Be Protected

If a user can simply edit:

```text
currentStock = 500
```

without a transaction, then:

```text
stock history becomes unreliable
```

Therefore:

```text
Normal stock change
→ Transaction required

Manual correction
→ Adjustment transaction + AuditLog
```

---

# 122. Index Recommendations

Useful indexes:

```javascript
{
    farmId: 1,
    inventoryCode: 1
}
```

```javascript
{
    farmId: 1,
    itemType: 1,
    status: 1
}
```

```javascript
{
    farmId: 1,
    expiryDate: 1
}
```

```javascript
{
    farmId: 1,
    referenceId: 1
}
```

These support:

```text
Farm inventory
Low-stock filtering
Expiry checks
Master-reference lookup
```

---

# 123. Inventory Transaction Indexes

For the separate transaction collection, useful indexes include:

```javascript
{
    farmId: 1,
    inventoryId: 1,
    transactionDate: -1
}
```

and:

```javascript
{
    farmId: 1,
    transactionType: 1,
    transactionDate: -1
}
```

This supports transaction history and reports.

---

# 124. Real-Time Stock Display

The UI can show:

```text
Current Stock
```

directly from Inventory.

When a transaction occurs:

```text
Transaction committed
      ↓
Inventory updated
      ↓
Frontend refresh / realtime event
      ↓
Stock badge changes
```

If the project later adds WebSocket/SSE infrastructure, the stock update can be pushed live.

The database model itself does not provide realtime delivery; that belongs to the application layer.

---

# 125. Realtime Low Stock

Possible architecture:

```text
Stock Transaction
      ↓
Inventory updated
      ↓
Status recalculated
      ↓
If threshold crossed
      ↓
Notification created
      ↓
Realtime event
      ↓
Dashboard updates
```

This fits Liventra's smart-management direction.

---

# 126. Example — Feed Purchase

Initial:

```text
Inventory:
Napier Grass

Stock:
100 kg

Average price:
৳35/kg
```

Purchase:

```text
50 kg
৳40/kg
```

After:

```text
Stock:
150 kg

Average price:
weighted average
```

Transaction:

```text
Purchase
+50 kg
```

Expense:

```text
Feed Purchase
৳2,000
reference → InventoryTransaction
```

---

# 127. Example — Feed Consumption

Initial:

```text
150 kg
```

FeedRecord:

```text
Quantity = 12 kg
```

Then:

```text
Inventory:
138 kg
```

Transaction:

```text
Consumption
12 kg
referenceType = FeedRecord
referenceId = FeedRecord._id
```

---

# 128. Example — Medicine Usage

Initial:

```text
Medicine:
Oxytetracycline

Stock:
20 bottles
```

Treatment uses:

```text
2 bottles
```

Then:

```text
Stock:
18 bottles
```

Transaction:

```text
Treatment Usage
quantity = 2
referenceType = Treatment
referenceId = Treatment._id
```

---

# 129. Example — Vaccine Usage

Initial:

```text
FMD Vaccine
20 bottles
```

Vaccination uses:

```text
1 bottle
```

Then:

```text
19 bottles
```

Transaction:

```text
Vaccination Usage
quantity = 1
referenceType = Vaccination
```

---

# 130. Example — Low Stock

```text
Current:
75 kg

Reorder Level:
80 kg
```

System:

```text
status = Low Stock
```

Notification:

```text
"Napier Grass stock is below reorder level."
```

---

# 131. Example — Out of Stock

```text
Current:
0 bottles
```

System:

```text
status = Out of Stock
```

Notification:

```text
"FMD Vaccine is out of stock."
```

---

# 132. Example — Damage

```text
Current:
100 bottles

Damaged:
3
```

Transaction:

```text
Damage
3 bottles
```

New stock:

```text
97 bottles
```

---

# 133. Example — Physical Adjustment

System:

```text
100 kg
```

Physical count:

```text
96 kg
```

Adjustment:

```text
-4 kg
```

Transaction:

```text
Adjustment
4 kg
remarks = "Physical stock reconciliation"
```

New stock:

```text
96 kg
```

---

# 134. Example — Expiry

```text
Expiry date:
2026-08-01

Today:
2026-08-17
```

System should identify:

```text
Expired
```

and prevent normal consumption.

---

# 135. Inventory + Financial Reporting

Inventory provides operational stock information:

```text
what is available
```

Expense provides financial information:

```text
what money was spent
```

Therefore:

```text
Inventory
+
Expense
```

together support:

```text
Cost analysis
Profit/Loss
Purchase analysis
```

The SRS explicitly requires financial records and profit analysis as well as inventory management. fileciteturn17file9

---

# 136. Inventory + Monthly Reports

Monthly inventory report can show:

```text
Opening stock
Purchases
Consumption
Damage
Expiry
Returns
Adjustments
Closing stock
```

This is derived from:

```text
Inventory
+
Inventory_Transaction
```

---

# 137. Stock Ledger

A useful Inventory detail page:

```text
Stock Ledger

Date       Type          Qty    Balance
Aug 01     Purchase      +100   100
Aug 03     Consumption    -20    80
Aug 05     Damage          -5    75
Aug 10     Purchase       +50   125
```

This provides an auditable stock history.

---

# 138. Final Inventory Architecture

```text
                         FARM
                          │
                          ▼
                    ┌───────────┐
                    │ Inventory │
                    └─────┬─────┘
                          │
             ┌────────────┼────────────┐
             │            │            │
             ▼            ▼            ▼
           Feed        Medicine      Vaccine
             │            │            │
             ▼            ▼            ▼
        FeedRecord     Treatment   Vaccination
             │            │            │
             └────────────┼────────────┘
                          ▼
                Inventory_Transaction
                          │
              ┌───────────┴───────────┐
              ▼                       ▼
          Stock Update              Expense
              │
              ▼
        Low Stock / Expiry
              │
              ▼
         Notification
```

---

# 139. Final Business Rules Checklist

```text
[✓] Inventory stores current stock state
[✓] Inventory_Transaction stores stock movement history
[✓] inventoryCode should be unique
[✓] farmId is required
[✓] all inventory access must be farm-scoped
[✓] itemType is controlled by enum
[✓] referenceId can connect inventory to master records
[✓] referenceId must match itemType
[✓] category provides finer classification
[✓] currentStock is the current available quantity
[✓] normal stock changes should create transactions
[✓] negative stock should normally be prevented
[✓] unit must be consistent
[✓] minimumStock is critical minimum threshold
[✓] reorderLevel triggers reorder/low-stock logic
[✓] maximumStock supports overstock prevention
[✓] averagePurchasePrice can be maintained using weighted average
[✓] supplierName provides basic supplier information
[✓] storageLocation identifies physical storage
[✓] expiryDate is important for medicines/vaccines
[✓] expired items should not normally be consumed
[✓] status is controlled by operational rules
[✓] low-stock notifications should avoid duplicates
[✓] out-of-stock alerts can have higher priority
[✓] purchase can integrate with Expense
[✓] FeedRecord can consume Feed inventory
[✓] Treatment can consume Medicine inventory
[✓] Vaccination can consume Vaccine inventory
[✓] Damage reduces stock through transaction
[✓] Expired stock should be recorded through transaction
[✓] Adjustment handles physical reconciliation
[✓] currentStock should not be casually edited
[✓] direct stock mutation should be protected
[✓] atomic stock updates are important for concurrency
[✓] critical stock operations should be transaction-safe
[✓] inventory changes should be auditable
[✓] hard deletion should be avoided after transactions exist
[✓] inactive status can replace destructive deletion
[✓] inventory supports stock reports
[✓] inventory supports low-stock alerts
[✓] inventory supports purchase history
[✓] inventory integrates with financial analysis
[✓] batch-level expiry is a current limitation
[✓] true FIFO/FEFO is not supported by the current schema
[✓] supplier is not a dedicated master model
[✓] equipment asset management is limited
[✓] reserved stock is not represented
```

---

# 140. Final Status

**Model:** `Inventory`  
**Status:** ✅ Finalized  
**Business Logic:** ✅ Documented  
**Inventory Transaction Integration:** ✅  
**Feed Integration:** ✅  
**Medicine Integration:** ✅  
**Vaccine Integration:** ✅  
**Expense Integration:** ✅  
**Notification Integration:** ✅  
**Auditability:** ✅  
**Stock Reconciliation:** ✅  
**Low Stock Logic:** ✅  
**Expiry Logic:** ✅  
**Financial Integration:** ✅  
**Reporting:** ✅  
**Historical Integrity:** ✅  

**Next Model:** `Inventory_Transaction`
