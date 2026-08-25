---
tags: [database, level/intermediate, infrastructure]
level: intermediate
topic: Database
---

# Prisma

> **Bir jumlada:** Prisma — type-safe ORM; u **NestJS'ning qismi emas**, NestJS uni shunchaki [[NestJS Dependency Injection]] orqali arxitektura ichiga joylashtiradi.

**Before learning:** [[NestJS Services]], [[NestJS Dependency Injection]]
**After learning:** [[Prisma Schema]], [[Prisma Migrations]], [[Database Transactions]]

> 💡 **Tushunmadingmi? Express'da qo'lda qanday yozilishini ko'r:** [[Express Prisma Setup]]


---

## 1. Definition

```text
@prisma/client   → generatsiya qilingan, typed client
prisma           → CLI (migrate, generate, studio)
PostgreSQL       → FotSpot ishlatadigan database
```

Arxitektura:

```text
Controller → Service → PrismaService → Prisma Client → PostgreSQL
```

## 2. Why — nega kerak?

Prisma uch narsani beradi:

```text
1. Type safety     → schema'dan TypeScript tiplar generatsiya qilinadi
2. Migration       → schema o'zgarishi versiyalanadi
3. O'qiladigan query API  → include, select, where
```

Eng qimmatlisi — birinchisi: `prisma.user.findUnique({ where: { emial: x } })` deb yozsang, **compile bosqichida** xato chiqadi.

## 3. Internal mental model

```text
schema.prisma       ← yagona haqiqat manbai
      ↓ prisma generate
Prisma Client (typed)
      ↓ prisma migrate
PostgreSQL sxemasi
```

NestJS bilan bog'lanish nuqtasi — `PrismaService`:

```ts
@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit, OnModuleDestroy {
  async onModuleInit() { await this.$connect(); }
  async onModuleDestroy() { await this.$disconnect(); }
}
```

`OnModuleInit` / `OnModuleDestroy` — NestJS lifecycle hook'lari; ular ulanishni application hayoti bilan bog'laydi (graceful shutdown).

## 4. FotSpot implementation

`PrismaModule` **global**:

```ts
@Global()
@Module({
  providers: [PrismaService],
  exports: [PrismaService],
})
export class PrismaModule {}
```

Shuning uchun har bir feature moduli `PrismaModule`ni alohida import qilmasdan `PrismaService`ni inject qila oladi.

Muhim tushuncha:

> Prisma NestJS'ning o'zi emas. NestJS Prisma'ni DI orqali application arxitekturasi ichiga joylashtiradi.

Bu farqni [[Domain vs Infrastructure]] da davom ettir.

## 5. Code example

```ts
@Injectable()
export class PlayersService {
  constructor(private readonly prisma: PrismaService) {}

  findAll(page = 1, take = 20) {
    return this.prisma.playerProfile.findMany({
      where: { status: 'ACTIVE' },
      include: { user: { select: { id: true, firstName: true } } },
      orderBy: { createdAt: 'desc' },
      skip: (page - 1) * take,
      take,
    });
  }

  async findOne(id: string) {
    const player = await this.prisma.playerProfile.findUnique({ where: { id } });
    if (!player) throw new NotFoundException('Player not found');
    return player;
  }
}
```

`select` va `include` farqi:

```text
select  → aynan qaysi ustunlar kerak (maxfiy maydonlarni kesish uchun)
include → bog'langan modelni ham olib kelish
```

## 6. Common mistakes

- **Controller'dan to'g'ridan-to'g'ri Prisma chaqirish** → qatlam buziladi ([[NestJS Controllers]]).
- **N+1 query**: har bir element uchun alohida `findUnique` chaqirish. Yechim: `include` yoki bitta `findMany` + `in`.
- **`select` ishlatmaslik** → `passwordHash` kabi maydonlar response'ga sizadi.
- **Har bir service'da `new PrismaClient()`** → connection pool portlaydi. Faqat bitta `PrismaService`.
- **`$disconnect()` ni unutish** → graceful shutdown ishlamaydi.
- **Prisma xatolarini ushlamaslik** → `P2002` (unique) 500 bo'lib chiqadi ([[NestJS Exception Filters]]).
- **`$queryRaw` ni parametrsiz yozish** → SQL injection.

## 7. Practice

1. `PrismaService` va global `PrismaModule` yoz.
2. `findAll` da `select` bilan faqat kerakli ustunlarni ol.
3. N+1 query yozib, keyin uni `include` bilan tuzat va query loglarini solishtir.
4. `P2002` xatosini 409 ga aylantiruvchi filter qo'sh.
5. `prisma studio` ochib, ma'lumotlarni ko'r.

## 8. Related concepts

[[Prisma Schema]] · [[Prisma Migrations]] · [[Database Transactions]] · [[Database Relations]] · [[Database Connection URLs]] · [[Domain vs Infrastructure]] · [[NestJS Dependency Injection]]
