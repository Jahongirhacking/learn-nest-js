---
tags: [production, level/advanced]
level: advanced
topic: Production
---

# Testing in NestJS

> **Bir jumlada:** DI test yozishni oson qiladi — dependency'ni almashtirish bir qator kod; test darajalari esa uch xil savolga javob beradi.

**Before learning:** [[NestJS Dependency Injection]], [[NestJS Services]]
**After learning:** CI/CD, [[ESLint and Prettier]]

> 💡 **Tushunmadingmi? Express'da qo'lda qanday yozilishini ko'r:** [[Express Testing]]


---

## 1. Definition

```text
Jest · @nestjs/testing · ts-jest · supertest
```

Uch daraja:

```text
Unit         → bitta service/class (mock dependency)
Integration  → bir necha qatlam birga (service + DB)
E2E          → real HTTP flow (POST /api/v1/...)
```

## 2. Why — nega kerak?

Har bir daraja boshqa savolga javob beradi:

```text
Unit        → "business qoida to'g'ri yozilganmi?"       tez, ko'p
Integration → "query va schema to'g'ri ishlaydimi?"      o'rtacha
E2E         → "butun zanjir (guard→pipe→controller) ishlaydimi?"  sekin, kam
```

Faqat unit test yozish — guard/pipe konfiguratsiyasidagi xatolarni ushlamaydi. Faqat E2E — sekin va sababni ko'rsatmaydi.

## 3. Internal mental model

`Test.createTestingModule` — bu o'sha DI container, faqat test uchun:

```ts
const moduleRef = await Test.createTestingModule({
  providers: [
    PlayersService,
    { provide: PrismaService, useValue: mockPrisma },   // ← almashtirish
  ],
}).compile();

const service = moduleRef.get(PlayersService);
```

Nimani mock qilish kerak:

```text
✓ tashqi servislar (email, SMS, S3, to'lov)
✓ vaqt va tasodif (Date.now, randomUUID)
~ database — unit'da mock, integration'da real (test DB)
```

## 4. FotSpot implementation

```text
Unit         → PlayersService, TrialsService business qoidalari
Integration  → Prisma query'lari (alohida test database)
E2E          → auth flow: register → login → me → refresh → logout
```

E2E'da guard zanjiri to'liq ishlaydi, ya'ni [[FotSpot Guard Chain]] konfiguratsiyasi ham test qilinadi — bu juda qimmatli.

## 5. Code example

```ts
// unit
describe('PlayersService', () => {
  let service: PlayersService;
  const prisma = { playerProfile: { findUnique: jest.fn() } };

  beforeEach(async () => {
    const ref = await Test.createTestingModule({
      providers: [PlayersService, { provide: PrismaService, useValue: prisma }],
    }).compile();
    service = ref.get(PlayersService);
  });

  it('throws when player is missing', async () => {
    prisma.playerProfile.findUnique.mockResolvedValue(null);
    await expect(service.findOne('x')).rejects.toThrow(NotFoundException);
  });
});
```

```ts
// e2e
describe('Auth (e2e)', () => {
  let app: INestApplication;

  beforeAll(async () => {
    const ref = await Test.createTestingModule({ imports: [AppModule] }).compile();
    app = ref.createNestApplication();
    app.useGlobalPipes(new ValidationPipe({ whitelist: true, forbidNonWhitelisted: true }));
    await app.init();
  });

  it('rejects extra properties', () =>
    request(app.getHttpServer())
      .post('/auth/register')
      .send({ email: 'a@b.com', password: 'password123', role: 'admin' })
      .expect(400));                       // ← forbidNonWhitelisted ishlayaptimi?

  afterAll(() => app.close());
});
```

## 6. Common mistakes

- **E2E'da global pipe/guard'ni sozlamaslik** → test production'dan boshqacha muhitda ishlaydi va yolg'on ishonch beradi.
- **Testlar orasida DB'ni tozalamaslik** → testlar bir-biriga ta'sir qiladi.
- **`app.close()` ni unutish** → Jest tugamay osilib qoladi.
- **Hamma narsani mock qilish** → test faqat mock'ni tekshiradi.
- **Tashqi servisni mock qilmaslik** → CI'da haqiqiy SMS ketadi.
- **Vaqtga bog'liq testlar** → `jest.useFakeTimers()`.
- **Faqat "happy path" testi** → xato holatlari sinovsiz qoladi.

## 7. Practice

1. `PlayersService.findOne` uchun unit test yoz (topilgan va topilmagan holat).
2. E2E test yozib, `forbidNonWhitelisted` ishlayotganini tekshir.
3. Guard'ni test qil: tokensiz 401, noto'g'ri role bilan 403.
4. Test uchun alohida database sozlab, integration test yoz.
5. Testlarni CI'da ishlatadigan script qo'sh.

## 8. Related concepts

[[NestJS Dependency Injection]] · [[NestJS Services]] · [[NestJS ValidationPipe]] · [[FotSpot Guard Chain]] · [[ESLint and Prettier]]
