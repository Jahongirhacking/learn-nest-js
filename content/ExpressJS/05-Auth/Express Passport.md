---
tags: [express/auth, level/intermediate]
level: intermediate
topic: Auth
---

# Express Passport

> **Bir jumlada:** Passport — Express kutubxonasi; NestJS uni o'zi ixtiro qilmagan, balki `@nestjs/passport` orqali **o'rab olgan** — shuning uchun Strategy va Guard farqi ikkalasida ham bir xil.

**Before learning:** [[Express Authentication]], [[Express Auth Middleware]]
**After learning:** [[Passport and JwtStrategy]]

**NestJS'dagi mos note:** [[Passport and JwtStrategy]]

---

## 1. Muammo

NestJS'da to'rtta narsa bir vaqtda paydo bo'ladi:

```text
PassportModule · JwtModule · JwtStrategy · JwtAuthGuard
```

Va ular bir-biridan nima bilan farq qilishi tushunarsiz.
Express'da Passport'ni sof holda ko'rsang, farq darhol ochiladi.

## 2. Sof Express + Passport

```js
// config/passport.js
const passport = require('passport');
const { Strategy: JwtStrategy, ExtractJwt } = require('passport-jwt');

passport.use(new JwtStrategy(
  {
    jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
    secretOrKey: process.env.JWT_SECRET,
    ignoreExpiration: false,
  },
  async (payload, done) => {
    try {
      const user = await prisma.user.findUnique({
        where: { id: payload.sub },
        select: { id: true, status: true, roles: { select: { role: true } } },
      });

      if (!user || user.status !== 'ACTIVE') return done(null, false);

      return done(null, {                      // ← bu obyekt req.user bo'ladi
        id: user.id,
        roles: user.roles.map(r => r.role.name),
      });
    } catch (err) {
      return done(err, false);
    }
  },
));

module.exports = passport;
```

```js
// app.js
app.use(passport.initialize());

// route himoyasi
app.get('/me',
  passport.authenticate('jwt', { session: false }),    // ← GUARD
  (req, res) => res.json(req.user),
);
```

## 3. Strategy va Guard farqi — endi ko'rinadi

```text
new JwtStrategy(options, verifyCallback)
   → "TOKENNI QANDAY tekshiraman va undan kimni yasayman?"
   → bir marta ro'yxatga olinadi
   → nomi bor: 'jwt'

passport.authenticate('jwt')
   → "QAYSI route'da tekshiruv kerak?"
   → har bir route'ga qo'lda ulanadi
   → o'zi MIDDLEWARE qaytaradi
```

NestJS'da xuddi shu ikkilik:

```ts
// Strategy — "qanday tekshiraman"
export class JwtStrategy extends PassportStrategy(Strategy, 'jwt') {
  async validate(payload) { ... }        // ← done(null, user) ning ekvivalenti
}

// Guard — "qayerda tekshiraman"
export class JwtAuthGuard extends AuthGuard('jwt') {}
//                                  ^^^^^^^^^^^^^^ passport.authenticate('jwt')
```

**`AuthGuard('jwt')` — bu so'zma-so'z `passport.authenticate('jwt')` ning o'ram.**

## 4. `done()` va `validate()`

| Passport (Express) | NestJS |
|---|---|
| `done(null, user)` | `return user` |
| `done(null, false)` | `throw new UnauthorizedException()` |
| `done(err, false)` | `throw err` |
| `verifyCallback` | `validate()` methodi |
| natija → `req.user` | natija → `request.user` |

Ya'ni NestJS `done` callback'ini `return`/`throw` ga aylantirgan — o'qishga qulayroq, mantiq bir xil.

## 5. Bir nechta strategiya

```js
// Express
passport.use('jwt', new JwtStrategy(...));
passport.use('local', new LocalStrategy(...));
passport.use('google', new GoogleStrategy(...));

app.post('/login',  passport.authenticate('local', { session: false }), loginHandler);
app.get('/auth/google', passport.authenticate('google', { scope: ['email'] }));
app.get('/me',      passport.authenticate('jwt', { session: false }), meHandler);
```

```ts
// NestJS
@UseGuards(AuthGuard('local'))
@Post('login')
login() {}

@UseGuards(AuthGuard('google'))
@Get('auth/google')
google() {}
```

FotSpot'da bir nechta kirish usuli bor ([[FotSpot Auth Architecture]]) — aynan shu naqsh bilan.

## 6. Passport shartmi?

Yo'q. [[Express Authentication]] dagi `requireAuth` middleware'i Passport'siz ham mukammal ishlaydi (20 qator kod).

```text
Passport foydali bo'ladi:
  ✓ bir nechta strategiya (jwt + google + telegram + local)
  ✓ OAuth (redirect, callback, state — ko'p mayda detal)
  ✓ tayyor, sinovdan o'tgan implementatsiyalar

Passport ortiqcha bo'ladi:
  ✓ faqat JWT kerak bo'lsa — 20 qatorlik middleware yetadi
```

NestJS ham Passport'ni **majburlamaydi** — `JwtService` bilan o'z guard'ingni yozishing mumkin.

## 7. Common mistakes

- **`session: false` ni unutish** → Passport session/cookie mexanizmini yoqishga urinadi va `express-session` talab qiladi.
- **`passport.initialize()` ni unutish** → `Unknown authentication strategy`.
- **`done(err)` o'rniga `throw`** → callback ichida xato yutiladi.
- **`ignoreExpiration: true`** → muddati o'tgan token qabul qilinaveradi.
- **`verifyCallback` da faqat payload'ni qaytarish** → bloklangan user token muddatigacha ishlayveradi.
- **`verifyCallback` da og'ir query** → har requestda DB yuklamasi ([[Redis Cache]]).
- **Strategiya nomini xato yozish** (`'jtw'`) → `Unknown authentication strategy`.

## 8. Practice problems

### Problem 1 — Passport'siz
[[Express Authentication]] dagi `requireAuth` middleware'ini yoz va ishlat.

### Problem 2 — Passport bilan
Xuddi shu himoyani `passport-jwt` bilan qayta yoz. Necha qator farq qildi?

### Problem 3 — `session: false` ni o'chir
Nima bo'ldi? Xato matnini o'qi.

### Problem 4 — `done` variantlari
`done(null, user)`, `done(null, false)`, `done(new Error('x'), false)` — uchalasini sinab ko'r.
Har birida HTTP javob qanday bo'ldi?

### Problem 5 — bloklangan user
`verifyCallback` da `user.status !== 'ACTIVE'` tekshiruvini qo'sh.
Userni bloklab, eski tokeni bilan so'rov yubor — 401 keldimi?

### Problem 6 — ikkinchi strategiya
`LocalStrategy` qo'shib, `POST /login` ni u orqali ishlat.

### Problem 7 — NestJS'ga o'gir
`JwtStrategy` ni `PassportStrategy(Strategy, 'jwt')` ga, `passport.authenticate('jwt')` ni `AuthGuard('jwt')` ga aylantir.
**Savol:** `done()` qayerga ketdi?

### Problem 8 — farqni ayt
"Strategy va Guard farqi nima?" degan savolga 2 jumlada javob yoz — Express misolidan foydalanib.

## 9. Related

[[Express Authentication]] · [[Express Auth Middleware]] · [[Passport and JwtStrategy]] · [[JWT Authentication]] · [[FotSpot Auth Architecture]] · [[NestJS Guards]]
