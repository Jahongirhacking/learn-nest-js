---
tags: [infrastructure, level/intermediate]
level: intermediate
topic: Infrastructure
---

# Redis

> **Bir jumlada:** Redis — database emas, **memory-oriented data store**: PostgreSQL haqiqat manbai bo'lsa, Redis tezkor va vaqtinchalik umumiy holat uchun.

**Before learning:** [[NestJS Providers]], [[Prisma]]
**After learning:** [[Redis Cache]], [[BullMQ]], [[Rate Limiting and Throttling]]

> 💡 **Tushunmadingmi? Express'da qo'lda qanday yozilishini ko'r:** [[Express Redis and Queues]]


---

## 1. Definition

FotSpot'da Redis uch maqsadda:

```text
Redis
 ├── cache            → [[Redis Cache]]
 ├── rate limiting    → [[Rate Limiting and Throttling]]
 └── BullMQ backend   → [[BullMQ]]
```

## 2. Why — nega kerak?

```text
PostgreSQL = source of truth  (ishonchli, doimiy, ACID)
Redis      = fast shared state (tez, vaqtinchalik, TTL bilan)
```

Ikki aniq sabab:

1. **Tezlik** — xotiradan o'qish diskdan o'qishdan sezilarli tez.
2. **Umumiylik** — app 3 ta instance'da ishlayotgan bo'lsa, ular holatni **birga** ko'rishi kerak. Node process ichidagi `Map` buni bera olmaydi.

Ikkinchisi ko'pincha e'tibordan chetda qoladi: rate limit counter'ni xotirada saqlasang, 3 instance = 3 barobar ko'p ruxsat.

## 3. Internal mental model

Redis'ning asosiy imkoniyatlari:

```text
key → value          GET / SET
TTL                  SET key val EX 60      ← o'z-o'zidan o'chadi
atomic counter       INCR                    ← rate limit uchun
list / stream        queue backend
pub/sub              instancelar orasida xabar
```

Nima Redis'ga mos:

```text
✓ cache
✓ counter, rate limit
✓ session/otp holati (TTL bilan)
✓ queue
✗ asosiy business ma'lumot (u PostgreSQL'da)
```

Muhim: Redis xotirada ishlaydi va qayta ishga tushganda ma'lumot yo'qolishi mumkin. Yo'qolsa **business buzilmasligi kerak** — bu qoidadan istisno queue'lar ([[Redis Cache]] da davomi).

## 4. FotSpot implementation

```text
Cache            → tez-tez o'qiladigan, kam o'zgaradigan ma'lumot
Rate limiting    → ThrottleGuard hisoblagichlari
BullMQ           → notification, email, background jobs
```

Ulanish `REDIS_URL` orqali, `ConfigService`dan ([[ConfigModule and Environment Variables]]).

Muhim arxitektura qarori: **cache uchun Redis va queue uchun Redis bir xil semantikaga ega emas** — [[Redis Cache]].

## 5. Code example

```ts
@Global()
@Module({
  providers: [
    {
      provide: 'REDIS_CLIENT',
      useFactory: (config: ConfigService) =>
        new Redis(config.getOrThrow<string>('REDIS_URL')),
      inject: [ConfigService],
    },
  ],
  exports: ['REDIS_CLIENT'],
})
export class RedisModule implements OnModuleDestroy {
  constructor(@Inject('REDIS_CLIENT') private readonly redis: Redis) {}
  async onModuleDestroy() { await this.redis.quit(); }
}
```

```ts
// atomik counter + TTL
const key = `otp:req:${userId}`;
const count = await this.redis.incr(key);
if (count === 1) await this.redis.expire(key, 3600);
if (count > 5) throw new TooManyRequestsException();
```

`INCR` atomik — ikkita instance bir vaqtda chaqirsa ham hisob to'g'ri qoladi.

## 6. Common mistakes

- **Redis'ni database sifatida ishlatish** → ma'lumot yo'qolishi mumkin.
- **TTL qo'ymaslik** → xotira to'lib ketadi va Redis eski kalitlarni chiqarib tashlay boshlaydi.
- **`KEYS *` ishlatish** → production'da butun Redis'ni bloklaydi. `SCAN` ishlat.
- **Har bir service'da yangi `new Redis()`** → connection portlashi.
- **`onModuleDestroy` da `quit()` qilmaslik** → graceful shutdown ishlamaydi.
- **Cache va queue uchun bir xil konfiguratsiya** → [[Redis Cache]].
- **Katta obyektlarni cache qilish** → tarmoq va xotira narxi foydadan oshadi.

## 7. Practice

1. `RedisModule`ni `useFactory` bilan yoz.
2. `INCR` + `EXPIRE` yordamida oddiy counter yasab, TTL tugashini kuzat.
3. Bitta kalitni TTL bilan yozib, `TTL key` bilan qolgan vaqtni ko'r.
4. Ikkita app instance ishga tushirib, counter umumiy ekanini tasdiqla.
5. `redis-cli MONITOR` bilan qanday buyruqlar ketayotganini kuzat.

## 8. Related concepts

[[Redis Cache]] · [[BullMQ]] · [[Rate Limiting and Throttling]] · [[NestJS Providers]] · [[ConfigModule and Environment Variables]] · [[Domain vs Infrastructure]]
