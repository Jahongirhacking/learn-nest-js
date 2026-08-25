---
tags: [production, level/intermediate, security]
level: intermediate
topic: Production
---

# CORS

> **Bir jumlada:** CORS — brauzerning "boshqa origin'dagi sahifa bu API'ga so'rov yubora oladimi?" degan siyosati; u authentication o'rnini **bosmaydi**.

**Before learning:** [[NestJS Application Bootstrap]]
**After learning:** [[Helmet]], [[Authentication]], [[WebSockets in NestJS]]

> 💡 **Tushunmadingmi? Express'da qo'lda qanday yozilishini ko'r:** [[Express Security Middleware]]


---

## 1. Definition

```text
Frontend origin (https://app.fotspot.uz)
        │ HTTP
        ▼
   FotSpot API (https://api.fotspot.uz)
        │
        ▼
   CORS policy → ruxsat bormi?
```

```ts
app.enableCors({
  origin: ['https://app.fotspot.uz'],
  credentials: true,
});
```

## 2. Why — nega kerak?

Brauzer default holda cross-origin so'rov natijasini JavaScriptga bermaydi. Bu **foydalanuvchini** himoya qiladi: boshqa sayt sizning nomingizdan API'ga so'rov yuborib, javobini o'qiy olmasin.

Muhim tushunish:

```text
CORS brauzerda ishlaydi.
   → curl, Postman, mobil ilova, server-to-server — CORS'ni umuman ko'rmaydi
   → demak CORS himoya emas, brauzer siyosati
```

Haqiqiy himoya — [[Authentication]] va [[Authorization]].

## 3. Internal mental model

```text
Simple request  → to'g'ridan-to'g'ri ketadi, javob CORS header bilan tekshiriladi
Preflight       → OPTIONS so'rovi oldindan yuboriladi
                  (custom header, PUT/DELETE, JSON content-type bo'lsa)
```

Asosiy sozlamalar:

```text
origin       → aniq ro'yxat (yulduzcha emas!)
credentials  → cookie/Authorization yuborilsinmi
methods      → GET, POST, PATCH, DELETE
allowedHeaders → Authorization, Content-Type
```

Muhim qoida:

```text
origin: '*'  +  credentials: true  →  BRAUZER RAD ETADI
```

Ikkalasi birga ishlamaydi — origin aniq ko'rsatilishi shart.

## 4. FotSpot implementation

Ruxsat etilgan originlar konfiguratsiyadan keladi ([[ConfigModule and Environment Variables]]):

```text
development → http://localhost:3000, http://localhost:5173
production  → https://app.fotspot.uz
```

Socket.IO uchun ham alohida CORS sozlanadi ([[WebSockets in NestJS]]).

## 5. Code example

```ts
const origins = config
  .getOrThrow<string>('CORS_ORIGINS')
  .split(',')
  .map(o => o.trim());

app.enableCors({
  origin: origins,
  credentials: true,
  methods: ['GET', 'POST', 'PATCH', 'PUT', 'DELETE', 'OPTIONS'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  maxAge: 86_400,          // preflight natijasini keshlash
});
```

## 6. Common mistakes

- **`origin: '*'` ni production'da qoldirish** → har qanday sayt so'rov yuboradi.
- **`origin: '*'` + `credentials: true`** → umuman ishlamaydi, xatosi chalkash.
- **CORS xatosini authentication muammosi deb o'ylash** → brauzer konsolidagi matnni diqqat bilan o'qi.
- **Preflight (`OPTIONS`) ni guard bilan bloklash** → 401 qaytadi va brauzer asosiy so'rovni yubormaydi.
- **CORS'ni himoya deb hisoblash** → Postman'da API baribir ochiq.
- **WebSocket uchun CORS'ni alohida sozlamaslik.**

## 7. Practice

1. Brauzerdan boshqa origin'dagi sahifa orqali API'ga so'rov yuborib, xatoni ko'r.
2. `enableCors` qo'shib, ishlaganini tasdiqla.
3. Xuddi shu so'rovni `curl` bilan yubor — CORS to'sadimi? Nega yo'q?
4. `credentials: true` bilan `origin: '*'` ni sinab, xatoni o'qi.
5. Preflight `OPTIONS` so'rovini `curl -X OPTIONS -i` bilan ko'r.

## 8. Related concepts

[[Helmet]] · [[Authentication]] · [[NestJS Application Bootstrap]] · [[ConfigModule and Environment Variables]] · [[WebSockets in NestJS]]
