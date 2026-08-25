# Liventra — Business Logic Documentation
## Model 14: Disease

**Document Version:** Business Logic v1.0  
**Project:** Liventra — Smart Livestock Farm Management System  
**Database:** MongoDB  
**Architecture:** MERN  
**Module:** Health & Vaccination

---

# 1. Source / Finalized Model

The finalized `Disease` model is:

```text
Disease Model:
_id
diseaseCode
diseaseName
category: Viral | Bacterial | Parasitic | Nutritional | Metabolic | Reproductive | Other
symptoms[]
description
prevention
recommendedVaccineIds[]
isActive
createdAt
updatedAt
```

The finalized DB model defines Disease as a health master-data entity and includes the disease-to-vaccine relationship through `recommendedVaccineIds[]`. fileciteturn14file0

---

# 2. Purpose

`Disease` is the **master/reference model for livestock diseases** in Liventra.

It answers:

> "What disease is this, what category does it belong to, what symptoms are associated with it, how can it be prevented, and which vaccines are recommended?"

Example:

```text
Disease:
Foot and Mouth Disease

Category:
Viral

Symptoms:
Fever
Excessive salivation
Mouth lesions
Lameness

Prevention:
Vaccination + biosecurity

Recommended Vaccines:
FMD Vaccine
```

---

# 3. Disease vs Treatment

This distinction is important.

## Disease

```text
What health condition does the animal have?
```

It is master/reference information.

## Treatment

```text
How was a specific animal treated for a disease?
```

Treatment contains the animal-specific event/history.

Therefore:

```text
Disease
   ↓
Master definition

Treatment
   ↓
Animal-specific treatment event
```

---

# 4. Disease vs Vaccination

Similarly:

```text
Disease
   ↓
recommendedVaccineIds[]
   ↓
Vaccine
   ↓
Vaccination
   ↓
Animal
```

Disease does not record:

```text
Which animal was vaccinated
```

That belongs to `Vaccination`.

Disease defines the health condition and its prevention relationship.

---

# 5. Disease's Role in Liventra

Disease sits at the center of several health workflows:

```text
                    ┌─────────────┐
                    │   Disease   │
                    └──────┬──────┘
                           │
          ┌────────────────┼────────────────┐
          │                │                │
          ▼                ▼                ▼
     Symptoms         Prevention       Recommended
                                         Vaccines
          │                                │
          ▼                                ▼
      Treatment                         Vaccine
          │                                │
          ▼                                ▼
       Animal                         Vaccination
```

This gives Liventra a reusable disease knowledge layer.

---

# 6. Why Disease Needs Its Own Collection

Without a Disease master collection, treatment records would have to store disease names repeatedly.

Example of bad duplication:

```text
Treatment 1:
diseaseName = FMD

Treatment 2:
diseaseName = Foot & Mouth Disease

Treatment 3:
diseaseName = Foot and Mouth
```

This creates inconsistent data.

With Disease:

```text
Disease
_id = D001
diseaseName = Foot and Mouth Disease
```

then:

```text
Treatment.diseaseIds = [D001]
```

All records reference the same disease definition.

---

# 7. Field-by-Field Business Logic

# 7.1 `_id`

### Type

MongoDB `ObjectId`

### Required

Yes

### Purpose

Unique identifier for the disease master record.

Example:

```text
ObjectId(...)
```

---

# 7.2 `diseaseCode`

### Type

String

### Required

Yes

### Example

```text
DIS-FMD-001
```

### Purpose

Human-readable unique identifier for the disease.

Uses:

```text
Search
Reports
Treatment records
Vaccination planning
Analytics
Audit
```

---

# 8. Disease Code Generation

Recommended format:

```text
DIS-{SHORTCODE}-{SEQUENCE}
```

Examples:

```text
DIS-FMD-001
DIS-BRD-001
DIS-MAST-001
```

The backend should generate/validate uniqueness.

The frontend should not be trusted to generate unique codes.

---

# 9. `diseaseName`

### Type

String

### Required

Yes

### Example

```text
Foot and Mouth Disease
```

### Purpose

Official display name of the disease.

Used in:

```text
Disease list
Treatment form
Health history
Reports
Analytics
Notifications
```

---

# 10. Disease Name Normalization

Avoid creating multiple disease records for the same condition because of spelling variations.

Bad:

```text
Foot and Mouth Disease
Foot & Mouth Disease
FMD
Foot-and-Mouth Disease
```

These may all represent the same disease.

Instead:

```text
One canonical Disease record
```

with:

```text
short/search terms
```

handled through the UI/search layer if necessary.

The current finalized model does not include a separate alias field.

---

# 11. `category`

### Type

Enum

### Allowed Values

```text
Viral
Bacterial
Parasitic
Nutritional
Metabolic
Reproductive
Other
```

These are the finalized categories. fileciteturn14file0

### Purpose

Classifies the disease.

---

# 12. Category Examples

Illustrative examples:

```text
Viral
→ FMD

Bacterial
→ Bacterial infection

Parasitic
→ Parasitic infestation

Nutritional
→ Nutrient-deficiency-related condition

Metabolic
→ Metabolic disorder

Reproductive
→ Reproductive disorder

Other
→ Other condition
```

The exact disease-category mapping should be validated against the project's clinical/veterinary knowledge base.

---

# 13. Why Category Is Useful

Category supports:

```text
Filtering
Reporting
Analytics
Disease dashboards
Prevention planning
Research
```

Example:

```text
Show all Parasitic diseases.
```

---

# 14. `symptoms[]`

### Type

Array of strings

### Required

Recommended

### Example

```text
[
    "Fever",
    "Excessive salivation",
    "Mouth lesions",
    "Lameness"
]
```

### Purpose

Stores commonly associated symptoms/signs of the disease.

---

# 15. Why Symptoms Are an Array

A disease can have multiple symptoms.

Therefore:

```text
symptoms[]
```

is appropriate.

Example:

```text
Disease
   |
   +--- Fever
   +--- Reduced appetite
   +--- Cough
   +--- Nasal discharge
```

---

# 16. Symptoms Are Reference Information, Not Diagnosis

This is an important business rule.

If a disease has:

```text
symptoms:
[
    Fever,
    Cough
]
```

that does **not** mean:

```text
Animal with fever + cough = this disease
```

Symptoms can overlap across diseases.

Therefore, the Disease model should support:

```text
Reference / education
```

rather than automatically declaring a diagnosis.

---

# 17. `description`

### Type

String

### Required

Optional

### Purpose

Provides a general description of the disease.

Example:

```text
A contagious livestock disease characterized by...
```

Uses:

```text
Disease details page
Farmer education
Reports
Reference
```

---

# 18. `prevention`

### Type

String

### Required

Optional

### Purpose

Stores general prevention guidance/configuration for the disease.

Example:

```text
Vaccination, biosecurity, isolation of affected animals.
```

The exact content should be based on the project's approved veterinary information.

---

# 19. Prevention vs Vaccination

`prevention` is general disease-level information.

Actual vaccination events belong to:

```text
Vaccination
```

Therefore:

```text
Disease.prevention
=
General prevention guidance

Vaccination
=
Actual preventive event
```

---

# 20. `recommendedVaccineIds[]`

### Type

Array of ObjectId references to Vaccine

### Required

Optional / at least one for vaccine-preventable diseases

### Purpose

Connects the disease to recommended vaccines.

Example:

```text
Disease:
FMD

recommendedVaccineIds:
[
    FMD Vaccine
]
```

---

# 21. Why `recommendedVaccineIds[]` Is an Array

A disease may have:

```text
multiple applicable vaccine options
```

depending on:

```text
farm protocol
vaccine product
animal context
veterinary recommendation
```

Therefore:

```text
recommendedVaccineIds[]
```

is more flexible than a single `recommendedVaccineId`.

---

# 22. Disease ↔ Vaccine Relationship

The relationship becomes:

```text
Disease
   ↓
recommendedVaccineIds[]

Vaccine
   ↓
targetDiseaseIds[]
```

This is conceptually bidirectional.

From Disease:

```text
What vaccines are recommended?
```

From Vaccine:

```text
What diseases does this vaccine target?
```

---

# 23. Reference Integrity

Every:

```text
recommendedVaccineIds[]
```

value should reference an existing Vaccine.

Backend should reject invalid references.

Example:

```text
recommendedVaccineIds:
[
    "nonexistent-id"
]
```

must not be accepted.

---

# 24. Important Circular Reference Consideration

Disease references Vaccine:

```text
Disease.recommendedVaccineIds[]
```

Vaccine references Disease:

```text
Vaccine.targetDiseaseIds[]
```

This creates a **logical bidirectional relationship**.

That is acceptable.

However, the backend should avoid blindly requiring both documents to be updated in one operation every time.

Recommended:

```text
Create Disease
      ↓
Create/attach Vaccine
      ↓
Maintain relationship consistency
```

The service layer should manage relationship updates.

---

# 25. `isActive`

### Type

Boolean

### Required

Yes

### Example

```text
true
```

### Purpose

Controls whether the disease is currently active in the master catalog.

---

# 26. Active vs Inactive Disease

If:

```text
isActive = true
```

the disease can be selected for:

```text
Treatment
Health records
Reports
Disease reference
```

If:

```text
isActive = false
```

it should generally not be selectable for new routine records.

Historical records should remain valid.

---

# 27. Why Not Delete Diseases?

Suppose:

```text
Disease D001
```

is referenced by many Treatment records.

Hard deletion could create:

```text
Treatment.diseaseIds → broken reference
```

Therefore:

```text
isActive = false
```

is safer than deleting the master disease.

---

# 28. `createdAt`

### Type

Date

### Required

Automatic

### Purpose

Stores when the Disease master record was created.

---

# 29. `updatedAt`

### Type

Date

### Required

Automatic

### Purpose

Stores when the Disease master record was last modified.

---

# 30. Master Data Lifecycle

Recommended:

```text
Create Disease
      ↓
Active
      ↓
Used in Treatment / Prevention
      ↓
May be updated
      ↓
May become Inactive
      ↓
Historical references remain
```

Avoid:

```text
Create
 ↓
Delete
```

when references already exist.

---

# 31. Disease Creation Workflow

```text
Admin / Authorized User
        ↓
Create Disease
        ↓
Disease Code
        ↓
Disease Name
        ↓
Category
        ↓
Symptoms
        ↓
Description
        ↓
Prevention
        ↓
Recommended Vaccines
        ↓
Activate
```

---

# 32. Disease and Treatment

The finalized Treatment model contains:

```text
diseaseIds[]
```

Therefore:

```text
Disease
   ↓
Disease._id
   ↓
Treatment.diseaseIds[]
   ↓
Animal
```

This allows one treatment event to be associated with multiple disease references.

The finalized Treatment model includes `diseaseIds[]`. fileciteturn14file0

---

# 33. Why Treatment Uses `diseaseIds[]`

One treatment event may involve:

```text
multiple diagnosed conditions
```

or a combined clinical situation.

Therefore:

```text
diseaseIds[]
```

is more flexible than a single `diseaseId`.

---

# 34. Disease + Treatment Example

Disease master:

```text
D001:
FMD
```

Treatment:

```text
animalId = AN-001

diseaseIds:
[
    D001
]
```

This means:

```text
Animal AN-001
was treated in relation to Disease D001.
```

---

# 35. Disease and Animal

The current Disease model does not contain:

```text
animalIds[]
```

This is intentional.

Why?

Because storing all animals inside Disease creates a large many-to-many relationship inside the master document.

Instead:

```text
Treatment.diseaseIds[]
```

and other event records identify which animals are associated with the disease.

---

# 36. Disease and Health Timeline

Animal health timeline can derive disease events from:

```text
Treatment
```

Example:

```text
Animal A
   ↓
Treatment
   ↓
Disease = FMD
```

The Disease master provides:

```text
Symptoms
Category
Description
Prevention
```

---

# 37. Disease and Vaccination

Vaccination does not need:

```text
diseaseId
```

because:

```text
Vaccination
   ↓
vaccineId
   ↓
Vaccine
   ↓
targetDiseaseIds[]
```

Therefore disease context can be derived from the selected vaccine.

---

# 38. Disease and Prevention Workflow

A disease page can show:

```text
Disease Information
        ↓
Symptoms
        ↓
Prevention
        ↓
Recommended Vaccines
        ↓
Schedule Vaccination
```

This creates a farmer-friendly health workflow.

---

# 39. Disease and Notification

Disease itself does not need to store notification data.

But Disease can be involved in notifications through:

```text
Treatment
Vaccination
Health alerts
```

Example:

```text
Animal diagnosed with disease
        ↓
Treatment
        ↓
Notification to Farm Manager
```

The current Notification model supports entity references for related records. fileciteturn14file17

---

# 40. Disease and Reports

The finalized Report model supports:

```text
reportType = Disease
```

Therefore Liventra can generate:

```text
Disease Report
```

Examples:

```text
Disease occurrence
Disease treatment history
Disease category summary
Disease-wise animal count
```

The actual animal-level occurrence should be derived from treatment/health event data, not from the Disease master alone.

---

# 41. Disease Report Logic

Suppose:

```text
Disease = FMD
```

and Treatment records contain:

```text
Animal A → FMD
Animal B → FMD
Animal C → FMD
```

Then:

```text
FMD affected/treatment-associated animals = 3
```

This is derived from event records.

The Disease document itself remains a master definition.

---

# 42. Disease Analytics

Possible dashboard metrics:

```text
Diseases by category
Most common disease
Animals treated by disease
Disease treatment trend
Disease-wise vaccination coverage
```

These should be calculated from:

```text
Disease
+
Treatment
+
Vaccination
```

as appropriate.

---

# 43. Symptoms Search

The system may allow:

```text
Search disease by symptom
```

Example:

```text
Symptom:
Cough
```

Potential results:

```text
Disease A
Disease B
Disease C
```

But this is a reference/search feature, not an automated diagnosis.

---

# 44. Important Clinical Safety Logic

Liventra should not claim:

```text
Symptom X → Disease Y
```

as a definitive diagnosis unless a validated clinical decision-support model is explicitly implemented and clinically evaluated.

For the current project:

```text
Disease
=
Reference/master information

Treatment
=
Recorded clinical management
```

---

# 45. Disease Code Uniqueness

Recommended:

```text
diseaseCode unique
```

Example:

```text
DIS-FMD-001
```

cannot be duplicated.

---

# 46. Disease Name Uniqueness

Recommended to prevent duplicate master records:

```text
diseaseName unique
```

or normalized uniqueness.

For example:

```text
"Foot and Mouth Disease"
```

and:

```text
"foot and mouth disease"
```

should not accidentally create two disease definitions.

---

# 47. Duplicate Disease Prevention

Before creating a new Disease, backend can search:

```text
normalized diseaseName
```

and potentially:

```text
diseaseCode
```

If a matching active disease already exists:

```text
Prevent duplicate
```

or ask the administrator to confirm.

---

# 48. Disease Editing

Authorized users may update:

```text
symptoms
description
prevention
recommended vaccines
```

But changes should be audited.

Especially important:

```text
category
diseaseName
recommendedVaccineIds
prevention
```

because these fields can affect system behavior and user interpretation.

---

# 49. Disease Master Data and Historical Records

Suppose:

```text
Disease:
FMD
```

has been used by 500 Treatment records.

Later:

```text
description
```

is updated.

Historical Treatment records should still reference the same disease identity.

If the disease itself has fundamentally changed in meaning, a new Disease record may be safer than changing the identity of the old one.

---

# 50. Disease Deactivation

When:

```text
isActive = false
```

the disease should generally:

```text
Disappear from new treatment selection
Remain visible in historical records
Remain available for reports
Remain resolvable through old references
```

This preserves historical integrity.

---

# 51. Recommended Authorization

### Admin

Can:

```text
Create
Update
Deactivate
View
```

### Farm Owner / Manager

Can generally:

```text
View
Use disease references in health workflows
```

according to final permission matrix.

### Worker

Can:

```text
View
Select disease during authorized treatment entry
```

if allowed.

Only authorized users should modify the Disease master catalog.

---

# 52. CRUD Business Rules

## Create

Requires:

```text
diseaseCode
diseaseName
category
```

and configured optional information.

---

## Read

Authorized users can view active and historical disease definitions.

---

## Update

Authorized users can update master information.

Important changes should generate AuditLog.

---

## Delete

Prefer:

```text
isActive = false
```

instead of hard deletion if referenced by Treatment/Vaccine.

---

# 53. Validation Rules

| Field | Validation |
|---|---|
| _id | MongoDB ObjectId |
| diseaseCode | Required, unique |
| diseaseName | Required |
| category | Controlled enum |
| symptoms | Array of strings |
| description | Optional |
| prevention | Optional |
| recommendedVaccineIds | Valid Vaccine references |
| isActive | Boolean |
| createdAt | Automatic |
| updatedAt | Automatic |

---

# 54. Symptoms Validation

Recommended:

```text
symptoms must be an array
```

Each item:

```text
non-empty string
```

Avoid:

```text
[
    "",
    null,
    123
]
```

The frontend should support adding/removing individual symptoms.

---

# 55. Recommended Vaccine Validation

Every:

```text
recommendedVaccineIds[]
```

should:

```text
exist
```

and ideally reference a valid Vaccine.

If a vaccine becomes inactive, historical Disease relationships can remain, while the UI can filter inactive recommendations according to business rules.

---

# 56. Bidirectional Relationship Maintenance

Suppose admin adds:

```text
Vaccine V001
```

to:

```text
Disease D001.recommendedVaccineIds
```

If the system maintains bidirectional consistency, it should also ensure:

```text
Disease D001
```

appears in:

```text
Vaccine V001.targetDiseaseIds
```

This should be handled by a service layer rather than requiring users to edit both sides manually.

---

# 57. Transaction for Relationship Update

When maintaining both sides:

```text
BEGIN

Update Disease
Update Vaccine

COMMIT
```

If one update fails:

```text
ROLLBACK
```

This prevents:

```text
Disease says Vaccine A
but Vaccine does not say Disease
```

from persisting unintentionally.

---

# 58. Index Recommendations

Useful indexes:

```javascript
diseaseCode: 1
diseaseName: 1
category: 1
isActive: 1
recommendedVaccineIds: 1
```

Potential compound index:

```javascript
{
    isActive: 1,
    category: 1
}
```

Useful for:

```text
Active diseases by category
```

---

# 59. Search

Disease search should support:

```text
diseaseCode
diseaseName
category
symptoms
```

Example:

```text
Search:
FMD
```

returns:

```text
Foot and Mouth Disease
```

Example:

```text
Search:
Fever
```

can return diseases whose symptom list contains:

```text
Fever
```

---

# 60. Disease Detail Page

A farmer/manager can see:

```text
Disease Name
Disease Code
Category
Symptoms
Description
Prevention
Recommended Vaccines
Active Status
```

Then:

```text
View related treatment history
```

can be derived through Treatment records.

---

# 61. Disease + Vaccine UI

Disease detail:

```text
Foot and Mouth Disease

Category:
Viral

Symptoms:
• Fever
• Salivation
• Mouth lesions

Prevention:
Vaccination and biosecurity

Recommended Vaccines:
• FMD Vaccine
```

This creates an intuitive workflow.

---

# 62. Disease + Treatment UI

Treatment form:

```text
Animal:
AN-000125

Disease:
FMD

Medicine:
...

Doctor:
...

Start Date:
...

End Date:
...
```

The disease selection comes from:

```text
Disease
```

not free-text input.

This improves consistency.

---

# 63. Disease + Vaccination UI

Vaccination form can use:

```text
Disease → Recommended Vaccine
```

as a selection aid.

Example:

```text
Disease:
FMD

Recommended:
FMD Vaccine
```

Then:

```text
Create Vaccination
```

The actual Vaccination still references:

```text
vaccineId
```

rather than storing diseaseId redundantly.

---

# 64. Disease + Inventory

Disease itself does not consume inventory.

Instead:

```text
Disease
   ↓
Vaccine
   ↓
Vaccination
   ↓
Inventory consumption
```

For treatment:

```text
Disease
   ↓
Treatment
   ↓
Medicine
   ↓
Inventory
```

This creates two distinct health-resource flows.

---

# 65. Disease + Treatment + Medicine

The finalized Treatment model contains:

```text
medicineIds[]
```

Therefore:

```text
Disease
   ↓
Treatment
   ↓
Medicine
```

Example:

```text
Disease:
D001

Treatment:
animalId = A001
diseaseIds = [D001]
medicineIds = [M001, M002]
```

This represents the animal-specific treatment event.

---

# 66. Disease and Reproduction

Because the Disease category includes:

```text
Reproductive
```

the system can represent reproductive-related diseases.

However, Pregnancy/Breeding remain separate models.

Therefore:

```text
Disease
=
Health condition

Pregnancy/Breeding
=
Reproductive event/state
```

Do not merge them.

---

# 67. Disease and Health Status

Animal has:

```text
healthStatus:
Healthy
Sick
Under Treatment
Quarantine
```

Disease is the detailed condition.

Example:

```text
Animal.healthStatus = Under Treatment

Treatment.diseaseIds = [D001]

Disease D001 = FMD
```

This gives:

```text
High-level animal state
+
Specific disease context
```

---

# 68. Disease and Quarantine

If an animal is:

```text
healthStatus = Quarantine
```

the system may show disease/treatment information associated with that animal.

However:

```text
Quarantine ≠ Disease
```

Quarantine is an animal health-management status.

Disease is a condition/reference.

---

# 69. Disease and AuditLog

Important audit events:

```text
Disease Created
Disease Updated
Disease Activated
Disease Deactivated
Category Changed
Symptoms Changed
Prevention Changed
Recommended Vaccine Changed
```

Because disease master data influences health workflows, changes should be traceable.

---

# 70. Disease and Reports

Possible reports:

```text
Disease Master List
Disease Category Summary
Disease Treatment Report
Disease Occurrence Report
Disease Prevention Report
Disease-Vaccine Mapping
```

The Report module supports `Disease` as a report type. fileciteturn14file17

---

# 71. Disease Occurrence vs Disease Master

Important:

```text
Disease master
=
What is the disease?
```

while:

```text
Treatment
=
Which animal was treated for it?
```

Therefore, if a report says:

```text
FMD cases = 25
```

the count should come from relevant event records such as Treatment, not:

```text
Disease document count
```

because there may be only one Disease master document for FMD.

---

# 72. Disease Analytics

Possible metrics:

```text
Top diseases by treatment count
Disease count by month
Disease count by farm
Disease category distribution
Disease-associated animal count
Vaccination coverage for preventable diseases
```

These are derived analytics.

---

# 73. Example — Disease Master

```javascript
{
    diseaseCode: "DIS-FMD-001",

    diseaseName: "Foot and Mouth Disease",

    category: "Viral",

    symptoms: [
        "Fever",
        "Excessive salivation",
        "Mouth lesions",
        "Lameness"
    ],

    description:
        "A contagious livestock disease.",

    prevention:
        "Vaccination and appropriate biosecurity practices.",

    recommendedVaccineIds: [
        "VAC-FMD-001"
    ],

    isActive: true
}
```

Values are illustrative; clinical content should be based on the project's approved veterinary source.

---

# 74. Example — Treatment Relationship

```javascript
{
    animalId: "AN-000125",

    diseaseIds: [
        "DIS-FMD-001"
    ],

    medicineIds: [
        "MED-001"
    ],

    startDate: "2026-08-17",

    status: "Active"
}
```

This means:

```text
Animal AN-000125
has a treatment associated with Disease DIS-FMD-001.
```

---

# 75. Example — Disease to Vaccine

```text
Disease:
DIS-FMD-001

recommendedVaccineIds:
[
    VAC-FMD-001
]
```

Vaccine:

```text
VAC-FMD-001

targetDiseaseIds:
[
    DIS-FMD-001
]
```

---

# 76. Example — Inactive Disease

```text
Disease:
D001

isActive:
false
```

Result:

```text
Cannot normally select for new treatment
```

but:

```text
Old Treatment records remain readable
Reports remain valid
Historical references remain resolvable
```

---

# 77. Error Scenario — Invalid Vaccine Reference

Input:

```text
recommendedVaccineIds:
[
    "invalid-id"
]
```

Backend:

```text
Reject
```

because the reference is invalid.

---

# 78. Error Scenario — Duplicate Disease

Existing:

```text
DIS-FMD-001
Foot and Mouth Disease
```

User creates:

```text
DIS-FMD-002
Foot & Mouth Disease
```

The system should detect a potential duplicate and ask the administrator to review rather than silently creating another master record.

---

# 79. Error Scenario — Deactivating Referenced Disease

If:

```text
Disease D001
```

is already referenced by Treatment records:

```text
isActive = false
```

is preferred over:

```text
DELETE
```

---

# 80. Error Scenario — Removing Recommended Vaccine

If:

```text
Vaccine V001
```

is removed from a Disease's recommended list, historical Vaccination records referencing V001 should remain unchanged.

This only changes the current recommendation relationship.

---

# 81. Historical Relationship Integrity

Important distinction:

```text
Current recommendation
```

can change.

Example:

```text
Disease D001
recommendedVaccineIds:
[V001]
```

Later:

```text
[V002]
```

This does not mean:

```text
Old vaccinations using V001 were invalid.
```

It only means the current master recommendation has changed.

AuditLog should preserve the change.

---

# 82. Disease Versioning Consideration

The current finalized model does not contain:

```text
version
```

Therefore, do not invent a versioning system in the DB model.

For the current project:

```text
AuditLog
+
updatedAt
+
controlled master-data updates
```

provide change tracking.

If thesis evaluation later requires reproducible historical clinical rules, versioning can be introduced as a separate enhancement.

---

# 83. Service-Layer Responsibilities

Recommended services:

```text
diseaseService.create()
diseaseService.update()
diseaseService.deactivate()
diseaseService.attachVaccine()
diseaseService.detachVaccine()
diseaseService.getRecommendedVaccines()
diseaseService.search()
```

This keeps relationship/business rules out of simple controllers.

---

# 84. Relationship Update Workflow

When attaching a Vaccine:

```text
Disease D001
      ↓
Attach Vaccine V001
      ↓
Update Disease.recommendedVaccineIds
      ↓
Update Vaccine.targetDiseaseIds
      ↓
Audit
```

When detaching:

```text
Remove relationship
      ↓
Update both sides
      ↓
Audit
```

---

# 85. Security Considerations

Disease master data can influence:

```text
Treatment
Vaccination recommendations
Health information
Reports
```

Therefore, only authorized users should modify it.

Regular workers should not be able to arbitrarily change:

```text
Disease category
Symptoms
Prevention
Recommended vaccines
```

---

# 86. Final Business Rules Checklist

```text
[✓] Disease is master/reference data
[✓] diseaseCode should be unique
[✓] diseaseName should be normalized to avoid duplicates
[✓] category is controlled by enum
[✓] symptoms is an array of strings
[✓] Symptoms are reference information, not definitive diagnosis
[✓] description provides general disease information
[✓] prevention provides general prevention guidance
[✓] recommendedVaccineIds references Vaccine
[✓] Disease ↔ Vaccine relationship is logically bidirectional
[✓] Invalid vaccine references must be rejected
[✓] isActive controls current availability
[✓] Deactivation is preferred over deletion
[✓] Treatment.diseaseIds references Disease
[✓] One treatment can reference multiple diseases
[✓] Disease does not need animalIds[]
[✓] Animal disease association comes from event records
[✓] Vaccination disease context can be derived through Vaccine
[✓] Disease category supports analytics
[✓] Disease supports reporting
[✓] Historical treatment references must remain valid
[✓] Current vaccine recommendations can change without invalidating historical vaccination
[✓] Master-data changes should be audited
[✓] Only authorized users should modify disease master data
[✓] Relationship updates should be handled by service logic
[✓] Multi-document relationship updates should be transaction-safe where required
```

---

# 87. Final Status

**Model:** `Disease`  
**Status:** ✅ Finalized  
**Business Logic:** ✅ Documented  
**Master Data:** ✅  
**Treatment Integration:** ✅  
**Vaccine Integration:** ✅  
**Vaccination Integration:** ✅  
**Health Timeline Integration:** ✅  
**Reporting:** ✅  
**Auditability:** ✅  
**Historical Integrity:** ✅  

**Next Model:** `Medicine`
