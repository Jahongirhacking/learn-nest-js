---
tags: [nestjs/core, level/beginner]
level: beginner
topic: NestJS Core
---

# NestJS Providers

> **Bir jumlada:** Provider — DI container boshqaradigan **istalgan** narsa; Service uning eng ko'p uchraydigan turi, xolos.

**Before learning:** [[NestJS Dependency Injection]], [[NestJS Modules]]
**After learning:** [[NestJS Services]], [[Passport and JwtStrategy]], [[Redis]]

> 💡 **Tushunmadingmi? Express'da qo'lda qanday yozilishini ko'r:** [[Express Manual Dependency Wiring]]


---

## 1. Definition

```ts
@Injectable()
export class PlayersService {}
```

va module'da:

```ts
@Module({ providers: [PlayersService] })
```

`providers` massiviga tushgan har bir narsa — provider.

## 2. Why — nega kerak?

Chunki "inject qilinadigan narsa" faqat service emas. Provider bo'lishi mumkin:

```text
Service                → PlayersService
Strategy               → JwtStrategy          ([[Passport and JwtStrategy]])
Guard                  → RolesGuard           ([[NestJS Guards]])
Interceptor / Filter   → LoggingInterceptor
Repository-like class  → PlayerRepository
Factory / helper       → PasswordHasher
Tashqi client          → Redis, S3Client
Konstanta / config     → useValue
```

Ya'ni DI faqat business logic uchun emas — **butun application graph** uchun.

## 3. Internal mental model

Provider e'lon qilishning 4 shakli:

```ts
// 1. Standart (class)
providers: [PlayersService]

// 2. useClass — interfeysga implementatsiya almashtirish
{ provide: StorageService, useClass: R2StorageService }

// 3. useValue — tayyor object / mock / konstanta
{ provide: 'APP_CONFIG', useValue: { pageSize: 20 } }

// 4. useFactory — yaratish uchun logika kerak bo'lganda
{
  provide: 'REDIS_CLIENT',
  useFactory: (config: ConfigService) => new Redis(config.getOrThrow('REDIS_URL')),
  inject: [ConfigService],
}
```

Token tushunchasi: Nest provider'ni **token** bo'yicha topadi. Class'ning o'zi ham token; string/`Symbol` ham token bo'la oladi (u holda `@Inject('TOKEN')` kerak).

## 4. FotSpot implementation

```text
PrismaService        → @Global() PrismaModule ichida provider + export
Redis client         → useFactory bilan, config'dan URL olib
JwtStrategy          → AuthModule providers ichida
RolesGuard           → APP_GUARD token orqali global provider
HttpExceptionFilter  → APP_FILTER token orqali global provider
```

Global guard/filter'ni provider sifatida berish — muhim naqsh, chunki u holda **guard ham DI'dan foydalana oladi**:

```ts
@Module({
  providers: [
    { provide: APP_GUARD, useClass: JwtAuthGuard },
    { provide: APP_GUARD, useClass: RolesGuard },
    { provide: APP_GUARD, useClass: PermissionsGuard },
    { provide: APP_FILTER, useClass: HttpExceptionFilter },
  ],
})
export class AppModule {}
```

Batafsil: [[FotSpot Guard Chain]]

## 5. Code example

```ts
// redis.module.ts
@Global()
@Module({
  providers: [
    {
      provide: 'REDIS_CLIENT',
      useFactory: (config: ConfigService) =>
        new Redis(config.getOrThrow<string>('REDIS_URL')),
      inject: [ConfigService],
    },
    RedisCacheService,
  ],
  exports: ['REDIS_CLIENT', RedisCacheService],
})
export class RedisModule {}

// ishlatish
@Injectable()
export class RedisCacheService {
  constructor(@Inject('REDIS_CLIENT') private readonly redis: Redis) {}
}
```

## 6. Common mistakes

- **String token ishlatib, `@Inject()` yozishni unutish** → Nest tipni topa olmaydi (string type'ning runtime metadata'si yo'q).
- **`useFactory` ichida `inject` ni unutish** → factory parametrlari `undefined` bo'ladi.
- **Guard'ni `@UseGuards(new RolesGuard(reflector))` ko'rinishida qo'lda yaratish** → DI ishlamaydi. `APP_GUARD` yoki `@UseGuards(RolesGuard)` ishlat.
- **Provider'ni bir necha modulda qayta provide qilish** → bir nechta instance ([[NestJS Modules]]).

## 7. Practice

1. `useValue` bilan `APP_CONFIG` provider yarat va service'ga inject qil.
2. `useFactory` bilan config'dan qiymat oladigan provider yoz.
3. `APP_GUARD` orqali global guard qo'sh va u DI orqali boshqa service ola olishini tekshir.
4. `useClass` bilan `StorageService`ning ikkita implementatsiyasini (local va R2) almashtirib ko'r.

## 8. Related concepts

[[NestJS Dependency Injection]] · [[NestJS Modules]] · [[NestJS Services]] · [[NestJS Guards]] · [[Passport and JwtStrategy]] · [[Redis]]
