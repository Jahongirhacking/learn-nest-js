---
tags: [fotspot, level/beginner, guide]
level: beginner
topic: FotSpot
---

# FotSpot Source Reading Order

> **Bir jumlada:** kodni domain featurelaridan emas, **framework fundamentidan** boshlab o'qi — aks holda hamma narsa magic bo'lib ko'rinadi.

**Before learning:** [[FotSpot Architecture Overview]]
**After learning:** [[FotSpot Feature Anatomy]], [[Build a Feature in 10 Steps]]

---

## 1. Tartib

```text
1.  backend/src/main.ts               → bootstrap, global sozlamalar
2.  backend/src/app.module.ts         → kompozitsiya, global providerlar
3.  backend/src/common/               → guard, filter, decorator, interceptor
4.  backend/src/auth/                 → authentication, strategy
5.  backend/src/prisma/               → PrismaService, @Global module
6.  backend/src/users/                → eng oddiy domain
7.  backend/src/players/              → tipik CRUD feature
8.  backend/src/rbac/                 → role va permission
9.  backend/src/redis/                → cache client
10. backend/src/rate-limit/           → throttling
11. backend/src/storage/              → R2 / S3
12. backend/src/media/                → upload flow
13. backend/src/moderation/           → holat mashinasi
14. backend/src/notifications/        → queue
15. backend/src/trials/               → murakkab domain
16. backend/src/recommendations/      → murakkab domain
17. backend/src/coaches/
18. backend/src/academies/
```

## 2. Why — nega aynan shunday?

```text
1-3  → "application qanday ishga tushadi va request qanday o'tadi?"
4-5  → "kim kirmoqda va ma'lumot qayerda?"
6-7  → "oddiy feature qanday yozilgan?"
8-11 → "kesib o'tuvchi imkoniyatlar qanday qo'shilgan?"
12-18 → "murakkab business jarayonlar qanday qurilgan?"
```

Agar `trials/` dan boshlasang, birinchi qatorda `@Roles`, `@CurrentUser`, `PrismaService`, `queue.add()` ko'rasan — va ularning hech biri tanish bo'lmaydi.

## 3. Har bir fayl uchun beriladigan savollar

O'qiyotganda quyidagi 6 savolga javob izla ([[NestJS Learning Path]] dagi usul):

```text
1. Bu nima?
2. Nega kerak?
3. NestJS'da qanday ishlaydi?
4. FotSpot'da qayerda ishlatilgan?
5. Request lifecycle'da qayerda turadi?
6. Buni o'zim noldan qanday quraman?
```

## 4. Har bosqichda nimaga e'tibor berish

```text
main.ts        → global pipe/filter/guard, prefix, CORS, helmet, Swagger tartibi
app.module.ts  → APP_GUARD providerlari va ularning KETMA-KETLIGI
common/        → @Public, @Roles, @Permissions, @CurrentUser — metadata naqshi
auth/          → strategy vs guard farqi; session yaratish joyi
prisma/        → @Global module, OnModuleInit/Destroy
users/         → eng sodda controller→service→prisma zanjiri
rbac/          → many-to-many relation o'qish mashqi
media/         → presigned URL: baytlar API'dan o'tmasligi
trials/        → transaction chegarasi va queue chaqiruvi qayerda
```

## 5. Practice — o'qish rejasi

```text
Kun 1:  main.ts + app.module.ts     → butun startup zanjirini chizib chiq
Kun 2:  common/                     → har bir decorator/guardni qayta yoz
Kun 3:  auth/ + prisma/             → login flow'ini diagramma qil
Kun 4:  users/ + players/           → bitta featureni noldan takrorla
Kun 5:  rbac/ + redis/ + rate-limit → guard zanjirini to'liq tushuntir
Kun 6:  storage/ + media/ + moderation → upload va holat mashinasini chiz
Kun 7:  trials/ + recommendations   → transaction va queue chegaralarini top
```

Har kun oxirida: **o'qilgan kodni yopib, xotiradan diagramma chiz.**

## 6. Common mistakes

- **Domain featuredan boshlash** → magic hissi.
- **Faqat o'qish, yozmaslik** → tushunish yuzaki qoladi.
- **Har bir qatorni tushunishga urinish** → avval oqim, keyin tafsilot.
- **Test fayllarini o'tkazib yuborish** → ular ko'pincha eng aniq hujjat.

## 7. Related concepts

[[FotSpot Architecture Overview]] · [[FotSpot Feature Anatomy]] · [[NestJS Learning Path]] · [[Build a Feature in 10 Steps]] · [[NestJS Application Bootstrap]] · [[FotSpot Guard Chain]]
