---
tags: [express/auth, level/intermediate, security]
level: intermediate
topic: Auth
---

# Express Authentication

> **Bir jumlada:** JWT'ni qo'lda chiqarish va tekshirish — bu 20 qatorlik kod; NestJS undan `JwtModule` + `JwtStrategy` + `JwtAuthGuard` uchligini yasagan, lekin mantiq o'zgarmagan.

**Before learning:** [[Express Middleware]], [[Express Request and Response]]
**After learning:** [[Express Auth Middleware]], [[Express Passport]]

**NestJS'dagi mos note:** [[Authentication]] · [[JWT Authentication]]

---

## 1. To'liq login flow — qo'lda

```js
// services/auth.service.js
const argon2 = require('argon2');
const jwt = require('jsonwebtoken');
const { randomUUID } = require('crypto');

class AuthService {
  constructor(prisma) { this.prisma = prisma; }

  async register({ email, password }) {
    const passwordHash = await argon2.hash(password, { type: argon2.argon2id });

    const user = await this.prisma.user.create({
      data: { email, passwordHash },
      select: { id: true, email: true },        // ← hash hech qachon qaytmaydi
    });
    return user;
  }

  async login({ email, password }, ctx) {
    const user = await this.prisma.user.findUnique({
      where: { email },
      include: { roles: { include: { role: true } } },
    });

    // MUHIM: ikkala holatda ham BIR XIL javob
    if (!user || !(await argon2.verify(user.passwordHash, password))) {
      throw Unauthorized('Invalid credentials');
    }

    return this.issueSession(user, ctx);
  }

  async issueSession(user, ctx) {
    const roles = user.roles.map(r => r.role.name);

    const accessToken = jwt.sign(
      { sub: user.id, roles },
      process.env.JWT_SECRET,
      { expiresIn: '15m' },
    );

    const refreshToken = randomUUID();
    const refreshTokenHash = await argon2.hash(refreshToken);

    await this.prisma.session.create({
      data: {
        userId: user.id,
        refreshTokenHash,
        deviceId: ctx.deviceId,
        userAgent: ctx.userAgent,
        ipAddress: ctx.ip,
        expiresAt: new Date(Date.now() + 30 * 24 * 3600 * 1000),
      },
    });

    return { accessToken, refreshToken };
  }
}
```

```js
// routes/auth.routes.js
router.post('/login', validate(loginSchema), asyncHandler(async (req, res) => {
  const tokens = await authService.login(req.body, {
    deviceId: req.headers['x-device-id'],
    userAgent: req.headers['user-agent'],
    ip: req.ip,
  });
  res.json(tokens);
}));
```

## 2. Token tekshirish

```js
// middleware/require-auth.js
const jwt = require('jsonwebtoken');

module.exports = function requireAuth(req, res, next) {
  const header = req.headers.authorization;

  if (!header?.startsWith('Bearer ')) {
    return res.status(401).json({ message: 'Unauthorized' });
  }

  try {
    const payload = jwt.verify(header.slice(7), process.env.JWT_SECRET);
    req.user = { id: payload.sub, roles: payload.roles || [] };    // ← MANA SHU
    next();
  } catch {
    return res.status(401).json({ message: 'Invalid or expired token' });
  }
};
```

**`req.user = ...`** — bu bitta qator NestJS'da butun bir zanjirga aylanadi:

```text
Express:  requireAuth  →  req.user = {...}
NestJS:   JwtAuthGuard → passport-jwt → JwtStrategy.validate() → req.user
```

Ikkalasi ham **aynan bir xil natija** beradi. NestJS'da bosqichlar ko'proq, chunki har biri almashtiriladigan qilingan.

## 3. Refresh flow

```js
async refresh(refreshToken, ctx) {
  const sessions = await this.prisma.session.findMany({
    where: { revokedAt: null, expiresAt: { gt: new Date() }, deviceId: ctx.deviceId },
  });

  let session = null;
  for (const s of sessions) {
    if (await argon2.verify(s.refreshTokenHash, refreshToken)) { session = s; break; }
  }
  if (!session) throw Unauthorized('Invalid refresh token');

  // ROTATION: yangi token, eskisi bekor
  const newRefresh = randomUUID();
  await this.prisma.session.update({
    where: { id: session.id },
    data: { refreshTokenHash: await argon2.hash(newRefresh) },
  });

  const user = await this.prisma.user.findUnique({
    where: { id: session.userId },
    include: { roles: { include: { role: true } } },
  });

  return {
    accessToken: jwt.sign({ sub: user.id, roles: user.roles.map(r => r.role.name) },
      process.env.JWT_SECRET, { expiresIn: '15m' }),
    refreshToken: newRefresh,
  };
}
```

Nazariy asosi: [[Session Architecture]] · [[JWT Authentication]]

## 4. Express va NestJS yonma-yon

| Express | NestJS |
|---|---|
| `jwt.sign(...)` | `this.jwtService.signAsync(...)` |
| `jwt.verify(...)` | `JwtStrategy` (passport ichida) |
| `req.user = payload` | `validate()` return qiymati |
| `requireAuth` middleware | `JwtAuthGuard` |
| `app.use(requireAuth)` | `{ provide: APP_GUARD, useClass: JwtAuthGuard }` |
| ochiq route'ni auth'dan oldin qo'yish | `@Public()` decoratori |
| `process.env.JWT_SECRET` | `configService.getOrThrow('JWT_SECRET')` |
| `argon2.hash/verify` | **aynan bir xil** |
| Prisma Session modeli | **aynan bir xil** |

E'tibor ber: **auth mantiqining o'zi hech o'zgarmagan.**
O'zgargan narsa — u qanday **ulanadi** va **unutish qanchalik oson**.

## 5. Common mistakes

- **"User topilmadi" va "parol noto'g'ri" ni ajratish** → user enumeration.
- **`passwordHash` ni javobga qo'shish** → `select` bilan kes.
- **Refresh tokenni plain saqlash** → DB leak = barcha hisoblar.
- **Rotation qilmaslik** → o'g'irlangan token abadiy.
- **Login endpointini rate limit'siz qoldirish** → [[Express Rate Limiting]].
- **`JWT_SECRET` ni kodda qoldirish** → [[Express Config and Env]].
- **Access token TTL'ini uzun qilish** (7 kun) → o'g'irlansa uzoq zarar.
- **Tokenni logga yozish** → [[Express Logging]].

## 6. Practice problems

### Problem 1 — register + login
argon2 bilan hash qilib, `POST /auth/register` va `POST /auth/login` yoz.

### Problem 2 — hash ko'rinishi
Bir xil parolni ikki marta hashlab, natijalar har xil ekanini tasdiqla. Nega?

### Problem 3 — `requireAuth`
Middleware yozib, `GET /users/me` ni himoyala. Tokensiz, buzilgan token va to'g'ri token bilan sinab ko'r.

### Problem 4 — token ichini o'qi
Chiqarilgan tokenni `jwt.io` ga qo'y yoki:
```js
console.log(JSON.parse(Buffer.from(token.split('.')[1], 'base64').toString()));
```
**Savol:** payload shifrlanganmi? Unga maxfiy ma'lumot solish mumkinmi?

### Problem 5 — enumeration
Mavjud bo'lmagan email va noto'g'ri parol uchun javoblarni solishtir. Bir xilmi?
Bir xil bo'lmasa — tuzat.

### Problem 6 — refresh + rotation
`POST /auth/refresh` yozib rotation qil. Eski refresh bilan qayta urin — rad etildimi?

### Problem 7 — NestJS'ga o'gir
`requireAuth` ni `JwtAuthGuard` + `JwtStrategy` ga aylantir.
**Savol:** `req.user = ...` qatori qayerga ketdi?

### Problem 8 — global qil
Express'da `app.use(requireAuth)` ni login route'idan **keyin** qo'yib ko'r.
Nima buzildi? Bu NestJS'dagi qaysi muammoga mos keladi?

## 7. Related

[[Express Auth Middleware]] · [[Express Authorization Middleware]] · [[Express Passport]] · [[Express Rate Limiting]] · [[Authentication]] · [[JWT Authentication]] · [[Password Hashing]] · [[Session Architecture]]
