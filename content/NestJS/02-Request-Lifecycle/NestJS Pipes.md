---
tags: [nestjs/lifecycle, level/beginner]
level: beginner
topic: Request Lifecycle
---

# NestJS Pipes

> **Bir jumlada:** Pipe controller handler'iga kirayotgan inputni **transform** yoki **validate** qiladi.

**Before learning:** [[NestJS Request Lifecycle]], [[NestJS Controllers]]
**After learning:** [[NestJS ValidationPipe]], [[NestJS DTO]]

> 💡 **Tushunmadingmi? Express'da qo'lda qanday yozilishini ko'r:** [[Express Validation Middleware]]


---

## 1. Definition

```ts
@Injectable()
export class ParsePositiveIntPipe implements PipeTransform {
  transform(value: string, metadata: ArgumentMetadata) {
    const n = Number(value);
    if (!Number.isInteger(n) || n <= 0) {
      throw new BadRequestException('Must be a positive integer');
    }
    return n;                     // ← controller shu qiymatni oladi
  }
}
```

Pipe qaytargan qiymat handler parametriga tushadi.

## 2. Why — nega kerak?

Chunki controller ichida quyidagi kod takrorlanmasligi kerak:

```ts
if (!body.email || typeof body.email !== 'string') { ... }   // ✗
```

Pipe input tekshiruvini **bir marta** e'lon qilish imkonini beradi va controller faqat toza, to'g'ri tipdagi ma'lumot oladi.

## 3. Internal mental model

```text
Guard (ruxsat)  →  Pipe (input)  →  Controller
```

Pipe ikki xil ish qiladi:

```text
transform  → '42' (string)  →  42 (number)
validate   → noto'g'ri bo'lsa exception tashlaydi
```

Built-in pipe'lar:

```text
ValidationPipe · ParseIntPipe · ParseBoolPipe · ParseUUIDPipe
ParseArrayPipe · DefaultValuePipe · ParseEnumPipe
```

Qo'llanish darajalari:

```text
parametr:   @Param('id', ParseUUIDPipe)
method:     @UsePipes(new ValidationPipe())
global:     app.useGlobalPipes(new ValidationPipe())
```

## 4. FotSpot implementation

FotSpot global `ValidationPipe` ishlatadi ([[NestJS Application Bootstrap]]):

```ts
app.useGlobalPipes(
  new ValidationPipe({
    whitelist: true,
    forbidNonWhitelisted: true,
    transform: true,
  }),
);
```

Demak har bir `@Body() dto: CreatePlayerDto` avtomatik tarzda:

```text
1. CreatePlayerDto class instance'iga aylantiriladi (transform)
2. class-validator qoidalari tekshiriladi
3. ortiqcha maydonlar olib tashlanadi / rad etiladi
```

Batafsil: [[NestJS ValidationPipe]]

## 5. Code example

```ts
@Controller('players')
export class PlayersController {
  // parametr darajasida pipe
  @Get(':id')
  findOne(@Param('id', ParseUUIDPipe) id: string) {
    return this.service.findOne(id);
  }

  // query default qiymat + tip
  @Get()
  findAll(@Query('page', new DefaultValuePipe(1), ParseIntPipe) page: number) {
    return this.service.findAll(page);
  }
}
```

## 6. Common mistakes

- **Pipe'ni Guard o'rniga ishlatish** → "ruxsat" savoli pipe'niki emas.
- **Pipe ichida database'ga murojaat qilib, business qoidani tekshirish** → bu service ishi; pipe faqat **shakl** bilan shug'ullanadi.
- **`transform: true` siz DTO'ni class deb o'ylash** → u oddiy plain object bo'lib qoladi, `class-transformer` konvertatsiyalari ishlamaydi.
- **Qaytarish qiymatini unutish** — `transform()` hech narsa qaytarmasa, parametr `undefined` bo'ladi.
- **Global pipe'ni ro'yxatdan o'tkazmay, DTO validation kutish.**

## 7. Practice

1. `ParsePositiveIntPipe` yoz va `@Query('page', ParsePositiveIntPipe)` da ishlat.
2. `ParseUUIDPipe` qo'shib, noto'g'ri id yubor — status code nechchi? Nega?
3. Guard va Pipe ikkalasiga `console.log` qo'yib tartibni tasdiqla.
4. Global pipe'ni vaqtincha o'chir va `{"isAdmin": true}` yuborib farqni ko'r.

## 8. Related concepts

[[NestJS ValidationPipe]] · [[NestJS DTO]] · [[NestJS Request Lifecycle]] · [[Middleware vs Guard vs Pipe vs Interceptor vs Filter]] · [[NestJS Guards]]
