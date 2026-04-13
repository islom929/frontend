# Modules — Interview Savollari

> CommonJS vs ES Modules, import/export, dynamic imports, live bindings, circular dependencies, tree shaking, bundlers haqida interview savollari.

---

## Nazariy savollar

### 1. CommonJS va ES Modules farqi nima? [Junior+]

<details>
<summary>Javob</summary>

| Xususiyat | CommonJS | ES Modules |
|-----------|----------|------------|
| Syntax | `require()` / `module.exports` | `import` / `export` |
| Loading | **Sinxron** | **Asinxron** |
| Tahlil | **Runtime** (dynamic) | **Compile-time** (static) |
| Binding | **Value copy** | **Live binding** |
| `this` | `exports` (dastlab `module.exports` ga teng, lekin qayta tayinlansa farqlanadi) | `undefined` |
| Strict mode | Yo'q | **Avtomatik** |
| Tree shaking | ❌ | ✅ |
| Browser | Bundler kerak | Native support |

```javascript
// CommonJS
const { readFile } = require('fs');
module.exports = { myFunction };

// ES Modules
import { readFile } from 'fs';
export function myFunction() {}
```

Eng muhim farq: ESM **static** — import/export'lar compile-time da tahlil qilinadi, bu tree shaking va optimization imkonini beradi. CJS **dynamic** — runtime da bajariladi.

</details>

### 2. `module.exports` va `exports` farqi nima? [Middle]

<details>
<summary>Javob</summary>

`exports` — `module.exports` ga **shortcut** (reference). Boshlang'ichda ikkalasi **bir xil ob'ekt**ga ishora qiladi. Lekin Node.js faqat `module.exports` ni export qiladi.

```javascript
// ✅ Property qo'shish — ishlaydi
exports.add = (a, b) => a + b;
// module.exports === exports → { add: fn }

// ❌ Qayta assign — ISHLAMAYDI
exports = { add: (a, b) => a + b };
// exports yangi ob'ektga ishora → module.exports hali eski {} ga

// ✅ module.exports ni assign — TO'G'RI
module.exports = { add: (a, b) => a + b };
```

Qoida: `module.exports` ishlatish xavfsizroq. `exports` ga faqat property qo'shish mumkin, qayta assign qilish MUMKIN EMAS.

</details>

### 3. Live bindings nima? [Middle+]

<details>
<summary>Javob</summary>

ESM da export qilingan qiymat **reference** sifatida ulangan. Source modul qiymatni o'zgartirsa — import qilgan modul ham yangi qiymatni ko'radi. CommonJS da esa **value copy** bo'ladi.

```javascript
// === ESM — LIVE BINDING ===
// counter.mjs
export let count = 0;
export function increment() { count++; }

// main.mjs
import { count, increment } from './counter.mjs';
console.log(count); // 0
increment();
increment();
console.log(count); // 2 ← O'ZGARDI! Live binding

// === CJS — VALUE COPY ===
// counter.cjs
let count = 0;
module.exports = { count, increment() { count++; } };

// main.cjs
const { count, increment } = require('./counter.cjs');
increment(); increment();
console.log(count); // 0 ← O'ZGARMADI! Copy
```

Bu ESM ning eng muhim xususiyatlaridan biri — modul ichidagi state'ni doim haqiqiy ko'rish imkonini beradi.

</details>

### 4. Named export va Default export farqi nima? [Junior+]

<details>
<summary>Javob</summary>

```javascript
// Named export — bir faylda bir nechta
export function add(a, b) { return a + b; }
export function sub(a, b) { return a - b; }
export const PI = 3.14;

// Import — aniq nom bilan, {} ichida
import { add, sub, PI } from './math.mjs';

// Default export — bir faylda BITTA
export default class Database { /* ... */ }

// Import — istalgan nom bilan, {} SIZ
import DB from './database.mjs';
import MyDB from './database.mjs'; // boshqa nom ham bo'ladi

// Ikkalasi birga
import Database, { createPool } from './database.mjs';
```

| | Named | Default |
|--|-------|---------|
| Bir faylda nechta | Ko'p | **Faqat 1** |
| Import syntax | `{ name }` | `name` ({}siz) |
| Nom o'zgartirish | `as` kerak | Erkin |
| Tree shaking | **Yaxshi** | O'rtacha |

</details>

### 5. `import()` dynamic import nima? Qachon ishlatiladi? [Middle]

<details>
<summary>Javob</summary>

`import()` — modulni **runtime** da, **asinxron** yuklaydigan funksiya-like syntax. Promise qaytaradi.

```javascript
// Static import — DOIM yuklanadi
import { Chart } from './chart.mjs'; // 200KB — sahifa ochilganda

// Dynamic import — KERAK BO'LGANDA yuklanadi
async function showChart() {
  const { Chart } = await import('./chart.mjs'); // faqat tugma bosilganda
  new Chart(data).render();
}
```

Asosiy use case'lar:
1. **Code splitting** — katta modulni kerak bo'lganda yuklash
2. **Route-based lazy loading** — sahifa bo'yicha (React.lazy, Vue async components)
3. **Conditional loading** — environment/feature flag bo'yicha
4. **CJS → ESM interop** — CommonJS da ESM import qilish

</details>

### 6. Circular dependency nima? Qanday oldini olish mumkin? [Middle+]

<details>
<summary>Javob</summary>

Circular dependency — A moduli B ni import qiladi, B esa A ni import qiladi. Natijada initialization tartibida muammo bo'ladi:

```javascript
// a.mjs
import { b } from './b.mjs';
export var a = "A + " + b;

// b.mjs
import { a } from './a.mjs';
export var b = "B + " + a;

// Natija: a = "A + B + undefined" — a hali assign bo'lmagan!
// DIQQAT: const/let ishlatilsa ReferenceError (TDZ) bo'ladi, var ishlatilsa undefined
```

Yechimlar:
1. **Umumiy dependency'ni alohida modulga chiqarish** — A va B ning umumiy qismini C ga
2. **Dynamic import** — `await import('./b.mjs')` lazy
3. **Dependency injection** — parametr orqali berish
4. **Arxitekturani qayta ko'rish** — ko'pincha circular = noto'g'ri arxitektura

</details>

### 7. Tree shaking nima? Nima uchun faqat ESM bilan ishlaydi? [Middle]

<details>
<summary>Javob</summary>

Tree shaking — bundle'dan **ishlatilmagan kodni** olib tashlash. Bundle hajmini kamaytiradi.

```javascript
// ✅ ESM — tree shaking ishlaydi
import { map } from 'lodash-es'; // faqat map, qolgani olib tashlanadi

// ❌ CJS — tree shaking ishlamaydi
const { map } = require('lodash'); // BUTUN lodash yuklanadi
```

Nima uchun faqat ESM? ESM import/export'lari **static** — compile-time da qaysi export ishlatilganini aniqlash mumkin. CJS `require()` **dynamic** — `require(condition ? 'a' : 'b')` kabi runtime expression bo'lishi mumkin — bundler oldindan bilmaydi.

`sideEffects: false` — package.json da bundler'ga side-effect-free ekanligini ko'rsatish:

```json
{ "sideEffects": false }
```

</details>

### 8. ESM da `require()` ishlatsa bo'ladimi? [Middle]

<details>
<summary>Javob</summary>

**Yo'q.** ESM da `require`, `module`, `exports`, `__filename`, `__dirname` — yo'q. Bular CommonJS wrapper'dan keladi.

```javascript
// ❌ ESM da ishlamaydi
const fs = require('fs'); // ReferenceError

// ✅ ESM ga mos usullar
import fs from 'fs';

// ✅ __dirname ekvivalenti
import { fileURLToPath } from 'url';
import { dirname } from 'path';
const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);

// ✅ Agar CJS modul kerak bo'lsa
import { createRequire } from 'module';
const require = createRequire(import.meta.url);
const cjsModule = require('./legacy.cjs');
```

</details>

### 9. `import * as` va `import { }` farqi nima? [Middle]

<details>
<summary>Javob</summary>

```javascript
// Named import — faqat keraklilarni
import { add, sub } from './math.mjs';
add(1, 2); // to'g'ridan-to'g'ri

// Namespace import — hammasini bitta ob'ektga
import * as Math from './math.mjs';
Math.add(1, 2); // ob'ekt orqali

// Default import — default export
import Calculator from './math.mjs';
```

| | `import { x }` | `import * as M` |
|--|----------------|-----------------|
| Tree shaking | **Yaxshi** | Bundler'ga bog'liq |
| Clarity | Qaysi export ishlatilgani aniq | Noaniq |
| Naming collision | `as` bilan hal qilinadi | M.x — collision yo'q |
| Use case | Ko'p hollarda | Ko'p export bo'lganda |

</details>

### 10. Node.js da CJS va ESM ni qanday farqlaydi? [Middle+]

<details>
<summary>Javob</summary>

Node.js modulni **CJS yoki ESM** deb aniqlaydigan qoidalar:

| Shart | Turi |
|-------|------|
| `.cjs` extension | CommonJS |
| `.mjs` extension | ES Module |
| `.js` + `"type": "module"` package.json da | ES Module |
| `.js` + `"type": "commonjs"` yoki yo'q | CommonJS (default) |

```json
// package.json
{
  "type": "module"  // barcha .js fayllar ESM
}
```

Kutubxona uchun ikkala formatni qo'llab-quvvatlash — **dual package**:

```json
{
  "exports": {
    ".": {
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs"
    }
  }
}
```

</details>

### 11. `import.meta` nima? [Middle+]

<details>
<summary>Javob</summary>

`import.meta` — joriy modul haqida meta-ma'lumot saqlovchi ob'ekt. Faqat ES Module'larda mavjud.

```javascript
// import.meta.url — modul faylining to'liq URL
console.log(import.meta.url);
// Node.js: "file:///home/user/project/app.mjs"
// Browser:  "https://example.com/app.mjs"

// __dirname ekvivalenti
import { fileURLToPath } from 'url';
import { dirname } from 'path';
const __dirname = dirname(fileURLToPath(import.meta.url));

// import.meta.resolve() — modul yo'lini aniqlash (Node.js 20+)
const lodashPath = import.meta.resolve('lodash');
```

</details>

### 12. Re-export (barrel file) nima? Performance muammosi bormi? [Middle+]

<details>
<summary>Javob</summary>

Re-export — bir nechta modulni bitta fayldan qayta export qilish:

```javascript
// components/index.mjs — barrel file
export { Button } from './Button.mjs';
export { Input } from './Input.mjs';
export { Modal } from './Modal.mjs';

// Qulay import:
import { Button, Input } from './components/index.mjs';
```

**Performance muammosi:** Bundler barrel file'dagi BARCHA modullarni parse qiladi — hatto faqat 1 ta import qilinsa ham. Katta loyihalarda build vaqtini oshiradi.

Yechim: to'g'ridan-to'g'ri import:

```javascript
import { Button } from './components/Button.mjs'; // faqat bitta fayl
```

</details>

### 13. ESM da top-level await ishlaydi, CJS da nima uchun ishlamaydi? [Middle+]

<details>
<summary>Javob</summary>

ESM — **asinxron** yuklaydi: construction → instantiation → evaluation. Top-level `await` evaluation bosqichida ishlaydi — bu modulni import qilgan boshqa modullar kutadi.

CJS — **sinxron** yuklaydi: `require()` blokirovka qiladi — chaqiruvchi thread'ni to'xtatadi. Sinxron kontekstda `await` semantik jihatdan ma'nosiz.

```javascript
// ✅ ESM — ishlaydi
const config = await fetch('/api/config').then(r => r.json());
export default config;

// ❌ CJS — ishlamaydi
const config = await fetch('/api/config'); // SyntaxError
module.exports = config;

// CJS da workaround:
async function init() {
  const config = await fetch('/api/config').then(r => r.json());
  module.exports = config;
}
init();
// Lekin require() sinxron — natija tayyor bo'lmaydi!
```

</details>

### 14. Module bundler nima? Webpack va Vite farqi? [Middle]

<details>
<summary>Javob</summary>

Bundler — bir nechta modul fayllarni bitta yoki bir nechta bundle'ga birlashtiradi.

| | Webpack | Vite |
|--|---------|------|
| Dev server | Bundle qiladi, keyin serve | **Bundlemasdan** ESM bilan serve |
| HMR tezligi | Sekin (rebundle kerak) | **Juda tez** (faqat o'zgargan modul) |
| Build | O'zi | **Rollup** ishlatadi |
| Config | Murakkab | Minimal |
| Use case | Legacy, katta production | Zamonaviy SPA |

Vite dev da tezroq chunki bundlemaydi — browser'ning native ESM support'idan foydalanadi. Webpack esa dev da ham butun dependency graph'ni bundle qiladi.

</details>

---

## Amaliy savollar (Coding Challenges)

### 1. Bu kodning output'ini ayting [Middle+]

**Savol:**

```javascript
// counter.cjs
let count = 0;
module.exports = {
  count,
  increment() { count++; },
  getCount() { return count; }
};

// main.cjs
const mod = require('./counter.cjs');
mod.increment();
mod.increment();
console.log(mod.count);     // ?
console.log(mod.getCount()); // ?
```

<details>
<summary>Javob</summary>

```
0
2
```

`mod.count` — export paytidagi qiymat (0) **copy** bo'lgan, hech qachon o'zgarmaydi. `mod.getCount()` — closure orqali ichki `count` ga murojaat qiladi, shuning uchun haqiqiy qiymat (2) qaytaradi. Bu CommonJS dagi value copy muammosi.

</details>
