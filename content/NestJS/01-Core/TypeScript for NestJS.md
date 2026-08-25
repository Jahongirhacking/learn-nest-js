---
tags: [typescript, level/beginner, foundation]
level: beginner
topic: Foundation
---

# TypeScript for NestJS

> **Bir jumlada:** NestJS "magic" bo'lib ko'rinsa — sabab ko'pincha NestJS emas, TypeScript conceptlarining yetishmasligi.

**Before learning:** JavaScript (async/await, class, module)
**After learning:** [[NestJS Overview]], [[NestJS Decorators]], [[NestJS Dependency Injection]]

---

## 1. Definition

NestJS TypeScript'ning uchta imkoniyatiga jiddiy suyanadi:

```text
1. class + decorator          → metadata
2. constructor parameter type  → DI resolution
3. access modifier (private readonly) → qisqa injection sintaksisi
```

## 2. Why — nega kerak?

Bu kod:

```ts
constructor(private readonly prisma: PrismaService) {}
```

aslida uchta ish qiladi:

```text
1. parametr qabul qiladi
2. this.prisma sifatida saqlaydi (access modifier tufayli)
3. PrismaService tipini runtime metadata sifatida qoldiradi
   (emitDecoratorMetadata) — Nest shundan qaysi provider kerakligini biladi
```

Uchinchisini bilmasang, DI abadiy sehr bo'lib qoladi.

## 3. Internal mental model

Kerakli minimal ro'yxat:

```text
class · interface · type · enum
generic (<T>)              → Promise<Player[]>, Repository<T>
union / literal type       → 'ACTIVE' | 'BLOCKED'
access modifier            → private / readonly
decorator                  → @Injectable()
async / await, Promise
utility types              → Partial<T>, Pick<T>, Omit<T>
```

Muhim tushuncha:

```text
TypeScript type  = faqat compile-time
DTO + validator  = runtime tekshiruv
```

Shuning uchun `firstName: string` runtime'da hech narsani himoya qilmaydi — [[NestJS DTO]] kerak bo'ladi.

## 4. FotSpot implementation

```text
enum         → UserRole, MediaStatus (Prisma enum'lari TS enum sifatida keladi)
generic      → Promise<PlayerProfile | null>
Partial<T>   → update DTO'lar (PartialType)
type guard   → user contextni tekshirish
strict mode  → null/undefined xatolarini compile bosqichida ushlash
```

Prisma alohida yutuq beradi: schema'dan **generatsiya qilingan tiplar** — `prisma.user.findUnique()` natijasi to'liq typed ([[Prisma]]).

## 5. Code example

```ts
// generic + union type
type MediaStatus = 'UNVERIFIED' | 'VERIFIED' | 'BLOCKED';

interface Paginated<T> {
  items: T[];
  total: number;
  page: number;
}

@Injectable()
export class PlayersService {
  constructor(private readonly prisma: PrismaService) {}   // ← DI + access modifier

  async findAll(page = 1): Promise<Paginated<PlayerProfile>> {
    const [items, total] = await this.prisma.$transaction([
      this.prisma.playerProfile.findMany({ skip: (page - 1) * 20, take: 20 }),
      this.prisma.playerProfile.count(),
    ]);
    return { items, total, page };
  }
}

// PartialType — update DTO
export class UpdatePlayerDto extends PartialType(CreatePlayerDto) {}
```

## 6. Common mistakes

- **`any` ishlatish** → TypeScript'ning butun foydasi yo'qoladi va DI metadata ham buziladi.
- **Type'ni validation deb o'ylash** → eng ko'p uchraydigan yangi boshlovchi xatosi. Runtime'da type yo'q.
- **`strict: false`** → `null` xatolari production'da chiqadi.
- **Interface'ni DI token sifatida ishlatish** → interface runtime'da mavjud emas; abstract class yoki string token ishlat ([[NestJS Providers]]).

## 7. Practice

1. `tsconfig.json`da `strict: true` qo'y va chiqqan barcha xatolarni tuzat.
2. `Paginated<T>` generic tipini yozib, ikkita turli service'da ishlat.
3. DTO'da `firstName: string` yoz, lekin validator **qo'shma** — keyin `{"firstName": 123}` yubor. Nima bo'ladi? Nega?
4. `PartialType` bilan update DTO yasab, farqni ko'r.

## 8. Related concepts

[[NestJS Decorators]] · [[NestJS Dependency Injection]] · [[NestJS DTO]] · [[Prisma]] · [[ESLint and Prettier]]
