---
tags: [production, level/intermediate]
level: intermediate
topic: Production
---

# Logging in NestJS

> **Bir jumlada:** `console.log` — development odati; production'da log daraja, kontekst va **maxfiy ma'lumotdan tozalik** bilan yoziladi.

**Before learning:** [[NestJS Application Bootstrap]]
**After learning:** [[Sentry]], [[NestJS Interceptors]]

> 💡 **Tushunmadingmi? Express'da qo'lda qanday yozilishini ko'r:** [[Express Logging]]


---

## 1. Definition

```ts
private readonly logger = new Logger(PlayersService.name);

this.logger.log('Player created');
this.logger.warn('Cache unavailable');
this.logger.error('Failed to send email', err.stack);
this.logger.debug('Query params', JSON.stringify(query));
```

`Logger.name` konteksti logda `[PlayersService]` bo'lib chiqadi.

## 2. Why — nega kerak?

Production'da savol shunday bo'ladi:

> "Kecha soat 14:32 da bu user nima qildi va nega 500 oldi?"

Bunga javob berish uchun log **tuzilgan** bo'lishi kerak: daraja, kontekst, vaqt, request id.

## 3. Internal mental model

Darajalar va ularning ma'nosi:

```text
error   → ishlamay qoldi, kimdir ko'rishi kerak
warn    → g'alati, lekin ishlashda davom etdik (cache miss, retry)
log     → muhim business hodisa (user ro'yxatdan o'tdi)
debug   → development uchun tafsilotlar
verbose → juda mayda tafsilotlar
```

Muhit bo'yicha:

```text
Development:  error, warn, log, debug, verbose
Production:   error, warn, log
```

Bu shunchaki "kamroq yozamiz" degani emas — production loglari **shovqinsiz** bo'lishi kerak, aks holda haqiqiy xato ko'rinmay qoladi.

## 4. FotSpot implementation

FotSpot startup'da log darajalari `NODE_ENV` ga qarab o'rnatiladi ([[NestJS Application Bootstrap]]).

Qatlamlar bo'linishi:

```text
Logger  → nima bo'lganini yozib borish
[[Sentry]] → xatolarni to'plash, guruhlash, ogohlantirish
```

Ikkalasi bir narsa emas: log — tarix, Sentry — signal.

## 5. Code example

```ts
// main.ts
const app = await NestFactory.create(AppModule, {
  logger:
    process.env.NODE_ENV === 'production'
      ? ['error', 'warn', 'log']
      : ['error', 'warn', 'log', 'debug', 'verbose'],
});
```

```ts
// service ichida — kontekst bilan
@Injectable()
export class NotificationsService {
  private readonly logger = new Logger(NotificationsService.name);

  async send(userId: string, channel: string) {
    try {
      await this.provider.send(userId, channel);
      this.logger.log(`Notification sent user=${userId} channel=${channel}`);
    } catch (err) {
      this.logger.error(
        `Notification failed user=${userId} channel=${channel}`,
        (err as Error).stack,
      );
      throw err;
    }
  }
}
```

E'tibor ber: logda **userId** bor, lekin telefon raqami, email yoki token **yo'q**.

## 6. Common mistakes

- **Token, parol, OTP yoki to'liq request body'ni loglash** → log fayli maxfiy ma'lumot omboriga aylanadi.
- **`console.log` ishlatish** → kontekst yo'q, daraja yo'q, formatlash yo'q.
- **Har bir 400 xatoni `error` bilan yozish** → alert shovqini.
- **Xatoni faqat `err.message` bilan yozish** → stack yo'q, sabab topilmaydi.
- **Loop ichida loglash** → disk va xarajat portlaydi.
- **Request id / correlation id qo'shmaslik** → bitta so'rovning loglarini yig'ib bo'lmaydi.
- **Xatoni loglab, keyin yutib yuborish** → filter va Sentry ko'rmaydi.

## 7. Practice

1. Service'ga `Logger` qo'shib, kontekst nomi logda ko'rinishini tasdiqla.
2. `NODE_ENV=production` bilan ishga tushirib, `debug` loglari yo'qolishini ko'r.
3. Request id generatsiya qiluvchi middleware yozib, uni loglarga qo'sh ([[NestJS Middleware]]).
4. Ataylab token loglaydigan kod yozib, keyin uni maskalash funksiyasi bilan tuzat.
5. `LoggingInterceptor` bilan har bir requestning davomiyligini yoz ([[NestJS Interceptors]]).

## 8. Related concepts

[[Sentry]] · [[NestJS Interceptors]] · [[NestJS Middleware]] · [[NestJS Exception Filters]] · [[NestJS Application Bootstrap]]
