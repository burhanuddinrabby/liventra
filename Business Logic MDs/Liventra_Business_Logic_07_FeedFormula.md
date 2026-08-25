# Liventra — Business Logic Documentation
## Model 07: FeedFormula

**Document Version:** Business Logic v1.0  
**Project:** Liventra — Smart Livestock Farm Management System  
**Database:** MongoDB  
**Architecture:** MERN

---

# 1. Source Model

The finalized `FeedFormula` model contains:

```text
_id
formulaCode
formulaName
breedId

stage:
    Calf | Growing | Heifer | Lactating | Dry | Breeding

purpose:
    Maintenance | Breeding | Growth | Milk Production

season:
    Summer | Winter | Rainy | All Season

targetWeightRange:
    min
    max

targetMilkYield:
    min
    max

targetADG

ingredients:
    [
        {
            ingredientId
            percentage
            quantity
            unit
        }
    ]

expectedNutrition:
    dryMatter
    crudeProtein
    energy

estimatedDailyCost
ruleVersion
description
isDefault
isActive
createdBy
updatedBy
createdAt
updatedAt
```

This exact finalized structure appears in the current DB model document. fileciteturn8file0

The source also defines `FeedFormula` as part of the Feed Module alongside `FeedIngredient`, `FeedRecommendation`, and `FeedRecord`. fileciteturn8file5

---

# 2. Purpose

`FeedFormula` is the **approved/reusable feeding blueprint** of Liventra.

It answers:

> **"For which type of animal, at which stage, for which purpose, in which season and under which target conditions, what combination of feed ingredients should be provided?"**

For example:

```text
Breed:
Holstein Friesian

Stage:
Lactating

Purpose:
Milk Production

Season:
Summer

Target Milk Yield:
15–20 L/day

        ↓

FeedFormula

        ↓

Napier Grass
Concentrate
Mineral
etc.
```

The formula is not yet a recommendation for one specific animal.

---

# 3. The Four Feed Layers

This distinction is extremely important.

## Layer 1 — FeedIngredient

```text
"What is this ingredient?"
```

Example:

```text
Napier Grass
Protein = ...
Fiber = ...
Energy = ...
Price = ...
```

---

## Layer 2 — FeedFormula

```text
"How should the ingredients be combined
for a defined animal condition?"
```

Example:

```text
Lactating
Summer
Milk Production

Napier = 12 kg
Concentrate = 3 kg
Mineral = 0.1 kg
```

---

## Layer 3 — FeedRecommendation

```text
"What should Animal A receive now?"
```

The recommendation uses the animal's current context and preserves an input/output snapshot.

---

## Layer 4 — FeedRecord

```text
"What was actually fed?"
```

This records the real feeding event.

The finalized DB model explicitly separates these four Feed Module collections. fileciteturn8file5

---

# 4. Why FeedFormula Is Necessary

Without a formula layer, the system might directly calculate recommendations from raw ingredients every time.

That creates several problems:

- Difficult expert validation
- Difficult rule management
- Difficult version tracking
- Difficult comparison between formulas
- Difficult historical evaluation
- Difficult reuse

Instead:

```text
Approved Formula
       ↓
Animal Context
       ↓
Recommendation
       ↓
Actual Feeding
```

This gives Liventra a controlled decision-support architecture.

---

# 5. Core Business Principle

A FeedFormula is a **reusable rule/template**, not an individual animal record.

Therefore:

```text
FeedFormula
    ↓
Many animals can use it
```

Example:

```text
Formula F-001
    ↓
Animal A
Animal B
Animal C
Animal D
```

Each animal can later receive its own `FeedRecommendation`.

---

# 6. Formula Lifecycle

A practical lifecycle is:

```text
Formula Created
       ↓
Validated
       ↓
Active
       ↓
Used for Recommendations
       ↓
Possibly Updated / New Version
       ↓
Old Version Preserved
       ↓
Deactivated
```

The project should avoid overwriting an important historical formula in a way that destroys the rule set used by previous recommendations.

---

# 7. Field-by-Field Business Logic

# 7.1 `_id`

### Type
MongoDB `ObjectId`

### Required
Yes

### Purpose

Internal database identifier.

### Used By

Most importantly:

```text
FeedRecommendation.formulaId
FeedRecord.recommendationId indirectly through recommendation history
```

---

# 7.2 `formulaCode`

### Type
String

### Required
Yes

### Example

```text
FF-000001
```

### Purpose

Human-readable unique identifier for a formula.

### Used In

- Formula management
- Search
- Reports
- Recommendation history
- Audit logs
- Expert review

### Business Rule

Formula code should be unique and should normally remain stable for that formula/version record.

---

# 7.3 `formulaName`

### Type
String

### Required
Yes

### Example

```text
Lactating Dairy Summer Formula
```

### Purpose

Human-readable name for the formula.

### Examples

```text
Calf Starter Formula
Growing Beef Summer Formula
Lactating Dairy Winter Formula
Dry Cow Maintenance Formula
Breeding Heifer Formula
```

### Business Rule

The name should describe the formula clearly enough for farm users and administrators.

---

# 7.4 `breedId`

### Type
ObjectId / Breed reference

### Required
Yes

### Purpose

Associates the formula with a specific breed.

```text
FeedFormula
      ↓
breedId
      ↓
Breed
```

### Example

```text
breedId = Holstein Friesian
```

### Business Logic

Breed can affect:

- Nutritional requirements
- Expected production
- Growth characteristics
- Feed strategy

Therefore, breed is one of the formula-selection criteria.

---

# 7.5 `stage`

### Type
Enum

### Required
Yes

### Allowed Values

```text
Calf
Growing
Heifer
Lactating
Dry
Breeding
```

These are the finalized source values. fileciteturn8file0

### Purpose

Defines the physiological/management stage for which the formula is intended.

### Example

```text
stage = Growing
```

### Business Logic

The same breed can require different feeding strategies at different stages.

Example:

```text
Holstein Friesian
      |
      +-- Calf Formula
      +-- Growing Formula
      +-- Heifer Formula
      +-- Lactating Formula
      +-- Dry Formula
```

---

# 7.6 `purpose`

### Type
Enum

### Required
Yes

### Allowed Values

```text
Maintenance
Breeding
Growth
Milk Production
```

### Purpose

Defines the nutritional/production objective of the formula.

### Example

```text
purpose = Milk Production
```

### Business Logic

Purpose helps distinguish formulas designed for:

```text
Maintenance
Growth
Breeding
Milk Production
```

A formula's purpose should be compatible with the animal context.

---

# 7.7 `season`

### Type
Enum

### Required
Yes

### Allowed Values

```text
Summer
Winter
Rainy
All Season
```

### Purpose

Allows seasonal feeding differences.

This is particularly relevant for Bangladesh because:

- forage availability can change
- environmental conditions change
- feed prices can change
- animal heat/cold stress can influence management

### Example

```text
season = Summer
```

### `All Season`

A formula marked:

```text
All Season
```

can be considered a general formula when no seasonal specialization is required.

---

# 7.8 `targetWeightRange`

### Type
Embedded Object

### Structure

```javascript
targetWeightRange: {
    min,
    max
}
```

### Example

```javascript
{
    min: 150,
    max: 250
}
```

### Unit

The project should use a consistent livestock weight unit, normally:

```text
kg
```

### Purpose

Defines the body-weight range for which the formula is intended.

### Business Logic

Suppose:

```text
Formula:
Growing Beef Formula

Target:
150–250 kg
```

An animal at:

```text
180 kg
```

may match.

An animal at:

```text
450 kg
```

would not normally match this formula.

---

# 7.9 `targetMilkYield`

### Type
Embedded Object

### Structure

```javascript
targetMilkYield: {
    min,
    max
}
```

### Example

```javascript
{
    min: 15,
    max: 20
}
```

### Unit

The project should establish one standard, normally:

```text
Liter/day
```

### Purpose

Defines the expected milk-production range for the formula.

### Business Logic

This is particularly useful for lactating animals.

Example:

```text
Target Milk Yield:
15–20 L/day
```

A cow producing:

```text
18 L/day
```

is a strong candidate.

---

# 7.10 `targetADG`

### Type
Number

### Required
Optional depending on formula purpose

### Example

```text
0.8
```

### Unit

Normally:

```text
kg/day
```

### Meaning

ADG = **Average Daily Gain**

It represents the target daily body-weight gain.

### Business Logic

This is particularly important for:

```text
Growing
Beef
Growth
```

formulas.

Example:

```text
Current weight = 180 kg
Target ADG = 0.8 kg/day
```

The feed formula is designed to support that growth objective.

---

# 7.11 `ingredients`

### Type
Array of Embedded Objects

### Structure

```javascript
ingredients: [
    {
        ingredientId,
        percentage,
        quantity,
        unit
    }
]
```

This structure is explicitly defined in the finalized DB model. fileciteturn8file0

### Purpose

Defines the actual ingredients that compose the formula.

---

# 8. Ingredient Subdocument

## 8.1 `ingredientId`

### Type
ObjectId / FeedIngredient reference

### Required
Yes

### Purpose

Identifies the ingredient being used.

Example:

```text
ingredientId → Napier Grass
```

### Business Rule

Ingredient must exist in FeedIngredient.

---

# 8.2 `percentage`

### Type
Number

### Required
Yes

### Example

```text
60
```

### Meaning

Represents the ingredient's percentage contribution to the formula according to the formula's defined basis.

### Important Validation

If the project defines `percentage` as the percentage of the complete formula:

```text
Sum of percentages = 100%
```

should normally be enforced.

### Critical Implementation Note

The team must define exactly what the percentage represents:

```text
percentage of total as-fed mix
```

or another approved basis.

Do not mix different interpretations between formulas.

---

# 8.3 `quantity`

### Type
Number

### Required
Yes

### Example

```text
12
```

### Meaning

The formula's defined quantity of that ingredient.

Example:

```text
quantity = 12
unit = kg
```

means:

```text
12 kg
```

under the formula's defined feeding basis.

### Important

The formula's quantity is not necessarily the final quantity consumed by every animal.

The recommendation engine may use the formula as a base and adjust according to animal-specific inputs.

---

# 8.4 `unit`

### Type
Enum / String aligned with FeedIngredient unit

### Required
Yes

### Examples

```text
kg
g
liter
ml
piece
```

### Business Rule

The formula unit should be compatible with the referenced FeedIngredient's unit or be convertible through the system's centralized unit-conversion logic.

---

# 9. Percentage vs Quantity

This needs careful business handling.

Suppose:

```text
Napier Grass
percentage = 70
quantity = 12 kg
```

The system should not blindly assume that:

```text
70% = 12 kg
```

without knowing the formula basis.

The project should define one consistent interpretation.

### Recommended interpretation

Use:

```text
percentage
```

for composition of the formula/mix.

Use:

```text
quantity
```

for the prescribed amount under the formula's feeding basis.

Then validate their relationship during formula creation.

---

# 10. Ingredient Duplication Rule

The same ingredient should not normally appear twice in the same formula.

Bad:

```text
Napier → 60%
Napier → 10%
```

Instead:

```text
Napier → 70%
```

unless there is a documented reason for duplicate entries.

This simplifies:

- Nutrition calculation
- Cost calculation
- Recommendation generation
- UI display

---

# 11. `expectedNutrition`

### Type
Embedded Object

### Structure

```javascript
expectedNutrition: {
    dryMatter,
    crudeProtein,
    energy
}
```

The finalized DB model defines these three fields. fileciteturn8file0

### Purpose

Stores the expected nutritional characteristics of the complete formula.

This is different from the nutrition values stored in FeedIngredient.

---

# 12. Ingredient Nutrition vs Formula Nutrition

## FeedIngredient

```text
Napier Grass
↓
Its own nutrition profile
```

## FeedFormula

```text
Napier
+
Concentrate
+
Mineral
↓
Combined formula nutrition
```

Therefore:

```text
FeedIngredient.crudeProtein
```

is ingredient-level.

Whereas:

```text
FeedFormula.expectedNutrition.crudeProtein
```

is formula-level.

---

# 13. `expectedNutrition.dryMatter`

### Type
Number

### Purpose

Stores expected dry matter contribution/value of the formula.

### Business Use

Useful for:

- DMI calculation
- Formula comparison
- Recommendation validation

---

# 14. `expectedNutrition.crudeProtein`

### Type
Number

### Purpose

Stores expected crude protein level of the complete formula.

### Business Use

Used when evaluating whether the formula meets the intended nutritional target.

---

# 15. `expectedNutrition.energy`

### Type
Number

### Purpose

Stores expected energy value of the formula.

### Important

The exact energy unit must be standardized across the project.

---

# 16. `estimatedDailyCost`

### Type
Number

### Required
Yes/Optional depending on formula state

### Example

```text
280
```

### Unit

BDT/day.

### Purpose

Stores the estimated daily cost of following the formula.

### Conceptual Calculation

```text
Ingredient Quantity
        ×
Current/reference price
        ↓
Ingredient cost

All ingredient costs
        ↓
Estimated daily formula cost
```

### Important

This is an estimate.

Actual cost can differ because market prices change.

---

# 17. Cost Snapshot Principle

When a FeedRecommendation is generated, it stores:

```text
estimatedCost
estimatedDailyCost
```

as part of the recommendation output. fileciteturn8file2

When actual feeding occurs, FeedRecord stores:

```text
unitPrice
totalCost
```

Therefore:

```text
Formula
→ estimated/base cost

Recommendation
→ estimated cost at recommendation time

FeedRecord
→ actual recorded feeding cost
```

This is a very important separation.

---

# 18. `ruleVersion`

### Type
String

### Required
Yes

### Example

```text
v1.0
```

or:

```text
FEED-RULE-2026-01
```

### Purpose

Identifies the rule/nutrition logic version used to create the formula.

### Why It Matters

Suppose:

```text
Rule Version = v1.0
```

later becomes:

```text
Rule Version = v2.0
```

Historical recommendations must still be traceable to the rule set that produced them.

---

# 19. Rule Version + Thesis Evaluation

This is particularly important for the project's academic evaluation.

Suppose:

```text
Animal weight = 180 kg

Formula:
F-001

Rule:
v1.0
```

Two months later:

```text
Animal weight = 230 kg

Rule:
v2.0
```

The system should be able to explain:

```text
Old recommendation
→ Formula/rule version v1.0

New recommendation
→ Formula/rule version v2.0
```

This prevents historical recommendations from becoming ambiguous.

---

# 20. `description`

### Type
String

### Required
Optional

### Example

```text
Summer feeding formula for lactating dairy cattle producing approximately 15–20 L of milk per day.
```

### Purpose

Human-readable explanation of the formula.

---

# 21. `isDefault`

### Type
Boolean

### Required
Yes

### Example

```text
true
```

### Purpose

Identifies whether the formula is the default formula for its matching context.

### Example

```text
Breed = Holstein
Stage = Lactating
Purpose = Milk Production
Season = Summer

Default Formula = F-001
```

### Critical Rule

There should not be multiple active default formulas for the exact same matching context unless the system intentionally supports priority.

---

# 22. `isActive`

### Type
Boolean

### Required
Yes

### Default

```text
true
```

### Purpose

Controls whether the formula is currently available for new recommendation generation/formula selection.

### Active

```text
isActive = true
```

Can be used.

### Inactive

```text
isActive = false
```

Should not be selected for new recommendations.

Historical recommendations remain valid.

---

# 23. `createdBy`

### Type
Reference to User

### Required
Yes

### Purpose

Identifies who created the formula.

### Example

```text
USR-2026-00001
```

### Business Use

Supports accountability and audit.

---

# 24. `updatedBy`

### Type
Reference to User

### Required
Optional

### Purpose

Identifies who last updated the formula.

---

# 25. `createdAt`

### Type
Date

### Required
Yes

### Generated
System

---

# 26. `updatedAt`

### Type
Date

### Required
Yes

### Generated
System

---

# 27. Formula Matching Logic

A recommendation engine can conceptually search for a formula using:

```text
Animal
 ↓
Breed
 ↓
Stage
 ↓
Purpose
 ↓
Season
 ↓
Weight Range
 ↓
Milk Yield Range
 ↓
Target ADG
 ↓
Active Formula
```

Example:

```text
Animal A

Breed:
Holstein

Stage:
Lactating

Purpose:
Milk Production

Season:
Summer

Weight:
520 kg

Milk:
18 L/day
```

System searches for:

```text
breedId = Holstein
stage = Lactating
purpose = Milk Production
season = Summer
weight = within target range
milk yield = within target range
isActive = true
```

---

# 28. Formula Selection Priority

The exact ranking algorithm is not defined by the DB model itself.

A reasonable application-level strategy is:

```text
1. Exact breed match
2. Exact stage match
3. Exact purpose match
4. Exact season match
5. Target weight range match
6. Target milk yield match
7. Target ADG compatibility
8. Active formula
9. Default formula preference
```

This is **business-logic recommendation**, not a field-level rule explicitly stated in the DB source.

The final matching algorithm should be documented separately when the recommendation engine is implemented.

---

# 29. What If Multiple Formulas Match?

Example:

```text
Formula A
Formula B
Formula C
```

all match the animal.

The system needs a deterministic selection strategy.

Possible approach:

```text
Exact match score
      ↓
Highest score
      ↓
isDefault = true
      ↓
Latest approved/active rule
```

If ambiguity remains:

```text
Do not silently choose.
Show available formulas / require selection.
```

This is especially useful for expert validation.

---

# 30. What If No Formula Matches?

The system should not fabricate a formula.

Possible result:

```text
No suitable active formula found.
```

Then:

```text
Warning
+
Manual formula selection
or
Expert review
```

This is safer than generating unsupported nutritional advice.

---

# 31. Formula Creation Workflow

```text
Authorized User
      ↓
Create FeedFormula
      ↓
Select Breed
      ↓
Select Stage
      ↓
Select Purpose
      ↓
Select Season
      ↓
Set Target Weight
      ↓
Set Target Milk Yield
      ↓
Set Target ADG
      ↓
Add Ingredients
      ↓
Validate Ingredient Status
      ↓
Validate Percentage
      ↓
Validate Quantity/Unit
      ↓
Calculate Expected Nutrition
      ↓
Calculate Estimated Cost
      ↓
Assign Rule Version
      ↓
Set Default/Active
      ↓
Save
      ↓
AuditLog
```

---

# 32. Formula Validation

Before saving a formula, validate:

```text
Breed exists
Stage valid
Purpose valid
Season valid
Ingredients exist
Ingredients active
Units valid
Percentages valid
Quantity > 0
Nutrition values valid
Cost >= 0
RuleVersion present
```

---

# 33. Formula Percentage Validation

If percentage represents total mix composition:

```text
Sum(percentages) = 100
```

Example:

```text
Napier = 60%
Concentrate = 30%
Mineral = 10%

Total = 100%
```

If the project's approved formulation method uses another basis, the validation must follow that documented method.

The important point is:

> One formula must have one consistent percentage interpretation.

---

# 34. Formula Cost Calculation

Conceptually:

```text
For each ingredient:

quantity × current/reference price
          ↓
ingredient estimated cost

Then:

sum(all ingredient costs)
          ↓
estimatedDailyCost
```

Example:

```text
Napier:
12 kg × ৳5 = ৳60

Concentrate:
3 kg × ৳70 = ৳210

Mineral:
0.1 kg × ৳100 = ৳10

Total:
৳280/day
```

These numbers are illustrative only.

The actual project must use approved ingredient price data.

---

# 35. Formula Nutrition Calculation

Conceptually:

```text
Ingredient quantity
      ×
Ingredient nutrition
      ↓
Contribution

All contributions
      ↓
Formula expectedNutrition
```

Example:

```text
Napier
   ↓
DM + Protein + Energy

Concentrate
   ↓
DM + Protein + Energy

Mineral
   ↓
Mineral contribution

        ↓

Expected formula nutrition
```

The exact equations and nutritional standard must be defined from the project's approved feed/nutrition sources.

---

# 36. Formula vs Animal Current State

A formula is reusable.

Animal state changes.

Example:

```text
Today:
Animal = 180 kg
```

Later:

```text
Animal = 230 kg
```

The formula may remain:

```text
F-001
```

but the recommendation engine may select:

```text
another formula
```

or generate a different recommendation based on the new animal context.

---

# 37. Formula Versioning

When a formula's scientifically meaningful composition/rule changes, do not simply destroy the old rule.

Example:

```text
Formula F-001
Rule v1.0

        ↓

New composition

        ↓

Formula F-002
Rule v2.0
```

or another versioning strategy approved by the team.

### Goal

Historical recommendations must remain traceable.

---

# 38. Formula Update vs New Formula

## Minor metadata change

Example:

```text
description
```

can generally be updated.

## Scientific/business-rule change

Example:

```text
ingredient percentage
quantity
target nutrition
target ADG
```

should be treated as a new rule version or new formula record where historical traceability matters.

This is an important business rule for the thesis.

---

# 39. Formula Deactivation

If a formula becomes outdated:

```text
isActive = false
```

Do not delete it if it has been used historically.

Historical recommendation:

```text
formulaId → old formula
```

remains valid.

---

# 40. Default Formula Logic

Suppose:

```text
Breed = Holstein
Stage = Lactating
Purpose = Milk Production
Season = Summer
```

There may be:

```text
Formula A → isDefault = true
Formula B → isDefault = false
```

The default formula can be automatically selected.

### Important Constraint

Avoid multiple active defaults for the same exact context.

---

# 41. FeedRecommendation Workflow

The relationship is:

```text
Animal
   ↓
Current context
   ↓
Find suitable FeedFormula
   ↓
Formula ingredients
   ↓
Calculate animal-specific recommendation
   ↓
FeedRecommendation
```

The finalized FeedRecommendation model stores `formulaId` and an input snapshot of animal context. fileciteturn8file2

---

# 42. Historical Recommendation Protection

This is one of the most important rules in Liventra.

Suppose:

```text
Day 1:
Weight = 180 kg
Formula = F-001
Rule = v1.0
```

Recommendation:

```text
12 kg Napier
3 kg Concentrate
```

Two months later:

```text
Weight = 230 kg
Formula = F-002
Rule = v2.0
```

The system must preserve:

```text
Old recommendation
```

instead of recalculating it using:

```text
new weight
new formula
new ingredient prices
new rules
```

The finalized FeedRecommendation design explicitly uses snapshots and `ruleVersion` for this purpose. fileciteturn8file2

---

# 43. Expert Validation

The formula itself should be created/managed by authorized users.

The project may later support:

```text
Draft
      ↓
Review
      ↓
Approved
      ↓
Active
```

The exact approval status is not a field in the finalized FeedFormula model; the recommendation model contains its own review/status fields. fileciteturn8file2

Therefore, if formula-level expert approval is required later, it should be added deliberately rather than assumed to already exist.

---

# 44. CRUD Business Rules

## Create

Allowed to authorized Feed/Administration users.

Requires:

- Valid breed
- Valid stage
- Valid purpose
- Valid season
- Valid ingredients
- Valid nutrition
- Valid cost
- Rule version

---

## Read

Farm users with Feed Management access can view active formulas.

Historical/inactive formulas may be visible to authorized users for audit/reporting.

---

## Update

Metadata can be updated normally.

Scientific composition changes should create a new version/record where historical traceability is important.

---

## Delete

Hard deletion is discouraged.

Use:

```text
isActive = false
```

for retired formulas.

---

# 45. Validation Rules

| Field | Validation |
|---|---|
| formulaCode | Required, unique |
| formulaName | Required |
| breedId | Valid Breed |
| stage | Enum |
| purpose | Enum |
| season | Enum |
| targetWeightRange.min | Non-negative |
| targetWeightRange.max | Greater than/equal to min |
| targetMilkYield.min | Non-negative |
| targetMilkYield.max | Greater than/equal to min |
| targetADG | Non-negative where used |
| ingredients | At least one valid ingredient |
| percentage | Valid numeric value |
| quantity | Positive |
| unit | Valid/compatible unit |
| expectedNutrition | Valid non-negative values |
| estimatedDailyCost | Non-negative |
| ruleVersion | Required |
| isDefault | Boolean |
| isActive | Boolean |
| createdBy | Valid User |
| updatedBy | Valid User |

---

# 46. Business Constraints

### Constraint 1

Formula must reference a valid Breed.

### Constraint 2

Every ingredient must reference a valid FeedIngredient.

### Constraint 3

Inactive ingredients should not be used in new active formulas.

### Constraint 4

Ingredient percentage must follow one consistent formulation basis.

### Constraint 5

Quantity and unit must be compatible.

### Constraint 6

Target ranges must be logically valid.

### Constraint 7

Estimated cost must not be negative.

### Constraint 8

Historical formulas must remain traceable.

### Constraint 9

Historical recommendations must not be overwritten by formula updates.

### Constraint 10

Multiple active default formulas for the same exact matching context should be prevented unless a priority system is intentionally introduced.

---

# 47. Audit Rules

Important Formula events:

```text
Formula Created
Formula Updated
Formula Activated
Formula Deactivated
Formula Made Default
Formula Removed as Default
Ingredient Added
Ingredient Removed
Ingredient Quantity Changed
Ingredient Percentage Changed
Target Range Changed
Nutrition Changed
Cost Changed
Rule Version Changed
```

These changes are especially important because they can affect future feed recommendations.

---

# 48. Notification Rules

Possible notifications:

```text
New formula available
Formula updated
Default formula changed
Formula deactivated
Important feeding rule updated
```

Notifications should be targeted only to relevant users to avoid unnecessary alerts.

---

# 49. Reporting Usage

FeedFormula supports:

```text
Formula List
Formula Comparison
Cost Comparison
Nutrition Comparison
Breed-wise Formula Report
Season-wise Formula Report
Stage-wise Formula Report
```

Example:

```text
Summer vs Winter
Lactating Formula
```

can show:

```text
Ingredient composition
Expected nutrition
Estimated cost
Rule version
```

---

# 50. Search and Filtering

Useful filters:

```text
formulaCode
formulaName
breedId
stage
purpose
season
isDefault
isActive
ruleVersion
```

Example:

```text
Stage = Lactating
Purpose = Milk Production
Season = Summer
```

returns active matching formulas.

---

# 51. Index Recommendations

These are implementation recommendations.

Useful indexes:

```javascript
formulaCode: 1
breedId: 1
stage: 1
purpose: 1
season: 1
isActive: 1
isDefault: 1
```

A useful compound index for formula matching:

```javascript
{
    breedId: 1,
    stage: 1,
    purpose: 1,
    season: 1,
    isActive: 1
}
```

Target range matching may require additional application logic rather than a simple index.

---

# 52. Example Scenario — Growing Beef Formula

Suppose an approved formula is:

```text
Formula Code:
FF-000010

Name:
Growing Beef Summer Formula

Breed:
Crossbred

Stage:
Growing

Purpose:
Growth

Season:
Summer

Target Weight:
150–250 kg

Target ADG:
0.8 kg/day
```

Ingredients:

```text
Napier Grass
Concentrate
Protein Supplement
Mineral
```

The formula stores:

```text
percentage
quantity
unit
```

for each ingredient.

Then:

```text
Animal
180 kg
Growing
Crossbred
Summer
Target ADG = 0.8

       ↓

Formula matched

       ↓

FeedRecommendation generated
```

---

# 53. Example Scenario — Lactating Dairy Formula

```text
Breed:
Holstein Friesian

Stage:
Lactating

Purpose:
Milk Production

Season:
Summer

Target Milk Yield:
15–20 L/day
```

Animal:

```text
Weight:
520 kg

Milk Yield:
18 L/day
```

System:

```text
Find active matching formula
       ↓
F-000021
       ↓
Generate FeedRecommendation
```

---

# 54. Example Scenario — Formula Becomes Outdated

Old:

```text
Formula F-001
Rule v1.0
Active
```

New research/rule:

```text
Formula F-002
Rule v2.0
Active
```

Then:

```text
F-001 → isActive = false
F-002 → isActive = true
```

Old recommendation:

```text
formulaId = F-001
ruleVersion = v1.0
```

remains unchanged.

New recommendation:

```text
formulaId = F-002
ruleVersion = v2.0
```

---

# 55. Example Scenario — Ingredient Price Changes

Formula:

```text
Napier = 12 kg
```

Current price:

```text
৳5/kg
```

Estimated cost:

```text
৳60
```

Later:

```text
Napier = ৳7/kg
```

Future recommendation cost may change.

But:

```text
Old FeedRecommendation
```

must retain its previously calculated estimated cost.

Similarly:

```text
Old FeedRecord
```

retains its historical unitPrice and totalCost.

---

# 56. Example Scenario — Ingredient Becomes Unavailable

Formula contains:

```text
Napier Grass
```

Ingredient:

```text
isAvailable = false
```

Then:

```text
New recommendation
      ↓
Detect unavailable ingredient
      ↓
Warning / alternative / manual review
```

The system should not silently claim that the farmer can use an unavailable ingredient.

The exact alternative-selection algorithm is a future business-rule component.

---

# 57. Important Separation: Formula vs Recommendation

### Formula

Reusable rule:

```text
Breed
Stage
Purpose
Season
Target
Ingredients
Nutrition
Cost
Rule Version
```

### Recommendation

Animal-specific decision:

```text
Animal
Current Weight
Age
Stage
Purpose
Milk Yield
ADG
Season
Formula
Recommended Quantities
Estimated Cost
Explanation
Warnings
```

This is why both models are necessary.

---

# 58. Important Separation: Formula vs Actual Feeding

Formula says:

```text
Recommended/base composition
```

FeedRecord says:

```text
Actual feeding event
```

Example:

```text
Formula:
Napier = 12 kg

Actual:
Napier = 10 kg
```

This difference should be preserved.

It allows Liventra to later analyze:

```text
Recommended vs Actual
```

---

# 59. Future Analytics

Once enough FeedFormula + FeedRecommendation + FeedRecord + WeightRecord + MilkRecord data exists, Liventra can calculate:

```text
Recommended Feed Cost
vs
Actual Feed Cost

Recommended Quantity
vs
Actual Quantity

Feed Cost
vs
Weight Gain

Feed Cost
vs
Milk Production

Formula
vs
ADG

Formula
vs
Milk Yield
```

This can make the project much stronger than a simple CRUD application.

---

# 60. Defense Questions

### Q: What is the difference between FeedIngredient and FeedFormula?

**Answer:**

> "FeedIngredient stores the properties of an individual feed component, while FeedFormula defines how multiple ingredients are combined for a specific breed, stage, purpose, season and target condition."

---

### Q: Why do you need both percentage and quantity?

**Answer:**

> "Percentage represents the ingredient's composition within the defined formula basis, while quantity represents the prescribed amount under the formula's feeding basis. Keeping both allows the formula to represent composition as well as operational quantity."

---

### Q: Why is FeedFormula not directly linked to Animal?

**Answer:**

> "Because a formula is reusable master/rule data. Multiple animals can use the same formula. The Animal-specific result is stored separately in FeedRecommendation."

---

### Q: What happens when an animal's weight changes?

**Answer:**

> "The formula itself does not necessarily change. The recommendation engine reevaluates the animal's current context and can select a different formula or generate a different recommendation. The previous recommendation remains historically preserved."

---

### Q: Why do you need ruleVersion?

**Answer:**

> "Because feeding rules can change over time. RuleVersion allows us to identify which rule set was used when a formula/recommendation was generated, which is important for traceability, expert validation and thesis evaluation."

---

### Q: What happens if an ingredient price changes?

**Answer:**

> "Future formula/recommendation cost estimation can use the new reference price, while historical recommendations and actual feeding records retain their own cost snapshots."

---

### Q: What happens if no formula matches an animal?

**Answer:**

> "The system should not fabricate a feeding formula. It should return a warning and either allow an authorized user to select an appropriate formula or request expert/manual review."

---

### Q: Why don't you simply calculate the recommendation directly from FeedIngredient?

**Answer:**

> "Because FeedIngredient only describes individual ingredients. FeedFormula provides validated combinations and target conditions. This creates a controlled and reusable knowledge layer between raw ingredient data and animal-specific recommendations."

---

# 61. Final Business Rules Checklist

```text
[✓] FeedFormula is reusable feed-rule/master data
[✓] Formula references Breed
[✓] Formula defines Stage
[✓] Formula defines Purpose
[✓] Formula defines Season
[✓] Formula can define target Weight Range
[✓] Formula can define target Milk Yield Range
[✓] Formula can define target ADG
[✓] Formula contains ingredient references
[✓] Ingredient percentages are validated consistently
[✓] Ingredient quantities are validated
[✓] Ingredient units must be compatible
[✓] Formula nutrition is separate from ingredient nutrition
[✓] Formula has estimated daily cost
[✓] Formula has ruleVersion
[✓] Formula can be default
[✓] Formula can be activated/deactivated
[✓] Historical formulas should remain traceable
[✓] Scientific rule changes should preserve historical traceability
[✓] Formula is used to generate animal-specific recommendations
[✓] FeedRecommendation stores the historical recommendation snapshot
[✓] FeedRecord stores actual feeding
[✓] Formula changes should be audited
[✓] Inactive formulas should not normally be used for new recommendations
[✓] No matching formula should result in warning/manual review rather than fabricated advice
```

---

# 62. Final Status

**Model:** `FeedFormula`  
**Status:** ✅ Finalized  
**Business Logic:** ✅ Documented  
**Database Structure:** 🔒 Frozen  
**Next Model:** `FeedRecommendation`
