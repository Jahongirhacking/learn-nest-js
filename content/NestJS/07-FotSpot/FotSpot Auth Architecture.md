---
tags: [fotspot, level/intermediate, security]
level: intermediate
topic: FotSpot
---

# FotSpot Auth Architecture

> **Bir jumlada:** bir nechta kirish usuli (parol, OTP, OAuth) bitta `User`ga olib keladi, keyin har bir qurilma uchun alohida session ochiladi.

**Before learning:** [[Authentication]], [[JWT Authentication]], [[Session Architecture]]
**After learning:** [[FotSpot Guard Chain]], [[RBAC]]

---

## 1. Definition

```text
                   ┌──→ Password        → [[Password Hashing]]
                   │
User  ←────────────┼──→ OTP             → [[OTP Security]]
                   │
                   ├──→ Google OAuth
                   │
                   └──→ Telegram OAuth
```

Keyin:

```text
User
 ├── Session 1  (telefon)
 ├── Session 2  (noutbuk)
 └── Session 3  (planshet)
```

## 2. Why — nega bir nechta usul?

```text
Password  → klassik, hamma biladi
OTP       → telefon raqami asosiy identifikator bo'lgan bozorda qulay
OAuth     → ro'yxatdan o'tish ishqalanishini kamaytiradi
```

Lekin arxitektura nuqtai nazaridan muhimi:

> Har xil kirish usuli — bir xil natija: **identity tasdiqlandi → session ochiladi.**

Shuning uchun `AuthService`da usul-specific qism kichik, umumiy qism (`issueSession`) bitta.

## 3. Internal mental model

```text
1. Identity provider (parol / OTP / OAuth)
        ↓
2. User topiladi yoki yaratiladi (account linking)
        ↓
3. Session yaratiladi (deviceId, userAgent, ip)
        ↓
4. access token + refresh token
        ↓
5. Keyingi requestlar: JwtAuthGuard → request.user
```

Eng nozik joy — **3-qadam: account linking**:

```text
Google bilan kirgan email allaqachon parol bilan ro'yxatdan o'tgan bo'lsa?
   ✓ email tasdiqlangan bo'lsa → bog'lash mumkin
   ✗ tasdiqlanmagan bo'lsa → hisobni egallab olish xavfi
```

Bu savolga oldindan javob bermaslik — real xavfsizlik zaifligi.

## 4. Implementation

```text
auth/
 ├── auth.module.ts        → PassportModule, JwtModule
 ├── auth.controller.ts    → register, login, refresh, logout, otp, oauth
 ├── auth.service.ts       → identity tekshiruvi
 ├── token.service.ts      → access/refresh chiqarish, rotation
 ├── strategies/           → JwtStrategy va boshqalar
 ├── guards/               → JwtAuthGuard
 └── dto/
```

Bog'liq notelar: [[Passport and JwtStrategy]], [[Session Architecture]], [[Public Routes Decorator]]

## 5. Endpointlar va ularning himoyasi

```text
POST /auth/register     @Public + qattiq throttle
POST /auth/login        @Public + qattiq throttle
POST /auth/otp/request  @Public + juda qattiq throttle (SMS puli)
POST /auth/otp/verify   @Public + attempts limiti
POST /auth/refresh      @Public (refresh tokenning o'zi autentifikatsiya)
POST /auth/logout       authenticated
GET  /auth/sessions     authenticated
DELETE /auth/sessions/:id authenticated + ownership
```

E'tibor ber: `@Public()` bo'lsa ham throttle ishlaydi ([[Rate Limiting and Throttling]]).

## 6. Common mistakes

- **Account linking qoidasini yozmaslik** → hisobni egallab olish.
- **Har bir kirish usuli uchun alohida token mantiqini yozish** → takrorlanish va nomuvofiqlik; `issueSession` bitta bo'lsin.
- **OAuth `state` parametrini tekshirmaslik** → CSRF.
- **Refresh tokenni plain saqlash** → [[Session Architecture]].
- **Parol o'zgarganda sessiyalarni bekor qilmaslik.**
- **OTP va login uchun bir xil (yumshoq) rate limit** → SMS xarajati va brute force.
- **Auth javoblarida foydalanuvchi mavjudligini oshkor qilish** → [[Authentication]].

## 7. Practice

1. `register → login → me → refresh → logout` flow'ini to'liq yoz.
2. OTP bilan kirishni qo'shib, xuddi shu `issueSession` ni qayta ishlat.
3. Account linking qoidasini yozib, hujjatlashtir.
4. `GET /auth/sessions` va aniq sessiyani bekor qilish endpointini qo'sh.
5. Parol o'zgarganda barcha sessiyalar yopilishini E2E test bilan tekshir.

## 8. Related concepts

[[Authentication]] · [[JWT Authentication]] · [[Passport and JwtStrategy]] · [[Session Architecture]] · [[OTP Security]] · [[Password Hashing]] · [[FotSpot Guard Chain]] · [[RBAC]]
