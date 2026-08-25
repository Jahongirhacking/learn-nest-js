---
tags: [auth, level/intermediate, security]
level: intermediate
topic: Authorization
---

# Public Routes Decorator

> **Bir jumlada:** `@Public()` — global authentication'dan **ataylab qilingan istisno**; arxitektura tamoyili: *default = secure, exception = explicit*.

**Before learning:** [[NestJS Guards]], [[NestJS Custom Decorators]], [[Passport and JwtStrategy]]
**After learning:** [[Authorization]], [[FotSpot Guard Chain]]

> 💡 **Tushunmadingmi? Express'da qo'lda qanday yozilishini ko'r:** [[Express Auth Middleware]]


---

## 1. Definition

```ts
export const IS_PUBLIC_KEY = 'isPublic';
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);
```

```ts
@Public()
@Post('login')
login(@Body() dto: LoginDto) {}
```

## 2. Why — nega kerak?

Ikki xil yondashuv bor:

```text
A) Default ochiq, kerakli joyga @UseGuards qo'yamiz
   → bitta endpointda unutilsa — teshik ochiq qoladi (jimgina)

B) Default yopiq, ochiq joyga @Public() qo'yamiz
   → unutilsa — endpoint 401 qaytaradi (shovqinli, darhol ko'rinadi)
```

FotSpot **B** ni tanlagan. Xato qilganda "jimgina teshik" emas, "shovqinli 401" bo'lishi — xavfsiz dizayn tamoyili (*fail closed*).

## 3. Internal mental model

```text
JwtAuthGuard global
        ↓
@Public() metadata bormi?
   ├── yo'q  → token majburiy
   └── ha    → token bo'lmasa ham o'tadi
              lekin token bo'lsa — user aniqlanadi
```

Oxirgi qatorga e'tibor ber. Bu **optional authentication**:

```text
Guest         → public endpoint → ruxsat
Authenticated → xuddi shu endpoint → ruxsat + request.user mavjud
```

Buning amaliy foydasi: bitta `GET /players` endpointi guest uchun ham ishlaydi, login qilgan user uchun esa qo'shimcha kontekst beradi ("saqlanganlar", "sizga tavsiya").

## 4. FotSpot implementation

Public endpointlar odatda:

```text
POST /auth/register
POST /auth/login
POST /auth/refresh
POST /auth/otp/request
GET  /health
GET  /players          (guest ham ko'radi, lekin faqat ACTIVE + VERIFIED)
```

Oxirgisi muhim: public bo'lishi "hamma ma'lumot ochiq" degani emas. Ko'rinish qoidasi baribir service'da qo'llanadi — [[FotSpot Media Moderation]].

Muhim: `@Public()` faqat **authentication**ni chetlab o'tadi. `ThrottleGuard` baribir ishlaydi — aks holda login endpointi brute force uchun ochiq qolardi ([[Rate Limiting and Throttling]]).

## 5. Code example

Guard tomoni — [[Passport and JwtStrategy]] dagi `JwtAuthGuard` ga qara.

Controller tomoni:

```ts
@Controller('players')
export class PlayersController {
  @Public()
  @Get()
  findAll(@Query() q: ListPlayersDto, @CurrentUser() user?: JwtUser) {
    // user bo'lishi ham, bo'lmasligi ham mumkin
    return this.players.findAllPublic(q, user?.id);
  }

  @Get('me/drafts')            // @Public() yo'q → token majburiy
  drafts(@CurrentUser() user: JwtUser) {
    return this.players.findDrafts(user.id);
  }
}
```

## 6. Common mistakes

- **`@Public()` ni butun controllerga qo'yish** → bitta method uchun kerak bo'lgan istisno butun feature'ni ochib yuboradi.
- **Public endpointda `@CurrentUser()` ni `undefined` bo'lmaydi deb o'ylash** → guest kelganda crash.
- **`@Public()` ni "hech qanday tekshiruv yo'q" deb tushunish** → throttle, validation, ko'rinish qoidalari baribir ishlaydi.
- **Public endpointda maxfiy maydonlarni qaytarish** (email, telefon) → ma'lumot sizishi.
- **Metadata kalitini guard va decorator'da har xil yozish** → `@Public()` jimgina ishlamay qoladi. Konstanta ishlat.

## 7. Practice

1. `@Public()` decorator + `JwtAuthGuard` integratsiyasini yoz.
2. `GET /players` ni public qil va tokensiz chaqir — ishladimi?
3. Endi token bilan chaqir va `request.user` mavjudligini logla.
4. `@Public()` ni olib tashla — 401 kelishini tasdiqla.
5. Public endpointdan email maydonini olib tashlab, response'ni tozala.

## 8. Related concepts

[[NestJS Custom Decorators]] · [[Passport and JwtStrategy]] · [[NestJS Guards]] · [[Authorization]] · [[Rate Limiting and Throttling]] · [[FotSpot Guard Chain]]
