---
tags: [express/production, level/beginner, security]
level: beginner
topic: Production
---

# Express Config and Env

> **Bir jumlada:** Express'da konfiguratsiya — `dotenv` + `process.env`; NestJS `ConfigModule` unga **validatsiya** va **fail-fast** qo'shadi.

**Before learning:** [[Express App Bootstrap]]
**After learning:** [[Express Security Middleware]], [[ConfigModule and Environment Variables]]

**NestJS'dagi mos note:** [[ConfigModule and Environment Variables]]

---

## 1. Eng oddiy variant

```js
// server.js — eng birinchi qator
require('dotenv').config();

const port = process.env.PORT || 3000;
const secret = process.env.JWT_SECRET;
```

Ishlaydi. Va shu bilan birga **xavfli**:

```text
JWT_SECRET yo'q bo'lsa → secret = undefined
                       → jwt.sign(payload, undefined) → xato... QAYERDA?
                       → birinchi login urinishida, production'da, tunda
```

## 2. Fail-fast — to'g'ri variant

```js
// config/index.js
require('dotenv').config();

function required(name) {
  const value = process.env[name];
  if (!value) {
    throw new Error(`Missing required env variable: ${name}`);   // ← STARTUP'da yiqiladi
  }
  return value;
}

function optional(name, fallback) {
  return process.env[name] ?? fallback;
}

const config = {
  env:  optional('NODE_ENV', 'development'),
  port: Number(optional('PORT', 3000)),

  databaseUrl: required('DATABASE_URL'),
  directUrl:   required('DIRECT_URL'),
  redisUrl:    required('REDIS_URL'),

  jwt: {
    secret:    required('JWT_SECRET'),
    accessTtl: optional('JWT_ACCESS_TTL', '15m'),
  },

  cors: {
    origins: optional('CORS_ORIGINS', 'http://localhost:5173').split(',').map(s => s.trim()),
  },
};

// qo'shimcha tekshiruv
if (config.env === 'production' && config.jwt.secret.length < 32) {
  throw new Error('JWT_SECRET must be at least 32 chars in production');
}

module.exports = config;
```

```js
// ishlatish
const config = require('./config');
const token = jwt.sign(payload, config.jwt.secret, { expiresIn: config.jwt.accessTtl });
```

**Bu — `ConfigModule` + Joi validation schema'ning qo'lda yozilgani.**

## 3. Nega fail-fast muhim

```text
Yomon:  JWT_SECRET yo'q → app ishga tushdi → deploy "muvaffaqiyatli"
                        → birinchi login → 500 → foydalanuvchi ko'rdi

Yaxshi: JWT_SECRET yo'q → app UMUMAN ishga tushmadi
                        → deploy to'xtadi
                        → hech kim ko'rmadi
```

FotSpot'da production startup'ida secretlar tekshiriladi ([[NestJS Application Bootstrap]], 3-qadam) — aynan shu sabab.

## 4. NestJS bilan solishtirish

| Express | NestJS |
|---|---|
| `require('dotenv').config()` | `ConfigModule.forRoot()` |
| `process.env.X` | `configService.get('X')` |
| `required('X')` (qo'lda) | `configService.getOrThrow('X')` |
| qo'lda tekshiruv | `validationSchema: Joi.object({...})` |
| `config/index.js` ni `require` | `constructor(private config: ConfigService)` |
| `isGlobal` tushunchasi yo'q | `isGlobal: true` |
| Tiplar yo'q (JS'da) | `get<string>('X')` |

Asosiy farq: NestJS'da validatsiya **deklarativ** va bitta joyda:

```ts
validationSchema: Joi.object({
  JWT_SECRET: Joi.string().min(32).required(),
  PORT: Joi.number().default(3000),
})
```

Express'da xuddi shuni qo'lda yozasan (yoki `envalid`/`zod` ishlatasan).

## 5. `.env` fayllari

```bash
# .env — HECH QACHON git'ga qo'shilmaydi
DATABASE_URL="postgresql://user:pass@host:6543/db?pgbouncer=true"
JWT_SECRET="uzun-tasodifiy-qiymat"

# .env.example — git'da, QIYMATSIZ shablon
DATABASE_URL=
DIRECT_URL=
REDIS_URL=
JWT_SECRET=
PORT=3000
```

```bash
# .gitignore
.env
.env.local
.env.*.local
```

Ikkala URL nega alohida — [[Database Connection URLs]].

## 6. Common mistakes

- **`.env` ni git'ga qo'shish** → sizib ketsa, **secretlarni almashtirish** shart.
- **`.env.example` ichiga haqiqiy qiymat yozish.**
- **`require('dotenv').config()` ni kech chaqirish** → undan oldin `require` qilingan modullar `process.env` ni bo'sh ko'radi. **Eng birinchi qator** bo'lsin.
- **`process.env` ni hamma joyda to'g'ridan-to'g'ri ishlatish** → tekshiruv yo'q, test qilish qiyin.
- **`process.env.PORT` ni number deb o'ylash** → **har doim string**. `Number()` kerak.
- **Config'ni startup'da to'liq loglash** → secretlar logga tushadi ([[Express Logging]]).
- **Zaif `JWT_SECRET`** → uzun tasodifiy qiymat ishlat.

## 7. Practice problems

### Problem 1 — `config/index.js`
Yuqoridagi config faylini yozib, hamma joyda `process.env` o'rniga uni ishlat.

### Problem 2 — fail-fast
`JWT_SECRET` ni `.env` dan olib tashla. App ishga tushdimi? Xato aniq va tushunarlimi?

### Problem 3 — kech `dotenv`
`require('dotenv').config()` ni `require('./config')` dan **keyin** qo'y.
Nima bo'ldi? Nega?

### Problem 4 — tip
`typeof process.env.PORT` ni logla. Keyin `config.port` ning tipini logla.

### Problem 5 — zod bilan validatsiya
`config/index.js` ni zod schema bilan qayta yoz.
**Savol:** bu NestJS'ning Joi schema'siga qanchalik yaqin?

### Problem 6 — `.env.example`
Yozib, `.env` ni `.gitignore` ga qo'sh. `git status` bilan tasdiqla.

### Problem 7 — NestJS'ga o'gir
`config/index.js` ni `ConfigModule.forRoot({ isGlobal: true, validationSchema })` ga aylantir.
**Savol:** `required()` funksiyasi nimaga aylandi?

## 8. Related

[[Express App Bootstrap]] · [[Express Security Middleware]] · [[Express Logging]] · [[ConfigModule and Environment Variables]] · [[Database Connection URLs]] · [[JWT Authentication]]
