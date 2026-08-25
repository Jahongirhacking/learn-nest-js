---
tags: [database, level/intermediate]
level: intermediate
topic: Database
---

# Database Relations

> **Bir jumlada:** NestJS'ni yaxshi bilish yetmaydi — relational thinking bo'lmasa, schema noto'g'ri chiqadi va query'lar sekinlashadi.

**Before learning:** [[Prisma Schema]]
**After learning:** [[RBAC]], [[Database Transactions]]

> 💡 **Tushunmadingmi? Express'da qo'lda qanday yozilishini ko'r:** [[Express Prisma Setup]]


---

## 1. Definition

Uch xil bog'lanish:

```text
1-1    User ── PlayerProfile          (foreign key + @unique)
1-N    Academy ──< Trial              (ko'p tomonda foreign key)
N-N    User ──< UserRole >── Role     (join model orqali)
```

## 2. Why — nega kerak?

Chunki bog'lanish turini noto'g'ri tanlash keyinchalik tuzatib bo'lmaydigan muammo bo'ladi:

```text
1-1 deb yozilgan, aslida 1-N ekan  → ma'lumot sig'maydi
N-N join modelsiz                   → qo'shimcha maydon qo'sha olmaysan
foreign key yo'q                    → yetim yozuvlar to'planadi
```

## 3. Internal mental model

RBAC zanjirini o'qishni mashq qil:

```text
User
 └── UserRole          (join: userId + roleId)
       └── Role
             └── RolePermission   (join: roleId + permissionId)
                   └── Permission
```

O'qilishi: *"User bir necha rolga ega bo'lishi mumkin; har bir role bir necha permissionga ega."*

Foreign key qayerda turadi:

```text
1-N da  → HAR DOIM "ko'p" tomonda
          Trial.academyId  (Academy'da trialIds ustuni YO'Q)
1-1 da  → bir tomonda + @unique
N-N da  → join jadvalning ikkala ustunida
```

## 4. FotSpot implementation

```text
User 1──1 PlayerProfile        (userId @unique)
User 1──N Session              → [[Session Architecture]]
User N──N Role                 → UserRole
Role N──N Permission           → RolePermission
Academy 1──N Trial
PlayerProfile 1──N Media       → [[FotSpot Media Moderation]]
Trial 1──N Recommendation
```

## 5. Code example

```prisma
// 1-1
model User          { id String @id  player PlayerProfile? }
model PlayerProfile {
  id     String @id @default(uuid())
  userId String @unique                        // ← @unique 1-1 qiladi
  user   User   @relation(fields: [userId], references: [id], onDelete: Cascade)
}

// 1-N
model Academy { id String @id  trials Trial[] }
model Trial {
  id        String  @id @default(uuid())
  academyId String                              // ← FK "ko'p" tomonda
  academy   Academy @relation(fields: [academyId], references: [id])
  @@index([academyId])
}

// N-N qo'shimcha maydon bilan
model UserRole {
  userId     String
  roleId     String
  assignedAt DateTime @default(now())           // ← join modelning foydasi
  assignedBy String?
  user User @relation(fields: [userId], references: [id], onDelete: Cascade)
  role Role @relation(fields: [roleId], references: [id], onDelete: Cascade)
  @@id([userId, roleId])
}
```

O'qish:

```ts
const user = await this.prisma.user.findUnique({
  where: { id },
  include: {
    player: true,
    roles: { include: { role: { include: { permissions: { include: { permission: true } } } } } },
  },
});
```

## 6. Common mistakes

- **N+1 query** — eng ko'p uchraydigan performance xatosi:

```ts
// ✗ 1 + N query
const trials = await prisma.trial.findMany();
for (const t of trials) {
  t.academy = await prisma.academy.findUnique({ where: { id: t.academyId } });
}

// ✓ 1 query
const trials = await prisma.trial.findMany({ include: { academy: true } });
```

- **Chuqur `include`lar** → bitta requestda ulkan JOIN; kerak bo'lsa alohida query yoki `select`.
- **Foreign key ustuniga index qo'ymaslik** → JOIN sekin.
- **N-N ni ikkita massiv bilan taqlid qilish** → butunlik kafolati yo'qoladi.
- **`onDelete` ni tanlamaslik** → default xatti-harakat kutilganidan boshqa bo'lishi mumkin.

## 7. Practice

1. RBAC zanjirini (`User → UserRole → Role → RolePermission → Permission`) yozib, seed qil.
2. Userning barcha permissionlarini bitta query bilan ol.
3. Ataylab N+1 yozib, query loglarini yoq va nechta so'rov ketganini sana.
4. `include` bilan tuzat va farqni o'lchab ko'r.
5. `UserRole`ga `assignedBy` qo'shib, kim kimga role berganini yozib boradigan qil.

## 8. Related concepts

[[Prisma Schema]] · [[Prisma]] · [[RBAC]] · [[Database Transactions]] · [[Session Architecture]]
