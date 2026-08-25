---
tags: [express/production, level/intermediate, security]
level: intermediate
topic: Production
---

# Express Rate Limiting

> **Bir jumlada:** `express-rate-limit` — middleware; NestJS `ThrottlerGuard` — guard; ikkalasi ham bir xil ish qiladi, lekin NestJS uni **metadata bilan har bir endpointga** moslashtira oladi.

**Before learning:** [[Express Middleware Order]], [[Express Redis and Queues]]
**After learning:** [[Rate Limiting and Throttling]]

**NestJS'dagi mos note:** [[Rate Limiting and Throttling]]

---

## 1. Asosiy sozlash

```js
const rateLimit = require('express-rate-limit');

const globalLimiter = rateLimit({
  windowMs: 60_000,          // 1 daqiqa
  max: 100,                  // har IP uchun 100 so'rov
  standardHeaders: true,     // RateLimit-* headerlari
  legacyHeaders: false,
  message: { statusCode: 429, message: 'Too many requests' },
});

app.use(globalLimiter);      // ← QIMMAT ishdan OLDIN
```

## 2. Endpointga xos limitlar

```js
const loginLimiter = rateLimit({
  windowMs: 60_000,
  max: 5,                                   // juda qattiq
  skipSuccessfulRequests: true,             // muvaffaqiyatli login hisoblanmasin
  keyGenerator: (req) => `${req.ip}:${req.body?.email ?? ''}`,   // IP + email
});

const otpLimiter = rateLimit({
  windowMs: 3600_000,        // 1 soat
  max: 5,                    // SMS puli!
  keyGenerator: (req) => `otp:${req.body?.phone ?? req.ip}`,
});

router.post('/login',       loginLimiter, validate(loginSchema), login);
router.post('/otp/request', otpLimiter,   validate(otpSchema),   requestOtp);
```

NestJS'da bu:

```ts
@Public()
@Throttle({ default: { ttl: 60_000, limit: 5 } })
@Post('login')
login(@Body() dto: LoginDto) {}
```

**Farq:** Express'da limiter'ni **route ta'rifiga qo'lda ulaysan**; NestJS'da metadata qo'yasan, global guard o'qiydi.

## 3. Redis store — nega majburiy

```js
// ✗ Default: xotirada
app.use(rateLimit({ windowMs: 60_000, max: 100 }));
```

```text
3 ta instance ishlayotgan bo'lsa:
   instance A: 100 ta hisobladi
   instance B: 100 ta hisobladi
   instance C: 100 ta hisobladi
   → jami 300 — limit AMALDA ISHLAMAYDI

Bundan tashqari: instance qayta ishga tushsa, hisob nolga qaytadi
```

```js
// ✓ Redis store
const RedisStore = require('rate-limit-redis').default;
const redis = require('./redis');

app.use(rateLimit({
  windowMs: 60_000,
  max: 100,
  store: new RedisStore({ sendCommand: (...args) => redis.call(...args) }),
}));
```

Bu — [[Redis]] notedagi "umumiy holat" g'oyasining eng aniq amaliy misoli.

## 4. Nega zanjirning boshida

```js
// ✗ XATO tartib
app.use(express.json());
app.use(requireAuth);                // jwt.verify + DB query
app.use(rateLimit({ max: 100 }));    // kech!

// ✓ TO'G'RI
app.use(rateLimit({ max: 100 }));    // eng arzon tekshiruv birinchi
app.use(express.json());
app.use(requireAuth);
```

```text
Xato tartibda:
   hujumchi minutiga 10 000 marta jwt.verify() va DB query'ni majburlaydi
   → rate limit ishlaydi, lekin ZARAR ALLAQACHON YETKAZILGAN
```

**Bu — [[FotSpot Guard Chain]] da `ThrottleGuard` birinchi turishining sababi.**

## 5. Public endpointlar

```js
app.use('/api/v1/auth', authRoutes);      // ochiq — lekin rate limit BOR
app.use(requireAuth);                      // shundan keyin hammasi yopiq
```

Muhim: **rate limit auth'dan oldin** bo'lgani uchun, ochiq route'lar ham himoyalangan.

NestJS'da `@Public()` faqat `JwtAuthGuard`ni chetlab o'tadi — `ThrottleGuard` baribir ishlaydi ([[Public Routes Decorator]]).

## 6. Common mistakes

- **Xotirada hisoblash (ko'p instance)** → limit amalda ishlamaydi.
- **Rate limit'ni auth'dan keyin qo'yish** → qimmat ish himoyasiz qoladi.
- **`trust proxy` ni sozlamaslik** → hamma so'rov bitta IP'dan ko'rinadi, butun sayt bloklanadi ([[Express Security Middleware]]).
- **Faqat IP bo'yicha cheklash** → NAT ortidagi ofis birga bloklanadi; login uchun `email` ni ham kalitga qo'sh.
- **`/health` ni ham cheklash** → monitoring buziladi (`skip` bilan chiqar).
- **429 javobida `Retry-After` bermaslik** → `standardHeaders: true` qo'y.
- **OTP endpointini yumshoq cheklash** → SMS bombing va real pul yo'qotish.

## 7. Practice problems

### Problem 1 — global limiter
`max: 5` qilib qo'y va 6 marta so'rov yubor:
```bash
for i in $(seq 1 6); do curl -s -o /dev/null -w "%{http_code}\n" localhost:3000/api/v1/players; done
```

### Problem 2 — headerlar
`standardHeaders: true` bilan `curl -I` qil. `RateLimit-*` headerlarini o'qi.

### Problem 3 — tartib tajribasi
`jwt.verify` ga `console.count()` qo'y. `rateLimit` ni auth'dan **keyin** qo'yib, 50 ta so'rov yubor.
Necha marta chaqirildi? Endi tartibni to'g'irlab, qayta o'lchab ko'r.

### Problem 4 — ikkita instance
`PORT=3000` va `PORT=3001` da ikkita instance ishga tushir (xotira store bilan).
Har biriga 5 tadan so'rov yubor — jami limit ishladimi?
Keyin Redis store qo'shib, qayta tekshir.

### Problem 5 — login limiter
`keyGenerator` bilan `IP + email` kaliti yasab, bir xil IP'dan turli emaillar bilan sinab ko'r.

### Problem 6 — `trust proxy`
`X-Forwarded-For: 1.2.3.4` header bilan so'rov yubor.
`trust proxy` bilan va busiz limitning xatti-harakatini solishtir.

### Problem 7 — NestJS'ga o'gir
`ThrottlerModule` + `APP_GUARD` bilan global qil, login'ga `@Throttle()` qo'y.
**Savol:** `loginLimiter` ni route'ga ulash qatori qayerga ketdi?

## 8. Related

[[Express Middleware Order]] · [[Express Security Middleware]] · [[Express Redis and Queues]] · [[Express Auth Middleware]] · [[Rate Limiting and Throttling]] · [[Redis]] · [[FotSpot Guard Chain]] · [[Public Routes Decorator]]
