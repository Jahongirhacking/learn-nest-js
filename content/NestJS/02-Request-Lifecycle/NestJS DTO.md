---
tags: [nestjs/lifecycle, level/beginner]
level: beginner
topic: Request Lifecycle
---

# NestJS DTO

> **Bir jumlada:** DTO — API qabul qiladigan inputning **runtime'da mavjud** shakli; TypeScript type'idan farqli o'laroq, u haqiqatan ham tekshiriladi.

**Before learning:** [[TypeScript for NestJS]], [[NestJS Controllers]]
**After learning:** [[NestJS ValidationPipe]], [[Swagger and OpenAPI]]

> 💡 **Tushunmadingmi? Express'da qo'lda qanday yozilishini ko'r:** [[Express Request Validation]]


---

## 1. Definition

```ts
export class CreatePlayerDto {
  @IsString()
  @Length(2, 50)
  firstName: string;

  @IsString()
  @Length(2, 50)
  lastName: string;

  @IsDateString()
  birthDate: string;

  @IsOptional()
  @IsEnum(Position)
  position?: Position;
}
```

DTO = **class** (interface emas!) + `class-validator` decoratorlari.

## 2. Why — nega kerak?

Chunki bu:

```ts
interface CreatePlayerDto { firstName: string }
```

compile'dan keyin **butunlay yo'qoladi**. Runtime'da `firstName` `123` ham, `null` ham, `{}` ham bo'lishi mumkin.

DTO uchta narsani beradi:

```text
1. Runtime validation    → noto'g'ri input 400 bilan qaytadi
2. API contract          → Swagger schema shundan generatsiya qilinadi
3. Security boundary     → ortiqcha maydonlar kesiladi (mass assignment himoyasi)
```

## 3. Internal mental model

```text
JSON body
   ↓
ValidationPipe
   ├── class-transformer → plain object'ni DTO class instance'iga aylantiradi
   └── class-validator   → decorator qoidalarini tekshiradi
   ↓
Controller (@Body() dto: CreatePlayerDto)
```

DTO turlari:

```text
Create DTO   → majburiy maydonlar
Update DTO   → PartialType(CreateDto) — hammasi optional
Query DTO    → pagination, filter, sort
Response DTO → tashqariga chiqadigan shakl (ixtiyoriy, lekin foydali)
```

## 4. FotSpot implementation

```text
players/dto/
  create-player.dto.ts
  update-player.dto.ts
  list-players.dto.ts
```

Muhim naqsh — **input DTO hech qachon xavfli maydonni qabul qilmaydi**:

```text
✗ CreateUserDto ichida roles, isAdmin, permissions
✓ role faqat serverda, [[RBAC]] qoidalari bo'yicha beriladi
```

Aynan shu sabab `forbidNonWhitelisted: true` FotSpot'da yoqilgan ([[NestJS ValidationPipe]]).

## 5. Code example

```ts
// create-player.dto.ts
export class CreatePlayerDto {
  @ApiProperty({ example: 'Ali' })
  @IsString()
  @Length(2, 50)
  firstName: string;

  @ApiProperty({ example: '2008-05-14' })
  @IsDateString()
  birthDate: string;

  @ApiPropertyOptional({ enum: Position })
  @IsOptional()
  @IsEnum(Position)
  position?: Position;
}

// update-player.dto.ts
export class UpdatePlayerDto extends PartialType(CreatePlayerDto) {}

// list-players.dto.ts — query params
export class ListPlayersDto {
  @IsOptional()
  @Type(() => Number)          // ← transform: 'page=2' → 2
  @IsInt()
  @Min(1)
  page = 1;

  @IsOptional()
  @IsString()
  search?: string;
}
```

## 6. Common mistakes

- **Interface ishlatish** → runtime'da mavjud emas, validation umuman ishlamaydi. DTO **class** bo'lishi shart.
- **Validator decoratorini unutish** → maydon hech qachon tekshirilmaydi va `whitelist: true` bo'lsa **jimgina o'chiriladi** (eng chalkash xato).
- **Query DTO'da `@Type(() => Number)` ni unutish** → `page` string bo'lib qoladi.
- **Nested obyektda `@ValidateNested()` + `@Type()` ni yozmaslik** → ichki obyekt tekshirilmaydi.
- **Entity/Prisma modelni DTO sifatida ishlatish** → database sxemasi API contract'ga aylanib qoladi.
- **Response'da DTO o'rniga to'g'ridan-to'g'ri model qaytarish** → `passwordHash` sizib chiqishi mumkin.

## 7. Practice

1. `CreatePlayerDto` yozib, `POST /players` ga ula.
2. `{"firstName": 123}` yubor — status va xato matnini ko'r.
3. `{"firstName":"Ali","isAdmin":true}` yubor — `forbidNonWhitelisted` bilan va bo'lmagan holda farqni yoz.
4. Nested DTO yasab, `@ValidateNested()` siz sinab ko'r — ichki obyekt tekshiriladimi?
5. `UpdatePlayerDto`ni `PartialType` bilan yasab, PATCH endpoint qil.

## 8. Related concepts

[[NestJS ValidationPipe]] · [[NestJS Pipes]] · [[NestJS Controllers]] · [[Swagger and OpenAPI]] · [[TypeScript for NestJS]] · [[Authorization]]
