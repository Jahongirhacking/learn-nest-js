---
tags: [production, level/intermediate]
level: intermediate
topic: Production
---

# Swagger and OpenAPI

> **Bir jumlada:** Swagger — kodning o'zidan generatsiya qilinadigan, ya'ni **eskirmaydigan** API dokumentatsiyasi.

**Before learning:** [[NestJS DTO]], [[NestJS Controllers]]
**After learning:** [[Testing in NestJS]], frontend integratsiyasi

---

## 1. Definition

```text
@nestjs/swagger
```

```text
Controller + DTO
      ↓
Decorators (@ApiTags, @ApiProperty)
      ↓
Swagger metadata
      ↓
OpenAPI spec
      ↓
Swagger UI  →  /docs
```

## 2. Why — nega kerak?

Qo'lda yozilgan dokumentatsiya **har doim** eskiradi. Kodga bog'langan dokumentatsiya esa yo'q.

Qo'shimcha foyda:

```text
✓ frontend uchun tirik contract
✓ OpenAPI spec'dan client SDK generatsiya qilish
✓ endpointlarni brauzerdan sinab ko'rish
✓ yangi dasturchi uchun kirish nuqtasi
```

## 3. Internal mental model

Swagger ma'lumotni ikki joydan oladi:

```text
1. TypeScript tiplari + DTO decoratorlari (avtomatik)
2. @Api* decoratorlari (qo'lda boyitish)
```

CLI plugin (`@nestjs/swagger` plugin) ko'p ishni avtomatlashtiradi, lekin quyidagilarni qo'lda yozish kerak:

```text
@ApiOperation   → endpoint nima qiladi
@ApiResponse    → mumkin bo'lgan javoblar
@ApiBearerAuth  → token kerakligi
@ApiPropertyOptional / example → tushunarli misollar
```

## 4. FotSpot implementation

`/docs` da interaktiv API reference. Har bir feature `@ApiTags` bilan guruhlanadi:

```text
auth · users · players · coaches · academies · trials · media · moderation
```

Muhim production savoli: **dokumentatsiya kimga ochiq?**

```text
public API   → ochiq bo'lishi mumkin
internal API → production'da yopish yoki himoyalash kerak
```

## 5. Code example

```ts
// main.ts
const config = new DocumentBuilder()
  .setTitle('FotSpot API')
  .setDescription('Football scouting platform API')
  .setVersion('1.0')
  .addBearerAuth()
  .addTag('players', 'Player profiles')
  .build();

const document = SwaggerModule.createDocument(app, config);
SwaggerModule.setup('docs', app, document, {
  swaggerOptions: { persistAuthorization: true },
});
```

```ts
@ApiTags('players')
@ApiBearerAuth()
@Controller('players')
export class PlayersController {
  @ApiOperation({ summary: 'Get player by id' })
  @ApiOkResponse({ type: PlayerResponseDto })
  @ApiNotFoundResponse({ description: 'Player not found' })
  @Get(':id')
  findOne(@Param('id') id: string) {}
}
```

```ts
export class CreatePlayerDto {
  @ApiProperty({ example: 'Ali', minLength: 2 })
  @IsString()
  @Length(2, 50)
  firstName: string;
}
```

## 6. Common mistakes

- **Production'da `/docs` ni himoyasiz qoldirish** → butun ichki API xaritasi ochiq.
- **`@ApiProperty` yozmaslik** → schema bo'sh ko'rinadi va frontend uchun foydasiz.
- **Response DTO'ni ko'rsatmaslik** → nima qaytishini hech kim bilmaydi.
- **`addBearerAuth()` ni unutish** → "Try it out" 401 qaytaradi va chalkashlik tug'diradi.
- **Interceptor response'ni o'zgartirgani holda schema'ni yangilamaslik** ([[NestJS Interceptors]]).
- **Dokumentatsiyada haqiqiy ma'lumot/misollarni ishlatish** → maxfiy ma'lumot sizishi.

## 7. Practice

1. Swagger'ni ulab, `/docs` ni och.
2. Bitta endpointga `@ApiOperation` va `@ApiOkResponse` qo'sh.
3. `addBearerAuth()` bilan token kiritib, himoyalangan endpointni sinab ko'r.
4. OpenAPI JSON'ni yuklab olib, undan client generatsiya qilishga urin.
5. Production'da `/docs`ni basic auth bilan yopadigan yechim yoz.

## 8. Related concepts

[[NestJS DTO]] · [[NestJS Controllers]] · [[NestJS Decorators]] · [[NestJS Application Bootstrap]] · [[Testing in NestJS]]
