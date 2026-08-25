---
tags: [practice, level/intermediate, project]
level: intermediate
topic: Practice
---

# Express to NestJS Rebuild

> **Bir jumlada:** bitta API'ni avval Express'da, keyin NestJS'da yozish — NestJS'ni tushunishning eng tez usuli, chunki har bir "magic" o'zing yozgan kodning o'rniga keladi.

**Before learning:** [[Express Project Structure Problem]], [[NestJS to Express Translation]]
**After learning:** [[Mini Project Football Academy API]], [[FotSpot Source Reading Order]]

---

## Metodika

```text
1. Express'da yoz      → hamma narsa ko'rinadi, hamma narsa qo'lda
2. Og'riqni his qil    → nima takrorlanyapti? nima unutilyapti?
3. NestJS'da qayta yoz → har bir og'riq qayerga ketganini TOP
4. Solishtir           → qaysi qator yo'qoldi va nima uning o'rniga keldi?
```

Uchinchi qadam eng muhim: **har bir yo'qolgan qator uchun "kim buni endi qilyapti?" degan savolga javob ber.**

---

## Loyiha: Mini Football API

```text
POST /auth/register
POST /auth/login
GET  /users/me

GET  /players            (public)
GET  /players/:id        (public)
POST /players            (auth + role: coach)
PATCH /players/:id       (auth + ownership)

POST /trials             (auth + role: coach)
POST /trials/:id/finish  (auth + ownership + transaction + queue)
```

---

## 1-bosqich — Express'da yoz

```text
src/
  app.js               → middleware zanjiri
  server.js            → listen
  db.js                → prisma singleton
  container.js         → qo'lda DI
  config/index.js      → env validatsiya
  middleware/
    require-auth.js
    require-role.js
    validate.js
    error-handler.js
    async-handler.js
  routes/
    auth.routes.js
    players.routes.js
    trials.routes.js
  services/
    auth.service.js
    players.service.js
    trials.service.js
  validation/
    *.schema.js
```

Kerakli notelar:
[[Express App Bootstrap]] · [[Express Router]] · [[Express Middleware Order]] ·
[[Express Auth Middleware]] · [[Express Validation Middleware]] ·
[[Express Error Handling Middleware]] · [[Express Async Errors]] ·
[[Express Manual Dependency Wiring]] · [[Express Prisma Setup]]

### Tekshiruv ro'yxati (1-bosqich oxirida)

```text
[ ] Har bir async handler asyncHandler bilan o'ralgan
[ ] Har bir route'da validate() bor
[ ] Har bir himoyalangan route'da requireAuth bor
[ ] Ownership tekshiruvi service'da
[ ] Error handler eng oxirida
[ ] container.js da barcha service ulangan
[ ] .env kodda emas
```

---

## 2-bosqich — og'riqni yozib ol

Bu qadamni **o'tkazib yuborma**. Quyidagi jadvalni to'ldir:

| Og'riq | Necha marta takrorlandi? |
|---|---|
| `try/catch` yoki `asyncHandler` | ___ |
| `validate(schema)` ulash | ___ |
| `requireAuth` ulash | ___ |
| `res.status(404).json({...})` | ___ |
| `require('../services/...')` | ___ |
| `container.js` da qo'lda ulash | ___ |

Va savollarga javob ber:

```text
1. Qaysi route'da nimadir UNUTILGAN? (audit qil)
2. Yangi endpoint qo'shish uchun nechta faylga tegish kerak?
3. Yangi dasturchi qaysi konvensiyalarni bilishi kerak? Ular qayerda yozilgan?
```

---

## 3-bosqich — NestJS'da qayta yoz

```text
src/
  main.ts
  app.module.ts
  prisma/
  common/
    decorators/  guards/  filters/
  auth/
  players/
  trials/
```

Kerakli notelar:
[[NestJS Modules]] · [[NestJS Controllers]] · [[NestJS Services]] ·
[[NestJS Dependency Injection]] · [[NestJS DTO]] · [[NestJS ValidationPipe]] ·
[[NestJS Guards]] · [[NestJS Exception Filters]] · [[Build a Feature in 10 Steps]]

---

## 4-bosqich — solishtirish jadvali

Har bir Express faylini ochib, "bu qayerga ketdi?" deb yoz:

| Express fayl/qator | NestJS'da qayerda | Kim bajaryapti |
|---|---|---|
| `async-handler.js` | — | framework (avtomatik) |
| `middleware/validate.js` | — | global `ValidationPipe` |
| `validation/*.schema.js` | `dto/*.dto.ts` | class-validator |
| `middleware/require-auth.js` | `JwtAuthGuard` + `JwtStrategy` | global guard |
| `middleware/require-role.js` | `@Roles()` + `RolesGuard` | metadata + guard |
| `middleware/error-handler.js` | `HttpExceptionFilter` | global filter |
| `container.js` | — | DI container |
| `require('../services/x')` | `constructor(private x: XService)` | DI |
| `db.js` | `PrismaService` + `@Global()` | provider |
| `routes/*.routes.js` | `*.controller.ts` | decoratorlar |
| `services/*.service.js` | `*.service.ts` | **deyarli o'zgarmaydi** |

Oxirgi qatorga e'tibor ber:

> **Business logic deyarli o'zgarmaydi.**
> O'zgaradigan narsa — atrofidagi "vodoprovod" (plumbing).

---

## 5-bosqich — yakuniy savollar

Quyidagilarga **yozma** javob ber:

```text
1. Necha qator kod yo'qoldi? (`git diff --stat` yoki qo'lda sana)
2. Qaysi xatolarni endi QILIB BO'LMAYDI?
   (masalan: asyncHandler'ni unutish, validate'ni unutish)
3. Qaysi narsa Express'da OSONROQ edi?
   (halol javob ber — masalan: boshlash tezligi, kam abstraksiya)
4. Bu loyiha uchun qaysi biri to'g'ri tanlov?
5. 10 barobar katta loyiha uchun-chi?
```

Beshinchi savolga javobing — bu FotSpot nima uchun NestJS'da yozilganini tushunganingning belgisi.

---

## Keyingi qadam

Bu mashqni tugatgach:

```text
[[Mini Project Football Academy API]]   → kattaroq, to'liq loyiha
[[FotSpot Source Reading Order]]        → real kodni o'qish
[[NestJS Interview Questions]]          → bilimni tekshirish
```

## Related

[[Express Project Structure Problem]] · [[NestJS to Express Translation]] · [[Express Learning Map]] · [[Build a Feature in 10 Steps]] · [[Mini Project Football Academy API]] · [[NestJS Learning Path]]
