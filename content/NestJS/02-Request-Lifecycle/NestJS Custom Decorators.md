---
tags: [nestjs/lifecycle, level/intermediate]
level: intermediate
topic: Request Lifecycle
---

# NestJS Custom Decorators

> **Bir jumlada:** o'z decoratoring — bu yo **metadata yozuvchi** (`SetMetadata`), yo **parametr chiqaruvchi** (`createParamDecorator`); ikkalasi ham controllerni toza saqlaydi.

**Before learning:** [[NestJS Decorators]], [[NestJS Guards]]
**After learning:** [[Public Routes Decorator]], [[Roles and Permissions]]

> 💡 **Tushunmadingmi? Express'da qo'lda qanday yozilishini ko'r:** [[Express Authorization Middleware]]


---

## 1. Definition

Ikki tur:

```ts
// 1. Metadata decorator — guard/interceptor o'qishi uchun
export const ROLES_KEY = 'roles';
export const Roles = (...roles: string[]) => SetMetadata(ROLES_KEY, roles);

// 2. Param decorator — handler parametriga qiymat beradi
export const CurrentUser = createParamDecorator(
  (data: keyof JwtUser | undefined, ctx: ExecutionContext) => {
    const user = ctx.switchToHttp().getRequest().user;
    return data ? user?.[data] : user;
  },
);
```

## 2. Why — nega kerak?

Ularsiz controller shunday ko'rinadi:

```ts
@Get('me')
me(@Req() req: Request) {
  const user = (req as any).user;      // ✗ tipsiz, Express'ga bog'langan
  ...
}
```

Ular bilan:

```ts
@Get('me')
me(@CurrentUser() user: JwtUser) { ... }   // ✓ toza, typed, testlanadigan
```

## 3. Internal mental model

```text
Metadata decorator:
   @Roles('admin')  →  SetMetadata  →  Reflector.get()  →  Guard qarori

Param decorator:
   @CurrentUser()   →  ExecutionContext'dan o'qish  →  handler parametri
```

Muhim: param decorator **guard'dan keyin** ishlaydi — shuning uchun `request.user` allaqachon mavjud ([[Passport and JwtStrategy]]).

## 4. FotSpot implementation

```text
@Public()          → JwtAuthGuard'ga "bu endpoint ochiq" deydi
@Roles('admin')    → RolesGuard o'qiydi
@Permissions('admin:moderate_video') → PermissionsGuard o'qiydi
@CurrentUser()     → request.user'ni handler'ga beradi
```

Ular birgalikda controllerni deklarativ qiladi:

```ts
@Roles('admin')
@Permissions('admin:moderate_video')
@Patch(':id/verify')
verify(@Param('id') id: string, @CurrentUser('id') adminId: string) {
  return this.moderation.verify(id, adminId);
}
```

Bir qarashda endpointning **kim uchun** ekani ko'rinib turibdi — bu [[FotSpot Media Moderation]] kabi sezgir joylarda juda qimmatli.

## 5. Code example

Bir nechta decoratorni birlashtirish (`applyDecorators`):

```ts
export function Auth(...roles: string[]) {
  return applyDecorators(
    Roles(...roles),
    UseGuards(JwtAuthGuard, RolesGuard),
    ApiBearerAuth(),
    ApiUnauthorizedResponse({ description: 'Unauthorized' }),
  );
}

// ishlatish
@Auth('admin')
@Delete(':id')
remove(@Param('id') id: string) {}
```

## 6. Common mistakes

- **Metadata kalitini string sifatida turli joyda qayta yozish** → konstanta chiqar (`ROLES_KEY`).
- **Param decorator ichida og'ir ish qilish** (DB query) → har bir requestda ishlaydi, service'ga ko'chir.
- **`@CurrentUser()` ni public endpointda `undefined` bo'lishini hisobga olmaslik** → [[Public Routes Decorator]] da user bo'lmasligi mumkin.
- **`createParamDecorator` natijasini Swagger bilmasligi** → `@ApiBearerAuth()` va tavsifni qo'lda qo'shish kerak.

## 7. Practice

1. `@CurrentUser()` yozib, `GET /users/me` da ishlat.
2. `@CurrentUser('id')` variantini (data parametri bilan) qo'shib ko'r.
3. `@Roles()` + `RolesGuard` ni noldan yoz.
4. `applyDecorators` bilan `@Auth('admin')` yasab, controller kodini qanchaga qisqarganini ko'r.

## 8. Related concepts

[[NestJS Decorators]] · [[NestJS Guards]] · [[Public Routes Decorator]] · [[Roles and Permissions]] · [[Authorization]] · [[Passport and JwtStrategy]]
