---
tags: [infrastructure, level/intermediate]
level: intermediate
topic: Infrastructure
---

# Redis Cache

> **Bir jumlada:** cache uchun Redis va queue uchun Redis **bir xil narsa emas** — chunki cache yo'qolsa request davom etadi, job yo'qolsa business jarayoni buziladi.

**Before learning:** [[Redis]]
**After learning:** [[BullMQ]], [[NestJS Interceptors]]

> 💡 **Tushunmadingmi? Express'da qo'lda qanday yozilishini ko'r:** [[Express Redis and Queues]]


---

## 1. Definition

```text
Cache miss  → request davom etadi (sekinroq, lekin to'g'ri)
Job yo'qolsa → xabar yuborilmaydi, hisob-kitob bajarilmaydi
```

Bu farq konfiguratsiyaga ham ta'sir qiladi — FotSpot'dagi muhim arxitektura qarori.

## 2. Why — nega kerak?

Savol shunday qo'yiladi:

> "Redis nima uchun kerak?" emas, balki **"Redis'dan foydalanish semantikasi qanday?"**

```text
Cache client:
   xato bo'lsa → jimgina o'tib ketish MUMKIN (degrade gracefully)
   qayta urinish → agressiv bo'lishi shart emas

Queue client (BullMQ):
   xato bo'lsa → jimgina o'tib bo'lMAYDI
   BullMQ bloklovchi buyruqlar ishlatadi (BRPOPLPUSH)
   → maxRetriesPerRequest: null kabi queue-specific sozlama kerak
```

## 3. Internal mental model

Cache naqshi — **cache-aside**:

```text
1. Redis'dan o'qi
2. bor bo'lsa   → qaytar
3. yo'q bo'lsa  → DB'dan o'qi → Redis'ga TTL bilan yoz → qaytar
```

Nimani cache qilish kerak:

```text
✓ tez-tez o'qiladigan + kam o'zgaradigan   (role/permission ro'yxati, sozlamalar)
✓ hisoblash qimmat bo'lgan natijalar        (reyting, tavsiya)
✗ har user uchun har safar boshqacha ma'lumot
✗ pul, balans, sessiya holati kabi aniqlik talab qiladigan narsa
```

Eng qiyin savol — **invalidation**:

```text
ma'lumot o'zgardi → cache'ni o'chir (yoki yangi qiymat yoz)
TTL              → xato invalidation'ni ham "o'z-o'zidan tuzatadi"
```

Shuning uchun TTL doim bo'lishi kerak, hatto invalidation yozilgan bo'lsa ham.

## 4. FotSpot implementation

Cache uchun tabiiy nomzodlar:

```text
user permissions        → har bir requestda og'ir JOIN bo'lmasligi uchun ([[RBAC]])
public player ro'yxati  → tez-tez o'qiladi, kam o'zgaradi
statik ma'lumotnomalar  → position, region ro'yxatlari
```

Queue tomoni butunlay boshqa hikoya — [[BullMQ]].

## 5. Code example

```ts
@Injectable()
export class CacheService {
  constructor(@Inject('REDIS_CLIENT') private readonly redis: Redis) {}
  private readonly logger = new Logger(CacheService.name);

  async wrap<T>(key: string, ttlSec: number, loader: () => Promise<T>): Promise<T> {
    try {
      const cached = await this.redis.get(key);
      if (cached) return JSON.parse(cached) as T;
    } catch (e) {
      this.logger.warn(`Cache read failed: ${key}`);   // ← degrade, crash emas
    }

    const value = await loader();

    try {
      await this.redis.set(key, JSON.stringify(value), 'EX', ttlSec);
    } catch { /* jimgina o'tamiz */ }

    return value;
  }

  async invalidate(key: string) {
    await this.redis.del(key);
  }
}
```

```ts
// ishlatish
getPermissions(userId: string) {
  return this.cache.wrap(`perm:${userId}`, 300, () =>
    this.rbac.loadPermissions(userId),
  );
}

// role o'zgarganda
await this.cache.invalidate(`perm:${userId}`);
```

## 6. Common mistakes

- **Cache xatosida butun requestni yiqitish** → cache "ixtiyoriy tezlashtirish" bo'lishi kerak.
- **TTL'siz cache** → eskirgan ma'lumot abadiy yashaydi.
- **Invalidation'ni unutish** → user role'ni oldi, lekin 1 soat davomida eski huquq bilan ishlayveradi.
- **Kalit nomlarida konvensiya yo'qligi** → `perm:user:<id>` kabi prefiksli tizim tuz.
- **Cache va queue uchun bir xil client/config** → BullMQ o'z sozlamasini talab qiladi.
- **Xavfsizlikka bog'liq qarorni faqat cache'ga suyanib qilish** → permission cache TTL'i qisqa bo'lsin.

## 7. Practice

1. `wrap()` methodini yozib, `getPermissions` ni cache qil.
2. DB query'ga `console.log` qo'yib, ikkinchi requestda chiqmasligini ko'r.
3. Role o'zgartirib, invalidation'siz nima bo'lishini kuzat — keyin invalidation qo'sh.
4. Redis'ni to'xtatib, application baribir ishlashini tasdiqla (degrade).
5. TTL'ni 10 soniya qilib, eskirish jarayonini kuzat.

## 8. Related concepts

[[Redis]] · [[BullMQ]] · [[RBAC]] · [[NestJS Interceptors]] · [[Rate Limiting and Throttling]]
