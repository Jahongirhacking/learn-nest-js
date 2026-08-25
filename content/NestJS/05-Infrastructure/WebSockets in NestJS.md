---
tags: [infrastructure, level/advanced]
level: advanced
topic: Infrastructure
---

# WebSockets in NestJS

> **Bir jumlada:** REST — "so'ra va javob ol", WebSocket — "ulanish ochiq turadi va server o'zi xabar yubora oladi".

**Before learning:** [[NestJS Modules]], [[Authentication]], [[BullMQ]]
**After learning:** production scaling, Redis adapter

---

## 1. Definition

```text
@nestjs/websockets
@nestjs/platform-socket.io
socket.io
```

```text
REST:       Client → request → response  (har safar yangi)
WebSocket:  Client ←──── doimiy ulanish ────→ Server
```

## 2. Why — nega kerak?

Real-time featurelar uchun:

```text
notification         → "sizga yangi tavsiya keldi"
live events          → trial jarayonidagi yangilanishlar
status updates       → media moderatsiyadan o'tdi
real-time UI updates
```

Alternativa — polling (`har 5 soniyada so'rash`): oddiy, lekin isrofgar va sekin.

## 3. Internal mental model

NestJS'da WebSocket ham xuddi HTTP kabi modular:

```text
Gateway     ≈ Controller   (@WebSocketGateway)
@SubscribeMessage ≈ @Get/@Post
Service     — aynan o'sha service ishlatiladi
Guard/Pipe  — ishlaydi, lekin context boshqacha (switchToWs)
Room        → foydalanuvchi/guruh bo'yicha adreslash
```

Ikki muhim tafsilot:

```text
1. Authentication qo'lda qilinadi — handshake paytida token tekshiriladi
   (Authorization header o'rniga auth payload / query)

2. Bir nechta instance bo'lsa, Redis adapter kerak:
   instance A dagi emit → instance B dagi clientga yetib borishi uchun
```

Ikkinchisi — eng ko'p unutiladigan production muammosi.

## 4. FotSpot implementation

```text
NotificationsGateway
   ├── handleConnection  → JWT tekshiriladi, user o'z room'iga qo'shiladi
   ├── room: `user:<userId>`
   └── emit('notification', payload)
```

Manba odatda background ish bo'ladi:

```text
BullMQ worker → notification yaratdi → gateway.emitToUser(userId, payload)
```

Ya'ni [[BullMQ]] va WebSocket birga ishlaydi: og'ir ish fon'da bajariladi, natija real-time yetkaziladi.

## 5. Code example

```ts
@WebSocketGateway({ namespace: '/notifications', cors: { origin: allowedOrigins } })
export class NotificationsGateway implements OnGatewayConnection {
  @WebSocketServer() server: Server;

  constructor(private readonly jwt: JwtService, private readonly config: ConfigService) {}

  async handleConnection(client: Socket) {
    try {
      const token = client.handshake.auth?.token as string;
      const payload = await this.jwt.verifyAsync(token, {
        secret: this.config.getOrThrow<string>('JWT_SECRET'),
      });
      client.data.userId = payload.sub;
      await client.join(`user:${payload.sub}`);
    } catch {
      client.disconnect(true);          // ← autentifikatsiyasiz ulanish yopiladi
    }
  }

  emitToUser(userId: string, event: string, payload: unknown) {
    this.server.to(`user:${userId}`).emit(event, payload);
  }
}
```

## 6. Common mistakes

- **Ulanishda authentication qilmaslik** → har kim ulanib, boshqalarning xabarlarini eshitadi.
- **Room ishlatmasdan `server.emit()` qilish** → xabar **hamma**ga ketadi (ma'lumot sizishi).
- **Bir nechta instance'da Redis adapter'siz ishlatish** → xabarlar tasodifiy yetib bormaydi.
- **WebSocket'ni ishonchli yetkazish kanali deb o'ylash** → ulanish uzilishi mumkin; muhim xabarlar DB'da ham saqlansin.
- **Business logikani gateway ichiga yozish** → xuddi controller kabi, gateway ham yupqa bo'lishi kerak ([[NestJS Services]]).
- **CORS/origin'ni ochiq qoldirish** → [[CORS]].

## 7. Practice

1. `NotificationsGateway` yozib, JWT bilan himoyalangan ulanish qil.
2. `user:<id>` room'iga qo'shib, faqat o'sha userga xabar yubor.
3. Tokensiz ulanishga urinib, uzilishini tasdiqla.
4. BullMQ worker'dan gateway orqali xabar yubor.
5. Ikkita instance ishga tushirib, Redis adapter'siz muammoni ko'r, keyin adapter qo'sh.

## 8. Related concepts

[[BullMQ]] · [[Authentication]] · [[Redis]] · [[NestJS Services]] · [[CORS]] · [[NestJS Modules]]
