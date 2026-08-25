---
tags: [express/middleware, level/intermediate]
level: intermediate
topic: Middleware
---

# Express Error Handling Middleware

> **Bir jumlada:** Express'da error handler — **4 ta argumentli** middleware; NestJS'dagi Exception Filter aynan shu, faqat `throw` bilan ishlaydigan qilib qulaylashtirilgan.

**Before learning:** [[Express next Function]], [[Express Middleware Order]]
**After learning:** [[Express Async Errors]], [[NestJS Exception Filters]]

**NestJS'dagi mos note:** [[NestJS Exception Filters]]

---

## 1. Sirli qoida

```js
app.use((err, req, res, next) => { ... });
//       ^^^ 4 ta argument
```

Express funksiyaning **argumentlari soniga qarab** uni error handler deb biladi:

```text
(req, res, next)        → oddiy middleware
(err, req, res, next)   → ERROR middleware
```

**Bu haqiqatan ham shunday ishlaydi** — `fn.length === 4` tekshiriladi.
Shuning uchun `next` ishlatilmasa ham, uni yozib qo'yish **majburiy**:

```js
// ✗ 3 ta argument — Express buni error handler deb bilmaydi!
app.use((err, req, res) => { ... });

// ✓
app.use((err, req, res, next) => { ... });
```

Bu — Express'ni birinchi marta o'rganayotganda eng ko'p vaqt yeydigan tuzoqlardan biri.

## 2. To'liq error handler

```js
// utils/http-error.js
class HttpError extends Error {
  constructor(status, message) {
    super(message);
    this.status = status;
  }
}

const NotFound   = (msg = 'Not Found')   => new HttpError(404, msg);
const BadRequest = (msg = 'Bad Request') => new HttpError(400, msg);
const Forbidden  = (msg = 'Forbidden')   => new HttpError(403, msg);

module.exports = { HttpError, NotFound, BadRequest, Forbidden };
```

```js
// middleware/error-handler.js
const { Prisma } = require('@prisma/client');

module.exports = function errorHandler(err, req, res, next) {
  // 1. javob allaqachon yuborilgan bo'lsa — Express'ning o'ziga topshir
  if (res.headersSent) return next(err);

  let status = err.status || 500;
  let message = err.message || 'Internal server error';

  // 2. Prisma xatolarini tarjima qil
  if (err instanceof Prisma.PrismaClientKnownRequestError) {
    if (err.code === 'P2002') { status = 409; message = 'Resource already exists'; }
    if (err.code === 'P2025') { status = 404; message = 'Not found'; }
  }

  // 3. 5xx ni logla, 4xx ni yo'q
  if (status >= 500) {
    console.error(`[ERROR] ${req.method} ${req.originalUrl}`, err.stack);
    message = 'Internal server error';        // ichki detal chiqmasin
  }

  // 4. yagona format
  res.status(status).json({
    statusCode: status,
    message,
    path: req.originalUrl,
    timestamp: new Date().toISOString(),
  });
};
```

```js
// app.js — ENG OXIRIDA
app.use(errorHandler);
```

## 3. Xatoni qanday yuborish kerak

```js
// sinxron kodda — throw ishlaydi
app.get('/x', (req, res) => {
  throw NotFound('Player not found');       // ✓ Express ushlaydi
});

// ASYNC kodda — next() SHART (Express 4)
app.get('/y', async (req, res, next) => {
  try {
    const player = await prisma.player.findUnique({ where: { id: req.params.id } });
    if (!player) return next(NotFound('Player not found'));    // ✓
    res.json(player);
  } catch (err) {
    next(err);                                                  // ✓
  }
});
```

Ikkinchisidagi `try/catch` — Express 4'ning eng katta og'rig'i. Batafsil: [[Express Async Errors]].

## 4. NestJS bilan solishtirish

| Express | NestJS |
|---|---|
| `class HttpError extends Error` | `NotFoundException` va boshqalar — **tayyor** |
| `next(NotFound('...'))` | `throw new NotFoundException('...')` |
| `app.use(errorHandler)` — oxirida | `app.useGlobalFilters(...)` yoki `APP_FILTER` |
| `err.status \|\| 500` | `exception.getStatus()` |
| `res.status(s).json({...})` | filter ichida `response.status(s).json({...})` |
| Har bir async handler'da `try/catch` | **kerak emas** — Nest o'zi ushlaydi |
| 4 argument qoidasi | `@Catch()` decoratori |

Eng katta amaliy farq — oxirgi ikki qator:

```text
Express: 50 ta handler = 50 ta try/catch
NestJS:  50 ta handler = 0 ta try/catch
```

## 5. Common mistakes

- **3 argumentli error handler yozish** → Express uni oddiy middleware deb biladi.
- **Error handler'ni route'lardan oldin qo'yish** → hech qachon ishlamaydi ([[Express Middleware Order]]).
- **`res.headersSent` ni tekshirmaslik** → ikki marta javob yozish.
- **Stack trace'ni clientga qaytarish** → ichki struktura oshkor bo'ladi.
- **Barcha xatoni `500` qilish** → 404 va 400 ham 500 bo'lib chiqadi.
- **Xatoni ushlab, `next(err)` chaqirmaslik** → xato yo'qoladi.
- **`console.log(err)` bilan cheklanish** → `err.stack` yo'qoladi.

## 6. Practice problems

### Problem 1 — error handler yoz
Yuqoridagi `errorHandler` ni yozib ulang. `throw NotFound()` qilib sinab ko'r.

### Problem 2 — 4 argument qoidasi
Handler'ni 3 argumentli qil (`(err, req, res)`). Nima o'zgardi? Xato javobi qanday ko'rinishga keldi?

### Problem 3 — tartib
Error handler'ni `app.use(errorHandler)` ni route'lardan oldinga ko'chir. Natijani tushuntir.

### Problem 4 — async tuzoq
```js
app.get('/z', async (req, res) => {
  throw new Error('async boom');       // try/catch YO'Q
});
```
Chaqir. Nima bo'ldi? Server yiqildimi? Javob keldimi?
**Bog'lanish:** [[Express Async Errors]]

### Problem 5 — Prisma xatosi
Unique constraint buzadigan yozuv yaratishga urin va `P2002` ni `409` ga aylantir.

### Problem 6 — maxfiylik
5xx xatoda `err.stack` javobga chiqmasligini, lekin logda borligini tasdiqla.

### Problem 7 — NestJS'ga tarjima
Xuddi shu error handler'ni NestJS `HttpExceptionFilter` sifatida yoz.
**Savol:** `try/catch` lar qayerga ketdi?

## 7. Related

[[Express Async Errors]] · [[Express Middleware Order]] · [[Express next Function]] · [[Express Middleware]] · [[NestJS Exception Filters]] · [[Prisma]]
