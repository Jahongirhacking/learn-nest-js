---
tags: [infrastructure, level/intermediate]
level: intermediate
topic: Infrastructure
---

# BullMQ

> **Bir jumlada:** BullMQ — Redis ustidagi navbat tizimi: uzoq davom etadigan ishni HTTP requestdan ajratib, background worker'ga beradi.

**Before learning:** [[Redis]], [[NestJS Services]]
**After learning:** [[Queue vs Synchronous Request]], [[WebSockets in NestJS]]

> 💡 **Tushunmadingmi? Express'da qo'lda qanday yozilishini ko'r:** [[Express Redis and Queues]]


---

## 1. Definition

```text
HTTP Request
    ↓
Queue.add(job)
    ↓
Redis
    ↓
Worker (alohida process bo'lishi mumkin)
    ↓
Long-running task
```

Asosiy tushunchalar:

```text
Queue    → job qo'shiladigan joy
Job      → bajariladigan vazifa + ma'lumot (payload)
Worker   → jobni oladigan va bajaradigan process
Retry    → xato bo'lsa qayta urinish
Backoff  → qayta urinishlar orasidagi kutish (odatda exponential)
```

## 2. Why — nega kerak?

Yomon:

```text
POST /trials/:id/finish
   ├── database update
   ├── email yuborish           (2s)
   ├── SMS yuborish             (1s)
   ├── notification yaratish
   └── statistikani qayta hisoblash (3s)
   ↓
response  ← 6+ soniyadan keyin
```

Yaxshi:

```text
POST /trials/:id/finish
   ├── database update
   └── queue.add('trial-finished', { trialId })
   ↓
fast response  ← ~50ms

Worker (fon)
   ├── email
   ├── SMS
   └── statistika
```

Batafsil solishtirish: [[Queue vs Synchronous Request]]

## 3. Internal mental model

Job hayoti:

```text
waiting → active → completed
                 ↘ failed → (retry, backoff) → active
                          ↘ failed (attempts tugadi) → dead
```

Muhim tamoyillar:

```text
Idempotentlik  → bitta job ikki marta bajarilsa ham zarar bo'lmasin
Kichik payload → butun obyektni emas, ID'ni yubor (worker DB'dan o'qiydi)
Retry + backoff → tashqi servis vaqtincha yiqilsa, o'zi tiklanadi
Dead letter    → ko'p marta yiqilgan joblarni saqlab, keyin ko'rib chiqish
```

Redis konfiguratsiyasi cache'nikidan farq qiladi:

```ts
{ maxRetriesPerRequest: null, enableReadyCheck: false }
```

Sababi — [[Redis Cache]].

## 4. FotSpot implementation

```text
notifications queue → email / SMS / Telegram xabarlari
media queue         → yuklangan media bilan bog'liq fon ishlari
cleanup jobs        → eskirgan sessiyalar, vaqtinchalik fayllar
```

Naqsh:

```text
Service → queue.add()  (transaction TASHQARISIDA → [[Database Transactions]])
Worker  → Processor    (xuddi shu domain service'ni chaqiradi)
```

## 5. Code example

```ts
// module
BullModule.forRootAsync({
  inject: [ConfigService],
  useFactory: (config: ConfigService) => ({
    connection: {
      url: config.getOrThrow<string>('REDIS_URL'),
      maxRetriesPerRequest: null,       // ← queue-specific
    },
  }),
}),
BullModule.registerQueue({ name: 'notifications' }),
```

```ts
// producer
@Injectable()
export class NotificationsService {
  constructor(@InjectQueue('notifications') private readonly queue: Queue) {}

  async enqueueTrialFinished(trialId: string) {
    await this.queue.add(
      'trial-finished',
      { trialId },                        // ← kichik payload
      {
        attempts: 5,
        backoff: { type: 'exponential', delay: 2000 },
        removeOnComplete: 1000,
        removeOnFail: 5000,
        jobId: `trial-finished:${trialId}`,   // ← dublikatdan himoya
      },
    );
  }
}
```

```ts
// consumer
@Processor('notifications')
export class NotificationsProcessor extends WorkerHost {
  private readonly logger = new Logger(NotificationsProcessor.name);

  constructor(private readonly mailer: MailerService, private readonly prisma: PrismaService) {
    super();
  }

  async process(job: Job) {
    switch (job.name) {
      case 'trial-finished': {
        const trial = await this.prisma.trial.findUnique({ where: { id: job.data.trialId } });
        if (!trial) return;                  // idempotent: yo'q bo'lsa jim chiqamiz
        await this.mailer.sendTrialFinished(trial);
        return;
      }
    }
  }

  @OnWorkerEvent('failed')
  onFailed(job: Job, err: Error) {
    this.logger.error(`Job ${job.id} failed: ${err.message}`);
  }
}
```

## 6. Common mistakes

- **Butun obyektni payload sifatida yuborish** → eskirgan ma'lumot bilan ishlash; ID yubor.
- **Idempotent bo'lmagan job** → retry ikkinchi marta email yuboradi.
- **Retry/backoff qo'ymaslik** → tashqi servis 1 soniya yiqilsa, job butunlay yo'qoladi.
- **`removeOnComplete` ni sozlamaslik** → Redis xotirasi to'ladi.
- **Xatolarni yutib yuborish** → job "muvaffaqiyatli" hisoblanadi va muammo yashirin qoladi.
- **Job'ni transaction ichida qo'shish** → rollback bo'lsa ham xabar ketadi ([[Database Transactions]]).
- **Worker'ni ishga tushirishni unutish** → joblar `waiting`da to'planib qoladi.

## 7. Practice

1. `notifications` queue yaratib, `POST /notifications` requestini background'ga chiqar.
2. Job ichida `throw` qilib, retry va backoff'ni loglarda kuzat.
3. `jobId` bilan bir xil jobni ikki marta qo'shib, dublikat bo'lmasligini tasdiqla.
4. Worker'ni to'xtatib, joblar `waiting`da to'planishini ko'r; keyin yoqib, ular bajarilishini kuzat.
5. Bull Board yoki `redis-cli` bilan navbat holatini kuzat.

## 8. Related concepts

[[Queue vs Synchronous Request]] · [[Redis]] · [[Redis Cache]] · [[Database Transactions]] · [[NestJS Services]] · [[Logging in NestJS]]
