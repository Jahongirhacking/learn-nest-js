---
tags: [nestjs/lifecycle, level/beginner]
level: beginner
topic: Request Lifecycle
---

# NestJS Exception Filters

> **Bir jumlada:** Filter — lifecycle'ning oxirgi to'ri: istalgan joyda otilgan exception'ni ushlab, uni **yagona formatdagi** HTTP response'ga aylantiradi.

**Before learning:** [[NestJS Request Lifecycle]], [[NestJS Services]]
**After learning:** [[Sentry]], [[Logging in NestJS]]

> 💡 **Tushunmadingmi? Express'da qo'lda qanday yozilishini ko'r:** [[Express Error Handling Middleware]]


---

## 1. Definition

```ts
@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const res = ctx.getResponse<Response>();
    const status = exception.getStatus();

    res.status(status).json({
      statusCode: status,
      message: exception.getResponse(),
      path: ctx.getRequest().url,
      timestamp: new Date().toISOString(),
    });
  }
}
```

## 2. Why — nega kerak?

Service'lar shunchaki domain xatosini aytadi:

```ts
throw new NotFoundException('Player not found');
throw new ForbiddenException('Not your academy');
```

Ular HTTP javob formatini bilmasligi kerak. Filter esa **barcha** endpointlar uchun bir xil JSON strukturasini kafolatlaydi — frontend uchun bu juda muhim.

## 3. Internal mental model

```text
Controller / Service / Guard / Pipe
        ↓ throw
   Exception Filter
        ↓
   standard JSON response
```

Nest'ning built-in exceptionlari:

```text
BadRequestException        400
UnauthorizedException      401
ForbiddenException         403
NotFoundException          404
ConflictException          409
UnprocessableEntityException 422
TooManyRequestsException   429
InternalServerErrorException 500
```

`@Catch()` (parametrsiz) — **hamma narsani** ushlaydi, shu jumladan Prisma xatolarini ham.

## 4. FotSpot implementation

FotSpot'da global `HttpExceptionFilter` bor ([[NestJS Application Bootstrap]]).

Uning vazifalari:

```text
1. Yagona response shakli
2. 5xx xatolarni loglash, 4xx larni shovqin qilmaslik   → [[Logging in NestJS]]
3. Ichki detallarni yashirish (stack trace clientga chiqmaydi)
4. Kuzatuv tizimiga yuborish                            → [[Sentry]]
```

To'rtinchi nuqta muhim: `PrismaClientKnownRequestError` (masalan `P2002` — unique constraint) clientga "Internal server error" emas, `409 Conflict` bo'lib chiqishi kerak.

## 5. Code example

```ts
@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  private readonly logger = new Logger('Exception');

  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const res = ctx.getResponse<Response>();
    const req = ctx.getRequest<Request>();

    let status = HttpStatus.INTERNAL_SERVER_ERROR;
    let message: unknown = 'Internal server error';

    if (exception instanceof HttpException) {
      status = exception.getStatus();
      message = exception.getResponse();
    } else if (exception instanceof Prisma.PrismaClientKnownRequestError) {
      if (exception.code === 'P2002') {
        status = HttpStatus.CONFLICT;
        message = 'Resource already exists';
      }
    }

    if (status >= 500) {
      this.logger.error(`${req.method} ${req.url}`, (exception as Error)?.stack);
    }

    res.status(status).json({
      statusCode: status,
      message,
      path: req.url,
      timestamp: new Date().toISOString(),
    });
  }
}
```

## 6. Common mistakes

- **Controller'da `try/catch` qilib xatoni yutib yuborish** → filter hech qachon ko'rmaydi, xato yashirin qoladi.
- **Xato matnida ichki ma'lumot berish** (`"column users.password_hash"`) → axborot sizishi.
- **Har bir 400 xatoni `error` darajasida loglash** → production loglari shovqinga to'ladi.
- **`res.status(...).json(...)` ni controller'da qo'lda yozish** → format buziladi.
- **Filter ichidan yana exception tashlash** → javob umuman ketmaydi.

## 7. Practice

1. Global filter yozib, response formatini `{ statusCode, message, path, timestamp }` ga keltir.
2. Service'da `NotFoundException` tashlab, natijani tekshir.
3. Prisma unique constraint xatosini (`P2002`) 409'ga aylantir.
4. 500 xatoda stack trace faqat logda qolib, response'da chiqmasligini tasdiqla.

## 8. Related concepts

[[NestJS Request Lifecycle]] · [[Middleware vs Guard vs Pipe vs Interceptor vs Filter]] · [[NestJS Services]] · [[Sentry]] · [[Logging in NestJS]] · [[Prisma]]
