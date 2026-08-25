---
tags: [practice, level/intermediate, project]
level: intermediate
topic: Practice
---

# Mini Project Football Academy API

> **Bir jumlada:** FotSpot'ga kirishdan oldin xuddi shu naqshdagi kichik backendni **mustaqil** qur — bu tushunishni eng tez tezlashtiruvchi mashq.

**Before learning:** [[Build a Feature in 10 Steps]], [[NestJS Learning Path]]
**After learning:** [[FotSpot Source Reading Order]]

---

## Loyiha

```text
Football Academy API
```

Entitylar:

```text
User · Player · Coach · Academy · Trial
```

## Bosqich 1 — Skeleton

```text
POST /auth/register
POST /auth/login
GET  /users/me
```

Kerak: [[NestJS Modules]], [[NestJS Controllers]], [[NestJS Services]], [[NestJS DTO]], [[Password Hashing]], [[JWT Authentication]]

Yakunda: ro'yxatdan o'tib, login qilib, o'z profilingni ko'ra olasan.

## Bosqich 2 — CRUD

```text
GET  /players
POST /players
GET  /players/:id

GET  /academies
POST /academies
```

Kerak: [[Prisma]], [[Prisma Schema]], [[Database Relations]], [[NestJS ValidationPipe]]

Yakunda: pagination, filter va `NotFoundException` to'g'ri ishlaydi.

## Bosqich 3 — Trial flow

```text
GET  /trials
POST /trials
POST /trials/:id/result
```

Kerak: [[Database Transactions]], [[Authorization]]

Yakunda: trial natijasi bir nechta yozuvni **atomik** yangilaydi.

## Bosqich 4 — Xavfsizlik

```text
JWT + RBAC + permissions + @Public()
```

Kerak: [[RBAC]], [[Roles and Permissions]], [[Public Routes Decorator]], [[NestJS Guards]]

Yakunda: `admin`, `coach`, `player` rollari turli endpointlarga turlicha kira oladi.

## Bosqich 5 — Infrastructure

```text
Redis cache · rate limiting · BullMQ notification
```

Kerak: [[Redis]], [[Redis Cache]], [[Rate Limiting and Throttling]], [[BullMQ]], [[Queue vs Synchronous Request]]

Yakunda: trial natijasidan keyin xabar **background**da yuboriladi.

## Bosqich 6 — Production tayyorgarligi

```text
Swagger · global exception filter · logging · config validation · Helmet · CORS
```

Kerak: [[Swagger and OpenAPI]], [[NestJS Exception Filters]], [[Logging in NestJS]], [[ConfigModule and Environment Variables]], [[Helmet]], [[CORS]]

## Bosqich 7 — Testlar

```text
unit: service business qoidalari
e2e:  auth flow + guard zanjiri
```

Kerak: [[Testing in NestJS]]

---

## Yakuniy talablar

```text
[ ] Har bir endpoint DTO bilan himoyalangan
[ ] Ochiq endpointlar ATAYLAB @Public()
[ ] Role va permission ishlaydi
[ ] Ownership tekshiruvi bor
[ ] Uzoq ishlar queue'da
[ ] Bir nechta yozuv transaction ichida
[ ] Xatolar yagona formatda
[ ] Swagger to'liq
[ ] Test yozilgan
[ ] .env kodda emas
```

## Nega bu foydali?

```text
FotSpot kodini o'qiyotganda har bir naqsh tanish bo'ladi:
   "Ha, buni men ham yozganman — faqat kattaroq masshtabda."
```

Shundan keyin [[FotSpot Source Reading Order]] bo'yicha real kodga o't.

## Related concepts

[[Build a Feature in 10 Steps]] · [[NestJS Practice Challenges]] · [[NestJS Learning Path]] · [[FotSpot Source Reading Order]] · [[FotSpot Architecture Overview]]
