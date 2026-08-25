---
tags: [infrastructure, level/intermediate]
level: intermediate
topic: Infrastructure
---

# S3 and Cloudflare R2

> **Bir jumlada:** S3 — protokol, R2 — shu protokolni gapiradigan Cloudflare xizmati; shuning uchun AWS SDK ishlatilishi "biz AWS'damiz" degani emas.

**Before learning:** [[NestJS Providers]], [[ConfigModule and Environment Variables]]
**After learning:** [[Presigned Upload Flow]], [[FotSpot Media Moderation]]

---

## 1. Definition

```text
S3 API protocol
   ├── AWS S3
   ├── Cloudflare R2
   ├── MinIO
   └── boshqa S3-compatible storage
```

Package'lar:

```text
@aws-sdk/client-s3
@aws-sdk/s3-request-presigner
```

## 2. Why — nega kerak?

Chunki S3 API amalda **standart**ga aylangan. Bir marta shu API'ga yozsang:

```text
provayderni almashtirish = endpoint + credentials o'zgarishi
kod o'zgarmaydi
```

R2 tanlanishining amaliy sababi — egress (chiqish trafigi) uchun to'lov modeli, bu media ko'p uzatiladigan loyihada sezilarli farq qiladi.

## 3. Internal mental model

```text
Bucket   → konteyner
Key      → fayl yo'li: "players/<userId>/<uuid>.mp4"
Object   → faylning o'zi + metadata
Presign  → vaqtincha, imzolangan URL
```

Muhim arxitektura tamoyili:

```text
API      = control plane   (ruxsat, key, metadata)
Storage  = data plane      (fayl baytlari)
```

Ya'ni fayl baytlari **NestJS orqali o'tmaydi** — batafsil [[Presigned Upload Flow]].

Key dizayni ham arxitektura qarori:

```text
✓ players/<userId>/<uuid>.mp4     → egasi ko'rinadi, nom to'qnashmaydi
✗ uploads/video.mp4               → ustiga yozib yuborilishi mumkin
✗ foydalanuvchi bergan fayl nomi   → path traversal xavfi
```

## 4. FotSpot implementation

```text
StorageService
 ├── S3Client (R2 endpoint bilan)
 ├── getUploadUrl(key, contentType)   → presigned PUT
 ├── getDownloadUrl(key)              → presigned GET (private obyektlar uchun)
 └── deleteObject(key)
```

Credentiallar env orqali keladi va hech qachon kodda bo'lmaydi ([[ConfigModule and Environment Variables]]).

Yuklangan media darhol ochiq bo'lmaydi — u moderatsiyadan o'tadi: [[FotSpot Media Moderation]].

## 5. Code example

```ts
@Injectable()
export class StorageService {
  private readonly client: S3Client;
  private readonly bucket: string;

  constructor(private readonly config: ConfigService) {
    this.bucket = config.getOrThrow<string>('R2_BUCKET');
    this.client = new S3Client({
      region: 'auto',                                        // R2 uchun
      endpoint: config.getOrThrow<string>('R2_ENDPOINT'),
      credentials: {
        accessKeyId: config.getOrThrow<string>('R2_ACCESS_KEY_ID'),
        secretAccessKey: config.getOrThrow<string>('R2_SECRET_ACCESS_KEY'),
      },
    });
  }

  buildKey(userId: string, ext: string) {
    return `players/${userId}/${randomUUID()}.${ext}`;
  }

  getUploadUrl(key: string, contentType: string, expiresIn = 300) {
    return getSignedUrl(
      this.client,
      new PutObjectCommand({ Bucket: this.bucket, Key: key, ContentType: contentType }),
      { expiresIn },
    );
  }
}
```

## 6. Common mistakes

- **"AWS SDK ishlatyapmiz, demak AWS'damiz" deb o'ylash** → bu faqat protokol.
- **`region`ni R2 uchun noto'g'ri berish** → `'auto'` bo'lishi kerak.
- **Bucket'ni public qilib qo'yish** → moderatsiyadan o'tmagan media internetga chiqadi.
- **Credentiallarni kodga yozish** yoki frontendga berish.
- **Foydalanuvchi bergan fayl nomini key sifatida ishlatish** → to'qnashuv va path traversal.
- **Fayl o'chirilganda DB yozuvini (yoki aksincha) tozalamaslik** → yetim obyektlar.
- **Content-Type va hajmni cheklamaslik** → bucket fayl saqlash xizmatiga aylanadi.

## 7. Practice

1. `StorageService` yozib, presigned PUT URL qaytar.
2. `curl -X PUT --upload-file` bilan faylni to'g'ridan-to'g'ri yukla.
3. URL muddati tugagach qayta urin — nima bo'ladi?
4. Key strukturasini `players/<userId>/<uuid>` qilib, ikkita fayl to'qnashmasligini tasdiqla.
5. Bucket'ni private qoldirib, o'qish uchun presigned GET yasab ko'r.

## 8. Related concepts

[[Presigned Upload Flow]] · [[FotSpot Media Moderation]] · [[ConfigModule and Environment Variables]] · [[NestJS Providers]] · [[Domain vs Infrastructure]]
