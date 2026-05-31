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
import { Chart } from './chart.mjs'; // sahifa ochilganda, butun bundle'ga kiradi

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
4. **Architecture'ni qayta ko'rish** — ko'pincha circular = noto'g'ri architecture

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

### 2. Bu kodda nima xato? [Middle]

**Savol:**

```javascript
// utils.mjs
console.log("utils loaded");
export function format(x) { return String(x); }

// app.mjs
console.log("app start");
import './utils.mjs';
console.log("app end");
```

Output nima va nima uchun?

<details>
<summary>Javob</summary>

```
utils loaded
app start
app end
```

ESM `import` declaration'lari **hoisted** — kod yozilishidan qat'i nazar, engine ularni modul tepasiga ko'taradi. Parse → Link → Evaluate bosqichlarida import qilingan modullar **dependency tartibida** evaluate qilinadi. Shuning uchun `utils.mjs` top-level kodi `app.mjs` ning birinchi `console.log` dan **oldin** ishlaydi.

**Amaliy ta'sir:**
- Side effect'larga tayanish xavfli — "birinchi qator" `console.log` aslida birinchi emas
- Conditional `if (cond) import './x'` SyntaxError beradi (static import top-level)
- Shart bilan side effect kerak bo'lsa — `await import('./utils.mjs')` (dynamic) ishlatish

**Deep Dive:** Spec'da ESM 3 bosqichli loading bor — Parse (import/export declaration'larni topish), Link (binding'lar ulanadi, qiymat hali yo'q), Evaluate (top-level code dependency tartibida ishlaydi). `import` declaration'ni funksiya yoki `if` ichiga joylash SyntaxError beradi — chunki linker compile-time'da dependency graph qurishi kerak.

</details>

### 3. Dynamic import xato bilan qanday ishlash kerak? [Middle+]

<details>
<summary>Javob</summary>

`import()` Promise qaytaradi — modul topilmasa yoki yuklashda xato bo'lsa rejected bo'ladi. `try/catch` (async/await) yoki `.catch()` bilan ushlash kerak. Fallback va retry pattern'lari production'da muhim.

```javascript
// ✅ try/catch bilan
async function loadAnalytics() {
  try {
    const module = await import('./analytics.mjs');
    return module.default;
  } catch (err) {
    console.error('Analytics yuklanmadi:', err);
    return { track: () => {} }; // no-op fallback
  }
}

// ✅ Retry pattern — network xato uchun
async function loadWithRetry(path, retries = 3) {
  for (let i = 0; i < retries; i++) {
    try {
      return await import(path);
    } catch (err) {
      if (i === retries - 1) throw err;
      await new Promise(r => setTimeout(r, 1000 * (i + 1)));
    }
  }
}

// ✅ Browser'da CSP/MIME xatolari
try {
  const m = await import('https://cdn.example.com/lib.mjs');
} catch (err) {
  if (err instanceof TypeError) {
    // MIME type xato, CORS, yoki specifier resolution
  }
}
```

**Xato turlari:**
- `TypeError` — browser'da loading xatosi (network fail, CORS, MIME type, 404/500, specifier resolution) hammasi `TypeError` bo'lib reject bo'ladi
- `SyntaxError` — modul kodi parse bo'lmadi
- `MODULE_NOT_FOUND` / `ERR_MODULE_NOT_FOUND` (Node.js) — fayl topilmadi (browser'dan farqli, Node generic `Error` ishlatadi)

**Deep Dive:** Dynamic import internal'da `HostImportModuleDynamically` spec abstract operation'ni chaqiradi — bu host (browser/Node.js) tomonidan implement qilinadi. Modul uch bosqichda yuklanadi: loading (fetch) → linking (parse) → evaluation. Spec bo'yicha **faqat evaluation bosqichidagi xato cache'lanadi** — keyingi import shu saqlangan xatoni qaytaradi. Loading yoki linking bosqichidagi xato (network, MIME, CORS, 404) cache'lanmaydi — keyingi import qayta urinib ko'rishi mumkin. Amalda Chromium-based browser'lar `fetch` natijasini HTTP darajasida cache'laydi, shuning uchun deploy'dan keyin eski chunk nomiga so'rov sticky failure berishi mumkin — bu module-system emas, browser cache xatti-harakati. Cache bust uchun query string ishlatiladi: `import('./mod.mjs?v=' + Date.now())`.

</details>

### 4. Top-level await ESM'da qanday ishlaydi? Performance ta'siri qanday? [Senior]

<details>
<summary>Javob</summary>

Top-level await (TLA) — ES2022'da kiritilgan. ESM top-level kodida `await` ishlatish mumkin — funksiya ichida bo'lishi shart emas. Modul evaluation paytida `await` resolve bo'lguncha **dependent modul'lar kutadi**.

```javascript
// config.mjs
const response = await fetch('/api/config');
export const config = await response.json();

// app.mjs
import { config } from './config.mjs';
// app.mjs config.mjs evaluate tugaguncha kutadi
console.log(config.apiUrl); // config tayyor
```

**Spec mexanizmi:** TLA bo'lgan modul `[[AsyncEvaluation]]` flag bilan belgilanadi. Module graph evaluate bo'lganda — async modul'lar Promise chain qoldiradi. Importer modul async modul evaluate tugaguncha kutadi (parent async-blocked).

**Performance afzalliklari:**
- **Sequential dependency'lar** uchun cleaner — `await import()` chain yo'q
- **Module-level resource init** — DB connection, config fetch oldindan tayyor
- **Conditional default exports** — environment bo'yicha turli export

**Performance kamchiliklari:**
- **Module graph delay** — TLA modul barcha dependent'larini bloklaydi
- **Waterfall loading** — agar TLA modullari ketma-ket bog'lansa, parallel ishlamaydi
- **Bundler complexity** — Webpack/Rollup TLA'ni alohida processing qiladi (async wrapper)

```javascript
// ❌ Anti-pattern — TLA waterfall:
// a.mjs:
export const a = await fetch('/a').then(r => r.json());
// b.mjs:
import { a } from './a.mjs';
export const b = await fetch('/b').then(r => r.json());
// b a tugagunicha kutadi — sequential!

// ✅ Promise.all bilan parallel:
// data.mjs:
const [a, b] = await Promise.all([
  fetch('/a').then(r => r.json()),
  fetch('/b').then(r => r.json()),
]);
export { a, b };
```

CJS'da nima uchun ishlamaydi: `require()` sinxron — chaqiruvchi thread'ni bloklaydi. `await` sinxron kontekstda semantik jihatdan ma'nosiz.

**Deep Dive:** Spec'da TLA `AsyncBlock` evaluation mexanizmi orqali implement qilingan. Modul `[[CycleRoot]]` va `[[AsyncEvaluation]]` slot'lari bilan kelishiladi. Bundler'lar (Webpack 5+, Rollup, Vite) TLA modullarni `__webpack_async__` yoki o'xshash wrapper bilan o'raydi — chunki bundled output sinxron IIFE bo'lishi mumkin emas. Bu wrapper bundle hajmiga oz miqdorda runtime overhead qo'shadi.

</details>

### 5. ESM dan CommonJS modulni import qilganda qanday muammolar yuzaga keladi? [Middle+]

<details>
<summary>Javob</summary>

CJS modulni ESM'dan import qilish mumkin, lekin **named import** ko'p hollarda ishlamaydi. Sababi: CJS `module.exports` runtime'da aniqlanadi, ESM esa static analysis kutadi.

```javascript
// lodash CJS modul
// ✅ Default import — module.exports butun qiymati
import lodash from 'lodash';
lodash.map([1,2,3], x => x*2);

// ❌ Named import — ko'p CJS modullarda ishlamaydi
import { map } from 'lodash';
// SyntaxError: The requested module does not provide an export named 'map'

// ✅ Workaround 1: destructuring default'dan
import lodash from 'lodash';
const { map, filter } = lodash;

// ✅ Workaround 2: createRequire
import { createRequire } from 'module';
const require = createRequire(import.meta.url);
const { map } = require('lodash');

// ✅ Workaround 3: Dynamic import
const lodash = await import('lodash');
const { map } = lodash.default; // CJS module.exports → default
```

**Nima uchun named import ko'pincha ishlamaydi:**

CJS modul `module.exports = { map, filter }` qiladi — bu runtime'da tahlil qilinadi. ESM esa **compile-time** da export'larni bilishi kerak. Node.js CJS modulni "best-effort" parse qiladi — agar oddiy literal export bo'lsa, named import ishlashi mumkin (Node.js 14+ static analysis). Lekin dynamic export'lar (`exports[name] = ...`) topilmaydi.

**Modern Node.js (22.12+):** `require(esm)` ham qo'llab-quvvatlanadi — CJS modul'dan ESM modulni `require()` orqali olish (synchronous, faqat sync ESM uchun — TLA bo'lsa ishlamaydi).

**Deep Dive:** Node.js ESM loader CJS modulni o'qiyotganda `cjs-module-lexer` package'idan foydalanadi — bu static analysis bilan `module.exports.X = ...` va `exports.X = ...` pattern'larini topadi. Murakkab pattern'lar (`Object.assign(module.exports, ...)`, ternary, function-based) topilmaydi. Bu sababli kutubxonalar `package.json.exports` bilan dual package (CJS + ESM build) yaratadilar.

</details>
