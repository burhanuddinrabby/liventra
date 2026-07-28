Perfect. 🔥

এখন আমরা **ER Diagram Phase-2** শুরু করব।

এখন আর High-Level না।

এখন **Professional Database Design** করব।

---

# 🎯 PHASE 2

# Complete MongoDB Database Design

আমি অনেক চিন্তা করে একটা Architecture Design করেছি।

**আমাদের Project-এ ২৪টা Collection থাকবে।**

এর কম হলে Future Feature Add করতে সমস্যা হবে।

---

# Database Architecture

```text
                         Smart Livestock System

                                  │

 ─────────────────────────────────────────────────────────────

                    MASTER DATA COLLECTIONS

 ─────────────────────────────────────────────────────────────

Farm

Users

Roles

Permissions

Breed

FeedIngredient

FeedFormula

Disease

Medicine

Vaccine

─────────────────────────────────────────────────────────────

                TRANSACTION COLLECTIONS

─────────────────────────────────────────────────────────────

Animal

FeedRecord

MilkRecord

WeightRecord

Vaccination

Treatment

Pregnancy

Breeding

Expense

Income

Inventory

Purchase

Notification

ActivityLog

─────────────────────────────────────────────────────────────

                 ANALYTICS COLLECTIONS

─────────────────────────────────────────────────────────────

Dashboard

Reports

─────────────────────────────────────────────────────────────

                FUTURE IOT COLLECTIONS

─────────────────────────────────────────────────────────────

Device

SensorData

DeviceAssignment

```

---

# এখন প্রতিটা Collection Design করবো।

---

# COLLECTION 1

# Farm

```javascript
Farm

{
    _id,

    farmName,

    farmCode,

    ownerId,

    address,

    district,

    upazila,

    village,

    latitude,

    longitude,

    phone,

    email,

    establishedDate,

    farmType,

    totalArea,

    description,

    logo,

    status,

    createdAt,

    updatedAt
}
```

---

Relationship

```
Farm

1

↓

Many Users

Many Animals

Many Inventory

Many Expenses

Many Income

Many Reports
```

---

# COLLECTION 2

# User

```javascript
User

{

    _id,

    farmId,

    roleId,

    fullName,

    phone,

    email,

    password,

    profilePicture,

    gender,

    dateOfBirth,

    nid,

    joiningDate,

    salary,

    address,

    isActive,

    lastLogin
}
```

---

Relationship

```
Farm

1

↓

Many Users
```

---

# COLLECTION 3

# Role

```javascript
Role

{

_id,

name,

description

}
```

Example

```
Admin

Farm Owner

Manager

Worker

Veterinarian
```

---

# COLLECTION 4

# Animal ⭐⭐⭐⭐⭐

এটাই পুরো Project-এর Main Collection।

```javascript
Animal

{

_id,

farmId,

breedId,

animalCode,

rfid,

qrCode,

tagNumber,

name,

gender,

purpose,

status,

color,

birthDate,

age,

purchaseDate,

purchasePrice,

purchaseFrom,

currentWeight,

currentLocation,

photo,

remarks,

isPregnant,

isLactating,

isSold,

isDead,

createdAt,

updatedAt

}
```

---

Relationship

```
Animal

↓

Breed

↓

Milk

↓

Weight

↓

Feed

↓

Health

↓

Vaccination

↓

Treatment

↓

Breeding

↓

Pregnancy
```

---

# COLLECTION 5

# Breed

```javascript
Breed

{

_id,

breedName,

origin,

category,

purpose,

averageMilk,

averageWeight,

feedRequirement,

heatResistance,

diseaseResistance,

description

}
```

---

# COLLECTION 6

# FeedIngredient

```javascript
{

_id,

ingredientName,

category,

protein,

fiber,

fat,

energy,

calcium,

phosphorus,

pricePerKg,

unit,

availability

}
```

---

# COLLECTION 7

# FeedFormula

```javascript
{

_id,

breedId,

stage,

purpose,

season,

ingredients:[

{

ingredientId,

percentage

}

],

remarks

}
```

---

Relationship

```
Feed Formula

↓

Multiple Ingredients
```

---

# COLLECTION 8

# FeedRecord

```javascript
{

_id,

animalId,

formulaId,

feedingDate,

quantity,

cost,

feedingTime,

fedBy

}
```

---

# COLLECTION 9

# MilkRecord

```javascript
{

_id,

animalId,

date,

morningMilk,

eveningMilk,

totalMilk,

fat,

protein,

quality

}
```

---

# COLLECTION 10

# WeightRecord

```javascript
{

_id,

animalId,

date,

weight,

remarks

}
```

---

# COLLECTION 11

# Vaccination

```javascript
{

_id,

animalId,

vaccineId,

doctorId,

date,

nextDueDate,

remarks

}
```

---

# COLLECTION 12

# Treatment

```javascript
{

_id,

animalId,

doctorId,

diseaseId,

medicineId,

startDate,

endDate,

cost,

status

}
```

---

# COLLECTION 13

# Disease

```javascript
{

_id,

diseaseName,

symptoms,

description

}
```

---

# COLLECTION 14

# Medicine

```javascript
{

_id,

medicineName,

company,

dose,

description

}
```

---

# COLLECTION 15

# Pregnancy

```javascript
{

_id,

animalId,

inseminationDate,

pregnancyConfirmed,

expectedDelivery,

actualDelivery

}
```

---

# COLLECTION 16

# Breeding

```javascript
{

_id,

animalId,

heatDate,

inseminationDate,

method,

bullName,

status

}
```

---

# COLLECTION 17

# Inventory

```javascript
{

_id,

farmId,

itemName,

category,

quantity,

unit,

minimumStock,

supplier,

purchasePrice

}
```

---

# COLLECTION 18

# Expense

```javascript
{

_id,

farmId,

category,

amount,

date,

description

}
```

---

# COLLECTION 19

# Income

```javascript
{

_id,

farmId,

source,

amount,

date,

description

}
```

---

# COLLECTION 20

# Notification

```javascript
{

_id,

userId,

title,

message,

type,

status,

date

}
```

---

# COLLECTION 21

# Report

```javascript
{

_id,

farmId,

reportType,

generatedBy,

generatedDate,

fileUrl

}
```

---

# COLLECTION 22

# Device (Future)

```javascript
{

_id,

deviceType,

serialNumber,

company,

model,

status

}
```

---

# COLLECTION 23

# SensorData (Future)

```javascript
{

_id,

deviceId,

animalId,

temperature,

activity,

rumination,

waterIntake,

battery,

recordedAt

}
```

---

# COLLECTION 24

# DeviceAssignment (Future)

```javascript
{

_id,

deviceId,

animalId,

assignedDate,

removedDate

}
```

---

# 🔥 এখন একটা জিনিস Notice করো

আমরা এখন আর Student Project Level-এ নেই।

এই Database-এ—

* ✅ Multi Farm Support
* ✅ Multiple Users
* ✅ Feed Formula Engine
* ✅ Health Management
* ✅ Breeding
* ✅ Pregnancy
* ✅ Finance
* ✅ Inventory
* ✅ Notification
* ✅ Future IoT

সব Ready.

---

# 🚀 এখন আমার Recommendation

এখন Database Design-এর **সবচেয়ে গুরুত্বপূর্ণ অংশ** বাকি:

## **Relationship Mapping (Crow's Foot ER Diagram)**

এখানে আমরা প্রতিটি Collection-এর মধ্যে:

* **1 : 1**
* **1 : Many**
* **Many : Many (Reference Collections দিয়ে)**

সব Relationship Professionalভাবে Design করব।

এটাই পরে Mongoose Schema, Populate Query, Aggregation Pipeline, এবং REST API Design-এর ভিত্তি হবে।

**আমার মতে, এটাই আমাদের পরবর্তী Step হওয়া উচিত।**
