---
tags: [nestjs/lifecycle, level/beginner]
level: beginner
topic: Request Lifecycle
---

# NestJS Guards

> **Bir jumlada:** Guard bitta savolga javob beradi — "**bu request davom etishi mumkinmi?**" — va faqat `true` yoki `false` qaytaradi.

**Before learning:** [[NestJS Request Lifecycle]], [[NestJS Decorators]]
**After learning:** [[Authentication]], [[Authorization]], [[FotSpot Guard Chain]]

> 💡 **Tushunmadingmi? Express'da qo'lda qanday yozilishini ko'r:** [[Express Auth Middleware]]


---

## 1. Definition

```ts
@Injectable()
export class MyGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean | Promise<boolean> {
    return true;   // yoki false / throw
  }
}
```

Guard controller handler'idan **oldin**, pipe'lardan ham **oldin** ishlaydi.

## 2. Why — nega kerak?

Chunki "kirish mumkinmi?" savoli har bir controller ichida takrorlanmasligi kerak.
Guard uni **kesib o'tuvchi (cross-cutting)** qatlamga chiqaradi: bitta `JwtAuthGuard` butun application'ni himoya qiladi.

## 3. Internal mental model

```text
Request
   ↓
Guard.canActivate(context)
   ├── true   → keyingi bosqich (pipe → controller)
   ├── false  → 403 Forbidden
   └── throw  → o'zing bergan exception (401, 429, ...)
```

`ExecutionContext` — guard'ning ko'zi:

```ts
const req = context.switchToHttp().getRequest();
const handler = context.getHandler();   // method metadata
const cls = context.getClass();         // controller metadata
```

Guard qayerda e'lon qilinadi:

```text
@UseGuards(RolesGuard)        → method yoki controller darajasida
APP_GUARD provider            → global, DI bilan ([[NestJS Providers]])
app.useGlobalGuards(...)      → global, DI'siz
```

## 4. FotSpot implementation

FotSpot'da 4 ta global guard, aniq tartibda:

```text
Request
   ↓
ThrottleGuard        → abuse/DDoS'ni eng arzon bosqichda to'xtatadi
   ↓
JwtAuthGuard         → "kim?"  → request.user to'ldiriladi
   ↓
RolesGuard           → "qaysi toifadagi user?"
   ↓
PermissionsGuard     → "aynan shu action'ga ruxsat bormi?"
   ↓
Controller
```

Tartibning **sababi** (yodlash emas, tushunish):

- Throttle birinchi — chunki token tekshirish ham, DB query ham qimmat; hujumni undan oldin to'xtatish kerak.
- Jwt Roles'dan oldin — role'ni bilish uchun avval user'ni bilish kerak.
- Roles Permissions'dan oldin — permission role orqali keladi ([[Roles and Permissions]]).

Batafsil: [[FotSpot Guard Chain]]

## 5. Code example

```ts
// roles.guard.ts
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private readonly reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const required = this.reflector.getAllAndOverride<string[]>(ROLES_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (!required?.length) return true;         // metadata yo'q → cheklov yo'q

    const { user } = context.switchToHttp().getRequest();
    if (!user) throw new UnauthorizedException();

    const ok = user.roles?.some((r: string) => required.includes(r));
    if (!ok) throw new ForbiddenException('Insufficient role');
    return true;
  }
}
```

```ts
// ishlatilishi
@Roles('admin', 'super_admin')
@Delete(':id')
remove(@Param('id') id: string) {}
```

## 6. Common mistakes

- **Guard ichida business logic yozish** → guard faqat ruxsat beradi; qolgani [[NestJS Services]] ishi.
- **Guard ichida DTO'ga ishonish** → pipe'lar guard'dan **keyin** ishlaydi, ya'ni body hali validatsiyadan o'tmagan.
- **`false` qaytarish o'rniga sukut saqlash** → `undefined` ham `false` deb qabul qilinadi, lekin xato sababi ko'rinmaydi. Aniq exception tashla.
- **Global guard'ni `useGlobalGuards` bilan qo'yib, unga service inject qilmoqchi bo'lish** → DI ishlamaydi; `APP_GUARD` ishlat.
- **Metadata yo'qligida `false` qaytarish** → hamma endpoint yopilib qoladi. Odatda `true` qaytarish kerak.

## 7. Practice

1. `@Roles()` decorator + `RolesGuard` ni noldan yoz.
2. Guard'da `console.log` qo'yib, pipe/controller loglaridan oldin chiqishini tasdiqla.
3. Guard'ni `false` qaytaradigan qil — pipe ishlaydimi? Javobni oldindan ayt, keyin tekshir.
4. Ikkita guard qo'y (`@UseGuards(A, B)`) va A `false` qaytarsa B ishlaydimi — tekshir.

## 8. Related concepts

[[NestJS Request Lifecycle]] · [[Middleware vs Guard vs Pipe vs Interceptor vs Filter]] · [[Authentication]] · [[Authorization]] · [[Public Routes Decorator]] · [[FotSpot Guard Chain]] · [[Rate Limiting and Throttling]]
