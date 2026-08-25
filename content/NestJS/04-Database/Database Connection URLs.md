---
tags: [database, level/advanced, infrastructure]
level: advanced
topic: Database
---

# Database Connection URLs

> **Bir jumlada:** `DATABASE_URL` — application query'lari uchun (pooler orqali), `DIRECT_URL` — migratsiya uchun (to'g'ridan-to'g'ri); ikkalasi bir xil emas va sababi connection pooling'da.

**Before learning:** [[Prisma Migrations]], [[ConfigModule and Environment Variables]]
**After learning:** [[Redis]], production deploy

---

## 1. Definition

```prisma
datasource db {
  provider  = "postgresql"
  url       = env("DATABASE_URL")     // application queries → pooler
  directUrl = env("DIRECT_URL")       // migrations → direct connection
}
```

## 2. Why — nega kerak?

Managed PostgreSQL (Supabase, Neon, RDS + PgBouncer) odatda **connection pooler** orqasida turadi.

```text
Application queries
      ↓
DATABASE_URL  →  pooler (PgBouncer)  →  PostgreSQL

Migration CLI
      ↓
DIRECT_URL    →  PostgreSQL (to'g'ridan-to'g'ri)
```

Sabab: pooler `transaction mode`da ishlaganda **session-level** operatsiyalarni qo'llab-quvvatlamaydi. Migratsiya esa aynan shularga muhtoj:

```text
advisory lock   → ikkita migratsiya bir vaqtda ishlamasligi uchun
prepared statements / session state
DDL uzun tranzaksiyalari
```

Pooler orqali migratsiya qilinsa — u osilib qoladi yoki tushunarsiz xato beradi.

## 3. Internal mental model

```text
Nega pooler kerak?
   Serverless / ko'p instance → har biri o'z connectionini ochadi
   PostgreSQL'da max_connections cheklangan (masalan 100)
   → pooler ularni multiplekslaydi

Nega migratsiyaga to'g'ridan-to'g'ri kerak?
   Migratsiyaga session state va lock kerak
   → pooler buni bermaydi
```

Qisqa qoida:

```text
Ko'p, qisqa query   → pooler
Kam, uzun DDL       → direct
```

## 4. FotSpot implementation

FotSpot schema'sida ikkala URL alohida e'lon qilingan.

```text
runtime  (NestJS app)  → DATABASE_URL  (pooled)
CI/CD    (migrate deploy) → DIRECT_URL (direct)
```

Ikkalasi ham secret — kodga yozilmaydi, `.env` yoki platform secrets orqali keladi ([[ConfigModule and Environment Variables]]).

## 5. Code example

```bash
# .env.example  — HAQIQIY qiymatlar bu faylga YOZILMAYDI
DATABASE_URL="postgresql://user:pass@host:6543/db?pgbouncer=true&connection_limit=1"
DIRECT_URL="postgresql://user:pass@host:5432/db"
```

E'tibor ber: port ham har xil (`6543` — pooler, `5432` — PostgreSQL) va pooled URL'da `pgbouncer=true` bayrog'i bor.

```json
{
  "scripts": {
    "start:prod": "prisma migrate deploy && node dist/main.js"
  }
}
```

## 6. Common mistakes

- **Migratsiyani pooled URL orqali ishlatish** → advisory lock xatosi yoki cheksiz kutish.
- **`?pgbouncer=true` ni unutish** → Prisma prepared statement ishlatib, xato beradi.
- **`connection_limit` ni sozlamaslik** → har bir instance ko'p connection ochadi, limit tugaydi.
- **Ikkala URL'ni bir xil qilib qo'yish** → kichik loyihada ishlaydi, yuklamada buziladi.
- **Haqiqiy credentiallarni `.env.example` yoki git'ga qo'yish** → secret sizishi.

## 7. Practice

1. `.env.example` yozib, ikkala URL uchun shablon qo'y (haqiqiy parolsiz).
2. `directUrl`ni schema'ga qo'shib, `migrate deploy` ni sinab ko'r.
3. Ataylab pooled URL bilan migratsiya qilib, xato matnini o'qi.
4. `connection_limit` ni o'zgartirib, parallel so'rovlar xatti-harakatini kuzat.

## 8. Related concepts

[[Prisma Migrations]] · [[Prisma]] · [[ConfigModule and Environment Variables]] · [[Redis]]
