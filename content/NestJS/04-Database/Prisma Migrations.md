---
tags: [database, level/intermediate]
level: intermediate
topic: Database
---

# Prisma Migrations

> **Bir jumlada:** migration — database sxemasining versiya nazorati; `migrate dev` yangi migratsiya **yaratadi**, `migrate deploy` esa mavjudlarini **qo'llaydi**.

**Before learning:** [[Prisma Schema]]
**After learning:** [[Database Connection URLs]], [[NestJS Application Bootstrap]]

---

## 1. Definition

```bash
# development
npx prisma migrate dev --name add_player_position

# production
npx prisma migrate deploy
```

Farqi:

```text
migrate dev
   = schema o'zgarishi → yangi SQL migration fayli → local DB'ga qo'llash
     (kerak bo'lsa DB'ni reset qilishi ham mumkin!)

migrate deploy
   = mavjud migration fayllarini production DB'ga qo'llash
     (hech narsa generatsiya qilmaydi, hech narsani o'chirmaydi)
```

## 2. Why — nega kerak?

Chunki database sxemasi ham **kod** — u ham versiyalanishi, review qilinishi va takrorlanadigan tarzda qo'llanishi kerak.

```text
migration'siz:  "serverda qo'lda ALTER TABLE qildim"
                → dev/staging/prod bir-biriga o'xshamaydi
migration bilan: git history = database history
```

## 3. Internal mental model

```text
prisma/migrations/
  20260101120000_init/migration.sql
  20260115093000_add_sessions/migration.sql
  ...
```

Prisma database'da `_prisma_migrations` jadvalini yuritadi: qaysi migratsiya qachon qo'llanganini shu yerda ko'radi.

Ish oqimi:

```text
1. schema.prisma o'zgartiriladi
2. npx prisma migrate dev --name <ma'noli_nom>
3. SQL fayl generatsiya qilinadi → git'ga commit
4. CI/CD → npx prisma migrate deploy
```

## 4. FotSpot implementation

FotSpot production startup'ida `migrate deploy` script orqali ishlatiladi:

```text
container start
   ↓
prisma migrate deploy
   ↓
node dist/main.js
```

Migratsiya `DIRECT_URL` orqali bajariladi, application query'lari esa `DATABASE_URL` orqali ketadi — sababi [[Database Connection URLs]] da.

## 5. Code example

```json
// package.json
{
  "scripts": {
    "prisma:generate": "prisma generate",
    "migrate:dev": "prisma migrate dev",
    "migrate:deploy": "prisma migrate deploy",
    "db:seed": "ts-node prisma/seed.ts",
    "start:prod": "prisma migrate deploy && node dist/main.js"
  }
}
```

Xavfsiz migratsiya naqshi (ustun nomini o'zgartirish):

```text
1. Yangi ustun qo'shiladi (nullable)
2. Kod ikkalasiga ham yozadi
3. Ma'lumot ko'chiriladi (backfill)
4. Kod faqat yangisidan o'qiydi
5. Eski ustun o'chiriladi
```

Bir qadamda qilinsa — deploy paytida ishlab turgan eski kod buziladi.

## 6. Common mistakes

- **Production'da `migrate dev` ishlatish** → ma'lumot yo'qolishi mumkin. Faqat `deploy`.
- **Migratsiya fayllarini git'ga qo'shmaslik** → boshqa muhitlar sinxronlanmaydi.
- **Generatsiya qilingan SQL'ni o'qimaslik** → `DROP COLUMN` sezilmay o'tib ketadi.
- **Migratsiyani qo'lda tahrirlash va checksum'ni buzish** → `migrate deploy` xato beradi.
- **Ustun nomini bir qadamda o'zgartirish** → deploy paytida downtime.
- **Backup'siz katta migratsiya qilish.**
- **`prisma generate` ni CI'da unutish** → eski typed client bilan build.

## 7. Practice

1. `PlayerProfile`ga yangi maydon qo'shib `migrate dev` qil, generatsiya qilingan SQL'ni **o'qi**.
2. Ustun nomini o'zgartirish uchun yuqoridagi 5 bosqichli xavfsiz naqshni bajarib ko'r.
3. `prisma migrate status` bilan holatni tekshir.
4. Seed script yozib, role va permissionlarni to'ldir ([[RBAC]]).
5. `start:prod` scriptini yozib, migratsiya avtomatik qo'llanishini sinab ko'r.

## 8. Related concepts

[[Prisma Schema]] · [[Prisma]] · [[Database Connection URLs]] · [[NestJS Application Bootstrap]] · [[RBAC]]
