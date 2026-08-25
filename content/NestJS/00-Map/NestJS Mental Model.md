---
tags: [nestjs/core, level/beginner, summary]
level: beginner
topic: NestJS Core
---

# NestJS Mental Model

> **Bir jumlada:** NestJS — "class yozish frameworki" emas, **modular application architecture**; har bir concept bitta aniq savolga javob beradi.

**Before learning:** [[NestJS Overview]]
**After learning:** [[NestJS Request Lifecycle]], [[FotSpot Architecture Overview]]

> 💡 **Tushunmadingmi? Express'da qo'lda qanday yozilishini ko'r:** [[Express Mental Model]]


---

## 1. Definition

Butun framework'ni bitta rasmda ushlab turadigan model:

```text
                 NESTJS APPLICATION
                        |
              +---------+---------+
              |                   |
           MODULES             GLOBALS
              |                   |
       +------+-------+      +----+----+
       |              |      |         |
 Controller       Provider   Guards   Filters
       |              |      Pipes    Config
       |              |
       +-------> Service
                     |
          +----------+----------+
          |          |          |
        Prisma     Redis     External
          |          |        Services
          v          v
      PostgreSQL   BullMQ
```

## 2. Why — nega kerak?

Chunki yangi kod ko'rganda savol "bu nima?" emas, **"bu qaysi qutiga tushadi?"** bo'lishi kerak.
Agar javob bera olsang — kod magic bo'lib ko'rinmaydi.

## 3. Internal mental model — har biri bitta jumla

| Concept | Savoli |
|---|---|
| [[NestJS Modules]] | "Bu narsalar qaysi boundary ichida?" |
| [[NestJS Controllers]] | "Requestni qabul qilaman." |
| [[NestJS Services]] | "Business logicni bajaraman." |
| [[NestJS Dependency Injection]] | "Dependency'ni kim yaratadi va kimga beradi?" |
| [[NestJS Guards]] | "Kirish mumkinmi?" |
| [[NestJS Pipes]] | "Input to'g'rimi?" |
| [[NestJS Interceptors]] | "Execution atrofida nima qilaman?" |
| [[NestJS Exception Filters]] | "Xatoni qanday response qilaman?" |
| [[Prisma]] | "Database bilan gaplashaman." |
| [[Redis]] | "Tezkor/shared state saqlayman." |
| [[BullMQ]] | "Ishni backgroundga beraman." |

## 4. FotSpot implementation

FotSpot'da bu model quyidagicha to'ldiriladi:

```text
MODULES   → auth, users, players, coaches, academies, trials,
            recommendations, media, moderation, notifications
GLOBALS   → ThrottleGuard, JwtAuthGuard, RolesGuard, PermissionsGuard,
            ValidationPipe, HttpExceptionFilter, ConfigModule, PrismaModule
INFRA     → Prisma/PostgreSQL, Redis, BullMQ, Cloudflare R2, Sentry
```

## 5. Code example

Bitta feature — butun model kichraytirilgan holda:

```ts
@Module({                                  // boundary
  controllers: [PlayersController],        // HTTP
  providers: [PlayersService],             // logic
})
export class PlayersModule {}

@Controller('players')                     // "requestni qabul qilaman"
export class PlayersController {
  constructor(private readonly players: PlayersService) {}   // DI

  @Get(':id')
  findOne(@Param('id') id: string) {
    return this.players.findOne(id);       // logic service'da
  }
}

@Injectable()                              // "meni DI boshqaradi"
export class PlayersService {
  constructor(private readonly prisma: PrismaService) {}     // infra
}
```

## 6. Common mistakes

- Controller ichida database query yozish → boundary buziladi.
- Har bir narsani `AppModule`ga tiqish → modular architecture yo'qoladi.
- Decoratorlarni "sehr" deb qabul qilish → [[NestJS Decorators]] va [[TypeScript for NestJS]] ni o'qi.
- Infrastructure'ni (Prisma, Redis) NestJS concepti deb o'ylash → [[Domain vs Infrastructure]].

## 7. Practice

Qog'ozga qarab turmasdan yuqoridagi diagrammani chiz va har bir qutiga FotSpot'dan bitta real misol yoz.
Keyin lifecycle'ni yod aytib ber: `Request → Throttle → Jwt → Roles → Permissions → Validation → Controller → Service → Prisma → Response → Filter`.

## 8. Related concepts

[[NestJS Request Lifecycle]] · [[NestJS Learning Map]] · [[Domain vs Infrastructure]] · [[NestJS Interview Questions]]
