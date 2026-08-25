# Liventra — Business Logic Documentation
## Model 04: Breed

**Document Version:** Business Logic v1.0  
**Project:** Liventra — A Smart Livestock Farm Management System  
**Database:** MongoDB  
**Architecture:** MERN

---

# 1. Source Model

The finalized `Breed` model contains:

```text
_id
breedName
origin
category: Local | Crossbred | Exotic
purpose: Dairy | Beef | Breeding | Dual Purpose
averageMilk
averageWeight
feedRequirement
heatResistance
diseaseResistance
description
isActive
createdAt
updatedAt
```

The finalized database specification defines Breed as a master/reference entity used by animals and feed logic. fileciteturn6file1

> **Important:** This document explains the business logic around the finalized model. It does not introduce a separate BreedCategory collection or replace the existing structure.

---

# 2. Purpose

The `Breed` model stores the master information of cattle breeds supported by Liventra.

It provides standardized breed information that can be reused across many animals.

Instead of entering breed characteristics separately for every animal:

```text
Animal A
   ↓
Breed = Holstein

Animal B
   ↓
Breed = Holstein

Animal C
   ↓
Breed = Holstein
```

all shared breed-level information is maintained once in:

```text
Breed
```

---

# 3. Why This Model Exists

Breed is important because cattle characteristics differ by breed.

Breed-level information can influence:

- Animal classification
- Expected milk production
- Expected body weight
- Purpose
- Feed recommendation
- Farm analytics
- Health analysis
- Breeding management

The `Animal` model therefore references a Breed rather than duplicating all breed information.

---

# 4. Core Business Principle

`Breed` is a **master/reference data model**.

That means:

```text
Breed
   ↓
Referenced by many Animals
```

The Breed record should be relatively stable.

Animal-specific values should remain in the `Animal` model.

For example:

```text
Breed.averageMilk
```

is a breed-level reference value.

Whereas:

```text
Animal.currentWeight
Animal-specific milk production
```

belongs to the individual animal.

---

# 5. Breed Lifecycle

```text
Breed Created
      ↓
Active
      ↓
Used by Animals / Feed Logic
      ↓
Possibly Updated
      ↓
Deactivated
```

Breed records should normally be deactivated rather than hard-deleted once they have been used by animals.

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
Animal.breedId
FeedFormula.breedId
```

---

# 6.2 `breedName`

### Type
String

### Required
Yes

### Example

```text
Holstein Friesian
```

Other examples may include the breed names approved by the project's Bangladesh breed dataset/master list.

### Purpose

Human-readable breed name.

### Business Rules

- Cannot be empty.
- Should be normalized consistently.
- Duplicate active breed names should generally be prevented unless the system intentionally supports distinct variants.

### Important

Breed naming should be standardized.

For example, avoid accidentally creating:

```text
Holstein Friesian
Holstein-Friesian
HF
Holstein friesen
```

as four unrelated master records if they represent the same standardized breed.

A `shortCode` is not part of the finalized model, so do not add one merely for convenience.

---

# 6.3 `origin`

### Type
String

### Required
Yes/Optional according to master-data policy

### Example

```text
Bangladesh
Netherlands
United States
India
```

### Purpose

Stores the breed's origin.

### Business Use

Can be shown in:

- Breed information page
- Animal profile
- Breed comparison
- Reports

### Important

`origin` describes the breed, not the individual animal's purchase location.

---

# 6.4 `category`

### Type
Enum

### Required
Yes

### Allowed Values

```text
Local
Crossbred
Exotic
```

### Purpose

Classifies the breed into a broad breed category.

### Example

```text
category = Crossbred
```

### Business Uses

- Breed filtering
- Farm analytics
- Animal statistics
- Feed/management comparison
- Research analysis

### Validation

Only the predefined enum values should be accepted.

---

# 6.5 `purpose`

### Type
Enum

### Required
Yes

### Allowed Values

```text
Dairy
Beef
Breeding
Dual Purpose
```

### Purpose

Defines the primary production/management purpose associated with the breed.

### Example

```text
purpose = Dairy
```

### Business Logic

This can help the system determine which management information is most relevant.

For example:

```text
Dairy
 ↓
Milk-related monitoring

Beef
 ↓
Growth/weight-related monitoring

Breeding
 ↓
Reproductive management

Dual Purpose
 ↓
Milk + growth/breeding considerations
```

### Important

Breed purpose is a **reference characteristic**.

The actual purpose of an individual animal is stored separately in:

```text
Animal.purpose
```

This distinction is important.

An animal's operational purpose may differ from the general purpose classification of its breed.

---

# 6.6 `averageMilk`

### Type
Number

### Required
Optional

### Example

```text
18
```

### Unit

The finalized model does not define a separate unit field.

The project should use one consistent unit, e.g.:

```text
Liter/day
```

if that is the agreed project standard.

### Purpose

Stores a reference average milk-production value for the breed.

### Business Logic

Can be used for:

- Breed comparison
- Dashboard reference
- Feed recommendation context
- Production benchmarking
- Research analysis

### Critical Rule

This is **not** an animal's actual milk production.

Actual production belongs in:

```text
MilkRecord
```

---

# 6.7 `averageWeight`

### Type
Number

### Required
Optional

### Example

```text
550
```

### Unit

Use one consistent project-wide unit, normally:

```text
kg
```

### Purpose

Stores reference average body weight for the breed.

### Business Logic

Useful for:

- Breed comparison
- Animal benchmarking
- Feed recommendation
- Growth analysis

### Critical Rule

Do not overwrite this value with an animal's current weight.

Animal-specific weight is stored in:

```text
Animal.currentWeight
WeightRecord.weight
```

---

# 6.8 `feedRequirement`

### Type
Object / structured data

### Required
Optional

### Purpose

Stores breed-level reference information about feed requirements.

### Example Concept

```javascript
feedRequirement: {
    notes: "Higher nutritional requirement for high-producing dairy cattle"
}
```

The exact internal structure should follow the finalized implementation contract.

### Important Design Rule

`feedRequirement` should **not** become the actual daily feeding plan for an animal.

Actual formula/recommendation logic belongs to:

```text
FeedFormula
FeedRecommendation
FeedRecord
```

The Breed model only provides reference information.

---

# 6.9 `heatResistance`

### Type
String / Number depending on the finalized implementation representation

### Required
Optional

### Example

```text
High
```

### Purpose

Stores a breed-level reference characteristic regarding tolerance/resistance to heat conditions.

### Business Use

Can support:

- Breed comparison
- Farm planning
- Climate-aware management
- Future heat-stress analytics

### Important

This should not be interpreted as a live animal health measurement.

Actual animal temperature/activity signals come from:

```text
SensorData
```

and actual health state comes from:

```text
Animal.healthStatus
```

---

# 6.10 `diseaseResistance`

### Type
String / Number depending on the finalized implementation representation

### Required
Optional

### Example

```text
Moderate
```

### Purpose

Stores breed-level reference information about general disease resistance.

### Business Logic

This can be displayed as reference information.

It should **not** be used as proof that an individual animal is disease-free or resistant to a particular disease.

Animal health is managed through:

```text
Disease
Vaccination
Treatment
Animal.healthStatus
```

---

# 6.11 `description`

### Type
String

### Required
Optional

### Example

```text
High-yield dairy breed commonly used in intensive milk production systems.
```

### Purpose

Stores additional breed information.

### Used In

- Breed detail page
- Education/reference section
- Animal registration
- Reports
- Farmer guidance

---

# 6.12 `isActive`

### Type
Boolean

### Required
Yes

### Default

```text
true
```

### Purpose

Controls whether the breed is currently available for selection.

### Active

```text
isActive = true
```

The breed can be selected when registering/editing animals.

### Inactive

```text
isActive = false
```

The breed should no longer appear as a selectable option for new animals.

### Important

Existing animals using that breed must continue to reference it.

Therefore:

```text
Inactive Breed
      ↓
Existing Animals remain valid
```

---

# 6.13 `createdAt`

### Type
Date

### Required
Yes

### Generated
System

### Purpose

Stores when the master breed record was created.

---

# 6.14 `updatedAt`

### Type
Date

### Required
Yes

### Generated
System

### Purpose

Stores the latest update time.

---

# 7. Breed Creation Workflow

Breed is master data.

A typical workflow:

```text
Authorized User/Admin
       ↓
Create Breed
       ↓
Validate Name
       ↓
Validate Category
       ↓
Validate Purpose
       ↓
Validate Reference Values
       ↓
Save Breed
       ↓
AuditLog
```

Normal farm workers should not automatically be allowed to create arbitrary breed records.

---

# 8. Who Should Manage Breed Master Data?

Recommended policy:

```text
SuperAdmin / Admin
        ↓
Primary Breed Master Management
```

Farm Owner/Manager may be allowed to **use/select** existing breeds.

Whether farm owners can create new breeds should be controlled carefully.

### Why?

If every farm can create its own spelling/variant:

```text
Holstein
Holstein Friesian
HF
Holstein-Friesian
```

the database becomes inconsistent.

For a university project, centralized master-data management is cleaner.

---

# 9. Breed Selection During Animal Registration

Typical workflow:

```text
Create Animal
      ↓
Select Breed
      ↓
System loads active Breed records
      ↓
Store breedId
      ↓
Animal created
```

The Animal should store:

```text
breedId
```

not duplicate:

```text
breedName
origin
averageMilk
averageWeight
```

---

# 10. Breed + Animal Relationship

```text
Breed (1)
   ↓
Animal (Many)
```

Example:

```text
Holstein Friesian
       |
       +---- Animal A
       +---- Animal B
       +---- Animal C
       +---- Animal D
```

This is a one-to-many relationship.

---

# 11. Breed + FeedFormula Relationship

Breed also connects to FeedFormula:

```text
Breed
  ↓
FeedFormula
  ↓
Stage / Purpose / Season
```

Example:

```text
Breed = Holstein Friesian

Stage = Lactating
Purpose = Dairy
Season = Summer

        ↓

FeedFormula
```

This is important because breed characteristics can be one of the inputs used to select an appropriate feeding formula.

---

# 12. FeedRecommendation Relationship

Breed information can contribute to recommendation generation:

```text
Animal
  ↓
breedId
  ↓
Breed characteristics
  +
Animal current weight
  +
Stage
  +
Milk yield
  +
Season
  ↓
FeedRecommendation
```

However, the recommendation must not simply copy the Breed's `feedRequirement`.

The actual recommendation engine should use the approved formula/rules.

---

# 13. Breed Data vs Animal Data

This distinction is critical.

| Breed-level | Animal-level |
|---|---|
| averageMilk | actual milk |
| averageWeight | currentWeight |
| feedRequirement | actual feeding |
| heatResistance | current health/temperature |
| diseaseResistance | actual disease |
| breed purpose | animal purpose |

Example:

```text
Breed.averageWeight = 550 kg

Animal A.currentWeight = 480 kg
Animal B.currentWeight = 610 kg
```

Both can belong to the same breed.

---

# 14. Updating Breed Data

Breed master information can be updated by authorized users.

However, changes to reference values should be treated carefully because they may affect future recommendations.

Example:

```text
averageWeight
old = 500 kg
new = 550 kg
```

This could influence future analytics or rule selection.

### Recommended Business Rule

Any material Breed master-data update should generate an AuditLog.

---

# 15. Historical Recommendation Protection

Suppose:

```text
2026
Breed.feedRequirement = Version/Rule A
```

and later the reference data changes.

Old FeedRecommendation records must **not** be recalculated automatically.

This follows the same snapshot principle already established for `FeedRecommendation`.

Historical recommendations preserve their:

```text
inputSnapshot
recommendedItems
ruleVersion
```

Therefore:

```text
Breed changes
      ↓
Future recommendation may change
      ↓
Old recommendation remains unchanged
```

---

# 16. Breed Deactivation Workflow

Suppose a breed is no longer used as a new selection.

```text
Breed
 ↓
isActive = false
```

Then:

```text
New Animal
   ↓
Breed not selectable
```

But:

```text
Existing Animal
   ↓
breedId remains valid
```

This prevents historical data from breaking.

---

# 17. CRUD Business Rules

## Create

Allowed to authorized master-data managers.

Validation:

- Unique/standardized breed name
- Valid category
- Valid purpose
- Valid reference metrics

---

## Read

Most farm users can read active breed information because breed data is reference information.

Detailed sensitive information is generally not an issue here.

---

## Update

Authorized administrators/master-data managers.

Material changes should be audited.

---

## Delete

Hard deletion is not recommended.

Use:

```text
isActive = false
```

instead.

---

# 18. Validation Rules

| Field | Validation |
|---|---|
| breedName | Required, non-empty |
| origin | Valid text if provided |
| category | Enum |
| purpose | Enum |
| averageMilk | Non-negative if provided |
| averageWeight | Positive if provided |
| feedRequirement | Valid structured data if provided |
| heatResistance | Valid standardized representation |
| diseaseResistance | Valid standardized representation |
| description | Optional text |
| isActive | Boolean |

---

# 19. Business Constraints

### Constraint 1

Breed name should be standardized.

### Constraint 2

Only predefined category values are accepted.

### Constraint 3

Only predefined purpose values are accepted.

### Constraint 4

Average milk/weight are reference values, not live animal measurements.

### Constraint 5

Breed deactivation must not break existing Animal references.

### Constraint 6

Breed master-data changes should not rewrite historical FeedRecommendations.

### Constraint 7

Only authorized users should manage the Breed master list.

---

# 20. Notification Rules

Breed changes normally do not require farmer notifications.

However, if a material master-data change affects a farm's operational rules, an administrative notification can be generated.

Example:

```text
"Feed reference data for Holstein Friesian has been updated."
```

This is optional and should not create notification noise.

---

# 21. Audit Rules

Recommended Breed audit events:

```text
Breed Created
Breed Updated
Breed Deactivated
Breed Reactivated
Breed Reference Data Changed
```

For updates:

```text
oldData
newData
changedFields
```

Example:

```text
Old averageWeight = 500
New averageWeight = 550
ChangedField = averageWeight
```

---

# 22. Dashboard Usage

Breed data can support dashboard analytics such as:

```text
Animals by Breed

Holstein Friesian     45
Sahiwal                28
Local                  37
Crossbred              15
```

And:

```text
Average Milk by Breed
Average Weight by Breed
Breed Distribution
```

These should preferably be calculated from Animal and MilkRecord/WeightRecord data rather than permanently storing aggregate counters in Breed.

---

# 23. Reporting Usage

Breed can be used as a report filter:

```text
Animal Report
    ↓
Breed = Holstein Friesian
```

Possible reports:

- Animals by breed
- Milk production by breed
- Weight performance by breed
- Health statistics by breed
- Feed cost by breed
- Breeding performance by breed

---

# 24. API-Level Business Flow

Logical API operations:

```text
Breed
├── Create Breed
├── List Breeds
├── Get Breed
├── Update Breed
├── Activate Breed
└── Deactivate Breed
```

Animal registration uses:

```text
GET active breeds
      ↓
User selects breed
      ↓
Animal.breedId stored
```

---

# 25. Edge Cases

## Case 1 — Breed is already used by animals

Do not hard-delete.

Use:

```text
isActive = false
```

---

## Case 2 — Breed name already exists

Reject duplicate master record unless there is a clearly defined reason for multiple variants.

---

## Case 3 — Breed data is updated

Existing Animal records should not suddenly receive copied historical values.

They continue referencing the Breed master record, while historical operational records remain unchanged.

---

## Case 4 — Feed formula exists for an inactive breed

Historical formula remains available for historical records.

It should not necessarily be offered for new operations unless the business rules permit it.

---

## Case 5 — Breed average milk differs from actual animal milk

This is expected.

Example:

```text
Breed average = 18 L/day

Animal actual:
12 L/day
```

The actual value comes from MilkRecord.

---

# 26. Example Scenario — Creating a Breed

Authorized admin creates:

```text
breedName:
Holstein Friesian

origin:
Netherlands

category:
Exotic

purpose:
Dairy

averageMilk:
18

averageWeight:
550

heatResistance:
Low

diseaseResistance:
Moderate
```

System:

```text
Validate
   ↓
Create Breed
   ↓
isActive = true
   ↓
AuditLog
```

---

# 27. Example Scenario — Registering Animal

Farmer creates:

```text
Animal:
AN-000125

Breed:
Holstein Friesian
```

System stores:

```text
Animal.breedId = Breed._id
```

It does not duplicate:

```text
averageMilk
averageWeight
origin
```

inside Animal.

---

# 28. Example Scenario — Breed Used in Feed Recommendation

```text
Animal
  ↓
breedId
  ↓
Holstein Friesian
  ↓
Stage = Lactating
  ↓
Weight = 520 kg
  ↓
Milk Yield = 18 L/day
  ↓
Season = Summer
  ↓
FeedFormula matching
  ↓
FeedRecommendation
```

The recommendation is generated from the complete animal context and approved feeding rules, not from breed name alone.

---

# 29. Defense Questions

### Q: Why did you create a separate Breed model?

**Answer:**

> "Breed is master/reference data shared by many animals. Keeping it separately avoids duplicating breed characteristics in every Animal document and allows standardized breed-based analytics and feed-rule selection."

---

### Q: Why isn't averageMilk stored in Animal?

**Answer:**

> "AverageMilk is a breed-level reference value, while an individual animal's actual milk production changes over time and is stored in MilkRecord. Separating them prevents confusion between benchmark data and actual production."

---

### Q: What happens if a breed is no longer used?

**Answer:**

> "We deactivate it using isActive=false instead of deleting it. Existing animals can still reference the breed, while it is removed from new breed-selection lists."

---

### Q: How can Breed influence FeedRecommendation?

**Answer:**

> "Breed is one contextual input. The system combines breed, animal weight, stage, purpose, season, pregnancy/lactation status, milk yield and other approved inputs to select or generate the appropriate feeding recommendation."

---

### Q: Can the same breed be used by many animals?

**Answer:**

> "Yes. Breed has a one-to-many relationship with Animal. Many animals can reference the same Breed document."

---

# 30. Final Business Rules Checklist

```text
[✓] Breed is master/reference data
[✓] Many animals can reference one breed
[✓] Breed has controlled category enum
[✓] Breed has controlled purpose enum
[✓] Breed reference values are not animal measurements
[✓] Animal stores breedId rather than duplicated breed data
[✓] Breed can support feed-rule selection
[✓] Breed can support analytics and reporting
[✓] Breed master changes are auditable
[✓] Breed should be deactivated rather than hard-deleted
[✓] Existing animals remain valid after breed deactivation
[✓] Historical FeedRecommendations remain unchanged
[✓] No separate Role-like collection is needed for Breed
```

---

# 31. Final Status

**Model:** `Breed`  
**Status:** ✅ Finalized  
**Business Logic:** ✅ Documented  
**Database Structure:** 🔒 Frozen  
**Next Model:** `Animal`
