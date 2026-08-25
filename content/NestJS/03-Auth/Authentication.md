---
tags: [auth, level/intermediate, hub]
level: intermediate
topic: Authentication
---

# Authentication

> **Bir jumlada:** Authentication bitta savolga javob beradi — "**sen kimsan?**". "Senga nima qilishga ruxsat bor?" degani boshqa note: [[Authorization]].

**Before learning:** [[NestJS Guards]], [[NestJS Request Lifecycle]]
**After learning:** [[JWT Authentication]], [[Passport and JwtStrategy]], [[Session Architecture]], [[Authorization]]

> 💡 **Tushunmadingmi? Express'da qo'lda qanday yozilishini ko'r:** [[Express Authentication]]


---

## 1. Definition

Authentication — foydalanuvchining kimligini isbotlash jarayoni.

```text
Credentials  →  tekshiruv  →  identity  →  token
```

## 2. Why — nega kerak?

Chunki authorization'ning butun asosi shu: user kimligini bilmasdan turib, unga ruxsat berish yoki bermaslik haqida gapirib bo'lmaydi.

Shu sababli [[FotSpot Guard Chain]] da `JwtAuthGuard` `RolesGuard`dan **oldin** turadi.

## 3. Internal mental model

Authentication uch bosqichdan iborat:

```text
1. Identification   → "men aliyev@mail.com man"
2. Verification     → parol / OTP / OAuth provider tasdiqlaydi
3. Session issuance → access token + refresh token beriladi
```

Keyingi har bir request uchun:

```text
Authorization: Bearer <access_token>
        ↓
JwtAuthGuard → token verify → request.user
```

## 4. FotSpot implementation

FotSpot'da authentication faqat login/parol emas — bir nechta usul bitta `User`ga olib keladi:

```text
                   ┌──→ Password         → [[Password Hashing]]
                   │
User  ←────────────┼──→ OTP (SMS/email)  → [[OTP Security]]
                   │
                   ├──→ Google OAuth
                   │
                   └──→ Telegram OAuth
```

Muvaffaqiyatli authentication'dan keyin session yaratiladi:

```text
User
 ├── Session (telefon)
 ├── Session (noutbuk)
 └── Session (planshet)
```

Bu **multi-device authentication** — [[Session Architecture]].

Batafsil: [[FotSpot Auth Architecture]]

## 5. Code example

```ts
@Injectable()
export class AuthService {
  constructor(
    private readonly prisma: PrismaService,
    private readonly tokens: TokenService,
  ) {}

  async login(dto: LoginDto, ctx: DeviceContext) {
    const user = await this.prisma.user.findUnique({ where: { email: dto.email } });

    // Muhim: user topilmasa ham, parol noto'g'ri bo'lsa ham — bir xil javob
    if (!user || !(await argon2.verify(user.passwordHash, dto.password))) {
      throw new UnauthorizedException('Invalid credentials');
    }

    return this.tokens.issueSession(user, ctx);   // access + refresh
  }
}
```

## 6. Common mistakes

- **"User topilmadi" va "parol noto'g'ri" ni ajratib aytish** → user enumeration: hujumchi qaysi email ro'yxatdan o'tganini biladi.
- **Authentication'ni Authorization bilan aralashtirish** → ikkalasi ikki alohida qatlam.
- **Tokenni logga yozish** → log fayli kalitga aylanadi.
- **Login endpointini rate limit'siz qoldirish** → brute force ([[Rate Limiting and Throttling]]).
- **Parolni qaytarib bo'ladigan shaklda saqlash** → [[Password Hashing]].

## 7. Practice

1. `register` va `login` endpointlarini yoz; parolni argon2 bilan hashla.
2. Login javobida user obyektini qaytar — lekin `passwordHash` chiqmasligini ta'minla.
3. Noto'g'ri email va noto'g'ri parol uchun javoblar **bir xil** ekanini tekshir.
4. `GET /users/me` ni `JwtAuthGuard` bilan himoyala va `@CurrentUser()` orqali user'ni ol.

## 8. Related concepts

[[JWT Authentication]] · [[Passport and JwtStrategy]] · [[Password Hashing]] · [[OTP Security]] · [[Session Architecture]] · [[Authorization]] · [[FotSpot Auth Architecture]]
