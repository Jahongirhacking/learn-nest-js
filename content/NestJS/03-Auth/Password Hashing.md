---
tags: [auth, level/intermediate, security]
level: intermediate
topic: Authentication
---

# Password Hashing

> **Bir jumlada:** parol hech qachon qaytarib bo'ladigan shaklda saqlanmaydi — faqat **bir tomonlama hash** saqlanadi va login'da hash solishtiriladi.

**Before learning:** [[Authentication]]
**After learning:** [[OTP Security]], [[Session Architecture]]

> 💡 **Tushunmadingmi? Express'da qo'lda qanday yozilishini ko'r:** [[Express Authentication]]


---

## 1. Definition

FotSpot **argon2** ishlatadi.

```text
Ro'yxatdan o'tish:
plain password → Argon2 → passwordHash → database

Login:
input password → Argon2.verify(hash, input) → true / false
```

Parolni "decrypt qilish" degan tushuncha **yo'q**.

## 2. Why — nega kerak?

Chunki database ertami-kechmi sizib chiqishi mumkin. Savol shu: sizib chiqqanda nima bo'ladi?

```text
plain saqlangan   → barcha parollar darhol o'g'irlandi
                     (va userlar boshqa saytlarda ham shu parolni ishlatgan)
MD5/SHA1          → GPU bilan soniyalar ichida ochiladi
bcrypt/argon2     → har bir parolni ochish qimmat va sekin
```

Argon2 **ataylab sekin** va **xotira talab qiladigan** algoritm — bu GPU bilan parallel hujumni qimmatlashtiradi.

## 3. Internal mental model

```text
hash ≠ encryption
   encryption → kalit bilan qaytariladi
   hash       → qaytarilmaydi

salt → har bir parol uchun tasodifiy qo'shimcha
   → bir xil parollar bir xil hash bermaydi
   → rainbow table hujumi ishlamaydi

argon2 salt'ni hash satrining ichida saqlaydi:
   $argon2id$v=19$m=65536,t=3,p=4$<salt>$<hash>
```

Shuning uchun alohida `salt` ustuni kerak emas.

## 4. FotSpot implementation

```text
User.passwordHash   → argon2 hash
Session.refreshTokenHash → refresh token ham hashlanadi
OTP hash            → OTP ham plain saqlanmaydi
```

Naqsh bitta va takrorlanadi:

> **Foydalanuvchini tasdiqlaydigan har qanday sir database'da hash holida yotadi.**

[[JWT Authentication]], [[OTP Security]], [[Session Architecture]] — hammasi shu qoidaga bo'ysunadi.

## 5. Code example

```ts
import * as argon2 from 'argon2';

@Injectable()
export class AuthService {
  async register(dto: RegisterDto) {
    const passwordHash = await argon2.hash(dto.password, {
      type: argon2.argon2id,
    });

    return this.prisma.user.create({
      data: { email: dto.email, passwordHash },
      select: { id: true, email: true },        // ← hash hech qachon qaytmaydi
    });
  }

  async validate(email: string, password: string) {
    const user = await this.prisma.user.findUnique({ where: { email } });

    if (!user || !(await argon2.verify(user.passwordHash, password))) {
      throw new UnauthorizedException('Invalid credentials');
    }
    return user;
  }
}
```

## 6. Common mistakes

- **MD5 / SHA256 ishlatish** → tez algoritmlar parol uchun mos emas.
- **O'z "hashing" algoritmini yozish** → deyarli har doim zaif chiqadi.
- **`passwordHash`ni response'ga qo'shib yuborish** → Prisma `select`/`@Exclude()` bilan kes ([[NestJS Interceptors]]).
- **Parol uzunligini tekshirmaslik** → `@MinLength(8)` DTO'da ([[NestJS DTO]]).
- **Login javobida "email topilmadi" deb aytish** → user enumeration ([[Authentication]]).
- **Hashni logga yozish** yoki xato matnida ko'rsatish.
- **Login endpointini throttle qilmaslik** → [[Rate Limiting and Throttling]].

## 7. Practice

1. `register` da argon2 hash yasab saqla; DB'da qanday ko'rinishini o'z ko'zing bilan ko'r.
2. Bir xil parolni ikki marta hashlab, natijalar **har xil** ekanini tasdiqla. Nega?
3. `login` da `argon2.verify` bilan tekshir.
4. Response'da `passwordHash` chiqmasligini `select` orqali ta'minla va testda tekshir.
5. Login endpointiga rate limit qo'y va 10 ta noto'g'ri urinishdan keyin 429 kelishini ko'r.

## 8. Related concepts

[[Authentication]] · [[OTP Security]] · [[Session Architecture]] · [[JWT Authentication]] · [[Rate Limiting and Throttling]] · [[NestJS DTO]]
