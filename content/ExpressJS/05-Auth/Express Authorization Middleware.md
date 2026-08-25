---
tags: [express/auth, level/intermediate, security]
level: intermediate
topic: Auth
---

# Express Authorization Middleware

> **Bir jumlada:** `requireRole('admin')` — bu **middleware qaytaruvchi funksiya**; NestJS'dagi `@Roles('admin')` + `RolesGuard` juftligi aynan shu naqshning metadata bilan yozilgan varianti.

**Before learning:** [[Express Auth Middleware]]
**After learning:** [[Express Middleware Order]], [[RBAC]]

**NestJS'dagi mos note:** [[Authorization]] · [[Roles and Permissions]] · [[RBAC]]

---

## 1. Middleware factory naqshi

```js
// middleware/require-role.js
module.exports = function requireRole(...allowed) {
  return function (req, res, next) {          // ← MIDDLEWARE qaytaradi
    if (!req.user) return res.status(401).json({ message: 'Unauthorized' });

    const ok = req.user.roles?.some(r => allowed.includes(r));
    if (!ok) return res.status(403).json({ message: 'Insufficient role' });

    next();
  };
};
```

```js
app.delete('/players/:id', requireAuth, requireRole('admin', 'super_admin'), deletePlayer);
```

Nima bo'lyapti:

```text
requireRole('admin')
   → funksiya CHAQIRILADI (route ro'yxatga olinayotganda)
   → u yangi middleware funksiyasini QAYTARADI
   → o'sha middleware har bir requestda ishlaydi

Ya'ni 'admin' qiymati CLOSURE ichida saqlanadi.
```

**NestJS'da xuddi shu ma'lumot metadata'da saqlanadi:**

```ts
export const Roles = (...roles: string[]) => SetMetadata(ROLES_KEY, roles);
//                                            ^^^^^^^^^^^ closure o'rniga metadata

// guard uni o'qiydi
const required = this.reflector.getAllAndOverride<string[]>(ROLES_KEY, [...]);
```

| Express | NestJS |
|---|---|
| `requireRole('admin')` — closure | `@Roles('admin')` — metadata |
| Middleware factory | `SetMetadata` + `Reflector` |
| Har route'da qo'lda ulanadi | Global guard metadata'ni o'qiydi |

**Nega NestJS metadata'ni tanladi?** Chunki global guard bitta bo'ladi va u **har bir** endpoint uchun metadata'ni o'qiy oladi — ulashni unutib bo'lmaydi.

## 2. Permission tekshiruvi

```js
// middleware/require-permission.js
module.exports = function requirePermission(...required) {
  return function (req, res, next) {
    if (!req.user) return res.status(401).json({ message: 'Unauthorized' });

    const has = required.every(p => req.user.permissions?.includes(p));
    if (!has) return res.status(403).json({ message: 'Missing permission' });

    next();
  };
};
```

```js
app.patch('/media/:id/verify',
  requireAuth,
  requireRole('admin'),
  requirePermission('admin:moderate_video'),
  verifyMedia,
);
```

Permissionlarni qayerdan olish kerak? Ikki variant — ikkalasining ham narxi bor:

```js
// A) Tokenga solish — tez, lekin eskiradi
const accessToken = jwt.sign({ sub: user.id, roles, permissions }, secret, { expiresIn: '15m' });
// ✓ DB query yo'q
// ✗ huquq olib qo'yilsa, token muddatigacha ishlayveradi

// B) Har requestda DB'dan o'qish — aniq, lekin qimmat
async function loadPermissions(req, res, next) {
  req.user.permissions = await rbacService.getPermissions(req.user.id);   // JOIN
  next();
}
// ✓ doim dolzarb
// ✗ har requestda og'ir query → [[Redis Cache]] kerak
```

Bu tanlov NestJS'da ham **aynan bir xil** — framework uni hal qilmaydi ([[RBAC]]).

## 3. RBAC query — qo'lda

```js
// services/rbac.service.js
async function getPermissions(userId) {
  const user = await prisma.user.findUnique({
    where: { id: userId },
    include: {
      roles: {
        include: {
          role: { include: { permissions: { include: { permission: true } } } },
        },
      },
    },
  });

  return [...new Set(
    user.roles.flatMap(ur => ur.role.permissions.map(rp => rp.permission.name)),
  )];
}
```

Bu kod NestJS'da **so'zma-so'z bir xil** — chunki bu Prisma kodi, framework emas.
Nazariya: [[RBAC]] · [[Database Relations]]

## 4. Ownership — uchinchi daraja

```text
1. Role       → "admin bo'lsa mumkin"           → middleware
2. Permission → "admin:moderate_video bo'lsa"   → middleware
3. Ownership  → "bu o'zining resursimi?"        → SERVICE
```

```js
// ✗ Middleware'da qilishga urinish
function requireOwnership(req, res, next) {
  // resursni o'qish kerak → DB → bu business mantiq
  // va har bir resurs turi uchun boshqacha
}

// ✓ Service ichida
async function update(playerId, dto, user) {
  const player = await prisma.playerProfile.findUnique({ where: { id: playerId } });
  if (!player) throw NotFound('Player not found');

  const isOwner = player.userId === user.id;
  const isAdmin = user.roles.includes('admin');
  if (!isOwner && !isAdmin) throw Forbidden('Not your profile');

  return prisma.playerProfile.update({ where: { id: playerId }, data: dto });
}
```

**Bu eng ko'p unutiladigan qatlam.** Role tekshirildi deb, ownership'ni tashlab ketish — IDOR zaifligi.

## 5. Common mistakes

- **`requireRole` ni `requireAuth`siz ishlatish** → `req.user` `undefined` → crash yoki noto'g'ri ruxsat.
- **Ownership'ni unutish** → har qanday `coach` boshqa coach'ning trial'ini tahrirlaydi.
- **Role nomini har joyda string sifatida yozish** → `'admn'` xatosi jimgina 403 beradi. Konstanta ishlat.
- **Permissionni tokenga solib, hech qachon yangilamaslik** → huquq olib qo'yilgandan keyin ham ishlaydi.
- **Frontendda yashirib, backendda tekshirmaslik** → endpoint ochiq.
- **403 va 401 ni chalkashtirish** → 401 "kim ekaning noma'lum", 403 "kim ekaning ma'lum, lekin ruxsat yo'q".

## 6. Practice problems

### Problem 1 — `requireRole` yoz
Middleware factory sifatida yozib, `admin` va `coach` uchun turli endpointlar yasab ko'r.

### Problem 2 — closure'ni tushun
```js
const mw = requireRole('admin');
console.log(typeof requireRole, typeof mw);
```
Nima chiqdi? `'admin'` qayerda saqlanyapti?

### Problem 3 — `requireAuth`siz
`requireRole` ni `requireAuth`siz ishlatib ko'r. Nima bo'ldi?
Buni himoyalash uchun middleware ichiga nima qo'shding?

### Problem 4 — permission
`requirePermission` yozib, `admin:moderate_video` talab qiluvchi endpoint yasab ko'r.
Permissionlarni avval tokendan, keyin DB'dan olib ko'r — farqni yoz.

### Problem 5 — RBAC query
`getPermissions(userId)` ni yozib, natijani logla.
Necha marta SQL query ketdi? (Prisma query loglarini yoq.)

### Problem 6 — ownership
`PATCH /players/:id` yozib, boshqa userning profilini o'zgartirishga urin.
Avval ownership tekshiruvisiz, keyin bilan. Farqni yozib ol.

### Problem 7 — huquqni olib qo'yish
Userdan `admin` rolini DB'dan olib tashla, lekin uning eski tokeni bilan so'rov yubor.
Tokenga solingan variantda nima bo'ldi? DB'dan o'qiydigan variantda-chi?

### Problem 8 — NestJS'ga o'gir
`requireRole` → `@Roles()` + `RolesGuard`, `requirePermission` → `@Permissions()` + `PermissionsGuard`.
**Savol:** closure qayerga ketdi? Uning o'rniga nima keldi?

## 7. Related

[[Express Auth Middleware]] · [[Express Authentication]] · [[Express Middleware Order]] · [[Authorization]] · [[RBAC]] · [[Roles and Permissions]] · [[NestJS Custom Decorators]] · [[FotSpot Guard Chain]]
