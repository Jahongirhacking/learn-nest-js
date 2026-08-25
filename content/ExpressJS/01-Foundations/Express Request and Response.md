---
tags: [express/foundation, level/beginner]
level: beginner
topic: Express Foundations
---

# Express Request and Response

> **Bir jumlada:** `req` va `res` — Node'ning stream obyektlari; NestJS ularni decoratorlar ortiga yashiradi, lekin ular hech qayoqqa yo'qolmaydi.

**Before learning:** [[Node HTTP Server]]
**After learning:** [[Express Middleware]], [[Express Request Validation]]

**NestJS'dagi mos note:** [[NestJS Controllers]] · [[NestJS Custom Decorators]]

---

## 1. Muammo

NestJS'da shunday yozasan:

```ts
@Get(':id')
findOne(
  @Param('id') id: string,
  @Query('full') full: string,
  @Headers('authorization') auth: string,
  @CurrentUser() user: JwtUser,
) {}
```

To'rtta decorator, to'rtta parametr. **Ular qayerdan keladi?**
Javob: hammasi bitta `req` obyektidan.

## 2. `req` — nima bor ichida

```js
app.get('/players/:id', (req, res) => {
  console.log(req.method);        // 'GET'
  console.log(req.originalUrl);   // '/api/v1/players/abc?full=true'
  console.log(req.path);          // '/api/v1/players/abc'

  console.log(req.params);        // { id: 'abc' }        ← @Param()
  console.log(req.query);         // { full: 'true' }     ← @Query()
  console.log(req.body);          // { ... }              ← @Body()
  console.log(req.headers);       // { authorization: 'Bearer ...' } ← @Headers()
  console.log(req.ip);

  console.log(req.user);          // ← BU YO'Q! Buni sen qo'shasan
});
```

Oxirgi qator — eng muhim tushuncha:

```text
req.user Express'da MAVJUD EMAS.
Uni auth middleware qo'lda qo'yadi:

   req.user = { id, roles };

NestJS'da @CurrentUser() ham AYNAN shu maydonni o'qiydi.
```

Ya'ni `@CurrentUser()` decoratori — bu `req.user` ni o'qiydigan 3 qatorlik funksiya, boshqa hech narsa emas.

## 3. `res` — javob yozish

```js
res.status(201).json({ id: 1 });      // status + JSON
res.status(204).send();               // bo'sh javob
res.setHeader('X-Total-Count', '42');
res.redirect('/login');
```

Muhim qoidalar:

```text
1. Javob FAQAT BIR MARTA yoziladi
      res.json(a); res.json(b);   →  ERR_HTTP_HEADERS_SENT

2. Javob yozgandan keyin return qil
      if (!player) return res.status(404).json({...});
                   ^^^^^^ bu return juda muhim

3. Javob yozilmasa — request osilib qoladi
```

Ikkinchi nuqta Express'dagi eng ko'p uchraydigan xato:

```js
// ✗ XATO — return yo'q, kod davom etadi
if (!player) res.status(404).json({ message: 'Not found' });
res.json(player);           // ← bu ham ishlaydi → crash

// ✓ TO'G'RI
if (!player) return res.status(404).json({ message: 'Not found' });
res.json(player);
```

**NestJS'da bu xatoni qilib bo'lmaydi**, chunki sen `throw new NotFoundException()` yozasan va funksiya darhol to'xtaydi.

## 4. Tarjima jadvali

| Express | NestJS | Izoh |
|---|---|---|
| `req.params.id` | `@Param('id') id` | |
| `req.query` | `@Query() dto` | Nest'da DTO'ga aylantiriladi |
| `req.body` | `@Body() dto` | Nest'da validatsiyadan o'tgan |
| `req.headers.authorization` | `@Headers('authorization')` | |
| `req.user` (sen qo'yasan) | `@CurrentUser()` | ikkalasi ham `req.user` |
| `req` ning o'zi | `@Req() req` | **aynan bir xil obyekt** |
| `res.status(201).json(x)` | `@HttpCode(201)` + `return x` | |
| `res.json(x)` | `return x` | Nest o'zi yozadi |
| `res` ning o'zi | `@Res() res` | ishlatsang, Nest javob yozishni to'xtatadi! |

Oxirgi qator muhim tuzoq: NestJS'da `@Res()` ishlatsang, framework "javobni sen yozasan" deb hisoblaydi va `return` qilgan qiymating **e'tiborsiz qoladi**.

## 5. `req.user` ning to'liq yo'li

```js
// 1. Auth middleware qo'yadi
function requireAuth(req, res, next) {
  const payload = jwt.verify(token, secret);
  req.user = { id: payload.sub, roles: payload.roles };   // ← MANA SHU YER
  next();
}

// 2. Keyingi middleware o'qiydi
function requireRole(role) {
  return (req, res, next) => {
    if (!req.user.roles.includes(role)) return res.status(403).json({...});
    next();
  };
}

// 3. Handler o'qiydi
app.get('/me', requireAuth, (req, res) => res.json(req.user));
```

NestJS'da xuddi shu zanjir:

```text
JwtStrategy.validate()  →  qaytgan obyekt req.user ga yoziladi
RolesGuard              →  req.user.roles ni o'qiydi
@CurrentUser()          →  req.user ni handler'ga beradi
```

Batafsil: [[Express Auth Middleware]] · [[Passport and JwtStrategy]]

## 6. Common mistakes

- **`return res.json(...)` dagi `return` ni unutish** → ikki marta javob yozish.
- **`req.body` `undefined`** → `express.json()` qo'yilmagan.
- **`req.query.page` ni number deb o'ylash** → **har doim string**. `Number(req.query.page)` kerak. NestJS'da buni `transform: true` qiladi ([[NestJS ValidationPipe]]).
- **`req` ga xohlagancha maydon qo'shish** (`req.foo`, `req.bar`) → tartibsizlik; TypeScript'da tip kengaytirish kerak.
- **`res` ni async funksiya ichida `await` dan keyin ishlatib, xatoni ushlamaslik** → [[Express Async Errors]].

## 7. Practice problems

### Problem 1 — hamma maydonni ko'r
`GET /debug/:id?a=1&b=2` endpoint yozib, `req.params`, `req.query`, `req.headers` ni JSON qilib qaytar.

### Problem 2 — query tipi
`GET /players?page=2` da `typeof req.query.page` ni logla.
**Kutilgan natija:** `'string'`. **Savol:** NestJS'da bu qanday `number` bo'lib qoladi?

### Problem 3 — ikki marta javob
Ataylab `res.json(a)` dan keyin `res.json(b)` yoz. Xato matnini to'liq o'qi.

### Problem 4 — `return` ni unut
```js
if (!player) res.status(404).json({ message: 'Not found' });
res.json(player);
```
Mavjud bo'lmagan id bilan chaqir. Nima bo'ldi?

### Problem 5 — `req.user` ni o'zing qo'y
Middleware yozib, `req.user = { id: 'test' }` qil va handler'da o'qi.
**Savol:** bu `@CurrentUser()` dan qanday farq qiladi?

### Problem 6 — NestJS'da `@Req()`
NestJS controller'da `@Req() req: Request` olib, `req.params` va `req.query` ni logla.
**Savol:** bu Express obyektimi? Qanday tekshirasan?

## 8. Related

[[Node HTTP Server]] · [[Express Middleware]] · [[Express Auth Middleware]] · [[Express Request Validation]] · [[NestJS Controllers]] · [[NestJS Custom Decorators]]
