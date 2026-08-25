---
tags: [express/structure, level/intermediate, architecture]
level: intermediate
topic: Express Structure
---

# Express Project Structure Problem

> **Bir jumlada:** NestJS shu note'da tasvirlangan muammoni hal qilish uchun paydo bo'lgan — Express'da hamma narsa **mumkin**, lekin hech narsa **majburiy emas**.

**Before learning:** [[Express Router]], [[Express Service Layer]], [[Express Manual Dependency Wiring]]
**After learning:** [[Express to NestJS Rebuild]], [[NestJS Overview]]

**NestJS'dagi mos note:** [[NestJS Overview]] · [[NestJS Modules]] · [[Domain vs Infrastructure]]

---

## 1. Loyihaning hayoti

### 1-hafta — mukammal

```text
app.js          (80 qator)
routes/
  players.js
  auth.js
```

Hammasi tushunarli. Express to'g'ri tanlov edi.

### 3-oy — biroz o'sdi

```text
app.js
routes/       services/     middleware/    utils/
  players.js    players.js    auth.js        helpers.js
  auth.js       auth.js       validate.js    format.js
  trials.js     trials.js
```

Hali ham boshqarish mumkin. Lekin savollar paydo bo'ldi:
- `utils/helpers.js` ichida nima bor? (javob: hamma narsa)
- `validate.js` qaysi route'larda ishlatilgan? (javob: ba'zilarida)

### 1-yil — chalkashlik

```text
routes/     (18 fayl)
services/   (23 fayl)
middleware/ (11 fayl)
utils/      (14 fayl, ichida 2000 qatorli helpers.js)
lib/        (utils dan nimasi bilan farq qiladi? hech kim bilmaydi)
config/
db/
```

Va endi haqiqiy muammolar:

```text
1. "Bu funksiyani kim ishlatadi?"
   → 23 ta faylda global qidiruv

2. "Bu route'da auth bormi?"
   → har birini qo'lda tekshirish kerak

3. "playersService trialsService ni require qiladi,
    trialsService ham playersService ni..."
   → circular, undefined, tushunarsiz xato

4. "Yangi dasturchi qaerdan boshlaydi?"
   → hech qayerdan; naqsh yo'q

5. "Bu validatsiya qayerda?"
   → ba'zi route'da inline if, ba'zisida middleware, ba'zisida yo'q

6. "Nega bu endpoint 500 qaytaryapti?"
   → try/catch unutilgan
```

## 2. Har bir muammoning ildizi

| Muammo | Ildizi | NestJS yechimi |
|---|---|---|
| Kim kimni ishlatadi? | `require` cheksiz | Module `imports`/`exports` |
| Auth unutilgan | har route'da qo'lda | global guard + `@Public()` |
| Circular dependency | qo'lda `require` | DI + `forwardRef` + aniq xato |
| Naqsh yo'q | konvensiya yo'q | Module/Controller/Service majburiy |
| Validatsiya turlicha | qoida yo'q | global `ValidationPipe` + DTO |
| `try/catch` unutilgan | Express 4 | avtomatik exception handling |
| `utils/` axlat qutisi | boundary yo'q | feature moduli |

**Diqqat qil:** hech bir muammo "Express yomon" degani emas.
Ularning hammasi bitta sababdan: **Express hech narsani majburlamaydi**.

## 3. Express'da toza qurish — mumkin, lekin qimmat

Barcha muammolarni Express'da ham hal qilish mumkin:

```text
✓ feature-based papkalar (players/, trials/)
✓ container.js bilan DI
✓ global auth middleware + ochiq route'lar ro'yxati
✓ zod/Joi bilan validatsiya middleware
✓ asyncHandler wrapper
✓ index.js orqali "public API" konvensiyasi
✓ ESLint qoidalari bilan chegaralarni majburlash
```

**Lekin:**

```text
1. Buni O'ZING quryapsan (bir necha kun ish)
2. Uni O'ZING hujjatlashtiryapsan
3. Yangi dasturchiga O'ZING o'rgatyapsan
4. Har bir PR'da qoidaga rioya qilinishini O'ZING tekshiryapsan
5. Boshqa loyihaga o'tsang — hammasi boshqacha
```

NestJS'ning qiymati aynan shu: **bu ishlar allaqachon qilingan va butun ekotizim bir xil qiladi.**

## 4. Qachon qaysi biri

```text
Express to'g'ri tanlov:
  ✓ 5-15 endpoint
  ✓ 1-2 dasturchi
  ✓ prototip, ichki tool, webhook receiver
  ✓ maxsus, nostandart talablar

NestJS to'g'ri tanlov:
  ✓ 10+ feature (FotSpot: auth, players, trials, media, moderation...)
  ✓ jamoa
  ✓ uzoq muddatli loyiha
  ✓ auth + validation + queue + testing — hammasi kerak
```

**Yomon tanlov ikkala tomonda ham bor:** 5 ta endpoint uchun NestJS — ortiqcha; 200 ta endpoint uchun sof Express — qiyinchilik.

## 5. FotSpot misolida

FotSpot'da nechta narsa bir vaqtda kerak:

```text
10+ domain moduli · JWT + OAuth + OTP · RBAC + permissions
Prisma + PostgreSQL · Redis cache · BullMQ · R2 storage
WebSocket · Swagger · Sentry · rate limiting · testing
```

Bularning har birini Express'da qo'lda ulash mumkin — lekin natijada sen **o'z NestJS'ingni** yozgan bo'lasan, faqat hujjatsiz va testsiz.

Batafsil: [[FotSpot Architecture Overview]]

## 6. Practice problems

### Problem 1 — muammoni his qil
15 ta endpointli Express loyihasi yoz (auth, players, trials).
Har birini eng oson yo'l bilan yoz — refactoring qilma.
Keyin savollarga javob ber:
- Qaysi route'larda auth bor?
- `playersService` ni kim ishlatadi?
- Validatsiya qayerda?

### Problem 2 — auditni o'tkaz
Yuqoridagi loyihada har bir route uchun jadval tuz:
`route | auth bormi | validatsiya bormi | try/catch bormi`
**Savol:** nechta teshik topding?

### Problem 3 — o'z frameworkingni qur
Loyihani tozalash uchun quyidagilarni yoz:
`container.js`, global auth + public ro'yxat, `validate()` middleware, `asyncHandler`.
**Savol:** necha soat ketdi?

### Problem 4 — NestJS'ga o'gir
Xuddi shu loyihani NestJS'da qayta qur ([[Express to NestJS Rebuild]]).
2-masaladagi jadvalni qayta to'ldir — teshiklar bormi?

### Problem 5 — hukm chiqar
O'z tajribangdan kelib chiqib yoz: bu loyiha uchun qaysi biri to'g'ri tanlov edi va **nega**?
Javobingda "NestJS yaxshiroq" emas, aniq sabablar bo'lsin.

## 7. Related

[[Express Router]] · [[Express Service Layer]] · [[Express Manual Dependency Wiring]] · [[Express Middleware Order]] · [[Express to NestJS Rebuild]] · [[NestJS Overview]] · [[NestJS Modules]] · [[Domain vs Infrastructure]] · [[FotSpot Architecture Overview]]
