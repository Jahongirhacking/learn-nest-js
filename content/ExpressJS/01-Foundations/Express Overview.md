---
tags: [express/foundation, level/beginner]
level: beginner
topic: Express Foundations
---

# Express Overview

> **Bir jumlada:** Express — Node'ning `http` moduliga **routing** va **middleware** qo'shadi; boshqa hech narsa qo'shmaydi — arxitektura, struktura, DI hammasi senga qoladi.

**Before learning:** [[Node HTTP Server]]
**After learning:** [[Express App Bootstrap]], [[Express Middleware]]

**NestJS'dagi mos note:** [[NestJS Overview]]

---

## 1. Express nima qo'shadi

Aniq uchta narsa:

```text
1. Routing        → app.get('/players/:id', handler)
2. Middleware     → app.use(fn)   — funksiyalar zanjiri
3. Qulayliklar    → res.json(), req.body (express.json bilan), req.params
```

Va tamom. Express — **mikro-framework**, bu uning ataylab tanlangan dizayni.

## 2. Express nima qo'shMAYDI

Bu ro'yxat NestJS'ni tushunish uchun eng muhim:

```text
✗ Application arxitekturasi (papka strukturasi senga qoladi)
✗ Dependency Injection
✗ Validation
✗ Konfiguratsiya boshqaruvi
✗ Testing infratuzilmasi
✗ TypeScript integratsiyasi (qo'shimcha ish talab qiladi)
✗ Async xatolarni avtomatik ushlash (Express 4)
✗ Bir xil kod yozish qoidalari
```

**Har bir jamoa bularni o'zicha hal qiladi** — natijada ikkita Express loyihasi bir-biriga umuman o'xshamaydi.

## 3. Solishtirish — bir xil endpoint

```js
// Express
const express = require('express');
const app = express();

app.use(express.json());

app.get('/players/:id', async (req, res) => {
  const player = await prisma.playerProfile.findUnique({
    where: { id: req.params.id },
  });
  if (!player) return res.status(404).json({ message: 'Not found' });
  res.json(player);
});

app.listen(3000);
```

```ts
// NestJS
@Controller('players')
export class PlayersController {
  constructor(private readonly players: PlayersService) {}

  @Get(':id')
  findOne(@Param('id') id: string) {
    return this.players.findOne(id);      // xato → NotFoundException
  }
}
```

Kichik loyihada Express **qisqaroq** ko'rinadi. Muammo 50 ta endpointdan keyin boshlanadi — [[Express Project Structure Problem]].

## 4. Qachon Express, qachon NestJS?

```text
Express mos keladi:
  ✓ kichik servis, 5-15 endpoint
  ✓ prototip, hackathon
  ✓ bitta dasturchi
  ✓ to'liq nazorat kerak bo'lgan maxsus holatlar

NestJS mos keladi:
  ✓ katta domain (FotSpot kabi: 10+ feature)
  ✓ jamoa bilan ishlash
  ✓ TypeScript-first
  ✓ auth, validation, queue, testing — hammasi bir joyda kerak
```

Muhim: NestJS **Express ustida** ishlaydi (default adapter). Ya'ni Express'ni bilish NestJS'da behuda ketmaydi — aksincha, u NestJS'ni ochib beradi.

## 5. NestJS Express'ni qayerda ishlatadi

```text
NestJS @Controller / @Get   →  Express router'iga registratsiya
NestJS Middleware           →  aynan Express middleware
NestJS req/res              →  aynan Express req/res obyektlari
                               (@Req() bilan olsang, shuni ko'rasan)
app.use(helmet())           →  NestJS'da ham AYNAN shunday yoziladi
```

Shuning uchun [[Helmet]] va [[CORS]] notelaridagi kod Express'dagidek ko'rinadi — chunki u aynan o'sha.

## 6. Common mistakes

- **"Express eskirgan" deb o'ylash** → yo'q, u boshqa maqsad uchun.
- **Express'da NestJS strukturasini majburan qurish** → foydali, lekin qo'lda ko'p ish; o'shanda NestJS'ga o'tish mantiqiy.
- **Express'ni o'rganmasdan NestJS'ga o'tish** → guard/middleware/`next()` mavzulari abadiy magic bo'lib qoladi.

## 7. Practice problems

### Problem 1
Yuqoridagi Express misolini ishga tushir va `curl` bilan sinab ko'r.

### Problem 2
"Express nima qo'shmaydi" ro'yxatidan 3 tasini tanlab, har biri uchun **o'zing yechim** yozib ko'r (masalan config o'qish, validatsiya).
**Maqsad:** NestJS qancha ish qilib berayotganini his qilish.

### Problem 3
`@Req()` bilan NestJS controller'da raw request obyektini ol va `console.log(req.constructor.name)` qil.
**Savol:** nima chiqdi? Nega?

### Problem 4
Bitta endpointni Express'da yoz, keyin NestJS'da yoz. Ikkalasi uchun ham qator sonini sana.
Keyin **5 ta endpoint** uchun xuddi shunday qil. Farq qanday o'zgardi?

## 8. Related

[[Node HTTP Server]] · [[Express App Bootstrap]] · [[Express Middleware]] · [[Express Project Structure Problem]] · [[NestJS Overview]] · [[NestJS to Express Translation]]
