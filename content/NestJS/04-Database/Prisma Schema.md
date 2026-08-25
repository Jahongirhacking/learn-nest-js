---
tags: [database, level/intermediate]
level: intermediate
topic: Database
---

# Prisma Schema

> **Bir jumlada:** `schema.prisma` — application domainining yagona haqiqat manbai: undan ham TypeScript tiplar, ham PostgreSQL sxemasi kelib chiqadi.

**Before learning:** [[Prisma]]
**After learning:** [[Prisma Migrations]], [[Database Relations]], [[RBAC]]

---

## 1. Definition

```text
Application domain
       ↓
schema.prisma
       ↓
Migration
       ↓
PostgreSQL
```

Asosiy qurilish bloklari:

```text
model · enum · relation · @id · @unique · @default · @index · @@map
```

## 2. Why — nega kerak?

Schema — bu **dizayn hujjati**, faqat konfiguratsiya emas. U quyidagilarni majburan aniqlashtiradi:

```text
Qaysi maydon majburiy?           (String vs String?)
Qaysi maydon noyob?              (@unique)
Bog'lanish qanday?               (1-1, 1-N, N-N)
Ota yozuv o'chsa nima bo'ladi?   (onDelete)
Qaysi query tez bo'lishi kerak?  (@@index)
```

Bu savollarga kod yozishdan **oldin** javob berish — keyinchalik og'riqli migratsiyalardan qutqaradi.

## 3. Internal mental model

```prisma
model Player {
  id        String   @id @default(uuid())   // primary key
  userId    String   @unique                // 1-1 bog'lanish uchun
  status    Status   @default(ACTIVE)       // enum + default
  birthDate DateTime
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([status, createdAt])               // ro'yxat query'si uchun
}

enum Status { ACTIVE INACTIVE BLOCKED }
```

Muhim tushunchalar:

```text
@unique     → ma'lumot butunligi (bir email — bir user)
@@index     → tezlik (WHERE / ORDER BY ustunlari)
onDelete    → Cascade | SetNull | Restrict
@default    → uuid(), now(), enum qiymati
?           → nullable
```

Index qoidasi: `WHERE`, `ORDER BY` yoki `JOIN`da tez-tez ishlatiladigan ustunlarga index kerak. Lekin har bir index yozish (`INSERT`/`UPDATE`) tezligini pasaytiradi — hammasiga qo'yib chiqish yechim emas.

## 4. FotSpot implementation

```text
model User          → identity
model Role          → [[RBAC]]
model Permission
model UserRole      → many-to-many join
model RolePermission
model PlayerProfile → domain
model Session       → [[Session Architecture]]
model Media         → [[FotSpot Media Moderation]]
model Trial
model Recommendation
```

Ko'p-ko'pga bog'lanishlar aniq `join model` orqali yozilgan (`UserRole`, `RolePermission`) — bu qo'shimcha maydon qo'shish imkonini beradi (masalan `assignedAt`, `assignedBy`).

## 5. Code example

```prisma
model User {
  id           String     @id @default(uuid())
  email        String     @unique
  phone        String?    @unique
  passwordHash String
  status       UserStatus @default(ACTIVE)
  createdAt    DateTime   @default(now())

  roles    UserRole[]
  sessions Session[]
  player   PlayerProfile?

  @@index([status])
}

model PlayerProfile {
  id        String   @id @default(uuid())
  userId    String   @unique
  position  Position
  birthDate DateTime

  user  User    @relation(fields: [userId], references: [id], onDelete: Cascade)
  media Media[]

  @@index([position, birthDate])
}

enum UserStatus { ACTIVE INACTIVE BLOCKED }
enum Position   { GK DF MF FW }
```

## 6. Common mistakes

- **Hamma narsani `String?` qilib qo'yish** → database hech narsani kafolatlamaydi, tekshiruv kodga tushadi.
- **`@unique` ni unutish** → ikkita bir xil email paydo bo'ladi va keyin tozalash kerak.
- **Indexsiz ro'yxat query'lari** → 10 000 yozuvdan keyin sekinlashuv.
- **Barcha ustunga index qo'yish** → yozish sekinlashadi, disk o'sadi.
- **`onDelete` ni o'ylamaslik** → yetim yozuvlar yoki kutilmagan cascade o'chirish.
- **Enum o'rniga free string** → `"activ"`, `"ACTIVE"`, `"active"` aralashib ketadi.
- **Pulni `Float` bilan saqlash** → `Decimal` ishlat.

## 7. Practice

1. `User` va `PlayerProfile` modellarini 1-1 bog'lanish bilan yoz.
2. `UserRole` join modelini qo'shib many-to-many yasab ko'r.
3. `@@index` qo'shishdan oldin va keyin `EXPLAIN ANALYZE` bilan query vaqtini solishtir.
4. `onDelete: Cascade` va `Restrict` farqini amalda sinab ko'r.
5. Enum qo'shib, noto'g'ri qiymat yozishga urin — nima bo'ladi?

## 8. Related concepts

[[Prisma]] · [[Prisma Migrations]] · [[Database Relations]] · [[RBAC]] · [[Session Architecture]] · [[Database Transactions]]
