---
tags: [practice, level/intermediate, checklist]
level: intermediate
topic: Practice
---

# Build a Feature in 10 Steps

> **Bir jumlada:** NestJS'da feature qurish — ijod emas, takrorlanadigan 10 qadamli naqsh; uni bir marta o'zlashtirsang, qolgani mexanik ish.

**Before learning:** [[FotSpot Feature Anatomy]], [[NestJS Modules]]
**After learning:** [[Mini Project Football Academy API]], [[NestJS Practice Challenges]]

> 💡 **Tushunmadingmi? Express'da qo'lda qanday yozilishini ko'r:** [[Express to NestJS Rebuild]]


---

## Naqsh

### Step 1 — Module

```text
players/players.module.ts
```

Boundary'ni birinchi belgilaysan: nima ichkarida, nima tashqarida ([[NestJS Modules]]).

### Step 2 — Controller

```text
players.controller.ts

GET  /players
POST /players
GET  /players/:id
```

Faqat marshrut va ruxsat metadata'si ([[NestJS Controllers]]).

### Step 3 — DTO

```text
dto/create-player.dto.ts
dto/update-player.dto.ts
dto/list-players.dto.ts
```

Input contract — validator decoratorlari bilan ([[NestJS DTO]]).

### Step 4 — Service

```text
players.service.ts
```

Business qoidalar shu yerda ([[NestJS Services]]).

### Step 5 — Database

```text
Prisma model: PlayerProfile
+ relation, index, migration
```

([[Prisma Schema]], [[Prisma Migrations]])

### Step 6 — Authentication

```text
JwtAuthGuard (global) — kerak bo'lsa @Public()
```

([[Authentication]], [[Public Routes Decorator]])

### Step 7 — Authorization

```text
@Roles(...)  @Permissions(...)
+ service ichida ownership tekshiruvi
```

([[Authorization]], [[Roles and Permissions]])

### Step 8 — Error handling

```text
NotFoundException · BadRequestException · ForbiddenException · ConflictException
```

([[NestJS Exception Filters]])

### Step 9 — Swagger

```text
@ApiTags · @ApiOperation · @ApiResponse · @ApiProperty
```

([[Swagger and OpenAPI]])

### Step 10 — Test

```text
unit (service) + e2e (guard zanjiri bilan)
```

([[Testing in NestJS]])

---

## Nega aynan bu tartib?

```text
1-2   → chegara va tashqi shakl
3-4   → kirish va mantiq
5     → doimiy saqlash
6-7   → xavfsizlik (kech qo'shilsa, unutiladi)
8     → xato holatlari
9-10  → hujjat va isbot
```

Ko'p dasturchi 6-7 ni oxirida qiladi — natijada ba'zi endpointlar himoyasiz qoladi. Ularni **4-qadamdan keyin darhol** yoz.

---

## Yakuniy checklist

```text
[ ] Module boundary aniq, exports faqat kerakli
[ ] Controller yupqa (business `if` yo'q)
[ ] Har bir input DTO bilan, validator decoratorlari bor
[ ] Service'da business qoidalar va domain exceptionlar
[ ] Prisma modelida index va relation to'g'ri
[ ] Ochiq endpointlar @Public() bilan ATAYLAB belgilangan
[ ] Ownership tekshiruvi bor (nafaqat role)
[ ] Response'da maxfiy maydon yo'q (select / serializer)
[ ] Uzoq ishlar queue'ga chiqarilgan
[ ] Bir nechta yozuv — transaction ichida
[ ] Swagger to'ldirilgan
[ ] Unit + e2e test yozilgan
```

---

## Practice

1. Bu 10 qadam bilan `academies` featureni to'liq qur.
2. Har qadamdan keyin to'xtab, "hozir nima ishlaydi?" deb sinab ko'r.
3. Checklistni yopib, xotiradan qayta yozib chiq.
4. Keyin xuddi shu naqsh bilan `trials` featureni qur — bu safar transaction va queue bilan.

## Related concepts

[[FotSpot Feature Anatomy]] · [[Mini Project Football Academy API]] · [[NestJS Practice Challenges]] · [[NestJS Learning Path]] · [[Testing in NestJS]]
