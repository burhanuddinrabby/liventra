# Liventra — Business Logic Documentation
## Model 06: FeedIngredient

**Document Version:** Business Logic v1.0  
**Project:** Liventra — Smart Livestock Farm Management System  
**Database:** MongoDB  
**Architecture:** MERN

---

# 1. Source Model

The current finalized DB model defines `FeedIngredient` as:

```text
_id
ingredientName
ingredientCode
unit: kg | g | liter | ml | piece
category: Roughage | Concentrate | Protein Supplement | Mineral | Vitamin | Additive
dryMatterPercentage
crudeProtein
crudeFiber
energy
calcium
phosphorus
pricePerUnit
availability
supplierId
description
isAvailable
isActive
createdAt
updatedAt
```

This is the FeedIngredient structure in the current DB model document. fileciteturn8file0

The same document shows that `FeedFormula.ingredients[]` references `ingredientId` and stores formula-specific `percentage`, `quantity`, and `unit`. fileciteturn8file2

---

# 2. Purpose

`FeedIngredient` is the **master database of individual feed ingredients** used by Liventra's Feed Management module.

Examples:

```text
Napier Grass
Rice Bran
Maize
Soybean Meal
Mineral Mixture
Salt
```

The model stores the reusable properties of an ingredient:

```text
Identity
+
Category
+
Nutritional composition
+
Price
+
Availability
+
Supplier
```

It is then referenced by:

```text
FeedFormula
FeedRecommendation
FeedRecord
Inventory
```

---

# 3. Why FeedIngredient Is Necessary

This is an important part of the Feed Management architecture.

Suppose a formula contains:

```text
Napier Grass → 12 kg
Concentrate → 3 kg
```

The system should not store:

```text
Napier Grass
protein = ...
fiber = ...
energy = ...
price = ...
```

inside every formula.

Instead:

```text
FeedIngredient
      ↓
Napier Grass
      ↓
Nutritional + price master data

             ↓
FeedFormula
             ↓
ingredientId = Napier Grass
```

This gives centralized ingredient information.

---

# 4. Core Business Principle

`FeedIngredient` is **master/reference data**, while `FeedFormula` defines how ingredients are combined.

Therefore:

```text
FeedIngredient
    =
"What is this feed ingredient?"

FeedFormula
    =
"How much of this ingredient should be used in a particular formula?"

FeedRecommendation
    =
"What should this specific animal receive now?"

FeedRecord
    =
"What was actually fed?"
```

This separation is extremely important for Liventra.

---

# 5. Feed Data Architecture

```text
                 FeedIngredient
                       |
          +------------+-------------+
          |                          |
          ↓                          ↓
     FeedFormula              Inventory
          |
          ↓
 FeedRecommendation
          |
          ↓
      FeedRecord
```

A single ingredient can therefore participate in many formulas and feeding events.

---

# 6. Field-by-Field Business Logic

## 6.1 `_id`

### Type
MongoDB `ObjectId`

### Required
Yes

### Purpose

Internal primary identifier.

### Used By

Most importantly:

```text
FeedFormula.ingredients[].ingredientId
FeedRecommendation.recommendedItems[].ingredientId
FeedRecord.items[].ingredientId
```

---

# 6.2 `ingredientName`

### Type
String

### Required
Yes

### Example

```text
Napier Grass
```

### Other Examples

```text
Rice Bran
Maize
Soybean Meal
Wheat Bran
Mineral Mixture
Salt
```

### Purpose

Human-readable ingredient name.

### Business Rules

- Cannot be empty.
- Should use standardized naming.
- Avoid creating duplicate ingredients under slightly different names.

Bad:

```text
Rice Bran
RiceBran
rice bran
Rice bran feed
```

if all represent the same master ingredient.

### Recommended

Maintain one canonical ingredient record.

---

# 6.3 `ingredientCode`

### Type
String

### Required
Yes

### Example

```text
ING-000001
```

### Purpose

Human-readable unique identifier for the ingredient.

### Used In

- Feed ingredient list
- Formula management
- Inventory
- Reports
- Audit logs
- Search

### Business Rule

`ingredientCode` should be unique and normally should not change after creation.

---

# 6.4 `unit`

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
```

These are the units defined in the current DB model. fileciteturn8file0

### Purpose

Defines the standard measurement unit used for the ingredient.

### Examples

```text
Napier Grass → kg
Mineral Mixture → kg
Liquid Supplement → liter
Additive → g
```

### Important Business Rule

The unit must be interpreted consistently across:

```text
Ingredient
Formula
Recommendation
FeedRecord
Inventory
```

---

# 7. Unit Consistency

Suppose:

```text
Ingredient unit = kg
```

and a formula stores:

```text
quantity = 12
unit = kg
```

This is straightforward.

But if the formula stores:

```text
quantity = 12000
unit = g
```

the backend may need unit conversion before calculating nutrition/cost.

Therefore, the Feed module should have a consistent unit-conversion policy.

### Concept

```text
1 kg = 1000 g
1 liter = 1000 ml
```

The exact conversion rules should be implemented centrally rather than duplicated across controllers.

---

# 8. `category`

### Type
Enum

### Required
Yes

### Allowed Values

```text
Roughage
Concentrate
Protein Supplement
Mineral
Vitamin
Additive
```

These categories are defined in the current DB model. fileciteturn8file0

### Purpose

Classifies the ingredient by its functional feed category.

### Examples

```text
Napier Grass
→ Roughage

Rice Bran
→ Concentrate

Soybean Meal
→ Protein Supplement

Mineral Mixture
→ Mineral
```

### Business Uses

Category can support:

- Ingredient filtering
- Formula construction
- Nutrition balancing
- Feed reports
- Inventory grouping
- Farmer UI

---

# 9. `dryMatterPercentage`

### Type
Number

### Required
Yes/Optional depending on ingredient data availability

### Example

```text
22
```

meaning a reference dry matter percentage of 22%.

### Purpose

Represents the proportion of the ingredient that is dry matter rather than water.

### Why It Matters

Feed recommendation should distinguish between:

```text
As-fed quantity
```

and:

```text
Dry Matter Intake (DMI)
```

For example, conceptually:

```text
As-fed quantity × DM fraction
=
Dry matter contribution
```

### Important

The exact nutritional reference values must come from the project's approved feed/nutrition sources.

Do not invent ingredient nutrition values in production data.

---

# 10. `crudeProtein`

### Type
Number

### Required
Yes/Optional depending on source data

### Example

```text
12
```

### Purpose

Stores the ingredient's reference crude protein content.

### Business Use

Used when evaluating the nutritional contribution of an ingredient to a formula.

Conceptually:

```text
Ingredient quantity
      ×
Protein content
      ↓
Protein contribution
```

This contributes to formula-level nutrition calculations.

---

# 11. `crudeFiber`

### Type
Number

### Required
Yes/Optional depending on source data

### Example

```text
25
```

### Purpose

Stores reference crude fiber content.

### Business Use

Useful when comparing ingredients and evaluating formula composition.

---

# 12. `energy`

### Type
Number

### Required
Yes/Optional depending on source data

### Example

```text
Reference energy value
```

### Purpose

Stores the ingredient's energy contribution.

### Important

The current DB model specifies the field but does not specify its exact unit in the snippet. Therefore, the project should select and document one standard energy unit before production implementation.

Do not mix different energy units between ingredients.

---

# 13. `calcium`

### Type
Number

### Required
Optional depending on source data

### Purpose

Stores reference calcium content.

### Business Use

Contributes to mineral/nutritional evaluation of formulas.

---

# 14. `phosphorus`

### Type
Number

### Required
Optional depending on source data

### Purpose

Stores reference phosphorus content.

### Business Use

Used in formula nutrition evaluation alongside calcium and other nutrients.

---

# 15. Nutritional Data Principle

The nutritional fields:

```text
dryMatterPercentage
crudeProtein
crudeFiber
energy
calcium
phosphorus
```

are **ingredient-level reference values**.

They are not animal-specific values.

Therefore:

```text
FeedIngredient
   ↓
Nutrition profile

Animal
   ↓
Requirement/context
```

Then:

```text
FeedFormula / Recommendation
   ↓
Combines the two
```

---

# 16. `pricePerUnit`

### Type
Number

### Required
Yes/Optional depending on inventory/pricing workflow

### Example

```text
18
```

If:

```text
unit = kg
```

then conceptually:

```text
pricePerUnit = ৳18/kg
```

### Purpose

Stores the current/reference price of the ingredient per defined unit.

### Used For

- Formula cost estimation
- Feed recommendation cost
- Feed cost analysis
- Budgeting

### Important

This is a **current/reference master price**, not necessarily the historical purchase price.

---

# 17. Price vs Historical Cost

This distinction is important.

### FeedIngredient

```text
pricePerUnit
```

= current/reference ingredient price.

### FeedRecord

The finalized FeedRecord stores:

```text
unitPrice
totalCost
```

inside each actual feeding item. fileciteturn8file2

Therefore, a historical feeding record can preserve the price used at that time even if the current FeedIngredient price changes later.

Example:

```text
August:
Rice Bran = ৳35/kg

September:
Rice Bran = ৳40/kg
```

Old FeedRecord should still retain:

```text
unitPrice = 35
```

rather than recalculating the old feeding cost using ৳40.

---

# 18. `availability`

### Type
Number / quantity representation

### Required
Optional

### Example

```text
250
```

### Purpose

Represents available quantity/reference availability for the ingredient.

### Important Distinction

The finalized model contains both:

```text
availability
isAvailable
```

These should not be treated as identical.

A sensible interpretation is:

```text
availability
=
how much is available / reference availability quantity

isAvailable
=
whether the ingredient is currently available for use
```

The exact semantics should be fixed in the implementation before coding to prevent confusion.

---

# 19. `supplierId`

### Type
Reference/ObjectId

### Required
Optional

### Purpose

Links the ingredient to its supplier/source.

Conceptually:

```text
FeedIngredient
      ↓
supplierId
      ↓
Supplier
```

### Business Uses

- Supplier tracking
- Purchase management
- Price comparison
- Inventory procurement
- Ingredient sourcing

### Important

The current FeedIngredient structure contains `supplierId`, but the finalized core collection list does not establish a separate Supplier collection in the shown Feed module. Therefore, the exact referenced entity should be finalized before implementation if supplier management is to be fully relational.

---

# 20. `description`

### Type
String

### Required
Optional

### Example

```text
High-fiber roughage commonly used as a major forage component.
```

### Purpose

Stores additional information about the ingredient.

### Used For

- Ingredient details
- Farmer guidance
- Admin reference
- Notes

---

# 21. `isAvailable`

### Type
Boolean

### Required
Yes

### Example

```text
true
```

### Purpose

Controls whether the ingredient is currently available for operational use.

### True

```text
isAvailable = true
```

Ingredient can normally be selected for new formulas/recommendations, subject to other rules.

### False

```text
isAvailable = false
```

Ingredient should not normally be selected for new operational recommendations.

---

# 22. `isAvailable` vs `availability`

These fields should have clearly separated responsibilities.

Recommended interpretation:

```text
availability
    ↓
Quantity/reference availability

isAvailable
    ↓
Boolean operational availability
```

Example:

```text
availability = 250 kg
isAvailable = true
```

If stock becomes zero:

```text
availability = 0
isAvailable = false
```

### Important

If actual stock is fully managed by the Inventory module, `availability` should not become a second independent stock ledger.

Instead, it should either be treated as reference information or synchronized from Inventory according to the final implementation policy.

---

# 23. `isActive`

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

Controls whether the ingredient master record itself is active.

### Difference from `isAvailable`

```text
isActive
=
Does this ingredient master record still belong to the supported catalog?

isAvailable
=
Can it currently be used/obtained?
```

Example:

```text
Ingredient permanently removed from catalog:
isActive = false
```

versus:

```text
Ingredient temporarily out of stock:
isActive = true
isAvailable = false
```

This distinction is useful.

---

# 24. `createdAt`

### Type
Date

### Required
Yes

### Generated
System

### Purpose

Stores when the ingredient master record was created.

---

# 25. `updatedAt`

### Type
Date

### Required
Yes

### Generated
System

### Purpose

Stores when the ingredient master record was last modified.

---

# 26. FeedIngredient Lifecycle

A typical lifecycle:

```text
Ingredient Created
       ↓
Active + Available
       ↓
Used in FeedFormula
       ↓
Used in Recommendations
       ↓
Used in FeedRecords
       ↓
Temporarily Unavailable
       ↓
Available Again
       ↓
Possibly Deactivated
```

---

# 27. Ingredient Creation Workflow

```text
Authorized Feed Admin
        ↓
Create Ingredient
        ↓
Enter Name + Code
        ↓
Select Unit
        ↓
Select Category
        ↓
Enter Nutrition Data
        ↓
Enter Reference Price
        ↓
Set Availability
        ↓
Set Supplier if applicable
        ↓
Validate
        ↓
Create Ingredient
        ↓
AuditLog
```

---

# 28. Who Should Manage FeedIngredient?

Recommended policy:

```text
Admin / SuperAdmin
        ↓
Master ingredient management
```

Farm Owner/Manager may be allowed to:

```text
View
Use in formulas
View price/availability
```

but unrestricted creation/editing of nutritional master data should be controlled.

### Why?

If users arbitrarily change:

```text
crudeProtein
energy
dryMatterPercentage
```

the feed recommendation system could become scientifically inconsistent.

---

# 29. Why Nutrition Data Needs Strong Control

Consider:

```text
Napier Grass
crudeProtein = X
```

This value can affect:

```text
Formula nutrition
      ↓
Recommendation
      ↓
Estimated feed cost
      ↓
Farmer decision
```

Therefore, changing a nutritional value is not just a normal profile edit.

It can influence future feed recommendations.

---

# 30. Ingredient Update Workflow

For ordinary information:

```text
Update description
Update availability
Update supplier
```

For important nutritional data:

```text
Update nutrition value
       ↓
Validate source
       ↓
Authorized approval
       ↓
Update master data
       ↓
AuditLog
```

Historical FeedRecommendation records should not be recalculated automatically.

---

# 31. Historical Recommendation Protection

This is critical for the thesis/business logic.

Suppose:

```text
Old:
Napier Grass
DM = X
```

Recommendation generated:

```text
180 kg animal
↓
Formula Version 1
↓
Recommendation R-00001
```

Later:

```text
Napier Grass nutrition data changes
```

The old recommendation should remain:

```text
Recommendation R-00001
```

with its stored:

```text
inputSnapshot
recommendedItems
ruleVersion
```

as finalized in FeedRecommendation. fileciteturn8file2

The system should not silently recompute historical recommendations from current ingredient master data.

---

# 32. FeedIngredient + FeedFormula Relationship

This is a key relationship.

```text
FeedIngredient (1)
        ↓
FeedFormula ingredient reference (Many)
```

Example:

```text
FeedIngredient:
Napier Grass
```

may appear in:

```text
Dairy Lactating Formula
Beef Growing Formula
Heifer Formula
Summer Formula
Winter Formula
```

Therefore one ingredient can participate in many formulas.

---

# 33. FeedFormula Determines Quantity

FeedIngredient does **not** decide how much an animal should eat.

FeedFormula contains:

```text
ingredientId
percentage
quantity
unit
```

as shown in the finalized model. fileciteturn8file2

Conceptually:

```text
FeedIngredient
"What is Napier Grass?"

        +

FeedFormula
"Use 12 kg Napier Grass"

        ↓

Recommendation
"Animal A should receive 12 kg today"
```

---

# 34. FeedRecommendation Relationship

Recommendation uses ingredient information as part of the recommendation output.

The finalized model stores:

```text
ingredientId
ingredientName // Snapshot
quantity
unit
estimatedCost
```

inside `recommendedItems[]`. fileciteturn8file2

### Why Snapshot ingredientName?

Suppose an ingredient name changes:

```text
"Rice Bran"
```

to:

```text
"Rice Bran Premium"
```

Historical recommendation should still preserve what it displayed at generation time.

This is consistent with the project's snapshot-based recommendation design.

---

# 35. FeedRecord Relationship

Actual feeding records store:

```text
ingredientId
ingredientName
quantity
unit
unitPrice
totalCost
```

inside each feeding item. fileciteturn8file2

This means FeedRecord preserves historical operational data.

Example:

```text
FeedIngredient current price:
৳40/kg

Old FeedRecord:
unitPrice = ৳35/kg
```

The old record remains historically accurate.

---

# 36. Inventory Relationship

FeedIngredient is a master feed catalog item.

Inventory tracks farm-specific stock.

Conceptually:

```text
FeedIngredient
      ↓
What the item is

Inventory
      ↓
How much this farm currently has
```

Example:

```text
FeedIngredient:
Napier Grass

Farm A Inventory:
500 kg

Farm B Inventory:
1000 kg
```

Therefore FeedIngredient should not be treated as the farm's complete stock ledger.

---

# 37. Stock/Availability Business Flow

A practical flow is:

```text
Ingredient Master
      ↓
Inventory Purchase
      ↓
Inventory increases
      ↓
Feed Consumption
      ↓
Inventory decreases
      ↓
Low Stock Alert
```

The exact inventory transaction logic belongs to the Inventory module.

FeedIngredient provides the standardized ingredient identity.

---

# 38. Price Calculation

A formula's estimated cost can conceptually be calculated from:

```text
Ingredient Quantity
        ×
Price Per Unit
        =
Ingredient Cost
```

Then:

```text
All ingredient costs
        ↓
Total Formula Cost
```

For example:

```text
Napier:
12 kg × ৳5 = ৳60

Concentrate:
3 kg × ৳70 = ৳210

Total:
৳270/day
```

The actual price values must come from the project's maintained data.

---

# 39. Important Price Rule

Current FeedIngredient price should be used for:

```text
Future formula cost estimation
Future recommendation cost estimation
```

Historical records should use their own stored snapshot:

```text
FeedRecommendation.estimatedCost
FeedRecord.unitPrice
FeedRecord.totalCost
```

This prevents old records from changing when market prices change.

---

# 40. Availability Logic

Recommended operational behavior:

### Available

```text
isAvailable = true
```

Ingredient may be selected.

### Temporarily unavailable

```text
isAvailable = false
isActive = true
```

Ingredient remains in master catalog but should normally be excluded from new operational recommendations.

### Permanently retired

```text
isActive = false
```

Ingredient should not be selectable for new formulas.

Historical formulas/records remain intact.

---

# 41. Formula Validation Using Ingredients

When creating a FeedFormula:

```text
Select ingredient
      ↓
Check ingredient exists
      ↓
Check ingredient.isActive
      ↓
Check ingredient.isAvailable
      ↓
Validate quantity/unit
      ↓
Calculate nutrition
      ↓
Calculate estimated cost
      ↓
Save formula
```

A deactivated ingredient should not normally be allowed in a newly created active formula.

---

# 42. Recommendation Validation

When generating a recommendation:

```text
Formula selected
      ↓
Load ingredient references
      ↓
Check ingredient status
      ↓
Calculate quantity/nutrition/cost
      ↓
Generate recommendation
```

If an ingredient has become unavailable, the system should not blindly recommend it.

Possible business responses:

```text
Warning
Alternative ingredient
Recalculate formula
Reject recommendation
```

The exact substitution strategy should be part of the future recommendation engine.

---

# 43. Ingredient Substitution

The current FeedIngredient model does **not** contain a dedicated substituteIngredientId field.

Therefore substitution should not be stored directly in this model unless the project later decides to extend it.

For now:

```text
Unavailable ingredient
      ↓
Recommendation engine checks approved formulas/alternatives
      ↓
Generate warning or alternative
```

Do not invent an automatic substitution rule without approved nutritional/business rules.

---

# 44. CRUD Business Rules

## Create

Authorized feed/master-data manager.

Must validate:

- Unique ingredient code
- Valid unit
- Valid category
- Valid nutritional values
- Valid price
- Valid supplier reference if provided

---

## Read

Farm users with feed-management permission can view ingredient information.

---

## Update

Authorized users can update master information.

Nutritional and price changes should be treated as important changes and audited.

---

## Delete

Hard deletion is not recommended once the ingredient has been used.

Use:

```text
isActive = false
```

instead.

---

# 45. Validation Rules

| Field | Validation |
|---|---|
| ingredientName | Required, non-empty |
| ingredientCode | Required, unique |
| unit | Enum |
| category | Enum |
| dryMatterPercentage | Valid non-negative numeric value |
| crudeProtein | Valid non-negative numeric value |
| crudeFiber | Valid non-negative numeric value |
| energy | Valid non-negative numeric value |
| calcium | Valid non-negative numeric value |
| phosphorus | Valid non-negative numeric value |
| pricePerUnit | Non-negative |
| availability | Valid quantity representation |
| supplierId | Valid reference if provided |
| isAvailable | Boolean |
| isActive | Boolean |

---

# 46. Data Integrity Rules

### Rule 1

`ingredientCode` must be unique.

### Rule 2

Ingredient category must use the predefined enum.

### Rule 3

Unit must use the predefined enum.

### Rule 4

Nutritional values cannot be negative.

### Rule 5

Price cannot be negative.

### Rule 6

Deactivating an ingredient must not break historical formulas/recommendations/records.

### Rule 7

Historical recommendation and feeding records must not be recalculated from changed master data.

### Rule 8

Current availability must not silently rewrite historical feeding data.

---

# 47. Audit Rules

Important FeedIngredient audit events:

```text
Ingredient Created
Ingredient Updated
Ingredient Activated
Ingredient Deactivated
Ingredient Availability Changed
Ingredient Price Changed
Ingredient Nutrition Changed
Supplier Changed
```

Especially important:

```text
crudeProtein
crudeFiber
dryMatterPercentage
energy
calcium
phosphorus
pricePerUnit
```

changes should be traceable.

---

# 48. Notification Rules

Most ingredient changes do not require farmer notifications.

Potential notification cases:

```text
Ingredient becomes unavailable
Critical ingredient stock becomes unavailable
Ingredient price changes significantly
Important feed master data is updated
```

These can feed into the Notification module if the final UX requires it.

---

# 49. Reporting Usage

FeedIngredient supports reports such as:

```text
Ingredient List
Ingredient Cost Comparison
Feed Cost by Ingredient
Most Used Ingredients
Unavailable Ingredients
Ingredient Price Trends
Nutrition Reference Table
```

Some of these reports require FeedRecord/Inventory history rather than FeedIngredient alone.

---

# 50. Search and Filtering

Useful filters:

```text
ingredientCode
ingredientName
category
unit
isAvailable
isActive
supplierId
```

Example:

```text
Category = Roughage
        ↓
Napier Grass
Maize Silage
Other approved roughages
```

---

# 51. Index Recommendations

These are implementation recommendations and do not change the finalized model.

Useful indexes:

```javascript
ingredientCode: 1
ingredientName: 1
category: 1
isAvailable: 1
isActive: 1
supplierId: 1
```

A useful compound query index may be:

```javascript
{
    category: 1,
    isActive: 1,
    isAvailable: 1
}
```

for active ingredient selection screens.

---

# 52. Example Scenario — Create Ingredient

Admin creates:

```text
Ingredient Name:
Napier Grass

Code:
ING-000001

Unit:
kg

Category:
Roughage

Dry Matter:
[approved reference value]

Crude Protein:
[approved reference value]

Crude Fiber:
[approved reference value]

Energy:
[approved reference value]

Calcium:
[approved reference value]

Phosphorus:
[approved reference value]

Price:
[current reference price]

isAvailable:
true

isActive:
true
```

System:

```text
Validate
   ↓
Create FeedIngredient
   ↓
AuditLog
```

---

# 53. Example Scenario — Ingredient Used in Formula

Suppose:

```text
Ingredient:
Napier Grass
ingredientId = ING-000001
```

FeedFormula:

```text
Lactating Dairy Formula

ingredients:
[
    {
        ingredientId: ING-000001,
        percentage: ...,
        quantity: 12,
        unit: kg
    }
]
```

The Formula references the Ingredient instead of copying its complete nutrition profile.

---

# 54. Example Scenario — Price Change

Current:

```text
Rice Bran
pricePerUnit = ৳35/kg
```

Later:

```text
pricePerUnit = ৳40/kg
```

New formula cost:

```text
uses ৳40/kg
```

Old FeedRecord:

```text
unitPrice = ৳35/kg
```

remains unchanged.

This preserves historical cost accuracy.

---

# 55. Example Scenario — Ingredient Becomes Unavailable

```text
Napier Grass
isActive = true
isAvailable = false
```

Meaning:

```text
Ingredient still exists in master catalog
BUT
currently unavailable for new operations
```

Existing formulas and historical records are preserved.

A new recommendation should detect the unavailable ingredient and follow the approved warning/substitution logic.

---

# 56. Example Scenario — Ingredient Retired

If an ingredient is permanently removed from active use:

```text
isActive = false
```

The ingredient remains in the database because:

```text
Old FeedFormula
Old FeedRecommendation
Old FeedRecord
```

may still reference it.

---

# 57. Important Difference Between the Four Feed Layers

This should be understood clearly for both implementation and thesis defense.

## Layer 1 — FeedIngredient

```text
What is the ingredient?
```

Example:

```text
Napier Grass
Protein = ...
Fiber = ...
Price = ...
```

---

## Layer 2 — FeedFormula

```text
How should ingredients be combined?
```

Example:

```text
Napier = 12 kg
Concentrate = 3 kg
```

for a specific stage/purpose/season/target.

---

## Layer 3 — FeedRecommendation

```text
What should this particular animal receive?
```

Example:

```text
Animal A
Weight = 180 kg

Recommendation:
Napier = 12 kg
Concentrate = 3 kg
Estimated Cost = ...
```

The recommendation stores a snapshot of the relevant inputs/output.

---

## Layer 4 — FeedRecord

```text
What was actually fed?
```

Example:

```text
Morning:
Napier = 10 kg
Concentrate = 2.5 kg

Afternoon:
Napier = 8 kg
Concentrate = 1 kg
```

This separation makes the Feed module much more powerful.

---

# 58. Defense Questions

### Q: Why do you need a separate FeedIngredient model?

**Answer:**

> "FeedIngredient is master data for individual feed components. It stores standardized nutritional, pricing and availability information that can be reused by multiple feed formulas, recommendations and feeding records without duplicating the same information."

---

### Q: Why not store ingredient names directly inside FeedFormula?

**Answer:**

> "Because the same ingredient can be used in many formulas. Storing ingredientId provides a centralized reference and prevents duplication and inconsistent ingredient information."

---

### Q: What is the difference between FeedIngredient and FeedFormula?

**Answer:**

> "FeedIngredient describes an individual feed component, while FeedFormula describes how multiple ingredients are combined for a particular breed, physiological stage, purpose, season and target condition."

---

### Q: What happens if the price of an ingredient changes?

**Answer:**

> "The current FeedIngredient price can affect future formula and recommendation cost estimation. Historical FeedRecord and FeedRecommendation data retain their own cost snapshots, so old records are not recalculated."

---

### Q: What happens if an ingredient becomes unavailable?

**Answer:**

> "The ingredient can remain active in the master catalog while `isAvailable` becomes false. Existing historical records remain valid, but new formulas or recommendations should normally not use the unavailable ingredient unless the business rules explicitly allow it."

---

### Q: Why do you have both `isAvailable` and `isActive`?

**Answer:**

> "`isActive` represents whether the ingredient remains part of the supported master catalog, while `isAvailable` represents whether it can currently be obtained/used. An ingredient can therefore be active but temporarily unavailable."

---

### Q: Why do you store nutrition values in FeedIngredient?

**Answer:**

> "Because nutritional composition is a property of the feed ingredient. FeedFormula and FeedRecommendation can use these standardized values to evaluate the nutritional contribution of each ingredient."

---

# 59. Final Business Rules Checklist

```text
[✓] FeedIngredient is master/reference data
[✓] ingredientCode is a stable identifier
[✓] Unit is controlled by enum
[✓] Category is controlled by enum
[✓] Nutritional values belong to the ingredient
[✓] Ingredient does not determine animal-specific quantity
[✓] FeedFormula determines formula composition
[✓] FeedRecommendation determines animal-specific recommendation
[✓] FeedRecord stores actual feeding
[✓] Current price supports future cost estimation
[✓] Historical records preserve historical prices/costs
[✓] isAvailable controls operational availability
[✓] isActive controls master-record lifecycle
[✓] Historical references remain valid after deactivation
[✓] Nutrition changes should be audited
[✓] Price changes should be audited
[✓] Supplier relationship is optional
[✓] Ingredient data should come from approved nutrition sources
[✓] New formulas should validate ingredient status
[✓] Historical recommendations must not be silently recalculated
```

---

# 60. Final Status

**Model:** `FeedIngredient`  
**Status:** ✅ Finalized  
**Business Logic:** ✅ Documented  
**Database Structure:** 🔒 Frozen  
**Next Model:** `FeedFormula`
