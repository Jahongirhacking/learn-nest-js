---
tags: [nestjs/lifecycle, level/beginner, comparison]
level: beginner
topic: Request Lifecycle
---

# Middleware vs Guard vs Pipe vs Interceptor vs Filter

> **Bir jumlada:** beshtasi ham "request oralig'ida ishlaydigan kod", lekin har biri **boshqa savolga** javob beradi — ularni chalkashtirish NestJS'dagi eng ko'p uchraydigan arxitektura xatosi.

**Before learning:** [[NestJS Request Lifecycle]]
**After learning:** [[FotSpot Guard Chain]], [[NestJS Mental Model]]

> 💡 **Tushunmadingmi? Express'da qo'lda qanday yozilishini ko'r:** [[Express Middleware Order]]


---

## 1. Definition — bitta jadval

| | Savoli | Qachon | Nimani ko'radi | Nimani qaytaradi |
|---|---|---|---|---|
| [[NestJS Middleware]] | "Requestni tayyorlaymanmi?" | Eng birinchi | raw `req`, `res` | `next()` |
| [[NestJS Guards]] | "Kirishga ruxsat bormi?" | Middleware'dan keyin | `ExecutionContext` + metadata | `true` / `false` |
| [[NestJS Interceptors]] | "Execution atrofida nima qilaman?" | Guard'dan keyin, handler atrofida | context + response oqimi | `Observable` |
| [[NestJS Pipes]] | "Input to'g'ri va kerakli tipdami?" | Handler'dan bevosita oldin | parametr qiymati | o'zgartirilgan qiymat |
| [[NestJS Exception Filters]] | "Xatoni qanday response qilaman?" | Exception otilganda | exception + host | HTTP response |

## 2. Why — nega farqi muhim?

Chunki noto'g'ri qatlamni tanlash quyidagilarga olib keladi:

- Authorization **middleware**da → `@Public()` kabi metadata naqshlari ishlamaydi.
- Validation **guard**da → DTO hali validatsiyadan o'tmagan, ya'ni ishonchsiz ma'lumot bilan qaror qabul qilinadi.
- Error formatlash **interceptor**da → guard/pipe otgan xatolar boshqa formatda chiqadi.
- Business logic **pipe**da → test qilib bo'lmaydi va qayta ishlatilmaydi.

## 3. Internal mental model

```text
Request
  ↓
Middleware      request preparation
  ↓
Guard           permission to enter
  ↓
Interceptor     wrap execution (before)
  ↓
Pipe            input transformation / validation
  ↓
Controller → Service
  ↓
Interceptor     wrap execution (after)
  ↓
Response

throw ─────────→ Filter → error response
```

Qisqa formula:

```text
Middleware  = request preparation
Guard       = permission to enter
Pipe        = input transformation/validation
Interceptor = wrap execution
Filter      = exception handling
```

## 4. FotSpot implementation

```text
Middleware   → helmet, CORS, request logging
Guard        → ThrottleGuard, JwtAuthGuard, RolesGuard, PermissionsGuard
Pipe         → global ValidationPipe (whitelist, forbidNonWhitelisted, transform)
Interceptor  → logging/timing, response serialization
Filter       → HttpExceptionFilter (yagona xato formati)
```

## 5. Code example — "qaysi birini tanlayman?"

```text
"Har bir requestga request-id qo'shmoqchiman"        → Middleware
"Faqat admin kira olsin"                              → Guard
"Query 'page' string kelyapti, number kerak"          → Pipe
"Barcha response { data: ... } bo'lsin"               → Interceptor
"Javob 5 soniyadan oshsa uzilsin"                     → Interceptor
"Prisma P2002 xatosi 409 bo'lib chiqsin"              → Filter
"passwordHash response'ga chiqmasin"                  → Interceptor (serializer)
"Bir daqiqada 10 tadan ko'p login bo'lmasin"          → Guard (throttle)
"Trial faqat 18 yoshgacha o'yinchiga ochiq"           → Service (business rule!)
```

Oxirgisiga e'tibor ber: **business qoida hech qaysi qatlamga emas, [[NestJS Services]] ga tegishli.**

## 6. Common mistakes

- Guard ichida DTO'ga ishonish (pipe hali ishlamagan).
- Middleware'da role tekshirish (metadata ko'rinmaydi).
- Filter o'rniga interceptor'da `catchError` bilan hamma xatoni formatlash.
- Pipe ichida DB query qilish.
- Har bir yangi ehtiyoj uchun global qatlam qo'shish — ba'zan oddiy service methodi yetadi.

## 7. Practice

Beshta qatlamning har biriga bittadan `console.log` qo'y va bitta requestda tartibni yozib ol.
Keyin guard'ni `false` qildir, exception tashla — tartib qanday o'zgaradi?
Oxirida yuqoridagi "qaysi birini tanlayman?" ro'yxatini yopib, o'zing javob berib chiq.

## 8. Related concepts

[[NestJS Request Lifecycle]] · [[NestJS Middleware]] · [[NestJS Guards]] · [[NestJS Pipes]] · [[NestJS Interceptors]] · [[NestJS Exception Filters]] · [[NestJS Mental Model]]
