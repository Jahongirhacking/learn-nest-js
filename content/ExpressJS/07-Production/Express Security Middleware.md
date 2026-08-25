---
tags: [express/production, level/intermediate, security]
level: intermediate
topic: Production
---

# Express Security Middleware

> **Bir jumlada:** `helmet` va `cors` — Express kutubxonalari; NestJS'da ular **aynan xuddi shunday** yoziladi, chunki NestJS Express ustida ishlaydi.

**Before learning:** [[Express Middleware Order]], [[Express App Bootstrap]]
**After learning:** [[Helmet]], [[CORS]]

**NestJS'dagi mos note:** [[Helmet]] · [[CORS]]

---

## 1. Eng qiziq fakt

```js
// Express
app.use(helmet());

// NestJS main.ts
app.use(helmet());          // ← AYNAN BIR XIL QATOR
```

Chunki NestJS'ning `app.use()` — bu Express'ning `app.use()` ning o'zi.

Shuning uchun [[Helmet]] va [[CORS]] notelaridagi hamma narsa Express'ga **100% tegishli**.
Bu note faqat farqli joylarni va tartibni ko'rsatadi.

## 2. To'liq security qatlami

```js
const helmet = require('helmet');
const cors = require('cors');
const rateLimit = require('express-rate-limit');
const config = require('./config');

const app = express();

// 1. Proxy ortida bo'lsa — SHART
app.set('trust proxy', 1);      // aks holda req.ip har doim proxy IP'si bo'ladi

// 2. Security headerlar
app.use(helmet({
  crossOriginResourcePolicy: { policy: 'cross-origin' },   // media boshqa domendan
}));

// 3. CORS
app.use(cors({
  origin: config.cors.origins,          // ✗ '*' EMAS
  credentials: true,
  methods: ['GET','POST','PATCH','PUT','DELETE','OPTIONS'],
  allowedHeaders: ['Content-Type','Authorization'],
  maxAge: 86_400,
}));

// 4. Rate limit — qimmat ishdan OLDIN
app.use(rateLimit({ windowMs: 60_000, max: 100 }));

// 5. Body hajmi chegarasi
app.use(express.json({ limit: '1mb' }));
```

Har bir raqamning sababi [[Express Middleware Order]] da.

## 3. `trust proxy` — Express'ga xos tuzoq

```js
app.set('trust proxy', 1);
```

Nima uchun:

```text
Client → Nginx/Cloudflare → Express

trust proxy YO'Q:
   req.ip = proxy'ning IP'si (har doim bir xil!)
   → rate limit hamma uchun umumiy hisoblanadi
   → bitta user butun saytni bloklaydi

trust proxy BOR:
   req.ip = X-Forwarded-For dagi haqiqiy IP
```

**Ehtiyot bo'l:** `trust proxy` ni proxy bo'lmagan holda yoqish xavfli — har kim `X-Forwarded-For` header'ini o'zi yuborib, rate limit'ni chetlab o'tadi.

NestJS'da ham bu sozlama kerak:

```ts
app.set('trust proxy', 1);          // NestExpressApplication
```

## 4. CORS — asosiy qoidalar

```js
// ✗ XATO
app.use(cors());                                    // origin: '*'
app.use(cors({ origin: '*', credentials: true }));  // brauzer RAD ETADI

// ✓
app.use(cors({ origin: ['https://app.fotspot.uz'], credentials: true }));
```

Muhim tushuncha ([[CORS]]):

```text
CORS brauzerda ishlaydi.
   → curl, Postman, mobil ilova CORS'ni umuman ko'rmaydi
   → demak CORS himoya emas, brauzer siyosati
```

## 5. NestJS bilan farq

| Express | NestJS |
|---|---|
| `app.use(helmet())` | `app.use(helmet())` — **bir xil** |
| `app.use(cors({...}))` | `app.enableCors({...})` — o'ram |
| `app.set('trust proxy', 1)` | `app.set('trust proxy', 1)` — bir xil |
| `express.json({ limit })` | avtomatik (`bodyParser` opsiyalari bilan) |
| `express-rate-limit` | `@nestjs/throttler` (yoki xuddi shu paket) |

Ya'ni bu qatlamda **deyarli hech qanday farq yo'q**. Farq faqat throttling'da: NestJS uni guard qilib, per-endpoint metadata bilan boshqarish imkonini beradi ([[Rate Limiting and Throttling]]).

## 6. Common mistakes

- **`cors()` ni parametrsiz production'da qoldirish** → hamma origin ochiq.
- **`origin: '*'` + `credentials: true`** → umuman ishlamaydi, xatosi chalkash.
- **`trust proxy` ni unutish** → rate limit va IP logging buziladi.
- **`trust proxy` ni proxy'siz yoqish** → IP soxtalashtirish mumkin.
- **`express.json()` ga `limit` bermaslik** → xotira hujumi.
- **Preflight `OPTIONS` ni auth middleware bilan bloklash** → brauzer asosiy so'rovni yubormaydi.
- **Helmet'ni CSP muammosi tufayli butunlay o'chirish** → faqat kerakli direktivani sozla.

## 7. Practice problems

### Problem 1 — headerlarni solishtir
`helmet()` siz `curl -I` qil va headerlarni yozib ol. Keyin `helmet()` qo'shib, farqni solishtir.

### Problem 2 — CORS xatosi
Boshqa origin'dagi HTML sahifadan `fetch()` bilan API'ga so'rov yubor. Brauzer konsolidagi xatoni to'liq o'qi.

### Problem 3 — `curl` bilan
Xuddi shu so'rovni `curl` bilan yubor. CORS to'sdimi?
**Savol:** nega yo'q? CORS himoyami?

### Problem 4 — `origin: '*'` + credentials
Ikkalasini birga yoqib, brauzerdan sinab ko'r. Xato matnini yozib ol.

### Problem 5 — preflight
`curl -X OPTIONS -i -H 'Origin: http://localhost:5173' -H 'Access-Control-Request-Method: POST' ...`
Javob headerlarini o'qi.

### Problem 6 — `trust proxy`
`req.ip` ni logla. `X-Forwarded-For: 1.2.3.4` header bilan so'rov yubor.
`trust proxy` bilan va busiz farqni yoz.

### Problem 7 — NestJS'da takrorla
NestJS `main.ts` da xuddi shu 5 qatorni yoz. Qaysi biri o'zgardi?

## 8. Related

[[Express Middleware Order]] · [[Express Rate Limiting]] · [[Express App Bootstrap]] · [[Express Config and Env]] · [[Helmet]] · [[CORS]]
