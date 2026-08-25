---
tags: [auth, level/intermediate, security]
level: intermediate
topic: Authentication
---

# OTP Security

> **Bir jumlada:** OTP — bir martalik kod, va u ham parol kabi **hash holida** saqlanishi kerak; qolgan xavfsizlik esa muddat, urinishlar soni va rate limit orqali ta'minlanadi.

**Before learning:** [[Password Hashing]], [[Authentication]]
**After learning:** [[Rate Limiting and Throttling]], [[Redis]]

---

## 1. Definition

OTP (One-Time Password) — SMS yoki emailga yuboriladigan qisqa muddatli kod.

```text
OTP generatsiya → hash → database
                ↓
        plain kod faqat userga yuboriladi
```

Tekshiruv:

```text
user kiritgan kod → hash/verify → saqlangan hash
```

## 2. Why — nega kerak?

OTP'ni plain saqlash — parolni plain saqlash bilan bir xil xato:

```text
DB leak bo'lsa → hujumchi hozirda faol barcha OTP'larni o'qiydi
                → istalgan hisobga kiradi
```

Shuning uchun FotSpot'dagi konsept:

```text
OTP
 ↓
hash
 ↓
database
```

## 3. Internal mental model

OTP xavfsizligi **to'rt** ustunga tayanadi — bittasi yetarli emas:

```text
1. Hash          → DB leak'da kodlar o'qilmasin
2. Qisqa TTL     → 5 daqiqadan keyin o'lik (masalan)
3. Urinish limiti → 3-5 marta noto'g'ri kiritilsa bekor qilinadi
4. Rate limit     → so'rash ham, tekshirish ham cheklangan
```

Nega hammasi kerak? 6 xonali kod = 1 000 000 variant. Limitsiz bo'lsa, avtomatik skript uni bir necha daqiqada topadi.

Bir martalik: muvaffaqiyatli tekshiruvdan keyin OTP darhol **ishlatilgan** deb belgilanishi kerak (replay himoyasi).

## 4. FotSpot implementation

```text
POST /auth/otp/request     → kod generatsiya, hash saqlash, SMS/email yuborish
POST /auth/otp/verify      → hash solishtirish → session yaratish
```

Bog'liq qatlamlar:

```text
[[Rate Limiting and Throttling]] → so'rovlar soni cheklanadi (ThrottleGuard global)
[[Redis]]                        → counter va TTL uchun qulay
[[Session Architecture]]         → tasdiqdan keyin session ochiladi
```

## 5. Code example

```ts
@Injectable()
export class OtpService {
  private readonly TTL_MS = 5 * 60 * 1000;
  private readonly MAX_ATTEMPTS = 5;

  constructor(private readonly prisma: PrismaService) {}

  async request(userId: string, channel: 'sms' | 'email') {
    const code = String(randomInt(100_000, 999_999));       // crypto random!
    const codeHash = await argon2.hash(code);

    await this.prisma.otpCode.create({
      data: {
        userId,
        codeHash,
        channel,
        expiresAt: new Date(Date.now() + this.TTL_MS),
      },
    });

    await this.notifications.send(channel, userId, code);    // plain faqat shu yerda
    return { sent: true };                                   // kodni javobga QO'SHMA
  }

  async verify(userId: string, code: string) {
    const otp = await this.prisma.otpCode.findFirst({
      where: { userId, usedAt: null, expiresAt: { gt: new Date() } },
      orderBy: { createdAt: 'desc' },
    });
    if (!otp) throw new UnauthorizedException('Invalid or expired code');

    if (otp.attempts >= this.MAX_ATTEMPTS) {
      throw new UnauthorizedException('Too many attempts');
    }

    if (!(await argon2.verify(otp.codeHash, code))) {
      await this.prisma.otpCode.update({
        where: { id: otp.id },
        data: { attempts: { increment: 1 } },
      });
      throw new UnauthorizedException('Invalid or expired code');   // bir xil matn
    }

    await this.prisma.otpCode.update({
      where: { id: otp.id },
      data: { usedAt: new Date() },        // bir martalik
    });
    return true;
  }
}
```

## 6. Common mistakes

- **OTP'ni response'da qaytarish** (development qulayligi uchun) va uni production'da unutib qoldirish.
- **`Math.random()` ishlatish** → bashorat qilinadigan kod. `crypto.randomInt` ishlat.
- **TTL qo'ymaslik** → kod abadiy yashaydi.
- **Urinishlar sonini cheklamaslik** → brute force.
- **Ishlatilgan OTP'ni bekor qilmaslik** → replay hujumi.
- **"Kod noto'g'ri" va "kod eskirgan" ni ajratib aytish** → hujumchiga qo'shimcha ma'lumot.
- **OTP so'rash endpointini throttle qilmaslik** → SMS bombing va pul yo'qotish.

## 7. Practice

1. OTP request/verify flow'ini yoz: hash, TTL, attempts.
2. Kodni logga chiqarib development'da sinab ko'r, keyin **olib tashla**.
3. 6 marta noto'g'ri kod kiritib, bloklanishini tekshir.
4. Bir marta ishlatilgan kodni qayta yubor — rad etilishi kerak.
5. OTP so'rash endpointiga throttle qo'yib, 429 javobini ko'r.

## 8. Related concepts

[[Password Hashing]] · [[Authentication]] · [[Session Architecture]] · [[Rate Limiting and Throttling]] · [[Redis]] · [[FotSpot Auth Architecture]]
