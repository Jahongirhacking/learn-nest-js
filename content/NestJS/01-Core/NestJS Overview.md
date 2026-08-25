---
tags: [nestjs/core, level/beginner]
level: beginner
topic: NestJS Core
---

# NestJS Overview

> **Bir jumlada:** NestJS — Node.js uchun TypeScript-first backend framework bo'lib, uning asosiy qiymati kutubxona emas, **oldindan berilgan application architecture**.

**Before learning:** [[TypeScript for NestJS]]
**After learning:** [[NestJS Application Bootstrap]], [[NestJS Modules]], [[NestJS Mental Model]]

> 💡 **Tushunmadingmi? Express'da qo'lda qanday yozilishini ko'r:** [[Express Overview]]


---

## 1. Definition

NestJS Express yoki Fastify ustida ishlaydi, lekin ular bermaydigan narsani beradi: **struktura**.

```text
Express beradi:      route + middleware
NestJS qo'shadi:     Module, Controller, Provider, DI, Guard, Pipe, Interceptor, Filter
```

## 2. Why — nega kerak?

Express bilan boshlash oson:

```text
Route → Handler → Database
```

Loyiha o'sganda esa `routes/`, `controllers/`, `services/`, `auth/`, `utils/`, `middleware/` bir-biriga aralashib ketadi — chunki hech kim chegara qo'ymagan.

NestJS chegarani **oldindan** beradi:

```text
Module
 ├── Controller
 ├── Service
 ├── Providers
 ├── Guards
 ├── DTO
 └── ...
```

Natija: 5 kishilik jamoa ham, 50 ta feature ham bir xil naqshda yoziladi.

## 3. Internal mental model

NestJS'ni uch qatlam sifatida ko'r:

```text
1. Metadata layer   → decoratorlar (@Module, @Controller, @Injectable)
2. DI container     → metadata asosida object graph quriladi
3. HTTP adapter     → Express/Fastify route'larga bog'lanadi
```

Ya'ni sen "nima kerakligini" e'lon qilasan, Nest "qanday ulashni" o'zi hal qiladi.
Bu inversion of control — [[NestJS Dependency Injection]].

## 4. FotSpot implementation

FotSpot — NestJS + TypeScript REST API. **Feature-based architecture**:

```text
backend/src/
  auth/            users/          players/
  coaches/         academies/      trials/
  recommendations/ media/          moderation/
  notifications/   prisma/         redis/
  common/          rbac/           storage/
```

Har bir feature o'z moduliga ega — [[FotSpot Feature Anatomy]].

Ishlatilgan stack:

```text
NestJS · TypeScript · Prisma · PostgreSQL · Redis · BullMQ
Passport/JWT · argon2 · Socket.IO · Swagger · Helmet · Sentry
AWS S3 SDK (Cloudflare R2) · Jest · ESLint/Prettier
```

## 5. Code example

Eng kichik to'liq NestJS application:

```ts
// app.module.ts
@Module({
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}

// app.controller.ts
@Controller()
export class AppController {
  constructor(private readonly appService: AppService) {}

  @Get('health')
  health() {
    return { status: 'ok' };
  }
}
```

## 6. Common mistakes

- **NestJS'ni Express'ning "chiroyliroq sintaksisi" deb tushunish.** Asosiy farq — DI va modular boundary.
- **Framework'ni o'rganishni decoratorlardan boshlash.** Avval [[NestJS Modules]] va [[NestJS Dependency Injection]].
- **Prisma/Redis'ni NestJS qismi deb o'ylash.** Ular mustaqil texnologiyalar — [[Domain vs Infrastructure]].

## 7. Practice

1. `nest new` bilan bo'sh project yarat.
2. `GET /health` endpoint qo'sh (yuqoridagi kod).
3. `AppService`ga bitta method qo'shib, controller'dan chaqir — `new AppService()` **yozmasdan**.
4. O'zingga tushuntir: instance qayerdan keldi?

## 8. Related concepts

[[NestJS Modules]] · [[NestJS Dependency Injection]] · [[NestJS Application Bootstrap]] · [[NestJS Mental Model]] · [[FotSpot Architecture Overview]]
