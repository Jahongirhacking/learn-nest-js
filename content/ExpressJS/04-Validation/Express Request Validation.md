---
tags: [express/validation, level/beginner, security]
level: beginner
topic: Validation
---

# Express Request Validation

> **Bir jumlada:** DTO yo'q dunyoda input tekshiruvi — qo'lda yozilgan `if` lar; bu ish zerikarli bo'lgani uchun ko'pincha **unutiladi**, va aynan shu yerdan xavfsizlik teshiklari boshlanadi.

**Before learning:** [[Express Request and Response]]
**After learning:** [[Express Validation Middleware]], [[NestJS DTO]]

**NestJS'dagi mos note:** [[NestJS DTO]] · [[NestJS ValidationPipe]]

---

## 1. Muammo

NestJS'da:

```ts
export class CreatePlayerDto {
  @IsString() @Length(2, 50)  firstName: string;
  @IsDateString()             birthDate: string;
}

@Post()
create(@Body() dto: CreatePlayerDto) {}     // dto ALLAQACHON tekshirilgan
```

Savol: **kim tekshirdi va qachon?** Express'da bu ish to'liq ko'rinadi.

## 2. 1-daraja — hech qanday validatsiya

```js
app.post('/players', async (req, res) => {
  const player = await prisma.playerProfile.create({ data: req.body });
  res.status(201).json(player);
});
```

Bu kod **ishlaydi**. Va u xavfli:

```bash
# Xato tip
curl -X POST -d '{"firstName": 123}' ...
# → Prisma xatosi → 500

# Ortiqcha maydon
curl -X POST -d '{"firstName":"Ali","isVerified":true,"role":"admin"}' ...
# → hammasi database'ga yoziladi!
```

Oxirgisi — **mass assignment** zaifligi. Foydalanuvchi o'ziga `admin` rolini bera oladi.

## 3. 2-daraja — qo'lda `if` lar

```js
app.post('/players', async (req, res, next) => {
  const { firstName, lastName, birthDate, position } = req.body;
  const errors = [];

  if (typeof firstName !== 'string') errors.push('firstName must be a string');
  else if (firstName.length < 2 || firstName.length > 50) {
    errors.push('firstName must be 2-50 chars');
  }

  if (typeof lastName !== 'string') errors.push('lastName must be a string');

  if (typeof birthDate !== 'string' || isNaN(Date.parse(birthDate))) {
    errors.push('birthDate must be a valid date');
  }

  if (position !== undefined && !['GK','DF','MF','FW'].includes(position)) {
    errors.push('position must be one of GK, DF, MF, FW');
  }

  if (errors.length) return res.status(400).json({ message: errors });

  // MUHIM: faqat ruxsat etilgan maydonlarni ol (whitelist)
  const data = { firstName, lastName, birthDate: new Date(birthDate), position };

  const player = await prisma.playerProfile.create({ data });
  res.status(201).json(player);
});
```

**Ishlaydi, lekin:**

```text
✗ 20 qator — bitta endpoint uchun
✗ har bir endpointda takrorlanadi
✗ birini yangilasang, boshqasi eskiradi
✗ Swagger uchun hech narsa bermaydi
✗ tiplar yo'q (TypeScript bilsa ham, runtime bilmaydi)
```

E'tibor ber: `const data = { firstName, ... }` qatori — bu **`whitelist: true`** ning qo'lda yozilgani.

## 4. 3-daraja — schema kutubxonasi (zod)

```js
// validation/player.schema.js
const { z } = require('zod');

const createPlayerSchema = z.object({
  firstName: z.string().min(2).max(50),
  lastName:  z.string().min(2).max(50),
  birthDate: z.coerce.date(),                       // ← transform
  position:  z.enum(['GK','DF','MF','FW']).optional(),
}).strict();                                        // ← forbidNonWhitelisted

module.exports = { createPlayerSchema };
```

```js
app.post('/players', async (req, res, next) => {
  const parsed = createPlayerSchema.safeParse(req.body);
  if (!parsed.success) {
    return res.status(400).json({ message: parsed.error.issues });
  }
  const player = await prisma.playerProfile.create({ data: parsed.data });
  res.status(201).json(player);
});
```

**Endi NestJS'ga juda yaqin.** Solishtir:

| zod | class-validator (NestJS) |
|---|---|
| `z.string().min(2)` | `@IsString() @MinLength(2)` |
| `.optional()` | `@IsOptional()` |
| `z.enum([...])` | `@IsEnum(...)` |
| `z.coerce.date()` | `@Type(() => Date)` + `transform: true` |
| `.strict()` | `forbidNonWhitelisted: true` |
| `.parse()` natijasi | tozalangan DTO (`whitelist: true`) |
| `safeParse` har route'da | **global pipe** — avtomatik |

Oxirgi qator asosiy farq: zod bilan ham **har bir route'da `safeParse` chaqirishni unutmaslik** kerak.
NestJS'da esa pipe global — unutish imkoni yo'q.

Buni middleware'ga chiqarish — keyingi qadam: [[Express Validation Middleware]].

## 5. Nima uchun bu security masalasi

```text
whitelist yo'q  →  mass assignment
   { role: 'admin' }  →  DB'ga tushadi

tip tekshiruvi yo'q  →  kutilmagan crash (500)
   { firstName: { $ne: null } }  →  ORM'ga qarab xavfli bo'lishi mumkin

uzunlik cheklovi yo'q  →  10MB matn DB'ga yoziladi

enum tekshiruvi yo'q  →  "activ", "ACTIVE", "active" aralashadi
```

FotSpot'da `forbidNonWhitelisted: true` aynan birinchi qator uchun yoqilgan ([[NestJS ValidationPipe]]).

## 6. Common mistakes

- **`req.body` ni to'g'ridan-to'g'ri `create({ data: req.body })` ga berish** → mass assignment.
- **Faqat mavjudlikni tekshirish** (`if (!firstName)`) → tip va uzunlik tekshirilmaydi.
- **`req.query` ni number deb o'ylash** → har doim string.
- **Validatsiyani service ichida qilish** → har bir chaqiruvchi uchun takrorlanadi.
- **Xato matnida ichki maydon nomlarini oshkor qilish** (ba'zi loyihalarda muhim).
- **Nested obyektni tekshirmaslik** → `{ profile: { role: 'admin' } }`.

## 7. Practice problems

### Problem 1 — himoyasiz endpoint
1-darajadagi endpointni yozib, `{"firstName":"Ali","role":"admin"}` yubor.
Database'da nima paydo bo'ldi?

### Problem 2 — qo'lda `if` lar
2-darajaga o'tkaz. Necha qator bo'ldi? Endi buni yana 4 ta endpointga takrorla.

### Problem 3 — whitelist
`const data = { firstName, ... }` qatorini olib tashlab, `role` yana o'tib ketishini ko'r.
**Savol:** bu NestJS'dagi qaysi option?

### Problem 4 — zod
3-darajaga o'tkaz. `.strict()` bilan va busiz `{"role":"admin"}` yuborib farqni yoz.

### Problem 5 — query tipi
`GET /players?page=2` da `z.coerce.number()` ishlatib ko'r.
**Savol:** bu NestJS'dagi `transform: true` ning qaysi qismi?

### Problem 6 — unutish
5 ta endpointdan bittasida `safeParse` ni ataylab unut.
**Savol:** buni qanday topasan? NestJS'da bu mumkinmi?

### Problem 7 — NestJS'ga o'gir
zod schema'ni `CreatePlayerDto` ga aylantir va global pipe bilan ishlat.
Route kodidan nechta qator yo'qoldi?

## 8. Related

[[Express Validation Middleware]] · [[Express Request and Response]] · [[NestJS DTO]] · [[NestJS ValidationPipe]] · [[Authorization]]
