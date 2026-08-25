---
tags: [express/foundation, level/beginner]
level: beginner
topic: Express Foundations
---

# Node HTTP Server

> **Bir jumlada:** Express ham, NestJS ham oxir-oqibat Node'ning `http` moduli ustida ishlaydi — shuni bir marta ko'rsang, qolgan hamma qatlam tushunarli bo'ladi.

**Before learning:** JavaScript asoslari, `async/await`
**After learning:** [[Express Overview]], [[Express Request and Response]]

**NestJS'dagi mos note:** [[NestJS Application Bootstrap]]

---

## 1. Muammo — nima uchun bu kerak?

NestJS'da sen shunday yozasan:

```ts
@Get(':id')
findOne(@Param('id') id: string) {
  return this.service.findOne(id);
}
```

Va savol tug'iladi: **`return` qilgan obyekt qanday qilib HTTP javobga aylanadi?**

Javob eng pastki qatlamda. Keling, o'sha qatlamni ko'ramiz.

## 2. Express'siz server — sof Node

```js
// server.js
const http = require('http');

const server = http.createServer((req, res) => {
  // req  → kirayotgan so'rov
  // res  → chiqayotgan javob
  console.log(req.method, req.url);

  if (req.method === 'GET' && req.url === '/health') {
    res.statusCode = 200;
    res.setHeader('Content-Type', 'application/json');
    res.end(JSON.stringify({ status: 'ok' }));
    return;
  }

  res.statusCode = 404;
  res.setHeader('Content-Type', 'application/json');
  res.end(JSON.stringify({ message: 'Not Found' }));
});

server.listen(3000, () => console.log('listening on 3000'));
```

Ishga tushir:

```bash
node server.js
curl -i http://localhost:3000/health
```

## 3. Qator-qator tahlil

```text
http.createServer(fn)
   → har bir kelgan so'rov uchun fn chaqiriladi
   → fn(req, res)

req
   → o'qish uchun stream (IncomingMessage)
   → req.method, req.url, req.headers
   → BODY AVTOMATIK PARSE QILINMAYDI (buni o'zing qilasan!)

res
   → yozish uchun stream (ServerResponse)
   → res.statusCode, res.setHeader(), res.end()
   → res.end() chaqirilmasa — brauzer kutib qoladi

server.listen(port)
   → TCP portga bind qiladi
```

Diqqat: bu yerda **route yo'q**. `if (req.url === ...)` — bu senga qolgan ish.
100 ta endpoint bo'lsa, 100 ta `if`. Mana shuning uchun routing kerak.

## 4. Eng og'riqli joy — body'ni o'qish

Sof Node'da POST body'ni olish shunday ko'rinadi:

```js
const server = http.createServer((req, res) => {
  if (req.method === 'POST' && req.url === '/players') {
    let raw = '';

    req.on('data', chunk => { raw += chunk; });       // bo'laklab keladi

    req.on('end', () => {
      let body;
      try {
        body = JSON.parse(raw);
      } catch {
        res.statusCode = 400;
        return res.end(JSON.stringify({ message: 'Invalid JSON' }));
      }

      res.statusCode = 201;
      res.setHeader('Content-Type', 'application/json');
      res.end(JSON.stringify({ received: body }));
    });
    return;
  }
  res.statusCode = 404;
  res.end();
});
```

**Mana shuning uchun Express kerak.** Express'da bu bitta qator:

```js
app.use(express.json());   // endi req.body tayyor
```

Va NestJS'da hatto bu qator ham yo'q — u avtomatik.

## 5. NestJS buni qanday yashiradi

```text
Sof Node                          Express                    NestJS
──────────────────────────────────────────────────────────────────────────
http.createServer(fn)             express()                  NestFactory.create()
if (req.url === '/health')        app.get('/health', fn)     @Get('health')
req.on('data') + JSON.parse       express.json()             avtomatik + DTO
res.statusCode = 200              res.status(200)            @HttpCode(200)
res.end(JSON.stringify(x))        res.json(x)                return x
server.listen(3000)               app.listen(3000)           app.listen(3000)
```

Oxirgi qator eng muhimi:

```text
NestJS'da `return x` yozganingda
   → framework res.json(x) ni SEN uchun chaqiradi
```

Hech qanday sehr yo'q — shunchaki kimdir o'sha kodni sen uchun yozib qo'ygan.

## 6. Common mistakes

- **`res.end()` yoki `res.json()` ni chaqirmaslik** → so'rov osilib qoladi, xato ham chiqmaydi.
- **Javob yozgandan keyin yana yozish** → `ERR_HTTP_HEADERS_SENT`. `return` qilishni unutma.
- **Body avtomatik parse qilinadi deb o'ylash** → sof Node'da yo'q.
- **`res.setHeader` ni `res.end()` dan keyin chaqirish** → xato.
- **`req.on('data')` bilan hajmni cheklamaslik** → xotira hujumi.

## 7. Practice problems

### Problem 1 — sof Node server
`GET /health` uchun `{"status":"ok"}` qaytaruvchi server yoz (Express'siz).
**Kutilgan natija:** `curl -i` da `200` va `Content-Type: application/json`.

### Problem 2 — 404
Boshqa har qanday URL uchun `404` va `{"message":"Not Found"}` qaytar.

### Problem 3 — POST body
`POST /echo` yozib, kelgan JSON body'ni qaytar. `req.on('data')` bilan qo'lda.
**Tekshir:** `curl -X POST -d '{"a":1}' -H 'Content-Type: application/json' http://localhost:3000/echo`

### Problem 4 — buzilgan JSON
Body noto'g'ri JSON bo'lsa `400` qaytar. Ataylab `-d '{broken'` yubor.

### Problem 5 — `res.end()` ni o'chir
Handler ichidan `res.end()` ni olib tashla va `curl` ni ishga tushir.
**Savol:** nima bo'ladi? Qancha kutadi? Nega xato chiqmaydi?
**Bog'lanish:** aynan shu — Express'da `next()` chaqirmaslikning oqibati ([[Express next Function]]).

### Problem 6 — 5 ta endpoint
Sof Node bilan 5 ta endpoint yoz (`GET /players`, `GET /players/:id`, `POST /players`, ...).
`:id` ni qo'lda ajratib olishga urin (`req.url.split('/')`).
**Savol:** necha qator bo'ldi? Endi buni 50 ta endpointga ko'paytir.

### Problem 7 — solishtirish
Xuddi shu endpointlarni Express bilan qayta yoz ([[Express App Bootstrap]]).
Necha qator qisqardi? Qaysi qismlar butunlay yo'qoldi?

## 8. Related

[[Express Overview]] · [[Express App Bootstrap]] · [[Express Request and Response]] · [[Express Routing]] · [[NestJS Application Bootstrap]] · [[NestJS to Express Translation]]
