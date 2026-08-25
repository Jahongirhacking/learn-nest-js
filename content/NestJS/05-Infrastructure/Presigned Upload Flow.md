---
tags: [infrastructure, level/intermediate, architecture]
level: intermediate
topic: Infrastructure
---

# Presigned Upload Flow

> **Bir jumlada:** API fayl baytlarini o'zidan o'tkazmaydi — u faqat **ruxsat beradi** va vaqtincha imzolangan URL qaytaradi: *API — control plane, storage — data plane*.

**Before learning:** [[S3 and Cloudflare R2]], [[Authorization]]
**After learning:** [[FotSpot Media Moderation]], [[BullMQ]]

---

## 1. Definition

```text
Client
  │  1. upload URL so'raydi
  ▼
NestJS API
  │  2. ruxsat tekshiradi, key yasaydi, presigned URL qaytaradi
  ▼
Client
  │  3. PUT fayl  (to'g'ridan-to'g'ri)
  ▼
Cloudflare R2
  │
  ▼
Client → 4. API'ga "yuklandi" deb xabar beradi → metadata saqlanadi
```

## 2. Why — nega kerak?

Agar fayl API orqali o'tsa:

```text
✗ 100MB video Node.js process xotirasidan o'tadi
✗ request uzoq ochiq qoladi (timeout, memory)
✗ API instancelarini fayl hajmi uchun masshtablash kerak
✗ bandwidth ikki barobar (client→API→storage)
```

Presigned URL bilan:

```text
✓ API faqat bir necha millisekundlik ish qiladi
✓ baytlar to'g'ridan-to'g'ri storage'ga ketadi
✓ API masshtabi fayl hajmiga bog'liq emas
```

## 3. Internal mental model

API nima qiladi va nima qilmaydi:

```text
API QILADI:
  ✓ authentication / authorization tekshiradi
  ✓ key yaratadi (kim, qayerga yuklashi mumkin — server hal qiladi)
  ✓ content-type va hajm chegarasini imzoga kiritadi
  ✓ qisqa muddatli presigned URL beradi
  ✓ metadata saqlaydi (DB)

API QILMAYDI:
  ✗ fayl baytlarini qabul qilmaydi
```

Xavfsizlik nuqtalari:

```text
key'ni SERVER yasaydi        → client "kimningdir papkasiga" yozolmaydi
expiresIn qisqa (5-15 daq)   → URL tarqalsa ham tez o'ladi
ContentType imzoga kiradi    → boshqa turdagi fayl yuklab bo'lmaydi
DB status: PENDING → UPLOADED → moderatsiya
```

## 4. FotSpot implementation

```text
POST /media/upload-url
   → ruxsat tekshiriladi
   → key: players/<userId>/<uuid>.mp4
   → presigned PUT URL + mediaId qaytadi
   → DB: Media { status: UNVERIFIED, key, ownerId }

Client → PUT → R2

POST /media/:id/complete
   → API obyekt haqiqatan mavjudligini tekshiradi (HeadObject)
   → DB yangilanadi
   → [[BullMQ]] job: moderatsiya uchun tayyorlash
```

Yuklangandan keyin ham media darhol ochiq bo'lmaydi — [[FotSpot Media Moderation]].

## 5. Code example

```ts
@Controller('media')
export class MediaController {
  constructor(private readonly media: MediaService) {}

  @Post('upload-url')
  createUploadUrl(@Body() dto: CreateUploadUrlDto, @CurrentUser('id') userId: string) {
    return this.media.createUploadUrl(dto, userId);
  }
}
```

```ts
@Injectable()
export class MediaService {
  private readonly ALLOWED = ['video/mp4', 'image/jpeg', 'image/png'];

  async createUploadUrl(dto: CreateUploadUrlDto, userId: string) {
    if (!this.ALLOWED.includes(dto.contentType)) {
      throw new BadRequestException('Unsupported content type');
    }

    const key = this.storage.buildKey(userId, extFor(dto.contentType));

    const media = await this.prisma.media.create({
      data: { ownerId: userId, key, contentType: dto.contentType, status: 'UNVERIFIED' },
      select: { id: true },
    });

    const uploadUrl = await this.storage.getUploadUrl(key, dto.contentType, 300);
    return { mediaId: media.id, uploadUrl, expiresIn: 300 };
  }
}
```

## 6. Common mistakes

- **Key'ni clientdan qabul qilish** → boshqa userning papkasiga yozish mumkin bo'ladi.
- **`expiresIn` ni uzoq qo'yish** (bir kun) → URL tarqalsa, uzoq muddat ochiq.
- **ContentType'ni imzoga kiritmaslik** → `.mp4` deb aytib, boshqa narsa yuklanadi.
- **"Complete" qadamini tekshirmasdan ishonish** → DB'da bor, storage'da yo'q holatlar.
- **Yuklangan faylni darhol public qilish** → moderatsiya chetlab o'tiladi.
- **Yarim qolgan (PENDING) yozuvlarni tozalamaslik** → axlat to'planadi (cron/queue job kerak).
- **Fayl hajmini cheklamaslik** → `ContentLengthRange` bilan chegara qo'y.

## 7. Practice

1. `POST /media/upload-url` yozib, presigned URL qaytar.
2. `curl -X PUT` bilan faylni yukla, keyin `complete` chaqir.
3. Key'ni clientdan olishga urin va nega bu xavfli ekanini yozib ol.
4. `expiresIn: 10` qilib, muddat tugagach yuklashga urin.
5. `HeadObject` bilan fayl haqiqatan yuklanganini tekshiradigan qadamni qo'sh.

## 8. Related concepts

[[S3 and Cloudflare R2]] · [[FotSpot Media Moderation]] · [[Authorization]] · [[BullMQ]] · [[NestJS DTO]]
