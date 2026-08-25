---
tags: [fotspot, level/intermediate]
level: intermediate
topic: FotSpot
---

# FotSpot Feature Anatomy

> **Bir jumlada:** FotSpot'dagi har bir feature bitta takrorlanadigan naqshga ega — bir marta tushunsang, qolgan hammasini o'qiy olasan.

**Before learning:** [[NestJS Modules]], [[NestJS Controllers]], [[NestJS Services]]
**After learning:** [[Build a Feature in 10 Steps]], [[Domain vs Infrastructure]]

---

## 1. Definition

```text
trials/
 ├── trials.module.ts
 ├── trials.controller.ts
 ├── trials.service.ts
 ├── dto/
 │    ├── create-trial.dto.ts
 │    ├── update-trial.dto.ts
 │    └── list-trials.dto.ts
 └── (kerak bo'lsa) guards/ decorators/ processors/ helpers/
```

## 2. Why — nega bir xil naqsh?

```text
✓ yangi dasturchi faylni qidirmaydi — bilib turadi
✓ code review tez: naqshdan chetlanish darhol ko'rinadi
✓ yangi feature yozish mexanik ish bo'lib qoladi
```

Naqsh — bu cheklov emas, **tezlik**.

## 3. Internal mental model

Feature ichidagi bog'lanishlar:

```text
HTTP
 ↓
TrialsController          ← yupqa: DTO oladi, service chaqiradi
 ↓
TrialsService             ← business logic
 ├── PrismaService                  (infrastructure)
 ├── RecommendationsService         (boshqa domain)
 ├── NotificationsService           (boshqa domain, queue orqali)
 └── user context (request.user)
```

Feature o'sgani sari unga qo'shiladi:

```text
guards/        → featurega xos ruxsat qoidalari
decorators/    → qulaylik decoratorlari
processors/    → BullMQ worker'lari  ([[BullMQ]])
events/        → domain hodisalari
helpers/       → sof funksiyalar
```

Qoida: **avval oddiy struktura, ehtiyoj paydo bo'lganda kengaytir.**

## 4. Modullar orasidagi bog'lanish

```ts
// trials.module.ts
@Module({
  imports: [RecommendationsModule, NotificationsModule],
  controllers: [TrialsController],
  providers: [TrialsService],
  exports: [TrialsService],
})
export class TrialsModule {}
```

Bu yerda ikki muhim qaror:

```text
imports  → bu feature qaysi boshqa featurelarga bog'liq
exports  → bu feature'ni kim ishlatishi mumkin
```

Agar `imports` ro'yxati juda uzayib ketsa — bu boundary noto'g'ri chizilganining belgisi ([[NestJS Modules]]).

## 5. Code example

```ts
// trials.controller.ts — yupqa qatlam
@ApiTags('trials')
@Controller('trials')
export class TrialsController {
  constructor(private readonly trials: TrialsService) {}

  @Public()
  @Get()
  list(@Query() query: ListTrialsDto) {
    return this.trials.list(query);
  }

  @Roles('coach', 'academy_manager')
  @Permissions('trial:create')
  @Post()
  create(@Body() dto: CreateTrialDto, @CurrentUser('id') userId: string) {
    return this.trials.create(dto, userId);
  }

  @Roles('coach', 'academy_manager')
  @Patch(':id/finish')
  finish(@Param('id') id: string, @Body() dto: TrialResultDto, @CurrentUser('id') userId: string) {
    return this.trials.finish(id, dto, userId);
  }
}
```

Controller'da business qaror yo'q — faqat marshrutlash, ruxsat metadata'si va DTO.

## 6. Common mistakes

- **Controller'ni "qalinlashtirish"** → `if`lar va query'lar to'planadi ([[NestJS Controllers]]).
- **Har bir feature'ni boshqacha strukturada yozish** → o'qish qiyinlashadi.
- **Ikki feature bir-birini import qilishi (circular)** → uchinchi umumiy modulga chiqar.
- **DTO'larni feature tashqarisida umumiy papkaga to'plash** → boundary yo'qoladi.
- **`exports` ni ochib qo'yish "kelajakda kerak bo'lar"** → keraksiz bog'liqliklar paydo bo'ladi.

## 7. Practice

1. `trials` featureni to'liq strukturada yozib chiq.
2. Uni `notifications` moduliga bog'lab, `exports`siz sinab ko'r — xatoni o'qi.
3. Controller'dagi barcha business qarorni service'ga ko'chir.
4. Featurega BullMQ processor qo'shib, papkasini `processors/` ichiga joylashtir.
5. Xuddi shu naqsh bilan yangi feature (`academies`) yoz — [[Build a Feature in 10 Steps]].

## 8. Related concepts

[[NestJS Modules]] · [[NestJS Controllers]] · [[NestJS Services]] · [[Build a Feature in 10 Steps]] · [[Domain vs Infrastructure]] · [[FotSpot Architecture Overview]]
