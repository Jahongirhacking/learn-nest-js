---
tags: [nestjs/core, level/beginner]
level: beginner
topic: NestJS Core
---

# NestJS Services

> **Bir jumlada:** Service — "bu request kelganda business rule bo'yicha nima qilish kerak?" degan savolga javob beruvchi qatlam.

**Before learning:** [[NestJS Controllers]], [[NestJS Dependency Injection]]
**After learning:** [[Prisma]], [[Database Transactions]], [[Domain vs Infrastructure]]

> 💡 **Tushunmadingmi? Express'da qo'lda qanday yozilishini ko'r:** [[Express Service Layer]]


---

## 1. Definition

```ts
@Injectable()
export class PlayersService {
  constructor(private readonly prisma: PrismaService) {}
}
```

Service — `@Injectable()` bilan belgilangan, business logic saqlaydigan provider.

## 2. Why — nega kerak?

Vazifalar bo'linishi aniq:

```text
Controller → "HTTP request keldi."
Service    → "Bu holatda business rule nima deydi?"
Prisma     → "Ma'lumotni qanday saqlayman/o'qiyman?"
```

Service HTTP'ni bilmagani uchun uni **qayta ishlatish mumkin**: xuddi shu `NotificationsService`ni controller ham, [[BullMQ]] worker ham, [[WebSockets in NestJS]] gateway ham chaqiradi.

## 3. Internal mental model

Service ichida bo'lishi kerak:

```text
✓ business qoidalar ("18 yoshdan kichik o'yinchi trial'ga yozila olmaydi")
✓ orkestratsiya (bir nechta service/infra chaqiruvi)
✓ domain xatolari (NotFoundException, ForbiddenException)
✓ transaction chegarasi
```

Bo'lmasligi kerak:

```text
✗ HTTP status code hisoblash
✗ req/res object bilan ishlash
✗ SQL/Prisma detallarini controller'ga qaytarish
```

## 4. FotSpot implementation

```text
PlayersController
       ↓
PlayersService
       ├── PrismaService            → database
       ├── StorageService           → R2 presigned URL
       └── RecommendationsService   → boshqa domain service
```

Trial feature'da orkestratsiya yaqqol ko'rinadi:

```text
TrialsService.finishTrial()
   ├── Prisma: trial natijasini yozish
   ├── RecommendationsService: statusni yangilash
   ├── NotificationsService: queue'ga job qo'shish
   └── hammasi bitta transaction ichida  → [[Database Transactions]]
```

## 5. Code example

```ts
@Injectable()
export class TrialsService {
  constructor(
    private readonly prisma: PrismaService,
    private readonly notifications: NotificationsService,
  ) {}

  async create(dto: CreateTrialDto, userId: string) {
    const academy = await this.prisma.academy.findUnique({
      where: { id: dto.academyId },
    });
    if (!academy) throw new NotFoundException('Academy not found');
    if (academy.ownerId !== userId) {
      throw new ForbiddenException('Not your academy');
    }

    const trial = await this.prisma.trial.create({ data: { ...dto, createdBy: userId } });

    await this.notifications.enqueueTrialCreated(trial.id);   // background
    return trial;
  }
}
```

E'tibor ber: `throw new ForbiddenException()` — service HTTP status'ni bilmaydi, u **domain xatosini** aytadi; uni response'ga [[NestJS Exception Filters]] aylantiradi.

## 6. Common mistakes

- **Service'ni "Prisma wrapper" qilib qo'yish** (`findAll` faqat `prisma.x.findMany()`) → qatlam qiymat bermaydi. Business qoidalar service'da bo'lishi kerak.
- **God service** — 1500 qatorli `UsersService`. Feature bo'yicha bo'l.
- **Service ichida `res.json()`** → HTTP service'ga sizib kirdi.
- **Bir necha yozuv operatsiyasini transactionsiz bajarish** → yarim bajarilgan holat ([[Database Transactions]]).
- **Uzoq ishni request ichida bajarish** (email, video processing) → [[Queue vs Synchronous Request]].

## 7. Practice

1. `PlayersService`ga `findOne` yoz: topilmasa `NotFoundException`.
2. Unga bitta real business qoida qo'sh (masalan, faqat `ACTIVE` profillar public ko'rinadi).
3. Controller'dagi barcha `if`larni service'ga ko'chir.
4. Xuddi shu service methodini test'da mock Prisma bilan chaqir ([[Testing in NestJS]]).

## 8. Related concepts

[[NestJS Controllers]] · [[NestJS Providers]] · [[NestJS Dependency Injection]] · [[Prisma]] · [[Database Transactions]] · [[Domain vs Infrastructure]]
