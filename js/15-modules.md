# Bo'lim 15: Modules

> Module — kodni mustaqil, qayta ishlatiladigan bo'laklarga ajratish tizimi. Global scope ifloslanishining oldini oladi va zamonaviy JavaScript arxitekturasining asosini tashkil etadi.

---

## Mundarija

- [Nima Uchun Modullar Kerak](#nima-uchun-modullar-kerak)
- [Module Pattern (IIFE)](#module-pattern-iife)
- [CommonJS — Node.js Standart](#commonjs--nodejs-standart)
- [ES Modules — Zamonaviy Standart](#es-modules--zamonaviy-standart)
- [CommonJS vs ES Modules — To'liq Taqqoslash](#commonjs-vs-es-modules--toliq-taqqoslash)
- [Dynamic Imports — import()](#dynamic-imports--import)
- [Import Attributes (ES2025)](#import-attributes-es2025)
- [Circular Dependencies](#circular-dependencies)
- [Module Bundlers](#module-bundlers)
- [Tree Shaking](#tree-shaking)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Nima Uchun Modullar Kerak

### Nazariya

Modullar kiritilmasdan oldin JavaScript da barcha kodlar **bitta global scope** da yashardi. `<script>` teglar bilan ulangan barcha fayllar o'zgaruvchilarni bir-birining ustiga yozishi mumkin edi — bu **global scope pollution** deyiladi. Loyiha kattalashgan sari muammolar ko'payadi: nom to'qnashuvi (naming collision), kutilmagan o'zgarishlar (side effects), va dependency tartibini qo'lda boshqarish.

Modullar quyidagi muammolarni hal qiladi:
1. **Encapsulation** — har bir modul o'z private scope'iga ega, faqat export qilinganlar tashqaridan ko'rinadi
2. **Namespace** — modul nomi bilan ajratish (naming collision yo'q)
3. **Dependency management** — modul o'z dependency'larini aniq ko'rsatadi (`import`/`require`)
4. **Code reuse** — bir marta yozilgan modul istalgan joyda qayta ishlatiladi
5. **Lazy loading** — kerakli modulni kerak bo'lganda yuklash (performance)

<details>
<summary><strong>Under the Hood</strong></summary>

**Module system evolutsiyasi**:
- 2009: CommonJS (Node.js) — synchronous, file-based
- 2009: AMD (RequireJS) — browser uchun async
- 2011: UMD — universal format
- 2015: ES Modules (ES6 spec) — native, static, tree-shakable
- 2017: Browser ESM (`<script type="module">`)
- 2019: Node.js ESM (`.mjs`, `"type": "module"`)

**Module system engine perspektivasidan nima beradi**:
1. **Static analysis** — import/export compile-time'da aniq, bundler tree-shaking qila oladi
2. **Scope isolation** — global scope ifloslanishi yo'q, V8 har file uchun alohida hidden class optimizatsiyasi qila oladi
3. **Dead code elimination** — qaysi export ishlatilmayotganini aniqlash mumkin

**Module resolution**: har tizim o'z algoritmiga ega:
- **CommonJS**: `node_modules` traversal, parent directory'ga chiqish
- **ES Modules**: browser — URL based, Node.js — `package.json` `"exports"` field
- **Browser ESM**: relative (`./`), absolute (`/`), URL (`https://`)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Global scope pollution muammosi:

```javascript
// ❌ Modullar bo'lmasa — global scope pollution
// file1.js
var name = "Ali";
var age = 25;
function greet() { return "Salom " + name; }

// file2.js (boshqa developer yozgan)
var name = "Vali"; // ← file1.js dagi name USTIGA yozildi!
function greet() { return "Hello " + name; } // ← greet() ham override bo'ldi!

// index.html
// <script src="file1.js"></script>
// <script src="file2.js"></script>
// <script>
//   console.log(name);    // "Vali" — file1 ning name yo'qoldi!
//   console.log(greet()); // "Hello Vali" — file1 ning greet() yo'qoldi!
// </script>
```

</details>

---

## Module Pattern (IIFE)

### Nazariya

ES Modules va CommonJS dan oldin modullarni IIFE (Immediately Invoked Function Expression) bilan yaratish — JavaScript da private scope hosil qilishning yagona usuli edi. IIFE funksiya yaratib, darhol chaqiradi — funksiya ichidagi barcha o'zgaruvchilar **local scope** da qoladi, global scope'ga chiqmaydi. Bu pattern **Module Pattern** yoki **Revealing Module Pattern** deb ataladi.

Bu pattern bugun ham ba'zi hollarda uchraydi — masalan, browser da script bilan yuklangan legacy kutubxonalar (jQuery, Lodash), va bundler chiqishi (webpack bundles). Lekin zamonaviy loyihalarda ES Modules'ga almashtirilgan.

<details>
<summary><strong>Under the Hood</strong></summary>

**IIFE Module Pattern — closure asosida ishlaydi**: function darhol chaqiriladi, return qilingan object closure orqali private o'zgaruvchilarga reference saqlaydi.

```
IIFE Memory layout:

Global Scope:
  UserModule = { addUser, getUser, count } ← public API
                          ↓
                          [[Environment]] → IIFE Environment Record
                                              ├── users: []          ← private
                                              ├── nextId: 1          ← private
                                              └── validateEmail()    ← private
```

Function tugagandan keyin ham Environment Record heap'da qoladi — chunki returned object uni ushlab turadi. Bu **stale execution context** — call stack'da yo'q, lekin closure orqali tirik.

**Bundler output'da IIFE**: Webpack/Rollup/esbuild bundle'lar odatda IIFE yoki UMD formatida chiqariladi — browser'da global scope ifloslanishini oldini olish va legacy environment'larga moslik uchun.

**Performance**: IIFE darhol ishga tushadi (lazy emas). ES Modules'dan sekinroq optimization — chunki static analysis qiyin.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Classic Module Pattern:

```javascript
// IIFE — private scope yaratadi
const UserModule = (function() {
  // Private — tashqaridan ko'rinmaydi
  let users = [];
  let nextId = 1;

  function validateEmail(email) {
    return email.includes('@');
  }

  // Public API — return qilinganlar tashqaridan accessible
  return {
    addUser(name, email) {
      if (!validateEmail(email)) {
        throw new Error("Noto'g'ri email");
      }
      const user = { id: nextId++, name, email };
      users.push(user);
      return user;
    },

    getUser(id) {
      return users.find(u => u.id === id);
    },

    get count() {
      return users.length;
    }
  };
})();

// Ishlatish:
UserModule.addUser("Ali", "ali@mail.com");
console.log(UserModule.count);     // 1
console.log(UserModule.users);     // undefined — private!
console.log(UserModule.validateEmail); // undefined — private!
```

Namespace pattern — global scope'ga faqat bitta ob'ekt chiqarish:

```javascript
// Namespace — global scope'da faqat bitta nom
var MyApp = MyApp || {};

MyApp.utils = (function() {
  function formatDate(date) { /* ... */ }
  function parseJSON(str) { /* ... */ }
  return { formatDate, parseJSON };
})();

MyApp.api = (function() {
  function fetchUsers() { /* ... */ }
  return { fetchUsers };
})();

// Global scope'da faqat "MyApp" bor
MyApp.utils.formatDate(new Date());
```

</details>

---

## CommonJS — Node.js Standart

### Nazariya

CommonJS — Node.js uchun yaratilgan modul tizimi. 2009-yildan beri Node.js da standart. U `require()` bilan modul yuklaydi va `module.exports` bilan eksport qiladi.

CommonJS ning asosiy xususiyatlari:
1. **Synchronous loading** — `require()` sinxron bajariladi (faylni diskdan o'qiydi). Browser da ishlash uchun mos emas — network so'rov sinxron bo'lsa UI bloklanadi. Shu sababli Node.js uchun yaratilgan.
2. **Cached** — har bir modul **bir marta** bajariladi. Keyingi `require()` larida cache'dan olinadi (`require.cache`).
3. **Runtime** — `require()` istalgan joyda chaqirilishi mumkin — `if` ichida, funksiya ichida, loop ichida. Bu static analysis qilishni qiyinlashtiradi.
4. **Copy of value** — primitive lar export qilinganda **qiymat nusxasi** beriladi (reference emas). Object lar esa reference sifatida beriladi.

<details>
<summary><strong>Under the Hood</strong></summary>

Node.js `require()` chaqirilganda quyidagi jarayonni bajaradi:

```
require('./myModule') ichki jarayoni:

1. Resolve    → To'liq fayl yo'lini aniqlash (extension resolution: .js, .json, .node)
2. Load       → Faylni diskdan o'qish
3. Wrap       → Kodni funksiya ichiga o'rash (module wrapper)
4. Evaluate   → Wrapped funksiyani bajarish
5. Cache      → Natijani require.cache ga saqlash
```

Node.js har bir modulni maxsus **wrapper funksiya** ichiga o'raydi:

```javascript
// Biz yozamiz:
const x = 10;
module.exports = { x };

// Node.js aslida BUNI bajaradi:
(function(exports, require, module, __filename, __dirname) {
  const x = 10;
  module.exports = { x };
});
// Shu wrapper tufayli har bir modulda o'z scope bor
// exports, require, module, __filename, __dirname — shu wrapper'dan keladi
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Asosiy export/import:

```javascript
// math.js — eksport
function add(a, b) { return a + b; }
function multiply(a, b) { return a * b; }
const PI = 3.14159;

module.exports = { add, multiply, PI };

// app.js — import
const { add, multiply, PI } = require('./math');
console.log(add(2, 3));      // 5
console.log(multiply(4, 5)); // 20
console.log(PI);              // 3.14159
```

`module.exports` vs `exports` farqi — muhim nuance:

```javascript
// `exports` — module.exports ga SHORTCUT (reference)
// Boshlang'ichda: exports === module.exports → true

// ✅ To'g'ri — exports ga property qo'shish
exports.add = (a, b) => a + b;
exports.sub = (a, b) => a - b;
// module.exports = { add, sub } — chunki bir xil ob'ekt

// ❌ XATO — exports ni to'liq qayta assign qilish
exports = { add, sub };
// Bu exports o'zgaruvchisini yangi ob'ektga yo'naltiradi
// Lekin module.exports HALI ESKIsiga ishora qiladi
// Node.js module.exports ni qaytaradi — exports ni emas!

// ✅ To'g'ri — module.exports ni qayta assign qilish
module.exports = { add, sub };
// Bu to'g'ri — Node.js aynan module.exports ni export qiladi

// ✅ Class yoki funksiya export
module.exports = class Database {
  constructor(url) { /* ... */ }
};

// const Database = require('./database');
// const db = new Database('...');
```

Caching mexanizmi:

```javascript
// counter.js
let count = 0;
module.exports = {
  increment() { count++; },
  getCount() { return count; }
};

// a.js
const counter = require('./counter');
counter.increment();
counter.increment();
console.log(counter.getCount()); // 2

// b.js
const counter = require('./counter');
console.log(counter.getCount()); // 2 ← XUDDI SHU instance! Cache'dan olindi
counter.increment();
console.log(counter.getCount()); // 3

// Cache ni ko'rish:
console.log(require.cache); // barcha yuklangan modullar
// Cache ni tozalash (testing uchun):
delete require.cache[require.resolve('./counter')];
```

Conditional require — runtime da:

```javascript
// CommonJS da require() istalgan joyda ishlaydi
function loadDatabase() {
  if (process.env.DB === 'postgres') {
    return require('pg');        // faqat kerak bo'lganda yuklaydi
  } else {
    return require('better-sqlite3');
  }
}

// Loop ichida ham:
const plugins = ['auth', 'logging', 'cache'];
const loaded = plugins.map(name => require(`./plugins/${name}`));
```

</details>

---

## ES Modules — Zamonaviy Standart

### Nazariya

ES Modules (ESM) — ECMAScript standartida belgilangan rasmiy modul tizimi (ES2015+). `import` va `export` keyword'lari bilan ishlaydi. CommonJS dan farqli ravishda ESM **static** — import/export'lar compile-time da tahlil qilinadi, runtime da emas. Bu static analysis, tree shaking, va zamonaviy tooling uchun asos bo'ladi.

ESM ning asosiy xususiyatlari:
1. **Static structure** — import/export'lar faqat top-level da bo'lishi kerak (`if` ichida bo'lmaydi). Bu compile-time optimization imkonini beradi.
2. **Live bindings** — export qilingan qiymat **reference** sifatida ulangan. Source modul qiymatni o'zgartirsa — import qilgan modul ham yangi qiymatni ko'radi. CommonJS dagi "copy" dan farqli.
3. **Asynchronous** — browser da network orqali async yuklaydi (CommonJS sinxron).
4. **Strict mode** — har bir ES Module avtomatik `"use strict"` da ishlaydi.
5. **O'z scope'i** — har bir modul o'z top-level scope'iga ega (global scope'ga chiqmaydi).

<details>
<summary><strong>Under the Hood</strong></summary>

ES Module yuklash uch bosqichda amalga oshadi:

```
ES Module Loading Pipeline:

1. CONSTRUCTION (Parse)
   ├── import statement'larni topish
   ├── Modulni network/disk dan yuklash
   ├── Parse → Module Record yaratish
   └── Dependency graph qurish (static!)

2. INSTANTIATION (Link)
   ├── Har bir modul uchun memory joy ajratish
   ├── Export bindings yaratish
   ├── Import → Export bo'g'lash (live binding)
   └── Hali hech qanday kod BAJARILMAGAN

3. EVALUATION (Execute)
   ├── Modullarni dependency tartibida bajarish
   ├── Top-level kod ishlaydi
   └── Export qiymatlari aniqlanadi
```

**Live bindings** mexanizmi — CommonJS dan asosiy farq:

```javascript
// CommonJS — VALUE COPY
// counter.cjs
let count = 0;
module.exports = { count, increment() { count++; } };

// app.cjs
const { count, increment } = require('./counter.cjs');
increment();
console.log(count); // 0 ← ESKIsini ko'ryapti! Copy olgan

// ---

// ES Modules — LIVE BINDING
// counter.mjs
export let count = 0;
export function increment() { count++; }

// app.mjs
import { count, increment } from './counter.mjs';
increment();
console.log(count); // 1 ← YANGI qiymat! Live binding
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Named exports:

```javascript
// utils.mjs

// Inline export
export function formatDate(date) {
  return date.toISOString().split('T')[0];
}

export const MAX_RETRIES = 3;

export class ApiError extends Error {
  constructor(status, message) {
    super(message);
    this.status = status;
  }
}

// Yoki pastda yig'ib export:
function validate(data) { /* ... */ }
function sanitize(input) { /* ... */ }

export { validate, sanitize };

// Nomi bilan export (alias):
export { validate as validateInput, sanitize as sanitizeHTML };
```

Named imports:

```javascript
// Kerakligini import
import { formatDate, MAX_RETRIES } from './utils.mjs';

// Alias bilan import (nomi to'qnashsa)
import { formatDate as formatDateISO } from './utils.mjs';
import { formatDate as formatDateLocale } from './locale-utils.mjs';

// Namespace import — hammasini bitta ob'ektga
import * as Utils from './utils.mjs';
Utils.formatDate(new Date());
Utils.MAX_RETRIES; // 3
```

Default export:

```javascript
// database.mjs — bitta asosiy export
export default class Database {
  constructor(url) {
    this.url = url;
  }
  async connect() { /* ... */ }
  async query(sql) { /* ... */ }
}

// Import — istalgan nom bilan
import Database from './database.mjs';
import DB from './database.mjs'; // boshqa nom ham bo'ladi

// Default + named bir faylda:
export default class Database { /* ... */ }
export function createPool(config) { /* ... */ }
export const DEFAULT_PORT = 5432;

// Import:
import Database, { createPool, DEFAULT_PORT } from './database.mjs';
```

Re-exporting — barrel files (index.mjs):

```javascript
// components/Button.mjs
export function Button() { /* ... */ }

// components/Input.mjs
export function Input() { /* ... */ }

// components/Modal.mjs
export function Modal() { /* ... */ }

// components/index.mjs — BARREL FILE (re-export)
export { Button } from './Button.mjs';
export { Input } from './Input.mjs';
export { Modal } from './Modal.mjs';

// Yoki hammasi:
export * from './Button.mjs';
export * from './Input.mjs';
export * from './Modal.mjs';

// Endi bitta import bilan:
import { Button, Input, Modal } from './components/index.mjs';
```

Browser da ESM:

```html
<!-- type="module" SHART -->
<script type="module">
  import { formatDate } from './utils.mjs';
  console.log(formatDate(new Date()));
</script>

<!-- type="module" xususiyatlari: -->
<!-- 1. Avtomatik strict mode -->
<!-- 2. Avtomatik defer (DOM tayyor bo'lganda ishlaydi) -->
<!-- 3. O'z scope'i (global emas) -->
<!-- 4. CORS kerak (cross-origin) -->
<!-- 5. Har bir URL bir marta yuklaydi (cached) -->

<!-- Fallback — ESM ni qo'llab-quvvatlamaydigan browser uchun -->
<script nomodule src="fallback.js"></script>
```

</details>

---

## CommonJS vs ES Modules — To'liq Taqqoslash

### Nazariya

Bu ikki tizim o'rtasidagi farqlar fundamental — ular turli paytlarda, turli maqsadlar uchun yaratilgan:

| Xususiyat | CommonJS | ES Modules |
|-----------|----------|------------|
| **Syntax** | `require()` / `module.exports` | `import` / `export` |
| **Loading** | **Sinxron** | **Asinxron** (browser), sinxron (Node.js) |
| **Tahlil** | **Runtime** (dynamic) | **Compile-time** (static) |
| **Binding** | **Value copy** (primitives) | **Live bindings** (reference) |
| **Conditional** | ✅ `if` ichida `require()` | ❌ static import top-level da (dynamic `import()` bilan mumkin) |
| **Strict mode** | Yo'q (qo'lda qo'shish kerak) | **Avtomatik** |
| **`this`** | `exports` (dastlab `module.exports` bilan bir xil) | `undefined` |
| **Tree shaking** | ❌ Qiyin | ✅ Static tahlil tufayli oson |
| **File extension** | `.js`, `.cjs` | `.mjs` yoki `"type": "module"` |
| **Node.js** | ✅ Default | ✅ 12+ (flag), 14+ (stable) |
| **Browser** | ❌ Bundler kerak | ✅ Native `<script type="module">` |
| **Top-level await** | ❌ | ✅ (ES2022) |

<details>
<summary><strong>Kod Misollari</strong></summary>

Live binding vs value copy — eng muhim farq:

```javascript
// === CommonJS — VALUE COPY ===
// counter.cjs
let count = 0;
module.exports = { count, increment() { count++; } };

// main.cjs
const mod = require('./counter.cjs');
console.log(mod.count); // 0
mod.increment();
mod.increment();
console.log(mod.count); // 0 ← O'ZGARMADI! Export paytidagi qiymat (0) copy bo'lgan

// To'g'ri olish uchun getter kerak:
// module.exports = { get count() { return count; }, increment() { count++; } };

// === ES Modules — LIVE BINDING ===
// counter.mjs
export let count = 0;
export function increment() { count++; }

// main.mjs
import { count, increment } from './counter.mjs';
console.log(count); // 0
increment();
increment();
console.log(count); // 2 ← O'ZGARDI! Live binding — doim haqiqiy qiymat
```

Node.js da ikkala tizimni birga ishlatish (interop):

```javascript
// ESM dan CommonJS modulni import qilish:
// ✅ Default import ishlaydi
import lodash from 'lodash'; // CJS module
// CJS modulning module.exports → ESM da default export

// ❌ Named import ISHLAMAYDI (ko'p hollarda)
// import { map } from 'lodash'; // Error!
// Chunki CJS export'lari static tahlil qilinmaydi

// ✅ Workaround:
import lodash from 'lodash';
const { map, filter } = lodash;

// CommonJS dan ESM modulni import qilish:
// ❌ require() ESM modulni YUKLAMAYDI
// const mod = require('./esm-module.mjs'); // Error!

// ✅ Dynamic import() ishlatish:
const mod = await import('./esm-module.mjs');
```

</details>

---

## Dynamic Imports — import()

### Nazariya

`import()` — modulni **runtime** da, **asinxron** yuklash uchun funksiya-like syntax. U **Promise** qaytaradi — resolve bo'lganda modul ob'ekti keladi. Static `import` dan farqli ravishda, `import()` istalgan joyda ishlatilishi mumkin — `if` ichida, funksiya ichida, event handler ichida.

Dynamic import'ning asosiy use case'lari:
1. **Code splitting** — ilovani bo'laklarga ajratib, har bir bo'lakni kerak bo'lganda yuklash (performance)
2. **Conditional loading** — faqat kerakli modulni yuklash (masalan, locale bo'yicha)
3. **Lazy loading** — faqat user kerak qilganda yuklash (route-based splitting)
4. **CommonJS → ESM interop** — CJS modulda ESM import qilish uchun

<details>
<summary><strong>Under the Hood</strong></summary>

**`import()` — operator, function emas**: Spec'da `import()` syntactic operator sifatida belgilangan. Bu degani `import.call(...)`, `const fn = import` — hammasi `SyntaxError`. Engine `import()` ni alohida AST node sifatida ko'radi — bu bundler'lar code splitting va browser runtime resolution uchun ishlatadi.

**Module loading pipeline (browser)**:
```
import('./module.js')
  ↓
1. Specifier resolution → cache lookup
2. Network fetch → MIME type check (application/javascript)
3. Parse → AST → nested import'larni topish
4. Recursive resolution (cycle detection bilan)
5. Link — import binding'lar live'ga ulangan
6. Evaluate → module namespace object
7. Promise resolve(namespace)
```

**Module caching**: har resolved module cache'da saqlanadi. Ikkinchi `import('./module.js')` network request yubormaydi. Muhim: cache key URL string asosida:
```javascript
await import('./module.js');         // entry 1
await import('./module.js?v=2');     // entry 2 — turli URL
await import('./folder/../module.js'); // entry 3 — turli string
```

**Bundler code splitting**: Webpack/Rollup/Vite har `import()` chaqiruvini alohida **chunk**'ga ajratadi. Runtime'da faqat kerakli chunk fetch qilinadi — bu SPA'larda route-based lazy loading uchun asos.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Asosiy ishlatish:

```javascript
// Static import — doim yuklaydi (top-level da)
import { heavyLibrary } from './heavy.mjs'; // 500KB doim yuklanadi

// Dynamic import — kerak bo'lganda
async function processData() {
  const { heavyLibrary } = await import('./heavy.mjs');
  // Faqat shu funksiya chaqirilganda yuklanadi
  return heavyLibrary.process(data);
}
```

Route-based code splitting (SPA):

```javascript
// router.mjs — sahifa bo'yicha lazy loading
const routes = {
  '/':        () => import('./pages/Home.mjs'),
  '/about':   () => import('./pages/About.mjs'),
  '/profile': () => import('./pages/Profile.mjs'),
  '/admin':   () => import('./pages/Admin.mjs'), // faqat admin kirsa yuklanadi
};

async function navigate(path) {
  const loader = routes[path];
  if (!loader) {
    const { NotFound } = await import('./pages/NotFound.mjs');
    return NotFound();
  }

  const module = await loader();
  return module.default(); // default export — page component
}
```

Conditional loading:

```javascript
// Locale bo'yicha til faylini yuklash
async function loadTranslations(locale) {
  try {
    const translations = await import(`./i18n/${locale}.mjs`);
    return translations.default;
  } catch {
    // Fallback — default til
    const fallback = await import('./i18n/en.mjs');
    return fallback.default;
  }
}

// Platform bo'yicha
async function getStorage() {
  if (typeof window !== 'undefined') {
    const { BrowserStorage } = await import('./storage/browser.mjs');
    return new BrowserStorage();
  } else {
    const { FileStorage } = await import('./storage/node.mjs');
    return new FileStorage();
  }
}
```

Feature flag bilan lazy loading:

```javascript
async function initApp() {
  const core = await import('./core.mjs');

  if (featureFlags.analytics) {
    const { Analytics } = await import('./analytics.mjs');
    Analytics.init();
  }

  if (featureFlags.darkMode) {
    const { DarkMode } = await import('./themes/dark.mjs');
    DarkMode.apply();
  }

  core.start();
}
```

</details>

---

## Import Attributes (ES2025)

### Nazariya

Import Attributes (oldin "Import Assertions" deb atalgan) — import statement'ga qo'shimcha metadata berish imkonini beradi. Asosan JSON, CSS, va boshqa non-JavaScript modullarni type-safe import qilish uchun ishlatiladi. `with` keyword'i bilan yoziladi.

Bu xususiyat xavfsizlik uchun muhim: server JSON fayl o'rniga JavaScript qaytarishi mumkin, va bu kodni browser bajarishi — XSS hujumiga olib keladi. `with { type: 'json' }` browser ga "bu faqat JSON bo'lishi kerak" deydi — JavaScript kelsa reject qiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Tarixi**: Birinchi proposal "Import Assertions" (`assert { type: 'json' }`) — loader behavior'iga ta'sir qilmagan. Committee yangi versiyani tasdiqladi: `with` keyword'i va **loading behavior**ga ta'sir qiladigan semantika. Eski `assert` deprecated.

**MIME type security**: Browser Content-Type'ni tekshiradi:
```
GET /data.json
Content-Type: application/json  → ✅ Parse as JSON
Content-Type: text/javascript   → ❌ TypeError, rejected
```

Bu XSS himoyasi: server kompromis bo'lsa va JSON o'rniga JavaScript qaytarsa — browser uni execute qilmaydi.

**Cache key attributes bilan**: bir xil URL, turli type → turli cache entry'lar:
```javascript
import data from './file' with { type: 'json' };  // Entry 1
import data from './file' with { type: 'css' };   // Entry 2 — turli!
```

**Standart type'lar (ES2025)**:
- `type: 'json'` — browser + Node.js 22+
- `type: 'css'` — faqat browser (`CSSStyleSheet` object)
- Kelajakda: `wasm`, `html`, `text`

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// JSON import
import config from './config.json' with { type: 'json' };
console.log(config.database.host);

// CSS import (browser, bundler support)
import styles from './styles.css' with { type: 'css' };
document.adoptedStyleSheets.push(styles);

// Dynamic import bilan
const data = await import('./data.json', { with: { type: 'json' } });

// Node.js 22+ da JSON import
import pkg from './package.json' with { type: 'json' };
console.log(pkg.version);
```

</details>

---

## Circular Dependencies

### Nazariya

Circular dependency — A moduli B ni import qiladi, B esa A ni import qiladi. Bu holat real loyihalarda tez-tez uchraydi va CommonJS va ESM da **turlicha** natija beradi.

**CommonJS da:** `require()` modul kodini bajaradi. Circular bo'lganda — ikkinchi `require()` modulning **hozircha tayyor bo'lgan qismini** qaytaradi (to'liq emas). Bu kutilmagan `undefined` qiymatlarga olib keladi.

**ESM da:** Live bindings tufayli holat biroz yaxshiroq — import qilingan binding keyinchalik to'ldiriladi. Lekin initialization tartibiga e'tibor berish kerak — agar binding hali assign bo'lmagan bo'lsa, `ReferenceError` yoki `undefined` bo'lishi mumkin.

<details>
<summary><strong>Under the Hood</strong></summary>

**CommonJS — cache-based circular detection**: Node.js har modul uchun `require.cache` entry yaratadi **kod ishga tushishdan oldin**. Circular bo'lganda, ikkinchi `require()` kimga qaytsa — o'sha modulning **yarim to'la** exports object'ini oladi.

```
T0: require('a')
T1: cache['a'] = { exports: {} }  ← bo'sh entry
T2: a code executing... require('b')
T3:   cache['b'] = { exports: {} }
T4:   b code: require('a') → cache hit! a.exports = {} (yarim to'la!)
T5:   b finishes, b.exports = {fromB: ...}
T6: back to a, a.exports = {fromA: ..., b}
```

**ES Modules — 3-fazali loading**:
1. **Parsing** — import/export deklaratsiyalarini topish
2. **Linking** — binding'larni ulashish (qiymat hali yo'q, faqat reference)
3. **Evaluation** — top-level code execution, binding'lar qiymat bilan to'ldiriladi

Circular dependency **Linking faza**da hal qilinadi — binding'lar ikkala yo'nalishda ham ulanadi, qiymat keyinroq keladi.

**Live bindings — ESM vs CJS farqi**:
```javascript
// counter.mjs
export let count = 0;
export function increment() { count++; }

// app.mjs
import { count, increment } from './counter.mjs';
console.log(count); // 0
increment();
console.log(count); // 1 ← YANGILANADI! (CJS'da 0 qolar edi)
```

ESM'da import — **read-only reference** (getter), CJS'da — **value copy** (destructuring).

**ESM TDZ xatosi** — CJS dan qattiqroq: agar import qilingan binding hali initialize bo'lmagan bo'lsa, `ReferenceError` tashlaydi (CJS sukutlash bilan `undefined` qaytarardi — debugging qiyin edi).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

CommonJS da circular dependency:

```javascript
// a.cjs
console.log("a.cjs boshlandi");
exports.fromA = "A dan";
const b = require('./b.cjs'); // b ni yuklash → b ichida a ni require qiladi
console.log("a.cjs da b.fromB:", b.fromB);

// b.cjs
console.log("b.cjs boshlandi");
const a = require('./a.cjs'); // a HALI TUGAMAGAN → hozircha tayyor qismi olinadi
console.log("b.cjs da a.fromA:", a.fromA); // "A dan" ← bu tayyor
exports.fromB = "B dan";

// node a.cjs output:
// a.cjs boshlandi
// b.cjs boshlandi
// b.cjs da a.fromA: A dan   ← tayyor qismi (exports.fromA assign bo'lgan)
// a.cjs da b.fromB: B dan
```

Circular dependency'ni oldini olish:

```javascript
// ❌ Circular: A ↔ B
// a.mjs: import { x } from './b.mjs'
// b.mjs: import { y } from './a.mjs'

// ✅ Yechim 1: Umumiy dependency'ni alohida modulga chiqarish
// shared.mjs — A va B ning umumiy qismi
// a.mjs: import { shared } from './shared.mjs'
// b.mjs: import { shared } from './shared.mjs'

// ✅ Yechim 2: Dependency injection
// a.mjs
export function createA(b) {
  return { doSomething: () => b.getValue() };
}

// ✅ Yechim 3: Dynamic import (lazy)
// a.mjs
export async function getB() {
  const { B } = await import('./b.mjs');
  return new B();
}

// ✅ Yechim 4: Mediator pattern
// mediator.mjs — A va B o'rtasidagi aloqani boshqaradi
```

</details>

---

## Module Bundlers

### Nazariya

Module bundler — bir nechta modul fayllarni **bitta yoki bir nechta bundle** faylga birlashtiradigan tool. Browser da yuzlab `import` statement'lar yuzlab HTTP so'rovlarga aylanadi — bu sekin. Bundler buni bir-ikkita faylga qisqartiradi.

Zamonaviy bundler'lar:

| Bundler | Xususiyat | Tezlik | Use Case |
|---------|-----------|--------|----------|
| **Webpack** | Eng to'liq, plugin ekotizimi katta | O'rtacha | Katta production loyihalar |
| **Vite** | Dev da bundlamasdan ishlaydi (ESM), build da Rollup | Juda tez (dev) | Zamonaviy SPA (React, Vue, Svelte) |
| **esbuild** | Go da yozilgan, juda tez | Eng tez | CLI toollar, library build |
| **Rollup** | Tree shaking yaxshi, library uchun ideal | Tez | NPM kutubxonalar |
| **Parcel** | Zero-config | Tez | Kichik/o'rta loyihalar |

<details>
<summary><strong>Under the Hood</strong></summary>

**Bundler 5 fazali jarayon**:
1. **Entry resolution** — entry point'larni topish
2. **Dependency graph** — har faylni parse qilish, import'larni recursive traverse qilish
3. **Transformation** — loader'lar (TS → JS, SCSS → CSS, JSX), tree shaking, dead code elimination
4. **Chunk splitting** — dynamic import boundary'lari alohida chunk'ga, vendor chunks
5. **Output generation** — module wrapping (IIFE/UMD/ESM), minification, source maps, file hashing

**Module resolution algorithm** — bundler `import './foo'` uchun:
```
1. ./foo.js
2. ./foo.json
3. ./foo/index.js
4. ./foo/package.json (main/module/exports field)
```
Bare specifier (`'react'`) uchun `node_modules` traversal — parent directory'ga chiqadi.

**Vite — farqli approach**: dev mode'da **bundlamaydi**, browser native ESM'ni ishlatadi. Har module alohida HTTP request orqali yuboriladi (HTTP/2 multiplexing). Bu dev startup'ni sezilarli tezlashtiradi — ayniqsa katta loyihalarda farq yaqqol sezilarli (aniq farq project hajmi, dependency'lar soni va cache holatiga bog'liq). Production'da esa Rollup bilan bundle qiladi.

**esbuild/swc nima uchun tez?** — Go/Rust'da yozilgan native tool'lar, parallel ishlaydi. Babel/Webpack esa JavaScript'da — single-threaded va V8 GC overhead'i bor. Native tool'lar sezilarli tezroq bo'lishi mumkin (aniq farq workload va benchmark setup'iga bog'liq — single file transform'da farq kattaroq, end-to-end bundle'da esa transformation'dan tashqari plugin'lar ham hissasiz o'zgartiradi). Shu sababli Vite, Next.js (swc), Parcel (swc) native tool'larga o'tmoqda.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Bundler nima qiladi — soddalashtirilgan misol:

```javascript
// INPUT: 3 ta fayl

// math.mjs
export const add = (a, b) => a + b;
export const sub = (a, b) => a - b;

// utils.mjs
import { add } from './math.mjs';
export const double = x => add(x, x);

// app.mjs (entry point)
import { double } from './utils.mjs';
console.log(double(21)); // 42

// ─────────────────────────────────────

// OUTPUT: 1 ta bundle fayl (soddalashtirilgan)
// bundle.js
const add = (a, b) => a + b;
const double = x => add(x, x);
console.log(double(21)); // 42
// `sub` — ishlatilmagan → tree shaking bilan olib tashlangan
```

</details>

---

## Tree Shaking

### Nazariya

Tree shaking — bundle'dan **ishlatilmagan kodni** olib tashlash jarayoni. Bundler dependency graph'ni tahlil qilib, hech qayerda import qilinmagan export'larni **dead code** deb belgilaydi va bundle'dan olib tashlaydi. Bu bundle hajmini sezilarli kamaytiradi.

Tree shaking **faqat ES Modules** bilan samarali ishlaydi. Sabab: ESM import/export'lari **static** — compile-time da qaysi export ishlatilganini aniqlash mumkin. CommonJS `require()` **dynamic** — runtime da qaysi property olinishini oldindan bilish mumkin emas.

<details>
<summary><strong>Under the Hood</strong></summary>

```
Tree Shaking jarayoni:

1. Dependency Graph qurish
   app.mjs → utils.mjs → math.mjs

2. Export/Import tahlil
   math.mjs exports: { add, sub, multiply }
   utils.mjs imports from math: { add }  ← faqat add
   app.mjs imports from utils: { double } ← faqat double

3. Mark — ishlatilganlarni belgilash
   ✅ add     — utils.mjs da ishlatilgan
   ✅ double  — app.mjs da ishlatilgan
   ❌ sub     — hech qayerda import qilinmagan
   ❌ multiply — hech qayerda import qilinmagan

4. Sweep — ishlatilmaganlarni olib tashlash
   sub, multiply — bundle'dan O'CHIRILADI
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Tree shaking ishlashi va ishlamasligi:

```javascript
// ✅ Tree shaking ISHLAYDI — named import
import { map } from 'lodash-es'; // faqat map yuklanadi, qolgani olib tashlanadi

// ❌ Tree shaking ISHLAMAYDI — butun kutubxona
import _ from 'lodash'; // BUTUN lodash bundle'ga tushadi (70KB+)
_.map([1,2,3], x => x*2);

// ❌ Tree shaking ISHLAMAYDI — CommonJS
const { map } = require('lodash'); // runtime da — bundler bilmaydi

// ❌ Tree shaking ISHLAMAYDI — side effects
// analytics.mjs
console.log("Analytics loaded!"); // side effect — olib tashlab bo'lmaydi
export function track() { /* ... */ }

// Bundler bu faylni olib tashlay olmaydi — console.log side effect bor
```

`sideEffects` field — package.json da bundler'ga yordam:

```json
{
  "name": "my-library",
  "sideEffects": false
}
```

`"sideEffects": false` — bundler'ga "bu kutubxonada side effect yo'q, ishlatilmagan fayllarni bemalol olib tashla" deydi.

```json
{
  "sideEffects": ["./src/polyfills.js", "*.css"]
}
```

Faqat ko'rsatilgan fayllar side effect'li — qolganlari safe to remove.

</details>

---

## Edge Cases va Gotchas

### ESM import'lar **hoisted** — side effects faylning `console.log` dan oldin ishlaydi

Static `import` deklaratsiyalari har doim modul tepasiga **hoisted** qilinadi. Siz ularni fayl o'rtasida yoki oxirida yozsangiz ham — engine ularni birinchi navbatda evaluate qiladi. Bu degani: import qilingan modul'ning top-level kodi (side effects) sizning **har qanday koddingizdan oldin** bajariladi.

```javascript
// analytics.mjs
console.log("analytics loaded"); // side effect

// app.mjs
console.log("app start");
import './analytics.mjs'; // syntactically pastda, lekin hoisted
console.log("app end");

// Output:
// analytics loaded   ← Import side effect BIRINCHI!
// app start
// app end
```

**Nima uchun:** ECMAScript spec'da ESM 3 bosqichli loading'ga ega: **Parse → Link → Evaluate**. Parse bosqichida engine barcha `import` deklaratsiyalarini topadi (ular qaerda yozilganidan qat'i nazar) va dependency graph'ga qo'shadi. Evaluation bosqichida **dependency tartibida** modullar bajariladi — shuning uchun import qilingan modul side effect'lari har doim sizning kodi'ngizdan oldin ishlaydi.

**Amaliy ta'sir:**
- **Conditional import ishlamaydi** — `if (cond) import './foo.mjs'` SyntaxError. Static import runtime shart'larga bog'liq bo'la olmaydi. Shart kerak bo'lsa — `import()` (dynamic).
- **Side effect'ga tayangan kod xavfli** — agar sizning "birinchi qator" `console.log("start")` bo'lsa, aslida u birinchi emas. Imported modul log'lari oldin chiqadi.

**Yechim:** Side effect'ni kutilgan joyga qo'yish kerak bo'lsa — **dynamic import** ishlating:

```javascript
console.log("app start");
await import('./analytics.mjs'); // endi bu yerda
console.log("app end");
```

---

### Dynamic `import()` cache key — URL string, semantik normalizatsiya YO'Q

`import()` har chaqirilganda module'ni cache'dan oladi — lekin cache key **aynan shu string**. Bir xil module'ning turli yozuvlari (`./a.mjs` va `./folder/../a.mjs`) **turli cache entry** yaratadi, memory waste va unexpected behavior keltirib chiqaradi.

```javascript
// ❌ Turli string → turli cache entry (lekin bir xil fayl!)
await import('./module.mjs');            // Entry 1
await import('./folder/../module.mjs');  // Entry 2 — "turli URL" deb hisoblangan
await import('./module.mjs?v=2');        // Entry 3 — query string turli
await import('./MODULE.mjs');            // Entry 4 — case-sensitive OS'larda turli

// Har biri alohida network fetch + parse + evaluation
// Har biri alohida module namespace object — memory bo'yicha alohida
```

**Nima uchun:** ESM spec'da module identity **Module Specifier Resolution** natijasi bilan aniqlanadi. Browser URL'ni normalize qiladi (masalan `/./` → `/`), lekin **query string** va **fragment** (`#`) hisobga olinadi. Node.js `?v=2` ni turli entry deb biladi — shu sababli cache busting uchun (HMR kabi) bu ishlatiladi.

**Amaliy oqibatlar:**
- **Memory waste** — bir xil modul bir necha marta yuklanadi
- **Singleton pattern buziladi** — har import yangi module state'ini yaratadi
- **Unexpected duplication** — `import './shared.mjs'` va `import '/abs/path/shared.mjs'` — ikkalasi bir xil faylga ishora qilsa ham turli modullar

**Yechim:**
- **Bitta canonical path** ishlating — import'larni normalize qiling (tooling, ESLint rule)
- **Cache busting** kerak bo'lsa — query string ataylab ishlating (HMR, versioning)
- **`import.meta.resolve()`** (ES2024+) — URL'ni canonical form'ga o'tkazish uchun

```javascript
// ✅ import.meta.resolve — standart yo'l
const url = import.meta.resolve('./module.mjs'); // absolute URL
const module = await import(url);
```

---

### `import *` namespace imports — tree shaking **ishlamaydi**

`import * as Lib from 'library'` — butun namespace'ni import qiladi. Bundler tree shake qila olmaydi — chunki namespace object dinamik ravishda accessible (`Lib[someKey]`) va static analiz xavfsizlik uchun **butun modul**ni saqlab qoladi.

```javascript
// ❌ Tree shaking ISHLAMAYDI — butun library bundle'ga tushadi
import * as lodash from 'lodash-es';
lodash.map([1, 2, 3], x => x * 2); // Faqat map ishlatilmoqda, lekin butun lodash bundled

// Bundler fikrlashi: "Lib['map']" yozish mumkin edi, shuning uchun HAMMA narsa kerak
// Result: ~70KB bundle (~4KB map o'rniga)

// ✅ Tree shaking ISHLAYDI — named imports
import { map, filter } from 'lodash-es';
map([1, 2, 3], x => x * 2); // Faqat map + filter bundle'da

// Yoki qaytroq — faqat kerakli fayl:
import map from 'lodash-es/map.js';
map([1, 2, 3], x => x * 2); // Eng minimal bundle
```

**Nima uchun:** Tree shaking `mark & sweep` algoritmi bilan ishlaydi — bundler dependency graph bo'ylab yurib, ishlatilgan export'larni mark qiladi. Lekin `namespace[key]` pattern — bu **dynamic property access**, compile-time'da qaysi key ishlatilishi noma'lum. Bundler **xavfsizlik** uchun butun namespace'ni saqlab qoladi (agar ba'zi key runtime'da ishlatilsa, ular mavjud bo'lishi kerak).

**Xavfli tomoni:** Kutubxona namespace import bilan yozilgan bo'lsa (masalan React types), foydalanuvchi bundle ekstra KB'lar bilan sarflanadi — development'da ko'rinmaydi, lekin production bundle'da sezilarli.

**Yechim:**
- **Named imports** — har doim afzal
- **Sub-path imports** — `lodash-es/map.js` kabi (eng mayda granulatsiya)
- **Namespace'ni faqat type-only yoki debug uchun** ishlating (masalan TypeScript `import type *`)

---

### `package.json` conditional exports — import va require turli fayllarga resolve bo'lishi

Modern kutubxonalar `package.json`'da **conditional exports** ishlatadi — bir xil modul nomi turli kontekstlarda (ESM, CJS, browser, Node.js) **turli fayllarga** resolve bo'ladi. Bu foydali, lekin debug'da chalkashlik manbai.

```json
// package.json
{
  "name": "my-lib",
  "exports": {
    ".": {
      "import": "./dist/esm/index.mjs",   // ESM uchun
      "require": "./dist/cjs/index.cjs",  // CJS uchun
      "browser": "./dist/browser/index.js", // Browser bundler uchun
      "default": "./dist/fallback.js"
    },
    "./utils": {
      "import": "./dist/esm/utils.mjs",
      "require": "./dist/cjs/utils.cjs"
    }
  }
}
```

**Gotcha — ikki holat bir loyihada:**

```javascript
// ESM fayl (app.mjs)
import lib from 'my-lib';         // → ./dist/esm/index.mjs
console.log(lib.VERSION);          // "1.0.0"

// CJS fayl (legacy.cjs)
const lib = require('my-lib');     // → ./dist/cjs/index.cjs
console.log(lib.VERSION);          // "1.0.0"

// ❌ Lekin: state alohida!
// import'dan kelgan lib va require'dan kelgan lib —
// IKKITA alohida modul instance, alohida state, alohida cache
```

**Nima uchun:** Har resolution context (import vs require) `package.json.exports` field'ning turli branch'iga boradi. ESM va CJS har biri alohida module system — ular orasida module state **share qilmaydi**. Bu "dual package hazard" deb ataladi va singleton pattern'larni buzadi (masalan, React bir xil versiyada ikki marta yuklanishi mumkin — ESM va CJS alohida).

**Yechim:**
- **Loyihada bitta module system'ni ishlating** — `"type": "module"` (ESM) yoki to'liq CJS
- **Library muallif'lari** uchun: `package.json.exports.import` va `require` **bir xil JavaScript state'ga qaytsin** (masalan, ikkalasi ham re-export pattern orqali bitta core file'dan)
- **Debug:** Node.js `--experimental-specifier-resolution` va `--conditions` flag'lari bilan resolution tracing

---

### Side-effect-only imports — `import './polyfill.mjs'` tree shaking'ga chidamli

`import './foo.mjs'` (named/default import'siz) — "shu modulni ishga tushir, ma'lumot olma" ma'nosida. Bu pattern **polyfill'lar**, **global registration** (Web Components), **CSS injection** uchun ishlatiladi. Bundler uni **tree shake qila olmaydi**, chunki side effect kutilmoqda.

```javascript
// ✅ Polyfill import — tree shaking OLIB TASHLAMAYDI
import 'core-js/stable';                // Butun polyfill
import 'regenerator-runtime/runtime';    // Generator polyfill

// Hech qanday variable assign qilinmaydi, lekin faylning top-level kodi ishlaydi
// → polyfill'lar globalThis'ga Array.prototype.flat kabi method'lar qo'shadi

// Web Components registration
import './my-element.mjs'; // customElements.define() ichida chaqiradi

// CSS modules (bundler support)
import './styles.css'; // bundler CSS'ni document.head'ga qo'shadi

// ✅ Side-effect + named import — ikkalasi ham ishlaydi
import { something } from './module.mjs'; // named + side effects
```

**Gotcha — accidental tree shaking:**

Agar sizning library side-effect'ga tayansa va `package.json`'da `"sideEffects": false` belgilansa — bundler sizning modulni ishlatilmasa **olib tashlaydi**, hatto import bo'lgan bo'lsa ham:

```json
// package.json (NOTO'G'RI sozlama):
{
  "sideEffects": false
}

// Foydalanuvchi kodi:
import 'my-lib/register'; // Customisers.define() chaqiradi

// ❌ Bundler bu faylni olib tashlaydi — "hech qanday import yo'q, side effect yo'q deb aytilgan"
// Web Component ro'yxatdan o'tmaydi, crash production'da
```

**Nima uchun:** `sideEffects: false` bundler'ga ruxsat beradi — "ishlatilmagan import'larni olib tashlash xavfsiz". Agar moduling side effect'ga tayansa (polyfill, register), bu sozlama **buzuq bundle** yaratadi.

**Yechim:**
```json
// ✅ Aniq ayting — qaysi fayllarda side effect bor
{
  "sideEffects": [
    "./src/polyfills.js",
    "./src/register.js",
    "*.css"  // CSS har doim side-effect
  ]
}
```

Bu bundler'ga aniq signal beradi: "qolgan fayllar pure, lekin bular side effect bor — olib tashlamang".

---

## Common Mistakes

### ❌ Xato 1: `exports` ni Qayta Assign Qilish (CommonJS)

```javascript
// ❌ XATO — exports o'zgaruvchisi yangi ob'ektga ishora qildi
// lekin Node.js module.exports ni export qiladi
exports = { add, sub }; // module.exports HALI ESKIsiga ishora!

// require('./module') → {} (bo'sh ob'ekt)
```

### ✅ To'g'ri usul:

```javascript
// ✅ module.exports ga assign
module.exports = { add, sub };

// ✅ Yoki exports ga property qo'shish
exports.add = add;
exports.sub = sub;
```

**Nima uchun:** `exports` — `module.exports` ga shortcut. `exports = {...}` bilan siz shortcut'ni o'zgartirdingiz, lekin `module.exports` hali eski ob'ektga ishora qiladi. Node.js faqat `module.exports` ni qaytaradi.

---

### ❌ Xato 2: Named va Default Import Aralashtirib Yuborish

```javascript
// module.mjs
export function helper() { /* ... */ }
export default class App { /* ... */ }

// ❌ XATO — default import uchun {} ishlatildi
import { App } from './module.mjs'; // Bu NAMED import — App topilmaydi!

// ❌ XATO — named import uchun {} YO'Q
import helper from './module.mjs'; // Bu DEFAULT import — App olinadi, helper emas!
```

### ✅ To'g'ri usul:

```javascript
// ✅ Default — {} siz
import App from './module.mjs';

// ✅ Named — {} bilan
import { helper } from './module.mjs';

// ✅ Ikkalasi birga
import App, { helper } from './module.mjs';
```

**Nima uchun:** `import X from ...` — default export. `import { X } from ...` — named export. Bu ikki turli mexanizm.

---

### ❌ Xato 3: ESM da require() Ishlatish

```javascript
// ❌ ES Module da require() ishlamaydi
// module.mjs
const fs = require('fs'); // ReferenceError: require is not defined
```

### ✅ To'g'ri usul:

```javascript
// ✅ ESM da import ishlatish
import fs from 'fs';
import { readFile } from 'fs/promises';

// ✅ Agar CJS modul kerak bo'lsa — createRequire
import { createRequire } from 'module';
const require = createRequire(import.meta.url);
const cjsModule = require('./legacy.cjs');
```

**Nima uchun:** ESM da `require`, `module`, `exports`, `__filename`, `__dirname` — yo'q. Bular CommonJS wrapper'dan keladi. ESM da `import.meta.url` va `import` ishlatiladi.

---

### ❌ Xato 4: Barrel File Performance Muammosi

```javascript
// components/index.mjs
export * from './Button.mjs';
export * from './Input.mjs';
export * from './Modal.mjs';
export * from './Table.mjs';    // 500 qator
export * from './Chart.mjs';    // 2000 qator + d3 dependency
// ... 50 ta component

// app.mjs
import { Button } from './components/index.mjs';
// ❌ Faqat Button kerak — lekin 50 ta component HAMMASI parse bo'ladi!
// Tree shaking ishlaydi, lekin parse/build vaqti ko'payadi
```

### ✅ To'g'ri usul:

```javascript
// ✅ Direct import — faqat kerakli fayl yuklanadi
import { Button } from './components/Button.mjs';

// ✅ Yoki sub-path exports (package.json):
// { "exports": { "./Button": "./src/components/Button.mjs" } }
import { Button } from 'my-lib/Button';
```

**Nima uchun:** Barrel files kichik loyihalarda qulay, lekin kattalashganda build vaqtini oshiradi. Bundler barcha fayllarni parse qiladi, keyin ishlatilmaganlarini olib tashlaydi — bu ortiqcha ish.

---

### ❌ Xato 5: Circular Dependency Muammosi

```javascript
// ❌ a.mjs
import { b } from './b.mjs';
export const a = "A + " + b;

// b.mjs
import { a } from './a.mjs';
export const b = "B + " + a;

// a import qilinganda: a = "A + B + undefined"
// Chunki a hali assign bo'lmagan paytda b uni ishlatadi
```

### ✅ To'g'ri usul:

```javascript
// ✅ Umumiy dependency'ni alohida modulga chiqarish
// shared.mjs
export const shared = "Shared value";

// a.mjs
import { shared } from './shared.mjs';
export const a = "A + " + shared;

// b.mjs
import { shared } from './shared.mjs';
export const b = "B + " + shared;
```

**Nima uchun:** Circular dependency'da initialization tartibi noaniq bo'ladi. Eng yaxshi yechim — arxitekturani qayta ko'rib chiqish va circular'ni yo'qotish.

---

## Amaliy Mashqlar

### Mashq 1: Module Pattern (Oson)

**Savol:** IIFE module pattern bilan `Counter` modul yarating — `increment()`, `decrement()`, `getCount()`, `reset()` public method'lari bo'lsin, `count` private bo'lsin.

<details>
<summary>Javob</summary>

```javascript
const Counter = (function() {
  let count = 0; // private

  return {
    increment() { count++; return this; },
    decrement() { count--; return this; },
    getCount() { return count; },
    reset() { count = 0; return this; },
  };
})();

Counter.increment().increment().increment();
console.log(Counter.getCount()); // 3
Counter.decrement();
console.log(Counter.getCount()); // 2
Counter.reset();
console.log(Counter.getCount()); // 0

console.log(Counter.count); // undefined — private!
```

**Tushuntirish:** IIFE closure yaratadi — `count` tashqaridan accessible emas. Return qilingan ob'ekt public API. Method chaining uchun `return this`.

</details>

---

### Mashq 2: ESM vs CJS Farqini Tushunish (O'rta)

**Savol:** Quyidagi kodning output nima bo'ladi? CJS va ESM da farqi nima?

```javascript
// CJS versiya:
// counter.cjs
let count = 0;
module.exports = { count, increment() { count++; } };

// main.cjs
const { count, increment } = require('./counter.cjs');
increment(); increment();
console.log(count); // ?
```

```javascript
// ESM versiya:
// counter.mjs
export let count = 0;
export function increment() { count++; }

// main.mjs
import { count, increment } from './counter.mjs';
increment(); increment();
console.log(count); // ?
```

<details>
<summary>Javob</summary>

```
CJS: 0   — value copy (export paytidagi qiymat)
ESM: 2   — live binding (doim haqiqiy qiymat)
```

**Tushuntirish:**
- **CJS:** `module.exports = { count }` — export paytida `count` ning qiymati (0) **copy** qilinadi. `increment()` ichki `count` ni o'zgartiradi, lekin export qilingan `count` property hali 0.
- **ESM:** `export let count` — live binding. `count` import qilingan joyda har doim **haqiqiy qiymat**ga ishora qiladi. `increment()` o'zgartirsa — import qilgan joy ham ko'radi.

</details>

---

### Mashq 3: Dynamic Import bilan Lazy Loading (Qiyin)

**Savol:** `loadFeature(name)` funksiyasini yozing:
- Feature'ni dynamic import bilan yuklash
- Cache qilish (bir marta yuklangan modulni qayta yuklamaslik)
- Xatolik bo'lsa fallback qaytarish

<details>
<summary>Javob</summary>

```javascript
const moduleCache = new Map();

async function loadFeature(name) {
  // Cache tekshirish
  if (moduleCache.has(name)) {
    return moduleCache.get(name);
  }

  try {
    const module = await import(`./features/${name}.mjs`);
    const feature = module.default || module;

    // Cache ga saqlash
    moduleCache.set(name, feature);
    return feature;
  } catch (err) {
    console.error(`Feature "${name}" yuklanmadi:`, err.message);

    // Fallback
    return {
      init() { console.warn(`${name} — fallback mode`); },
      isLoaded: false,
    };
  }
}

// Ishlatish:
const analytics = await loadFeature('analytics');
analytics.init();

// Qayta chaqirish — cache'dan olinadi (network so'rov yo'q):
const analytics2 = await loadFeature('analytics');
console.log(analytics === analytics2); // true — bir xil instance
```

**Tushuntirish:** `Map` cache sifatida ishlatiladi — module yuklanganidan keyin saqlaydi. Dynamic `import()` xato bersa — fallback ob'ekt qaytariladi. Bu pattern production da feature flag'lar bilan birgalikda ishlatiladi.

</details>

---

## Xulosa

1. **Modullar global scope pollution ni hal qiladi** — har bir modul o'z scope'iga ega, faqat export qilinganlar tashqaridan ko'rinadi.

2. **Module Pattern (IIFE)** — ES Modules dan oldingi yechim. Closure orqali private scope yaratadi. Legacy kodda hali uchraydi.

3. **CommonJS** — Node.js standart (`require`/`module.exports`). Sinxron, runtime, value copy. `module.exports` vs `exports` farqini bilish muhim.

4. **ES Modules** — ECMAScript standart (`import`/`export`). Static, live bindings, avtomatik strict mode. Zamonaviy JavaScript'ning asosi.

5. **Live bindings vs Value copy** — ESM da export o'zgarsa import ham ko'radi. CJS da esa export paytidagi qiymat copy bo'ladi.

6. **Dynamic import (`import()`)** — runtime da modul yuklash. Code splitting, lazy loading, conditional loading uchun. Promise qaytaradi.

7. **Import Attributes (ES2025)** — `with { type: 'json' }` — non-JS modullarni type-safe import qilish.

8. **Circular dependencies** — A→B→A. Muammoga olib keladi. Yechim: umumiy dependency'ni alohida modulga chiqarish yoki arxitekturani o'zgartirish.

9. **Tree shaking** — ishlatilmagan kodni bundle'dan olib tashlash. Faqat ESM bilan samarali ishlaydi (static analysis tufayli).

10. **Bundlers** — Vite (dev tez), Webpack (to'liq), esbuild (eng tez), Rollup (library uchun).

---

> **Oldingi bo'lim:** [14-iterators-generators.md](14-iterators-generators.md) — Iterators va Generators: Symbol.iterator, yield, async generators, lazy evaluation.
>
> **Keyingi bo'lim:** [16-memory.md](16-memory.md) — Memory Management: stack vs heap, garbage collection, memory leaks, WeakRef, FinalizationRegistry.
