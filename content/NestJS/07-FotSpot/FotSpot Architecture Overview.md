---
tags: [fotspot, level/beginner, hub]
level: beginner
topic: FotSpot
---

# FotSpot Architecture Overview

> **Bir jumlada:** FotSpot — NestJS + TypeScript REST API; feature-based modullar, global guard zanjiri va aniq ajratilgan infrastructure qatlami.

**Before learning:** [[NestJS Overview]], [[NestJS Modules]]
**After learning:** [[FotSpot Guard Chain]], [[FotSpot Feature Anatomy]], [[FotSpot Source Reading Order]]

> 💡 **Tushunmadingmi? Express'da qo'lda qanday yozilishini ko'r:** [[Express Project Structure Problem]]


---

## 1. Definition

To'liq request oqimi:

```text
Client
  ↓
HTTP Request
  ↓
NestJS Application
  ├── Global ThrottleGuard
  ├── JwtAuthGuard
  ├── RolesGuard
  └── PermissionsGuard
  ↓
Controller
  ↓
DTO / Validation
  ↓
Service
  ├── Prisma → PostgreSQL
  ├── Redis
  ├── BullMQ → Redis
  ├── R2 / S3-compatible Storage
  └── Email / SMS / Telegram
  ↓
Response
  ↓
HttpExceptionFilter
```

Bu diagrammani kodga qarab **o'zing tushuntira olishing** — asosiy maqsad.

## 2. Why — nega shunday qurilgan?

Uchta qaror va sabablari:

```text
1. Feature-based modullar
   → har bir domain o'z boundary'sida, jamoa parallel ishlay oladi

2. Global guardlar (default = secure)
   → yangi endpoint yozilganda himoyani UNUTIB bo'lmaydi
     ([[Public Routes Decorator]])

3. Infrastructure alohida
   → Prisma/Redis/R2 almashtirilishi mumkin bo'lgan detallar
     ([[Domain vs Infrastructure]])
```

## 3. Internal mental model

```text
backend/src/
  main.ts            ← bootstrap, global sozlamalar
  app.module.ts      ← kompozitsiya
  common/            ← guard, filter, decorator, interceptor
  prisma/            ← @Global() PrismaModule
  redis/             ← Redis client
  rate-limit/        ← throttling
  auth/              ← authentication + strategy
  rbac/              ← role & permission
  storage/           ← R2 / S3
  users/  players/  coaches/  academies/
  trials/ recommendations/ media/ moderation/ notifications/
```

Yuqoridagi ro'yxatning birinchi yarmi — **fundament**, ikkinchi yarmi — **domain**.

## 4. Texnologiyalar va ularning roli

| Texnologiya | Roli | Note |
|---|---|---|
| NestJS | application architecture | [[NestJS Overview]] |
| TypeScript | tip xavfsizligi | [[TypeScript for NestJS]] |
| Prisma + PostgreSQL | source of truth | [[Prisma]] |
| Redis | cache, rate limit, queue backend | [[Redis]] |
| BullMQ | background jobs | [[BullMQ]] |
| Passport + JWT | authentication | [[JWT Authentication]] |
| argon2 | parol hashlash | [[Password Hashing]] |
| Socket.IO | real-time | [[WebSockets in NestJS]] |
| Swagger | API contract | [[Swagger and OpenAPI]] |
| Helmet / CORS | HTTP xavfsizlik | [[Helmet]] · [[CORS]] |
| Sentry | error monitoring | [[Sentry]] |
| AWS S3 SDK → R2 | media storage | [[S3 and Cloudflare R2]] |
| Jest | testing | [[Testing in NestJS]] |

## 5. Domain modullari

```text
users            → identity va profil
players          → o'yinchi profillari
coaches          → murabbiylar
academies        → akademiyalar
trials           → sinov jarayoni
recommendations  → scout tavsiyalari
media            → video/rasm  → [[FotSpot Media Moderation]]
moderation       → admin tekshiruvi
notifications    → xabarlar (queue orqali)
```

## 6. Common mistakes

- **Domain modullarni o'qishdan boshlash** → avval `main.ts`, `common/`, `auth/` ([[FotSpot Source Reading Order]]).
- **Prisma/Redis'ni NestJS qismi deb tushunish** → [[Domain vs Infrastructure]].
- **Guard tartibini yodlash, sababini bilmaslik** → [[FotSpot Guard Chain]].
- **Har bir featureni noldan boshqacha yozish** → naqsh bitta: [[Build a Feature in 10 Steps]].

## 7. Practice

1. Yuqoridagi request oqimini yoddan chizib ber.
2. Har bir qutiga FotSpot'dagi bitta aniq fayl/class nomini yoz.
3. Bitta feature tanla va uning barcha bog'liqliklarini diagramma qilib chiz.
4. "Nega guardlar global?" degan savolga 3 jumlada javob yoz.

## 8. Related concepts

[[FotSpot Guard Chain]] · [[FotSpot Feature Anatomy]] · [[FotSpot Auth Architecture]] · [[FotSpot Media Moderation]] · [[Domain vs Infrastructure]] · [[NestJS Request Lifecycle]] · [[NestJS Mental Model]]
