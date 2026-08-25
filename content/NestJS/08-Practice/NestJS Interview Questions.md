---
tags: [practice, level/intermediate, review]
level: intermediate
topic: Practice
---

# NestJS Interview Questions

> **Bir jumlada:** har bir savolga **ovoz chiqarib** javob bera olsang — o'sha concept o'zlashtirilgan; javob bera olmasang, yonidagi note'ga qayt.

**Before learning:** [[NestJS Learning Path]]
**After learning:** [[NestJS Mental Model]]

---

## NestJS fundamentals

```text
1. NestJS va Express farqi nima?              → [[NestJS Overview]]
2. Module nima va nima uchun kerak?           → [[NestJS Modules]]
3. Provider nima? Faqat service'mi?           → [[NestJS Providers]]
4. Dependency Injection qanday ishlaydi?      → [[NestJS Dependency Injection]]
5. @Injectable() aslida nima qiladi?          → [[NestJS Decorators]]
6. Controller nima qilishi KERAK EMAS?        → [[NestJS Controllers]]
7. Service qatlami nima uchun kerak?          → [[NestJS Services]]
8. exports bo'lmasa nima bo'ladi?             → [[NestJS Modules]]
```

## Lifecycle

```text
9.  Middleware, Guard, Pipe, Interceptor, Filter — tartibi qanday?
10. Guard va middleware farqi nima?
11. Nega Guard ichida DTO'ga ishonib bo'lmaydi?
12. Interceptor qachon Filter o'rnini bosa olmaydi?
13. next.handle() ni chaqirmaslik nimaga olib keladi?
    → [[NestJS Request Lifecycle]] · [[Middleware vs Guard vs Pipe vs Interceptor vs Filter]]
```

## Validation

```text
14. TypeScript type va DTO farqi nima?
15. whitelist va forbidNonWhitelisted nima qiladi?
16. Bu ikkisi qaysi hujumdan himoya qiladi?
17. transform: true bo'lmasa nima ishlamaydi?
    → [[NestJS DTO]] · [[NestJS ValidationPipe]]
```

## Authentication va Authorization

```text
18. Authentication va Authorization farqi nima?
19. JWT qanday ishlaydi? U shifrlanganmi?
20. Guard va Strategy farqi nima?              → [[Passport and JwtStrategy]]
21. Access va refresh token farqi nima?
22. Refresh token nega hash qilib saqlanadi?
23. Session table nega kerak, JWT stateless-ku?  → [[Session Architecture]]
24. Nega @Public() naqshi @UseGuards dan xavfsizroq? → [[Public Routes Decorator]]
25. Role va Permission farqi nima?             → [[Roles and Permissions]]
26. Ownership tekshiruvi nega guard'da emas?   → [[Authorization]]
27. OTP nega hash qilinadi va yana nima kerak? → [[OTP Security]]
```

## Database

```text
28. ORM nima beradi va nimani yashiradi?       → [[Prisma]]
29. migrate dev va migrate deploy farqi nima?  → [[Prisma Migrations]]
30. Transaction qachon kerak?                  → [[Database Transactions]]
31. Index nima uchun va qachon zarar keltiradi? → [[Prisma Schema]]
32. N+1 query nima va qanday tuzatiladi?       → [[Database Relations]]
33. DATABASE_URL va DIRECT_URL nega alohida?   → [[Database Connection URLs]]
```

## Redis va queues

```text
34. Redis nega kerak, PostgreSQL yetmaydimi?   → [[Redis]]
35. Cache va queue semantikasi nega har xil?   → [[Redis Cache]]
36. BullMQ uchun nega boshqa Redis config kerak?
37. Qaysi ishni queue'ga chiqarish kerak?      → [[Queue vs Synchronous Request]]
38. Job idempotent bo'lishi nima demak?        → [[BullMQ]]
```

## Security

```text
39. CORS nima va u himoyami?                   → [[CORS]]
40. Helmet nima qiladi?                        → [[Helmet]]
41. Rate limiting nega guard zanjirining boshida? → [[Rate Limiting and Throttling]]
42. DTO validation securityga qanday yordam beradi?
43. Presigned URL nega API orqali yuklashdan xavfsizroq va tejamli? → [[Presigned Upload Flow]]
```

## Architecture

```text
44. Controller nega database bilan bevosita ishlamasligi kerak?
45. Module boundary nima?
46. Domain va infrastructure farqi nima?       → [[Domain vs Infrastructure]]
47. Guard zanjiri tartibi nega aynan shunday?  → [[FotSpot Guard Chain]]
48. Yangi feature qurishning 10 qadami qanday? → [[Build a Feature in 10 Steps]]
```

---

## Yakuniy test

Quyidagi oqimni **kodga qaramasdan** tushuntirib ber:

```text
Request
 → ThrottleGuard
 → JwtAuthGuard
 → RolesGuard
 → PermissionsGuard
 → DTO Validation
 → Controller
 → Service
 → Prisma / Redis / Queue
 → Response
 → Exception Filter
```

Har bir bo'g'in uchun ayt: **nima qiladi, nega shu yerda turadi, bo'lmasa nima buziladi.**

Agar buni erkin tushuntira olsang — NestJS arxitekturasini o'zlashtirgansan.

## Related concepts

[[NestJS Mental Model]] · [[NestJS Request Lifecycle]] · [[NestJS Learning Path]] · [[FotSpot Architecture Overview]]
