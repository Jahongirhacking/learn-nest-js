---
tags: [express/validation, level/intermediate]
level: intermediate
topic: Validation
---

# Express Validation Middleware

> **Bir jumlada:** validatsiyani middleware'ga chiqarish — bu `ValidationPipe`ni qo'lda qurish; farqi shundaki, Express'da uni har bir route'ga **qo'lda ulash** kerak.

**Before learning:** [[Express Request Validation]], [[Express Middleware]]
**After learning:** [[NestJS ValidationPipe]], [[Express to NestJS Rebuild]]

**NestJS'dagi mos note:** [[NestJS Pipes]] · [[NestJS ValidationPipe]]

---

## 1. Maqsad

Har bir route'da takrorlanadigan bu kodni:

```js
const parsed = schema.safeParse(req.body);
if (!parsed.success) return res.status(400).json({ message: parsed.error.issues });
req.body = parsed.data;
```

**bitta qayta ishlatiladigan middleware**ga aylantirish.

## 2. `validate()` middleware

```js
// middleware/validate.js
module.exports = function validate(schema, source = 'body') {
  return (req, res, next) => {
    const result = schema.safeParse(req[source]);

    if (!result.success) {
      return res.status(400).json({
        statusCode: 400,
        message: result.error.issues.map(i => ({
          field: i.path.join('.'),
          message: i.message,
        })),
      });
    }

    req[source] = result.data;      // ← tozalangan + transform qilingan
    next();
  };
};
```

Ishlatilishi:

```js
const validate = require('../middleware/validate');
const { createPlayerSchema, listPlayersSchema } = require('../validation/player.schema');

router.post('/', validate(createPlayerSchema), asyncHandler(async (req, res) => {
  res.status(201).json(await playersService.create(req.body, req.user.id));
}));

router.get('/', validate(listPlayersSchema, 'query'), asyncHandler(async (req, res) => {
  res.json(await playersService.findAll(req.query));
}));
```

Bu — **NestJS `ValidationPipe`ning aynan o'zi**, faqat qo'lda ulanadi.

## 3. Uchta option qo'lda

NestJS:

```ts
new ValidationPipe({ whitelist: true, forbidNonWhitelisted: true, transform: true })
```

zod'da ularning ekvivalenti:

```js
// whitelist: true  →  ortiqcha maydonlar KESILADI (zod default xatti-harakati)
z.object({ firstName: z.string() })
// { firstName: 'Ali', role: 'admin' }  →  { firstName: 'Ali' }

// forbidNonWhitelisted: true  →  ortiqcha maydon bo'lsa RAD ETILADI
z.object({ firstName: z.string() }).strict()
// { firstName: 'Ali', role: 'admin' }  →  ✗ "Unrecognized key: role"

// transform: true  →  tipga o'tkazish
z.object({ page: z.coerce.number().int().min(1).default(1) })
// { page: '2' }  →  { page: 2 }
```

Uch qatorli jadval:

| NestJS option | zod ekvivalenti | Nima qiladi |
|---|---|---|
| `whitelist` | `z.object({...})` (default) | ortiqchasini kesadi |
| `forbidNonWhitelisted` | `.strict()` | ortiqchasi bo'lsa 400 |
| `transform` | `z.coerce.*`, `.default()` | tip va default |

## 4. Global qilish — va uning chegarasi

Express'da validatsiyani "global" qilib bo'lmaydi, chunki **har bir route'ning schema'si boshqacha**.

Maksimal yaqinlashish — schema'ni route ta'rifiga bog'lash:

```js
// routes/players.routes.js
const routes = [
  { method: 'get',  path: '/',    schema: { query: listPlayersSchema }, handler: list },
  { method: 'post', path: '/',    schema: { body: createPlayerSchema }, auth: true, handler: create },
  { method: 'get',  path: '/:id', schema: { params: idSchema },         handler: getOne },
];

routes.forEach(r => {
  const mws = [];
  if (r.auth) mws.push(requireAuth);
  if (r.schema?.body)   mws.push(validate(r.schema.body, 'body'));
  if (r.schema?.query)  mws.push(validate(r.schema.query, 'query'));
  if (r.schema?.params) mws.push(validate(r.schema.params, 'params'));
  router[r.method](r.path, ...mws, asyncHandler(r.handler));
});
```

**Diqqat qil — sen hozir nima yozding?**

```text
Route metadata ro'yxati
   ↓
undan middleware zanjiri quriladi
   ↓
route ro'yxatga olinadi
```

**Bu — NestJS decorator tizimining qo'lda yozilgani.**
`@Post()`, `@UseGuards()`, `@Body() dto: CreatePlayerDto` — aynan shu metadata, faqat decorator sintaksisi bilan.

## 5. Xato formatini bir xillashtirish

Validatsiya xatosi ham [[Express Error Handling Middleware]] orqali o'tishi kerak:

```js
// yaxshiroq: res.status(400) o'rniga xato tashlash
module.exports = function validate(schema, source = 'body') {
  return (req, res, next) => {
    const result = schema.safeParse(req[source]);
    if (!result.success) {
      const err = new HttpError(400, result.error.issues.map(i => `${i.path.join('.')}: ${i.message}`));
      return next(err);                 // ← markaziy error handler'ga
    }
    req[source] = result.data;
    next();
  };
};
```

Endi barcha xatolar — validatsiya ham, business ham — **bitta formatda** qaytadi.
NestJS'da bu avtomatik: `ValidationPipe` `BadRequestException` tashlaydi, filter uni formatlaydi.

## 6. Common mistakes

- **`req.body = result.data` ni unutish** → tozalangan ma'lumot ishlatilmaydi, whitelist ta'sir qilmaydi.
- **`validate()` ni ba'zi route'ga ulashni unutish** → jimgina teshik.
- **`params` va `query` ni tekshirmaslik** → faqat `body` tekshiriladi, `:id` esa har qanday narsa bo'lishi mumkin.
- **Xatoni `res.status(400)` bilan qaytarib, error handler'ni chetlab o'tish** → format nomuvofiqligi.
- **`.strict()` ni unutish** → mass assignment ochiq qoladi.
- **Bir xil schema'ni create va update uchun ishlatish** → update'da hamma maydon majburiy bo'lib qoladi (NestJS'da `PartialType`).

## 7. Practice problems

### Problem 1 — `validate()` yoz
Yuqoridagi middleware'ni yozib, 3 ta route'ga ula.

### Problem 2 — uchta option
`.strict()` bilan va busiz, `z.coerce.number()` bilan va busiz sinab ko'r.
Har bir holat uchun `curl` natijasini yozib ol.

### Problem 3 — `req.body = result.data` ni o'chir
Nima o'zgardi? `role: 'admin'` yana o'tib ketdimi?

### Problem 4 — query validatsiyasi
`GET /players?page=abc` uchun 400 qaytadigan qil.
`page` ni `number` ga aylantirib, service'da `typeof` bilan tasdiqla.

### Problem 5 — metadata bilan route qurish
4-bo'limdagi `routes.forEach(...)` naqshini yozib ko'r.
**Savol:** bu NestJS decoratorlariga qanchalik yaqin? Nimasi yetishmayapti?

### Problem 6 — unutishni modellashtir
6 ta route'dan bittasida `validate()` ni unut va `{"role":"admin"}` yuborib teshikni top.

### Problem 7 — NestJS'ga o'gir
Barcha zod schema'larni DTO class'lariga aylantir va global `ValidationPipe` yoq.
Route fayllaridan nechta qator yo'qoldi? `validate()` middleware qayerga ketdi?

## 8. Related

[[Express Request Validation]] · [[Express Middleware]] · [[Express Error Handling Middleware]] · [[Express Middleware Order]] · [[NestJS ValidationPipe]] · [[NestJS Pipes]] · [[NestJS DTO]]
