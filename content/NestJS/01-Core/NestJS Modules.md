---
tags: [nestjs/core, level/beginner]
level: beginner
topic: NestJS Core
---

# NestJS Modules

> **Bir jumlada:** Module — application'ning logical chegarasi (boundary): bir feature'ga tegishli controller, provider va dependency'larni bitta joyga yig'adi.

**Before learning:** [[NestJS Overview]]
**After learning:** [[NestJS Controllers]], [[NestJS Providers]], [[NestJS Dependency Injection]]

> 💡 **Tushunmadingmi? Express'da qo'lda qanday yozilishini ko'r:** [[Express Router]]


---

## 1. Definition

```ts
@Module({
  imports: [],      // boshqa modullar
  controllers: [],  // HTTP endpointlar
  providers: [],    // injectable classlar
  exports: [],      // tashqariga ochiladigan providerlar
})
export class PlayersModule {}
```

Application root module'dan boshlanadi — `AppModule`, va u boshqa modullarni `imports` orqali yig'adi.

## 2. Why — nega kerak?

Module ikki narsani beradi:

1. **Tashkiliy chegara** — "player bilan bog'liq hamma narsa shu yerda".
2. **DI scope** — provider faqat o'z moduli ichida va `exports` qilingan joyda ko'rinadi.

Ikkinchisi muhimroq: module boundary — bu **kim kimga bog'lanishi mumkin** degan qoidadir.

## 3. Internal mental model

To'rt maydonni shunday eslab qol:

```text
imports      → "menga kerak bo'lgan boshqa modullar"
controllers  → "mening HTTP kirish nuqtalarim"
providers    → "menga tegishli injectable narsalar"
exports      → "mening providerlarimdan boshqalarga ruxsat berilganlari"
```

Muhim qoida:

```text
Provider exports qilinmasa —
uni import qilgan module ham ko'ra olmaydi.
```

`@Global()` bu qoidani chetlab o'tadi (masalan `PrismaModule`), lekin uni tez-tez ishlatish — architecture hidining belgisi.

## 4. FotSpot implementation

```ts
@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true }),
    PrismaModule,
    RedisModule,
    AuthModule,
    UsersModule,
    PlayersModule,
    CoachesModule,
    AcademiesModule,
    MediaModule,
    TrialsModule,
    RecommendationsModule,
    NotificationsModule,
    ModerationModule,
  ],
})
export class AppModule {}
```

Ko'rinib turibdiki, `AppModule` o'zi hech qanday business logic saqlamaydi — u faqat **kompozitsiya**.

Feature moduli esa:

```ts
@Module({
  imports: [NotificationsModule],           // boshqa feature'ga bog'lanish
  controllers: [TrialsController],
  providers: [TrialsService],
  exports: [TrialsService],                 // boshqa modullar foydalanishi uchun
})
export class TrialsModule {}
```

Batafsil: [[FotSpot Feature Anatomy]]

## 5. Code example

```ts
// players.module.ts
@Module({
  controllers: [PlayersController],
  providers: [PlayersService],
  exports: [PlayersService],
})
export class PlayersModule {}

// trials.module.ts — PlayersService'ni ishlatmoqchi
@Module({
  imports: [PlayersModule],        // ← exports qilingani uchun ishlaydi
  controllers: [TrialsController],
  providers: [TrialsService],
})
export class TrialsModule {}
```

## 6. Common mistakes

- **`exports` ni unutish** → `Nest can't resolve dependencies of TrialsService (?)` xatosi. Eng ko'p uchraydigan xato.
- **Provider'ni ikkita modulda `providers` ga yozish** → ikkita alohida instance paydo bo'ladi (state yo'qoladi). To'g'risi: bir joyda provide qil, boshqasida import.
- **Hamma narsani `AppModule`ga tiqish** → modular architecture yo'qoladi.
- **Circular import** (`A imports B`, `B imports A`) → `forwardRef()` kerak bo'ladi, lekin avval dizaynni qayta ko'rib chiq.
- **`@Global()` ni qulaylik uchun ishlatish** → dependency graph ko'rinmas bo'lib qoladi.

## 7. Practice

1. `players` va `trials` modullarini yarat.
2. `TrialsService` ichida `PlayersService`ni ishlatishga urin — `exports` **yozmasdan**. Xatoni o'qi.
3. Endi `exports` qo'sh va ishlaganini ko'r.
4. `PlayersService`ni ikkala modulning `providers`iga yoz va nima buzilishini kuzat (masalan, service ichidagi counter).

## 8. Related concepts

[[NestJS Dependency Injection]] · [[NestJS Providers]] · [[NestJS Controllers]] · [[FotSpot Feature Anatomy]] · [[Domain vs Infrastructure]]
