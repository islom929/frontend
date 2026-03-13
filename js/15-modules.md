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

### Kod Misollari

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

---

## Module Pattern (IIFE)

### Nazariya

ES Modules va CommonJS dan oldin modullarni IIFE (Immediately Invoked Function Expression) bilan yaratish — JavaScript da private scope hosil qilishning yagona usuli edi. IIFE funksiya yaratib, darhol chaqiradi — funksiya ichidagi barcha o'zgaruvchilar **local scope** da qoladi, global scope'ga chiqmaydi. Bu pattern **Module Pattern** yoki **Revealing Module Pattern** deb ataladi.

Bu pattern bugun ham ba'zi hollarda uchraydi — masalan, browser da script bilan yuklangan legacy kutubxonalar (jQuery, Lodash), va bundler chiqishi (webpack bundles). Lekin zamonaviy loyihalarda ES Modules'ga almashtirilgan.

### Kod Misollari

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

---

## CommonJS — Node.js Standart

### Nazariya

CommonJS — Node.js uchun yaratilgan modul tizimi. 2009-yildan beri Node.js da standart. U `require()` bilan modul yuklaydi va `module.exports` bilan eksport qiladi.

CommonJS ning asosiy xususiyatlari:
1. **Synchronous loading** — `require()` sinxron bajariladi (faylni diskdan o'qiydi). Browser da ishlash uchun mos emas — network so'rov sinxron bo'lsa UI bloklanadi. Shu sababli Node.js uchun yaratilgan.
2. **Cached** — har bir modul **bir marta** bajariladi. Keyingi `require()` larida cache'dan olinadi (`require.cache`).
3. **Runtime** — `require()` istalgan joyda chaqirilishi mumkin — `if` ichida, funksiya ichida, loop ichida. Bu static analysis qilishni qiyinlashtiradi.
4. **Copy of value** — primitive lar export qilinganda **qiymat nusxasi** beriladi (reference emas). Object lar esa reference sifatida beriladi.

### Under the Hood

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

### Kod Misollari

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

### Under the Hood

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

### Kod Misollari

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
| **`this`** | `module.exports` | `undefined` |
| **Tree shaking** | ❌ Qiyin | ✅ Static tahlil tufayli oson |
| **File extension** | `.js`, `.cjs` | `.mjs` yoki `"type": "module"` |
| **Node.js** | ✅ Default | ✅ 12+ (flag), 14+ (stable) |
| **Browser** | ❌ Bundler kerak | ✅ Native `<script type="module">` |
| **Top-level await** | ❌ | ✅ (ES2022) |

### Kod Misollari

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

---

## Dynamic Imports — import()

### Nazariya

`import()` — modulni **runtime** da, **asinxron** yuklash uchun funksiya-like syntax. U **Promise** qaytaradi — resolve bo'lganda modul ob'ekti keladi. Static `import` dan farqli ravishda, `import()` istalgan joyda ishlatilishi mumkin — `if` ichida, funksiya ichida, event handler ichida.

Dynamic import'ning asosiy use case'lari:
1. **Code splitting** — ilovani bo'laklarga ajratib, har bir bo'lakni kerak bo'lganda yuklash (performance)
2. **Conditional loading** — faqat kerakli modulni yuklash (masalan, locale bo'yicha)
3. **Lazy loading** — faqat user kerak qilganda yuklash (route-based splitting)
4. **CommonJS → ESM interop** — CJS modulda ESM import qilish uchun

### Kod Misollari

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

---

## Import Attributes (ES2025)

### Nazariya

Import Attributes (oldin "Import Assertions" deb atalgan) — import statement'ga qo'shimcha metadata berish imkonini beradi. Asosan JSON, CSS, va boshqa non-JavaScript modullarni type-safe import qilish uchun ishlatiladi. `with` keyword'i bilan yoziladi.

Bu xususiyat xavfsizlik uchun muhim: server JSON fayl o'rniga JavaScript qaytarishi mumkin, va bu kodni browser bajarishi — XSS hujumiga olib keladi. `with { type: 'json' }` browser ga "bu faqat JSON bo'lishi kerak" deydi — JavaScript kelsa reject qiladi.

### Kod Misollari

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

---

## Circular Dependencies

### Nazariya

Circular dependency — A moduli B ni import qiladi, B esa A ni import qiladi. Bu holat real loyihalarda tez-tez uchraydi va CommonJS va ESM da **turlicha** natija beradi.

**CommonJS da:** `require()` modul kodini bajaradi. Circular bo'lganda — ikkinchi `require()` modulning **hozircha tayyor bo'lgan qismini** qaytaradi (to'liq emas). Bu kutilmagan `undefined` qiymatlarga olib keladi.

**ESM da:** Live bindings tufayli holat biroz yaxshiroq — import qilingan binding keyinchalik to'ldiriladi. Lekin initialization tartibiga e'tibor berish kerak — agar binding hali assign bo'lmagan bo'lsa, `ReferenceError` yoki `undefined` bo'lishi mumkin.

### Kod Misollari

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

### Kod Misollari

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

---

## Tree Shaking

### Nazariya

Tree shaking — bundle'dan **ishlatilmagan kodni** olib tashlash jarayoni. Nomi "daraxtni silkitish" dan olingan — daraxtni silkitsangiz qurugan barglar tushadi. Bundler dependency graph'ni tahlil qilib, hech qayerda import qilinmagan export'larni **dead code** deb belgilaydi va bundle'dan olib tashlaydi. Bu bundle hajmini sezilarli kamaytiradi.

Tree shaking **faqat ES Modules** bilan samarali ishlaydi. Sabab: ESM import/export'lari **static** — compile-time da qaysi export ishlatilganini aniqlash mumkin. CommonJS `require()` **dynamic** — runtime da qaysi property olinishini oldindan bilish mumkin emas.

### Under the Hood

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

### Kod Misollari

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
