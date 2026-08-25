---
tags: [express/middleware, level/beginner, fundamental]
level: beginner
topic: Middleware
---

# Express Middleware

> **Bir jumlada:** middleware — `(req, res, next)` qabul qiladigan oddiy funksiya; NestJS'dagi Guard, Pipe, Interceptor va Filter — hammasi shu bitta g'oyaning ixtisoslashgan ko'rinishi.

**Before learning:** [[Express App Bootstrap]], [[Express Request and Response]]
**After learning:** [[Express next Function]], [[Express Middleware Order]], [[Express Error Handling Middleware]]

**NestJS'dagi mos note:** [[NestJS Request Lifecycle]] · [[NestJS Middleware]]

Bu papkadagi **eng fundamental note**. NestJS lifecycle'i tushunarsiz bo'lsa — shu yerdan boshla.

---

## 1. Muammo

NestJS'da beshta turli tushuncha bor:

```text
Middleware · Guard · Pipe · Interceptor · Filter
```

Va ular bir-biridan nima bilan farq qilishini eslab qolish qiyin.

**Express'da javob juda oddiy: ularning hammasi bitta narsa — middleware.**
NestJS ularni ataylab **ixtisoslashtirgan**, chunki katta loyihada "hammasi bir xil" — bu tartibsizlik demak.

## 2. Middleware nima?

```js
function myMiddleware(req, res, next) {
  // 1. request bilan biror ish qil
  // 2. keyin uchtadan birini tanla:
  next();                            // a) zanjirni davom ettir
  // res.status(403).json({...});    // b) javob yozib, TO'XTAT
  // next(new Error('boom'));        // c) xatoga o't
}

app.use(myMiddleware);
```

Uchta tanlov — bu **butun Express**. Boshqa variant yo'q.

## 3. Beshta NestJS tushunchasi — Express'da

### Guard → shunchaki `if` + `res.status(403)`

```js
function requireAuth(req, res, next) {
  const token = req.headers.authorization?.split(' ')[1];
  if (!token) return res.status(401).json({ message: 'Unauthorized' });

  try {
    req.user = jwt.verify(token, process.env.JWT_SECRET);
    next();                                    // ← "true" qaytarish
  } catch {
    return res.status(401).json({ message: 'Invalid token' });   // ← "false"
  }
}
```

NestJS'da bu `canActivate()` bo'lib, `true`/`false` qaytaradi. Mantiq **aynan bir xil**.

### Pipe → validatsiya qiluvchi middleware

```js
function validateCreatePlayer(req, res, next) {
  const { firstName, birthDate } = req.body;
  const errors = [];

  if (typeof firstName !== 'string' || firstName.length < 2) {
    errors.push('firstName must be a string of at least 2 chars');
  }
  if (isNaN(Date.parse(birthDate))) errors.push('birthDate must be a date');

  if (errors.length) return res.status(400).json({ message: errors });

  next();
}
```

NestJS'da bu `ValidationPipe` + DTO decoratorlari. Qilayotgan ishi bir xil — faqat deklarativ.

### Interceptor → response'ni o'rash

```js
function wrapResponse(req, res, next) {
  const originalJson = res.json.bind(res);
  res.json = (body) => originalJson({ data: body });   // ← "after" qismi
  next();
}
```

Yoki vaqt o'lchash:

```js
function timing(req, res, next) {
  const start = Date.now();                            // "before"
  res.on('finish', () => {                             // "after"
    console.log(`${req.method} ${req.originalUrl} ${Date.now() - start}ms`);
  });
  next();
}
```

NestJS'da bu RxJS `pipe(map(...))` bilan chiroyliroq yoziladi, lekin **g'oya bir xil**.

### Filter → 4 argumentli middleware

```js
app.use((err, req, res, next) => {          // ← 4 ta argument = error handler
  res.status(err.status || 500).json({ message: err.message });
});
```

Batafsil: [[Express Error Handling Middleware]]

### Middleware → o'zi

```js
app.use(express.json());
app.use(helmet());
```

## 4. Butun zanjir — bitta misolda

```js
const app = express();

app.use(express.json());                       // 1. body parse

app.use((req, res, next) => {                  // 2. logging (Interceptor'ga o'xshash)
  const start = Date.now();
  res.on('finish', () => console.log(`${res.statusCode} ${Date.now() - start}ms`));
  next();
});

app.post('/players',
  rateLimit,                                   // 3. Throttle Guard
  requireAuth,                                 // 4. JwtAuthGuard
  requireRole('coach'),                        // 5. RolesGuard
  validateCreatePlayer,                        // 6. ValidationPipe
  async (req, res, next) => {                  // 7. Controller
    try {
      const player = await playersService.create(req.body, req.user.id);
      res.status(201).json(player);
    } catch (err) { next(err); }
  },
);

app.use((err, req, res, next) => {             // 8. ExceptionFilter
  res.status(err.status || 500).json({ message: err.message });
});
```

Endi buni FotSpot'ning guard zanjiri bilan solishtir:

```text
Express:  rateLimit → requireAuth → requireRole → validate → handler → errorHandler
NestJS:   Throttle  → JwtAuth     → Roles       → Validation → Controller → Filter
```

**Aynan bir xil zanjir.** Farq shundaki:

```text
Express'da zanjirni HAR BIR ROUTE'da qo'lda yozasan (yoki unutasan)
NestJS'da zanjir GLOBAL va unutib bo'lmaydi
```

Mana shu — [[FotSpot Guard Chain]] ning butun ma'nosi.

## 5. Nima uchun NestJS ularni ajratdi?

```text
Express'da hammasi (req,res,next) bo'lgani uchun:
   ✗ funksiya nomiga qarab taxmin qilasan
   ✗ tartibni har joyda qo'lda saqlaysan
   ✗ bittasini unutsang — jimgina teshik

NestJS ajratgani uchun:
   ✓ Guard → faqat true/false (business logic yozib bo'lmaydi)
   ✓ Pipe  → faqat input (javob yoza olmaydi)
   ✓ Filter → faqat xato
   ✓ Tartib framework'da qat'iy
```

Ya'ni NestJS **imkoniyatni kamaytirib**, xato qilish ehtimolini kamaytirgan.

## 6. Common mistakes

- **`next()` ni chaqirmaslik** → request osilib qoladi ([[Express next Function]]).
- **`next()` dan keyin ham kod yozish** → ikkala shox ham ishlaydi. `return next();` yoz.
- **Javob yozib, `next()` ni ham chaqirish** → `ERR_HTTP_HEADERS_SENT`.
- **Middleware'ni route'dan keyin qo'yish** → ishlamaydi ([[Express Middleware Order]]).
- **`async` middleware ichidagi xatoni ushlamaslik** → [[Express Async Errors]].

## 7. Practice problems

### Problem 1 — uchta tanlovni ko'r
Bitta middleware yozib, uni navbat bilan uch xil qil: `next()`, `res.json()`, `next(new Error())`.
Har birida `curl` natijasini yozib ol.

### Problem 2 — Guard'ni qo'lda yoz
`requireAuth` middleware yoz. Tokensiz 401, noto'g'ri token bilan 401, to'g'ri token bilan `req.user` to'ldirilsin.

### Problem 3 — Pipe'ni qo'lda yoz
`validateCreatePlayer` yoz. `{"firstName": 1}` yuborganda 400 va tushunarli xato matni qaytsin.

### Problem 4 — Interceptor'ni qo'lda yoz
Barcha javoblar `{ data: ... }` ichida qaytadigan middleware yoz (`res.json` ni o'rab).
**Savol:** error handler'dagi javob ham o'raldimi? Nega?

### Problem 5 — to'liq zanjir
Yuqoridagi 8 bosqichli misolni to'liq yozib ishga tushir. Har bosqichga log qo'y.
`curl` bilan chaqirib, log tartibini yozib ol.

### Problem 6 — bittasini unut
`requireAuth` ni bitta route'dan olib tashla (masalan `DELETE /players/:id` dan).
**Savol:** buni qanday sezasan? Test yozmasang, qachon bilib qolasan?
**Xulosa:** NestJS'da global guard nega afzal ([[Public Routes Decorator]]).

### Problem 7 — tarjima qil
Yuqoridagi Express zanjirini NestJS'da qayta yoz: guard, pipe, filter sifatida.
Ikkalasini yonma-yon qo'yib, qaysi biri **xatoga chidamli**roq ekanini ayt.

## 8. Related

[[Express next Function]] · [[Express Middleware Order]] · [[Express Error Handling Middleware]] · [[Express Auth Middleware]] · [[Express Validation Middleware]] · [[NestJS Request Lifecycle]] · [[Middleware vs Guard vs Pipe vs Interceptor vs Filter]] · [[FotSpot Guard Chain]]
