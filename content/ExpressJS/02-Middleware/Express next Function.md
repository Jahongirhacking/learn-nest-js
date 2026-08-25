---
tags: [express/middleware, level/beginner]
level: beginner
topic: Middleware
---

# Express next Function

> **Bir jumlada:** `next()` — "zanjirdagi keyingi funksiyani chaqir" degani; NestJS lifecycle'i uni **sen uchun** chaqiradi, shuning uchun NestJS'da bu tushuncha ko'rinmaydi.

**Before learning:** [[Express Middleware]]
**After learning:** [[Express Middleware Order]], [[Express Error Handling Middleware]]

**NestJS'dagi mos note:** [[NestJS Request Lifecycle]]

---

## 1. `next` aslida nima?

`next` — Express beradigan **funksiya**. Uni chaqirsang, Express ro'yxatdagi keyingi mos middleware'ni ishga tushiradi.

```js
app.use((req, res, next) => {
  console.log('1-chi');
  next();               // ← "meniki tugadi, keyingisiga o't"
  console.log('1-chi tugadi');    // ← bu next() DAN KEYIN ishlaydi!
});

app.use((req, res, next) => {
  console.log('2-chi');
  next();
});

app.get('/x', (req, res) => {
  console.log('handler');
  res.json({ ok: true });
});
```

**Log tartibi:**

```text
1-chi
2-chi
handler
1-chi tugadi        ← e'tibor ber!
```

Ya'ni `next()` — bu **oddiy funksiya chaqiruvi**, `return` emas. U tugagach, kod davom etadi. Bu Interceptor'ning "after" qismi qanday ishlashini tushuntiradi.

## 2. Uchta ishlatish shakli

```js
next();              // keyingi middleware
next(err);           // XATO zanjiriga sakra → error handler
next('route');       // shu router'dagi qolgan handler'larni o'tkazib yubor
```

Ikkinchisi eng muhim:

```text
next(nimadir)  →  agar argument bor bo'lsa, Express uni XATO deb biladi
               →  oddiy middleware'lar o'tkazib yuboriladi
               →  to'g'ridan-to'g'ri 4-argumentli error handler'ga boradi
```

**Tuzoq:** `next(null)` va `next(undefined)` — xato emas, oddiy davom etish. Lekin `next(false)`, `next('')` — Express versiyasiga qarab kutilmagan xatti-harakat berishi mumkin. Har doim `next()` yoki `next(err)` yoz.

## 3. `next()` chaqirilmasa nima bo'ladi?

**Hech narsa.** Va bu eng chalkash qismi:

```js
app.use((req, res, next) => {
  console.log('kirdim');
  // next() YO'Q va res ham YO'Q
});

app.get('/x', (req, res) => res.json({ ok: true }));   // ← hech qachon ishlamaydi
```

```bash
curl localhost:3000/x
# ... kutadi ... kutadi ... timeout
```

```text
✗ Xato chiqmaydi
✗ Log chiqmaydi
✗ Crash bo'lmaydi
✓ Shunchaki JAVOB KELMAYDI
```

Bu Express'dagi eng ko'p vaqt yo'qotadigan bug. **NestJS'da bu mumkin emas** — chunki `next()` ni sen chaqirmaysan.

## 4. Ikki marta chaqirish

```js
app.use((req, res, next) => {
  next();
  next();          // ✗ zanjir ikki marta ishlaydi
});
```

Natija: keyingi middleware ikki marta chaqiriladi → ko'pincha `ERR_HTTP_HEADERS_SENT`.

Xuddi shu xato ko'pincha shunday ko'rinadi:

```js
// ✗ XATO — return yo'q
app.use((req, res, next) => {
  if (!req.headers.authorization) next(new Error('no token'));
  next();                                    // ← baribir ishlaydi!
});

// ✓ TO'G'RI
app.use((req, res, next) => {
  if (!req.headers.authorization) return next(new Error('no token'));
  next();
});
```

**Qoida: `next()` doim `return` bilan.**

## 5. NestJS bu tushunchani qayerga yashirgan

| Express | NestJS |
|---|---|
| `next()` | Guard `return true` |
| `res.status(403).json(...)` (next'siz) | Guard `return false` yoki `throw` |
| `next(err)` | `throw new SomeException()` |
| `next()` dan keyingi kod | Interceptor'ning `pipe(tap(...))` qismi |
| `next()` ni unutish | **mumkin emas** |

Ya'ni NestJS `next()` ni **abstraksiya qilib olib tashlagan** va shu bilan butun bir xatolar sinfini yo'q qilgan.

## 6. Common mistakes

- **`next()` ni unutish** → request osilib qoladi, hech qanday signal yo'q.
- **`return next()` o'rniga `next()`** → kod davom etadi, ikki marta ishlaydi.
- **Javob yozib, keyin `next()` chaqirish** → `ERR_HTTP_HEADERS_SENT`.
- **`next(err)` o'rniga `throw err` (async funksiyada)** → Express 4 uni ushlamaydi ([[Express Async Errors]]).
- **`next()` ni `return` deb tushunish** → undan keyingi kod baribir ishlaydi.

## 7. Practice problems

### Problem 1 — tartibni ko'r
Yuqoridagi 1-bo'limdagi kodni yozib ishga tushir. Log tartibini **oldindan yozib qo'y**, keyin tekshir.

### Problem 2 — osilib qolish
Middleware'dan `next()` ni olib tashla va `curl -m 5 localhost:3000/x` qil.
**Savol:** xato qayerdan keldi — serverdanmi yoki curl'danmi?

### Problem 3 — ikki marta
`next()` ni ikki marta chaqir. Xato matnini to'liq o'qi va nima uchun chiqqanini tushuntir.

### Problem 4 — `return` yo'qligi
```js
app.use((req, res, next) => {
  if (!req.query.token) next(new Error('no token'));
  console.log('davom etdim');
  next();
});
```
`?token` siz chaqir. `'davom etdim'` chiqdimi? Tuzat.

### Problem 5 — `next(err)`
Middleware'da `next(new Error('boom'))` chaqir va error handler'da ushla.
**Savol:** oraliqdagi oddiy middleware'lar ishladimi?

### Problem 6 — before/after
`next()` dan oldin va keyin log yozadigan middleware yoz.
**Savol:** bu NestJS'ning qaysi tushunchasiga mos keladi? ([[NestJS Interceptors]])

### Problem 7 — NestJS'da izla
NestJS Guard yozib, `return true` va `return false` holatlarini sinab ko'r.
`next()` ga mos keladigan joyni topib ko'rsat.

## 8. Related

[[Express Middleware]] · [[Express Middleware Order]] · [[Express Error Handling Middleware]] · [[Express Async Errors]] · [[NestJS Request Lifecycle]] · [[NestJS Interceptors]]
