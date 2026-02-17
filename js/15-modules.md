# Bo'lim 15: Modules — Kodni Tashkillashtirish Sanʼati

> Module — kodni mustaqil, qayta ishlatiladigan bo'laklarga ajratish tizimi. Global scope ifloslanishining yechimi, zamonaviy JavaScript arxitekturasining asosi.

---

## Mundarija

- [Nima Uchun Modullar Kerak](#nima-uchun-modullar-kerak)
- [Module Pattern (IIFE)](#module-pattern-iife)
- [CommonJS — Node.js Standart](#commonjs--nodejs-standart)
- [ES Modules — Zamonaviy Standart](#es-modules--zamonaviy-standart)
- [CommonJS vs ES Modules — To'liq Taqqoslash](#commonjs-vs-es-modules--toliq-taqqoslash)
- [Dynamic Imports — import()](#dynamic-imports--import)
- [Circular Dependencies](#circular-dependencies)
- [Module Bundlers](#module-bundlers)
- [Tree Shaking](#tree-shaking)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Nima Uchun Modullar Kerak

### Nazariya

Modullar mavjud bo'lmasdan oldingi dunyoni tasavvur qilaylik: barcha JavaScript bitta global scope'da yashaydi. Loyiha o'sgan sari muammolar ham eksponensial o'sadi — global scope pollution (o'zgaruvchilar bir-birini ustiga yozadi), namespace collision (turli fayllar bir xil nom ishlatadi), dependency management yo'qligi (qaysi fayl qaysi faylga bog'liq ekanini bilish imkonsiz), va maintainability pasayishi.

JavaScript dastlab modullar tizimisiz yaratilgan, chunki 1995-yilda u faqat oddiy browser skripting uchun mo'ljallangan edi. Lekin tilning popularlik ortishi bilan modul tizimlari paydo bo'ldi: 2009-da Node.js uchun CommonJS, 2015-da ES6 bilan rasmiy ES Modules standart. Har qanday modul tizimi uchta asosiy muammoni hal qiladi: **encapsulation** (private/public ajratish), **dependency management** (bog'liqliklarni aniq ifodalash), va **namespace isolation** (nomlar to'qnashmasligini ta'minlash).

```javascript
// file1.js
var userName = "Ali";
var calculateTotal = function() { /* ... */ };

// file2.js
var userName = "Vali"; // ❌ Ali yo'qoldi!
var calculateTotal = function() { /* boshqa narsa */ }; // ❌ Oldingi funksiya yo'q bo'ldi!

// HTML da
// <script src="file1.js"></script>
// <script src="file2.js"></script>
// userName = "Vali" — file1 ning qiymati ustiga yozildi
```

**2. Namespace Collision** — katta jamoalarda har xil developerlar bir xil nom ishlatishi:

```javascript
// Ali yozdi
function validate(data) {
  return data.length > 0;
}

// Vali yozdi — shu faylda yoki boshqa faylda
function validate(data) {
  return data !== null && data !== undefined;
}

// Qaysi validate ishlaydi? — Oxirgi yuklangani! ❌
```

**3. Dependency Management** — qaysi fayl qaysi faylga bog'liq ekanini bilmaymiz:

```html
<!-- Tartib muhim! Xato tartib = xatolar -->
<script src="jquery.js"></script>         <!-- avval -->
<script src="jquery-plugin.js"></script>   <!-- keyin -->
<script src="app.js"></script>             <!-- eng oxirida -->

<!-- Agar tartib buzilsa: -->
<script src="app.js"></script>             <!-- ❌ jQuery hali yo'q! -->
<script src="jquery.js"></script>
```

**4. Maintainability** — 5000 qatorlik bitta faylda ishlash deyarli mumkin emas.

### Under the Hood

Nima uchun JavaScript dastlab modullar tizimisiz yaratildi?

1995-yilda Brendan Eich JavaScript ni **10 kunda** yaratdi. U faqat oddiy browser skripting uchun mo'ljallangan edi — form validation, kichik animatsiyalar. Hech kim JavaScript da katta applicationlar yozishni xayoliga keltirmagan edi.

```
1995: JavaScript yaratildi — faqat <script> tag
       ↓
1999-2005: Loyihalar kattaroq bo'la boshladi — muammolar paydo bo'ldi
       ↓
2009: Node.js yaratildi → CommonJS kerak bo'ldi
       ↓
2015: ES6 → ES Modules standarti — native module tizimi!
       ↓
2020+: ESM barcha zamonaviy muhitlarda qo'llab-quvvatlanadi
```

### Modul Tizimining Asosiy Prinsiplari

Har qanday modul tizimi 3 ta muammoni hal qiladi:

```
┌─────────────────────────────────────────────────────────┐
│                Module System Vazifalari                  │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  1. ENCAPSULATION                                       │
│     ┌──────────────────┐                                │
│     │  Private: a, b   │ ← Tashqaridan ko'rinmaydi     │
│     │  Public:  c()    │ ← Faqat export qilingan       │
│     └──────────────────┘                                │
│                                                         │
│  2. DEPENDENCY MANAGEMENT                               │
│     A ──depends──▶ B ──depends──▶ C                     │
│     (Aniq bo'ladi: kim kimga bog'liq)                   │
│                                                         │
│  3. NAMESPACE ISOLATION                                 │
│     Module A: { validate }  ← to'qnashmaydi            │
│     Module B: { validate }  ← o'z scope'ida            │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## Module Pattern (IIFE)

### Nazariya

ES Modules va CommonJS paydo bo'lmasdan oldin, JavaScript developerlar **IIFE** (Immediately Invoked Function Expression) yordamida modullar yaratishgan. Bu closure va function scope'ning kuchini ishlatgan aqlli pattern bo'lib, funksiya scope'da **private** o'zgaruvchilar yaratadi, faqat qaytarilgan narsalar **public** bo'ladi.

Revealing Module Pattern — bu yondashuvning yanada toza versiyasi bo'lib, barcha funksiyalar private yoziladi, keyin faqat kerakli nomlar ochiq ko'rsatiladi. Bu pattern hali ham foydali bilim — closure mexanizmini, data privacy asoslarini tushunish uchun muhim. Lekin uning kamchiliklari (dependency management yo'qligi, async loading yo'qligi, static analysis mumkin emasligi) CommonJS va ES Modules paydo bo'lishiga sabab bo'ldi.

```javascript
// Eng oddiy module pattern
var CounterModule = (function() {
  // ✅ PRIVATE — tashqaridan ko'rinmaydi
  var count = 0;

  // ✅ PUBLIC — qaytarilgan object orqali ko'rinadi
  return {
    increment: function() {
      count++;
      return count;
    },
    decrement: function() {
      count--;
      return count;
    },
    getCount: function() {
      return count;
    }
  };
})();

CounterModule.increment(); // 1
CounterModule.increment(); // 2
CounterModule.getCount();  // 2

// ❌ Private ga kirish mumkin emas
console.log(CounterModule.count); // undefined
```

### Revealing Module Pattern

Bu module pattern ning yanada toza versiyasi — barcha funksiyalar private, keyin nomlarini ochiq ko'rsatamiz:

```javascript
var UserService = (function() {
  // Barcha implementatsiya PRIVATE
  var users = [];

  function addUser(name, email) {
    var user = { id: users.length + 1, name: name, email: email };
    users.push(user);
    return user;
  }

  function removeUser(id) {
    users = users.filter(function(u) { return u.id !== id; });
  }

  function findUser(id) {
    return users.find(function(u) { return u.id === id; });
  }

  function getAllUsers() {
    // Copy qaytaramiz — ichki array ni himoya qilish uchun
    return users.slice();
  }

  // 🔍 Faqat shu yerda "reveal" qilamiz — qaysilari public
  return {
    add: addUser,
    remove: removeUser,
    find: findUser,
    getAll: getAllUsers
    // users massiv — PRIVATE qoladi
  };
})();

UserService.add("Ali", "ali@test.com");
UserService.add("Vali", "vali@test.com");
console.log(UserService.getAll()); // [{id: 1, ...}, {id: 2, ...}]
console.log(UserService.users);    // undefined — private!
```

### Under the Hood

IIFE module pattern qanday ishlaydi? Bu closure mexanizmi orqali:

```
Qadam 1: IIFE bajariladi
┌─────────────────────────────────────────────┐
│  IIFE Execution Context                     │
│  ┌───────────────────────────────────────┐  │
│  │  Variable Environment:                │  │
│  │    count = 0                          │  │
│  │    increment = function() {...}       │  │
│  │    decrement = function() {...}       │  │
│  │    getCount  = function() {...}       │  │
│  └───────────────────────────────────────┘  │
│                                             │
│  return { increment, decrement, getCount }  │
└─────────────────────────────────────────────┘

Qadam 2: IIFE tugadi, lekin closure saqlanadi
┌─────────────────────────────────────────────┐
│  Global Scope                               │
│  ┌───────────────────────────────────────┐  │
│  │  CounterModule = {                    │  │
│  │    increment: fn ──────┐              │  │
│  │    decrement: fn ──────┤  CLOSURE     │  │
│  │    getCount:  fn ──────┘  (count = 0  │  │
│  │  }                        ga havola)  │  │
│  └───────────────────────────────────────┘  │
│                                             │
│  count = 0 ← GC tozalamaydi,               │
│              closure reference bor           │
└─────────────────────────────────────────────┘
```

Har bir method `count` ga **closure** orqali kirishadi. `count` faqat shu funksiyalar orqali o'zgartirilishi mumkin — bu **data privacy** ning asosi.

### Augmentation Pattern

Katta loyihalarda modul kengaytirish kerak bo'lganda:

```javascript
// Modul yaratish
var MyApp = (function() {
  var appName = "SuperApp";
  return {
    getName: function() { return appName; }
  };
})();

// Modul kengaytirish — augmentation
MyApp = (function(module) {
  module.getVersion = function() {
    return "2.0.0";
  };
  return module;
})(MyApp);

MyApp.getName();    // "SuperApp"
MyApp.getVersion(); // "2.0.0"
```

### IIFE Modul Pattern ning Kamchiliklari

```javascript
// ❌ Muammolar:
// 1. Dependency management yo'q — tartibni o'zing kuzatishing kerak
// 2. Async loading yo'q — hamma narsa sync yuklaydi
// 3. Static analysis mumkin emas — bundler optimize qila olmaydi
// 4. Verbose syntax — ko'p boilerplate kod

// ✅ Shu sababli CommonJS va ES Modules paydo bo'ldi
```

---

## CommonJS — Node.js Standart

### Nazariya

CommonJS (CJS) — Node.js yaratilganda (2009) uning modul tizimi sifatida tanlangan standart. `require()` va `module.exports` — Node.js developerning kundalik qurollari. CJS **sinxron** yuklaydi — `require()` faylni o'qiydi, parse qiladi, bajaradi, va natijani qaytaradi. Bu server (Node.js) uchun yaxshi, chunki fayllar diskda tez o'qiladi, lekin browser'da muammo bo'ladi.

CJS ning muhim xususiyatlari: modullar **faqat birinchi marta** bajariladi, keyingi `require()` lar **cache**'dan oladi (singleton pattern uchun ideal); `module.exports` va `exports` farqini bilish muhim — `exports` faqat shortcut, uni qayta tayinlash xato; va `require()` ichdida faylni wrapper function'ga o'rab bajaradi, shu sababli har bir modulda `__filename`, `__dirname`, `require`, `module` mavjud bo'ladi.

```javascript
// math.js — module yaratish
function add(a, b) {
  return a + b;
}

function multiply(a, b) {
  return a * b;
}

// Eksport qilish
module.exports = {
  add: add,
  multiply: multiply
};

// yoki qisqacha (ES6 shorthand bilan)
module.exports = { add, multiply };
```

```javascript
// app.js — module ishlatish
const math = require('./math');

console.log(math.add(2, 3));      // 5
console.log(math.multiply(4, 5)); // 20

// Destructuring bilan
const { add, multiply } = require('./math');
console.log(add(2, 3)); // 5
```

### Synchronous Loading

CommonJS **sinxron** yuklaydi — `require()` faylni o'qiydi, bajaradi, va natijani qaytaradi. Keyingi qator faqat require tugagandan keyin ishlaydi.

```javascript
console.log("1 — require dan oldin");

const fs = require('fs');        // ⏳ fayl o'qiladi, parse qilinadi, bajariladi
console.log("2 — require dan keyin"); // faqat require tugagandan keyin ishlaydi

const path = require('path');    // ⏳ yana sync
console.log("3 — ikkinchi require dan keyin");

// Output: 1, 2, 3 — doim shu tartibda
```

Bu server (Node.js) uchun OK, chunki fayllar diskda — tez o'qiladi. Lekin **browser** da bu muammo — har bir fayl uchun HTTP so'rov kerak, foydalanuvchi kutishi kerak. Shu sababli browser'larda ESM va bundler'lar ishlatiladi.

### Cache Mexanizmi

Modul **faqat birinchi marta** to'liq bajariladi. Keyingi `require()` lar **cache**'dan oladi:

```javascript
// counter.js
console.log("counter.js YUKLANDI!"); // faqat 1 marta chiqadi

let count = 0;

module.exports = {
  increment() { return ++count; },
  getCount() { return count; }
};
```

```javascript
// app.js
const counter1 = require('./counter'); // "counter.js YUKLANDI!" — birinchi marta
const counter2 = require('./counter'); // 🔇 hech narsa — cache'dan

counter1.increment(); // 1
counter2.getCount();  // 1 — bir xil instance!

console.log(counter1 === counter2); // true — aynan bir xil object
```

Cache `require.cache` objectda saqlanadi:

```javascript
// Cache ni ko'rish
console.log(require.cache);
// {
//   '/path/to/counter.js': Module { exports: {...}, ... },
//   '/path/to/app.js': Module { ... }
// }

// ⚠️ Cache ni tozalash (odatda testing da kerak)
delete require.cache[require.resolve('./counter')];
const freshCounter = require('./counter'); // endi qayta bajaradi
```

### `module.exports` vs `exports` Farqi

Bu ko'p chalkashlik keltiradigan nuqta. Aslida `exports` — bu `module.exports` ga **shortcut** (reference):

```javascript
// Node.js ichida har bir module boshlanishida:
// var exports = module.exports = {};
// Ya'ni ikkalasi BIR XIS object ga ishora qiladi

// ✅ To'g'ri — exports ga property qo'shish
exports.add = function(a, b) { return a + b; };
exports.sub = function(a, b) { return a - b; };
// module.exports = { add: fn, sub: fn } — ishlaydi

// ✅ To'g'ri — module.exports ni to'liq almashtirish
module.exports = {
  add: function(a, b) { return a + b; },
  sub: function(a, b) { return a - b; }
};

// ✅ To'g'ri — class yoki function eksport
module.exports = class Calculator {
  add(a, b) { return a + b; }
};
```

```javascript
// ❌ XATO — exports ni qayta tayinlash
exports = {
  add: function(a, b) { return a + b; }
};
// Bu exports o'zgaruvchisini yangi object ga bog'laydi,
// lekin module.exports hali eski bo'sh object!
// require() DOIM module.exports ni qaytaradi

// 🔍 Nima uchun?
// Dastlab: exports ──▶ {} ◀── module.exports
// Keyin:   exports ──▶ { add: fn }   (yangi object)
//          module.exports ──▶ {}      (eski bo'sh object — shu qaytariladi!)
```

```
exports va module.exports aloqasi:

BOSHLANG'ICH HOLAT:
  exports ────────┐
                  ▼
                { }  ◀──── module.exports
                  ▲
                  │
          require() DOIM SHUNI QAYTARADI


exports = { add: fn } dan KEYIN:
  exports ──────▶ { add: fn }    ← YANGI object (require() buni QAYTARMAYDI)

  module.exports ──▶ { }         ← ESKI object (require() BUNI qaytaradi!)


XULOSA: Hech qachon exports = ... qilmang.
         Doim module.exports = ... yoki exports.prop = ... ishlating.
```

### Under the Hood — `require()` Ichidan

`require()` chaqirilganda Node.js ichida nima sodir bo'ladi:

```
require('./math') chaqirilganda:

┌─────────────────────────────────────────────────┐
│  1. RESOLVE — fayl yo'lini aniqlash             │
│     './math' → '/Users/app/math.js'             │
│     Tartib: .js → .json → .node → directory     │
├─────────────────────────────────────────────────┤
│  2. CACHE CHECK — avval cache tekshir           │
│     require.cache['/Users/app/math.js']         │
│     Bor? → Darhol exports ni qaytar ──▶ TUGADI  │
│     Yo'q? → keyingi qadam                       │
├─────────────────────────────────────────────────┤
│  3. MODULE YARATISH                             │
│     module = {                                  │
│       id: '/Users/app/math.js',                 │
│       exports: {},                              │
│       loaded: false,                            │
│       children: [],                             │
│       paths: [...]                              │
│     }                                           │
├─────────────────────────────────────────────────┤
│  4. CACHE GA QO'SHISH                           │
│     require.cache[id] = module                  │
│     (⚠️ execute dan OLDIN qo'shiladi —          │
│      circular dependency uchun muhim!)           │
├─────────────────────────────────────────────────┤
│  5. WRAP va EXECUTE                             │
│     Fayl kodi wrapper function ga o'raladi:     │
│     (function(exports, require, module,         │
│              __filename, __dirname) {            │
│       // === SIZNING KODINGIZ ===               │
│       module.exports = { add, multiply };       │
│     });                                         │
│     va bajariladi                               │
├─────────────────────────────────────────────────┤
│  6. RETURN                                      │
│     module.loaded = true                        │
│     return module.exports                       │
└─────────────────────────────────────────────────┘
```

**Qadam 5** ga e'tibor bering — har bir modul aslida wrapper function ichida bajariladi. Shuning uchun modulda `__filename`, `__dirname`, `exports`, `require`, `module` mavjud bo'ladi. Ular magik emas — funksiya parametrlari!

```javascript
// Biz yozamiz:
const x = 10;
module.exports = { x };

// Node.js aslida bajaradi:
(function(exports, require, module, __filename, __dirname) {
  const x = 10;
  module.exports = { x };
})(module.exports, require, module, '/path/to/file.js', '/path/to');
```

### Resolve Algoritmi

`require()` fayl topish uchun murakkab algoritm ishlatadi:

```javascript
// 1. Core modules — eng yuqori priority
require('fs');     // Node.js built-in
require('path');   // Node.js built-in

// 2. Fayl yo'li bilan (. yoki / bilan boshlanadi)
require('./math');          // → ./math.js → ./math.json → ./math/index.js
require('../utils/helper'); // nisbiy yo'l
require('/abs/path/file');  // absolyut yo'l

// 3. node_modules (. yoki / BOSHLANMAYDI)
require('lodash');
// Qidirish tartibi:
//   ./node_modules/lodash
//   ../node_modules/lodash
//   ../../node_modules/lodash
//   ... root gacha
```

```
require('lodash') resolve tartibi:

/Users/app/src/controllers/userController.js dan chaqirilsa:

  1. /Users/app/src/controllers/node_modules/lodash
  2. /Users/app/src/node_modules/lodash
  3. /Users/app/node_modules/lodash        ← odatda shu yerda topiladi
  4. /Users/node_modules/lodash
  5. /node_modules/lodash
  ❌ Topilmadi → MODULE_NOT_FOUND error
```

### Kod Misollari — Real-World CJS

```javascript
// config.js — konfiguratsiya moduli
const path = require('path');

const config = {
  port: process.env.PORT || 3000,
  db: {
    host: process.env.DB_HOST || 'localhost',
    port: process.env.DB_PORT || 5432,
    name: process.env.DB_NAME || 'myapp_dev'
  },
  paths: {
    uploads: path.join(__dirname, 'uploads'),
    logs: path.join(__dirname, 'logs')
  }
};

module.exports = config;
```

```javascript
// logger.js — singleton pattern (cache tufayli)
const fs = require('fs');
const path = require('path');

class Logger {
  constructor() {
    this.logFile = path.join(__dirname, 'app.log');
  }

  log(level, message) {
    const timestamp = new Date().toISOString();
    const entry = `[${timestamp}] [${level.toUpperCase()}] ${message}\n`;
    fs.appendFileSync(this.logFile, entry);
    console.log(entry.trim());
  }

  info(msg) { this.log('info', msg); }
  warn(msg) { this.log('warn', msg); }
  error(msg) { this.log('error', msg); }
}

// Singleton — require cache tufayli doim BIR XIS instance
module.exports = new Logger();
```

```javascript
// userService.js — dependency injection pattern
const config = require('./config');
const logger = require('./logger');

class UserService {
  constructor() {
    this.apiUrl = `http://${config.db.host}:${config.db.port}`;
    logger.info(`UserService initialized: ${this.apiUrl}`);
  }

  async getUser(id) {
    logger.info(`Fetching user ${id}`);
    // ...
  }
}

module.exports = UserService;
```

---

## ES Modules — Zamonaviy Standart

### Nazariya

ES Modules (ESM) — ECMAScript 2015 (ES6) da standartlashtirilgan rasmiy JavaScript modul tizimi. Bu **til darajasida** qo'llab-quvvatlanadigan yagona modul standart — CommonJS Node.js ning o'zi edi, ESM esa JavaScript **tilining** qismi. ESM `import`/`export` statement'lar bilan ishlaydi va **statik struktura**ga ega — import'lar fayl boshida, compile-time da aniqlanadi.

ESM ning asosiy afzalliklari: **static analysis** mumkin (tree shaking, IDE intellisense), **named va default export**'lar, **re-exporting** (barrel files), **asinxron yuklash** (browser uchun ideal), va **live bindings** (export qilingan qiymat o'zgarganda import ham yangilanadi). Browser'larda `<script type="module">`, Node.js da `.mjs` yoki `"type": "module"` package.json orqali ishlatiladi.

```javascript
// math.js — Named exports
export function add(a, b) {
  return a + b;
}

export function multiply(a, b) {
  return a * b;
}

export const PI = 3.14159;
```

```javascript
// app.js — Named imports
import { add, multiply, PI } from './math.js';

console.log(add(2, 3));      // 5
console.log(multiply(4, 5)); // 20
console.log(PI);              // 3.14159
```

### Named Exports vs Default Export

**Named Export** — bir moduldan bir nechta narsani eksport qilish:

```javascript
// utils.js — Named exports
export function formatDate(date) {
  return date.toLocaleDateString('uz-UZ');
}

export function formatCurrency(amount) {
  return `${amount.toLocaleString()} so'm`;
}

export const SUPPORTED_LOCALES = ['uz', 'ru', 'en'];

// Yoki pastda bir joyda:
function capitalize(str) {
  return str.charAt(0).toUpperCase() + str.slice(1);
}
function lowercase(str) {
  return str.toLowerCase();
}
export { capitalize, lowercase };
```

```javascript
// Ishlatish — faqat kerakli narsalarni import qilish
import { formatDate, formatCurrency } from './utils.js';
import { capitalize } from './utils.js';

// ✅ Aniq — nima import qilinganini ko'ramiz
// ✅ Tree shaking mumkin — ishlatilmaganlar olib tashlanadi
```

**Default Export** — modulning "asosiy" eksporti:

```javascript
// UserService.js — Default export
export default class UserService {
  constructor(apiUrl) {
    this.apiUrl = apiUrl;
  }

  async getUser(id) {
    const response = await fetch(`${this.apiUrl}/users/${id}`);
    return response.json();
  }

  async getAll() {
    const response = await fetch(`${this.apiUrl}/users`);
    return response.json();
  }
}
```

```javascript
// Import — ixtiyoriy nom bilan
import UserService from './UserService.js';
import MyService from './UserService.js';  // boshqa nom ham ishlaydi!

const service = new UserService('https://api.example.com');
```

**Named + Default birgalikda:**

```javascript
// api.js
export default class API {
  // asosiy class
}

export function createClient(url) {
  return new API(url);
}

export const VERSION = '2.0.0';
```

```javascript
// Import — aralash
import API, { createClient, VERSION } from './api.js';
```

### Namespace Import — `import * as`

```javascript
// Hamma narsani bitta nom ostida import qilish
import * as MathUtils from './math.js';

console.log(MathUtils.add(2, 3));    // 5
console.log(MathUtils.PI);            // 3.14159
console.log(MathUtils.default);       // agar default export bo'lsa

// 🔍 Qachon foydali?
// 1. Modulda juda ko'p export bor — hammasi kerak
// 2. Nomlarda to'qnashuv bor — namespace ajratish
// 3. Qaysi moduldan kelganini aniq ko'rsatish
```

### Re-exporting — `export ... from`

Barrel files va modul fasadlari yaratish uchun:

```javascript
// models/User.js
export class User { /* ... */ }

// models/Product.js
export class Product { /* ... */ }

// models/Order.js
export class Order { /* ... */ }
export function createOrder(items) { /* ... */ }
```

```javascript
// models/index.js — Barrel file (re-export hub)
export { User } from './User.js';
export { Product } from './Product.js';
export { Order, createOrder } from './Order.js';

// Default export ni re-export qilish
export { default as User } from './User.js';

// Hamma narsani re-export
export * from './User.js';
export * from './Product.js';
// ⚠️ default export `export *` bilan re-export qilinMAYDI
```

```javascript
// app.js — endi bitta joydan import
import { User, Product, Order, createOrder } from './models/index.js';
// yoki
import { User, Product, Order } from './models/index.js';

// ✅ Toza import — 3 ta fayl o'rniga bitta joydan
// ✅ Refactoring oson — ichki struktura o'zgarsa, faqat barrel file o'zgaradi
```

### Rename (Alias) bilan Import/Export

```javascript
// utils.js
function internalValidate(data) { /* ... */ }

// Eksport paytida nom o'zgartirish
export { internalValidate as validate };
```

```javascript
// app.js — Import paytida nom o'zgartirish
import { validate as validateUser } from './userValidator.js';
import { validate as validateProduct } from './productValidator.js';

// Endi ikkalasi bor — to'qnashmaydi
validateUser(userData);
validateProduct(productData);
```

### Under the Hood — ESM Loading Pipeline

ESM yuklanishi **3 bosqichli** asynchronous jarayondir. Bu CJS dan tubdan farq qiladi:

```
ES Module Loading Pipeline:

┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  Source Code: import { add } from './math.js'                │
│                                                              │
│  ═══════════════════════════════════════════════════════════  │
│                                                              │
│  BOSQICH 1: CONSTRUCTION (Parse)                             │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  1. URL resolve: './math.js' → to'liq yo'l            │  │
│  │  2. Fetch: faylni yuklash (URL/disk dan)              │  │
│  │  3. Parse: source code → Module Record                │  │
│  │     - import/export larni aniqlash (static analysis)  │  │
│  │     - KOD BAJARILMAYDI — faqat PARSE                  │  │
│  │                                                        │  │
│  │  Module Record:                                        │  │
│  │  {                                                     │  │
│  │    RequestedModules: ['./helper.js'],                  │  │
│  │    ImportEntries: [{ name: 'add', from: './math.js' }],│  │
│  │    ExportEntries: [{ name: 'add' }, { name: 'PI' }],  │  │
│  │    Status: 'unlinked'                                  │  │
│  │  }                                                     │  │
│  └────────────────────────────────────────────────────────┘  │
│                          ▼                                   │
│  BOSQICH 2: INSTANTIATION (Link)                             │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  1. Module Environment Record yaratish                 │  │
│  │  2. Export lar uchun memory joylari ajratish           │  │
│  │  3. Import larni eksport memory joylariga bog'lash     │  │
│  │     (LIVE BINDINGS — reference, copy emas!)            │  │
│  │  4. KOD HALI BAJARILMAYDI                              │  │
│  │                                                        │  │
│  │  math.js:  [ add: <uninitialized> ] [ PI: <uninit> ]  │  │
│  │                 ▲                        ▲              │  │
│  │                 │ (live binding)         │              │  │
│  │  app.js:   [ add: ─────────────── ] [ PI: ──────── ]  │  │
│  └────────────────────────────────────────────────────────┘  │
│                          ▼                                   │
│  BOSQICH 3: EVALUATION (Execute)                             │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  1. Dependency graph bo'yicha depth-first tartibda     │  │
│  │  2. Har bir modul FAQAT 1 MARTA bajariladi            │  │
│  │  3. Eksportlar qiymat oladi                            │  │
│  │                                                        │  │
│  │  math.js bajariladi:                                   │  │
│  │    [ add: function(){...} ] [ PI: 3.14159 ]           │  │
│  │         ▲                        ▲                     │  │
│  │         │                        │                     │  │
│  │  app.js import lari avtomatik yangilanadi              │  │
│  │  (chunki bir xil memory joyiga ishora qiladi)         │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Live Bindings — CJS dan Asosiy Farq

Bu ESM ning eng muhim xususiyatlaridan biri. ESM import — bu **live binding** (tirik bog'lanish), CJS esa **copy** (nusxa):

```javascript
// ===== CommonJS — COPY =====
// counter.cjs
let count = 0;
function increment() { count++; }
function getCount() { return count; }
module.exports = { count, increment, getCount };
```

```javascript
// app.cjs
const counter = require('./counter.cjs');

console.log(counter.count);  // 0
counter.increment();
console.log(counter.count);  // 0 ❌ — O'ZGARMADI! count COPY bo'lgan!
console.log(counter.getCount()); // 1 — funksiya orqali haqiqiy qiymat
```

```javascript
// ===== ES Modules — LIVE BINDING =====
// counter.mjs
export let count = 0;
export function increment() { count++; }
```

```javascript
// app.mjs
import { count, increment } from './counter.mjs';

console.log(count);  // 0
increment();
console.log(count);  // 1 ✅ — YANGILANDI! Live binding!

// ⚠️ Lekin import qilingan binding READ-ONLY
// count = 5; // ❌ TypeError: Assignment to constant variable
// Faqat eksport qilgan modul o'zgartira oladi
```

```
LIVE BINDINGS mexanizmi:

CommonJS (Copy):
  counter.js:  count = 0
                   │ (copy qiymati)
  app.js:      counter.count = 0  ← alohida nusxa

  counter.js:  count = 1  (increment() dan keyin)
  app.js:      counter.count = 0  ← hali eski qiymat!


ES Modules (Live Binding):
  counter.mjs: ┌───────────┐
               │ count = 0  │ ← BIR XIS memory joyi
               └─────▲──────┘
                     │ (reference)
  app.mjs:     count ─┘ ← shu joyga ishora qiladi

  increment() dan keyin:
  counter.mjs: ┌───────────┐
               │ count = 1  │ ← qiymat o'zgardi
               └─────▲──────┘
                     │
  app.mjs:     count ─┘ ← avtomatik yangi qiymatni ko'radi!
```

### ESM — Strict Mode

ES Modules **doimo** strict mode'da ishlaydi. `"use strict"` yozish shart emas:

```javascript
// module.mjs — avtomatik strict mode
x = 10;           // ❌ ReferenceError (undeclared variable)
delete Object.prototype; // ❌ TypeError
var arguments = 5; // ❌ SyntaxError

// this — undefined (global object emas)
console.log(this); // undefined (CJS da this = module.exports)
```

### Node.js da ESM Ishlatish

```javascript
// 1-usul: .mjs extension
// math.mjs
export const add = (a, b) => a + b;

// app.mjs
import { add } from './math.mjs';
```

```json
// 2-usul: package.json da "type": "module"
{
  "name": "my-app",
  "version": "1.0.0",
  "type": "module"
}
// Endi barcha .js fayllar ESM sifatida ishlaydi
// CJS kerak bo'lsa — .cjs extension ishlatiladi
```

```json
// 3-usul: Dual Package — CJS ham ESM ham qo'llab-quvvatlash
// package.json
{
  "name": "my-library",
  "version": "1.0.0",
  "main": "./dist/index.cjs",
  "module": "./dist/index.mjs",
  "exports": {
    ".": {
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs"
    },
    "./utils": {
      "import": "./dist/utils.mjs",
      "require": "./dist/utils.cjs"
    }
  }
}
```

### Static Analysis — Compile-Time Imkoniyat

ESM import/export lar **statik** — ular compile-time da ma'lum (kod bajarilmasdan oldin). Bu bundler'larga katta imkoniyat beradi:

```javascript
// ✅ ESM — STATIK, compile-time da aniq
import { add } from './math.js';      // doim fayl boshida
import { format } from './utils.js';   // doim string literal

// ❌ Bu mumkin EMAS (dynamic bo'lgani uchun):
if (condition) {
  import { add } from './math.js'; // ❌ SyntaxError
}

const moduleName = './math.js';
import { add } from moduleName;   // ❌ SyntaxError — o'zgaruvchi ishlatib bo'lmaydi

// ✅ CJS — DINAMIK, runtime da aniqlanadi
const moduleName = './math';
const math = require(moduleName); // ✅ Ishlaydi!

if (condition) {
  const extra = require('./extra'); // ✅ Ishlaydi!
}
```

Statik tahlil imkoniyati sababli:
1. **Tree shaking** — ishlatilmagan kod olib tashlanadi
2. **Faster module resolution** — import grafni oldindan tuzish mumkin
3. **IDE support** — autocomplete va type checking yaxshiroq ishlaydi
4. **Circular dependency detection** — oldinroq aniqlash mumkin

### ECMAScript Spec: Module Record

Spec bo'yicha har bir ESM uchun **Source Text Module Record** yaratiladi:

```
Source Text Module Record (ECMAScript Spec):

┌─────────────────────────────────────────────────┐
│  [[Realm]]:              Realm Record            │
│  [[Environment]]:        Module Environment      │
│  [[Namespace]]:          Module Namespace Object  │
│  [[Status]]:             "unlinked" | "linking"  │
│                          | "linked" | "evaluated"│
│  [[EvaluationError]]:    undefined | Error       │
│  [[ECMAScriptCode]]:     Parse Node (AST)        │
│  [[RequestedModules]]:   ["./math.js", "./a.js"] │
│  [[ImportEntries]]:      [ImportEntry Records]   │
│  [[LocalExportEntries]]: [ExportEntry Records]   │
│  [[StarExportEntries]]:  [ExportEntry Records]   │
└─────────────────────────────────────────────────┘

Module Environment Record:
  - Lexical Environment ning maxsus turi
  - Har bir import uchun "indirect binding" yaratadi
  - CreateImportBinding(name, sourceModule, sourceBinding)
  - Bu binding read-only va LIVE — source module dagi
    qiymat o'zgarganda bu yerda ham o'zgaradi
```

---

## CommonJS vs ES Modules — To'liq Taqqoslash

### Farqlar Jadvali

| Xususiyat | CommonJS (CJS) | ES Modules (ESM) |
|-----------|-----------------|-------------------|
| **Syntax** | `require()` / `module.exports` | `import` / `export` |
| **Loading** | **Synchronous** | **Asynchronous** |
| **Parsing** | Runtime (dynamic) | Compile-time (static) |
| **Binding** | **Copy** (qiymat nusxasi) | **Live binding** (reference) |
| **this** | `module.exports` (top-level) | `undefined` |
| **Strict mode** | Non-strict (default) | **Doimo strict** |
| **Hoisting** | Yo'q — qayerda yozilsa shu yerda bajariladi | Import lar **hoist** bo'ladi |
| **Conditional import** | ✅ `if` ichida `require()` mumkin | ❌ Static — `import()` ishlatish kerak |
| **File extension** | `.js` (default) yoki `.cjs` | `.mjs` yoki `.js` (`"type":"module"`) |
| **Top-level await** | ❌ Mumkin emas | ✅ Mumkin (ES2022) |
| **`__filename`** | ✅ Mavjud | ❌ Yo'q (`import.meta.url` ishlatiladi) |
| **`__dirname`** | ✅ Mavjud | ❌ Yo'q |
| **JSON import** | ✅ `require('./data.json')` | ⚠️ Import assertion kerak |
| **Dynamic import** | ✅ `require()` | ✅ `import()` (Promise qaytaradi) |
| **Tree shaking** | ❌ Qiyin/mumkin emas | ✅ Oson (static analysis) |
| **Cache** | `require.cache` object | Module Map (spec level) |
| **Circular deps** | Partial exports (to'liq emas) | Live bindings (yaxshiroq) |
| **Browser support** | ❌ (bundler kerak) | ✅ Native (`<script type="module">`) |

### Loading Farqi Vizualizatsiya

```
CommonJS Loading (Synchronous, Runtime):

  require('A') ──▶ LOAD A ──▶ EXECUTE A ──▶ return exports
                                    │
                              require('B') ──▶ LOAD B ──▶ EXECUTE B ──▶ return
                                                              │
                                                        require('C') ──▶ ...
  Har bir require() BLOKLAYDI — keyingi qator kutadi


ES Modules Loading (Asynchronous, Multi-phase):

  BOSQICH 1 — PARSE (parallel mumkin):
  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │ Parse A  │  │ Parse B  │  │ Parse C  │
  └────┬─────┘  └────┬─────┘  └────┬─────┘
       │             │             │
  BOSQICH 2 — LINK (barcha modullar parse bo'lgandan keyin):
  ┌──────────────────────────────────────────┐
  │  A.add ◀── live binding ──▶ B.add        │
  │  Memory joylar ajratiladi va bog'lanadi  │
  └──────────────────────────────────────────┘
       │
  BOSQICH 3 — EVALUATE (depth-first, bir marta):
  C evaluate → B evaluate → A evaluate
```

### Kod Misoli — Bir Xil Module Ikki Tizimda

```javascript
// ===== CommonJS =====
// logger.cjs
const levels = { INFO: 'INFO', WARN: 'WARN', ERROR: 'ERROR' };
let logCount = 0;

function log(level, message) {
  logCount++;
  console.log(`[${level}] #${logCount}: ${message}`);
}

module.exports = { levels, log, getLogCount: () => logCount };
```

```javascript
// ===== ES Modules =====
// logger.mjs
export const levels = { INFO: 'INFO', WARN: 'WARN', ERROR: 'ERROR' };
let logCount = 0;

export function log(level, message) {
  logCount++;
  console.log(`[${level}] #${logCount}: ${message}`);
}

export function getLogCount() {
  return logCount;
}

// yoki default export bilan
// export default { levels, log, getLogCount };
```

### ESM da CJS ni Import Qilish

```javascript
// ESM faylda CJS modul import qilish (Node.js)
// CJS ning module.exports → ESM default import sifatida
import pkg from './legacy-module.cjs';
const { helper, config } = pkg;

// Yoki createRequire bilan
import { createRequire } from 'node:module';
const require = createRequire(import.meta.url);
const legacyLib = require('./legacy-lib');

// __dirname ekvivalenti ESM da
import { fileURLToPath } from 'node:url';
import { dirname } from 'node:path';
const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);
```

---

## Dynamic Imports — `import()`

### Nazariya

Static `import` doim fayl boshida bo'lishi kerak va compile-time da aniqlanadi. Lekin ba'zan modulni **shart bo'yicha**, **keyin**, yoki **foydalanuvchi harakati asosida** yuklash kerak bo'ladi. Buning uchun **dynamic `import()`** mavjud — bu funksiya emas, bu **operator** bo'lib, **Promise** qaytaradi.

Dynamic import ayniqsa **code splitting** (sahifa bo'yicha kod ajratish), **lazy loading** (kerak bo'lganda yuklash), **conditional loading** (platforma yoki tilga qarab modul tanlash), va **plugin architecture** (runtime da modul qo'shish) uchun ishlatiladi. Bundler'lar (Webpack, Vite) dynamic import'ni avtomatik code splitting nuqtasi sifatida tan oladi.

```javascript
// Dynamic import — basic
const module = await import('./math.js');
console.log(module.add(2, 3)); // 5

// yoki .then bilan
import('./math.js').then(module => {
  console.log(module.add(2, 3)); // 5
  console.log(module.default);    // agar default export bo'lsa
});
```

### Conditional Imports

```javascript
// ✅ Shart bo'yicha yuklash — static import da mumkin emas!
async function loadTranslations(lang) {
  let translations;

  if (lang === 'uz') {
    translations = await import('./locales/uz.js');
  } else if (lang === 'ru') {
    translations = await import('./locales/ru.js');
  } else {
    translations = await import('./locales/en.js');
  }

  return translations.default;
}

// ✅ Computed path bilan
async function loadModule(moduleName) {
  const module = await import(`./modules/${moduleName}.js`);
  return module;
}
```

### Code Splitting va Lazy Loading

Dynamic import — code splitting ning asosi. Foydalanuvchi faqat kerakli kodni yuklaydi:

```javascript
// ❌ Hammasi birdaniga yuklanadi — sekin!
import { AdminDashboard } from './components/AdminDashboard.js';
import { UserProfile } from './components/UserProfile.js';
import { Analytics } from './components/Analytics.js';

// ✅ Kerak bo'lganda yuklanadi — tez!
async function navigateTo(route) {
  let component;

  switch (route) {
    case '/admin':
      const { AdminDashboard } = await import('./components/AdminDashboard.js');
      component = AdminDashboard;
      break;
    case '/profile':
      const { UserProfile } = await import('./components/UserProfile.js');
      component = UserProfile;
      break;
    case '/analytics':
      const { Analytics } = await import('./components/Analytics.js');
      component = Analytics;
      break;
  }

  renderComponent(component);
}
```

### Route-Based Splitting — React va Vue da

```javascript
// ===== React — React.lazy + Suspense =====
import React, { Suspense, lazy } from 'react';

// lazy() ichida dynamic import
const AdminPage = lazy(() => import('./pages/AdminPage'));
const UserPage = lazy(() => import('./pages/UserPage'));
const SettingsPage = lazy(() => import('./pages/SettingsPage'));

function App() {
  return (
    <Suspense fallback={<div>Yuklanmoqda...</div>}>
      <Routes>
        <Route path="/admin" element={<AdminPage />} />
        <Route path="/user" element={<UserPage />} />
        <Route path="/settings" element={<SettingsPage />} />
      </Routes>
    </Suspense>
  );
}

// Har bir sahifa ALOHIDA chunk sifatida yuklanadi:
// admin-page.chunk.js — faqat /admin ga borganda
// user-page.chunk.js — faqat /user ga borganda
```

```javascript
// ===== Vue — defineAsyncComponent =====
import { defineAsyncComponent } from 'vue';

const AdminPage = defineAsyncComponent(
  () => import('./pages/AdminPage.vue')
);

// Vue Router bilan
const routes = [
  {
    path: '/admin',
    component: () => import('./pages/AdminPage.vue')
  },
  {
    path: '/user',
    component: () => import('./pages/UserPage.vue')
  }
];
```

### Feature Detection va Polyfill Loading

```javascript
// ✅ Faqat kerakli polyfill yuklanadi
async function loadPolyfills() {
  const polyfills = [];

  if (!window.IntersectionObserver) {
    polyfills.push(import('intersection-observer'));
  }

  if (!window.fetch) {
    polyfills.push(import('whatwg-fetch'));
  }

  if (!Array.prototype.at) {
    polyfills.push(import('core-js/actual/array/at'));
  }

  await Promise.all(polyfills);
  console.log('Polyfills yuklandi');
}
```

### Under the Hood

```
import('./math.js') chaqirilganda:

┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  1. import('./math.js') ──▶ Promise yaratiladi               │
│                                                              │
│  2. Module Specifier resolve qilinadi                        │
│     './math.js' → 'https://example.com/math.js'             │
│                                                              │
│  3. Module Map tekshiriladi (cache)                          │
│     ┌──────────────────────────┐                             │
│     │ Module Map:              │                             │
│     │ './math.js' → Module?    │──▶ Bor? → Qaytarish         │
│     └──────────────────────────┘──▶ Yo'q? → Yuklash          │
│                                                              │
│  4. Fetch → Parse → Instantiate → Evaluate                   │
│     (Xuddi static import kabi, lekin async)                  │
│                                                              │
│  5. Promise resolve bo'ladi Module Namespace Object bilan    │
│     { add: fn, multiply: fn, default: ... }                  │
│                                                              │
│  ⚠️ import() har doim MODULE NAMESPACE OBJECT qaytaradi     │
│     (default export ham .default propertyda)                 │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

```javascript
// import() natijasi — Module Namespace Object
const mathModule = await import('./math.js');

// mathModule = {
//   add: [Function: add],
//   multiply: [Function: multiply],
//   PI: 3.14159,
//   default: ...,                // agar default export bo'lsa
//   [Symbol.toStringTag]: 'Module'
// }

// Default export ni olish
const { default: MyClass } = await import('./MyClass.js');
```

---

## Circular Dependencies

### Nazariya

Circular dependency — ikki yoki undan ortiq modul **bir-biriga bog'liq** bo'lganda paydo bo'ladi (A imports B, B imports A). Bu **xato emas** — JavaScript buni handle qiladi, lekin natija kutilganidek bo'lmasligi mumkin va debug qilish juda qiyin.

CommonJS va ESM circular dependency'ni turlicha handle qiladi. CJS da modul bajarilish jarayonida cache ga qo'yiladi, shuning uchun circular `require()` **to'liq emas** (partially loaded) exports oladi. ESM da esa live bindings tufayli vaziyat biroz yaxshiroq — lekin `undefined` qiymatlar olish xavfi bor. Eng yaxshi yechim — circular dependency'dan **umuman qochish**, arxitekturani qayta ko'rib chiqish, yoki bog'liqlikni uchinchi modulga chiqarish.

### CommonJS da Circular Dependency

CJS da circular dependency bo'lganda, Node.js **partial exports** qaytaradi — modul to'liq bajarilmasdan, shu paytgacha eksport qilingan narsalarni beradi:

```javascript
// a.cjs
console.log('a.cjs boshlanmoqda');
const b = require('./b.cjs');  // b.cjs ga boradi
console.log('a.cjs da b =', b);
module.exports = { fromA: 'salom A dan' };
console.log('a.cjs tugadi');
```

```javascript
// b.cjs
console.log('b.cjs boshlanmoqda');
const a = require('./a.cjs');  // a.cjs ga qaytadi — lekin a hali tugamagan!
console.log('b.cjs da a =', a);
module.exports = { fromB: 'salom B dan' };
console.log('b.cjs tugadi');
```

```javascript
// main.cjs
require('./a.cjs');

// Output:
// a.cjs boshlanmoqda
// b.cjs boshlanmoqda
// b.cjs da a = {}              ← ⚠️ BO'SH OBJECT! a hali tugamagan edi!
// b.cjs tugadi
// a.cjs da b = { fromB: 'salom B dan' }  ← b to'liq
// a.cjs tugadi
```

```
CJS Circular Dependency Mexanizmi:

1. main: require('a')
   ┌──────────────────────────────────┐
   │ a.cjs ishga tushadi             │
   │ module.exports = {} (hali bo'sh) │
   │ require('b') chaqiriladi         │
   │ ┌────────────────────────────┐   │
   │ │ b.cjs ishga tushadi       │   │
   │ │ require('a') chaqiriladi   │   │
   │ │ ⚠️ a.cjs CACHE DA BOR!   │   │
   │ │ a.exports = {} (BO'SH!)   │   │
   │ │ b.exports = { fromB }     │   │
   │ └────────────────────────────┘   │
   │ b tayyor: { fromB }              │
   │ a.exports = { fromA }            │
   └──────────────────────────────────┘
```

### ES Modules da Circular Dependency

ESM **live bindings** tufayli bunday holatni yaxshiroq handle qiladi — lekin ehtiyot bo'lish kerak:

```javascript
// a.mjs
import { fromB } from './b.mjs';

console.log('a.mjs da fromB =', fromB);  // "salom B dan"
export const fromA = 'salom A dan';
```

```javascript
// b.mjs
import { fromA } from './a.mjs';

console.log('b.mjs da fromA =', fromA);
// ⚠️ ReferenceError: Cannot access 'fromA' before initialization
// Chunki a.mjs hali evaluate bo'lmagan — fromA TDZ da!
export const fromB = 'salom B dan';
```

```javascript
// ✅ Yechim — function ishlatish (hoisting tufayli ishlaydi)
// a.mjs
import { getFromB } from './b.mjs';

export function getFromA() {
  return 'salom A dan';
}

// Keyin chaqirganda ishlaydi:
console.log(getFromB()); // "salom B dan"
```

```javascript
// b.mjs
import { getFromA } from './a.mjs';

export function getFromB() {
  return 'salom B dan';
}

// Keyin chaqirganda ishlaydi:
console.log(getFromA()); // "salom A dan"
```

**Nima uchun function ishlaydi?** Chunki function declarations **hoist** bo'ladi — link bosqichida funksiya allaqachon mavjud. `const`/`let` esa TDZ da bo'ladi — evaluate bo'lmaguncha qiymat yo'q.

### Circular Dependency ni Aniqlash va Oldini Olish

```
Circular Dependency ni aniqlash:
  ✅ ESLint plugin: eslint-plugin-import → import/no-cycle
  ✅ Bundler warnings (Webpack, Vite)
  ✅ madge — dependency visualization tool

Oldini olish strategiyalari:

1. Dependency Inversion
   ❌ A ←→ B (ikki tomonlama)
   ✅ A → C ← B (umumiy modul orqali)

2. Event-Based Communication
   ❌ userService.js ←→ orderService.js
   ✅ userService.js → eventBus ← orderService.js

3. Lazy Require / Dynamic Import
   ❌ Import faylning boshida
   ✅ Import funksiya ichida (kerak bo'lganda)
```

```javascript
// ❌ Circular dependency
// userService.js
import { getOrders } from './orderService.js';
export function getUser(id) { /* ... */ }
export function getUserOrders(id) { return getOrders(id); }

// orderService.js
import { getUser } from './userService.js'; // ❌ CIRCULAR!
export function getOrders(userId) { /* ... */ }
export function getOrderWithUser(orderId) {
  const order = /* ... */;
  order.user = getUser(order.userId);
  return order;
}
```

```javascript
// ✅ Yechim 1: Umumiy modul
// userRepository.js — faqat data access
export function getUserById(id) { /* DB query */ }

// orderRepository.js — faqat data access
export function getOrdersByUserId(userId) { /* DB query */ }

// userService.js — business logic
import { getUserById } from './userRepository.js';
import { getOrdersByUserId } from './orderRepository.js';
export function getUserWithOrders(id) {
  const user = getUserById(id);
  user.orders = getOrdersByUserId(id);
  return user;
}
```

```javascript
// ✅ Yechim 2: Dynamic import
// orderService.js
export async function getOrderWithUser(orderId) {
  const order = /* ... */;
  // Faqat kerak bo'lganda import qiladi — circular bo'lmaydi
  const { getUser } = await import('./userService.js');
  order.user = getUser(order.userId);
  return order;
}
```

---

## Module Bundlers

### Nazariya

Module bundler — bir nechta JavaScript fayllarni **bitta** (yoki bir nechta optimallashtirilgan) faylga birlashtiruvchi vosita. Nima uchun kerak? HTTP/1.1 da 100 ta fayl o'rniga 1 ta yuklash ancha tez, eski browserlar ESM ni qo'llab-quvvatlamaydi, TypeScript/JSX/Sass kabi tillarni transform qilish kerak, va minification, tree shaking, code splitting kabi optimizatsiyalar zarur.

Bundler'lar ishlash prinsipi: entry point fayldan boshlab dependency graph quriladi, keyin barcha modullar bitta (yoki bir nechta) bundle faylga birlashtiriladi. Zamonaviy bundler'lar — **Webpack** (eng keng tarqalgan, konfiguratsiyaga boy), **Vite** (development da tezkor, ESM-based), **esbuild** (Go'da yozilgan, juda tez), **Rollup** (library'lar uchun ideal, yaxshi tree shaking), va **Parcel** (zero-config).

```
Bundler Ishlash Prinsipi:

  Entry Point                    Dependency Graph            Bundle(s)
  ┌──────────┐                  ┌───────────────┐           ┌──────────┐
  │ index.js │───analyze───▶   │   index.js    │──build──▶│ bundle.js│
  └──────────┘                  │   ├── app.js  │           │ (yoki)   │
                                │   │   ├── A.js│           │ main.js  │
  import './app.js'             │   │   └── B.js│           │ vendor.js│
  import './utils.js'           │   └── utils.js│           │ chunk.js │
                                └───────────────┘           └──────────┘

  1. Entry point dan boshlab import/require larni kuzatadi
  2. Barcha dependency larni topadi (dependency graph)
  3. Hammani birlashtiradi va optimize qiladi
  4. Natija: 1 yoki bir nechta optimized bundle
```

### Webpack

Eng mashhur va eng katta ekosistemaga ega bundler (2012-dan beri):

```javascript
// webpack.config.js — asosiy konfiguratsiya
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin');

module.exports = {
  // Entry — qayerdan boshlash
  entry: './src/index.js',

  // Output — natija qayerga yoziladi
  output: {
    path: path.resolve(__dirname, 'dist'),
    filename: '[name].[contenthash].js', // cache busting uchun hash
    clean: true, // eski fayllarni tozalash
  },

  // Mode — development yoki production
  mode: 'production', // auto-minification, tree shaking

  // Module rules — fayllarni qanday qayta ishlash
  module: {
    rules: [
      {
        test: /\.jsx?$/,
        exclude: /node_modules/,
        use: 'babel-loader', // ES6+ → ES5
      },
      {
        test: /\.css$/,
        use: ['style-loader', 'css-loader'], // CSS import qilish
      },
    ],
  },

  // Plugins — qo'shimcha funksionallik
  plugins: [
    new HtmlWebpackPlugin({
      template: './src/index.html',
    }),
  ],

  // Optimization
  optimization: {
    splitChunks: {
      chunks: 'all', // code splitting
    },
  },
};
```

### Vite

Zamonaviy, tez development server va bundler (ESM-based):

```javascript
// vite.config.js
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  build: {
    // Production build (Rollup ishlatadi)
    target: 'es2020',
    outDir: 'dist',
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['react', 'react-dom'],
        },
      },
    },
  },
  server: {
    port: 3000,
    open: true,
  },
});
```

**Vite nima uchun tez?**

```
Webpack (Traditional Bundler):
  ┌────────────────────────────────────────────────────┐
  │  Dev Server Start:                                 │
  │  1. HAMMA faylni bundle qiladi (sekin!)            │
  │  2. Bir katta bundle yaratadi                      │
  │  3. Keyin serve qiladi                             │
  │                                                    │
  │  src/ ──bundle──▶ [===katta===] ──▶ Browser        │
  │  (1000 ta fayl)    (hamma birga)                   │
  └────────────────────────────────────────────────────┘

Vite (ESM-based):
  ┌────────────────────────────────────────────────────┐
  │  Dev Server Start:                                 │
  │  1. Bundle QILMAYDI — native ESM ishlatadi         │
  │  2. Faqat so'ralgan fayllarni transform qiladi     │
  │  3. Browser o'zi import larni resolve qiladi       │
  │                                                    │
  │  Browser ──request──▶ Vite Server ──transform──▶   │
  │  import './App.jsx'    App.jsx → esbuild → JS      │
  │  import './utils.js'   utils.js → serve            │
  │                                                    │
  │  ✅ Darhol ishga tushadi                            │
  │  ✅ Faqat kerakli fayllar transform bo'ladi         │
  │  ✅ HMR juda tez (faqat o'zgargan faylni yangilash) │
  └────────────────────────────────────────────────────┘
```

### esbuild

Go tilida yozilgan — juda tez (Webpack dan 10-100x):

```javascript
// esbuild — CLI bilan
// npx esbuild src/index.js --bundle --outfile=dist/bundle.js --minify

// esbuild — API bilan
import * as esbuild from 'esbuild';

await esbuild.build({
  entryPoints: ['src/index.js'],
  bundle: true,
  minify: true,
  sourcemap: true,
  target: ['es2020'],
  outfile: 'dist/bundle.js',
});
```

### Development vs Production

```
┌──────────────────────────┬──────────────────────────────────┐
│      DEVELOPMENT         │         PRODUCTION               │
├──────────────────────────┼──────────────────────────────────┤
│ Source maps: BOR          │ Source maps: optional/hidden     │
│ Minification: YO'Q       │ Minification: HA                │
│ Tree shaking: YO'Q       │ Tree shaking: HA                │
│ HMR (Hot Module Replace) │ Code splitting: HA              │
│ Debug helpers: BOR       │ Compression (gzip/brotli): HA   │
│ Verbose errors: BOR      │ Asset hashing: HA (cache)       │
│ Fast rebuild: muhim      │ Bundle size: muhim              │
└──────────────────────────┴──────────────────────────────────┘
```

### Bundlerlar Taqqoslash

| Xususiyat | Webpack | Vite | esbuild |
|-----------|---------|------|---------|
| **Tezlik (dev)** | Sekin | Juda tez | Eng tez |
| **Tezlik (build)** | O'rta | Tez (Rollup) | Eng tez |
| **Ekosistema** | Eng katta | O'sib bormoqda | Kichik |
| **Config** | Murakkab | Oddiy | Oddiy |
| **HMR** | O'rta | Juda tez | Cheklangan |
| **Code Splitting** | To'liq | To'liq (Rollup) | Asosiy |
| **Plugin tizimi** | Kuchli (loader + plugin) | Rollup plugins | Cheklangan |
| **Maqsad** | Universal | SPA, modern apps | Speed-critical |

---

## Tree Shaking

### Nazariya

Tree shaking — **ishlatilmagan kodni** final bundle dan olib tashlash texnologiyasi. Nomi: "daraxtni silkitish — quruq barglar tushadi." Bu faqat ES Modules bilan ishlaydi, chunki ESM ning **statik struktura**si bundler'ga compile-time da qaysi export ishlatilganini aniqlash imkonini beradi.

Tree shaking ishlashi uchun: ESM `import`/`export` ishlatish kerak (CommonJS `require()` statik analiz qilib bo'lmaydi), side-effect-free kodlar yozish kerak (modul import qilinganda global state o'zgarmasligi), va `package.json` da `"sideEffects": false` belgilash kerak. `import *` yoki default export o'rniga **named export**'lar ishlatish ham tree shaking'ni yaxshilaydi.

```javascript
// utils.js — 4 ta funksiya bor
export function add(a, b) { return a + b; }
export function subtract(a, b) { return a - b; }
export function multiply(a, b) { return a * b; }
export function divide(a, b) { return a / b; }

// app.js — faqat add ishlatilmoqda
import { add } from './utils.js';
console.log(add(2, 3));
```

```
Tree Shaking natijasi:

TREE SHAKING DAN OLDIN (bundle):
┌────────────────────────────────────┐
│  function add(a, b) { ... }       │ ← ISHLATILGAN ✅
│  function subtract(a, b) { ... }  │ ← ishlatilMAGAN ❌
│  function multiply(a, b) { ... }  │ ← ishlatilMAGAN ❌
│  function divide(a, b) { ... }    │ ← ishlatilMAGAN ❌
│  console.log(add(2, 3));          │ ← ISHLATILGAN ✅
└────────────────────────────────────┘
  Bundle hajmi: ~500 bytes

TREE SHAKING DAN KEYIN (optimized bundle):
┌────────────────────────────────────┐
│  function add(a, b) { ... }       │ ← ✅ qoldi
│  console.log(add(2, 3));          │ ← ✅ qoldi
└────────────────────────────────────┘
  Bundle hajmi: ~150 bytes (70% kichikroq!)
```

### Nima Uchun ESM Kerak?

Tree shaking **faqat ES Modules** bilan yaxshi ishlaydi. Sabab: **static analysis**.

```javascript
// ✅ ESM — Static, compile-time da aniq
import { add } from './utils.js';
// Bundler ANIQ biladi: faqat add kerak, qolganini o'chirish mumkin

// ❌ CJS — Dynamic, runtime da aniqlanadi
const utils = require('./utils');
// Bundler BILMAYDI: utils ning qaysi methodlari ishlatiladi?
// utils[someVariable] ham mumkin — hamma narsani saqlab qolishi kerak!
```

```javascript
// ❌ CJS — tree shaking ishlamaydi
const _ = require('lodash'); // BUTUN lodash bundle ga kiradi (~70KB)
_.get(obj, 'a.b.c');

// ✅ ESM — tree shaking ishlaydi
import { get } from 'lodash-es'; // faqat get funksiya (~2KB)
get(obj, 'a.b.c');

// ✅ Cherry-picking — CJS da ham ishlaydi (lekin manual)
const get = require('lodash/get'); // faqat get (~2KB)
```

### Side Effects

Tree shaking da eng muhim tushuncha — **side effects**. Agar modul import qilinganda biror narsa "bajarsa" (faqat export qilmasdan), bu side effect:

```javascript
// ❌ SIDE EFFECT — import qilishning o'zi biror narsa qiladi
// polyfill.js
Array.prototype.customMethod = function() { /* ... */ };
// Bu faylni import qilish — global Array ni o'zgartiradi
// Tree shaking buni O'CHIRA OLMAYDI (olib tashlasa, polyfill ishlamay qoladi)

// ❌ SIDE EFFECT — CSS import
import './styles.css';
// Import qilishning o'zi stylesheet ni qo'shadi

// ✅ SIDE EFFECT YO'Q — pure export
export function add(a, b) {
  return a + b;
}
// Import qilmasangiz — hech narsa sodir bo'lmaydi
// Tree shaking xavfsiz olib tashlashi mumkin
```

### `sideEffects` in package.json

Kutubxona mualliflariga bundler ga ko'mak berish uchun:

```json
// package.json

// Variant 1: Hech qanday side effect yo'q
{
  "name": "my-math-lib",
  "sideEffects": false
}
// Bundler ishonch bilan ishlatilmagan exportlarni o'chiradi

// Variant 2: Ba'zi fayllarda side effect bor
{
  "name": "my-ui-lib",
  "sideEffects": [
    "*.css",
    "*.scss",
    "./src/polyfills.js"
  ]
}
// Faqat bu fayllar saqlanadi, qolganlarini tree-shake qilish mumkin
```

### Tree Shaking Vizualizatsiya

```
Dependency Graph → Tree Shaking → Final Bundle

app.js
├── import { Button } from './components'
│   ├── components/index.js (barrel)
│   │   ├── export { Button } from './Button'     ✅ ISHLATILGAN
│   │   ├── export { Modal } from './Modal'        ❌ ishlatilMAGAN
│   │   ├── export { Tooltip } from './Tooltip'    ❌ ishlatilMAGAN
│   │   └── export { Table } from './Table'        ❌ ishlatilMAGAN
│   └── Button.js
│       ├── import { cn } from '../utils'          ✅ ISHLATILGAN
│       └── import { Icon } from './Icon'          ✅ ISHLATILGAN
└── import { formatDate } from './utils'
    └── utils/index.js
        ├── export { formatDate }                  ✅ ISHLATILGAN
        ├── export { formatCurrency }              ❌ ishlatilMAGAN
        ├── export { cn }                          ✅ ISHLATILGAN (Button orqali)
        └── export { debounce }                    ❌ ishlatilMAGAN

NATIJA — Bundle faqat quyidagilarni o'z ichiga oladi:
  ✅ app.js (entry)
  ✅ Button.js
  ✅ Icon.js
  ✅ formatDate (utils dan)
  ✅ cn (utils dan)
  ❌ Modal, Tooltip, Table — olib tashlandi
  ❌ formatCurrency, debounce — olib tashlandi
```

### Tree Shaking Ishlashi Uchun Shartlar

```javascript
// ✅ Tree shaking ishlaydi:
// 1. ES Modules ishlatish (import/export)
// 2. Pure functions (side effects yo'q)
// 3. sideEffects: false (package.json)
// 4. Production mode (bundler da)

// ❌ Tree shaking ISHLAMAYDI:
// 1. CommonJS (require/module.exports)
// 2. Dynamic property access
const method = 'add';
utils[method](1, 2);  // bundler nima ishlatilganini bilmaydi

// 3. export default object — juda katta anti-pattern!
export default {
  add(a, b) { return a + b; },
  subtract(a, b) { return a - b; }
};
// Bundler object ichidagi methodlarni individual track qila olmaydi
// BUTUN object bundle ga kiradi!

// ✅ To'g'ri — named exports
export function add(a, b) { return a + b; }
export function subtract(a, b) { return a - b; }
```

---

## Common Mistakes

### ❌ Xato 1: `exports` ni Qayta Tayinlash (CJS)

```javascript
// ❌ Noto'g'ri
exports = { add, subtract };
// require() bo'sh object qaytaradi — chunki exports reference uzildi!
```

### ✅ To'g'ri usul:

```javascript
// ✅ module.exports ishlatish
module.exports = { add, subtract };

// ✅ yoki exports ga property qo'shish
exports.add = add;
exports.subtract = subtract;
```

**Nima uchun:** `exports` — bu `module.exports` ga **reference** (shortcut). `exports = ...` yangi object yaratadi, lekin `require()` doim `module.exports` ni qaytaradi. Reference uziladi.

---

### ❌ Xato 2: ESM da File Extension Unutish (Node.js)

```javascript
// ❌ Noto'g'ri — Node.js ESM da extension SHART
import { add } from './math';     // ❌ ERR_MODULE_NOT_FOUND
import utils from './utils';       // ❌ ERR_MODULE_NOT_FOUND
```

### ✅ To'g'ri usul:

```javascript
// ✅ Extension ko'rsatish — Node.js ESM shart qiladi
import { add } from './math.js';
import utils from './utils.js';
import config from './config.json' with { type: 'json' };
```

**Nima uchun:** CJS da Node.js avtomatik `.js`, `.json`, `.node` qo'shadi. ESM da bu yo'q — aniq bo'lishi kerak. Bundlerlar (Webpack, Vite) bu talabni yumshatishi mumkin, lekin Node.js ESM da shart.

---

### ❌ Xato 3: Default Export ni Noto'g'ri Import Qilish

```javascript
// module.js
export default function processData(data) { /* ... */ }
export function validate(data) { /* ... */ }
```

```javascript
// ❌ Noto'g'ri — default ni {} ichida import qilish
import { processData } from './module.js';
// SyntaxError yoki undefined — processData NAMED export emas!

// ❌ Noto'g'ri — default va named ni chalkashtirib yuborish
import processData, validate from './module.js'; // SyntaxError
```

### ✅ To'g'ri usul:

```javascript
// ✅ Default — {} siz
import processData from './module.js';

// ✅ Named — {} ichida
import { validate } from './module.js';

// ✅ Ikkalasi birga
import processData, { validate } from './module.js';
```

**Nima uchun:** `import { x }` — named export qidiradi. `import x` — default export oladi. Ular boshqa mexanizm. Aralashtirib yuborish keng tarqalgan xato.

---

### ❌ Xato 4: Barrel File da Tree Shaking Buzilishi

```javascript
// components/index.js — barrel file
export { Button } from './Button.js';
export { Modal } from './Modal.js';
export { Table } from './Table.js';
export { Chart } from './Chart.js';  // 📦 katta dependency (chart.js)
```

```javascript
// ❌ Bitta narsa import qildim, lekin hammasi yuklanishi mumkin
import { Button } from './components/index.js';
// Ba'zi bundler konfiguratsiyalarida Chart.js ham bundle ga tushishi mumkin!
```

### ✅ To'g'ri usul:

```javascript
// ✅ To'g'ridan-to'g'ri import (eng xavfsiz)
import { Button } from './components/Button.js';

// ✅ Yoki barrel file + sideEffects: false + zamonaviy bundler
// package.json: { "sideEffects": false }
// Zamonaviy Webpack/Vite buni to'g'ri handle qiladi
```

**Nima uchun:** Barrel files qulay, lekin agar bundler yoki modul side effects to'g'ri sozlanmagan bo'lsa, keraksiz kod ham bundle ga tushishi mumkin. Katta kutubxonalar (lodash, MUI) uchun ayniqsa muhim.

---

### ❌ Xato 5: CJS va ESM ni Chalkashtirib Yuborish

```javascript
// ❌ Noto'g'ri — ESM faylda require
import { readFile } from 'fs';
const config = require('./config.json'); // ❌ ReferenceError: require is not defined

// ❌ Noto'g'ri — CJS faylda import
const fs = require('fs');
import { add } from './math.js'; // ❌ SyntaxError
```

### ✅ To'g'ri usul:

```javascript
// ✅ ESM faylda — hamma narsa import
import { readFile } from 'node:fs/promises';
import config from './config.json' with { type: 'json' };

// ✅ Yoki createRequire bilan (zarur holatlarda)
import { createRequire } from 'node:module';
const require = createRequire(import.meta.url);
const legacyPackage = require('legacy-package');
```

```javascript
// ✅ CJS faylda — hamma narsa require
const fs = require('fs');
const config = require('./config.json');

// Agar ESM module kerak bo'lsa CJS ichida — dynamic import
async function main() {
  const { add } = await import('./math.mjs');
  console.log(add(2, 3));
}
main();
```

**Nima uchun:** CJS va ESM — ikki **turli** modul tizimi. Bir faylda aralashtirib ishlatib bo'lmaydi. `package.json` dagi `"type"` va fayl extension (`.mjs`/`.cjs`) qaysi tizim ishlatilishini belgilaydi.

---

## Amaliy Mashqlar

### Mashq 1: Module Pattern (Oson)

**Savol:** IIFE module pattern bilan `BankAccount` modul yarating. U quyidagi xususiyatlarga ega bo'lsin:
- Private: `balance` (boshlang'ich 0), `transactions` massivi
- Public: `deposit(amount)`, `withdraw(amount)`, `getBalance()`, `getHistory()`
- `withdraw` balans yetarli bo'lmasa xato xabarini qaytarsin

<details>
<summary>Javob</summary>

```javascript
const BankAccount = (function() {
  // Private
  let balance = 0;
  const transactions = [];

  function recordTransaction(type, amount) {
    transactions.push({
      type,
      amount,
      balance,
      date: new Date().toISOString()
    });
  }

  // Public API
  return {
    deposit(amount) {
      if (amount <= 0) return "Noto'g'ri summa";
      balance += amount;
      recordTransaction('deposit', amount);
      return `Kiritildi: ${amount}. Balans: ${balance}`;
    },

    withdraw(amount) {
      if (amount <= 0) return "Noto'g'ri summa";
      if (amount > balance) return `Yetarli mablag' yo'q! Balans: ${balance}`;
      balance -= amount;
      recordTransaction('withdraw', amount);
      return `Yechildi: ${amount}. Balans: ${balance}`;
    },

    getBalance() {
      return balance;
    },

    getHistory() {
      return [...transactions]; // copy qaytaramiz
    }
  };
})();

BankAccount.deposit(1000);    // "Kiritildi: 1000. Balans: 1000"
BankAccount.withdraw(300);    // "Yechildi: 300. Balans: 700"
BankAccount.withdraw(1000);   // "Yetarli mablag' yo'q! Balans: 700"
BankAccount.getBalance();     // 700
BankAccount.getHistory();     // [{type: 'deposit', ...}, {type: 'withdraw', ...}]
console.log(BankAccount.balance);      // undefined — private!
console.log(BankAccount.transactions); // undefined — private!
```

**Tushuntirish:** `balance` va `transactions` IIFE ning local scope'ida — tashqaridan kirish mumkin emas. `return` qilingan object faqat public methodlarni o'z ichiga oladi. Bu **closure** orqali ishlaydi — methodlar tashqi scope'dagi o'zgaruvchilarni "eslab qoladi".
</details>

---

### Mashq 2: CJS va ESM Farqini Tushunish (O'rta)

**Savol:** Quyidagi kodni CJS va ESM da bajaring. Natija farqini tushuntiring.

```javascript
// counter module:
// export let count = 0;
// export function increment() { count++; }

// main module:
// import { count, increment } from './counter';
// console.log('Boshlang\'ich:', count);
// increment();
// increment();
// console.log('Keyin:', count);
```

CJS va ESM da natija nima bo'ladi? Nima uchun?

<details>
<summary>Javob</summary>

**CJS versiya:**

```javascript
// counter.cjs
let count = 0;
function increment() { count++; }
module.exports = { count, increment };

// main.cjs
const { count, increment } = require('./counter.cjs');
console.log('Boshlang\'ich:', count); // 0
increment();
increment();
console.log('Keyin:', count); // 0 ❌ — O'ZGARMADI!
```

**ESM versiya:**

```javascript
// counter.mjs
export let count = 0;
export function increment() { count++; }

// main.mjs
import { count, increment } from './counter.mjs';
console.log('Boshlang\'ich:', count); // 0
increment();
increment();
console.log('Keyin:', count); // 2 ✅ — YANGILANDI!
```

**Tushuntirish:**

- **CJS** da `module.exports = { count, increment }` — `count` ning **qiymati nusxalanadi** (copy). `count` primitive (number), shuning uchun `{ count: 0 }` bo'ladi. `increment()` original `count` ni oshiradi, lekin eksport qilingan nusxaga ta'sir qilmaydi.

- **ESM** da `import { count }` — bu **live binding**. `count` original `count` ga reference. `increment()` original `count` ni oshirganda, import qilingan `count` ham avtomatik yangilanadi — chunki ular **bir xis memory joyiga** ishora qiladi.

Bu ESM ning eng muhim farqlaridan biri — live bindings pattern.
</details>

---

### Mashq 3: Dynamic Import bilan Lazy Loading (O'rta)

**Savol:** Quyidagi vazifani bajaring:
1. `calculators/` papkada 3 ta modul bor: `basic.js`, `scientific.js`, `financial.js`
2. Foydalanuvchi tanlagan calculator turini **dynamic import** bilan yuklang
3. Error handling qo'shing

<details>
<summary>Javob</summary>

```javascript
// calculators/basic.js
export function add(a, b) { return a + b; }
export function subtract(a, b) { return a - b; }

// calculators/scientific.js
export function sin(x) { return Math.sin(x); }
export function cos(x) { return Math.cos(x); }
export function log(x) { return Math.log(x); }

// calculators/financial.js
export function compound(principal, rate, time) {
  return principal * Math.pow(1 + rate, time);
}
export function simpleInterest(principal, rate, time) {
  return principal * rate * time;
}
```

```javascript
// app.js
const VALID_CALCULATORS = ['basic', 'scientific', 'financial'];

async function loadCalculator(type) {
  // 1. Validation
  if (!VALID_CALCULATORS.includes(type)) {
    throw new Error(`Noto'g'ri calculator turi: "${type}". Mavjud: ${VALID_CALCULATORS.join(', ')}`);
  }

  try {
    // 2. Dynamic import
    const module = await import(`./calculators/${type}.js`);
    console.log(`✅ ${type} calculator yuklandi`);
    return module;
  } catch (error) {
    // 3. Error handling
    if (error.code === 'ERR_MODULE_NOT_FOUND') {
      throw new Error(`Calculator modul topilmadi: ${type}`);
    }
    throw error;
  }
}

// Ishlatish
async function main() {
  // Faqat kerakli calculator yuklanadi
  const basic = await loadCalculator('basic');
  console.log(basic.add(10, 5));  // 15

  const fin = await loadCalculator('financial');
  console.log(fin.compound(1000, 0.05, 10));  // 1628.89...

  try {
    await loadCalculator('quantum'); // ❌ Xato
  } catch (err) {
    console.error(err.message); // "Noto'g'ri calculator turi..."
  }
}

main();
```

**Tushuntirish:** `import()` Promise qaytaradi, shuning uchun `async/await` yoki `.then()` bilan ishlatiladi. Har bir calculator faqat kerak bo'lganda yuklanadi — bu **lazy loading** va **code splitting** asosi. Real loyihalarda bu pattern route-based splitting (React.lazy), feature flags, va conditional polyfills uchun keng ishlatiladi.
</details>

---

### Mashq 4: Circular Dependency Topish va Tuzatish (Qiyin)

**Savol:** Quyidagi kodda circular dependency bor. Muammoni toping va tuzating:

```javascript
// auth.js
import { getUserById } from './user.js';
export function isAuthenticated(token) {
  const userId = decodeToken(token);
  const user = getUserById(userId);
  return user && !user.disabled;
}
export function decodeToken(token) {
  return parseInt(token.split('.')[1]);
}

// user.js
import { isAuthenticated } from './auth.js';
export function getUserById(id) {
  return { id, name: 'Ali', disabled: false };
}
export function getProtectedUser(id, token) {
  if (!isAuthenticated(token)) throw new Error('Unauthorized');
  return getUserById(id);
}
```

<details>
<summary>Javob</summary>

**Muammo:** `auth.js` → `user.js` (`getUserById`), `user.js` → `auth.js` (`isAuthenticated`) — circular dependency!

**Yechim 1: Umumiy modul ajratish**

```javascript
// tokenUtils.js — auth logic dan ajratildi
export function decodeToken(token) {
  return parseInt(token.split('.')[1]);
}

// user.js — faqat user data
export function getUserById(id) {
  return { id, name: 'Ali', disabled: false };
}

// auth.js — user va tokenUtils ga bog'liq, lekin ular bir-biriga bog'liq emas
import { getUserById } from './user.js';
import { decodeToken } from './tokenUtils.js';

export function isAuthenticated(token) {
  const userId = decodeToken(token);
  const user = getUserById(userId);
  return user && !user.disabled;
}

// protectedUser.js — auth va user ni birlashtiradi
import { isAuthenticated } from './auth.js';
import { getUserById } from './user.js';

export function getProtectedUser(id, token) {
  if (!isAuthenticated(token)) throw new Error('Unauthorized');
  return getUserById(id);
}
```

```
OLDIN (Circular):
  auth.js ←──────→ user.js  ❌

KEYIN (Clean):
  tokenUtils.js         ← yangi modul
       ↑
  auth.js ──→ user.js   ← bir tomonlama ✅
       ↑          ↑
  protectedUser.js       ← yangi modul, ikkalasini ishlatadi
```

**Yechim 2: Dynamic import**

```javascript
// user.js — circular bo'lgan import ni dynamic qilish
export function getUserById(id) {
  return { id, name: 'Ali', disabled: false };
}

export async function getProtectedUser(id, token) {
  // Lazy import — circular dependency oldini oladi
  const { isAuthenticated } = await import('./auth.js');
  if (!isAuthenticated(token)) throw new Error('Unauthorized');
  return getUserById(id);
}
```

**Tushuntirish:** Circular dependency o'z-o'zidan xato emas, lekin kodni murakkablashtiradi va kutilmagan xatoliklarga olib kelishi mumkin (ayniqsa CJS da partial exports). Eng yaxshi yechim — kodni qayta tashkillashtirish va bir tomonlama dependency yaratish. ESLint `import/no-cycle` rule bu muammolarni avtomatik topishga yordam beradi.
</details>

---

### Mashq 5: Tree Shaking Uchun Optimizatsiya (Qiyin)

**Savol:** Quyidagi kutubxona kodini tree shaking uchun optimallashtirib qayta yozing:

```javascript
// ❌ Hozirgi holat — tree shaking ishlamaydi
const MyLib = {
  formatDate(date) {
    return date.toLocaleDateString();
  },
  formatCurrency(amount) {
    return `$${amount.toFixed(2)}`;
  },
  slugify(text) {
    return text.toLowerCase().replace(/\s+/g, '-');
  },
  capitalize(text) {
    return text.charAt(0).toUpperCase() + text.slice(1);
  }
};
export default MyLib;
```

<details>
<summary>Javob</summary>

```javascript
// ✅ Named exports — tree shaking ishlaydi
// utils/formatDate.js
export function formatDate(date) {
  return date.toLocaleDateString();
}

// utils/formatCurrency.js
export function formatCurrency(amount) {
  return `$${amount.toFixed(2)}`;
}

// utils/slugify.js
export function slugify(text) {
  return text.toLowerCase().replace(/\s+/g, '-');
}

// utils/capitalize.js
export function capitalize(text) {
  return text.charAt(0).toUpperCase() + text.slice(1);
}

// utils/index.js — barrel file
export { formatDate } from './formatDate.js';
export { formatCurrency } from './formatCurrency.js';
export { slugify } from './slugify.js';
export { capitalize } from './capitalize.js';
```

```json
// package.json
{
  "name": "my-lib",
  "version": "1.0.0",
  "type": "module",
  "main": "./dist/index.cjs",
  "module": "./dist/index.mjs",
  "exports": {
    ".": {
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs"
    },
    "./formatDate": "./dist/formatDate.mjs",
    "./formatCurrency": "./dist/formatCurrency.mjs",
    "./slugify": "./dist/slugify.mjs",
    "./capitalize": "./dist/capitalize.mjs"
  },
  "sideEffects": false
}
```

```javascript
// Endi foydalanuvchi:
import { formatDate } from 'my-lib';
// Faqat formatDate bundle ga tushadi — qolganlari yo'q!

// Yoki to'g'ridan-to'g'ri:
import { slugify } from 'my-lib/slugify';
```

**Tushuntirish:**
1. `export default { ... }` — bundler object ichidagi individual methodlarni track qila olmaydi, shuning uchun **BUTUN** object bundle ga kiradi
2. **Named exports** — har bir funksiya alohida export, bundler aniq biladi qaysi birni olib tashlash mumkin
3. `sideEffects: false` — bundlerga "xavfsiz o'chirish mumkin" deb signal beradi
4. `exports` field — to'g'ridan-to'g'ri sub-path import imkonini beradi
5. Har bir funksiya alohida faylda — eng yaxshi tree shaking natijasi
</details>

---

## Xulosa

```
MODULLAR EVOLYUTSIYASI:

  1. Global Scripts (1995-2009)
     <script src="a.js">
     <script src="b.js">
     ❌ Global pollution, dependency chaos

  2. IIFE Module Pattern (2005-2015)
     var Module = (function() { return {...} })();
     ✅ Encapsulation (closure)
     ❌ Manual dependency, no async loading

  3. CommonJS (2009-hozir, Node.js)
     const x = require('./x');
     module.exports = { ... };
     ✅ Server-side modullar
     ❌ Sync only, no tree shaking

  4. ES Modules (2015-hozir, standart)
     import { x } from './x.js';
     export function y() { ... }
     ✅ Static, live bindings, tree shaking
     ✅ Browser + Node.js native
     ✅ Async loading, dynamic import()
```

| Mavzu | Asosiy Nuqta |
|-------|-------------|
| **Nima uchun modullar** | Global scope pollution, namespace collision, dependency management |
| **IIFE Module Pattern** | Closure orqali encapsulation — eski, lekin asosni tushunish muhim. [05-closures.md](05-closures.md), [09-functions.md](09-functions.md) |
| **CommonJS** | `require`/`module.exports` — sync, cached, copy binding. Node.js standart |
| **ES Modules** | `import`/`export` — async, static, live bindings. Til standarti |
| **CJS vs ESM** | Binding (copy vs live), loading (sync vs async), analysis (runtime vs compile-time) |
| **Dynamic import** | `import()` → Promise. Code splitting, lazy loading, conditional imports |
| **Circular deps** | CJS: partial exports. ESM: TDZ xavfi. Yechim: kodni qayta tashkillashtirish |
| **Bundlers** | Webpack (universal), Vite (tez, ESM-based), esbuild (eng tez) |
| **Tree shaking** | Ishlatilmagan kodni o'chirish. Faqat ESM + named exports + sideEffects: false |

**Amaliy Maslahatlar:**

1. **Yangi loyihalar** — doim ESM ishlatish (`"type": "module"` in package.json)
2. **Kutubxona yozsangiz** — dual package (CJS + ESM) tayyorlang
3. **Tree shaking** uchun — named exports, `sideEffects: false`, default object export qilmang
4. **Barrel files** ehtiyot bilan ishlating — katta loyihalarda tree shaking ga ta'sir qilishi mumkin
5. **Circular dependency** dan qoching — `import/no-cycle` ESLint rule yoqing
6. **Dynamic import** — route-based splitting, feature flags, polyfill loading uchun ishlating
