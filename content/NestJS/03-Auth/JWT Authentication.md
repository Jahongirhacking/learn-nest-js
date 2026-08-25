---
tags: [auth, level/intermediate]
level: intermediate
topic: Authentication
---

# JWT Authentication

> **Bir jumlada:** JWT — server imzolagan, o'zi ma'lumot saqlaydigan token; qisqa umrli **access token** bilan uzoq umrli **refresh token** juftligi butun auth arxitekturasining asosi.

**Before learning:** [[Authentication]]
**After learning:** [[Passport and JwtStrategy]], [[Session Architecture]]

> 💡 **Tushunmadingmi? Express'da qo'lda qanday yozilishini ko'r:** [[Express Authentication]]


---

## 1. Definition

JWT uch qismdan iborat:

```text
header.payload.signature
```

- **header** — algoritm
- **payload** — `sub` (user id), `roles`, `iat`, `exp`
- **signature** — server siri bilan imzo

Imzo tufayli server tokenni **database'siz** tekshira oladi.

## 2. Why — nega kerak?

```text
Login
  ↓
credentials verified
  ↓
access token  (qisqa umrli, har bir API requestda)
      +
refresh token (uzoq umrli, faqat yangi access olish uchun)
```

Nega ikkitasi?

```text
access token uzoq umrli bo'lsa   → o'g'irlansa uzoq muddat zarar
access token qisqa umrli bo'lsa  → user har 15 daqiqada login qilishi kerak
```

Refresh token bu ziddiyatni yechadi: access qisqa, refresh esa **bekor qilinadigan** ([[Session Architecture]]).

## 3. Internal mental model

```text
Request
  ↓
Authorization: Bearer <access_token>
  ↓
JwtAuthGuard → imzo + exp tekshiriladi
  ↓
request.user = payload
```

Muhim tushunchalar:

```text
JWT shifrlangan EMAS — faqat imzolangan.
   → payload'ni har kim o'qiy oladi (base64)
   → maxfiy ma'lumot solmang!

JWT o'z-o'zidan bekor qilinmaydi (stateless).
   → logout qilish uchun server tomonda holat kerak → Session
```

## 4. FotSpot implementation

```text
access token   → qisqa TTL, API requestlarda
refresh token  → Session yozuvi bilan bog'langan, hash holida saqlanadi
```

Refresh token'ning **hash**i saqlanishi muhim: database sizib chiqsa ham, tokenlarni to'g'ridan-to'g'ri ishlatib bo'lmaydi — xuddi [[Password Hashing]] va [[OTP Security]] dagi mantiq.

Refresh flow:

```text
POST /auth/refresh  (refresh token bilan)
   ↓
Session topiladi → hash solishtiriladi → revokedAt/expiresAt tekshiriladi
   ↓
yangi access + yangi refresh (rotation)
   ↓
eski refresh bekor qilinadi
```

## 5. Code example

```ts
@Injectable()
export class TokenService {
  constructor(
    private readonly jwt: JwtService,
    private readonly config: ConfigService,
  ) {}

  async issueAccess(user: { id: string; roles: string[] }) {
    return this.jwt.signAsync(
      { sub: user.id, roles: user.roles },
      {
        secret: this.config.getOrThrow<string>('JWT_SECRET'),
        expiresIn: this.config.get('JWT_ACCESS_TTL', '15m'),
      },
    );
  }
}
```

Module konfiguratsiyasi:

```ts
JwtModule.registerAsync({
  imports: [ConfigModule],
  inject: [ConfigService],
  useFactory: (config: ConfigService) => ({
    secret: config.getOrThrow<string>('JWT_SECRET'),
    signOptions: { expiresIn: '15m' },
  }),
})
```

## 6. Common mistakes

- **Payload'ga maxfiy ma'lumot solish** → JWT shifrlanmagan.
- **Access tokenga uzoq `expiresIn` berish** (7 kun) → o'g'irlangan token uzoq yashaydi.
- **Refresh tokenni plain saqlash** → DB leak = barcha sessiyalar egallanadi.
- **Rotation qilmaslik** → bir marta o'g'irlangan refresh abadiy ishlaydi.
- **Secretni kodga yozish** → [[ConfigModule and Environment Variables]].
- **"Logout qildim, token o'lik" deb o'ylash** → JWT o'z-o'zidan o'lmaydi; session yozuvini `revokedAt` bilan yopish kerak.
- **Access va refresh uchun bir xil secret + bir xil payload** → refreshni access sifatida ishlatish mumkin bo'lib qoladi.

## 7. Practice

1. Login'da access + refresh chiqar; refreshni argon2 bilan hashlab `Session`ga yoz.
2. `jwt.io` da o'z tokeningni ochib payload'ni o'qi — nima ko'rinyapti?
3. `POST /auth/refresh` yozib, rotation qil: eski refreshni bekor qil.
4. Bekor qilingan refresh bilan qayta urin — 401 kelyaptimi?
5. `expiresIn` ni 10 soniya qilib, token muddati tugashini kuzat.

## 8. Related concepts

[[Passport and JwtStrategy]] · [[Session Architecture]] · [[Authentication]] · [[Authorization]] · [[Password Hashing]] · [[ConfigModule and Environment Variables]]
