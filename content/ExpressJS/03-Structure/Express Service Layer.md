---
tags: [express/structure, level/intermediate]
level: intermediate
topic: Express Structure
---

# Express Service Layer

> **Bir jumlada:** Express'da service qatlami — bu shunchaki **oddiy fayl**; NestJS'da esa u DI, lifecycle va boundary bilan birga keladigan birinchi darajali tushuncha.

**Before learning:** [[Express Route Handlers]]
**After learning:** [[Express Manual Dependency Wiring]], [[Express Project Structure Problem]]

**NestJS'dagi mos note:** [[NestJS Services]]

---

## 1. Muammo

NestJS'da service shunday:

```ts
@Injectable()
export class PlayersService {
  constructor(private readonly prisma: PrismaService) {}
}
```

Savollar:
- `@Injectable()` nima qiladi?
- `prisma` qayerdan keladi?
- Nega `new PlayersService()` yozmayapmiz?

Express'da javob ochiq — chunki u yerda **hech qanday sehr yo'q**.

## 2. Express'da service — uch xil usul

### Usul A — funksiyalar to'plami (eng keng tarqalgan)

```js
// services/players.service.js
const prisma = require('../db');                 // global singleton
const { NotFound } = require('../utils/http-error');

async function findAll({ page = 1, take = 20 }) {
  return prisma.playerProfile.findMany({
    where: { status: 'ACTIVE' },
    skip: (page - 1) * take,
    take,
  });
}

async function findOne(id) {
  const player = await prisma.playerProfile.findUnique({ where: { id } });
  if (!player) throw NotFound('Player not found');
  return player;
}

module.exports = { findAll, findOne };
```

```js
// routes/players.routes.js
const playersService = require('../services/players.service');
```

**Muammo:** `prisma` qattiq bog'langan (`require` orqali). Testda uni almashtirish qiyin.

### Usul B — class + `new` (NestJS'ga yaqinroq)

```js
// services/players.service.js
class PlayersService {
  constructor(prisma) {
    this.prisma = prisma;              // ← dependency PARAMETR sifatida
  }

  async findOne(id) {
    const player = await this.prisma.playerProfile.findUnique({ where: { id } });
    if (!player) throw NotFound('Player not found');
    return player;
  }
}

module.exports = PlayersService;
```

```js
// container.js — qo'lda "DI"
const prisma = require('./db');
const PlayersService = require('./services/players.service');

module.exports = {
  playersService: new PlayersService(prisma),      // ← qo'lda ulash
};
```

**Bu — NestJS DI'sining qo'lda yozilgan varianti.** Batafsil: [[Express Manual Dependency Wiring]]

### Usul C — factory funksiya

```js
module.exports = function createPlayersService({ prisma, storage }) {
  return {
    async findOne(id) { ... },
  };
};

// ulash
const playersService = createPlayersService({ prisma, storage });
```

## 3. Service nima qilishi kerak

Bu qoidalar Express'da ham, NestJS'da ham **bir xil**:

```text
✓ business qoidalar
    if (trial.status === 'FINISHED') throw Conflict(...)
✓ orkestratsiya (bir necha service/infra chaqiruvi)
✓ transaction chegarasi
✓ domain xatolari (NotFound, Forbidden)

✗ req / res bilan ishlash
✗ res.status(404) qaytarish
✗ HTTP haqida biror narsa bilish
```

Oxirgi qoidaning amaliy foydasi:

```text
playersService.findOne(id) ni chaqira oladi:
   ✓ HTTP handler
   ✓ BullMQ worker
   ✓ WebSocket gateway
   ✓ CLI script
   ✓ cron job
   ✓ test
```

Agar service ichida `res` bo'lsa — bularning hech biri ishlamaydi.

## 4. Testlash — asosiy farq

### Usul A (require bilan) — qiyin

```js
// prisma'ni almashtirish uchun modul cache'ini buzish kerak
jest.mock('../db', () => ({
  playerProfile: { findUnique: jest.fn() },
}));
```

Ishlaydi, lekin mo'rt: fayl yo'liga bog'liq, refactoring'da sinadi.

### Usul B (constructor bilan) — oson

```js
const service = new PlayersService({
  playerProfile: { findUnique: async () => null },     // mock
});

await expect(service.findOne('x')).rejects.toThrow('Player not found');
```

**Hech qanday mocking kutubxonasi kerak emas.**

### NestJS — o'rtasi

```ts
const module = await Test.createTestingModule({
  providers: [
    PlayersService,
    { provide: PrismaService, useValue: mockPrisma },
  ],
}).compile();
```

Usul B ning o'zi, faqat framework tomonidan boshqariladi.

**Xulosa:** DI — bu "sehr" emas, bu **Usul B ni majburiy qilish**.

## 5. Common mistakes

- **Service ichida `res` ishlatish** → qayta ishlatib bo'lmaydi.
- **Service'ni "prisma wrapper" qilib qo'yish** (`findAll` = `prisma.x.findMany()`) → qatlam qiymat bermaydi.
- **Har bir service faylida `new PrismaClient()`** → connection pool portlaydi ([[Express Prisma Setup]]).
- **God service** — 1500 qatorli `usersService`.
- **Service'da HTTP status code qaytarish** (`return { status: 404 }`) → HTTP sizib kirdi.
- **Circular require** (`a.js` ↔ `b.js`) → `undefined` export, tushunarsiz xato.

## 6. Practice problems

### Problem 1 — Usul A
`players.service.js` ni funksiyalar to'plami sifatida yozib, route'dan chaqir.

### Problem 2 — Usul B
Xuddi shu service'ni class + constructor injection bilan qayta yoz.

### Problem 3 — ikkalasini test qil
Ikkala variantni ham test qil (mock prisma bilan).
**Savol:** qaysi biri kamroq kod talab qildi? Nega?

### Problem 4 — qayta ishlatish
`playersService.findOne()` ni HTTP'siz — oddiy `node script.js` dan chaqir.
Agar service ichida `res` bo'lganida, bu mumkin bo'larmidi?

### Problem 5 — circular require
`a.service.js` va `b.service.js` yozib, bir-birini `require` qil.
Nima chiqdi? Buni qanday tuzatasan?
**Savol:** NestJS'da bu qanday ko'rinadi? (javob: `forwardRef` va aniq xato matni)

### Problem 6 — NestJS'ga o'gir
Usul B dagi service'ni `@Injectable()` qilib, NestJS module'ga qo'sh.
**Savol:** `new PlayersService(prisma)` qatori qayerga ketdi?

## 7. Related

[[Express Manual Dependency Wiring]] · [[Express Route Handlers]] · [[Express Prisma Setup]] · [[Express Testing]] · [[NestJS Services]] · [[NestJS Dependency Injection]]
