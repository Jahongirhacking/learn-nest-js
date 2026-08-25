---
tags: [express, moc, translation]
type: moc
---

# NestJS to Express Translation

> **Bu jadval — butun `ExpressJS/` papkasining maqsadi.**
> NestJS'da biror mavzu tushunarsiz bo'lsa, shu jadvaldan uning **Express'dagi qo'lda yozilgan varianti**ni top va o'sha note'ni o'qi. Keyin NestJS note'ga qayt — endi u magic bo'lmaydi.

**Boshlash:** [[Express Learning Map]] · **Qaytish:** [[NestJS Learning Map]]

---

## Asosiy tarjima jadvali

| NestJS'da tushunmagan narsang | Express'da uning haqiqiy ko'rinishi | Note |
|---|---|---|
| [[NestJS Overview]] | Express nima beradi va nima bermaydi | [[Express Overview]] |
| [[NestJS Application Bootstrap]] | `express()` + `app.listen()` | [[Express App Bootstrap]] |
| [[NestJS Request Lifecycle]] | **Middleware zanjiri** — hammasi shu | [[Express Middleware]] |
| [[NestJS Controllers]] | `app.get('/players', handler)` | [[Express Route Handlers]] |
| [[NestJS Modules]] | `express.Router()` + `require` | [[Express Router]] |
| [[NestJS Services]] | Oddiy class/funksiya, qo'lda ajratilgan | [[Express Service Layer]] |
| [[NestJS Dependency Injection]] | `new Service(dep)` — qo'lda ulash | [[Express Manual Dependency Wiring]] |
| [[NestJS Providers]] | `module.exports = new Service()` | [[Express Manual Dependency Wiring]] |
| [[NestJS Decorators]] | Decorator yo'q — `app.get()` chaqiruvining o'zi | [[Express Routing]] |
| [[NestJS Middleware]] | `app.use(fn)` | [[Express Middleware]] |
| [[NestJS Guards]] | `function requireAuth(req, res, next)` | [[Express Auth Middleware]] |
| [[NestJS Pipes]] | Validatsiya qiluvchi middleware | [[Express Validation Middleware]] |
| [[NestJS Interceptors]] | `res.json` ni o'rash / `res.on('finish')` | [[Express Middleware Order]] |
| [[NestJS Exception Filters]] | 4 argumentli error middleware | [[Express Error Handling Middleware]] |
| [[NestJS DTO]] | Qo'lda yozilgan `if` lar yoki zod/Joi schema | [[Express Request Validation]] |
| [[NestJS ValidationPipe]] | Har bir route uchun qo'lda validate | [[Express Validation Middleware]] |
| [[Middleware vs Guard vs Pipe vs Interceptor vs Filter]] | Express'da **hammasi bitta narsa: middleware** | [[Express Middleware Order]] |
| [[Authentication]] | `jsonwebtoken` bilan qo'lda | [[Express Authentication]] |
| [[Authorization]] | `requireRole('admin')` middleware | [[Express Authorization Middleware]] |
| [[Passport and JwtStrategy]] | `passport.use(new JwtStrategy(...))` | [[Express Passport]] |
| [[Public Routes Decorator]] | Middleware'ni qaysi route'ga qo'yish masalasi | [[Express Middleware Order]] |
| [[Prisma]] | `const prisma = new PrismaClient()` — global singleton | [[Express Prisma Setup]] |
| [[Redis]] · [[BullMQ]] | Aynan bir xil kutubxonalar, DI'siz | [[Express Redis and Queues]] |
| [[ConfigModule and Environment Variables]] | `dotenv` + `process.env` | [[Express Config and Env]] |
| [[Helmet]] · [[CORS]] | `app.use(helmet())`, `app.use(cors())` | [[Express Security Middleware]] |
| [[Rate Limiting and Throttling]] | `express-rate-limit` | [[Express Rate Limiting]] |
| [[Logging in NestJS]] | `morgan` yoki qo'lda `console.log` | [[Express Logging]] |
| [[Testing in NestJS]] | `supertest(app)` | [[Express Testing]] |
| [[Database Transactions]] | `prisma.$transaction` — **aynan bir xil** | [[Express Prisma Setup]] |

---

## NestJS'da mavjud, Express'da MAVJUD EMAS

Bu ro'yxat muhim — NestJS'ning qo'shgan qiymati aynan shu:

```text
Dependency Injection container    → Express'da yo'q, qo'lda ulaysan
Module tizimi va boundary         → Express'da yo'q, papka nomlari bilan taqlid qilasan
Decorator metadata                → Express'da yo'q, funksiya chaqiruvi
Guard / Pipe / Interceptor farqi  → Express'da hammasi middleware
Avtomatik DTO validatsiya         → Express'da qo'lda
Async xatoni avtomatik ushlash    → Express 4'da qo'lda try/catch
Testing module (mock almashtirish) → Express'da qo'lda mock qilasan
Bir xil kod yozish qoidasi         → Express'da har jamoa o'zicha
```

## Express'da ochiq, NestJS'da yashirilgan

```text
req / res — Node'ning o'zi         → NestJS ularni decorator ortiga yashiradi
next()                             → NestJS lifecycle uni o'zi chaqiradi
Middleware TARTIBI                 → NestJS tartibni belgilab qo'ygan
Route ro'yxatga olish              → NestJS decoratorlardan quradi
Javob yozish (res.json)            → NestJS `return` ni o'zi res.json qiladi
```

**Muhim xulosa:**

> NestJS Express ustida ishlaydi. NestJS'da "magic" ko'ringan har bir narsa —
> aslida kimdir Express'da qo'lda yozadigan kodning avtomatlashtirilgan varianti.

---

## Qanday ishlatish kerak

```text
1. NestJS note'ni o'qidim, tushunmadim
2. Yuqoridagi jadvaldan Express variantini topdim
3. Express note'ni o'qidim va KODNI O'Z QO'LIM BILAN yozdim
4. Practice problem'larni yechdim
5. NestJS note'ga qaytdim → endi "nima uchun shunday" tushunarli
```

Batafsil metodika: [[Express to NestJS Rebuild]]

## Related

[[Express Learning Map]] · [[Express Mental Model]] · [[NestJS Learning Map]] · [[NestJS Request Lifecycle]]
