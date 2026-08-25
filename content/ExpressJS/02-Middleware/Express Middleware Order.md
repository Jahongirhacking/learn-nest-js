---
tags: [express/middleware, level/beginner]
level: beginner
topic: Middleware
---

# Express Middleware Order

> **Bir jumlada:** Express'da tartib = mantiq; NestJS'da tartib framework tomonidan qat'iy belgilangan — mana shu ikkalasining eng katta farqi.

**Before learning:** [[Express Middleware]], [[Express next Function]]
**After learning:** [[Express Error Handling Middleware]], [[Express Project Structure Problem]]

**NestJS'dagi mos note:** [[Middleware vs Guard vs Pipe vs Interceptor vs Filter]] · [[FotSpot Guard Chain]]

---

## 1. Muammo

NestJS'da guard zanjiri qat'iy:

```text
Throttle → JwtAuth → Roles → Permissions → ValidationPipe → Controller → Filter
```

Va savol: **kim bu tartibni belgilagan?** Express'da javob: **sen**. Har bir qatorda.

## 2. Asosiy qoida

```text
Express middleware ro'yxatni FAYLDA YOZILGAN tartibda ishlatadi.
Yuqoridagi middleware har doim pastdagisidan oldin ishlaydi.
```

```js
app.use(a);        // 1
app.use(b);        // 2
app.get('/x', c);  // 3
app.use(d);        // 4 ← faqat c() next() chaqirsa ishlaydi
```

## 3. To'g'ri tartib — production shabloni

```js
const app = express();

// 1. Xavfsizlik headerlari — eng birinchi
app.use(helmet());

// 2. CORS — preflight route'lardan oldin javob berishi kerak
app.use(cors({ origin: allowed }));

// 3. Rate limit — QIMMAT ishdan OLDIN
app.use(rateLimit({ windowMs: 60_000, max: 100 }));

// 4. Body parsing
app.use(express.json({ limit: '1mb' }));

// 5. Logging
app.use(requestLogger);

// 6. Route'lar
app.use('/api/v1/auth', authRoutes);
app.use('/api/v1/players', playersRoutes);

// 7. 404 — barcha route'lardan KEYIN
app.use((req, res) => res.status(404).json({ message: 'Not Found' }));

// 8. Error handler — ENG OXIRIDA
app.use(errorHandler);
```

Har bir raqamning **sababi** bor:

```text
1-2  brauzer siyosati — javobga header qo'shadi, tez
3    rate limit body parse va DB'dan OLDIN
     → hujumchi qimmat ishni majburlay olmasin
4    body parse route'dan oldin (aks holda req.body undefined)
5    logging body parse'dan keyin (body'ni loglash uchun — ehtiyot bo'l!)
6    route'lar
7    404 — route topilmaganda
8    error — hamma narsadan keyin (4 argumentli)
```

**Bu FotSpot guard zanjirining aynan o'zi** — faqat qo'lda yozilgan ([[FotSpot Guard Chain]]).

## 4. Uchta klassik xato

### Xato 1 — error handler yuqorida

```js
app.use(errorHandler);              // ✗
app.get('/x', (req, res, next) => next(new Error('boom')));
```

**Natija:** error handler hech qachon chaqirilmaydi; Express'ning default HTML xatosi chiqadi.
Sabab: `next(err)` **oldinga** qaraydi, orqaga emas.

### Xato 2 — 404 yuqorida

```js
app.use((req, res) => res.status(404).json({ message: 'Not Found' }));   // ✗
app.use('/api/v1/players', playersRoutes);
```

**Natija:** hamma so'rov 404. Route'lar hech qachon ishlamaydi.

### Xato 3 — rate limit pastda

```js
app.use(express.json());
app.use(requireAuth);            // qimmat: jwt.verify + DB
app.use(rateLimit(...));         // ✗ kech
```

**Natija:** hujumchi minutiga 10 000 marta `jwt.verify` va DB query'ni majburlay oladi.

## 5. Global va lokal middleware

```js
// GLOBAL — hamma route uchun
app.use(requireAuth);

// PREFIX bo'yicha
app.use('/api/v1/admin', requireAuth, requireRole('admin'));

// FAQAT bitta route uchun
app.get('/players/:id', requireAuth, getPlayer);
```

Bu yerda [[Public Routes Decorator]] mavzusining Express varianti ochiladi:

```js
// Variant A: default OCHIQ, kerakli joyga qo'yamiz
app.get('/players', listPlayers);                    // ochiq
app.post('/players', requireAuth, createPlayer);     // yopiq
app.delete('/players/:id', deletePlayer);            // ✗ UNUTILDI → teshik!

// Variant B: default YOPIQ, ochiq joyni ajratamiz
app.use('/api/v1/auth', authRoutes);                 // ochiq (auth'dan oldin)
app.use(requireAuth);                                // ← shundan keyin hammasi yopiq
app.use('/api/v1/players', playersRoutes);
```

```text
Variant A → unutsang, JIMGINA teshik qoladi
Variant B → unutsang, SHOVQINLI 401 chiqadi
```

FotSpot **Variant B** ni tanlagan: global guard + `@Public()` istisno.
Endi `@Public()` naqshining nima uchun xavfsizroq ekani aniq bo'ldi.

## 6. Interceptor tartibi — "before" va "after"

```js
app.use((req, res, next) => {
  console.log('A before');
  next();
  console.log('A after');
});

app.use((req, res, next) => {
  console.log('B before');
  next();
  console.log('B after');
});

app.get('/x', (req, res) => { console.log('handler'); res.json({}); });
```

**Natija:**

```text
A before
B before
handler
B after
A after
```

Bu — **matryoshka** modeli. NestJS interceptor'lari aynan shunday ishlaydi:
tashqi interceptor ichkisini o'raydi ([[NestJS Interceptors]]).

> Diqqat: `res.json()` **async** bo'lgani uchun "after" loglari haqiqiy javob yuborilishidan
> oldin chiqishi mumkin. Aniq "javob ketdi" nuqtasi uchun `res.on('finish')` ishlat.

## 7. Common mistakes

- **Error handler'ni oxiriga qo'ymaslik.**
- **404 handler'ni route'lardan oldin qo'yish.**
- **`express.json()` ni route'lardan keyin qo'yish** → `req.body` `undefined`.
- **Rate limit'ni auth'dan keyin qo'yish.**
- **Body'ni loglash** → parol va token logga tushadi ([[Express Logging]]).
- **Global `app.use(requireAuth)` ni login route'idan oldin qo'yish** → hech kim login qila olmaydi.

## 8. Practice problems

### Problem 1 — tartibni buz va tuzat
Yuqoridagi 8 bosqichli shablonni yoz. Keyin navbat bilan:
error handler'ni yuqoriga, 404 ni yuqoriga, `express.json()` ni pastga ko'chir.
Har birida nima buzilganini **yozib ol**.

### Problem 2 — matryoshka
6-bo'limdagi kodni yozib, log tartibini oldindan ayt, keyin tekshir.

### Problem 3 — Variant A tuzog'i
5 ta route yozib, ulardan bittasiga `requireAuth` qo'yishni ataylab unut.
**Savol:** buni qanday topasan? Qancha vaqt ketdi?

### Problem 4 — Variant B
Xuddi shu 5 ta route'ni global `app.use(requireAuth)` bilan yoz va login route'ini undan **oldin** qo'y.
**Savol:** endi biror route'ni unutish mumkinmi?

### Problem 5 — rate limit joyi
`rateLimit` ni auth'dan oldin va keyin qo'yib, har ikkala holatda `jwt.verify` necha marta chaqirilishini logla.

### Problem 6 — NestJS'ga tarjima
Yuqoridagi 8 bosqichni NestJS'da qayerga yozilishini ko'rsat:
qaysi biri `main.ts` da, qaysi biri `APP_GUARD` da, qaysi biri decoratorda?

## 9. Related

[[Express Middleware]] · [[Express next Function]] · [[Express Error Handling Middleware]] · [[Express Auth Middleware]] · [[FotSpot Guard Chain]] · [[Middleware vs Guard vs Pipe vs Interceptor vs Filter]] · [[Public Routes Decorator]] · [[NestJS Interceptors]]
