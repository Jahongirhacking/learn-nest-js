---
tags: [auth, level/intermediate]
level: intermediate
topic: Authentication
---

# Session Architecture

> **Bir jumlada:** JWT stateless — shuning uchun "logout", "qurilmani chiqarish" va "barcha sessiyalarni yopish" uchun database'da **Session** modeli kerak bo'ladi.

**Before learning:** [[JWT Authentication]], [[Password Hashing]]
**After learning:** [[FotSpot Auth Architecture]], [[Database Relations]]

> 💡 **Tushunmadingmi? Express'da qo'lda qanday yozilishini ko'r:** [[Express Authentication]]


---

## 1. Definition

```text
User
 └── Session
       ├── refreshTokenHash
       ├── deviceId
       ├── userAgent
       ├── ipAddress
       ├── expiresAt
       └── revokedAt
```

Har bir qurilma = bitta Session yozuvi.

## 2. Why — nega kerak?

JWT o'z-o'zidan bekor qilinmaydi. Ya'ni:

```text
"Logout" tugmasini bosdim → access token baribir amal qiladi (exp gacha)
```

Session modeli aynan shu bo'shliqni to'ldiradi:

```text
✓ device tracking            → "qaysi qurilmalardan kirilgan?"
✓ refresh token rotation     → har refreshda yangi token
✓ individual session revoke  → "bu telefonni chiqarib yubor"
✓ logout specific device
✓ revoke all sessions        → parol o'zgarganda hammasini yop
```

## 3. Internal mental model

```text
Access token   → qisqa umrli, stateless, tekshirish arzon (DB'siz)
Refresh token  → uzoq umrli, stateful, Session yozuvi bilan bog'langan
```

Refresh flow (rotation bilan):

```text
POST /auth/refresh
   ↓
Session topiladi (deviceId / token id bo'yicha)
   ↓
argon2.verify(session.refreshTokenHash, kelgan token)
   ↓
revokedAt == null va expiresAt > now  tekshiriladi
   ↓
yangi access + yangi refresh chiqariladi
   ↓
eski refresh hash almashtiriladi (rotation)
```

**Reuse detection**: agar allaqachon ishlatilgan (eski) refresh token qaytib kelsa — bu token o'g'irlangan degan kuchli signal. To'g'ri javob: o'sha user'ning **barcha** sessiyalarini bekor qilish.

## 4. FotSpot implementation

```text
User
 ├── Session 1  (telefon)
 ├── Session 2  (noutbuk)
 └── Session 3  (planshet)
```

Bu **multi-device authentication** naqshi. Foydalanuvchi bir qurilmadan chiqsa, boshqalari ishlashda davom etadi.

`refreshTokenHash` — plain emas, xuddi [[Password Hashing]] dagidek.

## 5. Code example

```prisma
model Session {
  id               String    @id @default(uuid())
  userId           String
  refreshTokenHash String
  deviceId         String?
  userAgent        String?
  ipAddress        String?
  expiresAt        DateTime
  revokedAt        DateTime?
  createdAt        DateTime  @default(now())

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([userId])
  @@index([userId, revokedAt])
}
```

```ts
async logoutAll(userId: string) {
  await this.prisma.session.updateMany({
    where: { userId, revokedAt: null },
    data: { revokedAt: new Date() },
  });
}

async listSessions(userId: string) {
  return this.prisma.session.findMany({
    where: { userId, revokedAt: null },
    select: { id: true, userAgent: true, ipAddress: true, createdAt: true },
  });
}
```

## 6. Common mistakes

- **Refresh tokenni plain saqlash** → DB leak = barcha hisoblar egallanadi.
- **Rotation qilmaslik** → o'g'irlangan refresh abadiy ishlaydi.
- **Reuse detection'ni qo'shmaslik** → o'g'irlanganini hech qachon bilmaysan.
- **Eskirgan sessiyalarni tozalamaslik** → jadval cheksiz o'sadi (cron yoki [[BullMQ]] job kerak).
- **Parol o'zgarganda sessiyalarni bekor qilmaslik** → "parolni o'zgartirdim" hujumchini chiqarib yubormaydi.
- **`userId` ustuniga index qo'ymaslik** → har bir refresh sekinlashadi ([[Prisma Schema]]).

## 7. Practice

1. `Session` modelini yozib migration qil.
2. Login'da session yarat, refresh hash'ini saqla.
3. `POST /auth/refresh` da rotation qil.
4. `GET /auth/sessions` va `DELETE /auth/sessions/:id` yoz.
5. Parol o'zgarganda barcha sessiyalarni bekor qiluvchi mantiqni qo'sh.
6. Eskirgan sessiyalarni tozalaydigan background job yoz ([[BullMQ]]).

## 8. Related concepts

[[JWT Authentication]] · [[Authentication]] · [[Password Hashing]] · [[Database Relations]] · [[FotSpot Auth Architecture]] · [[BullMQ]]
