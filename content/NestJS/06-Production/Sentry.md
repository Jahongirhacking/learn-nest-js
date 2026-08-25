---
tags: [production, level/advanced]
level: advanced
topic: Production
---

# Sentry

> **Bir jumlada:** local `console.log` va production observability bir xil narsa emas — Sentry xatolarni to'playdi, guruhlaydi va xabar beradi.

**Before learning:** [[Logging in NestJS]], [[NestJS Exception Filters]]
**After learning:** production monitoring, alerting

---

## 1. Definition

```text
@sentry/nestjs
```

```text
Application
    ↓
Exception / error
    ↓
Sentry
    ↓
Dashboard / alerting
```

## 2. Why — nega kerak?

Log fayli savolga javob beradi, agar **qayerga qarashni bilsang**. Sentry esa o'zi keladi:

```text
✓ bir xil xatolarni bitta muammoga guruhlaydi (va sonini ko'rsatadi)
✓ stack trace + request konteksti + user (id bilan)
✓ yangi xato paydo bo'lganda darhol xabar beradi
✓ release bo'yicha ajratadi → "bu xato oxirgi deploy'dan keyin chiqdi"
```

## 3. Internal mental model

```text
Logger  → tarix (hamma narsa yoziladi)
Sentry  → signal (nimadir buzildi, e'tibor ber)
Metrics → tendensiya (qancha, qanchalik tez)
```

Nimani yuborish kerak:

```text
✓ 5xx xatolar, kutilmagan exception'lar
✓ background job xatolari ([[BullMQ]])
✗ 400/401/403 — bular kutilgan holatlar, shovqin qiladi
```

## 4. FotSpot implementation

```text
Sentry init  → main.ts, DSN configdan   ([[ConfigModule and Environment Variables]])
Filter       → 5xx xatolarni Sentry'ga yuboradi  ([[NestJS Exception Filters]])
Worker       → job failure ham yuboriladi
```

Foydali kontekst: `userId`, `requestId`, endpoint. Foydasiz va xavfli kontekst: token, parol, to'liq body.

## 5. Code example

```ts
// main.ts — Nest yaratishdan OLDIN
Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV,
  release: process.env.APP_VERSION,
  tracesSampleRate: 0.1,
  enabled: process.env.NODE_ENV === 'production',
});
```

```ts
// filter ichida — faqat 5xx
if (status >= 500) {
  Sentry.withScope(scope => {
    scope.setUser({ id: req.user?.id });
    scope.setTag('endpoint', `${req.method} ${req.route?.path}`);
    scope.setContext('request', { query: req.query });   // body EMAS
    Sentry.captureException(exception);
  });
}
```

## 6. Common mistakes

- **DSN'ni kodga yozish** → env orqali ber.
- **Development'da yoqib qo'yish** → dashboard shovqinga to'ladi.
- **Barcha 4xx larni yuborish** → haqiqiy muammolar ko'rinmay qoladi.
- **PII (email, telefon, token) yuborish** → maxfiylik buzilishi; `beforeSend` bilan tozala.
- **`release` bermaslik** → xato qaysi deploy'dan kelganini bilib bo'lmaydi.
- **Sentry'ni loglar o'rniga ishlatish** → u tarix uchun emas.
- **Background job xatolarini yubormaslik** → eng ko'p jimgina yiqiladigan joy aynan o'sha.

## 7. Practice

1. Sentry'ni sozlab, ataylab `throw new Error('test')` qil va dashboard'da ko'r.
2. Faqat 5xx yuboriladigan qilib filterni sozla.
3. `beforeSend` yozib, `authorization` header'ini olib tashla.
4. `release` qo'shib, ikki xil versiyadan xato yuborib, ajratilishini ko'r.
5. BullMQ worker'dagi xatoni ham Sentry'ga ulang ([[BullMQ]]).

## 8. Related concepts

[[Logging in NestJS]] · [[NestJS Exception Filters]] · [[ConfigModule and Environment Variables]] · [[BullMQ]]
