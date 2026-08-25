---
tags: [auth, level/intermediate]
level: intermediate
topic: Authentication
---

# Passport and JwtStrategy

> **Bir jumlada:** **Strategy** tokenni qanday tekshirishni biladi, **Guard** esa qaysi endpointda tekshirish kerakligini hal qiladi — bu ikkisini chalkashtirmaslik kerak.

**Before learning:** [[JWT Authentication]], [[NestJS Guards]], [[NestJS Providers]]
**After learning:** [[Session Architecture]], [[Public Routes Decorator]]

> 💡 **Tushunmadingmi? Express'da qo'lda qanday yozilishini ko'r:** [[Express Passport]]


---

## 1. Definition

```text
PassportModule   → integratsiya qatlami
JwtModule        → token imzolash/tekshirish
passport-jwt     → strategiya implementatsiyasi
JwtStrategy      → validate() — payload'dan user yasaydi
JwtAuthGuard     → AuthGuard('jwt') — endpointga biriktiriladi
```

## 2. Why — nega kerak?

Passport turli authentication usullarini (jwt, local, google, telegram) **bir xil interfeys** ostiga oladi:

```text
Strategy → validate() → natija request.user ga yoziladi
```

Ya'ni `GoogleStrategy` qo'shsang ham, controller kodi o'zgarmaydi.

## 3. Internal mental model

To'liq zanjir:

```text
Authorization: Bearer <token>
        ↓
JwtAuthGuard (AuthGuard('jwt'))
        ↓
passport-jwt: tokenni headerdan chiqaradi
        ↓
imzo + exp tekshiriladi (secret bilan)
        ↓
JwtStrategy.validate(payload)
        ↓
qaytgan qiymat → request.user
        ↓
Controller / @CurrentUser()
```

Aynan farq:

```text
Strategy → "Bu token haqiqiymi va uning egasi kim?"
Guard    → "Bu endpointda umuman token talab qilinadimi?"
```

Shuning uchun [[Public Routes Decorator]] **guard**da amalga oshiriladi, strategiyada emas.

## 4. FotSpot implementation

```text
AuthModule
 ├── PassportModule
 ├── JwtModule.registerAsync(...)     ← secret ConfigService'dan
 ├── JwtStrategy      (provider)
 └── JwtAuthGuard     (APP_GUARD sifatida global)
```

`validate()` ichida FotSpot faqat payloadga ishonmaydi — user hali ham mavjud va faol ekanini tekshirish mumkin (masalan bloklangan userni darhol chiqarib yuborish uchun).

## 5. Code example

```ts
// jwt.strategy.ts
@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy, 'jwt') {
  constructor(
    config: ConfigService,
    private readonly prisma: PrismaService,
  ) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: config.getOrThrow<string>('JWT_SECRET'),
    });
  }

  async validate(payload: { sub: string }) {
    const user = await this.prisma.user.findUnique({
      where: { id: payload.sub },
      select: { id: true, status: true, roles: { select: { role: true } } },
    });
    if (!user || user.status !== 'ACTIVE') throw new UnauthorizedException();

    return {                       // ← bu obyekt request.user bo'ladi
      id: user.id,
      roles: user.roles.map(r => r.role.name),
    };
  }
}
```

```ts
// jwt-auth.guard.ts — @Public() qo'llab-quvvatlashi bilan
@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {
  constructor(private readonly reflector: Reflector) { super(); }

  canActivate(ctx: ExecutionContext) {
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      ctx.getHandler(), ctx.getClass(),
    ]);
    if (isPublic) {
      super.canActivate(ctx);      // token bo'lsa user'ni aniqlashga harakat
      return true;
    }
    return super.canActivate(ctx);
  }

  handleRequest(err: any, user: any, _info: any, ctx: ExecutionContext) {
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      ctx.getHandler(), ctx.getClass(),
    ]);
    if (isPublic) return user ?? null;          // guest ham o'tadi
    if (err || !user) throw new UnauthorizedException();
    return user;
  }
}
```

## 6. Common mistakes

- **`ignoreExpiration: true`** → muddati o'tgan token qabul qilinaveradi.
- **`validate()` da faqat payload'ni qaytarish** → o'chirilgan yoki bloklangan user token muddatigacha ishlayveradi.
- **`validate()` ichida og'ir query** → har bir requestda DB yuklamasi; kerak bo'lsa [[Redis Cache]].
- **Strategy'ni `providers`ga qo'shishni unutish** → `Unknown authentication strategy "jwt"`.
- **Guard va Strategy vazifasini chalkashtirish** → `@Public()` mantiqini strategiyaga yozishga urinish.

## 7. Practice

1. `JwtStrategy` yozib, `validate()` da `console.log(payload)` qil.
2. `validate()` dan qaytgan obyekt aynan `request.user` ekanini `@CurrentUser()` orqali tasdiqla.
3. `ignoreExpiration: true` qilib, eskirgan token bilan sinab ko'r (keyin qaytar).
4. Userni `status: 'BLOCKED'` qilib, keyingi request darhol 401 bo'lishini ta'minla.

## 8. Related concepts

[[JWT Authentication]] · [[NestJS Guards]] · [[Public Routes Decorator]] · [[Authentication]] · [[Session Architecture]] · [[NestJS Providers]]
