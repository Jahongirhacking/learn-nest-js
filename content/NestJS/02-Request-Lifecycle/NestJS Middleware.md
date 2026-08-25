---
tags: [nestjs/lifecycle, level/beginner]
level: beginner
topic: Request Lifecycle
---

# NestJS Middleware

> **Bir jumlada:** Middleware — lifecycle'ning eng birinchi bosqichi; u hali Nest context'ini bilmaydi, faqat raw `request` / `response` bilan ishlaydi.

**Before learning:** [[NestJS Request Lifecycle]]
**After learning:** [[NestJS Guards]], [[Logging in NestJS]], [[Helmet]]

> 💡 **Tushunmadingmi? Express'da qo'lda qanday yozilishini ko'r:** [[Express Middleware]]


---

## 1. Definition

```ts
@Injectable()
export class RequestLoggerMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    console.log(`${req.method} ${req.originalUrl}`);
    next();                       // ← chaqirilmasa request osilib qoladi
  }
}
```

Bu Express middleware'ning aynan o'zi, faqat class ko'rinishida (DI ishlashi uchun).

## 2. Why — nega kerak?

Ba'zi ishlar routing va Nest qarorlaridan **oldin** bo'lishi kerak:

```text
request id generatsiya qilish
raw body saqlash (webhook signature uchun)
security header qo'yish        → [[Helmet]]
CORS                            → [[CORS]]
request logging / timing
```

## 3. Internal mental model

```text
HTTP Request
   ↓
Middleware        ← raw req/res, ExecutionContext YO'Q
   ↓
Guard             ← ExecutionContext BOR (handler/class metadata ko'rinadi)
   ↓
Interceptor → Pipe → Controller
```

Asosiy farq:

```text
Middleware → "qaysi route ekanini deyarli bilmaydi"
Guard      → "qaysi handler, qaysi metadata — hammasini biladi"
```

Shuning uchun authorization **guard**da yoziladi, middleware'da emas.

Ro'yxatdan o'tkazish:

```ts
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(RequestLoggerMiddleware)
      .exclude({ path: 'health', method: RequestMethod.GET })
      .forRoutes('*');
  }
}
```

## 4. FotSpot implementation

FotSpot'da middleware qatlami asosan infratuzilma uchun:

```text
helmet()          → security headerlar        → [[Helmet]]
CORS              → origin siyosati            → [[CORS]]
request logging   → method, URL, davomiylik    → [[Logging in NestJS]]
```

Authentication esa **middleware emas**, guard: `JwtAuthGuard` — chunki u `@Public()` metadata'sini o'qishi kerak ([[Public Routes Decorator]]), bu esa faqat `ExecutionContext` bilan mumkin.

## 5. Code example

```ts
@Injectable()
export class RequestTimingMiddleware implements NestMiddleware {
  private readonly logger = new Logger('HTTP');

  use(req: Request, res: Response, next: NextFunction) {
    const start = Date.now();
    res.on('finish', () => {
      this.logger.log(
        `${req.method} ${req.originalUrl} ${res.statusCode} ${Date.now() - start}ms`,
      );
    });
    next();
  }
}
```

## 6. Common mistakes

- **`next()` ni chaqirmaslik** → request javobsiz osilib qoladi (timeout).
- **Middleware'da authorization qilish** → metadata ko'rinmaydi, `@Public()` kabi naqshlar ishlamaydi.
- **Middleware'ni Interceptor bilan chalkashtirish** → interceptor response'ni ham o'zgartira oladi, middleware amalda faqat oldin ishlaydi.
- **Og'ir ishni (DB query) har bir requestda middleware'da qilish** → butun app sekinlashadi.

## 7. Practice

1. `RequestTimingMiddleware` yozib, `forRoutes('*')` bilan ulan.
2. `/health` ni `exclude` qil va logdan yo'qolganini tekshir.
3. `next()` ni ataylab o'chir — nima bo'ladi?
4. Middleware, guard, pipe va controller'ga log qo'yib, to'liq tartibni yozib ol.

## 8. Related concepts

[[NestJS Request Lifecycle]] · [[Middleware vs Guard vs Pipe vs Interceptor vs Filter]] · [[NestJS Guards]] · [[NestJS Interceptors]] · [[Helmet]] · [[CORS]]
