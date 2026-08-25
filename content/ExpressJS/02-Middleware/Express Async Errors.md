---
tags: [express/middleware, level/intermediate, gotcha]
level: intermediate
topic: Middleware
---

# Express Async Errors

> **Bir jumlada:** Express 4 `async` funksiyadagi `throw` ni **ushlamaydi** — har bir handler'da `try/catch` yozishga majbursan; NestJS bu muammoni butunlay yo'q qilgan.

**Before learning:** [[Express Error Handling Middleware]]
**After learning:** [[Express Service Layer]], [[NestJS Exception Filters]]

**NestJS'dagi mos note:** [[NestJS Exception Filters]] · [[NestJS Services]]

---

## 1. Muammoni ko'rish

```js
// Sinxron — ISHLAYDI
app.get('/sync', (req, res) => {
  throw new Error('boom');            // ✓ error handler ushlaydi
});

// Async — ISHLAMAYDI (Express 4)
app.get('/async', async (req, res) => {
  throw new Error('boom');            // ✗ error handler ko'rmaydi
});
```

`/async` ni chaqirsang:

```text
✗ Error handler chaqirilmaydi
✗ Javob kelmaydi — request OSILIB QOLADI
⚠ Terminal'da: UnhandledPromiseRejection ogohlantirishi
```

**Nega?** Express handler'ni shunday chaqiradi:

```js
try {
  handler(req, res, next);      // async funksiya darhol Promise qaytaradi
} catch (err) {
  next(err);                    // sinxron xatoni ushlaydi
}
```

`async` funksiya **darhol qaytadi**, xato esa keyinroq — Promise ichida. `try/catch` uni ko'rmaydi.

## 2. Uchta yechim

### Yechim A — har bir handler'da `try/catch` (klassik)

```js
app.get('/players/:id', async (req, res, next) => {
  try {
    const player = await playersService.findOne(req.params.id);
    res.json(player);
  } catch (err) {
    next(err);
  }
});
```

**Muammo:** 50 ta endpoint = 50 ta bir xil `try/catch`. Bittasini unutsang — jimgina osilib qolish.

### Yechim B — wrapper funksiya

```js
// utils/async-handler.js
const asyncHandler = (fn) => (req, res, next) =>
  Promise.resolve(fn(req, res, next)).catch(next);

module.exports = asyncHandler;
```

```js
app.get('/players/:id', asyncHandler(async (req, res) => {
  res.json(await playersService.findOne(req.params.id));
}));
```

Ancha yaxshi — lekin har bir route'ni `asyncHandler()` bilan **o'rashni unutmaslik** kerak.

### Yechim C — Express 5

Express 5'da async xatolar avtomatik ushlanadi. Lekin ko'p loyihalar hali 4'da.

## 3. Mana shu — NestJS'ning eng ko'rinmas foydasi

```ts
// NestJS
@Get(':id')
async findOne(@Param('id') id: string) {
  const player = await this.prisma.player.findUnique({ where: { id } });
  if (!player) throw new NotFoundException('Player not found');
  return player;
}
```

```text
try/catch      → YO'Q
asyncHandler   → YO'Q
next(err)      → YO'Q
```

Nest har bir handler'ni o'zi o'raydi va Promise rejection'ni Exception Filter'ga yo'naltiradi.

**Bu kichik qulaylik emas** — bu butun bir xatolar sinfining yo'q qilinishi:

```text
Express:  50 endpoint × unutish ehtimoli = doimiy xavf
NestJS:   0 ta joy unutish uchun
```

Yodda tut: bu foyda **service qatlamiga ham** taalluqli — [[Express Service Layer]] dagi service'lar ham `throw` qila oladi va Nest ularni ushlaydi.

## 4. Common mistakes

- **`try/catch` ni unutish** → request jimgina osilib qoladi.
- **`catch (err) { res.status(500).json(...) }`** → error handler chetlab o'tiladi, format buziladi. `next(err)` yoz.
- **`catch` ichida `console.log` qilib, `next` chaqirmaslik** → xato yutiladi.
- **`asyncHandler` bilan o'rashni ba'zi route'da unutish** → aynan o'sha route'da tuzoq.
- **`Promise` qaytaruvchi, lekin `async` bo'lmagan funksiyani unutish** → `.catch(next)` baribir kerak.
- **`process.on('unhandledRejection')` ni qo'ymaslik** → muammoni umuman ko'rmaysan.

## 5. Practice problems

### Problem 1 — muammoni ko'r
```js
app.get('/async', async (req, res) => { throw new Error('boom'); });
```
`curl -m 5 localhost:3000/async` qil. Nima bo'ldi? Terminal'da nima yozildi?

### Problem 2 — `try/catch` bilan tuzat
Xuddi shu handler'ni `try/catch` + `next(err)` bilan tuzat va error handler ushlaganini tasdiqla.

### Problem 3 — `asyncHandler` yoz
`asyncHandler` wrapper'ini yozib, 3 ta route'ni o'ra. Kod qancha qisqardi?

### Problem 4 — unutishni modellashtir
5 ta route yozib, bittasini `asyncHandler`siz qoldir. Keyin hammasini sinab, qaysi biri buzilganini top.
**Savol:** test yozmasang, buni production'da qachon bilib qolasan?

### Problem 5 — global signal
```js
process.on('unhandledRejection', (reason) => {
  console.error('UNHANDLED REJECTION:', reason);
});
```
Qo'shib, 1-masalani qayta ishga tushir. Endi ko'rinyaptimi?

### Problem 6 — NestJS'da takrorlashga urin
NestJS controller'da `async` handler ichida `throw new Error('boom')` yoz.
**Savol:** request osilib qoldimi? Qanday javob keldi? Nega?

### Problem 7 — service qatlamida
`playersService.findOne()` ichida `throw` qil va controller'da hech qanday `try/catch` yozma.
Express'da nima bo'ldi? NestJS'da nima bo'ldi?

## 6. Related

[[Express Error Handling Middleware]] · [[Express next Function]] · [[Express Service Layer]] · [[Express Route Handlers]] · [[NestJS Exception Filters]] · [[NestJS Services]]
