---
tags: [nestjs/lifecycle, level/beginner, security]
level: beginner
topic: Request Lifecycle
---

# NestJS ValidationPipe

> **Bir jumlada:** global `ValidationPipe` — DTO qoidalarini butun application bo'ylab avtomatik qo'llaydigan, ayni paytda **security qatlami** bo'lgan pipe.

**Before learning:** [[NestJS DTO]], [[NestJS Pipes]]
**After learning:** [[Authorization]], [[Swagger and OpenAPI]]

> 💡 **Tushunmadingmi? Express'da qo'lda qanday yozilishini ko'r:** [[Express Validation Middleware]]


---

## 1. Definition

```ts
app.useGlobalPipes(
  new ValidationPipe({
    whitelist: true,
    forbidNonWhitelisted: true,
    transform: true,
  }),
);
```

Uchta option — uchta alohida qaror.

## 2. Why — nega kerak?

### `whitelist: true`

DTO'da e'lon qilinmagan barcha maydonlarni **olib tashlaydi**.

```text
kelgan body: { firstName, lastName, isAdmin }
DTO:         { firstName, lastName }
natija:      { firstName, lastName }        ← isAdmin yo'q qilindi
```

### `forbidNonWhitelisted: true`

Ortiqcha maydon bo'lsa, jimgina o'chirmasdan **requestni rad etadi**:

```json
{ "firstName": "Ali", "isAdmin": true }
```

→ `400 Bad Request: property isAdmin should not exist`

Bu **mass assignment** hujumidan himoya: hujumchi `role`, `isVerified`, `balance` kabi maydonlarni tiqishga urinsa, so'rov darhol rad etiladi.

### `transform: true`

Plain JSON'ni DTO **class instance**iga aylantiradi va `@Type()` konvertatsiyalarini ishga soladi (`'2'` → `2`).

## 3. Internal mental model

```text
whitelist            → "ortiqchasini kes"
forbidNonWhitelisted → "ortiqchasi bo'lsa umuman rad et"
transform            → "to'g'ri tip va class instance qil"
```

Ikkinchisi birinchisiga bog'liq: `forbidNonWhitelisted` faqat `whitelist: true` bilan ma'noga ega.

Foydali qo'shimchalar:

```ts
new ValidationPipe({
  whitelist: true,
  forbidNonWhitelisted: true,
  transform: true,
  transformOptions: { enableImplicitConversion: false },  // aniqlik yaxshiroq
  disableErrorMessages: process.env.NODE_ENV === 'production',
})
```

## 4. FotSpot implementation

FotSpot `main.ts` da aynan uchala option yoqilgan ([[NestJS Application Bootstrap]]).

Bu quyidagini kafolatlaydi:

```text
Har bir @Body() DTO:
  ✓ tekshirilgan
  ✓ ortiqcha maydonlarsiz
  ✓ to'g'ri tipda
                     — controller'ga yetib kelgunicha.
```

Shuning uchun FotSpot service'larida `if (typeof dto.email !== 'string')` kabi kod yo'q — u qatlam allaqachon bajarilgan.

## 5. Code example

```ts
export class CreateUserDto {
  @IsEmail()
  email: string;

  @IsString()
  @MinLength(8)
  password: string;
}
```

```text
POST /auth/register
{ "email": "a@b.com", "password": "12345678", "role": "admin" }

→ 400 Bad Request
  "property role should not exist"
```

Role bu yerdan kelmaydi — u serverda [[RBAC]] qoidalari bo'yicha beriladi.

## 6. Common mistakes

- **Global pipe'ni ro'yxatdan o'tkazmaslik** → DTO decoratorlari bezak bo'lib qoladi.
- **`whitelist` yoqilgan, lekin maydonga validator qo'yilmagan** → maydon jimgina yo'qoladi va "nega `undefined`?" degan soatlab davom etadigan debug boshlanadi.
- **`transform: true` siz `@Type()` kutish** → konvertatsiya bo'lmaydi.
- **`enableImplicitConversion: true` ni o'ylamasdan yoqish** → `'abc'` → `NaN` kabi kutilmagan natijalar.
- **Validatsiya xatolarini production'da to'liq ochiq qoldirish** → ichki maydon nomlari ko'rinadi (ba'zi loyihalarda muhim).

## 7. Practice

1. Uchala optionni navbatma-navbat o'chirib yoqib, bir xil requestning natijasini yozib ol.
2. `role` maydonini DTO'siz yuborib, `forbidNonWhitelisted` ta'sirini ko'r.
3. Query DTO'da `@Type(() => Number)` bilan va busiz `page` tipini `typeof` orqali logla.
4. `@ValidateNested()` bilan ichma-ich DTO yozib tekshir.

## 8. Related concepts

[[NestJS DTO]] · [[NestJS Pipes]] · [[NestJS Application Bootstrap]] · [[Authorization]] · [[RBAC]] · [[Swagger and OpenAPI]]
