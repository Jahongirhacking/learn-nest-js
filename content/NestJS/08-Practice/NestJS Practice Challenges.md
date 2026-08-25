---
tags: [practice, level/beginner, challenges]
level: beginner
topic: Practice
---

# NestJS Practice Challenges

> **Bir jumlada:** 15 ta kichik, mustaqil vazifa — oson'dan qiyinga; har biri bitta concept'ni **yozib** mustahkamlaydi.

**Before learning:** [[NestJS Learning Path]]
**After learning:** [[Mini Project Football Academy API]], [[NestJS Interview Questions]]

> 💡 **Tushunmadingmi? Express'da qo'lda qanday yozilishini ko'r:** [[Express Practice Problems]]


Har bir challenge'dan keyin o'zingga savol ber: *"Buni nega shunday qildim?"*

---

## Beginner

### Challenge 1 — `GET /health`
Health endpoint yarat. Keyin uni `@Public()` va `@SkipThrottle()` qil.
→ [[NestJS Controllers]]

### Challenge 2 — Global ValidationPipe
`whitelist`, `forbidNonWhitelisted`, `transform` bilan yoq va uchalasining ta'sirini alohida sinab ko'r.
→ [[NestJS ValidationPipe]]

### Challenge 3 — CreatePlayerDto
Validator decoratorlari bilan DTO yoz; `{"firstName": 123}` va `{"isAdmin": true}` yuborib javoblarni solishtir.
→ [[NestJS DTO]]

### Challenge 4 — Prisma Player model
`User` bilan 1-1 relation, index va migration bilan.
→ [[Prisma Schema]] · [[Prisma Migrations]]

---

## Intermediate

### Challenge 5 — JWT login
argon2 hash + access/refresh token chiqarish.
→ [[Password Hashing]] · [[JWT Authentication]]

### Challenge 6 — `@Public()` decorator
`SetMetadata` + `JwtAuthGuard` integratsiyasi; guest va authenticated ikkalasi ham o'tsin.
→ [[Public Routes Decorator]]

### Challenge 7 — `@Roles('admin')` decorator
Metadata yozuvchi decorator.
→ [[NestJS Custom Decorators]]

### Challenge 8 — RolesGuard
`Reflector` bilan metadata o'qib, 403 qaytar.
→ [[NestJS Guards]]

### Challenge 9 — Permission system
`Role → RolePermission → Permission` zanjiri + `PermissionsGuard`.
→ [[RBAC]] · [[Roles and Permissions]]

### Challenge 10 — Redis counter
`INCR` + `EXPIRE` bilan hisoblagich; ikkita instance'da umumiy ekanini tekshir.
→ [[Redis]]

### Challenge 11 — BullMQ queue
Retry va backoff bilan job; ataylab yiqitib, qayta urinishni kuzat.
→ [[BullMQ]]

### Challenge 12 — R2 presigned upload
`upload-url` → `PUT` → `complete` flow.
→ [[Presigned Upload Flow]]

---

## Advanced

### Challenge 13 — Swagger documentation
`@ApiOperation`, `@ApiResponse`, `addBearerAuth()`; `/docs` ni himoyala.
→ [[Swagger and OpenAPI]]

### Challenge 14 — Global exception filter
Prisma `P2002` ni 409 ga aylantir, 5xx ni logla, stack trace'ni yashir.
→ [[NestJS Exception Filters]]

### Challenge 15 — WebSocket notification
JWT bilan himoyalangan gateway, `user:<id>` room, BullMQ worker'dan emit.
→ [[WebSockets in NestJS]]

---

## Qo'shimcha challenge'lar

```text
16. Transaction: uchta bog'liq yozuvni atomik qil        → [[Database Transactions]]
17. Cache: permissionlarni Redis'da 5 daqiqaga sakla     → [[Redis Cache]]
18. Interceptor: barcha response { data: ... } bo'lsin   → [[NestJS Interceptors]]
19. Middleware: request-id qo'shib, loglarga ulang       → [[Logging in NestJS]]
20. E2E test: guard zanjirini to'liq qamra               → [[Testing in NestJS]]
```

## Related concepts

[[NestJS Learning Path]] · [[Build a Feature in 10 Steps]] · [[Mini Project Football Academy API]] · [[NestJS Interview Questions]]
