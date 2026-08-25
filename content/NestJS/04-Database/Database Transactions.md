---
tags: [database, level/advanced]
level: advanced
topic: Database
---

# Database Transactions

> **Bir jumlada:** bir nechta yozuv operatsiyasi mantiqan bitta bo'lsa — ular ham **birga bajarilishi, ham birga bekor bo'lishi** kerak.

**Before learning:** [[Prisma]], [[NestJS Services]]
**After learning:** [[BullMQ]], [[Testing in NestJS]]

> 💡 **Tushunmadingmi? Express'da qo'lda qanday yozilishini ko'r:** [[Express Prisma Setup]]


---

## 1. Definition

```text
BEGIN
  operation A
  operation B
  operation C
COMMIT

xato bo'lsa → ROLLBACK (hech biri saqlanmaydi)
```

## 2. Why — nega kerak?

FotSpot'dagi real misol:

```text
Recommendation status update
        +
Scout statistics update
        +
Notification create
```

Agar birinchi ikkitasi bajarilib, uchinchisi xato bersa — ma'lumotlar **nomuvofiq** holatga tushadi: tavsiya qabul qilingan, statistika o'zgargan, lekin hech kim xabardor emas. Bunday holatni keyinchalik topish va tuzatish juda qiyin.

## 3. Internal mental model

Prisma ikki xil transaction beradi:

```ts
// 1. Sequential array — bir-biriga bog'liq bo'lmagan operatsiyalar
await prisma.$transaction([
  prisma.a.update(...),
  prisma.b.create(...),
]);

// 2. Interactive — oraliq natija kerak bo'lganda
await prisma.$transaction(async (tx) => {
  const rec = await tx.recommendation.update({ ... });
  await tx.scoutStats.update({ where: { id: rec.scoutId }, ... });
  return rec;
});
```

Muhim qoidalar:

```text
✓ tx ichida FAQAT tx ishlatiladi (this.prisma emas!)
✓ transaction imkon qadar QISQA bo'lsin
✗ tx ichida HTTP call / email / fayl yuklash QILINMAYDI
```

Oxirgisi juda muhim: tashqi chaqiruv sekin bo'lsa, transaction ochiq qoladi va boshqa so'rovlarni bloklaydi. Bunday ishlar [[BullMQ]] ga chiqariladi.

## 4. FotSpot implementation

```text
TrialsService.finishTrial()
   ↓ $transaction
   ├── trial.update(result)
   ├── recommendation.update(status)
   └── scoutStats.update(counters)
   ↓ COMMIT
   ↓ (transaction TASHQARISIDA)
   └── notificationsQueue.add(...)     → background
```

Diqqat: notification transaction'dan **keyin** qo'shiladi. Aks holda transaction rollback bo'lsa ham, xabar allaqachon yuborilgan bo'lardi.

## 5. Code example

```ts
async finishTrial(trialId: string, dto: TrialResultDto, userId: string) {
  const trial = await this.prisma.$transaction(async (tx) => {
    const t = await tx.trial.findUnique({ where: { id: trialId } });
    if (!t) throw new NotFoundException('Trial not found');
    if (t.status === 'FINISHED') throw new ConflictException('Already finished');

    const updated = await tx.trial.update({
      where: { id: trialId },
      data: { status: 'FINISHED', result: dto.result, finishedBy: userId },
    });

    await tx.recommendation.updateMany({
      where: { trialId },
      data: { status: dto.result === 'PASSED' ? 'ACCEPTED' : 'REJECTED' },
    });

    await tx.scoutStats.update({
      where: { userId: t.scoutId },
      data: { finishedTrials: { increment: 1 } },
    });

    return updated;
  });

  await this.notifications.enqueueTrialFinished(trial.id);   // tx tashqarisida
  return trial;
}
```

## 6. Common mistakes

- **`tx` o'rniga `this.prisma` ishlatish** → o'sha operatsiya transaction'dan tashqarida qoladi va rollback qilinmaydi. Eng ko'p uchraydigan xato.
- **Transaction ichida tashqi API chaqirish** → uzoq lock, timeout.
- **Hamma narsani transaction'ga o'rash** → bitta `update` uchun kerak emas (u allaqachon atomik).
- **Deadlock'ni o'ylamaslik** → yozuvlarni har doim bir xil tartibda yangila.
- **Timeout'ni sozlamaslik** → `$transaction(fn, { timeout: 10_000, maxWait: 5_000 })`.
- **Concurrency'ni hisobga olmaslik** → ikki request bir vaqtda "ikki marta qabul qilish"; `status` tekshiruvi transaction ichida bo'lishi kerak.

## 7. Practice

1. Uchta bog'liq yozuvni transaction'siz yozib, o'rtasida ataylab xato tashla — DB holatini ko'r.
2. Xuddi shu kodni `$transaction` ichiga ol va rollback ishlaganini tasdiqla.
3. `tx` o'rniga `this.prisma` yozib xatoni takrorlab ko'r — nima saqlanib qoladi?
4. Transaction ichiga `await fetch(...)` qo'shib, davomiyligini o'lchab ko'r, keyin uni queue'ga chiqar.
5. Bir vaqtda ikkita "finish" so'rovini yubor — ikkalasi ham o'tib ketmasin.

## 8. Related concepts

[[Prisma]] · [[NestJS Services]] · [[BullMQ]] · [[Queue vs Synchronous Request]] · [[Database Relations]]
