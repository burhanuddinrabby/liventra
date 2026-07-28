Excellent. 🔥

এখন আমরা **Database Design-এর সবচেয়ে গুরুত্বপূর্ণ অংশ** শুরু করবো।

> **Relationship Mapping (Crow's Foot Design)**

এই Phase শেষ হলে তুমি Backend লিখতে পারবে কোনো confusion ছাড়াই।

---

# 📘 SDS - Phase 3

# Complete Relationship Mapping

প্রথমে বুঝি Relationship কত ধরনের হবে।

---

# 1️⃣ One-to-One (1:1)

কম ব্যবহার হবে।

Example

```text
Animal
    │
    │ 1 : 1
    ▼
Current Status
```

আমাদের Project-এ খুব কম 1:1 লাগবে।

---

# 2️⃣ One-to-Many (1:N)

সবচেয়ে বেশি ব্যবহার হবে।

Example

```text
Farm

1

↓

Many Animals
```

---

# 3️⃣ Many-to-One (N:1)

Example

```text
Many Animals

↓

One Breed
```

---

# 4️⃣ Many-to-Many (M:N)

Example

```text
Animal

↓

Many Vaccines

↓

Many Animals
```

MongoDB-তে Reference ব্যবহার করব।

---

# এখন আসল Relationship শুরু করি

---

# 1. Farm Relationship

```text
                 Farm
                  │
      ┌───────────┼───────────────┐
      │           │               │
      ▼           ▼               ▼

    User       Animal       Inventory
      │                           │
      ▼                           ▼

 Notification                Purchase
      │
      ▼

 Reports
```

Relationship

| Parent | Child     | Type |
| ------ | --------- | ---- |
| Farm   | User      | 1:N  |
| Farm   | Animal    | 1:N  |
| Farm   | Inventory | 1:N  |
| Farm   | Expense   | 1:N  |
| Farm   | Income    | 1:N  |
| Farm   | Report    | 1:N  |

---

# 2. User Relationship

```text
Role

↓

Users

↓

Farm
```

Relationship

| Parent | Child | Type |
| ------ | ----- | ---- |
| Role   | User  | 1:N  |
| Farm   | User  | 1:N  |

---

# 3. Animal Relationship ⭐⭐⭐⭐⭐

এটাই পুরো Database-এর Heart।

```text
                      Animal

                         │

 ───────────────────────────────────────────────

        │        │         │          │

        ▼        ▼         ▼          ▼

      Breed    Milk     Weight     FeedRecord

        │                    │

        ▼                    ▼

  FeedFormula         Vaccination

                             │

                             ▼

                        Treatment

                             │

                             ▼

                        Disease

                             │

                             ▼

                        Medicine

                             │

                             ▼

                      Pregnancy

                             │

                             ▼

                        Breeding
```

---

Relationship Table

| Parent | Child        | Type |
| ------ | ------------ | ---- |
| Breed  | Animal       | 1:N  |
| Animal | FeedRecord   | 1:N  |
| Animal | MilkRecord   | 1:N  |
| Animal | WeightRecord | 1:N  |
| Animal | Vaccination  | 1:N  |
| Animal | Treatment    | 1:N  |
| Animal | Pregnancy    | 1:N  |
| Animal | Breeding     | 1:N  |

---

# 4. Feed Relationship

```text
Feed Formula

↓

Multiple Ingredients

↓

Feed Record

↓

Animal
```

Relationship

| Parent      | Child          | Type |
| ----------- | -------------- | ---- |
| FeedFormula | FeedIngredient | 1:N  |
| FeedFormula | FeedRecord     | 1:N  |
| Animal      | FeedRecord     | 1:N  |

---

# 5. Health Relationship

```text
Animal

↓

Vaccination

↓

Doctor

↓

Treatment

↓

Disease

↓

Medicine
```

Relationship

| Parent   | Child       | Type |
| -------- | ----------- | ---- |
| Animal   | Vaccination | 1:N  |
| Animal   | Treatment   | 1:N  |
| Disease  | Treatment   | 1:N  |
| Medicine | Treatment   | 1:N  |
| Doctor   | Treatment   | 1:N  |
| Doctor   | Vaccination | 1:N  |

---

# 6. Production Relationship

```text
Animal

↓

Milk Record

↓

Weight Record
```

Relationship

| Parent | Child        | Type |
| ------ | ------------ | ---- |
| Animal | MilkRecord   | 1:N  |
| Animal | WeightRecord | 1:N  |

---

# 7. Finance Relationship

```text
Farm

↓

Expense

↓

Income

↓

Reports
```

Relationship

| Parent | Child   | Type |
| ------ | ------- | ---- |
| Farm   | Expense | 1:N  |
| Farm   | Income  | 1:N  |
| Farm   | Report  | 1:N  |

---

# 8. Inventory Relationship

```text
Farm

↓

Inventory

↓

Purchase
```

Relationship

| Parent    | Child     | Type |
| --------- | --------- | ---- |
| Farm      | Inventory | 1:N  |
| Inventory | Purchase  | 1:N  |

---

# 9. IoT Relationship (Future)

```text
Device

↓

Device Assignment

↓

Animal

↓

Sensor Data
```

Relationship

| Parent | Child            | Type |
| ------ | ---------------- | ---- |
| Device | DeviceAssignment | 1:N  |
| Animal | DeviceAssignment | 1:N  |
| Device | SensorData       | 1:N  |
| Animal | SensorData       | 1:N  |

---

# Master ER Relationship

```text
Farm
│
├── Users
│     └── Role
│
├── Animals
│     ├── Breed
│     ├── FeedRecord
│     │      └── FeedFormula
│     │             └── FeedIngredient
│     │
│     ├── MilkRecord
│     ├── WeightRecord
│     ├── Vaccination
│     ├── Treatment
│     │      ├── Disease
│     │      ├── Medicine
│     │      └── Doctor
│     ├── Pregnancy
│     └── Breeding
│
├── Inventory
├── Expense
├── Income
├── Reports
└── Notifications
```

---

# 🚨 কিন্তু এখানে একটা Design Problem আছে

Database Engineer হিসেবে আমি একটা বড় Issue দেখতে পাচ্ছি।

আমরা এখন **Animal** Collection-এ রেখেছি:

* currentWeight
* isPregnant
* isLactating
* isSold
* isDead

আবার আলাদা Collection-এও রেখেছি:

* WeightRecord
* Pregnancy
* Breeding

এতে **Data Duplication** হতে পারে।

---

# 🔥 Professional Solution

আমি Animal Collection-কে **Minimal Entity** বানানোর প্রস্তাব দিচ্ছি।

Animal Collection-এ থাকবে শুধু Identity Information:

* Animal Code
* Breed
* Gender
* Birth Date
* Farm
* Status (Current Overall Status)

আর History এবং পরিবর্তনশীল সব তথ্য আলাদা Collection-এ থাকবে:

* WeightRecord
* PregnancyRecord
* MilkRecord
* FeedRecord
* Treatment
* Vaccination

এতে Database **Normalized**, **Scalable**, এবং **Industry-grade** হবে।

---

# 📌 এখান থেকে আমাদের সামনে দুইটা পথ

### **Option A — Student Project (Easy)**

* ২০–২২টি Collection
* কিছু Current Status Animal Collection-এ থাকবে
* Develop করা সহজ

### **Option B — Enterprise Architecture (Recommended) ⭐**

* ২৫–৩০টি Collection
* Fully Normalized
* Audit Trail
* History Tracking
* Multi-Farm SaaS Ready
* IoT Integration Ready
* Future AI Ready

---

## আমি **Option B** Recommend করছি।

কারণ তোমার Goal শুধু FYP পাশ করা না। তুমি বলেছিলে **Q1-level Research Mindset** নিয়ে Project করতে চাও। তাহলে Database-ও সেই Level-এর হওয়া উচিত।

### আমার Plan:

পরের ধাপে আমরা **MongoDB Collections-গুলোর Field-by-Field Final Schema**, **Indexes**, **Reference Strategy**, এবং **Mongoose Model Design** করব। এরপর REST API Design শুরু করলে Backend Development প্রায় সরাসরি শুরু করা যাবে।
