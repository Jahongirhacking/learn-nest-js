---
tags: [express/auth, level/intermediate, security]
level: intermediate
topic: Auth
---

# Express Auth Middleware

> **Bir jumlada:** Guard — bu `(req, res, next)` funksiyasi bo'lib, `next()` chaqirsa "ha", `res.status(401)` qaytarsa "yo'q" degani; NestJS Guard'i **aynan shu**, faqat `true`/`false` qaytaradi.

**Before learning:** [[Express Middleware]], [[Express Authentication]]
**After learning:** [[Express Authorization Middleware]], [[Express Middleware Order]]

**NestJS'dagi mos note:** [[NestJS Guards]] · [[Public Routes Decorator]]

---

## 1. Guard = middleware

```js
// Express
function requireAuth(req, res, next) {
  if (!valid) return res.status(401).json({ message: 'Unauthorized' });
  req.user = payload;
  next();                                   // ← "true"
}
```

```ts
// NestJS
@Injectable()
export class JwtAuthGuard implements CanActivate {
  canActivate(ctx: ExecutionContext): boolean {
    if (!valid) throw new UnauthorizedException();
    return true;                            // ← next()
  }
}
```

Xaritalash:

```text
next()                    →  return true
res.status(401).json(...) →  throw new UnauthorizedException()
req.user = payload        →  request.user = payload
```

## 2. Guard'larni zanjirlash

```js
app.post('/trials',
  rateLimit,                    // → ThrottleGuard
  requireAuth,                  // → JwtAuthGuard
  requireRole('coach'),         // → RolesGuard
  requirePermission('trial:create'),  // → PermissionsGuard
  validate(createTrialSchema),  // → ValidationPipe
  asyncHandler(createTrial),    // → Controller
);
```

**Bu FotSpot guard zanjirining aynan o'zi** ([[FotSpot Guard Chain]]).

Tartib sababi ham bir xil:

```text
rateLimit birinchi   → jwt.verify (kripto) va DB'dan oldin hujimni to'xtat
requireAuth ikkinchi → role'ni bilish uchun user kerak
requireRole uchinchi → keng, arzon filtr
requirePermission    → nozik, qimmat tekshiruv
```

## 3. Uch xil ulash usuli va ularning xavfi

```js
// A) Har route'da qo'lda
app.get('/players',       listPlayers);                 // ochiq
app.post('/players',      requireAuth, createPlayer);   // yopiq
app.delete('/players/:id', deletePlayer);               // ✗ UNUTILDI

// B) Prefiks bo'yicha
app.use('/api/v1/admin', requireAuth, requireRole('admin'));

// C) Global + istisno
app.use('/api/v1/auth', authRoutes);       // ochiq route'lar OLDIN
app.use(requireAuth);                       // ← shundan keyin hammasi yopiq
app.use('/api/v1/players', playersRoutes);
```

```text
A → unutsang: JIMGINA teshik (hech kim sezmaydi)
C → unutsang: SHOVQINLI 401 (darhol ko'rinadi)
```

FotSpot **C** ni tanlagan: global `JwtAuthGuard` + `@Public()`.
Endi [[Public Routes Decorator]] ning sababi to'liq ochiq.

## 4. Optional authentication

FotSpot'da `@Public()` endpointda token bo'lsa, user baribir aniqlanadi.
Express'da bu shunday yoziladi:

```js
// middleware/optional-auth.js
module.exports = function optionalAuth(req, res, next) {
  const header = req.headers.authorization;
  if (!header?.startsWith('Bearer ')) return next();     // guest — davom etadi

  try {
    const payload = jwt.verify(header.slice(7), process.env.JWT_SECRET);
    req.user = { id: payload.sub, roles: payload.roles || [] };
  } catch {
    // noto'g'ri token — guest sifatida davom etamiz
  }
  next();
};
```

```js
app.get('/players', optionalAuth, asyncHandler(async (req, res) => {
  res.json(await playersService.findAllPublic(req.query, req.user?.id));
  //                                                     ^^^^^^^^^^ bo'lishi ham, bo'lmasligi ham mumkin
}));
```

Xuddi shu mantiq NestJS `JwtAuthGuard.handleRequest()` da yozilgan ([[Passport and JwtStrategy]]).

## 5. Ownership — guard'da EMAS

```js
// ✗ Guard'da qilib bo'lmaydi — DB kerak, va bu business qarori
function requireOwnership(req, res, next) {
  // trial'ni o'qish kerak... bu guard ishi emas
}

// ✓ Service ichida
async function finish(trialId, dto, userId) {
  const trial = await prisma.trial.findUnique({ where: { id: trialId } });
  if (!trial) throw NotFound('Trial not found');
  if (trial.coachId !== userId) throw Forbidden('Not your trial');
  ...
}
```

Bu qoida ikkala frameworkda ham bir xil ([[Authorization]]).

## 6. Common mistakes

- **`return res.status(401)` dagi `return` ni unutish** → `next()` ham ishlaydi.
- **`requireAuth` ni login route'idan oldin global qo'yish** → hech kim login qila olmaydi.
- **Guard ichida business logic yozish** → test qilib bo'lmaydi, qayta ishlatilmaydi.
- **Guard ichida `req.body` ga ishonish** → validatsiya hali bo'lmagan ([[Express Middleware Order]]).
- **Optional auth'da xatoni yutmaslik** → noto'g'ri token bilan public endpoint 401 beradi.
- **`req.user` ni tekshirmasdan `req.user.id` o'qish** → public route'da crash.

## 7. Practice problems

### Problem 1 — `requireAuth`
Yozib, `GET /me` ni himoyala. Tokensiz 401 kelishini tasdiqla.

### Problem 2 — zanjir
`rateLimit → requireAuth → requireRole → validate → handler` zanjirini yoz.
Har biriga log qo'yib, tartibni tasdiqla.

### Problem 3 — tartibni buz
`rateLimit` ni `requireAuth` dan **keyinga** ko'chir.
`jwt.verify` ga log qo'yib, 20 ta noto'g'ri so'rov yubor. Necha marta chaqirildi?
Keyin tartibni to'g'irlab, qayta o'lchab ko'r.

### Problem 4 — Variant A tuzog'i
6 ta route yozib, bittasida `requireAuth` ni unut. Keyin uni **testsiz** topishga urin.

### Problem 5 — Variant C
Xuddi shu 6 ta route'ni global `app.use(requireAuth)` bilan qayta yoz.
Endi unutish mumkinmi?

### Problem 6 — optional auth
`optionalAuth` yozib, bitta endpointni ham guest, ham authenticated user ishlata olishini ta'minla.
`req.user?.id` ni logla.

### Problem 7 — NestJS'ga o'gir
`requireAuth` ni `JwtAuthGuard` ga, `optionalAuth` ni `@Public()` mantiqiga aylantir.
Ikkalasini yonma-yon qo'yib, qaysi biri xatoga chidamli ekanini yoz.

## 8. Related

[[Express Authentication]] · [[Express Authorization Middleware]] · [[Express Middleware Order]] · [[Express Passport]] · [[NestJS Guards]] · [[Public Routes Decorator]] · [[FotSpot Guard Chain]] · [[Authorization]]
