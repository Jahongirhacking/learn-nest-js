---
tags: [express/structure, level/beginner]
level: beginner
topic: Express Structure
---

# Express Route Handlers

> **Bir jumlada:** route handler — Controller'ning Express varianti; farqi shundaki, uni **yupqa ushlab turishga hech kim majburlamaydi**.

**Before learning:** [[Express Routing]], [[Express Middleware]]
**After learning:** [[Express Service Layer]], [[Express Project Structure Problem]]

**NestJS'dagi mos note:** [[NestJS Controllers]]

---

## 1. Boshlanishi — hammasi bitta joyda

Har bir Express loyihasi shunday boshlanadi:

```js
app.get('/players/:id', async (req, res, next) => {
  try {
    const player = await prisma.playerProfile.findUnique({
      where: { id: req.params.id },
      include: { user: true },
    });
    if (!player) return res.status(404).json({ message: 'Not found' });
    res.json(player);
  } catch (err) { next(err); }
});
```

Bitta endpoint uchun bu **mukammal**. Muammo keyinroq boshlanadi.

## 2. Olti oydan keyin

```js
app.post('/trials/:id/finish', async (req, res, next) => {
  try {
    // auth tekshiruvi
    if (!req.user) return res.status(401).json({ message: 'Unauthorized' });
    if (!req.user.roles.includes('coach')) {
      return res.status(403).json({ message: 'Forbidden' });
    }

    // validatsiya
    if (!req.body.result || !['PASSED', 'FAILED'].includes(req.body.result)) {
      return res.status(400).json({ message: 'result must be PASSED or FAILED' });
    }

    // ma'lumot o'qish
    const trial = await prisma.trial.findUnique({ where: { id: req.params.id } });
    if (!trial) return res.status(404).json({ message: 'Trial not found' });

    // business qoidalar
    if (trial.status === 'FINISHED') {
      return res.status(409).json({ message: 'Already finished' });
    }
    if (trial.coachId !== req.user.id) {
      return res.status(403).json({ message: 'Not your trial' });
    }

    // yozish
    const updated = await prisma.trial.update({
      where: { id: req.params.id },
      data: { status: 'FINISHED', result: req.body.result },
    });
    await prisma.recommendation.updateMany({
      where: { trialId: trial.id },
      data: { status: req.body.result === 'PASSED' ? 'ACCEPTED' : 'REJECTED' },
    });
    await prisma.scoutStats.update({
      where: { userId: trial.scoutId },
      data: { finishedTrials: { increment: 1 } },
    });

    // xabar yuborish
    await sendEmail(trial.playerEmail, 'Trial finished');
    await notificationQueue.add('trial-finished', { trialId: trial.id });

    res.json(updated);
  } catch (err) { next(err); }
});
```

**60 qator bitta funksiyada.** Va bu real Express kodining o'rtacha ko'rinishi.

## 3. Muammolar ro'yxati

```text
1. Test qilib bo'lmaydi
   → business qoidani sinash uchun HTTP request kerak

2. Qayta ishlatib bo'lmaydi
   → xuddi shu mantiq cron job'da kerak bo'lsa — copy-paste

3. Transaction yo'q
   → uchta update alohida; o'rtada xato bo'lsa — nomuvofiq holat

4. Email requestni bloklaydi
   → user 3 soniya kutadi ([[Queue vs Synchronous Request]])

5. Auth/validation har bir handler'da takrorlanadi
   → 50 ta joyda copy-paste, bittasi eskirib qoladi

6. HTTP va business logic aralashgan
   → res.status(403) business qoidaning ichida
```

## 4. Yechim — qatlamlarga bo'lish

```js
// routes/trials.routes.js — YUPQA
router.post('/:id/finish',
  requireAuth,                        // → Guard
  requireRole('coach'),               // → Guard
  validate(finishTrialSchema),        // → Pipe
  asyncHandler(async (req, res) => {  // → Controller
    const trial = await trialsService.finish(req.params.id, req.body, req.user.id);
    res.json(trial);
  }),
);
```

```js
// services/trials.service.js — BUSINESS LOGIC
async function finish(trialId, dto, userId) {
  const trial = await prisma.trial.findUnique({ where: { id: trialId } });
  if (!trial) throw NotFound('Trial not found');
  if (trial.status === 'FINISHED') throw Conflict('Already finished');
  if (trial.coachId !== userId) throw Forbidden('Not your trial');

  const updated = await prisma.$transaction(async (tx) => {
    const t = await tx.trial.update({ where: { id: trialId }, data: { ... } });
    await tx.recommendation.updateMany({ ... });
    await tx.scoutStats.update({ ... });
    return t;
  });

  await notificationQueue.add('trial-finished', { trialId });   // tx tashqarisida
  return updated;
}
```

Endi handler 3 qator. Va bu **aynan NestJS controller'i**:

```ts
@Roles('coach')
@Patch(':id/finish')
finish(@Param('id') id: string, @Body() dto: TrialResultDto, @CurrentUser('id') userId: string) {
  return this.trials.finish(id, dto, userId);
}
```

## 5. Farqni ko'r

| | Express | NestJS |
|---|---|---|
| Handler'ni yupqa qilish | **ixtiyoriy** — hech kim majburlamaydi | tabiiy (service inject qilinadi) |
| Auth | har route'da qo'lda qo'shasan | global guard |
| Validatsiya | har route'da qo'lda | global pipe + DTO |
| Xato → status code | `res.status(404)` qo'lda | `throw NotFoundException()` |
| `try/catch` | har joyda | kerak emas |
| Service topish | `require('../services/...')` | constructor injection |

**Xulosa:** Express'da toza arxitektura qurish **mumkin** — lekin u **intizomga** bog'liq.
NestJS'da u **default**.

## 6. Common mistakes

- **Handler'da Prisma chaqirish** → test va qayta ishlatish yo'qoladi.
- **Handler'da `res.status()` bilan business qaror** → mantiq HTTP'ga yopishib qoladi.
- **`return` ni unutish** → ikki marta javob ([[Express Request and Response]]).
- **`asyncHandler` ni unutish** → osilib qolish ([[Express Async Errors]]).
- **Har bir handler'da auth'ni qo'lda tekshirish** → bittasini unutasan.

## 7. Practice problems

### Problem 1 — 60 qatorlik handler'ni yoz
2-bo'limdagi handler'ni to'liq yozib ishga tushir. Ishlashiga ishonch hosil qil.

### Problem 2 — uni testlashga urin
Business qoidani (`Already finished` → 409) **HTTP'siz** test qilishga harakat qil.
**Savol:** mumkinmi? Nega yo'q?

### Problem 3 — qatlamlarga bo'l
4-bo'limdagidek `trials.service.js` ga ajrat. Endi xuddi shu qoidani test qil.
Necha qator test kodi kerak bo'ldi?

### Problem 4 — transaction qo'sh
Uchta `update` ni `prisma.$transaction` ichiga ol.
O'rtasida ataylab `throw` qilib, rollback ishlaganini tekshir ([[Database Transactions]]).

### Problem 5 — emailni queue'ga chiqar
`sendEmail` ni queue'ga ko'chir va response vaqtini oldin/keyin o'lchab ko'r.

### Problem 6 — NestJS'ga o'gir
Xuddi shu featureni NestJS controller + service qilib yoz.
Controller necha qator? Qaysi qismlar butunlay yo'qoldi?

## 8. Related

[[Express Service Layer]] · [[Express Middleware]] · [[Express Async Errors]] · [[Express Project Structure Problem]] · [[NestJS Controllers]] · [[NestJS Services]] · [[Database Transactions]]
