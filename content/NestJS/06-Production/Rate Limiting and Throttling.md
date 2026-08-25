---
tags: [production, level/intermediate, security]
level: intermediate
topic: Production
---

# Rate Limiting and Throttling

> **Bir jumlada:** so'rovlar sonini cheklash — eng arzon himoya qatlami, shuning uchun u guard zanjirining **eng boshida** turadi.

**Before learning:** [[NestJS Guards]], [[Redis]]
**After learning:** [[FotSpot Guard Chain]], [[OTP Security]]

> 💡 **Tushunmadingmi? Express'da qo'lda qanday yozilishini ko'r:** [[Express Rate Limiting]]


---

## 1. Definition

```text
Request
  ↓
ThrottleGuard
  ├── limit ichida → davom etadi
  └── limitdan oshdi → 429 Too Many Requests
```

## 2. Why — nega kerak?

Uch xil tahdiddan himoya:

```text
1. Brute force      → parol/OTP'ni ketma-ket sinash    ([[Password Hashing]], [[OTP Security]])
2. Abuse / xarajat  → SMS bombing, qimmat endpointlar
3. DDoS / overload  → application va DB'ni ortiqcha yuklamadan saqlash
```

## 3. Internal mental model

Nega **global** va nega **birinchi**?

```text
rate limit
    before
expensive auth (token verify, argon2)
    before
database work
```

Agar throttle JWT'dan keyin bo'lsa, hujumchi har bir so'rovda qimmat kriptografik ishni bajarishga majbur qiladi.

Va nega faqat authenticated userlar uchun emas:

```text
login / register / otp — bular PUBLIC endpointlar
                       — aynan ular eng ko'p hujumga uchraydi
```

Shuning uchun `@Public()` throttle'ni **chetlab o'tmaydi** ([[Public Routes Decorator]]).

Storage tanlovi:

```text
xotirada (default)  → har bir instance alohida hisoblaydi
                     → 3 instance = 3× limit  ✗
Redis storage       → umumiy hisob            ✓
```

## 4. FotSpot implementation

```text
ThrottleGuard — global guard, zanjirning birinchi bo'g'ini

Request → ThrottleGuard → JwtAuthGuard → RolesGuard → PermissionsGuard → Controller
```

Endpointlarga xos qat'iyroq limitlar:

```text
POST /auth/login       → juda qattiq
POST /auth/otp/request → juda qattiq (SMS puli!)
GET  /players          → yumshoq
```

## 5. Code example

```ts
// module
ThrottlerModule.forRootAsync({
  inject: [ConfigService],
  useFactory: (config: ConfigService) => ({
    throttlers: [{ name: 'default', ttl: 60_000, limit: 100 }],
    storage: new ThrottlerStorageRedisService(config.getOrThrow('REDIS_URL')),
  }),
}),

// global guard
{ provide: APP_GUARD, useClass: ThrottlerGuard },
```

```ts
// endpointga xos limit
@Public()
@Throttle({ default: { ttl: 60_000, limit: 5 } })    // daqiqasiga 5 marta
@Post('login')
login(@Body() dto: LoginDto) {}

@SkipThrottle()                                       // masalan health check
@Get('health')
health() {}
```

## 6. Common mistakes

- **Faqat xotirada hisoblash** → ko'p instance'da limit ishlamaydi. Redis storage ishlat.
- **Throttle'ni auth'dan keyin qo'yish** → qimmat ish hujumchi tomonidan majburlanadi.
- **Public endpointlarni throttle'dan ozod qilish** → eng zaif joy ochiq qoladi.
- **IP bo'yicha cheklashda proxy'ni hisobga olmaslik** → hamma so'rov bitta IP'dan ko'rinadi; `trust proxy` sozlansin.
- **Faqat IP bo'yicha cheklash** → NAT ortidagi butun ofis bloklanadi; muhim endpointlarda `userId`/`email` bo'yicha ham hisobla.
- **429 javobida `Retry-After` bermaslik** → client qachon qayta urinishni bilmaydi.
- **Health check'ni throttle qilish** → monitoring buziladi.

## 7. Practice

1. Global `ThrottlerGuard` qo'shib, 429 javobini ko'r.
2. `POST /auth/login` ga qattiq limit qo'y va 6-urinishda bloklanishini tasdiqla.
3. Redis storage'ga o'tib, ikkita instance'da umumiy hisob ishlashini tekshir.
4. `@SkipThrottle()` ni `/health` ga qo'y.
5. OTP so'rash endpointiga soatlik limit qo'shib, SMS xarajatini cheklab ko'r.

## 8. Related concepts

[[NestJS Guards]] · [[FotSpot Guard Chain]] · [[Redis]] · [[OTP Security]] · [[Public Routes Decorator]] · [[Authentication]]
