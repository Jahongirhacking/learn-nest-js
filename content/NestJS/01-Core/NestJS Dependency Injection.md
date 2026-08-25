---
tags: [nestjs/core, level/beginner, fundamental]
level: beginner
topic: NestJS Core
---

# NestJS Dependency Injection

> **Bir jumlada:** sen `new` yozmaysan — class'ga nima kerakligini constructor'da e'lon qilasan, instance'ni Nest'ning DI container'i yaratadi va beradi.

**Before learning:** [[NestJS Modules]], [[TypeScript for NestJS]]
**After learning:** [[NestJS Providers]], [[NestJS Services]], [[Prisma]], [[Testing in NestJS]]

> 💡 **Tushunmadingmi? Express'da qo'lda qanday yozilishini ko'r:** [[Express Manual Dependency Wiring]]


Bu **fundamental note**. NestJS'ning qolgan hamma narsasi shunga suyanadi.

---

## 1. Definition

Oddiy TypeScript:

```ts
const service = new PlayersService(new PrismaService(), new StorageService());
```

NestJS:

```ts
constructor(private readonly playersService: PlayersService) {}
```

Nest constructor parametrining **tipiga** qarab, kerakli instance'ni topadi va uzatadi.

## 2. Why — nega kerak?

Uchta sabab:

1. **Dependency graph'ni framework boshqaradi.** `PlayersService` ichida `PrismaService` kerak bo'lsa, uni kim yaratishini o'ylamaysan.
2. **Singleton lifecycle.** Bitta `PrismaService` instance butun app bo'ylab ishlatiladi — 100 ta database connection ochilmaydi.
3. **Testability.** Testda real `PrismaService` o'rniga mock berish bir qator kod:

```ts
const module = await Test.createTestingModule({
  providers: [
    PlayersService,
    { provide: PrismaService, useValue: mockPrisma },   // ← almashtirildi
  ],
}).compile();
```

`new` bilan yozilgan kodda buni qilish uchun butun classni qayta yozish kerak bo'lardi.

## 3. Internal mental model

```text
@Injectable()  → "meni container boshqarishi mumkin"
providers[]    → "men shu modulda mavjudman"
exports[]      → "meni boshqa modullar ham olishi mumkin"
constructor    → "menga shular kerak"
```

Nest ishga tushganda:

```text
1. Metadata o'qiladi (decoratorlardan)
2. Dependency graph quriladi
3. Topologik tartibda instance yaratiladi
   (avval PrismaService, keyin PlayersService, keyin PlayersController)
4. Har biri cache'lanadi (singleton)
```

Ya'ni DI — **runtime'da emas, startup'da** hal qilinadi. Shuning uchun noto'g'ri konfiguratsiya app ishga tushganda darhol xato beradi, request kelganda emas.

## 4. FotSpot implementation

```text
PlayersController
       ↓
PlayersService
       ↓
PrismaService
       ↓
PostgreSQL
```

Controller **hech qachon** Prisma bilan bevosita ishlamaydi. Bu qoida DI orqali tabiiy ravishda saqlanadi: controller constructor'ida `PrismaService` yozilmaydi.

Murakkabroq feature:

```text
TrialsService
   ├── PrismaService            (infrastructure)
   ├── RecommendationsService   (domain)
   ├── NotificationsService     (domain)
   └── StorageService           (infrastructure)
```

Hammasi constructor injection orqali keladi.

## 5. Code example

```ts
@Injectable()
export class PlayersService {
  constructor(
    private readonly prisma: PrismaService,
    private readonly storage: StorageService,
  ) {}

  async findOne(id: string) {
    const player = await this.prisma.playerProfile.findUnique({ where: { id } });
    if (!player) throw new NotFoundException('Player not found');
    return player;
  }
}

@Controller('players')
export class PlayersController {
  constructor(private readonly playersService: PlayersService) {}

  @Get(':id')
  findOne(@Param('id') id: string) {
    return this.playersService.findOne(id);
  }
}
```

Custom provider (faqat Service emas):

```ts
{
  provide: 'REDIS_CLIENT',
  useFactory: (config: ConfigService) => new Redis(config.get('REDIS_URL')),
  inject: [ConfigService],
}

// ishlatish
constructor(@Inject('REDIS_CLIENT') private readonly redis: Redis) {}
```

## 6. Common mistakes

- **`@Injectable()` ni unutish** → Nest metadata topa olmaydi.
- **Service ichida `new OtherService()` yozish** → DI chetlab o'tiladi, testda mock qilib bo'lmaydi, singleton buziladi.
- **Provider'ni `providers`ga qo'shmaslik yoki `exports` qilmaslik** → `Nest can't resolve dependencies` xatosi. Xato matnidagi `(?)` — aynan topilmagan parametr.
- **Circular dependency** → `forwardRef()` bilan yopish mumkin, lekin bu ko'pincha noto'g'ri boundary belgisi ([[NestJS Modules]]).
- **Constructor ichida og'ir ish qilish** (DB query, HTTP call) → startup sekinlashadi; buning uchun `OnModuleInit` bor.

## 7. Practice

1. `PlayersService`ni `PrismaService`ga bog'la, lekin `PrismaModule`ni import **qilma**. Xatoni to'liq o'qi va nima yetishmayotganini ayt.
2. Service ichiga `console.log('created')` qo'y va ikkita controller'dan foydalanib ko'r — necha marta chiqadi? Nega?
3. `Test.createTestingModule` bilan `PrismaService` o'rniga mock ber va service'ni testda ishlatib ko'r.
4. `useFactory` bilan custom provider yoz (masalan config'dan URL oladigan client).

## 8. Related concepts

[[NestJS Providers]] · [[NestJS Modules]] · [[NestJS Services]] · [[NestJS Decorators]] · [[Testing in NestJS]] · [[Prisma]]
