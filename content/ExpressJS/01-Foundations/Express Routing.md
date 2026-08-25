---
tags: [express/foundation, level/beginner]
level: beginner
topic: Express Foundations
---

# Express Routing

> **Bir jumlada:** `app.get('/players/:id', fn)` — bu funksiya chaqiruvi bo'lib, route'ni ro'yxatga qo'shadi; NestJS'dagi `@Get(':id')` decoratori **aynan shu chaqiruvni** sen uchun bajaradi.

**Before learning:** [[Express App Bootstrap]]
**After learning:** [[Express Router]], [[Express Route Handlers]]

**NestJS'dagi mos note:** [[NestJS Decorators]] · [[NestJS Controllers]]

---

## 1. Muammo — decorator nima qiladi?

NestJS:

```ts
@Controller('players')
export class PlayersController {
  @Get(':id')
  findOne(@Param('id') id: string) {}
}
```

Bu qanday qilib `GET /players/:id` ga aylanadi? Express'da javob ochiq:

```js
app.get('/players/:id', (req, res) => {});
```

**Decorator — metadata yozadi, keyin Nest startup'da aynan yuqoridagi chaqiruvni bajaradi.** Xolos.

## 2. Route ro'yxatga olish

```js
app.get('/players', listPlayers);
app.get('/players/:id', getPlayer);
app.post('/players', createPlayer);
app.patch('/players/:id', updatePlayer);
app.delete('/players/:id', deletePlayer);
```

Har bir chaqiruv **ro'yxat oxiriga** yozuv qo'shadi:

```text
[
  { method: 'GET',    path: '/players',      handler: listPlayers },
  { method: 'GET',    path: '/players/:id',  handler: getPlayer },
  { method: 'POST',   path: '/players',      handler: createPlayer },
  ...
]
```

Request kelganda Express ro'yxatni **yuqoridan pastga** aylanib chiqadi va **birinchi mos kelgan**ini ishlatadi.

## 3. Eng klassik tuzoq — tartib

```js
// ✗ XATO
app.get('/players/:id', getPlayer);
app.get('/players/me', getMe);        // ← hech qachon ishlamaydi!

// GET /players/me  →  birinchi route mos keldi  →  req.params.id === 'me'
```

```js
// ✓ TO'G'RI — aniqroq route yuqorida
app.get('/players/me', getMe);
app.get('/players/:id', getPlayer);
```

**Bu xato NestJS'da ham bor!** Chunki NestJS ham decoratorlarni class ichidagi tartibda ro'yxatga oladi:

```ts
@Get('me')      // ← bu YUQORIDA bo'lishi kerak
me() {}

@Get(':id')
findOne() {}
```

Ya'ni bu Express'ning kamchiligi emas — bu **routing'ning tabiati**.

## 4. Route patternlari

```js
app.get('/players/:id', ...)              // req.params.id
app.get('/academies/:aid/trials/:tid', ...) // ikkita param
app.get('/files/*', ...)                   // wildcard
app.get(/^\/legacy\/\d+$/, ...)            // regex
app.all('/admin/*', requireAdmin)          // barcha method

// bir nechta handler — bu MIDDLEWARE ZANJIRI
app.post('/players', requireAuth, validateBody, createPlayer);
```

Oxirgi qator juda muhim:

```text
app.post(path, mw1, mw2, handler)
   → mw1 → next() → mw2 → next() → handler

NestJS'dagi:
   @UseGuards(JwtAuthGuard)  +  ValidationPipe  +  handler
   AYNAN SHU narsa.
```

## 5. Tarjima jadvali

| Express | NestJS |
|---|---|
| `app.get('/players', fn)` | `@Get()` + `@Controller('players')` |
| `app.post('/players', fn)` | `@Post()` |
| `'/players/:id'` | `@Get(':id')` |
| `req.params.id` | `@Param('id')` |
| `app.use('/api/v1', router)` | `setGlobalPrefix('api/v1')` |
| `app.post(p, mw1, mw2, h)` | `@UseGuards(...)` + pipe + handler |
| Route tartibi = fayldagi tartib | Decorator tartibi = class ichidagi tartib |

## 6. Common mistakes

- **`:id` ni statik route'dan oldin qo'yish** → `/players/me` sindiradi (yuqoridagi misol).
- **Route'ni ikki marta ro'yxatga olish** → birinchisi ishlaydi, ikkinchisi jimgina o'lik qoladi.
- **`app.use('/players', ...)` va `app.get('/players', ...)` ni chalkashtirish** → `use` prefiks bo'yicha mos keladi, `get` to'liq yo'l bo'yicha.
- **Wildcard'ni juda erta qo'yish** → keyingi hamma route o'lik.
- **Trailing slash** (`/players/` va `/players`) → default'da bir xil, lekin `strict routing` yoqilsa boshqacha.

## 7. Practice problems

### Problem 1 — tuzoqni ko'r
```js
app.get('/players/:id', (req, res) => res.json({ id: req.params.id }));
app.get('/players/me',  (req, res) => res.json({ me: true }));
```
`curl localhost:3000/players/me` — nima qaytdi? Nega? Tuzat.

### Problem 2 — xuddi shu xatoni NestJS'da takrorla
NestJS controller'da `@Get(':id')` ni `@Get('me')` dan **oldin** yoz va sinab ko'r.
**Xulosa:** bu framework muammosi emas.

### Problem 3 — middleware zanjiri
```js
const a = (req, res, next) => { console.log('a'); next(); };
const b = (req, res, next) => { console.log('b'); next(); };
app.get('/x', a, b, (req, res) => res.json({ ok: true }));
```
Loglar tartibini oldindan ayt, keyin ishga tushir.

### Problem 4 — `b` da `next()` ni o'chir
Nima bo'ladi? `curl` qancha kutadi?

### Problem 5 — ikkita param
`GET /academies/:aid/trials/:tid` yozib, ikkala parametrni qaytar.

### Problem 6 — route ro'yxatini chop et
```js
app._router.stack
  .filter(l => l.route)
  .forEach(l => console.log(Object.keys(l.route.methods)[0].toUpperCase(), l.route.path));
```
**Savol:** ro'yxat tartibi kod tartibiga mos keldimi?

## 8. Related

[[Express Router]] · [[Express Route Handlers]] · [[Express Middleware Order]] · [[Express App Bootstrap]] · [[NestJS Decorators]] · [[NestJS Controllers]]
