---
tags: [nestjs/core, level/beginner]
level: beginner
topic: NestJS Core
---

# NestJS Decorators

> **Bir jumlada:** Decorator — class, method yoki parametrga **metadata** yopishtiradigan mexanizm; NestJS shu metadata'ni o'qib route, DI va guard qarorlarini quradi.

**Before learning:** [[TypeScript for NestJS]]
**After learning:** [[NestJS Custom Decorators]], [[NestJS Guards]], [[NestJS Controllers]]

> 💡 **Tushunmadingmi? Express'da qo'lda qanday yozilishini ko'r:** [[Express Routing]]


---

## 1. Definition

```ts
@Module()   @Controller()   @Injectable()
@Get()      @Post()         @Patch()   @Delete()
@Body()     @Param()        @Query()   @Headers()
@UseGuards() @UseInterceptors() @UseFilters()
```

Hammasi bitta ishni qiladi: **metadata yozadi**. Ular sehr emas.

## 2. Why — nega kerak?

Chunki NestJS'ga konfiguratsiya kerak: qaysi class controller, qaysi method `GET /players/:id`, qaysi endpointga guard kerak.
Buni alohida config faylda yozish o'rniga, decorator konfiguratsiyani **kodning yoniga** qo'yadi.

## 3. Internal mental model

```text
@Controller('players')
        ↓
Reflect.defineMetadata('path', 'players', PlayersController)
        ↓
startup'da Nest metadata'ni o'qiydi
        ↓
Express route registratsiyasi: GET /players/:id → handler
```

Ya'ni:

```text
Decorator = metadata yozuvchi
Framework = metadata o'quvchi
```

`Reflector` — metadata o'qiydigan Nest servisi; guardlar aynan shundan foydalanadi:

```ts
const roles = this.reflector.getAllAndOverride<string[]>('roles', [
  context.getHandler(),
  context.getClass(),
]);
```

Bu — [[NestJS Guards]] va [[Public Routes Decorator]] ishlashining butun siri.

## 4. FotSpot implementation

FotSpot'da uch xil decorator ishlatiladi:

```text
1. Framework decoratorlari   → @Controller, @Get, @Body, @UseGuards
2. Kutubxona decoratorlari   → @IsString, @IsEmail (class-validator)
                              → @ApiTags, @ApiOperation (Swagger)
3. O'z decoratorlari          → @Public(), @Roles(), @Permissions(), @CurrentUser()
```

Uchinchisi eng qiziq — [[NestJS Custom Decorators]].

## 5. Code example

```ts
@ApiTags('players')                       // Swagger metadata
@Controller('players')                    // route prefix
export class PlayersController {

  @Roles('admin')                         // custom metadata → RolesGuard o'qiydi
  @UseGuards(RolesGuard)                  // guard biriktirish
  @ApiOperation({ summary: 'Create player' })
  @HttpCode(201)                          // status code
  @Post()
  create(
    @Body() dto: CreatePlayerDto,         // body → DTO
    @CurrentUser() user: JwtUser,         // custom param decorator
  ) {
    return this.service.create(dto, user.id);
  }
}
```

Metadata yozish va o'qish — to'liq zanjir:

```ts
// yozish
export const Roles = (...roles: string[]) => SetMetadata('roles', roles);

// o'qish
const roles = this.reflector.get<string[]>('roles', context.getHandler());
```

## 6. Common mistakes

- **`@Injectable()` yoki `@Controller()` ni unutish** → Nest classni ko'rmaydi.
- **`emitDecoratorMetadata` / `experimentalDecorators` o'chiq bo'lishi** (`tsconfig.json`) → DI umuman ishlamaydi.
- **Decorator tartibini muhim deb o'ylash** — ko'p hollarda muhim emas, lekin bir xil metadata kalitini ikki marta yozsang, oxirgisi ustun keladi.
- **`SetMetadata` kalitini string sifatida turli joyda har xil yozish** (`'roles'` va `'role'`) → guard hech qachon topmaydi. Konstanta chiqar: `export const ROLES_KEY = 'roles'`.
- **`@Param('id')` o'rniga `@Param()` ishlatib, obyektni string kutish.**

## 7. Practice

1. `SetMetadata` bilan `@Roles()` yoz, guard ichida `Reflector` orqali o'qi.
2. `context.getHandler()` va `context.getClass()` farqini sinab ko'r: metadata'ni class darajasida qo'yib, method darajasida o'qishga urin.
3. `tsconfig.json`da `emitDecoratorMetadata`ni o'chirib, DI qanday buzilishini ko'r (keyin qaytar).
4. O'z `@ApiOperation`ingga o'xshash oddiy metadata decorator yoz.

## 8. Related concepts

[[NestJS Custom Decorators]] · [[NestJS Guards]] · [[Public Routes Decorator]] · [[TypeScript for NestJS]] · [[NestJS Controllers]]
