---
tags: [express, moc, map]
type: moc
---

# Express Learning Map

> **Bu papkaning maqsadi:** NestJS'ni o'rganishda tushunarsiz bo'lgan mavzuni **Express'da qo'lda yozib** tushunish.
> Express — "fundament", NestJS — "shu fundament ustidagi avtomatlashtirish".

**Kalit note:** [[NestJS to Express Translation]] — qaysi NestJS mavzusi qaysi Express note'ga mos kelishini ko'rsatadi.

---

## 1. Fundament — HTTP va Express nima?

- [[Node HTTP Server]] — Express'siz, sof Node bilan server
- [[Express Overview]] — Express nima qo'shadi va nimani qo'shmaydi
- [[Express App Bootstrap]] — `express()` dan `app.listen()` gacha
- [[Express Request and Response]] — `req` va `res` obyektlari aslida nima
- [[Express Routing]] — route qanday ro'yxatga olinadi va topiladi
- [[Express Router]] — modularlikning Express'dagi varianti

## 2. Middleware — Express'ning yuragi

> Agar faqat bitta bo'limni o'qiydigan bo'lsang — shu bo'limni o'qi.
> NestJS'dagi Guard, Pipe, Interceptor, Filter — hammasi shu g'oyaning ixtisoslashgan ko'rinishi.

- [[Express Middleware]] — **eng fundamental note**
- [[Express next Function]] — `next()` aslida nima qiladi
- [[Express Middleware Order]] — tartib nega hamma narsani hal qiladi
- [[Express Error Handling Middleware]] — 4 argumentli sirli funksiya
- [[Express Async Errors]] — Express 4'ning eng ko'p tishlaydigan joyi

## 3. Struktura — kod qanday tarqab ketadi

- [[Express Route Handlers]] — controller'ning Express varianti
- [[Express Service Layer]] — business logikani qo'lda ajratish
- [[Express Manual Dependency Wiring]] — **DI'ni tushunish uchun eng muhim note**
- [[Express Project Structure Problem]] — NestJS nega paydo bo'lgan

## 4. Validatsiya

- [[Express Request Validation]] — DTO'siz dunyoda input tekshirish
- [[Express Validation Middleware]] — ValidationPipe'ni qo'lda qurish

## 5. Authentication va Authorization

- [[Express Authentication]] — JWT'ni qo'lda chiqarish va tekshirish
- [[Express Auth Middleware]] — Guard'ning haqiqiy ko'rinishi
- [[Express Authorization Middleware]] — role va permission tekshiruvi
- [[Express Passport]] — Passport Express'da qanday ishlaydi

## 6. Ma'lumot va infratuzilma

- [[Express Prisma Setup]] — DI'siz Prisma
- [[Express Redis and Queues]] — Redis va BullMQ qo'lda ulanganda

## 7. Production

- [[Express Config and Env]] — `dotenv` va `process.env`
- [[Express Security Middleware]] — helmet, cors
- [[Express Rate Limiting]] — `express-rate-limit`
- [[Express Logging]] — morgan va qo'lda logging
- [[Express Testing]] — `supertest`

## 8. Amaliyot

- [[Express to NestJS Rebuild]] — bir xil API'ni Express'da yozib, keyin NestJS'ga o'girish
- [[Express Practice Problems]] — 25 ta amaliy masala

---

## O'rganish tartibi

```text
[[Node HTTP Server]]
   ↓
[[Express Overview]] → [[Express App Bootstrap]] → [[Express Routing]]
   ↓
[[Express Middleware]] → [[Express next Function]] → [[Express Middleware Order]]
   ↓
[[Express Error Handling Middleware]] → [[Express Async Errors]]
   ↓
[[Express Route Handlers]] → [[Express Service Layer]] → [[Express Manual Dependency Wiring]]
   ↓
[[Express Request Validation]] → [[Express Authentication]] → [[Express Auth Middleware]]
   ↓
[[Express Project Structure Problem]]   ← bu yerda NestJS'ning ma'nosi ochiladi
   ↓
[[Express to NestJS Rebuild]]
```

## Related

[[NestJS to Express Translation]] · [[Express Mental Model]] · [[NestJS Learning Map]]
