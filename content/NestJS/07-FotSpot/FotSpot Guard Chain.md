---
tags: [fotspot, level/intermediate, security]
level: intermediate
topic: FotSpot
---

# FotSpot Guard Chain

> **Bir jumlada:** to'rtta global guard aniq tartibda ishlaydi — va bu tartibni yodlash emas, **sababini tushunish** kerak.

**Before learning:** [[NestJS Guards]], [[Authentication]], [[Authorization]]
**After learning:** [[Public Routes Decorator]], [[Roles and Permissions]]

> 💡 **Tushunmadingmi? Express'da qo'lda qanday yozilishini ko'r:** [[Express Middleware Order]]


---

## 1. Definition

```text
Request
   ↓
ThrottleGuard        → so'rov soni limitdami?
   ↓
JwtAuthGuard         → bu kim?
   ↓
RolesGuard           → qaysi toifadagi user?
   ↓
PermissionsGuard     → aynan bu action'ga ruxsat bormi?
   ↓
ValidationPipe → Controller → Service
```

## 2. Why — nega aynan bu tartib?

Har bir qadamning sababi bor:

```text
1. ThrottleGuard birinchi
   → eng arzon tekshiruv
   → JWT verify (kripto) va DB query'gacha hujumni to'xtatadi
   → public endpointlarni ham himoya qiladi (login, otp)

2. JwtAuthGuard ikkinchi
   → role'ni bilish uchun avval USER'ni bilish kerak

3. RolesGuard uchinchi
   → keng, arzon filtr: "admin emasmi? — darhol 403"

4. PermissionsGuard to'rtinchi
   → nozik, aniq tekshiruv (ko'proq ma'lumot talab qiladi)
```

Umumiy tamoyil: **arzon va keng tekshiruv oldin, qimmat va aniq tekshiruv keyin.**

## 3. Internal mental model

Har bir guardning kirish va chiqishi:

```text
ThrottleGuard      in: IP / userId          out: true | 429
JwtAuthGuard       in: Authorization header out: request.user | 401
RolesGuard         in: user.roles + @Roles  out: true | 403
PermissionsGuard   in: user.permissions     out: true | 403
```

Muhim nuance — `@Public()`:

```text
@Public()  →  JwtAuthGuard chetlab o'tiladi
           →  ThrottleGuard ISHLAYDI (aks holda login ochiq qolardi)
           →  token bo'lsa, user baribir aniqlanadi (optional auth)
```

Batafsil: [[Public Routes Decorator]]

## 4. Implementation

Global guardlar `APP_GUARD` provideri orqali beriladi — shunda ular DI'dan foydalana oladi:

```ts
@Module({
  providers: [
    { provide: APP_GUARD, useClass: ThrottlerGuard },
    { provide: APP_GUARD, useClass: JwtAuthGuard },
    { provide: APP_GUARD, useClass: RolesGuard },
    { provide: APP_GUARD, useClass: PermissionsGuard },
  ],
})
export class AppModule {}
```

> Bir nechta `APP_GUARD` e'lon qilinganda ular **ro'yxatdagi tartibda** ishlaydi. Shuning uchun tartib — arxitektura qarori, tasodif emas.

## 5. Code example

Endpoint darajasida bu shunday ko'rinadi:

```ts
// to'liq himoyalangan
@Roles('admin')
@Permissions('admin:moderate_video')
@Patch('media/:id/verify')
verify(@Param('id') id: string) {}

// authenticated, lekin role talab qilinmaydi
@Get('me')
me(@CurrentUser() user: JwtUser) {}

// ochiq, lekin throttle bor
@Public()
@Throttle({ default: { ttl: 60_000, limit: 5 } })
@Post('auth/login')
login(@Body() dto: LoginDto) {}
```

## 6. Common mistakes

- **Tartibni o'zgartirib, throttle'ni oxiriga qo'yish** → hujum qimmat qismlarga yetib boradi.
- **`@Public()` throttle'ni ham o'chiradi deb o'ylash.**
- **Guard ichida ownership tekshiruvi qilishga urinish** → bu service ishi ([[Authorization]]).
- **Guardlarni `useGlobalGuards` bilan ro'yxatdan o'tkazish** → DI ishlamaydi, `Reflector` topilmaydi.
- **RolesGuard'da metadata yo'q bo'lsa `false` qaytarish** → hamma endpoint yopilib qoladi.
- **Permissionni har bir requestda og'ir JOIN bilan o'qish** → [[Redis Cache]].

## 7. Practice

1. To'rtala guardni ham yozib, har biriga log qo'y va tartibni tasdiqla.
2. `@Public()` qo'yilgan endpointga limitdan ko'p so'rov yubor — 429 kelyaptimi?
3. Throttle'ni zanjir oxiriga ko'chirib, farqni o'lchab ko'r (keyin qaytar).
4. Noto'g'ri role va yetishmayotgan permission uchun 403 javoblarini ajratib tekshir.
5. Guard zanjirini E2E test bilan qamrab ol ([[Testing in NestJS]]).

## 8. Related concepts

[[NestJS Guards]] · [[Authentication]] · [[Authorization]] · [[RBAC]] · [[Roles and Permissions]] · [[Public Routes Decorator]] · [[Rate Limiting and Throttling]] · [[NestJS Request Lifecycle]]
