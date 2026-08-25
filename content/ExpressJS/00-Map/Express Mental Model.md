---
tags: [express, level/beginner, summary]
level: beginner
topic: Express Foundations
---

# Express Mental Model

> **Bir jumlada:** Express — bu funksiyalar ro'yxati; request shu ro'yxat bo'ylab yuqoridan pastga yuradi va kimdir javob yozguncha davom etadi.

**Before learning:** [[Node HTTP Server]]
**After learning:** [[Express Middleware]], [[NestJS Mental Model]]

**NestJS'dagi mos note:** [[NestJS Mental Model]]

---

## 1. Butun Express bitta rasmda

```text
                    app (funksiyalar ro'yxati)
                             │
   Request  ──────────────▶  │
                             ▼
                    ┌─────────────────┐
                    │ middleware 1    │  next() ──┐
                    └─────────────────┘           │
                    ┌─────────────────┐  ◀────────┘
                    │ middleware 2    │  next() ──┐
                    └─────────────────┘           │
                    ┌─────────────────┐  ◀────────┘
                    │ route handler   │  res.json(...)  ──▶ Response
                    └─────────────────┘
                             │
                    (xato bo'lsa next(err))
                             ▼
                    ┌─────────────────┐
                    │ error middleware│  (err, req, res, next)
                    └─────────────────┘
```

Hammasi shu. Express'da boshqa hech qanday tushuncha yo'q.

## 2. Uchta haqiqat

```text
1. Hamma narsa middleware
   route handler ham middleware — faqat u odatda javob yozadi

2. Tartib = mantiq
   Express faylda YOZILGAN tartibda ishlatadi.
   Yuqoriga ko'chirsang — xatti-harakat o'zgaradi.

3. next() chaqirilmasa — hech narsa bo'lmaydi
   Request osilib qoladi. Xato ham bermaydi. Shunchaki javob kelmaydi.
```

## 3. NestJS bu modelni qanday o'zgartiradi

| Express | NestJS |
|---|---|
| Funksiyalar ro'yxati | Struktura: Module → Controller → Service |
| Tartibni **sen** belgilaysan | Tartibni **framework** belgilaydi (Guard→Pipe→Handler) |
| Hammasi `(req,res,next)` | Har biri o'z tipi: Guard, Pipe, Interceptor, Filter |
| `next()` ni qo'lda chaqirasan | Lifecycle o'zi boshqaradi |
| Dependency'ni `require` qilasan | DI container beradi |
| Xatoni har joyda `try/catch` | `throw` qilasan, Filter ushlaydi |

Ya'ni:

```text
Express  = "sen boshqarasan, sen javobgarsan"
NestJS   = "framework boshqaradi, sen qoidaga bo'ysunasan"
```

Ikkalasi ham to'g'ri yondashuv — masshtabga bog'liq. Nega NestJS kerak bo'lib qolishini [[Express Project Structure Problem]] da ko'rasan.

## 4. Eslab qolish uchun formula

```text
Express'da savol:  "Bu funksiya ro'yxatning qayerida turadi?"
NestJS'da savol:   "Bu narsa lifecycle'ning qaysi bosqichi?"

Ikkalasi ham AYNI savol.
```

## 5. Practice

1. Bo'sh Express app yozib, 3 ta `app.use()` qo'y, har biriga `console.log` — tartibni ko'r.
2. O'rtadagisida `next()` ni o'chir — nima bo'ladi?
3. Ro'yxatdagi ikkita middleware o'rnini almashtir va natijani oldindan ayt, keyin tekshir.
4. Yuqoridagi jadvalni yopib, har bir Express tushunchasining NestJS'dagi ekvivalentini yozib chiq.

## Related

[[Express Middleware]] · [[Express Middleware Order]] · [[Express Project Structure Problem]] · [[NestJS Mental Model]] · [[NestJS to Express Translation]]
