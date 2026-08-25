---
tags: [nestjs/core, level/beginner]
level: beginner
topic: NestJS Core
---

# NestJS Application Bootstrap

> **Bir jumlada:** `main.ts` — application'ning boshlanish nuqtasi; `NestFactory.create()` DI container va module graph'ni quradi, `app.listen()` esa HTTP serverni yoqadi.

**Before learning:** [[NestJS Overview]]
**After learning:** [[NestJS Modules]], [[NestJS Request Lifecycle]], [[ConfigModule and Environment Variables]]

> 💡 **Tushunmadingmi? Express'da qo'lda qanday yozilishini ko'r:** [[Express App Bootstrap]]


---

## 1. Definition

Bootstrap — Node process boshlanganidan HTTP server request qabul qilishga tayyor bo'lgunicha bo'lgan bosqich.

```text
main.ts
   ↓
NestFactory.create(AppModule)
   ↓
AppModule → barcha Modules
   ↓
Controllers + Providers
   ↓
Global guards / pipes / filters
   ↓
HTTP server (app.listen)
```

## 2. Why — nega kerak?

Chunki **global** narsalarning hammasi shu yerda o'rnatiladi:

- global [[NestJS ValidationPipe]]
- global [[NestJS Exception Filters]]
- [[Helmet]], [[CORS]]
- [[Swagger and OpenAPI]]
- [[Logging in NestJS]] darajalari

Agar endpoint kutilmaganda 400 qaytarsa yoki xato formati boshqacha bo'lsa — javob ko'pincha `main.ts` da.

## 3. Internal mental model

`NestFactory.create()` ichida nima bo'ladi:

```text
1. Module graph quriladi (imports bo'yicha)
2. Providerlar topologik tartibda resolve qilinadi
3. Har bir provider instance yaratiladi (default: singleton)
4. Controllerlar route'larga bog'lanadi
5. HTTP adapter (Express) tayyorlanadi
```

Muhim: `create()` **yaratadi**, `listen()` **eshitadi**. Ikkalasi alohida.

## 4. FotSpot implementation

FotSpot'ning to'liq startup ketma-ketligi:

```text
1.  Node process starts
2.  main.ts executes
3.  production secrets checked        ← xavfsizlik: secret yo'q bo'lsa ishga tushmaydi
4.  NestFactory.create(AppModule)
5.  Config loaded                     ← [[ConfigModule and Environment Variables]]
6.  Modules initialized
7.  Providers instantiated
8.  Global guards / filters registered ← [[FotSpot Guard Chain]]
9.  Helmet / CORS configured
10. ValidationPipe configured
11. Swagger configured
12. HTTP server listens
    ─── endi request qabul qilinadi ───
13. Request → Guards → Pipes → Controller → Service
14. Prisma / Redis / external service
15. Response  (yoki Exception → Filter)
```

## 5. Code example

```ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule, {
    logger: process.env.NODE_ENV === 'production'
      ? ['error', 'warn', 'log']
      : ['error', 'warn', 'log', 'debug', 'verbose'],
  });

  app.use(helmet());
  app.enableCors({ origin: allowedOrigins, credentials: true });

  app.setGlobalPrefix('api/v1');

  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,
      forbidNonWhitelisted: true,
      transform: true,
    }),
  );

  app.useGlobalFilters(new HttpExceptionFilter());

  const config = new DocumentBuilder()
    .setTitle('FotSpot API')
    .setVersion('1.0')
    .addBearerAuth()
    .build();
  SwaggerModule.setup('docs', app, SwaggerModule.createDocument(app, config));

  const port = process.env.PORT ?? 3000;
  await app.listen(port, '0.0.0.0');
}
bootstrap();
```

> `'0.0.0.0'` — container ichida tashqi trafikni qabul qilish uchun. `localhost` bo'lsa, Docker tashqarisidan ko'rinmaydi.

## 6. Common mistakes

- **Global pipe/filter'ni ro'yxatdan o'tkazishni unutish** — DTO validation umuman ishlamaydi va sababi topilmaydi.
- **`useGlobalPipes`ni Swagger'dan keyin/oldin qo'yish tartibini muhim deb o'ylash** — bu qism tartibga sezgir emas, lekin `app.listen()` **eng oxirida** bo'lishi shart.
- **Secretlarni kod ichida qoldirish** — [[ConfigModule and Environment Variables]].
- **`app.listen()` ni `await` qilmaslik** — startup xatolari yo'qoladi.

## 7. Practice

1. Bo'sh projectda `main.ts` ga global ValidationPipe qo'sh va DTO'ga mos kelmaydigan request yubor.
2. Keyin pipe'ni o'chirib, xuddi shu requestni qayta yubor. Farqni tushuntir.
3. `setGlobalPrefix('api/v1')` qo'shib, URL qanday o'zgarganini ko'r.
4. Startup log darajasini env orqali o'zgartiruvchi kod yoz.

## 8. Related concepts

[[NestJS Modules]] · [[NestJS Request Lifecycle]] · [[ConfigModule and Environment Variables]] · [[Helmet]] · [[CORS]] · [[Swagger and OpenAPI]] · [[FotSpot Guard Chain]]
