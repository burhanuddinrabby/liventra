

# ✅ Industry Level ER Diagram

আমাদের Project-এ **Core Entity** আর **Supporting Entity** আলাদা হবে।

```text
                     ┌────────────┐
                     │    Farm    │
                     └─────┬──────┘
                           │
        ┌──────────────────┼───────────────────┐
        │                  │                   │
        ▼                  ▼                   ▼
   FarmUser            Animal             Inventory
        │                  │                   │
        │                  │                   │
        ▼                  ▼                   ▼
      User              Breed            InventoryItem
                           │
                           │
      ┌────────────────────┼────────────────────────┐
      │                    │                        │
      ▼                    ▼                        ▼
 FeedPlan            HealthRecord           ProductionRecord
      │                    │                        │
      ▼                    ▼                        ▼
FeedFormula        Vaccination           MilkRecord
      │                    │
      ▼                    ▼
FeedIngredient      DiseaseRecord

                    Animal
                      │
        ┌─────────────┼──────────────┐
        ▼             ▼              ▼
 WeightRecord   BreedingRecord   PregnancyRecord

                    Farm
                      │
          ┌───────────┼─────────────┐
          ▼           ▼             ▼
      Expense      Income       Report
```

---

# Database Architecture

আমরা ৪ Layer ব্যবহার করব।

## Layer 1 — Master Data

যেগুলো খুব কম Change হবে।

```text
Breed

FeedIngredient

Disease

Medicine

Vaccine

Roles

Permissions
```

---

## Layer 2 — Transaction Data

প্রতিদিন Change হবে।

```text
Animal

MilkRecord

FeedRecord

WeightRecord

Expense

Income

Inventory

Vaccination

Treatment
```

---

## Layer 3 — Analytics

```text
Dashboard

Reports

Growth Analysis

Feed Analysis

Profit Analysis
```

---

## Layer 4 — Future IoT

```text
RFID Device

Bolus Device

Smart Collar

Milk Meter

Sensor Data
```

---

# এখন আসল ER Diagram

---

## 1️⃣ Farm

```text
Farm

PK _id

name

ownerId

address

district

phone

farmType

createdAt
```

Relationship

```
1 Farm

↓

Many Users

Many Animals

Many Employees

Many Reports

Many Inventories
```

---

## 2️⃣ User

```text
User

PK _id

farmId FK

roleId FK

name

phone

email

password
```

Relationship

```
Farm

1

↓

Many Users
```

---

## 3️⃣ Animal ⭐

এটাই Core Entity।

```text
Animal

PK _id

farmId FK

breedId FK

animalCode

qrCode

rfid

name

gender

purpose

status

birthDate

purchaseDate

purchasePrice
```

Relationship

```
Animal

1

↓

Many Milk Records

Many Weight Records

Many Vaccinations

Many Treatments

Many Feed Records

Many Pregnancies
```

---

## 4️⃣ Breed

```text
Breed

PK _id

breedName

country

type

averageMilk

averageWeight

diseaseResistance
```

Relationship

```
Breed

1

↓

Many Animals
```

---

## 5️⃣ Feed Formula

```text
FeedFormula

PK _id

breedId FK

stage

purpose

season

remarks
```

---

## 6️⃣ Feed Ingredient

```text
FeedIngredient

PK _id

formulaId FK

ingredient

percentage

protein

energy

price
```

Relationship

```
Feed Formula

1

↓

Many Ingredients
```

---

## 7️⃣ Feed Record

```text
FeedRecord

PK _id

animalId FK

formulaId FK

date

quantity

cost
```

Relationship

```
Animal

1

↓

Many Feed Records
```

---

## 8️⃣ Vaccination

```text
Vaccination

PK _id

animalId FK

vaccine

date

nextDue

doctorId FK
```

---

## 9️⃣ Disease Record

```text
DiseaseRecord

PK _id

animalId FK

doctorId FK

disease

medicine

treatment

date
```

---

## 🔟 Doctor

```text
Doctor

PK _id

name

phone

specialization
```

---

## 11️⃣ Milk Record

```text
MilkRecord

PK _id

animalId FK

date

morningMilk

eveningMilk

totalMilk
```

---

## 12️⃣ Weight Record

```text
WeightRecord

PK _id

animalId FK

date

weight
```

---

## 13️⃣ Breeding

```text
BreedingRecord

PK _id

animalId FK

heatDate

inseminationDate

expectedCalving

status
```

---

## 14️⃣ Pregnancy

```text
PregnancyRecord

PK _id

animalId FK

pregnancyDate

expectedDelivery

actualDelivery
```

---

## 15️⃣ Inventory

```text
Inventory

PK _id

farmId FK

itemName

category

quantity

unit

minimumStock
```

---

## 16️⃣ Expense

```text
Expense

PK _id

farmId FK

category

amount

date
```

---

## 17️⃣ Income

```text
Income

PK _id

farmId FK

source

amount

date
```

---

# 🔥 Final High-Level Relationship

```text
Farm
├── Users
├── Animals
│   ├── Breed
│   ├── Feed Records
│   │     └── Feed Formula
│   │             └── Feed Ingredients
│   ├── Milk Records
│   ├── Weight Records
│   ├── Vaccinations
│   ├── Disease Records
│   ├── Breeding Records
│   └── Pregnancy Records
├── Inventory
├── Expenses
├── Income
└── Reports
```

---

# 📌 আমার মতে ER Diagram-টা ৩ ভাগে করা উচিত

একটা Image-এ ১৭–২০টা Entity দিলে Defense-এ বোঝা কঠিন হবে। তাই আমরা Industry Practice Follow করব:

1. **ERD-1 (Core System):** Farm, User, Animal, Breed, Feed, Milk, Health
2. **ERD-2 (Business System):** Inventory, Finance, Reports, Employees
3. **ERD-3 (Future Integration):** RFID, Bolus, Afimilk, Smart Collar, Sensor Data, IoT Gateway

এতে Documentation-ও পরিষ্কার হবে, আর পরে আমি এগুলোকে **professional Crow's Foot notation**-এ high-resolution diagram হিসেবে তৈরি করতে পারব।
একদম। **ER Diagram** দিয়েই শুরু করা উচিত।

তবে একটা গুরুত্বপূর্ণ Design Decision আগে নিই।

---

# ❌ সাধারণ Student Project

অনেকেই ER Diagram এ এমন করে:

```text
User
Animal
Feed
Milk
Expense
```

মাত্র ৫–৬টা Entity।

এটা ছোট Project-এর জন্য ঠিক আছে, কিন্তু **আমাদের Project-এর জন্য না।**

---

# ✅ Industry Level ER Diagram

আমাদের Project-এ **Core Entity** আর **Supporting Entity** আলাদা হবে।

```text
                     ┌────────────┐
                     │    Farm    │
                     └─────┬──────┘
                           │
        ┌──────────────────┼───────────────────┐
        │                  │                   │
        ▼                  ▼                   ▼
   FarmUser            Animal             Inventory
        │                  │                   │
        │                  │                   │
        ▼                  ▼                   ▼
      User              Breed            InventoryItem
                           │
                           │
      ┌────────────────────┼────────────────────────┐
      │                    │                        │
      ▼                    ▼                        ▼
 FeedPlan            HealthRecord           ProductionRecord
      │                    │                        │
      ▼                    ▼                        ▼
FeedFormula        Vaccination           MilkRecord
      │                    │
      ▼                    ▼
FeedIngredient      DiseaseRecord

                    Animal
                      │
        ┌─────────────┼──────────────┐
        ▼             ▼              ▼
 WeightRecord   BreedingRecord   PregnancyRecord

                    Farm
                      │
          ┌───────────┼─────────────┐
          ▼           ▼             ▼
      Expense      Income       Report
```

---

# Database Architecture

আমরা ৪ Layer ব্যবহার করব।

## Layer 1 — Master Data

যেগুলো খুব কম Change হবে।

```text
Breed

FeedIngredient

Disease

Medicine

Vaccine

Roles

Permissions
```

---

## Layer 2 — Transaction Data

প্রতিদিন Change হবে।

```text
Animal

MilkRecord

FeedRecord

WeightRecord

Expense

Income

Inventory

Vaccination

Treatment
```

---

## Layer 3 — Analytics

```text
Dashboard

Reports

Growth Analysis

Feed Analysis

Profit Analysis
```

---

## Layer 4 — Future IoT

```text
RFID Device

Bolus Device

Smart Collar

Milk Meter

Sensor Data
```

---

# এখন আসল ER Diagram

---

## 1️⃣ Farm

```text
Farm

PK _id

name

ownerId

address

district

phone

farmType

createdAt
```

Relationship

```
1 Farm

↓

Many Users

Many Animals

Many Employees

Many Reports

Many Inventories
```

---

## 2️⃣ User

```text
User

PK _id

farmId FK

roleId FK

name

phone

email

password
```

Relationship

```
Farm

1

↓

Many Users
```

---

## 3️⃣ Animal ⭐

এটাই Core Entity।

```text
Animal

PK _id

farmId FK

breedId FK

animalCode

qrCode

rfid

name

gender

purpose

status

birthDate

purchaseDate

purchasePrice
```

Relationship

```
Animal

1

↓

Many Milk Records

Many Weight Records

Many Vaccinations

Many Treatments

Many Feed Records

Many Pregnancies
```

---

## 4️⃣ Breed

```text
Breed

PK _id

breedName

country

type

averageMilk

averageWeight

diseaseResistance
```

Relationship

```
Breed

1

↓

Many Animals
```

---

## 5️⃣ Feed Formula

```text
FeedFormula

PK _id

breedId FK

stage

purpose

season

remarks
```

---

## 6️⃣ Feed Ingredient

```text
FeedIngredient

PK _id

formulaId FK

ingredient

percentage

protein

energy

price
```

Relationship

```
Feed Formula

1

↓

Many Ingredients
```

---

## 7️⃣ Feed Record

```text
FeedRecord

PK _id

animalId FK

formulaId FK

date

quantity

cost
```

Relationship

```
Animal

1

↓

Many Feed Records
```

---

## 8️⃣ Vaccination

```text
Vaccination

PK _id

animalId FK

vaccine

date

nextDue

doctorId FK
```

---

## 9️⃣ Disease Record

```text
DiseaseRecord

PK _id

animalId FK

doctorId FK

disease

medicine

treatment

date
```

---

## 🔟 Doctor

```text
Doctor

PK _id

name

phone

specialization
```

---

## 11️⃣ Milk Record

```text
MilkRecord

PK _id

animalId FK

date

morningMilk

eveningMilk

totalMilk
```

---

## 12️⃣ Weight Record

```text
WeightRecord

PK _id

animalId FK

date

weight
```

---

## 13️⃣ Breeding

```text
BreedingRecord

PK _id

animalId FK

heatDate

inseminationDate

expectedCalving

status
```

---

## 14️⃣ Pregnancy

```text
PregnancyRecord

PK _id

animalId FK

pregnancyDate

expectedDelivery

actualDelivery
```

---

## 15️⃣ Inventory

```text
Inventory

PK _id

farmId FK

itemName

category

quantity

unit

minimumStock
```

---

## 16️⃣ Expense

```text
Expense

PK _id

farmId FK

category

amount

date
```

---

## 17️⃣ Income

```text
Income

PK _id

farmId FK

source

amount

date
```

---

# 🔥 Final High-Level Relationship

```text
Farm
├── Users
├── Animals
│   ├── Breed
│   ├── Feed Records
│   │     └── Feed Formula
│   │             └── Feed Ingredients
│   ├── Milk Records
│   ├── Weight Records
│   ├── Vaccinations
│   ├── Disease Records
│   ├── Breeding Records
│   └── Pregnancy Records
├── Inventory
├── Expenses
├── Income
└── Reports
```

---

# 📌 আমার মতে ER Diagram-টা ৩ ভাগে করা উচিত

একটা Image-এ ১৭–২০টা Entity দিলে Defense-এ বোঝা কঠিন হবে। তাই আমরা Industry Practice Follow করব:

1. **ERD-1 (Core System):** Farm, User, Animal, Breed, Feed, Milk, Health
2. **ERD-2 (Business System):** Inventory, Finance, Reports, Employees
3. **ERD-3 (Future Integration):** RFID, Bolus, Afimilk, Smart Collar, Sensor Data, IoT Gateway

এতে Documentation-ও পরিষ্কার হবে, আর পরে আমি এগুলোকে **professional Crow's Foot notation**-এ high-resolution diagram হিসেবে তৈরি করতে পারব।
