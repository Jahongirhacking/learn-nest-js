---
tags: [production, level/intermediate, security]
level: intermediate
topic: Production
---

# Helmet

> **Bir jumlada:** Helmet — HTTP security headerlarini qo'yadigan middleware; bu authentication emas, **browser darajasidagi hardening**.

**Before learning:** [[NestJS Middleware]], [[NestJS Application Bootstrap]]
**After learning:** [[CORS]], [[Swagger and OpenAPI]]

> 💡 **Tushunmadingmi? Express'da qo'lda qanday yozilishini ko'r:** [[Express Security Middleware]]


---

## 1. Definition

```ts
app.use(helmet());
```

```text
NestJS response
      ↓
   Helmet
      ↓
Security headers
```

## 2. Why — nega kerak?

Header'lar brauzerga "nimaga ruxsat bermaslik" kerakligini aytadi:

```text
Content-Security-Policy    → qaysi manbalardan skript/stil yuklash mumkin
Strict-Transport-Security  → faqat HTTPS
X-Content-Type-Options     → MIME sniffing'ni o'chiradi
X-Frame-Options            → clickjacking (iframe ichiga solish)
Referrer-Policy            → URL sizishini kamaytiradi
```

Bular hujumni **butunlay** to'xtatmaydi, lekin ekspluatatsiyani qiyinlashtiradi — himoya qatlamlaridan biri.

## 3. Internal mental model

```text
Helmet  ≠ authentication
Helmet  ≠ authorization
Helmet  ≠ rate limiting

Helmet  = brauzerga beriladigan ko'rsatmalar
```

Muhim nuance: JSON API uchun CSP kabi headerlarning ta'siri cheklangan (brauzer sahifa render qilmaydi). Lekin:

```text
API bir xil domainda sahifa ham bersa (Swagger UI, error page) → CSP muhim
Brauzerdan to'g'ridan-to'g'ri ochiladigan javoblar             → sniffing himoyasi kerak
```

## 4. FotSpot implementation

FotSpot `helmet` ni `main.ts` da global middleware sifatida ishlatadi ([[NestJS Application Bootstrap]], 9-qadam).

Xavfsizlik qatlamlarining o'rni:

```text
Helmet              → brauzer siyosati
[[CORS]]            → qaysi origin so'rov yubora oladi
[[Rate Limiting and Throttling]] → so'rov soni
[[Authentication]]  → kim
[[Authorization]]   → nima qila oladi
[[NestJS ValidationPipe]] → input xavfsizligi
```

Hech biri boshqasining o'rnini bosmaydi.

## 5. Code example

```ts
app.use(
  helmet({
    contentSecurityPolicy: process.env.NODE_ENV === 'production' ? undefined : false,
    crossOriginResourcePolicy: { policy: 'cross-origin' },   // media boshqa domendan yuklansa
  }),
);
```

Swagger UI CSP tufayli ishlamay qolsa — CSP'ni butunlay o'chirish o'rniga, faqat `/docs` yo'li uchun yumshatish to'g'riroq.

## 6. Common mistakes

- **CSP muammosi chiqqanda helmet'ni butunlay o'chirish** → barcha himoya yo'qoladi. Faqat kerakli direktivani sozla.
- **Helmet'ni authentication o'rnini bosadi deb o'ylash.**
- **HSTS'ni HTTPS'siz muhitda yoqib qo'yish** → brauzer domenni eslab qoladi va local development buziladi.
- **Media/rasm boshqa domendan yuklanganda `crossOriginResourcePolicy`ni sozlamaslik** → rasmlar ko'rinmay qoladi ([[S3 and Cloudflare R2]]).
- **Header'larni deploy'dan keyin tekshirmaslik.**

## 7. Practice

1. `helmet()` ni qo'shib, `curl -I` bilan headerlarni solishtir (oldin/keyin).
2. Swagger UI ochib, CSP muammosi bor-yo'qligini ko'r.
3. `X-Frame-Options` ni sinab ko'r: API javobini iframe ichiga solishga urin.
4. Production va development uchun turli konfiguratsiya yoz.

## 8. Related concepts

[[CORS]] · [[NestJS Middleware]] · [[NestJS Application Bootstrap]] · [[Rate Limiting and Throttling]] · [[Swagger and OpenAPI]]
