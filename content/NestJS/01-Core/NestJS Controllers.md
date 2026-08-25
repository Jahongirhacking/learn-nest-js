---
tags: [nestjs/core, level/beginner]
level: beginner
topic: NestJS Core
---

# NestJS Controllers

> **Bir jumlada:** Controller — HTTP dunyosi bilan business logic o'rtasidagi yupqa chegara: requestni qabul qiladi, service'ni chaqiradi, natijani qaytaradi.

**Before learning:** [[NestJS Modules]], [[NestJS Decorators]]
**After learning:** [[NestJS Services]], [[NestJS DTO]], [[NestJS Guards]]

> 💡 **Tushunmadingmi? Express'da qo'lda qanday yozilishini ko'r:** [[Express Route Handlers]]


---

## 1. Definition

```ts
@Controller('players')      // → /players
export class PlayersController {
  @Get(':id')               // → GET /players/:id
  findOne(@Param('id') id: string) {}
}
```

`@Controller()` decorator classni route handler sifatida ro'yxatdan o'tkazadi.

## 2. Why — nega kerak?

Controller bo'lmasa, HTTP tafsilotlari (status code, header, query parsing) business logic ichiga kirib ketadi.
Controller ularni **bir joyda ushlab turadi**, service esa HTTP'dan bexabar qoladi — shuning uchun xuddi shu service'ni keyinchalik CLI, cron yoki [[BullMQ]] worker ham chaqira oladi.

## 3. Internal mental model

Controller faqat 5 ta ish qiladi:

```text
1. requestni qabul qiladi
2. DTO orqali inputni oladi          → [[NestJS DTO]]
3. authentication contextni oladi     → request.user
4. service'ni chaqiradi               → [[NestJS Services]]
5. response qaytaradi
```

Yomon:

```text
Controller
 ├── database query
 ├── pricing logic
 ├── permission logic
 └── calculations
```

Yaxshi:

```text
Controller → Service → Database / external service
```

Qoida: **controller ichida `if` deyarli bo'lmasligi kerak.**

## 4. FotSpot implementation

```text
GET  /api/v1/players
POST /api/v1/players
GET  /api/v1/players/:id
```

`api/v1` prefiksi `main.ts` dagi `setGlobalPrefix` dan keladi ([[NestJS Application Bootstrap]]).

Controller darajasida authorization metadata beriladi:

```ts
@Controller('trials')
export class TrialsController {
  @Roles('coach', 'admin')                 // ← RolesGuard o'qiydi
  @Permissions('trial:create')             // ← PermissionsGuard o'qiydi
  @Post()
  create(@Body() dto: CreateTrialDto, @CurrentUser() user: JwtUser) {
    return this.trialsService.create(dto, user.id);
  }
}
```

Bu yerda `@Roles`, `@Permissions`, `@CurrentUser` — [[NestJS Custom Decorators]].

## 5. Code example

```ts
@ApiTags('players')
@Controller('players')
export class PlayersController {
  constructor(private readonly playersService: PlayersService) {}

  @Public()                                  // [[Public Routes Decorator]]
  @Get()
  findAll(@Query() query: ListPlayersDto) {
    return this.playersService.findAll(query);
  }

  @Get(':id')
  findOne(@Param('id') id: string) {
    return this.playersService.findOne(id);
  }

  @Post()
  @HttpCode(201)
  create(@Body() dto: CreatePlayerDto, @CurrentUser() user: JwtUser) {
    return this.playersService.create(dto, user.id);
  }
}
```

Parameter decoratorlar:

```text
@Body()    → request body       @Param()  → URL parametri
@Query()   → query string       @Headers() → header
@Req()     → raw request (kerak bo'lmasa ishlatma)
```

## 6. Common mistakes

- **Controller ichida Prisma chaqirish** → boundary buziladi ([[NestJS Dependency Injection]]).
- **`@Req()` ni doim ishlatish** → controller Express'ga bog'lanib qoladi va test qilish qiyinlashadi.
- **DTO o'rniga `any` yoki inline object** → validation yo'q, security teshigi ([[NestJS ValidationPipe]]).
- **Xatoni qo'lda `res.status(404).json(...)` qilib qaytarish** → [[NestJS Exception Filters]] chetlab o'tiladi, response formati buziladi. To'g'risi: `throw new NotFoundException()`.
- **Route tartibi**: `@Get(':id')` ni `@Get('me')` dan oldin yozsang, `/players/me` `:id = "me"` bo'lib ketadi.

## 7. Practice

1. `players` controller yarat: `GET /players`, `GET /players/:id`, `POST /players`.
2. `@Get('me')` va `@Get(':id')` ni ataylab noto'g'ri tartibda yozib, buzilishni o'z ko'zing bilan ko'r.
3. Controller ichidagi bitta `if`ni service'ga ko'chir va farqni ayt.
4. `throw new NotFoundException('...')` qo'sh va response JSON'ini kuzat.

## 8. Related concepts

[[NestJS Services]] · [[NestJS DTO]] · [[NestJS Guards]] · [[NestJS Custom Decorators]] · [[NestJS Request Lifecycle]] · [[Swagger and OpenAPI]]
