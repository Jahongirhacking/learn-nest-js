---
tags: [express/structure, level/intermediate, fundamental]
level: intermediate
topic: Express Structure
---

# Express Manual Dependency Wiring

> **Bir jumlada:** Dependency Injection — sehr emas; u sen Express'da qo'lda yozadigan `new Service(dep)` zanjirining avtomatlashtirilgan varianti.

**Before learning:** [[Express Service Layer]]
**After learning:** [[Express Project Structure Problem]], [[Express Testing]]

**NestJS'dagi mos note:** [[NestJS Dependency Injection]] · [[NestJS Providers]]

**Agar NestJS'da faqat bitta narsa tushunarsiz bo'lsa va u DI bo'lsa — shu note aynan sen uchun.**

---

## 1. Muammo

NestJS'da bu kod ishlaydi:

```ts
@Injectable()
export class PlayersService {
  constructor(private readonly prisma: PrismaService) {}
}

@Controller('players')
export class PlayersController {
  constructor(private readonly players: PlayersService) {}
}
```

Va hech qayerda `new` yo'q. Savollar:

```text
1. PlayersService instance'ini kim yaratdi?
2. PrismaService qayerdan keldi?
3. Nechta instance yaratildi?
4. Nima uchun umuman shunday qilinadi?
```

Express'da bularning hammasiga javob **ko'z oldingda** turadi.

## 2. Express'da qo'lda ulash

```js
// ── 1. Eng past qatlam ────────────────────────────────
const { PrismaClient } = require('@prisma/client');
const prisma = new PrismaClient();

const Redis = require('ioredis');
const redis = new Redis(process.env.REDIS_URL);

// ── 2. Infratuzilma servislari ────────────────────────
const StorageService = require('./services/storage.service');
const storage = new StorageService(process.env.R2_ENDPOINT, process.env.R2_BUCKET);

const CacheService = require('./services/cache.service');
const cache = new CacheService(redis);

// ── 3. Domain servislari ──────────────────────────────
const NotificationsService = require('./services/notifications.service');
const notifications = new NotificationsService(prisma, redis);

const PlayersService = require('./services/players.service');
const players = new PlayersService(prisma, storage);

const TrialsService = require('./services/trials.service');
const trials = new TrialsService(prisma, notifications, players);   // ← 3 ta dependency

// ── 4. Eksport ────────────────────────────────────────
module.exports = { prisma, redis, storage, cache, notifications, players, trials };
```

**Mana shu fayl — DI container'ning qo'lda yozilgani.** `container.js` deb ataladi.

## 3. Nima bo'lyapti — qadam-baqadam

```text
Tartib MUHIM:
   prisma dan OLDIN players yaratib bo'lmaydi
   notifications dan OLDIN trials yaratib bo'lmaydi

Ya'ni bu — TOPOLOGIK TARTIB.
Sen uni qo'lda to'g'ri yozishing kerak.

Bog'liqlik grafi:
   prisma ──┬──▶ players ──┐
            │              ├──▶ trials
            ├──▶ notifications ──┘
   redis ───┘
```

**NestJS aynan shu grafni o'zi quradi** — decorator metadata'sidan o'qib, topologik tartibda yaratadi.

Shuning uchun NestJS xatosi shunday ko'rinadi:

```text
Nest can't resolve dependencies of the TrialsService (PrismaService, ?, PlayersService)
                                                                    ^
                                                 ikkinchi parametrni topa olmadim
```

Bu — "container.js da bir qatorni unutding" degani.

## 4. Singleton — nega muhim

```js
// container.js
const prisma = new PrismaClient();          // BITTA marta

const players = new PlayersService(prisma);
const trials  = new TrialsService(prisma, notifications);
//                                 ^^^^^^ AYNI o'sha obyekt
```

Agar shunday qilsang:

```js
// ✗ XATO
// players.service.js
const prisma = new PrismaClient();     // 1-instance

// trials.service.js
const prisma = new PrismaClient();     // 2-instance

// users.service.js
const prisma = new PrismaClient();     // 3-instance
```

**Natija:** har biri o'z connection pool'ini ochadi → PostgreSQL `too many connections` beradi.

NestJS'da bu xatoni qilish qiyin: `PrismaService` — provider, va default'da **singleton**.

## 5. Testlash — DI ning haqiqiy sababi

```js
// ✗ require bilan qattiq bog'langan
const prisma = require('../db');
async function findOne(id) {
  return prisma.player.findUnique({ where: { id } });
}
// Test uchun: jest.mock('../db') — fayl yo'liga bog'liq, mo'rt
```

```js
// ✓ constructor injection
class PlayersService {
  constructor(prisma) { this.prisma = prisma; }
}

// Test — hech qanday mocking kutubxonasisiz
const service = new PlayersService({
  playerProfile: { findUnique: async () => null },
});
await expect(service.findOne('x')).rejects.toThrow('Player not found');
```

```ts
// NestJS — xuddi shu, faqat framework boshqaradi
const module = await Test.createTestingModule({
  providers: [PlayersService, { provide: PrismaService, useValue: mockPrisma }],
}).compile();
```

**Xulosa:**

> DI — "yangi tushuncha" emas.
> DI = "dependency'ni `require` qilma, **parametr sifatida qabul qil**" degan qoidani majburiy qilish.

## 6. `container.js` qachon og'riqli bo'ladi

Kichik loyihada u toza va tushunarli. 30 ta service'da esa:

```text
✗ 150 qatorli container.js
✗ tartibni qo'lda saqlash (A dan oldin B)
✗ yangi dependency qo'shsang, o'nlab qatorni yangilash
✗ circular dependency'ni O'ZING topishing kerak
✗ test uchun butun grafni qayta qurish
```

**Mana shu nuqtada NestJS DI container'i o'zini oqlaydi:**

| | Express `container.js` | NestJS DI |
|---|---|---|
| Graf qurilishi | qo'lda, tartib bilan | avtomatik, metadata'dan |
| Singleton | o'zing kafolatlaysan | default |
| Xato signali | `undefined` (runtime, kech) | startup'da aniq xabar |
| Circular aniqlash | qo'lda | avtomatik + `forwardRef` |
| Test almashtirish | qo'lda qayta qurish | `{ provide, useValue }` |
| Scope (request-level) | juda qiyin | `Scope.REQUEST` |

## 7. Common mistakes

- **Har bir faylda `new PrismaClient()`** → connection portlashi.
- **`container.js` da tartibni buzish** → `undefined` dependency, tushunarsiz xato.
- **Circular require** → biri `undefined` bo'lib chiqadi, xato juda kech ko'rinadi.
- **Global obyektlarga tayanish** (`global.prisma`) → test va reasoning yo'qoladi.
- **Config'ni service ichida `process.env` dan o'qish** → parametr sifatida uzat ([[Express Config and Env]]).

## 8. Practice problems

### Problem 1 — `container.js` yoz
`prisma → playersService → trialsService` zanjirini qo'lda ulab, route'lardan foydalan.

### Problem 2 — tartibni buz
`container.js` da `trials` ni `notifications` dan **oldin** yarat.
**Savol:** qanday xato chiqdi? U aniq va tushunarlimi? Qachon chiqdi — startup'dami yoki request'damimi?

### Problem 3 — singleton'ni buz
Ikkita faylda alohida `new PrismaClient()` yarat va `prisma.$queryRaw` bilan connection sonini kuzat.

### Problem 4 — testni ikki xil yoz
Xuddi shu service'ni (a) `require` bilan, (b) constructor injection bilan yozib, ikkalasini test qil.
Test kodini yonma-yon qo'yib solishtir.

### Problem 5 — circular
`a.service.js` va `b.service.js` bir-birini `require` qilsin. `console.log` bilan qaysi biri `undefined` ekanini top.

### Problem 6 — grafni chiz
5 ta service'li `container.js` yozib, bog'liqlik grafini qog'ozga chiz.
Keyin shuni NestJS modullariga o'gir — graf o'zgardimi yoki faqat **kim quradi** o'zgardimi?

### Problem 7 — xato xabarlarini solishtir
NestJS'da ataylab `providers` ga qo'shishni unut va xatoni o'qi.
Express'dagi `undefined` xatosi bilan solishtir. Qaysi biri tezroq tuzatishga olib keladi?

## 9. Related

[[Express Service Layer]] · [[Express Prisma Setup]] · [[Express Project Structure Problem]] · [[Express Testing]] · [[NestJS Dependency Injection]] · [[NestJS Providers]] · [[NestJS Modules]] · [[Testing in NestJS]]
