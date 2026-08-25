---
tags: [express/foundation, level/beginner]
level: beginner
topic: Express Structure
---

# Express Router

> **Bir jumlada:** `express.Router()` — Express'dagi modularlikning yagona vositasi; u NestJS Module'ining eng yaqin qarindoshi, lekin **boundary ham, DI ham bermaydi**.

**Before learning:** [[Express Routing]]
**After learning:** [[Express Project Structure Problem]], [[Express Manual Dependency Wiring]]

**NestJS'dagi mos note:** [[NestJS Modules]]

---

## 1. Muammo

NestJS'da:

```ts
@Module({
  imports: [NotificationsModule],
  controllers: [PlayersController],
  providers: [PlayersService],
  exports: [PlayersService],
})
export class PlayersModule {}
```

To'rtta maydon — to'rtta ma'no. Express'da bularning **faqat bittasiga** yaqin narsa bor.

## 2. Router — mini-app

```js
// routes/players.routes.js
const express = require('express');
const router = express.Router();

const playersService = require('../services/players.service');
const requireAuth = require('../middleware/require-auth');

// faqat SHU router'ga tegishli middleware
router.use(someRouterLevelMiddleware);

router.get('/', async (req, res, next) => {
  try {
    res.json(await playersService.findAll(req.query));
  } catch (err) { next(err); }
});

router.get('/:id', async (req, res, next) => {
  try {
    res.json(await playersService.findOne(req.params.id));
  } catch (err) { next(err); }
});

router.post('/', requireAuth, async (req, res, next) => {
  try {
    res.status(201).json(await playersService.create(req.body, req.user.id));
  } catch (err) { next(err); }
});

module.exports = router;
```

```js
// app.js
app.use('/api/v1/players', require('./routes/players.routes'));
```

Router — bu **o'zi ham middleware**. `app.use(path, router)` router'ni zanjirga qo'shadi.

## 3. Router NestJS Module'ga qanchalik yaqin?

| NestJS Module | Express Router | Bormi? |
|---|---|---|
| `controllers` | router'dagi route'lar | ✅ ha |
| Route prefiksi | `app.use('/players', router)` | ✅ ha |
| Router-level middleware | `router.use(...)` | ✅ ha |
| `providers` (DI) | ❌ | yo'q — `require` bilan olasan |
| `imports` (bog'liqlik e'loni) | ❌ | yo'q — istalgan faylni `require` qilaverasan |
| `exports` (nima ochiq) | ❌ | yo'q — `module.exports` hamma uchun ochiq |
| Boundary nazorati | ❌ | **yo'q — eng katta farq** |

Oxirgi qator eng muhimi. Express'da:

```js
// players.service.js ichidan
const trialsService = require('../services/trials.service');   // hech kim to'smaydi
const prisma = require('../db');                                // hech kim to'smaydi
const someRandomHelper = require('../../utils/whatever');       // hech kim to'smaydi
```

**Hech qanday cheklov yo'q.** 6 oydan keyin har bir fayl har bir faylni `require` qiladi va bog'liqlik grafi chalkashib ketadi.

NestJS'da esa `exports` qilinmagan provider'ni olib bo'lmaydi — bu **majburiy chegaralar**.

## 4. Router'ni "modul"ga yaqinlashtirish

Express'da qo'lda shunday qilish mumkin:

```text
src/
  players/
    players.routes.js      ← controller
    players.service.js     ← business logic
    players.validation.js  ← DTO o'rniga
    index.js               ← faqat SHU fayl tashqariga chiqaradi
```

```js
// players/index.js — "public API"
module.exports = {
  router: require('./players.routes'),
  service: require('./players.service'),   // ataylab tanlangan
};
```

Bu **kelishuv** (convention). Uni hech kim majburlamaydi — kimdir `require('../players/players.service')` deb to'g'ridan-to'g'ri yozsa, hech narsa buzilmaydi.

```text
Express: chegara = intizom
NestJS:  chegara = framework qoidasi
```

Mana shu farq [[Express Project Structure Problem]] ning mag'zi.

## 5. Nested router

```js
// academies/academies.routes.js
const router = express.Router({ mergeParams: true });
router.use('/:aid/trials', require('../trials/trials.routes'));
```

`mergeParams: true` bo'lmasa, ichki router'da `req.params.aid` **ko'rinmaydi** — bu ko'p vaqt yo'qotadigan tuzoq.

## 6. Common mistakes

- **`mergeParams` ni unutish** → nested router'da ota parametrlar `undefined`.
- **`module.exports = router` ni unutish** → `app.use(path, undefined)` xatosi.
- **Router'ni `app.use()` ga xato tartibda qo'shish** → 404 handler'dan keyin qo'shsang, ishlamaydi.
- **Har bir route faylda `prisma` ni qayta yaratish** → [[Express Prisma Setup]].
- **Router'ni modul deb o'ylash** → u faqat routing guruhi, boundary emas.

## 7. Practice problems

### Problem 1 — router yarat
`players.routes.js` yozib, 3 ta endpoint qo'y va `app.js` da `/api/v1/players` ga ulang.

### Problem 2 — router-level middleware
`router.use()` bilan log qo'sh. **Savol:** u boshqa router'lardagi so'rovlarda ham chiqadimi?

### Problem 3 — boundary yo'qligini isbotla
`players.service.js` ni `trials.routes.js` dan to'g'ridan-to'g'ri `require` qil.
**Savol:** hech narsa to'sdimi? Endi NestJS'da `exports`siz shuni qilib ko'r — nima bo'ladi?

### Problem 4 — nested router
`GET /academies/:aid/trials` yozib, `req.params.aid` ni ol.
Avval `mergeParams`siz sinab ko'r — nima qaytdi?

### Problem 5 — bog'liqlik grafini chiz
5 ta fayldan iborat kichik loyiha yozib, kim kimni `require` qilishini chizib chiq.
Keyin xuddi shu loyihani NestJS modullariga o'gir. Graf qanday o'zgardi?

### Problem 6 — index.js konvensiyasi
Yuqoridagi `players/index.js` naqshini qo'llab, boshqa fayllardan faqat shu orqali foydalanishga **kelish**.
**Savol:** buni qanday majburlaysan? (javob: hech qanday — linter qoidasidan boshqa)

## 8. Related

[[Express Routing]] · [[Express Project Structure Problem]] · [[Express Manual Dependency Wiring]] · [[Express Service Layer]] · [[NestJS Modules]]
