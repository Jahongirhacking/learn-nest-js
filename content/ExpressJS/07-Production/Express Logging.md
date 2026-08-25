---
tags: [express/production, level/beginner]
level: beginner
topic: Production
---

# Express Logging

> **Bir jumlada:** Express'da logging — `morgan` yoki o'z middleware'ing; NestJS `Logger` unga **kontekst va daraja** qo'shadi, lekin muhim qoidalar ikkalasida ham bir xil.

**Before learning:** [[Express Middleware]]
**After learning:** [[Express Error Handling Middleware]], [[Logging in NestJS]]

**NestJS'dagi mos note:** [[Logging in NestJS]]

---

## 1. Tayyor yechim — morgan

```js
const morgan = require('morgan');

app.use(morgan(process.env.NODE_ENV === 'production' ? 'combined' : 'dev'));
```

Tez, lekin cheklangan: request-id yo'q, kontekst yo'q, strukturaviy emas.

## 2. O'z middleware'ing

```js
// middleware/request-logger.js
const { randomUUID } = require('crypto');

module.exports = function requestLogger(req, res, next) {
  req.id = req.headers['x-request-id'] || randomUUID();
  res.setHeader('X-Request-Id', req.id);

  const start = Date.now();

  res.on('finish', () => {                       // ← javob ketganda
    const duration = Date.now() - start;
    const level = res.statusCode >= 500 ? 'error'
                : res.statusCode >= 400 ? 'warn'
                : 'info';

    console[level === 'error' ? 'error' : 'log'](JSON.stringify({
      level,
      requestId: req.id,
      method: req.method,
      path: req.originalUrl,
      status: res.statusCode,
      duration,
      userId: req.user?.id,          // ← id, lekin email/telefon EMAS
    }));
  });

  next();
};
```

`res.on('finish')` — bu **Interceptor'ning "after" qismi** ([[NestJS Interceptors]]).

## 3. Request ID — nega muhim

```text
Request ID YO'Q:
   "Bu user 500 oldi" → loglarda 200 ta qator, qaysi biri uniki?

Request ID BOR:
   grep "a3f2-..." app.log  →  o'sha so'rovning HAMMA logi
```

Uni service'larga ham uzatish kerak:

```js
// Express'da — qo'lda uzatasan (yoki AsyncLocalStorage)
await playersService.create(dto, userId, { requestId: req.id });
```

NestJS'da ham bu muammo bor — yechim `nestjs-pino` yoki `AsyncLocalStorage`.

## 4. Nima loglash MUMKIN EMAS

```text
✗ parol (hatto xato holatida ham)
✗ JWT token, refresh token
✗ OTP kodi
✗ to'liq request body (parol shu yerda!)
✗ Authorization header
✗ karta ma'lumotlari
✗ email/telefon (ba'zi loyihalarda — GDPR)

✓ userId
✓ method, path, status, duration
✓ requestId
✓ xato turi va stack (faqat 5xx)
```

Eng ko'p uchraydigan xato:

```js
// ✗ XATO — parol logga tushdi
app.use((req, res, next) => {
  console.log(req.method, req.originalUrl, req.body);
  next();
});
```

`POST /auth/login` da `req.body` ichida **parol** bor.

## 5. Darajalar

```js
// utils/logger.js
const LEVELS = { error: 0, warn: 1, info: 2, debug: 3 };
const current = LEVELS[process.env.LOG_LEVEL ?? (process.env.NODE_ENV === 'production' ? 'info' : 'debug')];

const log = (level, context, message, extra) => {
  if (LEVELS[level] > current) return;
  console.log(JSON.stringify({ level, context, message, ...extra, ts: new Date().toISOString() }));
};

module.exports = (context) => ({
  error: (m, e) => log('error', context, m, e),
  warn:  (m, e) => log('warn',  context, m, e),
  info:  (m, e) => log('info',  context, m, e),
  debug: (m, e) => log('debug', context, m, e),
});
```

```js
const logger = require('../utils/logger')('PlayersService');
logger.info('Player created', { playerId: player.id });
```

**Bu — NestJS `Logger` ning qo'lda yozilgani:**

```ts
private readonly logger = new Logger(PlayersService.name);
this.logger.log('Player created');
```

| Express | NestJS |
|---|---|
| `require('../utils/logger')('PlayersService')` | `new Logger(PlayersService.name)` |
| `LOG_LEVEL` env bilan filtrlash | `NestFactory.create(App, { logger: [...] })` |
| `res.on('finish')` middleware | `LoggingInterceptor` |
| `req.id` qo'lda uzatish | AsyncLocalStorage / nestjs-pino |

## 6. Common mistakes

- **`req.body` ni loglash** → parol va token logga tushadi.
- **`console.log` ni hamma joyda ishlatish** → daraja yo'q, kontekst yo'q, filtrlash yo'q.
- **Har bir 400 xatoni `error` darajasida yozish** → alert shovqini, haqiqiy xato ko'rinmaydi.
- **Xatoni `err.message` bilan cheklash** → `err.stack` yo'qoladi.
- **Request ID qo'ymaslik** → loglarni bog'lab bo'lmaydi.
- **Loop ichida loglash** → disk va xarajat portlaydi.
- **Production'da `debug` darajasini qoldirish** → loglar to'lib ketadi.

## 7. Practice problems

### Problem 1 — morgan
`morgan('dev')` qo'shib, bir necha so'rov yubor. Chiqishni o'qi.

### Problem 2 — o'z logger'ing
2-bo'limdagi middleware'ni yozib, JSON log chiqar. `X-Request-Id` header'ini `curl -I` bilan tekshir.

### Problem 3 — xavfli log
`console.log(req.body)` qo'shib, `POST /auth/login` yubor.
**Logda nimani ko'rdingiz?** Endi uni maskalash funksiyasi bilan tuzat:
```js
const safe = { ...req.body }; if (safe.password) safe.password = '***';
```

### Problem 4 — darajalar
`LOG_LEVEL=info` va `LOG_LEVEL=debug` bilan ishga tushirib, farqni ko'r.

### Problem 5 — request id zanjiri
Service ichida ham `requestId` ni loglaydigan qil.
**Savol:** uni qanday uzatding? Bu qulaymi?

### Problem 6 — `res.on('finish')`
Uni `next()` dan **oldin** log yozadigan variantga o'zgartir.
**Savol:** endi `res.statusCode` to'g'ri chiqyaptimi? Nega yo'q?

### Problem 7 — NestJS'ga o'gir
Logger'ni `Logger` classiga, request logger'ni `LoggingInterceptor` ga aylantir.

## 8. Related

[[Express Middleware]] · [[Express Error Handling Middleware]] · [[Express Config and Env]] · [[Logging in NestJS]] · [[NestJS Interceptors]] · [[Sentry]]
