---
tags: [nestjs/core, level/beginner, hub]
level: beginner
topic: NestJS Core
---

# NestJS Request Lifecycle

> **Bir jumlada:** HTTP request NestJS ichida qat'iy belgilangan bosqichlardan o'tadi — va deyarli har bir NestJS concept shu zanjirning bitta bo'g'ini.

**Before learning:** [[NestJS Overview]], [[NestJS Modules]], [[NestJS Controllers]]
**After learning:** [[NestJS Middleware]], [[NestJS Guards]], [[NestJS Pipes]], [[NestJS Interceptors]], [[NestJS Exception Filters]]

> 💡 **Tushunmadingmi? Express'da qo'lda qanday yozilishini ko'r:** [[Express Middleware]]


Bu **markaziy note**. Yangi concept o'rganganda birinchi savol doim shu:
> "Bu narsa lifecycle'ning qaysi joyida turadi?"

---

## 1. Definition

Request lifecycle — request serverga kelganidan javob qaytgunicha bo'lgan bosqichlar ketma-ketligi.

```text
Client Request
   |
   v
Middleware              ← request tayyorlanadi (raw req/res)
   |
   v
Guards                  ← "kirishga ruxsat bormi?"
   |
   v
Interceptors (before)   ← execution'ni o'rab oladi
   |
   v
Pipes                   ← input validate / transform
   |
   v
Controller handler      ← route method
   |
   v
Service                 ← business logic
   |
   v
Prisma / Redis / Queue / Storage
   |
   v
Interceptors (after)    ← response transformation
   |
   v
Response

Exception istalgan joyda otilsa:
   |
   v
Exception Filter  →  standard JSON response
```

## 2. Why — nega kerak?

Chunki NestJS'dagi "magic"ning katta qismi shu tartibda tushuntiriladi:

- Nega `@Body()` DTO allaqachon validatsiyadan o'tgan holda keladi? → Pipe controller'dan **oldin** ishlaydi.
- Nega Guard ichida DTO validatsiyasiga ishonib bo'lmaydi? → Guard Pipe'dan **oldin** ishlaydi.
- Nega `request.user` Guard'dan keyin mavjud? → [[Passport and JwtStrategy]] Guard bosqichida user'ni qo'yadi.
- Nega barcha xatolar bir xil formatda qaytadi? → [[NestJS Exception Filters]] eng oxirida turadi.

## 3. Internal mental model

Har bir bosqichning **bitta savoli** bor:

```text
Middleware   → "Requestni tayyorlaymanmi / loglaymanmi?"
Guard        → "Bu request davom etsinmi? (ha / yo'q)"
Interceptor  → "Execution atrofida nima qilaman? (before/after)"
Pipe         → "Input to'g'rimi va kerakli tipdami?"
Controller   → "Qaysi service'ni chaqiraman?"
Service      → "Business rule bo'yicha nima qilinadi?"
Filter       → "Xatoni qanday response'ga aylantiraman?"
```

Farqlarni chalkashtirmaslik uchun: [[Middleware vs Guard vs Pipe vs Interceptor vs Filter]]

## 4. FotSpot implementation

FotSpot'da lifecycle'ning global qismi `main.ts` va `app.module.ts` da o'rnatilgan:

```text
Request
  |
  v
ThrottleGuard        ← rate limit (arzon tekshiruv birinchi)
  |
  v
JwtAuthGuard         ← identity (kim?)
  |
  v
RolesGuard           ← role (qaysi toifadagi user?)
  |
  v
PermissionsGuard     ← permission (aniq action'ga ruxsat?)
  |
  v
ValidationPipe       ← DTO validation (global)
  |
  v
Controller → Service → Prisma / Redis / BullMQ / R2
  |
  v
Response  /  HttpExceptionFilter
```

Batafsil: [[FotSpot Guard Chain]]

## 5. Code example

```ts
// main.ts — global lifecycle qismlari
const app = await NestFactory.create(AppModule);

app.useGlobalPipes(
  new ValidationPipe({
    whitelist: true,
    forbidNonWhitelisted: true,
    transform: true,
  }),
);

app.useGlobalFilters(new HttpExceptionFilter());

await app.listen(port, '0.0.0.0');
```

```ts
// route darajasida lifecycle
@Controller('players')
export class PlayersController {
  constructor(private readonly playersService: PlayersService) {}

  @Roles('admin')                 // ← metadata, RolesGuard o'qiydi
  @Post()
  create(@Body() dto: CreatePlayerDto) {   // ← Pipe bu yerda ishlagan
    return this.playersService.create(dto);
  }
}
```

## 6. Common mistakes

- **Guard ichida business logic yozish.** Guard faqat `true/false` qaytaradi.
- **Guard tartibini yodlash, sababini bilmaslik.** Nega Throttle birinchi? Chunki eng arzon va public endpointlarni ham himoya qiladi.
- **Middleware va Guard'ni bir xil deb o'ylash.** Middleware Nest context'ini (`ExecutionContext`) bilmaydi, Guard biladi.
- **Interceptor'ni Filter o'rniga ishlatish.** Exception → Filter.
- **Validatsiyani service ichida qayta yozish.** DTO + Pipe allaqachon qilgan.

## 7. Practice

1. Har bosqichga `console.log` qo'yib, bitta requestda log tartibini o'z ko'zing bilan ko'r:
   middleware → guard → interceptor(before) → pipe → controller → interceptor(after).
2. Ataylab noto'g'ri DTO yubor va logni kuzat: pipe controller'gacha to'xtatadimi?
3. Guard'ni `false` qaytaradigan qil — pipe umuman ishlaydimi? Nega?
4. Controller ichida `throw new BadRequestException()` qil va filter ushlashini ko'r.

## 8. Related concepts

- [[NestJS Middleware]] · [[NestJS Guards]] · [[NestJS Pipes]] · [[NestJS Interceptors]] · [[NestJS Exception Filters]]
- [[NestJS DTO]] · [[NestJS ValidationPipe]]
- [[FotSpot Guard Chain]] · [[NestJS Application Bootstrap]]
- [[NestJS Mental Model]]
