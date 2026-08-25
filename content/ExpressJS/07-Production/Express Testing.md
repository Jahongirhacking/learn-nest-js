---
tags: [express/production, level/intermediate]
level: intermediate
topic: Production
---

# Express Testing

> **Bir jumlada:** `supertest(app)` — Express'ni portsiz test qilish usuli; NestJS `Test.createTestingModule` unga **dependency almashtirish** qo'shadi.

**Before learning:** [[Express App Bootstrap]], [[Express Manual Dependency Wiring]]
**After learning:** [[Testing in NestJS]]

**NestJS'dagi mos note:** [[Testing in NestJS]]

---

## 1. Nega `app.js` va `server.js` alohida

```js
// app.js
const app = express();
// ... middleware, route'lar
module.exports = app;              // ← listen QILMAYDI

// server.js
require('./app').listen(3000);     // ← faqat shu yerda
```

Agar `app.js` ichida `listen()` bo'lsa, har bir test faylida port band bo'ladi va testlar bir-biriga xalaqit qiladi.

**NestJS'da ham xuddi shu:** `AppModule` alohida, `main.ts` alohida.

## 2. E2E test — supertest

```js
// tests/players.e2e.test.js
const request = require('supertest');
const app = require('../src/app');
const prisma = require('../src/db');

describe('Players API', () => {
  let token;

  beforeAll(async () => {
    await prisma.user.deleteMany();                    // toza holat
    const res = await request(app)
      .post('/api/v1/auth/register')
      .send({ email: 'test@example.com', password: 'password123' });
    token = res.body.accessToken;
  });

  afterAll(async () => {
    await prisma.$disconnect();
  });

  it('401 without token', () =>
    request(app).get('/api/v1/players/me/drafts').expect(401));

  it('200 with token', () =>
    request(app)
      .get('/api/v1/players/me/drafts')
      .set('Authorization', `Bearer ${token}`)
      .expect(200));

  it('400 on extra property', () =>
    request(app)
      .post('/api/v1/players')
      .set('Authorization', `Bearer ${token}`)
      .send({ firstName: 'Ali', role: 'admin' })       // ← ortiqcha maydon
      .expect(400));
});
```

`supertest` portga bind qilmaydi — u `app` ni to'g'ridan-to'g'ri chaqiradi.

NestJS'da bu **deyarli bir xil**:

```ts
const ref = await Test.createTestingModule({ imports: [AppModule] }).compile();
app = ref.createNestApplication();
await app.init();

request(app.getHttpServer()).get('/players').expect(200);
```

## 3. Unit test — va DI ning qiymati

```js
// ✗ require bilan qattiq bog'langan service
const prisma = require('../db');
async function findOne(id) { ... }

// test — modul cache'ini buzish kerak
jest.mock('../db', () => ({ playerProfile: { findUnique: jest.fn() } }));
```

```js
// ✓ constructor injection
class PlayersService {
  constructor(prisma) { this.prisma = prisma; }
  async findOne(id) {
    const p = await this.prisma.playerProfile.findUnique({ where: { id } });
    if (!p) throw NotFound('Player not found');
    return p;
  }
}

// test — hech qanday mocking kutubxonasi kerak emas
const service = new PlayersService({
  playerProfile: { findUnique: async () => null },
});
await expect(service.findOne('x')).rejects.toThrow('Player not found');
```

```ts
// NestJS — xuddi shu, framework boshqaradi
const module = await Test.createTestingModule({
  providers: [PlayersService, { provide: PrismaService, useValue: mockPrisma }],
}).compile();
```

**Xulosa:** DI ning eng aniq foydasi — aynan shu yerda ko'rinadi ([[Express Manual Dependency Wiring]]).

## 4. Middleware'ni test qilish

```js
// requireAuth ni sof funksiya sifatida test qil
const requireAuth = require('../src/middleware/require-auth');

function mockRes() {
  const res = {};
  res.status = jest.fn(() => res);
  res.json = jest.fn(() => res);
  return res;
}

it('rejects request without token', () => {
  const req = { headers: {} };
  const res = mockRes();
  const next = jest.fn();

  requireAuth(req, res, next);

  expect(res.status).toHaveBeenCalledWith(401);
  expect(next).not.toHaveBeenCalled();
});
```

NestJS Guard testi ham xuddi shunday — faqat `ExecutionContext` mock qilinadi.

## 5. Qamrov strategiyasi

```text
Unit        → business qoidalar (tez, ko'p)
              "trial FINISHED bo'lsa, qayta finish 409 beradi"

Integration → service + real test DB
              "query va schema to'g'ri ishlaydimi"

E2E         → to'liq HTTP zanjiri (sekin, kam)
              "auth → validation → handler zanjiri ishlaydimi"
```

E2E ayniqsa qimmatli, chunki u **middleware tartibini** ham tekshiradi — Express'da eng ko'p buziladigan joy ([[Express Middleware Order]]).

## 6. Common mistakes

- **`app.js` ichida `listen()`** → testlarda port konflikti.
- **`prisma.$disconnect()` ni chaqirmaslik** → Jest tugamay osilib qoladi.
- **Testlar orasida DB'ni tozalamaslik** → testlar bir-biriga ta'sir qiladi.
- **Real DB'ni test uchun ishlatish** → ma'lumot yo'qolishi. Alohida test DB.
- **Tashqi servisni (email, SMS) mock qilmaslik** → CI'da haqiqiy xabar ketadi.
- **Faqat "happy path" test qilish** → 401/403/400 holatlari sinovsiz.
- **`require` bilan bog'langan kodni test qilishga urinish** → mocking do'zaxi.

## 7. Practice problems

### Problem 1 — app/server ajratish
`app.js` va `server.js` ni ajratib, `supertest` bilan birinchi testni yoz.

### Problem 2 — auth testlari
Tokensiz 401, to'g'ri token bilan 200, noto'g'ri role bilan 403 — uchalasini test qil.

### Problem 3 — validatsiya testi
`{ firstName: 'Ali', role: 'admin' }` yuborib 400 kutayotgan test yoz.
**Savol:** bu test nimani himoya qilyapti?

### Problem 4 — ikki xil service, ikki xil test
Bitta service'ni (a) `require` bilan, (b) constructor injection bilan yozib, ikkalasini test qil.
Test kodini yonma-yon qo'yib, qaysi biri qisqa va barqarorroq ekanini ayt.

### Problem 5 — middleware testi
`requireAuth` uchun 3 ta unit test yoz: tokensiz, buzilgan token, to'g'ri token.

### Problem 6 — tartib buzilishini ushla
`app.js` da error handler'ni yuqoriga ko'chir va E2E testlar qaysi biri yiqilishini ko'r.
**Savol:** unit testlar buni ushladimi? Nega yo'q?

### Problem 7 — NestJS'ga o'gir
Xuddi shu testlarni `Test.createTestingModule` bilan qayta yoz.
**Savol:** `jest.mock('../db')` nimaga aylandi?

## 8. Related

[[Express App Bootstrap]] · [[Express Manual Dependency Wiring]] · [[Express Middleware Order]] · [[Express Service Layer]] · [[Testing in NestJS]] · [[NestJS Dependency Injection]]
