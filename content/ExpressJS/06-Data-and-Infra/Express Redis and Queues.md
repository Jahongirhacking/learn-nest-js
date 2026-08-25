---
tags: [express/infra, level/intermediate]
level: intermediate
topic: Data and Infrastructure
---

# Express Redis and Queues

> **Bir jumlada:** Redis va BullMQ — mustaqil kutubxonalar; ular NestJS'ga tegishli emas, shuning uchun Express'da ham **aynan bir xil** ishlaydi.

**Before learning:** [[Express Manual Dependency Wiring]], [[Express Service Layer]]
**After learning:** [[Redis]], [[BullMQ]], [[Queue vs Synchronous Request]]

**NestJS'dagi mos note:** [[Redis]] · [[BullMQ]] · [[Queue vs Synchronous Request]]

---

## 1. Asosiy tushuncha

```text
@nestjs/bullmq  →  BullMQ ustidagi YUPQA o'ram
@nestjs/cache-manager → cache ustidagi o'ram

Ostidagi kutubxonalar:  ioredis, bullmq
                        ↑ Express'da ham AYNAN shular
```

Ya'ni [[Redis]], [[Redis Cache]], [[BullMQ]], [[Queue vs Synchronous Request]] notelaridagi **butun nazariya** Express'ga ham tegishli. Faqat ulanish usuli boshqacha.

## 2. Redis — singleton

```js
// redis.js
const Redis = require('ioredis');

const redis = new Redis(process.env.REDIS_URL);

redis.on('error', (err) => console.error('[redis]', err.message));

module.exports = redis;
```

```js
// services/cache.service.js
const redis = require('../redis');

async function wrap(key, ttlSec, loader) {
  try {
    const cached = await redis.get(key);
    if (cached) return JSON.parse(cached);
  } catch {
    console.warn(`cache read failed: ${key}`);      // ← degrade, crash emas
  }

  const value = await loader();

  try {
    await redis.set(key, JSON.stringify(value), 'EX', ttlSec);
  } catch { /* jim */ }

  return value;
}

module.exports = { wrap, invalidate: (k) => redis.del(k) };
```

Bu kod NestJS `CacheService` bilan **so'zma-so'z bir xil** — faqat u yerda `@Inject('REDIS_CLIENT')` bo'ladi.

## 3. BullMQ — producer va worker

```js
// queues/notifications.queue.js
const { Queue } = require('bullmq');

const connection = {
  url: process.env.REDIS_URL,
  maxRetriesPerRequest: null,          // ← QUEUE uchun majburiy
};

const notificationsQueue = new Queue('notifications', { connection });

module.exports = { notificationsQueue, connection };
```

`maxRetriesPerRequest: null` — bu aynan [[Redis Cache]] da tasvirlangan farq: cache client va queue client bir xil sozlamada ishlay olmaydi.

```js
// producer — service ichida
const { notificationsQueue } = require('../queues/notifications.queue');

async function finish(trialId, dto, userId) {
  const trial = await prisma.$transaction(async (tx) => { ... });

  await notificationsQueue.add(
    'trial-finished',
    { trialId: trial.id },                       // ← kichik payload
    {
      attempts: 5,
      backoff: { type: 'exponential', delay: 2000 },
      removeOnComplete: 1000,
      jobId: `trial-finished:${trial.id}`,       // ← dublikatdan himoya
    },
  );

  return trial;
}
```

```js
// worker.js — ALOHIDA process
const { Worker } = require('bullmq');
const { connection } = require('./queues/notifications.queue');
const prisma = require('./db');
const mailer = require('./services/mailer.service');

const worker = new Worker('notifications', async (job) => {
  if (job.name === 'trial-finished') {
    const trial = await prisma.trial.findUnique({ where: { id: job.data.trialId } });
    if (!trial) return;                          // idempotent
    await mailer.sendTrialFinished(trial);
  }
}, { connection });

worker.on('failed', (job, err) => console.error(`job ${job?.id} failed: ${err.message}`));

process.on('SIGTERM', async () => { await worker.close(); process.exit(0); });
```

```bash
node server.js     # API
node worker.js     # worker — alohida ishga tushiriladi
```

## 4. NestJS bilan solishtirish

| Express | NestJS |
|---|---|
| `new Queue('notifications', {connection})` | `BullModule.registerQueue({ name })` |
| `require('../queues/...')` | `@InjectQueue('notifications')` |
| `new Worker('notifications', fn)` | `@Processor('notifications')` class |
| `worker.on('failed', ...)` | `@OnWorkerEvent('failed')` |
| `node worker.js` alohida | alohida Nest app yoki bir xil process |
| `queue.add(...)` | **aynan bir xil** |
| `attempts`, `backoff`, `jobId` | **aynan bir xil** |

**Butun BullMQ semantikasi o'zgarmagan.** O'zgargani — worker'ni qanday e'lon qilish.

## 5. Rate limit — Redis bilan

```js
const rateLimit = require('express-rate-limit');
const RedisStore = require('rate-limit-redis').default;
const redis = require('./redis');

const limiter = rateLimit({
  windowMs: 60_000,
  max: 100,
  standardHeaders: true,
  store: new RedisStore({ sendCommand: (...args) => redis.call(...args) }),
});

app.use(limiter);
```

Redis store nega kerak — [[Express Rate Limiting]] da batafsil (qisqasi: 3 instance = 3× limit).

## 6. Common mistakes

- **Cache va queue uchun bir xil Redis konfiguratsiyasi** → BullMQ `maxRetriesPerRequest: null` talab qiladi.
- **Worker'ni ishga tushirishni unutish** → joblar `waiting`da to'planadi va hech kim sezmaydi.
- **Butun obyektni payload sifatida yuborish** → eskirgan ma'lumot; ID yubor.
- **Idempotent bo'lmagan job** → retry ikkinchi marta email yuboradi.
- **`queue.add()` ni transaction ichida chaqirish** → rollback bo'lsa ham xabar ketadi ([[Database Transactions]]).
- **`await` ni unutish** (`queue.add(...)` `await`siz) → job jimgina yo'qolishi mumkin.
- **`removeOnComplete` ni sozlamaslik** → Redis xotirasi to'ladi.
- **Cache xatosida requestni yiqitish** → cache ixtiyoriy tezlashtirish bo'lishi kerak.

## 7. Practice problems

### Problem 1 — Redis singleton
`redis.js` yozib, `INCR` + `EXPIRE` bilan oddiy counter yasab ko'r.

### Problem 2 — cache wrap
`wrap()` funksiyasini yozib, og'ir query'ni cache qil.
DB query'ga log qo'yib, ikkinchi requestda chiqmasligini tasdiqla.

### Problem 3 — cache degrade
Redis'ni to'xtat va app baribir ishlashini tekshir. Ishlamasa — tuzat.

### Problem 4 — queue
`notifications` queue yaratib, `POST /notifications` ni background'ga chiqar.
Response vaqtini oldin/keyin o'lchab yoz.

### Problem 5 — worker'siz
Worker'ni ishga tushirma va 5 ta job qo'sh. Ular qayerda? `redis-cli` bilan ko'r.
Keyin worker'ni yoqib, ular bajarilishini kuzat.

### Problem 6 — retry
Job ichida `throw` qilib, `attempts: 3` va exponential backoff'ni loglarda kuzat.

### Problem 7 — idempotentlik
Bir xil `jobId` bilan ikki marta qo'shib, faqat bir marta bajarilishini tasdiqla.

### Problem 8 — NestJS'ga o'gir
Queue'ni `BullModule.registerQueue`, worker'ni `@Processor` ga aylantir.
**Savol:** `queue.add()` chaqiruvi o'zgardimi? Nega yo'q?

## 8. Related

[[Express Prisma Setup]] · [[Express Rate Limiting]] · [[Express Manual Dependency Wiring]] · [[Redis]] · [[Redis Cache]] · [[BullMQ]] · [[Queue vs Synchronous Request]] · [[Database Transactions]]
