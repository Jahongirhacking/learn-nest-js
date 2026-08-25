---
tags: [production, level/beginner, tooling]
level: beginner
topic: Tooling
---

# ESLint and Prettier

> **Bir jumlada:** bular runtime tool emas — developer experience toollari; ularni NestJS conceptlari bilan aralashtirmaslik kerak.

**Before learning:** [[TypeScript for NestJS]]
**After learning:** [[Testing in NestJS]], CI/CD

---

## 1. Definition

```text
ESLint   = code quality / mumkin bo'lgan xatolar
Prettier = formatting (probel, qator uzunligi, qavs)
```

Ular applicationning ishlashiga **hech qanday** ta'sir qilmaydi.

## 2. Why — nega kerak?

```text
Prettier → "tab yoki space?" bahsini butunlay yopadi
ESLint   → "bu await unutilgan", "bu o'zgaruvchi ishlatilmayapti" deb aytadi
```

Jamoada eng katta foydasi: PR review'da **format** haqida emas, **mantiq** haqida gaplashiladi.

## 3. Internal mental model

```text
Prettier → qanday ko'rinadi
ESLint   → nima yozilgan (potensial xatolar)
TypeScript → nima to'g'ri (tiplar)
Testlar  → nima ishlaydi
```

To'rttasi to'rt xil qatlam. Bittasi boshqasining o'rnini bosmaydi.

NestJS uchun ayniqsa foydali qoidalar:

```text
@typescript-eslint/no-floating-promises   → await unutilgan async chaqiruv
@typescript-eslint/no-explicit-any
@typescript-eslint/require-await
```

Birinchisi eng qimmatli: `this.queue.add(...)` ni `await`siz yozish jimgina yo'qolgan joblarga olib keladi ([[BullMQ]]).

## 4. FotSpot implementation

```text
eslint + @typescript-eslint  → statik tekshiruv
prettier                     → formatlash
package.json scripts         → lint, format
```

Ideal holda bular CI'da ham ishlaydi — shunda "menda ishlayapti" holati kamayadi.

## 5. Code example

```json
{
  "scripts": {
    "lint": "eslint \"{src,test}/**/*.ts\" --fix",
    "format": "prettier --write \"src/**/*.ts\" \"test/**/*.ts\""
  }
}
```

`no-floating-promises` nimani ushlaydi:

```ts
// ✗ xato: job qo'shilmasligi mumkin va hech kim bilmaydi
this.queue.add('trial-finished', { trialId });

// ✓
await this.queue.add('trial-finished', { trialId });
```

## 6. Common mistakes

- **ESLint va Prettier qoidalarini urishtirish** → `eslint-config-prettier` bilan yech.
- **`// eslint-disable` ni o'ylamasdan ishlatish** → ogohlantirish sababsiz o'chiriladi.
- **Lint'ni CI'ga qo'shmaslik** → hech kim ishlatmaydi.
- **Format o'zgarishlarini business commit bilan aralashtirish** → diff o'qib bo'lmaydi.
- **Linter'ni testning o'rnini bosadi deb o'ylash.**

## 7. Practice

1. `no-floating-promises` ni yoqib, `await`siz async chaqiruv yozib ko'r.
2. `npm run format` ishlatib, diff hajmiga qara — keyin alohida commit qil.
3. Bitta qoidani ataylab buzib, ESLint xabarini o'qi.
4. Lint va testni CI script'ga qo'sh.

## 8. Related concepts

[[TypeScript for NestJS]] · [[Testing in NestJS]] · [[BullMQ]] · [[NestJS Overview]]
