---
tags: [nestjs, moc, learning-path]
type: moc
---

# NestJS Learning Path

> **Bir jumlada:** conceptlarni tasodifiy emas, dependency tartibida o'rgan — har bosqich oldingisiga suyanadi.

**Before learning:** [[NestJS Learning Map]]
**After learning:** [[Mini Project Football Academy API]]

> 💡 **Tushunmadingmi? Express'da qo'lda qanday yozilishini ko'r:** [[Express Learning Map]]


Har bosqichda: **o'qi → yoz → tushuntir**. Uchinchisi bo'lmasa, bosqich tugamagan.

---

## Bosqich 1 — Fundamentals `beginner`

O'rgan: [[NestJS Overview]] → [[NestJS Application Bootstrap]] → [[NestJS Modules]] → [[NestJS Controllers]] → [[NestJS Providers]] → [[NestJS Services]] → [[NestJS Dependency Injection]] → [[NestJS Decorators]]

Amaliyot: `GET /hello` endpoint. Faqat module + controller + service.

Tekshiruv: "Nest `PlayersService` instance'ini kim yaratadi?" degan savolga javob bera olasanmi?

---

## Bosqich 2 — Request lifecycle `beginner`

O'rgan: [[NestJS Request Lifecycle]] → [[NestJS Middleware]] → [[NestJS Guards]] → [[NestJS Pipes]] → [[NestJS Interceptors]] → [[NestJS Exception Filters]] → [[Middleware vs Guard vs Pipe vs Interceptor vs Filter]]

Amaliyot: request logger middleware + oddiy auth guard + validation pipe + global exception filter.

Tekshiruv: log tartibini oldindan aytib, keyin tekshir.

---

## Bosqich 3 — DTO va validation `beginner`

O'rgan: [[NestJS DTO]] → [[NestJS ValidationPipe]] → [[NestJS Custom Decorators]]

Amaliyot: `POST /users` uchun `CreateUserDto`, `whitelist` + `forbidNonWhitelisted` + `transform` bilan.

Tekshiruv: `{"isAdmin": true}` yuborganda nima bo'ladi va **nega** bu security masalasi?

---

## Bosqich 4 — Database `intermediate`

O'rgan: [[Prisma]] → [[Prisma Schema]] → [[Database Relations]] → [[Prisma Migrations]] → [[Database Connection URLs]] → [[Database Transactions]]

Amaliyot: `User` va `PlayerProfile` modellari, relation, migration, seed.

---

## Bosqich 5 — Authentication `intermediate`

O'rgan: [[Authentication]] → [[Password Hashing]] → [[JWT Authentication]] → [[Passport and JwtStrategy]] → [[Session Architecture]] → [[OTP Security]]

Amaliyot: `register`, `login`, `me`, `refresh`, `logout` flow.

---

## Bosqich 6 — Authorization `intermediate`

O'rgan: [[Authorization]] → [[RBAC]] → [[Roles and Permissions]] → [[Public Routes Decorator]]

Amaliyot: `admin`, `coach`, `player` rollari bilan endpointlarni himoyalash.

---

## Bosqich 7 — Redis va queues `intermediate`

O'rgan: [[Redis]] → [[Redis Cache]] → [[Rate Limiting and Throttling]] → [[BullMQ]] → [[Queue vs Synchronous Request]]

Amaliyot: `POST /notifications` requestini queue orqali backgroundda ishlat.

---

## Bosqich 8 — External services `intermediate`

O'rgan: [[S3 and Cloudflare R2]] → [[Presigned Upload Flow]]

Amaliyot: upload URL berish + metadata saqlash flow'i.

---

## Bosqich 9 — Production architecture `advanced`

O'rgan: [[ConfigModule and Environment Variables]] → [[CORS]] → [[Helmet]] → [[Logging in NestJS]] → [[Sentry]] → [[Swagger and OpenAPI]] → [[Testing in NestJS]]

Amaliyot: health endpoint, env-based log level, Swagger `/docs`.

---

## Bosqich 10 — Advanced `advanced`

O'rgan: [[WebSockets in NestJS]] → [[Database Transactions]] → [[NestJS Custom Decorators]] → [[Domain vs Infrastructure]] → [[FotSpot Architecture Overview]]

Amaliyot: [[Mini Project Football Academy API]] ni to'liq qur, keyin [[FotSpot Source Reading Order]] bo'yicha real kodni o'qi.

---

## Umumiy roadmap (bir qarashda)

```text
TypeScript
   → NestJS Fundamentals (Module, Controller, Provider, DI)
   → Request Lifecycle (Middleware, Guard, Pipe, Interceptor, Filter)
   → DTO + Validation
   → PostgreSQL + Prisma
   → Authentication (JWT, Passport, Session, OTP)
   → Authorization (RBAC, Permissions)
   → Redis (cache, rate limit)
   → BullMQ (queue, worker, retry)
   → Storage (S3 / R2 / presigned)
   → Real-time (WebSocket)
   → Production (Docker, Security, Sentry, Swagger, Logging)
   → Advanced (Transactions, Testing, Performance, Observability)
```

## Related concepts

[[NestJS Learning Map]] · [[NestJS Practice Challenges]] · [[NestJS Interview Questions]] · [[Build a Feature in 10 Steps]]
