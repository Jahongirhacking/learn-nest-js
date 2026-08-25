---
tags: [auth, level/intermediate, hub]
level: intermediate
topic: Authorization
---

# Authorization

> **Bir jumlada:** Authorization "**senga nima qilishga ruxsat bor?**" degan savolga javob beradi — va u har doim authentication'dan **keyin** keladi.

**Before learning:** [[Authentication]], [[NestJS Guards]]
**After learning:** [[RBAC]], [[Roles and Permissions]], [[Public Routes Decorator]]

> 💡 **Tushunmadingmi? Express'da qo'lda qanday yozilishini ko'r:** [[Express Authorization Middleware]]


---

## 1. Definition

```text
Authentication → "Sen kimsan?"        → JwtAuthGuard
Authorization  → "Senga ruxsat bormi?" → RolesGuard, PermissionsGuard
```

Ikkalasi ham guard, lekin savoli boshqa.

## 2. Why — nega kerak?

Chunki barcha authenticated userlar teng emas:

```text
player          → o'z profilini tahrirlaydi
coach           → trial yaratadi
academy_manager → akademiyani boshqaradi
admin           → media moderatsiya qiladi
super_admin     → role va permissionlarni boshqaradi
```

## 3. Internal mental model

Uch xil authorization darajasi bor va uchalasi ham kerak:

```text
1. Role-based        → "admin bo'lsa mumkin"          → RolesGuard
2. Permission-based  → "admin:moderate_video bo'lsa"  → PermissionsGuard
3. Ownership-based   → "bu o'zining resursimi?"       → Service ichida!
```

Uchinchisiga alohida e'tibor: guard `userId === trial.createdBy` ekanini bilish uchun ma'lumotni database'dan o'qishi kerak bo'ladi — bu ko'pincha **service**ning ishi:

```ts
if (academy.ownerId !== userId) throw new ForbiddenException();
```

## 4. FotSpot implementation

```text
Request
   ↓
JwtAuthGuard        → request.user = { id, roles, permissions }
   ↓
RolesGuard          → @Roles('admin') metadata'sini tekshiradi
   ↓
PermissionsGuard    → @Permissions('admin:moderate_video') tekshiradi
   ↓
Controller → Service → ownership tekshiruvi
```

Model: `User → UserRole → Role → RolePermission → Permission` ([[RBAC]]).

Default holat — **yopiq**: guard global, ochish uchun aniq `@Public()` kerak ([[Public Routes Decorator]]).

## 5. Code example

```ts
@Injectable()
export class PermissionsGuard implements CanActivate {
  constructor(private readonly reflector: Reflector) {}

  canActivate(ctx: ExecutionContext): boolean {
    const required = this.reflector.getAllAndOverride<string[]>(PERMISSIONS_KEY, [
      ctx.getHandler(),
      ctx.getClass(),
    ]);
    if (!required?.length) return true;

    const { user } = ctx.switchToHttp().getRequest();
    const has = required.every(p => user?.permissions?.includes(p));
    if (!has) throw new ForbiddenException('Missing permission');
    return true;
  }
}
```

```ts
@Roles('admin')
@Permissions('admin:moderate_video')
@Patch('media/:id/verify')
verify(@Param('id') id: string) {}
```

## 6. Common mistakes

- **Ownership tekshiruvini unutish** → `admin` roli bor deb, boshqa userning resursini o'zgartirishga ruxsat berish; yoki `PATCH /players/:id` da `:id` kimniki ekanini tekshirmaslik (IDOR).
- **Role'ni JWT ichiga qotirib qo'yib, hech qachon yangilamaslik** → role o'zgarsa, eski token eski huquq bilan qoladi. Token muddatini qisqa qil.
- **Authorization'ni frontend'da qilish** → backend baribir tekshirishi shart.
- **Role'ni clientdan qabul qilish** (`CreateUserDto`da `role`) → [[NestJS ValidationPipe]] `forbidNonWhitelisted` shuning uchun kerak.

## 7. Practice

1. `RolesGuard` va `PermissionsGuard` yoz.
2. `PATCH /players/:id` ga ownership tekshiruvini qo'sh: faqat egasi yoki admin.
3. Ikki xil user bilan sinab ko'r: 403 to'g'ri holatda qaytyaptimi?
4. Role'ni token ichidan olish va DB'dan qayta o'qish variantlarini solishtir — qaysi biri qachon to'g'ri?

## 8. Related concepts

[[Authentication]] · [[RBAC]] · [[Roles and Permissions]] · [[NestJS Guards]] · [[Public Routes Decorator]] · [[FotSpot Guard Chain]]
