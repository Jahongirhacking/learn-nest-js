---
tags: [practice, level/beginner, challenges]
level: beginner
topic: Practice
---

# Express Practice Problems

> **Bir jumlada:** 25 ta masala — har biri NestJS'dagi bitta "magic"ni Express'da qo'lda yozib ochib beradi.

**Before learning:** [[Express Learning Map]]
**After learning:** [[Express to NestJS Rebuild]], [[NestJS Practice Challenges]]

Har bir masaladan keyin savol: **"NestJS bu ishni endi kim uchun qilyapti?"**

---

## Fundament

### 1 — Express'siz server
Sof Node `http` bilan `GET /health` yoz. → [[Node HTTP Server]]
**NestJS'da:** `NestFactory.create()` + `@Get('health')`

### 2 — POST body'ni qo'lda o'qi
`req.on('data')` bilan JSON body'ni yig'ib parse qil.
**Savol:** `express.json()` va NestJS'ning avtomatik parsingi qanchalik ish tejaydi?

### 3 — Express'ga o'gir
1 va 2-masalani Express bilan qayta yoz. Necha qator qisqardi? → [[Express Overview]]

### 4 — `app` funksiya ekanini isbotla
`http.createServer(app)` bilan ishga tushir. → [[Express App Bootstrap]]

### 5 — route tartibi tuzog'i
`/players/:id` ni `/players/me` dan oldin qo'yib, buzilishni ko'r. → [[Express Routing]]
**NestJS'da:** aynan shu xato mumkin — bu framework muammosi emas.

---

## Middleware — yurak

### 6 — uchta tanlov
Bitta middleware'ni navbat bilan `next()`, `res.json()`, `next(err)` qil. → [[Express Middleware]]

### 7 — `next()` ni unut
Request osilib qolishini ko'r. Qancha kutdi? Xato chiqdimi? → [[Express next Function]]
**NestJS'da:** bu xatoni qilib bo'lmaydi.

### 8 — matryoshka tartibi
`next()` dan oldin va keyin log yozib, tartibni tasdiqla. → [[Express Middleware Order]]
**NestJS'da:** bu Interceptor'ning before/after qismi.

### 9 — error handler
4 argumentli handler yozib ulang. Keyin 3 argumentli qil — nima o'zgardi? → [[Express Error Handling Middleware]]

### 10 — async tuzoq
`async` handler ichida `throw` qilib, request osilib qolishini ko'r. → [[Express Async Errors]]
**NestJS'da:** avtomatik ushlanadi.

### 11 — `asyncHandler` yoz
Wrapper yozib, 3 ta route'ni o'ra. Keyin bittasini o'rashni unut va oqibatni ko'r.

### 12 — 404 va error tartibi
404 handler'ni route'lardan oldinga ko'chirib, hammasi buzilishini ko'r.

---

## Struktura

### 13 — 60 qatorlik handler
Auth + validation + business + DB + email — hammasini bitta handler'ga yoz. → [[Express Route Handlers]]
Keyin uni **HTTP'siz test qilishga** urin.

### 14 — qatlamlarga bo'l
13-ni route + service ga ajrat va business qoidani unit test qil. → [[Express Service Layer]]

### 15 — `container.js`
`prisma → playersService → trialsService` zanjirini qo'lda ulang. → [[Express Manual Dependency Wiring]]
**NestJS'da:** bu faylni DI container yozadi.

### 16 — tartibni buz
`container.js` da `trials` ni `notifications` dan oldin yarat. Xato aniqmi?
NestJS'ning `Nest can't resolve dependencies` xabari bilan solishtir.

### 17 — singleton'ni buz
Ikkita faylda `new PrismaClient()` yaratib, connection sonini kuzat. → [[Express Prisma Setup]]

### 18 — circular require
Ikkita service bir-birini `require` qilsin. Qaysi biri `undefined`?

---

## Validation va Auth

### 19 — mass assignment
Validatsiyasiz endpointga `{"firstName":"Ali","role":"admin"}` yubor. → [[Express Request Validation]]
**NestJS'da:** `forbidNonWhitelisted: true` buni to'sadi.

### 20 — `validate()` middleware
zod schema bilan middleware yozib, 3 ta route'ga ula. → [[Express Validation Middleware]]
Keyin bittasida unut va teshikni top.

### 21 — JWT login
argon2 + jsonwebtoken bilan register/login yoz. → [[Express Authentication]]

### 22 — `requireAuth` + `requireRole`
Ikkalasini yozib zanjirlab ko'r. → [[Express Auth Middleware]] · [[Express Authorization Middleware]]
**Savol:** `requireRole('admin')` nima qaytaradi? `'admin'` qayerda saqlanadi?

### 23 — Variant A va C
Bir xil 6 ta route'ni (A) har birida qo'lda auth, (C) global auth + ochiq ro'yxat bilan yoz.
Bittasida auth'ni unut. Qaysi variantda tezroq topding? → [[Public Routes Decorator]]

---

## Production

### 24 — rate limit tartibi
`rateLimit` ni auth'dan oldin va keyin qo'yib, `jwt.verify` chaqiruvlarini sana. → [[Express Rate Limiting]]
**Bu — [[FotSpot Guard Chain]] tartibining sababi.**

### 25 — xavfli log
`console.log(req.body)` bilan login so'rovini logla va parolni ko'r. Keyin maskala. → [[Express Logging]]

---

## Yakuniy mashq

Yuqoridagi 25 ta masaladan chiqqan barcha kodni bitta loyihaga birlashtir,
keyin uni NestJS'ga o'gir: [[Express to NestJS Rebuild]]

Va har bir masala uchun jadvalni to'ldir:

```text
| Masala | Express'da necha qator | NestJS'da kim qilyapti |
```

## Related

[[Express Learning Map]] · [[NestJS to Express Translation]] · [[Express to NestJS Rebuild]] · [[NestJS Practice Challenges]] · [[Express Middleware]] · [[Express Manual Dependency Wiring]]
