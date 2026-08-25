---
tags: [express/infra, level/intermediate]
level: intermediate
topic: Data and Infrastructure
---

# Express Prisma Setup

> **Bir jumlada:** Prisma kodining o'zi Express'da ham, NestJS'da ham **so'zma-so'z bir xil** — farq faqat `PrismaClient` instance'i qanday yaratilishi va ulashilishida.

**Before learning:** [[Express Service Layer]], [[Express Manual Dependency Wiring]]
**After learning:** [[Prisma]], [[Express Testing]]

**NestJS'dagi mos note:** [[Prisma]] · [[NestJS Providers]]

---

## 1. Muammo

NestJS'da:

```ts
@Global()
@Module({ providers: [PrismaService], exports: [PrismaService] })
export class PrismaModule {}

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit, OnModuleDestroy {
  async onModuleInit() { await this.$connect(); }
  async onModuleDestroy() { await this.$disconnect(); }
}
```

Savollar: `@Global()` nima? `OnModuleInit` nega kerak? Express'da javob oddiy.

## 2. Express'da — singleton fayl

```js
// db.js
const { PrismaClient } = require('@prisma/client');

const prisma = new PrismaClient({
  log: process.env.NODE_ENV === 'production'
    ? ['error', 'warn']
    : ['query', 'error', 'warn'],
});

module.exports = prisma;         // ← BITTA instance, hamma uchun
```

```js
// services/players.service.js
const prisma = require('../db');       // ← xuddi o'sha obyekt
```

**Nega bu ishlaydi?** Node'ning modul tizimi natijani **cache'laydi**: `require('../db')` necha marta yozilsa ham, fayl **bir marta** ishlaydi.

```text
require cache  ≈  DI container'ning singleton scope'i
```

Mana `@Global()` va singleton'ning ma'nosi: **butun app bo'ylab bitta instance**.

## 3. Nega bu muhim

```js
// ✗ XATO — har bir faylda yangi client
// players.service.js
const prisma = new PrismaClient();      // pool #1
// trials.service.js
const prisma = new PrismaClient();      // pool #2
// users.service.js
const prisma = new PrismaClient();      // pool #3
```

**Natija:** har biri o'z connection pool'ini ochadi → PostgreSQL `too many connections` beradi.

Bu xato ayniqsa **hot reload** (nodemon) da yomon: har qayta yuklashda yangi client, eskisi yopilmaydi.

```js
// Development uchun himoya
const globalForPrisma = globalThis;
const prisma = globalForPrisma.prisma ?? new PrismaClient();
if (process.env.NODE_ENV !== 'production') globalForPrisma.prisma = prisma;
module.exports = prisma;
```

NestJS'da bu muammo yo'q — provider bir marta yaratiladi.

## 4. Graceful shutdown

```js
// server.js
const app = require('./app');
const prisma = require('./db');
const redis = require('./redis');

const server = app.listen(process.env.PORT || 3000);

async function shutdown(signal) {
  console.log(`${signal} received, shutting down`);

  server.close(async () => {                 // yangi so'rov qabul qilma
    await prisma.$disconnect();
    await redis.quit();
    process.exit(0);
  });

  setTimeout(() => process.exit(1), 10_000).unref();   // majburiy chiqish
}

process.on('SIGTERM', () => shutdown('SIGTERM'));
process.on('SIGINT',  () => shutdown('SIGINT'));
```

**Bu — `OnModuleDestroy` ning qo'lda yozilgani.**

| Express | NestJS |
|---|---|
| `db.js` faylida `new PrismaClient()` | `PrismaService extends PrismaClient` |
| `require` cache = singleton | provider scope = singleton |
| hamma joyda `require('../db')` | `constructor(private prisma: PrismaService)` |
| qo'lda `process.on('SIGTERM')` | `OnModuleDestroy` + `enableShutdownHooks()` |
| — | `@Global()` — import qilmasdan foydalanish |

## 5. Query kodi — hech qanday farq yo'q

```js
// Express
const player = await prisma.playerProfile.findUnique({
  where: { id },
  include: { user: { select: { id: true, firstName: true } } },
});

// Transaction — AYNAN BIR XIL
await prisma.$transaction(async (tx) => {
  await tx.trial.update({ ... });
  await tx.recommendation.updateMany({ ... });
});
```

```ts
// NestJS — faqat this.prisma
const player = await this.prisma.playerProfile.findUnique({ ... });
```

**Shuning uchun [[Prisma]], [[Prisma Schema]], [[Prisma Migrations]], [[Database Relations]], [[Database Transactions]] notelarining hammasi Express'da ham 100% amal qiladi.**

Prisma — framework emas, infratuzilma ([[Domain vs Infrastructure]]).

## 6. Common mistakes

- **Har bir faylda `new PrismaClient()`** → connection portlashi.
- **`$disconnect()` ni chaqirmaslik** → graceful shutdown yo'q, ulanishlar osilib qoladi.
- **Nodemon'da global himoya qo'ymaslik** → development'da connection oqishi.
- **Handler'da to'g'ridan-to'g'ri `prisma` chaqirish** → service qatlami yo'qoladi ([[Express Service Layer]]).
- **`select` ishlatmaslik** → `passwordHash` javobga chiqadi.
- **Prisma xatolarini tarjima qilmaslik** → `P2002` 500 bo'lib chiqadi ([[Express Error Handling Middleware]]).
- **`prisma.$queryRaw` ni string konkatenatsiya bilan yozish** → SQL injection.

## 7. Practice problems

### Problem 1 — `db.js`
Singleton `db.js` yozib, uch xil service'dan `require` qil.
`console.log(prisma === require('../db'))` bilan bir xilligini tasdiqla.

### Problem 2 — xatoni takrorla
Ikkita faylda alohida `new PrismaClient()` yarat.
PostgreSQL'da `SELECT count(*) FROM pg_stat_activity;` bilan connection sonini kuzat.

### Problem 3 — nodemon
Nodemon bilan ishga tushirib, faylni 5 marta saqla. Connection soni qanday o'zgardi?
Keyin `globalThis` himoyasini qo'shib, qayta tekshir.

### Problem 4 — graceful shutdown
`SIGTERM` handler yozib, `kill -TERM <pid>` yubor. Loglar to'g'ri chiqdimi?
`$disconnect()` ni olib tashlab, farqni ko'r.

### Problem 5 — transaction
`$transaction` bilan ikkita yozuvni atomik qil, o'rtasida `throw` qilib rollback'ni tekshir.
**Savol:** bu kod NestJS'da qanday farq qiladi? (javob: hech qanday)

### Problem 6 — query logi
`log: ['query']` yoqib, bitta endpoint necha marta SQL yuborayotganini sana.
N+1 muammosini top va `include` bilan tuzat ([[Database Relations]]).

### Problem 7 — NestJS'ga o'gir
`db.js` ni `PrismaService` + `@Global() PrismaModule` ga aylantir.
**Savol:** `require('../db')` qatorlari nimaga aylandi?

## 8. Related

[[Express Manual Dependency Wiring]] · [[Express Service Layer]] · [[Express Redis and Queues]] · [[Prisma]] · [[Database Transactions]] · [[Database Relations]] · [[Domain vs Infrastructure]]
