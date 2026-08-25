---
tags: [nestjs, moc, map]
type: moc
source: "[[FotSpot_NestJS_Learning_Guide]]"
---

# NestJS Learning Map

> 💡 **Biror mavzu tushunarsizmi?** Uning Express'da qo'lda qanday yozilishini ko'r:
> **[[NestJS to Express Translation]]** — to'liq tarjima jadvali.


> **Bu nota — butun knowledge base'ning kirish nuqtasi (MOC).**
> Har bir concept alohida note. Ular bir-biriga wikilink orqali bog'langan.
> Graph'da markazda [[NestJS Request Lifecycle]] va [[NestJS Dependency Injection]] turadi.

## Qanday ishlatiladi?

1. Birinchi marta o'qiyotgan bo'lsang — [[NestJS Learning Path]] tartibida yur.
2. Har bir note'da **Before learning** va **After learning** bor. O'sha zanjirni buzma.
3. Har bir note oxirida **Practice** bor — o'qib qo'yish emas, yozib ko'rish kerak.
4. Concept tushunilganini [[NestJS Interview Questions]] bilan tekshir.

---

## 1. Framework fundamentals (beginner)

- [[NestJS Overview]] — NestJS nima, Express'dan farqi
- [[NestJS Application Bootstrap]] — `main.ts`, `NestFactory`, startup ketma-ketligi
- [[NestJS Modules]] — application boundary
- [[NestJS Controllers]] — HTTP entry point
- [[NestJS Providers]] — injectable hamma narsa
- [[NestJS Services]] — business logic joyi
- [[NestJS Dependency Injection]] — **fundamental note**
- [[NestJS Decorators]] — metadata mexanizmi
- [[TypeScript for NestJS]] — magic'ni yo'qotuvchi baza

## 2. Request lifecycle (beginner → intermediate)

- [[NestJS Request Lifecycle]] — **markaziy note**
- [[NestJS Middleware]]
- [[NestJS Guards]]
- [[NestJS Pipes]]
- [[NestJS Interceptors]]
- [[NestJS Exception Filters]]
- [[NestJS DTO]]
- [[NestJS ValidationPipe]]
- [[NestJS Custom Decorators]]

## 3. Authentication (intermediate)

- [[Authentication]] — "sen kimsan?"
- [[JWT Authentication]]
- [[Passport and JwtStrategy]]
- [[Password Hashing]]
- [[OTP Security]]
- [[Session Architecture]]

## 4. Authorization (intermediate)

- [[Authorization]] — "senga ruxsat bormi?"
- [[RBAC]]
- [[Roles and Permissions]]
- [[Public Routes Decorator]]

## 5. Database (intermediate)

- [[Prisma]]
- [[Prisma Schema]]
- [[Prisma Migrations]]
- [[Database Relations]]
- [[Database Transactions]]
- [[Database Connection URLs]]

## 6. Infrastructure (intermediate → advanced)

- [[Redis]]
- [[Redis Cache]]
- [[BullMQ]]
- [[Queue vs Synchronous Request]]
- [[S3 and Cloudflare R2]]
- [[Presigned Upload Flow]]
- [[WebSockets in NestJS]]

## 7. Production (advanced)

- [[ConfigModule and Environment Variables]]
- [[Rate Limiting and Throttling]]
- [[Helmet]]
- [[CORS]]
- [[Sentry]]
- [[Logging in NestJS]]
- [[Swagger and OpenAPI]]
- [[Testing in NestJS]]
- [[ESLint and Prettier]]

## 8. FotSpot implementation (framework emas — real loyiha)

- [[FotSpot Architecture Overview]]
- [[FotSpot Guard Chain]]
- [[FotSpot Feature Anatomy]]
- [[FotSpot Auth Architecture]]
- [[FotSpot Media Moderation]]
- [[FotSpot Source Reading Order]]
- [[Domain vs Infrastructure]]

## 9. Practice

- [[Build a Feature in 10 Steps]]
- [[NestJS Practice Challenges]]
- [[Mini Project Football Academy API]]
- [[NestJS Interview Questions]]
- [[NestJS Mental Model]]

---

## Muhim ajratish

```text
NestJS framework concepts   ←→   FotSpot business implementation
Prisma / Redis / R2         ←→   NestJS'ning o'zi emas (infrastructure)
Authentication              ←→   Authorization (ikkita alohida narsa)
Redis Cache                 ←→   BullMQ Redis (bir xil emas)
```

Bu chegaralarni aralashtirsang, architecture tushunilmay qoladi.
Batafsil: [[Domain vs Infrastructure]]


---

## 10. Express fundamenti — "magic"ni ochish

NestJS Express ustida ishlaydi. Biror concept tushunarsiz bo'lsa, uning
**qo'lda yozilgan** variantini ko'rish eng tez yo'l.

- [[NestJS to Express Translation]] — **kirish nuqtasi**: qaysi NestJS mavzusi qaysi Express note'ga mos keladi
- [[Express Learning Map]] — butun Express bo'limining xaritasi
- [[Express Middleware]] — Guard, Pipe, Interceptor, Filter — hammasining ildizi
- [[Express Manual Dependency Wiring]] — DI aslida nima ekani
- [[Express Project Structure Problem]] — NestJS nega paydo bo'lgan
- [[Express to NestJS Rebuild]] — bir xil API'ni ikkala frameworkda yozish mashqi
