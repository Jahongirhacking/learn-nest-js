---
tags: [nestjs/lifecycle, level/intermediate]
level: intermediate
topic: Request Lifecycle
---

# NestJS Interceptors

> **Bir jumlada:** Interceptor handler'ni **o'rab oladi**: undan oldin ham, keyin ham kod ishlata oladi va response'ni o'zgartira oladi.

**Before learning:** [[NestJS Request Lifecycle]], [[NestJS Guards]]
**After learning:** [[Logging in NestJS]], [[Sentry]], [[Swagger and OpenAPI]]

> 💡 **Tushunmadingmi? Express'da qo'lda qanday yozilishini ko'r:** [[Express Middleware Order]]


---

## 1. Definition

```ts
@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const start = Date.now();
    return next
      .handle()                                    // ← handler shu yerda ishlaydi
      .pipe(tap(() => console.log(`${Date.now() - start}ms`)));
  }
}
```

`next.handle()` — controller method'ining o'zi. Undan oldingi kod "before", `pipe()` ichidagisi "after".

## 2. Why — nega kerak?

Guard "ha/yo'q" deydi, Pipe inputni tekshiradi — lekin quyidagilar uchun ikkalasi ham mos emas:

```text
response formatini bir xillashtirish   → { data, meta }
javob vaqtini o'lchash
cache qaytarish (handler'ni umuman chaqirmaslik)
serializatsiya (passwordHash'ni response'dan olib tashlash)
timeout qo'yish
xatoni Sentry'ga yuborish
```

## 3. Internal mental model

```text
Guard → Interceptor(before) → Pipe → Handler → Interceptor(after) → Response
```

Interceptor RxJS `Observable` bilan ishlaydi:

```ts
return next.handle().pipe(
  map(data => ({ data, timestamp: Date.now() })),   // response o'zgartirish
  timeout(5000),                                    // vaqt chegarasi
  catchError(err => { /* ... */ throw err; }),      // xatoni ushlash
);
```

Muhim imkoniyat: `next.handle()` ni **umuman chaqirmaslik** mumkin — masalan cache'dan javob qaytarish.

## 4. FotSpot implementation

Tipik ishlatilishi:

```text
LoggingInterceptor        → method, URL, davomiylik
TransformInterceptor      → response'ni yagona formatga solish
ClassSerializerInterceptor→ @Exclude() maydonlarni olib tashlash (passwordHash!)
Sentry integratsiyasi     → error context yig'ish   → [[Sentry]]
```

Xavfsizlik nuqtai nazaridan eng muhimi — serializatsiya: `User` obyekti hech qachon `passwordHash`, `refreshTokenHash`, `otpHash` bilan birga response'ga chiqmasligi kerak ([[Password Hashing]], [[OTP Security]]).

## 5. Code example

```ts
// response'ni yagona formatga solish
@Injectable()
export class TransformInterceptor<T> implements NestInterceptor<T, { data: T }> {
  intercept(context: ExecutionContext, next: CallHandler) {
    return next.handle().pipe(map(data => ({ data })));
  }
}

// timeout
@Injectable()
export class TimeoutInterceptor implements NestInterceptor {
  intercept(_: ExecutionContext, next: CallHandler) {
    return next.handle().pipe(
      timeout(5000),
      catchError(err =>
        err instanceof TimeoutError
          ? throwError(() => new RequestTimeoutException())
          : throwError(() => err),
      ),
    );
  }
}
```

Global ro'yxatdan o'tkazish (DI bilan):

```ts
{ provide: APP_INTERCEPTOR, useClass: TransformInterceptor }
```

## 6. Common mistakes

- **`next.handle()` ni qaytarishni unutish** → response hech qachon yuborilmaydi.
- **Interceptor'ni Exception Filter o'rniga ishlatish** → xatolarni markazlashgan formatlash [[NestJS Exception Filters]] ishi.
- **Response transformer qo'shib, Swagger schema'sini yangilamaslik** → dokumentatsiya yolg'on bo'ladi ([[Swagger and OpenAPI]]).
- **`ClassSerializerInterceptor` ishlatib, `@Exclude()` qo'yishni unutish** → maxfiy maydonlar response'ga chiqadi.
- **RxJS'ni bilmasdan `subscribe()` chaqirish** → ikki marta ishlash.

## 7. Practice

1. `TransformInterceptor` yozib, barcha response `{ data: ... }` bo'lishini ta'minla.
2. `LoggingInterceptor`da before/after loglarni chiqar va guard logi bilan tartibini solishtir.
3. Cache interceptor yoz: agar Redis'da kalit bo'lsa, `next.handle()` ni chaqirmasdan javob qaytar ([[Redis Cache]]).
4. `ClassSerializerInterceptor` + `@Exclude() passwordHash` bilan response'ni tozala.

## 8. Related concepts

[[NestJS Request Lifecycle]] · [[Middleware vs Guard vs Pipe vs Interceptor vs Filter]] · [[NestJS Exception Filters]] · [[Redis Cache]] · [[Logging in NestJS]] · [[Sentry]]
