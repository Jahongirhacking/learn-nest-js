---
tags: [production, level/intermediate, security]
level: intermediate
topic: Production
---

# ConfigModule and Environment Variables

> **Bir jumlada:** kod hamma muhitda bir xil bo'lishi kerak — farq faqat konfiguratsiyada; va hech qanday secret kodga yozilmaydi.

**Before learning:** [[NestJS Modules]], [[NestJS Providers]]
**After learning:** [[Database Connection URLs]], [[Sentry]], [[NestJS Application Bootstrap]]

> 💡 **Tushunmadingmi? Express'da qo'lda qanday yozilishini ko'r:** [[Express Config and Env]]


---

## 1. Definition

```ts
ConfigModule.forRoot({ isGlobal: true })
```

```text
.env / platform secrets
        ↓
ConfigModule
        ↓
ConfigService
        ↓
Nest providers
```

## 2. Why — nega kerak?

```text
✗ const secret = 'my-jwt-secret'      → git history'da abadiy qoladi
✓ config.getOrThrow('JWT_SECRET')     → muhitdan keladi
```

Bundan tashqari, bir xil image (Docker) dev, staging va production'da **o'zgarishsiz** ishlashi kerak — faqat env boshqacha.

## 3. Internal mental model

```text
isGlobal: true   → har bir modulda ConfigModule'ni import qilish shart emas
get()            → undefined qaytarishi mumkin
getOrThrow()     → yo'q bo'lsa STARTUP'da yiqiladi   ← xavfsizroq
validationSchema → barcha env'lar bir joyda tekshiriladi
```

Muhim tamoyil — **fail fast**:

```text
Yomon:  JWT_SECRET yo'q → app ishga tushdi → birinchi login'da 500
Yaxshi: JWT_SECRET yo'q → app umuman ishga tushmadi → deploy to'xtadi
```

FotSpot'da production startup'ida secretlar tekshiriladi ([[NestJS Application Bootstrap]], 3-qadam).

## 4. FotSpot implementation

Ishlatiladigan konfiguratsiya turlari (qiymatlarsiz):

```text
DATABASE_URL        → [[Database Connection URLs]]
DIRECT_URL
REDIS_URL           → [[Redis]]
JWT_SECRET          → [[JWT Authentication]]
PORT
R2 credentials      → [[S3 and Cloudflare R2]]
SENTRY_DSN          → [[Sentry]]
Telegram / SMS / Email credentials
CORS origins        → [[CORS]]
NODE_ENV
```

Repozitoriyda faqat `.env.example` bo'ladi — **qiymatlarsiz shablon**.

## 5. Code example

```ts
// app.module.ts
ConfigModule.forRoot({
  isGlobal: true,
  envFilePath: ['.env.local', '.env'],
  validationSchema: Joi.object({
    NODE_ENV: Joi.string().valid('development', 'test', 'production').required(),
    PORT: Joi.number().default(3000),
    DATABASE_URL: Joi.string().required(),
    DIRECT_URL: Joi.string().required(),
    REDIS_URL: Joi.string().required(),
    JWT_SECRET: Joi.string().min(32).required(),
    JWT_ACCESS_TTL: Joi.string().default('15m'),
  }),
})
```

```ts
// typed ishlatish
@Injectable()
export class TokenService {
  private readonly secret: string;

  constructor(config: ConfigService) {
    this.secret = config.getOrThrow<string>('JWT_SECRET');   // startup'da tekshiriladi
  }
}
```

## 6. Common mistakes

- **`.env` faylini git'ga qo'shish** → `.gitignore` ga qo'y va sizib ketgan bo'lsa **secretlarni almashtir**.
- **`.env.example` ichiga haqiqiy qiymat yozish.**
- **`process.env` ni to'g'ridan-to'g'ri hamma joyda ishlatish** → tekshiruv yo'q, test qilish qiyin.
- **`get()` ishlatib, `undefined` bilan davom etish** → xato ancha keyin, tushunarsiz joyda chiqadi.
- **Validation schema yozmaslik** → xato env bilan deploy o'tib ketadi.
- **Secretni logga chiqarish** (startup'da config'ni to'liq loglash) → [[Logging in NestJS]].
- **Zaif `JWT_SECRET`** → uzun tasodifiy qiymat ishlat.

## 7. Practice

1. `ConfigModule` ni `isGlobal` va Joi validation bilan sozla.
2. `JWT_SECRET`ni o'chirib, app ishga tushmasligini tasdiqla.
3. `getOrThrow` va `get` farqini amalda ko'r.
4. `.env.example` yozib, `.env` ni `.gitignore` ga qo'sh.
5. Startup'da barcha kerakli env'lar borligini tekshiruvchi kichik funksiya yoz.

## 8. Related concepts

[[NestJS Application Bootstrap]] · [[Database Connection URLs]] · [[JWT Authentication]] · [[S3 and Cloudflare R2]] · [[Sentry]] · [[Logging in NestJS]]
