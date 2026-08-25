---
tags: [fotspot, level/intermediate, architecture]
level: intermediate
topic: Architecture
---

# Domain vs Infrastructure

> **Bir jumlada:** domain — loyihaning **nima uchun** borligi, infrastructure — buni **qanday** amalga oshirayotgani; ikkalasini ajratish katta backend arxitekturasining asosi.

**Before learning:** [[NestJS Services]], [[Prisma]]
**After learning:** [[FotSpot Feature Anatomy]], [[Testing in NestJS]]

> 💡 **Tushunmadingmi? Express'da qo'lda qanday yozilishini ko'r:** [[Express Project Structure Problem]]


---

## 1. Definition

```text
Domain / business          Infrastructure
─────────────────          ──────────────
Players                    Prisma
Trials                     PostgreSQL
Recommendations            Redis
Academies                  BullMQ
Coaches                    Cloudflare R2
Moderation                 Sentry
                           Email / SMS / Telegram
```

```text
TrialsService   → business logic
PrismaService   → infrastructure
```

## 2. Why — nega kerak?

Uch amaliy sabab:

```text
1. Almashtirish
   R2 → S3 ga o'tish domain kodiga tegmasligi kerak

2. Test qilish
   Business qoidani test qilish uchun real DB shart emas ([[Testing in NestJS]])

3. Tushunish
   Yangi dasturchi "biz nima qilamiz" ni "qanday saqlaymiz" dan ajratib ko'radi
```

## 3. Internal mental model

Savol bilan tekshir:

```text
"Bu class biznesga tegishlimi yoki texnologiyagami?"

TrialsService.finishTrial()   → biznes: "trial tugadi, tavsiya yangilanadi"
PrismaService                 → texnologiya: "ma'lumotni PostgreSQL'da saqlayman"
StorageService                → texnologiya: "faylni R2'ga qo'yaman"
RecommendationsService        → biznes: "scout tavsiyasi qanday baholanadi"
```

Yo'nalish qoidasi:

```text
Domain  →  Infrastructure'ni ISHLATADI     ✓
Infrastructure  →  Domain'ni BILMAYDI       ✓
```

`PrismaService` hech qachon `TrialsService`ni import qilmasligi kerak.

## 4. FotSpot implementation

```text
backend/src/
  # infrastructure
  prisma/  redis/  storage/  rate-limit/  common/

  # domain
  users/  players/  coaches/  academies/
  trials/ recommendations/ media/ moderation/ notifications/

  # chegarada turadi
  auth/   → domain (kim kirishi mumkin) + infrastructure (JWT, Passport)
```

`auth/` ikkalasiga ham tegishi normal — muhimi, buni **bilib** turish.

## 5. Code example

```ts
// ✓ Domain infrastructure'ni interfeys orqali ishlatadi
@Injectable()
export class MediaService {
  constructor(
    private readonly prisma: PrismaService,     // infra
    private readonly storage: StorageService,   // infra
  ) {}

  async requestUpload(dto: CreateUploadUrlDto, userId: string) {
    // business qoida: kim, nimani, qanday sharoitda yuklay oladi
    if (!this.isAllowedType(dto.contentType)) {
      throw new BadRequestException('Unsupported content type');
    }
    const key = this.storage.buildKey(userId, extFor(dto.contentType));
    ...
  }
}
```

```ts
// ✗ infrastructure domainni bilyapti — noto'g'ri yo'nalish
@Injectable()
export class PrismaService extends PrismaClient {
  async createTrialAndNotify(...) {}     // bu yerda business logic bo'lmasligi kerak
}
```

## 6. Common mistakes

- **Prisma modelni domain modeli deb hisoblash** → database sxemasi business qoidalarini boshqarib ketadi.
- **Business qoidani `PrismaService` yoki `StorageService` ichiga yozish** → yo'nalish teskari.
- **Domain service ichida to'g'ridan-to'g'ri `fetch()` yoki SDK chaqirish** → almashtirib bo'lmaydi va test qilish qiyin.
- **Hamma narsani interfeys ortiga yashirish** → kerak bo'lmagan abstraksiya ham zarar (YAGNI).
- **`common/` papkasini axlat qutisiga aylantirish** → u ham struktura talab qiladi.

## 7. Practice

1. O'z loyihangdagi har bir service'ni ikki ustunga ajratib yoz: domain / infrastructure.
2. Chegarada turgan classlarni alohida belgila va nega shundayligini yoz.
3. `StorageService`ni abstract class qilib, ikkita implementatsiya (local, R2) yozib ko'r.
4. Domain service'ni real DB'siz unit test qil ([[Testing in NestJS]]).

## 8. Related concepts

[[FotSpot Architecture Overview]] · [[FotSpot Feature Anatomy]] · [[NestJS Services]] · [[Prisma]] · [[NestJS Modules]] · [[Testing in NestJS]]
