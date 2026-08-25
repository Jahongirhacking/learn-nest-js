---
tags: [infrastructure, level/intermediate, architecture]
level: intermediate
topic: Infrastructure
---

# Queue vs Synchronous Request

> **Bir jumlada:** request faqat **userga darhol kerak bo'lgan** ishni bajarsin; qolgani navbatga.

**Before learning:** [[BullMQ]], [[NestJS Services]]
**After learning:** [[WebSockets in NestJS]], [[Database Transactions]]

> 💡 **Tushunmadingmi? Express'da qo'lda qanday yozilishini ko'r:** [[Express Redis and Queues]]


---

## 1. Definition

```text
Sinxron:   request → hamma ish → response
Asinxron:  request → zarur ish + queue.add() → response
                                  ↓
                            worker → qolgan ish
```

## 2. Why — nega kerak?

Yomon:

```text
POST /something
   ├── send email
   ├── generate data
   ├── notify users
   └── long operation
   ↓
response
```

Yaxshi:

```text
POST /something
   ├── database update
   └── queue.add()
   ↓
fast response

Worker
   ├── email
   ├── notification
   └── expensive work
```

Uch xil foyda:

```text
1. Tezlik     → user kutmaydi
2. Bardoshlik → tashqi servis yiqilsa, retry qiladi ([[BullMQ]])
3. Masshtab   → workerlar sonini alohida oshirish mumkin
```

## 3. Internal mental model

Qaror daraxti:

```text
Bu natija javobda KERAKMI?
 ├── HA   → sinxron bajar
 └── YO'Q → queue

Bu ish 200ms dan uzoqmi yoki tashqi servisga bog'liqmi?
 ├── HA   → queue
 └── YO'Q → sinxron ham bo'ladi

Xato bo'lsa user ko'rishi SHARTMI?
 ├── HA   → sinxron
 └── YO'Q → queue + retry
```

Amaliy misollar:

```text
Sinxron:  parolni tekshirish, DB yozuv, ruxsat tekshiruvi, javob uchun kerak bo'lgan o'qish
Queue:    email/SMS, push notification, video/rasm qayta ishlash,
          hisobot generatsiyasi, tashqi API sinxronizatsiyasi, tozalash ishlari
```

## 4. FotSpot implementation

```text
POST /trials/:id/finish
   ├── [sinxron] trial natijasini yozish       ← javob uchun kerak
   ├── [sinxron] recommendation statusi        ← bitta transaction ichida
   └── [queue]   notification yuborish         ← userga darhol kerak emas

POST /media/upload-url
   ├── [sinxron] ruxsat + presigned URL        ← javobning o'zi
   └── [queue]   moderatsiya oldidan tayyorgarlik
```

Muhim: queue'ga qo'shish **transaction'dan keyin** bo'ladi ([[Database Transactions]]).

## 5. Code example

```ts
async finish(trialId: string, dto: TrialResultDto) {
  // 1. Javob uchun kerak bo'lgan qism — sinxron va atomik
  const trial = await this.prisma.$transaction(async (tx) => {
    const t = await tx.trial.update({
      where: { id: trialId },
      data: { status: 'FINISHED', result: dto.result },
    });
    await tx.recommendation.updateMany({
      where: { trialId }, data: { status: 'REVIEWED' },
    });
    return t;
  });

  // 2. Kutish shart bo'lmagan qism — background
  await this.queue.add('trial-finished', { trialId: trial.id });

  return trial;                       // user ~50ms da javob oladi
}
```

Foydalanuvchini natijadan xabardor qilish kerak bo'lsa — [[WebSockets in NestJS]] yoki notification.

## 6. Common mistakes

- **Hamma narsani queue'ga tashlash** → oddiy DB yozuvi ham asinxron bo'lib, ma'lumot "biroz keyinroq" paydo bo'ladi va UI buziladi.
- **Javob uchun kerak bo'lgan natijani queue'ga chiqarish** → frontend bo'sh ma'lumot oladi.
- **Queue'ga qo'shishni transaction ichida qilish** → rollback bo'lsa ham job ketadi.
- **User'ga "yuborildi" deb aytib, xatoni hech qachon ko'rsatmaslik** → holatni (`status: PENDING/SENT/FAILED`) saqlab, ko'rsatish kerak.
- **Worker'siz deploy qilish** → joblar to'planib qoladi.

## 7. Practice

1. Sekin (3 soniyalik) email chaqiruvini controller ichida yozib, response vaqtini o'lchab ko'r.
2. Uni queue'ga chiqar va vaqtni qayta o'lchab, farqni yozib ol.
3. Worker'ni to'xtatib, request baribir tez javob berishini tasdiqla.
4. Job holatini DB'da saqlab (`PENDING → SENT`), foydalanuvchiga ko'rsatadigan endpoint yoz.
5. Yuqoridagi qaror daraxti bo'yicha o'z loyihangdagi 5 ta operatsiyani tasnifla.

## 8. Related concepts

[[BullMQ]] · [[Redis]] · [[Database Transactions]] · [[WebSockets in NestJS]] · [[NestJS Services]]
