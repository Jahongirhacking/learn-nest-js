---
tags: [express/foundation, level/beginner]
level: beginner
topic: Express Foundations
---

# Express App Bootstrap

> **Bir jumlada:** `express()` — bu bo'sh funksiyalar ro'yxati; `app.use()` va `app.get()` ro'yxatga qo'shadi, `app.listen()` esa uni ishga tushiradi.

**Before learning:** [[Node HTTP Server]], [[Express Overview]]
**After learning:** [[Express Middleware]], [[Express Routing]]

**NestJS'dagi mos note:** [[NestJS Application Bootstrap]]

---

## 1. Muammo

NestJS'ning `main.ts` fayli sirli ko'rinadi:

```ts
const app = await NestFactory.create(AppModule);
app.useGlobalPipes(new ValidationPipe({ ... }));
app.useGlobalFilters(new HttpExceptionFilter());
await app.listen(3000);
```

Savol: **`useGlobalPipes` aslida nima qiladi?** Javob — Express'da bu shunchaki `app.use(...)`.

## 2. To'liq Express bootstrap

```js
// app.js
const express = require('express');
const helmet = require('helmet');
const cors = require('cors');

const app = express();

// ── 1. Global middleware (tartib MUHIM) ──────────────────
app.use(helmet());                                  // security headerlar
app.use(cors({ origin: ['http://localhost:5173'] }));
app.use(express.json({ limit: '1mb' }));            // req.body ni to'ldiradi
app.use(express.urlencoded({ extended: true }));

// ── 2. Logging ───────────────────────────────────────────
app.use((req, res, next) => {
  const start = Date.now();
  res.on('finish', () => {
    console.log(`${req.method} ${req.originalUrl} ${res.statusCode} ${Date.now() - start}ms`);
  });
  next();
});

// ── 3. Route'lar ─────────────────────────────────────────
app.use('/api/v1/auth', require('./routes/auth.routes'));
app.use('/api/v1/players', require('./routes/players.routes'));

// ── 4. 404 (barcha route'lardan KEYIN) ───────────────────
app.use((req, res) => {
  res.status(404).json({ statusCode: 404, message: 'Not Found', path: req.originalUrl });
});

// ── 5. Error handler (ENG OXIRIDA, 4 ta argument) ────────
app.use((err, req, res, next) => {
  const status = err.status || 500;
  if (status >= 500) console.error(err.stack);
  res.status(status).json({
    statusCode: status,
    message: status >= 500 ? 'Internal server error' : err.message,
    path: req.originalUrl,
    timestamp: new Date().toISOString(),
  });
});

module.exports = app;
```

```js
// server.js
const app = require('./app');
const port = process.env.PORT || 3000;
app.listen(port, '0.0.0.0', () => console.log(`listening on ${port}`));
```

## 3. Nima bo'layotgani

```text
express()        → bo'sh middleware ro'yxati (stack) yaratadi
app.use(fn)      → ro'yxat OXIRIGA fn qo'shadi
app.get(p, fn)   → ro'yxatga "faqat GET p bo'lsa ishlaydigan" fn qo'shadi
app.listen()     → http.createServer(app) qilib, portga bind qiladi
```

Ya'ni `app` ning o'zi — **`(req, res)` qabul qiladigan funksiya**. Shuning uchun `http.createServer(app)` ishlaydi.

Nega `app.js` va `server.js` alohida? Chunki test'da portga bind qilmasdan `app` ni ishlatish kerak — [[Express Testing]].

## 4. NestJS bilan yonma-yon

| Express | NestJS | Izoh |
|---|---|---|
| `const app = express()` | `NestFactory.create(AppModule)` | Nest DI container'ni ham quradi |
| `app.use(helmet())` | `app.use(helmet())` | **aynan bir xil** |
| `app.use(cors(...))` | `app.enableCors(...)` | o'ram (wrapper) |
| `app.use(express.json())` | avtomatik | Nest o'zi qo'shadi |
| `app.use('/api/v1', router)` | `app.setGlobalPrefix('api/v1')` | |
| validatsiya middleware'i | `app.useGlobalPipes(new ValidationPipe())` | |
| 4-argumentli error handler | `app.useGlobalFilters(new HttpExceptionFilter())` | |
| `app.listen(port)` | `app.listen(port)` | aynan bir xil |

**Asosiy tushuncha:**

```text
Express'da tartibni SEN yozasan → xato qilsang, buziladi
NestJS'da tartib QAT'IY belgilangan → xato qilish imkoni kam
```

Masalan Express'da error handler'ni route'lardan **oldin** qo'ysang — u hech qachon ishlamaydi. NestJS'da bunday xato qilib bo'lmaydi.

## 5. Common mistakes

- **Error handler'ni route'lardan oldin qo'yish** → hech qachon chaqirilmaydi.
- **404 handler'ni route'lardan oldin qo'yish** → hamma route 404 bo'ladi.
- **`express.json()` ni unutish** → `req.body` `undefined`.
- **`app.listen()` ni `app.js` ichida chaqirish** → testda port band bo'ladi.
- **`limit` bermaslik** → ulkan body bilan xotira hujumi.

## 6. Practice problems

### Problem 1 — to'liq bootstrap
Yuqoridagi `app.js` + `server.js` ni yozib ishga tushir. `curl` bilan mavjud bo'lmagan URL'ga murojaat qil — 404 JSON keldimi?

### Problem 2 — tartibni buz
404 handler'ni route'lardan **oldinga** ko'chir. `GET /api/v1/players` ni sinab ko'r.
**Kutilgan natija:** 404. **Savol:** nega? Bu NestJS'da mumkinmi?

### Problem 3 — error handler tartibi
Error handler'ni route'lardan oldinga ko'chir, keyin biror route'da `next(new Error('x'))` chaqir.
**Savol:** xato qanday ko'rinishda chiqdi? Nega sening handler'ing ishlamadi?

### Problem 4 — `express.json()` ni o'chir
Uni comment qilib, `POST` body yubor. `req.body` nima?

### Problem 5 — NestJS bilan solishtir
NestJS `main.ts` yozib, xuddi shu 5 bosqichni takrorla. Qaysi bosqichlar **umuman kerak bo'lmadi**?

### Problem 6 — `app` funksiya ekanini isbotla
```js
console.log(typeof app);          // ?
const http = require('http');
http.createServer(app).listen(3001);   // ishladimi?
```
Natijani tushuntir.

## 7. Related

[[Express Middleware]] · [[Express Middleware Order]] · [[Express Error Handling Middleware]] · [[Express Routing]] · [[Express Testing]] · [[NestJS Application Bootstrap]]
