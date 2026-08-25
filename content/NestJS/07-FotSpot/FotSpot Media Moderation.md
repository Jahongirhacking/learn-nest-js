---
tags: [fotspot, level/intermediate, domain]
level: intermediate
topic: FotSpot
---

# FotSpot Media Moderation

> **Bir jumlada:** yuklangan media avtomatik ravishda ochiq bo'lmaydi — u holat mashinasidan (`UNVERIFIED → VERIFIED / BLOCKED`) o'tadi, va public ko'rinish qoidasi `ACTIVE + VERIFIED`.

**Before learning:** [[Presigned Upload Flow]], [[Authorization]]
**After learning:** [[BullMQ]], [[Domain vs Infrastructure]]

---

## 1. Definition

```text
Upload
  ↓
UNVERIFIED
  ├──→ uploader o'z mediasini ko'radi
  ↓
Admin moderation
  ├──→ VERIFIED   → public ko'rinadi
  └──→ BLOCKED    → ko'rinmaydi
```

Public visibility business qoidasi:

```text
ACTIVE (profil holati)  +  VERIFIED (media holati)
```

## 2. Why — nega kerak?

Platformada yosh o'yinchilar videosi bo'lgani uchun moderatsiya majburiy:

```text
✓ nomaqbul kontent ochiq chiqmasin
✓ mualliflik huquqi buzilmasin
✓ platforma obro'si saqlansin
```

Texnik jihatdan esa bu **default deny** naqshi: kontent yangi bo'lsa — ko'rinmaydi, ochish uchun aniq harakat kerak. Xuddi [[Public Routes Decorator]] dagi mantiq.

## 3. Internal mental model

Holat mashinasi:

```text
UNVERIFIED ──verify──→ VERIFIED ──block──→ BLOCKED
     │                                        │
     └────────────block──────────────────────┘

BLOCKED → VERIFIED  (qayta ko'rib chiqish, admin qarori bilan)
```

NestJS'ning bu jarayondagi roli:

```text
1. upload authorization      → kim yuklay oladi          ([[Authorization]])
2. metadata saqlash          → key, owner, contentType   ([[Prisma]])
3. moderation endpointlari   → faqat admin              ([[Roles and Permissions]])
4. database state transition → holatlar o'tishi
5. public filtering          → so'rovlarda majburiy filtr
```

Eng oson unutiladigan qadam — beshinchisi.

## 4. Implementation

```text
media/
 ├── media.controller.ts     → upload-url, complete, my media
 ├── media.service.ts        → key, metadata, visibility qoidasi
 └── dto/

moderation/
 ├── moderation.controller.ts → verify / block  (admin)
 └── moderation.service.ts    → holat o'tishi + audit
```

Ruxsat:

```text
@Roles('admin')
@Permissions('admin:moderate_video')
```

## 5. Code example

```ts
// public ko'rinish qoidasi BITTA joyda
@Injectable()
export class MediaService {
  private publicWhere(): Prisma.MediaWhereInput {
    return { status: 'VERIFIED', owner: { status: 'ACTIVE' } };
  }

  listPublic(playerId: string) {
    return this.prisma.media.findMany({
      where: { ...this.publicWhere(), playerId },
      select: { id: true, key: true, createdAt: true },
    });
  }

  listOwn(userId: string) {
    return this.prisma.media.findMany({ where: { ownerId: userId } });   // hammasi
  }
}
```

```ts
// moderatsiya qarori — audit bilan
async verify(mediaId: string, adminId: string) {
  return this.prisma.$transaction(async (tx) => {
    const media = await tx.media.update({
      where: { id: mediaId },
      data: { status: 'VERIFIED', moderatedBy: adminId, moderatedAt: new Date() },
    });
    await tx.moderationLog.create({
      data: { mediaId, adminId, action: 'VERIFY' },
    });
    return media;
  });
}
```

## 6. Common mistakes

- **Public filtrni har bir query'da qo'lda yozish** → bitta joyda unutiladi va moderatsiyadan o'tmagan media chiqadi. Yagona helper ishlat.
- **Storage bucket'ini public qilib qo'yish** → key topilsa, DB filtri ahamiyatsiz bo'lib qoladi ([[S3 and Cloudflare R2]]).
- **Moderatsiya qarorini loglamaslik** → kim, qachon, nega tasdiqlagani ma'lum bo'lmaydi.
- **BLOCKED mediani storage'dan hech qachon o'chirmaslik** → xarajat va huquqiy masala; siyosat belgilangan bo'lsin.
- **Uploader'ga o'z mediasini ko'rsatmaslik** → foydalanuvchi tajribasi buziladi (u ko'rishi kerak, boshqalar yo'q).
- **Holatlarni `boolean` bilan modellashtirish** (`isVerified`) → uchinchi holat kerak bo'lganda schema o'zgaradi; enum ishlat.

## 7. Practice

1. `Media` modelini enum status bilan yoz ([[Prisma Schema]]).
2. `publicWhere()` helperini yozib, barcha public query'larda ishlat.
3. Admin uchun `verify` / `block` endpointlarini permission bilan himoyala.
4. `moderationLog` qo'shib, audit izini yarat.
5. Uploader o'z `UNVERIFIED` mediasini ko'ra olishini, boshqa user ko'ra olmasligini test qil.

## 8. Related concepts

[[Presigned Upload Flow]] · [[S3 and Cloudflare R2]] · [[Authorization]] · [[Roles and Permissions]] · [[Prisma Schema]] · [[Database Transactions]] · [[FotSpot Architecture Overview]]
