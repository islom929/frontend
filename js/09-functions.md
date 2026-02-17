# Bo'lim 9: Functions — First-Class Citizens

> Function — JavaScript ning yuragi. Ular "birinchi darajali fuqarolar" — o'zgaruvchiga saqlash, argument sifatida berish, funksiyadan qaytarish mumkin. Functional programming ning asosi shu yerda.

---

## Mundarija

- [Function Declaration vs Expression vs Arrow](#function-declaration-vs-expression-vs-arrow)
- [First-Class Functions](#first-class-functions)
- [IIFE — Immediately Invoked Function Expression](#iife--immediately-invoked-function-expression)
- [Higher-Order Functions](#higher-order-functions)
- [Callbacks](#callbacks)
- [Pure Functions va Side Effects](#pure-functions-va-side-effects)
- [Currying](#currying)
- [Partial Application](#partial-application)
- [Function Composition](#function-composition)
- [Memoization](#memoization)
- [Debounce va Throttle](#debounce-va-throttle)
- [`arguments` Object](#arguments-object)
- [Rest Parameters vs Arguments](#rest-parameters-vs-arguments)
- [Default Parameters](#default-parameters)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Function Declaration vs Expression vs Arrow

### Nazariya

JavaScript da funksiya yaratishning **3 ta asosiy usuli** bor va ularning farqini tushunish professional dasturlashning asosiy talabi. Har bir usul o'ziga xos xususiyatlarga ega — hoisting xulq-atvori, `this` binding mexanizmi, `arguments` ob'ekti, va `new` bilan chaqirish imkoniyati. Bu farqlarni bilmaslik production da jiddiy va qiyin topiladigan xatolarga olib keladi.

Nima uchun uchta usul kerak? JavaScript tarixan faqat Function Declaration ga ega edi. Keyin Function Expression qo'shildi — bu funksiyani qiymat sifatida ishlatish imkonini berdi (first-class function). ES6 da Arrow Function kiritildi — u `this` binding muammosini hal qildi va qisqa callback yozish imkonini berdi. Zamonaviy kodda uchala usul ham faol ishlatiladi, lekin har biri o'z joyida.

**1. Function Declaration** — klassik usul, `function` keyword bilan boshlanadi:

```javascript
function calculateTax(income, rate) {
  return income * rate;
}
```

**2. Function Expression** — funksiya o'zgaruvchiga tayinlanadi:

```javascript
const calculateTax = function(income, rate) {
  return income * rate;
};
```

**3. Arrow Function (ES6)** — qisqa syntax, lekin boshqacha xulq:

```javascript
const calculateTax = (income, rate) => income * rate;
```

### Under the Hood

ECMAScript spec bo'yicha uchala usul ham `Function` object yaratadi, lekin **ichki slotlari** farq qiladi:

```
Function Object Internal Slots:

┌─────────────────────────────────────────────────────────────────┐
│                    Function Declaration                         │
│  [[FunctionKind]]: "normal"                                    │
│  [[ThisMode]]: "global" (non-strict) / "strict" (strict mode)  │
│  [[IsClassConstructor]]: false                                 │
│  [[HomeObject]]: undefined                                     │
│  Hoisting: TO'LIQ (nomi + tanasi ko'tariladi)                 │
│  arguments: BOR                                                │
│  new bilan chaqirish: MUMKIN                                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    Function Expression                          │
│  [[FunctionKind]]: "normal"                                    │
│  [[ThisMode]]: "global" / "strict"                             │
│  [[IsClassConstructor]]: false                                 │
│  Hoisting: FAQAT variable (TDZ yoki undefined)                 │
│  arguments: BOR                                                │
│  new bilan chaqirish: MUMKIN                                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    Arrow Function                               │
│  [[FunctionKind]]: "arrow"                                     │
│  [[ThisMode]]: "lexical" ← MUHIM FARQ                         │
│  [[IsClassConstructor]]: false                                 │
│  [[Construct]]: YO'Q ← new bilan chaqirib bo'lmaydi           │
│  Hoisting: FAQAT variable                                      │
│  arguments: YO'Q                                               │
│  prototype: YO'Q                                               │
└─────────────────────────────────────────────────────────────────┘
```

V8 engine ichida funksiya yaratilganda:
1. **Parsing phase** — funksiya AST node'ga aylantiriladi
2. **SharedFunctionInfo** yaratiladi — funksiya "template" si
3. **JSFunction** object yaratiladi — runtime object
4. **[[Environment]]** slot'ga joriy LexicalEnvironment reference saqlanadi (bu closures uchun muhim — [05-closures.md](05-closures.md) da batafsil)
5. Arrow function uchun `[[ThisMode]]: "lexical"` belgilanadi — ya'ni o'zining `this` si yo'q, tashqi scope'dan oladi

### Hoisting Farqlari

```javascript
// ✅ Function Declaration — to'liq hoist bo'ladi
console.log(greet("Ali")); // "Salom, Ali!" — ishlaydi!

function greet(name) {
  return `Salom, ${name}!`;
}
```

```javascript
// ❌ Function Expression (const) — TDZ xato
console.log(greet("Ali")); // ReferenceError: Cannot access 'greet' before initialization

const greet = function(name) {
  return `Salom, ${name}!`;
};
```

```javascript
// ❌ Function Expression (var) — undefined
console.log(greet); // undefined (hoist bo'ldi, lekin faqat variable)
console.log(greet("Ali")); // TypeError: greet is not a function

var greet = function(name) {
  return `Salom, ${name}!`;
};
```

```javascript
// ❌ Arrow Function (const) — TDZ xato
console.log(greet("Ali")); // ReferenceError

const greet = (name) => `Salom, ${name}!`;
```

Bu haqda ko'proq [03-hoisting.md](03-hoisting.md) da.

### `this` Farqlari

Bu — **eng muhim farq**. Arrow function o'zining `this` kontekstiga ega emas.

```javascript
const user = {
  name: "Ali",
  
  // Regular method — this = user (implicit binding)
  greetRegular: function() {
    console.log(`Salom, ${this.name}`);
  },
  
  // Arrow method — this = TASHQI scope (lexical this)
  greetArrow: () => {
    console.log(`Salom, ${this.name}`); // this = window/global!
  }
};

user.greetRegular(); // "Salom, Ali" ✅
user.greetArrow();   // "Salom, undefined" ❌
```

Arrow function'ning `this` o'zlashtirilishi faqat bir holatda foydali — **ichki funksiyalarda**:

```javascript
class Timer {
  constructor() {
    this.seconds = 0;
  }
  
  start() {
    // ❌ Regular function — this yo'qoladi
    // setInterval(function() {
    //   this.seconds++; // this = window/undefined
    // }, 1000);
    
    // ✅ Arrow function — this = Timer instance (lexical)
    setInterval(() => {
      this.seconds++; // this = Timer instance
      console.log(this.seconds);
    }, 1000);
  }
}
```

Bu haqda ko'proq [10-this-keyword.md](10-this-keyword.md) da.

### Kod Misollari

**To'liq taqqoslash:**

```javascript
// 1. Function Declaration
function formatPrice(price, currency = "UZS") {
  return `${price.toLocaleString()} ${currency}`;
}

// 2. Named Function Expression
const formatPrice2 = function formatPriceFn(price, currency = "UZS") {
  // formatPriceFn — faqat funksiya ICHIDA ko'rinadi (recursion uchun foydali)
  return `${price.toLocaleString()} ${currency}`;
};

// 3. Arrow Function
const formatPrice3 = (price, currency = "UZS") => 
  `${price.toLocaleString()} ${currency}`;

// Hammasi bir xil natija beradi:
formatPrice(150000);  // "150,000 UZS"
formatPrice2(150000); // "150,000 UZS"
formatPrice3(150000); // "150,000 UZS"
```

**Arrow function qisqartirish qoidalari:**

```javascript
// Bitta parameter — qavslar shart emas (lekin tavsiya qilinadi)
const double = x => x * 2;
const double2 = (x) => x * 2; // ✅ Yaxshi — consistency uchun

// Parametr yo'q — qavslar shart
const getTimestamp = () => Date.now();

// Bir qatorli — return implicit
const square = (x) => x * x;

// Ko'p qatorli — {} va return kerak
const calculateDiscount = (price, discount) => {
  const discountAmount = price * (discount / 100);
  return price - discountAmount;
};

// Object qaytarish — () bilan o'rash kerak
const createUser = (name, age) => ({ name, age, createdAt: Date.now() });
// () siz { } ni function body deb tushunadi!
```

### Qachon Qaysi Birini Ishlatish?

| Holat | Tavsiya | Sabab |
|-------|---------|-------|
| Object method | Function Declaration / Expression | `this` kerak |
| Callback (map, filter) | Arrow Function | Qisqa, `this` kerak emas |
| Event handler (class ichida) | Arrow Function yoki bind | `this` saqlanishi kerak |
| Constructor | Function Declaration / Class | Arrow da `new` ishlamaydi |
| Top-level utility | Function Declaration | Hoisting foydali |
| Method chaining ichida | Arrow Function | Qisqa syntax |

---

## First-Class Functions

### Nazariya

JavaScript da funksiyalar — **birinchi darajali fuqarolar (first-class citizens)**. Bu degani funksiyalar `number`, `string`, `object` kabi boshqa qiymatlar bilan **teng huquqli** — ularni o'zgaruvchiga saqlash, argument sifatida berish, funksiyadan qaytarish, massiv va ob'ektlarda saqlash mumkin.

Bu xususiyat nima uchun muhim? Ko'pgina dasturlash tillarida (masalan, eski Java versiyalarida) funksiyalar "ikkinchi darajali" edi — ularni o'zgaruvchiga saqlash yoki argument sifatida berish mumkin emas edi. JavaScript funksiyalarning first-class bo'lishi butun tilning **functional programming** qobiliyatlarining asosi — closure, callback, higher-order function, decorator, memoization — bularning barchasi aynan shu xususiyat tufayli mumkin.

ECMAScript spetsifikatsiyasi bo'yicha funksiya aslida **callable object** — ya'ni oddiy ob'ektga `[[Call]]` internal method qo'shilgan maxsus versiya. Shu sababli funksiyaga property qo'shish, `typeof` bilan tekshirish, va ob'ekt kabi ishlash mumkin:

### Under the Hood

ECMAScript spec bo'yicha funksiya — bu **callable object**. Ya'ni, oddiy object'ga `[[Call]]` internal method qo'shilgan versiyasi:

```
┌─────────────────────────────────────┐
│         Function Object              │
├─────────────────────────────────────┤
│  [[Call]]         → executable code  │   ← Shu narsa uni "chaqiriladigan" qiladi
│  [[Construct]]    → new bilan        │   ← (arrow functions da yo'q)
│  [[Environment]]  → outer scope ref  │
│  [[FormalParams]] → parameter list   │
│  name             → "myFunc"         │
│  length           → param count      │
│  prototype        → { }             │   ← (arrow functions da yo'q)
├─────────────────────────────────────┤
│  ... boshqa object properties ...   │   ← Chunki function = object
└─────────────────────────────────────┘
```

Funksiya object bo'lgani uchun, unga **property qo'shish** ham mumkin:

```javascript
function validateEmail(email) {
  return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
}

// Funksiyaga property qo'shish — chunki u object!
validateEmail.errorMessage = "Email formati noto'g'ri";
validateEmail.maxLength = 254;

console.log(validateEmail.name);         // "validateEmail"
console.log(validateEmail.length);       // 1 (parameter soni)
console.log(validateEmail.errorMessage); // "Email formati noto'g'ri"
```

### Kod Misollari

**1. O'zgaruvchiga saqlash:**

```javascript
const log = console.log;
log("Salom!"); // "Salom!" — ishlaydi!

// Strategy pattern — algoritmni o'zgaruvchiga saqlash
const sortStrategies = {
  byName: (a, b) => a.name.localeCompare(b.name),
  byAge: (a, b) => a.age - b.age,
  byDate: (a, b) => new Date(a.createdAt) - new Date(b.createdAt)
};

function sortUsers(users, strategyName) {
  return [...users].sort(sortStrategies[strategyName]);
}
```

**2. Argument sifatida berish (Callback):**

```javascript
const products = [
  { name: "Telefon", price: 5000000 },
  { name: "Noutbuk", price: 12000000 },
  { name: "Quloqchin", price: 300000 }
];

// Funksiyani argument sifatida beryapmiz
const expensive = products.filter(product => product.price > 1000000);
// filter() ichida bizning funksiyamizni CHAQIRADI — shu callback
```

**3. Funksiyadan qaytarish (Factory):**

```javascript
function createFormatter(currency, locale = "uz-UZ") {
  // Funksiya qaytaryapmiz — closure orqali currency/locale ni eslab qoladi
  return function(amount) {
    return new Intl.NumberFormat(locale, {
      style: "currency",
      currency: currency
    }).format(amount);
  };
}

const formatUZS = createFormatter("UZS");
const formatUSD = createFormatter("USD", "en-US");

formatUZS(1500000); // "1 500 000,00 UZS"
formatUSD(99.99);   // "$99.99"
```

**4. Massivda saqlash:**

```javascript
// Middleware pattern (Express.js style)
const middlewares = [
  (req, next) => { req.timestamp = Date.now(); next(); },
  (req, next) => { req.user = authenticate(req); next(); },
  (req, next) => { logRequest(req); next(); }
];

function executeMiddlewares(req, middlewares) {
  let index = 0;
  function next() {
    if (index < middlewares.length) {
      middlewares[index++](req, next);
    }
  }
  next();
}
```

---

## IIFE — Immediately Invoked Function Expression

### Nazariya

**IIFE** (Immediately Invoked Function Expression) — bu funksiya yaratilishi bilan **darhol chaqiriladi**. U scope izolyatsiyasi, module pattern, va bir martalik initialization uchun ishlatiladi.

IIFE JavaScript tarixida juda muhim rol o'ynagan. ES6 dan oldin `let`, `const` va modullar yo'q edi — faqat `var` va function scope bor edi. Shu sababli dasturchilar global scope'ni ifloslantirmaslik uchun IIFE ishlatishgan. jQuery, Lodash, va boshqa mashhur kutubxonalarning barchasi IIFE ichida o'ralgan edi. Zamonaviy kodda ES modules va block scope (`let`/`const`) IIFE ning aksariyat use case'larini almashtirgani bo'lsa-da, legacy kod bilan ishlashda va top-level await mavjud bo'lmagan muhitda async initialization uchun IIFE hali ham kerak.

```javascript
// Klassik IIFE syntax
(function() {
  console.log("Darhol ishga tushdi!");
})();

// Arrow IIFE
(() => {
  console.log("Arrow IIFE!");
})();

// Parametrli IIFE
(function(name) {
  console.log(`Salom, ${name}!`);
})("Ali");
```

**Nima uchun kerak?**

1. **Scope izolyatsiyasi** — o'zgaruvchilar global scope'ni ifloslamaydi
2. **Module pattern** — public/private separation
3. **Bir martalik initialization** kod

### Under the Hood

Engine IIFE ni qanday process qiladi:

```
Source Code:   (function() { ... })()

1. Parser "(function" ni ko'radi:
   → Bu Expression! (Declaration emas, chunki ( bilan boshlangan)

2. Function Expression yaratiladi:
   → JSFunction object hosil bo'ladi
   → [[Environment]] = joriy scope

3. () orqali darhol chaqiriladi:
   → Yangi Execution Context yaratiladi
   → Funksiya tanasi bajariladi

4. Funksiya tugagach:
   → Execution Context stack dan chiqadi
   → Funksiya object ga HECH KIM reference qilmayapti
   → Garbage Collector yo'qotadi

┌──────────────────────────────────┐
│  Global Execution Context        │
│                                  │
│  (function() {                   │
│    ┌─────────────────────────┐   │
│    │  IIFE Execution Context │   │ ← Vaqtincha yaratiladi
│    │  Scope: isolated        │   │
│    │  Variables: local only  │   │
│    └─────────────────────────┘   │
│  })();                           │
│                                  │
│  // IIFE tugadi — hech narsa     │
│  // global scope da qolmadi      │
└──────────────────────────────────┘
```

**Nima uchun `()` kerak?**

```javascript
// ❌ SyntaxError — parser buni declaration deb tushunadi
function() {
  console.log("xato");
}();

// ✅ () ichiga o'rash — bu expression ekanini bildiradi
(function() {
  console.log("ishlaydi");
})();

// Boshqa usullar (kamroq ishlatiladi):
!function() { console.log("!"); }();
+function() { console.log("+"); }();
void function() { console.log("void"); }();
```

### Kod Misollari

**Module Pattern — IIFE bilan encapsulation:**

```javascript
const UserModule = (function() {
  // Private — tashqaridan ko'rinmaydi
  let users = [];
  let nextId = 1;
  
  function generateId() {
    return nextId++;
  }
  
  // Public API — faqat return qilinganlar tashqariga chiqadi
  return {
    add(name, email) {
      const user = { id: generateId(), name, email };
      users.push(user);
      return user;
    },
    
    getAll() {
      return [...users]; // copy qaytaramiz — original ni himoya qilamiz
    },
    
    findById(id) {
      return users.find(u => u.id === id) || null;
    },
    
    get count() {
      return users.length;
    }
  };
})();

UserModule.add("Ali", "ali@example.com");
UserModule.add("Vali", "vali@example.com");
console.log(UserModule.getAll());   // [{id: 1, ...}, {id: 2, ...}]
console.log(UserModule.count);      // 2
// console.log(users);              // ReferenceError — private!
// console.log(UserModule.nextId);  // undefined — private!
```

**Initialization bilan:**

```javascript
const config = (function() {
  const env = process.env.NODE_ENV || "development";
  
  const configs = {
    development: { apiUrl: "http://localhost:3000", debug: true },
    staging: { apiUrl: "https://staging.api.com", debug: true },
    production: { apiUrl: "https://api.com", debug: false }
  };
  
  return Object.freeze(configs[env]); // immutable config
})();

// config tayyor — initialization logic tashqariga chiqmadi
console.log(config.apiUrl); // "http://localhost:3000"
```

### Hozir IIFE Kerakmi?

ES6 modullar va block scope (`let`/`const`) kirgandan keyin, IIFE ning ko'p use case'lari **ortiqcha** bo'ldi:

```javascript
// ❌ Eski usul — IIFE bilan scope izolyatsiya
(function() {
  var secret = "maxfiy";
  // ...
})();

// ✅ Zamonaviy — block scope
{
  const secret = "maxfiy";
  // ...
}
// secret bu yerda ko'rinmaydi

// ✅ Zamonaviy — ES Modules
// module.js
const secret = "maxfiy"; // module scope — global emas
export function getSecret() { return secret; }
```

**Lekin IIFE hali ham kerak bo'ladigan holatlar:**

1. **Legacy kod** bilan ishlashda (`var` ishlatadigan kutubxonalar)
2. **Top-level await** mavjud bo'lmagan muhitda async initialization:

```javascript
// Top-level await yo'q bo'lsa:
(async () => {
  const data = await fetch("/api/config");
  const config = await data.json();
  initApp(config);
})();
```

3. **UMD (Universal Module Definition)** pattern'da

---

## Higher-Order Functions

### Nazariya

**Higher-Order Function (HOF)** — bu boshqa funksiyani **argument sifatida qabul qiladigan**, yoki funksiyani **natija sifatida qaytaradigan**, yoki ikkalasini ham qiladigan funksiya. Bu tushuncha matematikadan kelgan va functional programming ning **asosiy qurilish bloki**.

HOF nima uchun muhim? U kodni **abstraksiya** qilish imkonini beradi — "nima qilish" dan "qanday qilish" ni ajratadi. Masalan, `array.filter(predicate)` da filter "massivda yurib chiqish" logikasini o'z ichiga oladi, siz esa faqat "qaysi elementni olish" qoidasini berasiz. Bu separation of concerns prinsipi bo'lib, kodni o'qish va test qilishni osonlashtiradi.

JavaScript da HOF lar hamma joyda: `map`, `filter`, `reduce`, `addEventListener`, `setTimeout`, `Promise.then`, Express middleware — bularning barchasi HOF. First-class functions tushunchasining **amaliy qo'llanishi** aynan shu yerda namoyon bo'ladi.

```
┌───────────────────────────────────────────────────┐
│              Higher-Order Function                  │
│                                                     │
│  ┌─────────┐     ┌──────────┐     ┌──────────┐    │
│  │ Input   │     │   HOF    │     │ Output   │    │
│  │ fn(x)   │────►│ process  │────►│ fn(y)    │    │
│  └─────────┘     │ using fn │     └──────────┘    │
│                  └──────────┘                      │
│                                                     │
│  Variant 1: Funksiyani QABUL qiladi (callback)    │
│  Variant 2: Funksiyani QAYTARADI (factory)         │
│  Variant 3: Ikkalasi ham (decorator)               │
└───────────────────────────────────────────────────┘
```

### Under the Hood

HOF ning ishlash mexanizmi spec bo'yicha oddiy — funksiya argument sifatida kelganda, u oddiy qiymat kabi **reference** bo'yicha uzatiladi. Funksiya object'ga reference — bu oddiy pointer.

```javascript
function map(array, transformFn) {
  // transformFn — bu oddiy o'zgaruvchi, qiymati function object'ga reference
  // typeof transformFn === "function"
  const result = [];
  for (let i = 0; i < array.length; i++) {
    // transformFn ni CHAQIRYAPMIZ — [[Call]] internal method ishga tushadi
    result.push(transformFn(array[i], i, array));
  }
  return result;
}
```

V8 da HOF optimization: Agar callback **inline arrow function** bo'lsa, TurboFan uni **inline** qilishi mumkin — ya'ni funksiya chaqiruvini yo'qotib, kodni to'g'ridan-to'g'ri joylashtiradi. Bu sezilarli performance yutug'i.

### Kod Misollari

**1. Funksiyani qabul qiluvchi HOF — Array methods:**

```javascript
const transactions = [
  { id: 1, type: "income", amount: 5000000, category: "salary" },
  { id: 2, type: "expense", amount: 200000, category: "food" },
  { id: 3, type: "expense", amount: 1500000, category: "rent" },
  { id: 4, type: "income", amount: 800000, category: "freelance" },
  { id: 5, type: "expense", amount: 50000, category: "food" }
];

// filter — predicate funksiyani qabul qiladi
const expenses = transactions.filter(t => t.type === "expense");

// map — transformer funksiyani qabul qiladi
const summaries = transactions.map(t => `${t.category}: ${t.amount} UZS`);

// reduce — reducer funksiyani qabul qiladi
const totalExpenses = transactions
  .filter(t => t.type === "expense")
  .reduce((sum, t) => sum + t.amount, 0);

console.log(totalExpenses); // 1,750,000
```

**2. Funksiyani qaytaruvchi HOF — Factory:**

```javascript
// Validator factory — turli validatorlar yaratadi
function createValidator(rules) {
  return function validate(value) {
    const errors = [];
    
    for (const rule of rules) {
      const error = rule(value);
      if (error) errors.push(error);
    }
    
    return {
      isValid: errors.length === 0,
      errors
    };
  };
}

// Validation rules
const required = (value) => !value ? "Majburiy maydon" : null;
const minLength = (min) => (value) => 
  value.length < min ? `Kamida ${min} ta belgi` : null;
const isEmail = (value) => 
  !/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value) ? "Email formati noto'g'ri" : null;

// Validatorlar yaratamiz
const validateEmail = createValidator([required, isEmail]);
const validatePassword = createValidator([required, minLength(8)]);

console.log(validateEmail("ali@mail.com"));   // { isValid: true, errors: [] }
console.log(validateEmail("notanemail"));     // { isValid: false, errors: [...] }
console.log(validatePassword("123"));         // { isValid: false, errors: [...] }
```

**3. Ikkalasi ham — Decorator pattern:**

```javascript
// Decorator — funksiyani qabul qiladi VA funksiya qaytaradi
function withLogging(fn) {
  return function(...args) {
    console.log(`[LOG] ${fn.name} chaqirildi:`, args);
    const start = performance.now();
    
    const result = fn.apply(this, args);
    
    const duration = (performance.now() - start).toFixed(2);
    console.log(`[LOG] ${fn.name} tugadi: ${duration}ms, natija:`, result);
    
    return result;
  };
}

function calculateShippingCost(weight, distance) {
  const baseCost = 5000;
  return baseCost + (weight * 100) + (distance * 50);
}

const loggedCalculate = withLogging(calculateShippingCost);
loggedCalculate(2.5, 150);
// [LOG] calculateShippingCost chaqirildi: [2.5, 150]
// [LOG] calculateShippingCost tugadi: 0.02ms, natija: 12750
```

**4. O'zimiz Array.prototype.map yozamiz:**

```javascript
function myMap(array, callback) {
  const result = [];
  for (let i = 0; i < array.length; i++) {
    // callback har bir element uchun chaqiriladi
    // 3 ta argument beradi: element, index, original array
    result.push(callback(array[i], i, array));
  }
  return result;
}

const prices = [100000, 250000, 500000];
const withVAT = myMap(prices, (price) => price * 1.12); // 12% QQS
console.log(withVAT); // [112000, 280000, 560000]
```

---

## Callbacks

### Nazariya

**Callback** — boshqa funksiyaga argument sifatida beriladigan va ma'lum bir voqea yoki shart bajarilganda **chaqiriladigan** funksiya. "Meni keyin chaqir" (call me back) degani — nom aynan shu ma'nodan kelgan. Callback JavaScript ning asinxron dasturlash pattern'larining **tarixiy asosi** bo'lib, Promise va async/await paydo bo'lishidan oldin yagona asinxron mexanizm edi.

Callback ikki xil ishlatiladi: **sinxron callback** (darhol chaqiriladi — `map`, `filter`, `forEach`) va **asinxron callback** (keyinroq, biror voqea sodir bo'lganda chaqiriladi — `setTimeout`, `addEventListener`, `fs.readFile`). Bu ikki turni farqlash muhim — chunki ularning xatti-harakati tubdan farq qiladi.

```javascript
// Synchronous callback — darhol bajariladi
const numbers = [1, 2, 3, 4, 5];
numbers.forEach(function callback(num) {
  console.log(num); // 1, 2, 3, 4, 5 — darhol
});
console.log("Tugadi"); // forEach tugagandan keyin

// Asynchronous callback — keyinroq bajariladi
setTimeout(function callback() {
  console.log("2 sekunddan keyin"); // KEYINROQ
}, 2000);
console.log("Darhol"); // Bu AVVAL chiqadi!
```

### Under the Hood

Async callback'lar qanday ishlaydi — bu Event Loop bilan bog'liq:

```
┌────────────────────────────────────────────────────┐
│                  Runtime Architecture               │
│                                                     │
│  ┌──────────┐   ┌─────────────────┐                │
│  │Call Stack │   │   Web APIs      │                │
│  ├──────────┤   │  (Browser)      │                │
│  │          │   │                 │                │
│  │ main()   │──►│ setTimeout(cb,  │                │
│  │          │   │   2000)         │                │
│  └──────────┘   └───────┬─────────┘                │
│                         │ 2 sek keyin               │
│                         ▼                           │
│                ┌─────────────────┐                  │
│                │ Callback Queue  │                  │
│                │ ┌─────────────┐ │                  │
│                │ │  cb()       │ │                  │
│                │ └─────────────┘ │                  │
│                └────────┬────────┘                  │
│                         │ Event Loop:               │
│                         │ "Stack bo'shmi?"          │
│                         ▼                           │
│                ┌──────────┐                         │
│                │Call Stack │                         │
│                ├──────────┤                         │
│                │  cb()    │ ← Endi bajariladi       │
│                └──────────┘                         │
└────────────────────────────────────────────────────┘
```

Bu haqda ko'proq [11-event-loop.md](11-event-loop.md) da.

### Kod Misollari

**Synchronous callback — ma'lumotni filter qilish:**

```javascript
function processOrders(orders, filterFn, transformFn) {
  return orders
    .filter(filterFn)      // qaysi orderlarni olish — callback hal qiladi
    .map(transformFn);      // qanday o'zgartirish — callback hal qiladi
}

const orders = [
  { id: 1, total: 150000, status: "completed" },
  { id: 2, total: 500000, status: "pending" },
  { id: 3, total: 80000, status: "completed" },
  { id: 4, total: 1200000, status: "pending" }
];

const pendingTotals = processOrders(
  orders,
  (order) => order.status === "pending",          // filter callback
  (order) => ({ id: order.id, total: order.total }) // transform callback
);
// [{ id: 2, total: 500000 }, { id: 4, total: 1200000 }]
```

**Callback Hell — nima uchun Promises kerak bo'ldi:**

```javascript
// ❌ Callback Hell — "pyramid of doom"
getUser(userId, function(user) {
  getOrders(user.id, function(orders) {
    getOrderDetails(orders[0].id, function(details) {
      getShippingInfo(details.shippingId, function(shipping) {
        console.log(shipping);
        // ... 4 daraja chuqurlik — o'qib bo'lmaydi!
      }, handleError);
    }, handleError);
  }, handleError);
}, handleError);

// ✅ Promises bilan — tekis, o'qilishi oson
getUser(userId)
  .then(user => getOrders(user.id))
  .then(orders => getOrderDetails(orders[0].id))
  .then(details => getShippingInfo(details.shippingId))
  .then(shipping => console.log(shipping))
  .catch(handleError);

// ✅✅ Async/Await bilan — eng yaxshi
async function getShipping(userId) {
  const user = await getUser(userId);
  const orders = await getOrders(user.id);
  const details = await getOrderDetails(orders[0].id);
  return await getShippingInfo(details.shippingId);
}
```

Bu haqda ko'proq [12-promises.md](12-promises.md) va [13-async-await.md](13-async-await.md) da.

**Error-first callback pattern (Node.js style):**

```javascript
// Node.js convention: callback(error, result)
function readConfig(path, callback) {
  try {
    const data = fs.readFileSync(path, "utf8");
    const config = JSON.parse(data);
    callback(null, config); // Birinchi argument null = xato yo'q
  } catch (error) {
    callback(error, null); // Birinchi argument error = xato bor
  }
}

readConfig("./config.json", function(error, config) {
  if (error) {
    console.error("Config o'qishda xato:", error.message);
    return;
  }
  console.log("Config:", config);
});
```

---

## Pure Functions va Side Effects

### Nazariya

**Pure Function** — functional programming ning eng asosiy tushunchasi bo'lib, ikki shartni bajaradigan funksiya: birinchidan, **bir xil input doim bir xil output** beradi (deterministic); ikkinchidan, **side effect yo'q** — funksiya tashqi dunyoni hech qanday tarzda o'zgartirmaydi.

Nima uchun purelik muhim? Pure funksiyalar kodni **predictable** (oldindan aytib berish mumkin) qiladi, test yozish osonlashadi (faqat input/output tekshirish yetarli, mock kerak emas), parallel bajarish xavfsiz bo'ladi (shared state yo'q), va caching/memoization mumkin bo'ladi (bir xil input = bir xil output). React'ning butun rendering tizimi pure function konsepsiyasiga asoslangan — component pure bo'lishi kerak, side effect'lar esa `useEffect` ichida bo'lishi kerak.

**Side Effect** — funksiyaning tashqi dunyoga ta'sir qilishi: global o'zgaruvchini o'zgartirish, console ga yozish, DOM ni o'zgartirish, network request, file tizimiga yozish, yoki `Math.random()`/`Date.now()` ishlatish. Side effect'lar zarur (ularsiz dastur foydali ish qila olmaydi), lekin ularni **nazorat qilish** va **izolyatsiya qilish** kerak.

```javascript
// ✅ Pure — bir xil input, bir xil output, side effect yo'q
function calculateTotal(items) {
  return items.reduce((sum, item) => sum + item.price * item.quantity, 0);
}

// ❌ Impure — tashqi o'zgaruvchiga bog'liq
let taxRate = 0.12;
function calculateWithTax(amount) {
  return amount * (1 + taxRate); // taxRate o'zgarsa, natija o'zgaradi!
}

// ❌ Impure — side effect bor (tashqi o'zgaruvchini o'zgartiradi)
let total = 0;
function addToTotal(amount) {
  total += amount; // TASHQI o'zgaruvchini o'zgartirdi — side effect
  return total;
}
```

### Under the Hood

Pure function'lar nima uchun muhim?

1. **Predictable** — test qilish oson, natija oldindan ma'lum
2. **Cacheable** — bir xil input = bir xil output → memoization mumkin
3. **Parallelizable** — shared state yo'q → parallel bajarish xavfsiz
4. **Referential transparency** — funksiya chaqiruvini natija bilan almashtirish mumkin

```
Pure Function:
  f(x) = y    DOIM. Har safar. Istisnosiz.

  calculateTotal([{price: 100, qty: 2}])
  = 200
  = 200    ← har safar 200
  = 200    ← DOIM 200

Impure Function:
  f(x) = ???  Noaniq. Kontekstga bog'liq.

  getDiscount(user)
  = 10%    ← bugun
  = 15%    ← ertaga (aktsiya boshlandi)
  = 0%     ← keyingi oy
```

V8 engine ichida pure function'lar TurboFan tomonidan yanada yaxshi optimize qilinishi mumkin, chunki engine natijani predict qila oladi va **constant folding** qilishi mumkin.

### Kod Misollari

**Pure function yozish qoidalari:**

```javascript
// ❌ Impure — argumentni mutate qiladi
function addItem(cart, item) {
  cart.push(item); // ORIGINAL massivni o'zgartirdi!
  return cart;
}

// ✅ Pure — yangi massiv qaytaradi, originalni o'zgartirmaydi
function addItem(cart, item) {
  return [...cart, item]; // YANGI massiv
}

// ❌ Impure — Date.now() har safar boshqacha
function createUser(name) {
  return { name, createdAt: Date.now() }; // Deterministic emas
}

// ✅ Pure — tashqaridan berilgan timestamp
function createUser(name, timestamp) {
  return { name, createdAt: timestamp };
}

// ❌ Impure — tashqi o'zgaruvchiga bog'liq
const exchangeRate = 12500;
function convertToUSD(uzs) {
  return uzs / exchangeRate;
}

// ✅ Pure — hamma narsa argument orqali
function convertToUSD(uzs, rate) {
  return uzs / rate;
}
```

**Real-world: Redux reducer — pure function namunasi:**

```javascript
// Redux reducer — DOIM pure bo'lishi SHART
function cartReducer(state = { items: [], total: 0 }, action) {
  switch (action.type) {
    case "ADD_ITEM":
      const newItems = [...state.items, action.payload];
      return {
        items: newItems,
        total: newItems.reduce((sum, item) => sum + item.price, 0)
      };
      
    case "REMOVE_ITEM":
      const filtered = state.items.filter(item => item.id !== action.payload);
      return {
        items: filtered,
        total: filtered.reduce((sum, item) => sum + item.price, 0)
      };
      
    default:
      return state; // O'zgarishsiz qaytarish
  }
}

// Pure chunki:
// 1. Bir xil state + action = doim bir xil yangi state
// 2. Eski state ni O'ZGARTIRMAYDI — yangi object qaytaradi
// 3. Side effect yo'q — faqat hisob-kitob
```

**Side Effect'larni ajratish pattern'i:**

```javascript
// ❌ Aralash — logic va side effect bitta joyda
function processPayment(order) {
  const tax = order.total * 0.12;
  const finalAmount = order.total + tax;
  
  console.log(`To'lov: ${finalAmount}`);        // side effect
  sendEmail(order.user, finalAmount);            // side effect
  updateDatabase(order.id, { paid: true });      // side effect
  
  return finalAmount;
}

// ✅ Ajratilgan — pure logic alohida, side effects alohida
// Pure — faqat hisob-kitob
function calculatePayment(total, taxRate) {
  const tax = total * taxRate;
  return { tax, finalAmount: total + tax };
}

// Impure — side effects bu yerda
async function processPayment(order) {
  const { tax, finalAmount } = calculatePayment(order.total, 0.12); // pure
  
  console.log(`To'lov: ${finalAmount}`);
  await sendEmail(order.user, finalAmount);
  await updateDatabase(order.id, { paid: true, amount: finalAmount });
  
  return finalAmount;
}
```

---

## Currying

### Nazariya

**Currying** — ko'p argumentli funksiyani **bir argumentli funksiyalar ketma-ketligiga** aylantirish texnikasi. Ya'ni `f(a, b, c)` o'rniga `f(a)(b)(c)` shaklida chaqirish imkonini beradi. Bu tushuncha matematik Haskell Curry sharafiga nomlangan va functional programming ning fundamental tushunchalaridan biri.

Currying nima uchun kerak? U funksiyalarni **qayta ishlatish** (reusability) va **maxsuslashtirish** (specialization) imkonini beradi. Masalan, `multiply(2)(3)` da `multiply(2)` — bu "ikkiga ko'paytiruvchi" maxsus funksiya bo'lib, uni alohida saqlab boshqa joylarda ishlatish mumkin. Bu ayniqsa function composition va functional pipeline'larda juda foydali.

```
Oddiy funksiya:     f(a, b, c) → natija
Curried funksiya:   f(a)(b)(c) → natija
```

**Nima uchun kerak?**
1. **Partial application** — ba'zi argumentlarni oldindan berish
2. **Function composition** — funksiyalarni ulash
3. **Reusability** — maxsus funksiyalar yaratish

```javascript
// Oddiy funksiya
function multiply(a, b) {
  return a * b;
}
multiply(2, 3); // 6

// Curried versiya
function curriedMultiply(a) {
  return function(b) {
    return a * b;
  };
}
curriedMultiply(2)(3); // 6

// Arrow bilan qisqaroq
const curriedMultiply2 = (a) => (b) => a * b;
```

### Under the Hood

Currying — bu **closures** ning to'g'ridan-to'g'ri qo'llanishi. Har bir qaytarilgan funksiya tashqi scope'dagi argument ni closure orqali eslab qoladi.

```
curriedMultiply(2)(3)

Qadam 1: curriedMultiply(2)
  → Yangi funksiya qaytaradi
  → Bu funksiya a=2 ni CLOSURE orqali eslab qoladi
  ┌──────────────────────────┐
  │  function(b) {           │
  │    return a * b;          │
  │  }                        │
  │  [[Environment]]: {a: 2}  │ ← closure
  └──────────────────────────┘

Qadam 2: (qaytarilgan funksiya)(3)
  → b = 3
  → a ni closure dan oladi: a = 2
  → return 2 * 3 = 6
```

### Kod Misollari

**Universal `curry` funksiyasi:**

```javascript
function curry(fn) {
  return function curried(...args) {
    // Agar yetarli argument bo'lsa — original funksiyani chaqir
    if (args.length >= fn.length) {
      return fn.apply(this, args);
    }
    // Aks holda — keyingi argumentlarni kutuvchi funksiya qaytar
    return function(...nextArgs) {
      return curried.apply(this, [...args, ...nextArgs]);
    };
  };
}

// Ishlatish:
function calculatePrice(basePrice, taxRate, discount) {
  return basePrice * (1 + taxRate) - discount;
}

const curriedPrice = curry(calculatePrice);

// Turli usullarda chaqirish mumkin:
curriedPrice(100000)(0.12)(5000);     // 107000
curriedPrice(100000, 0.12)(5000);     // 107000
curriedPrice(100000)(0.12, 5000);     // 107000
curriedPrice(100000, 0.12, 5000);     // 107000

// Partial application — qayta ishlatish uchun
const withUzbekTax = curriedPrice(100000)(0.12); // taxRate oldindan berildi
withUzbekTax(5000);  // 107000
withUzbekTax(10000); // 102000
```

**Real-world: API so'rovlar uchun curried helper:**

```javascript
const request = curry(function(method, baseUrl, endpoint, data) {
  return fetch(`${baseUrl}${endpoint}`, {
    method,
    headers: { "Content-Type": "application/json" },
    body: data ? JSON.stringify(data) : undefined
  }).then(res => res.json());
});

// Maxsus funksiyalar yaratamiz
const apiGet = request("GET")("https://api.example.com");
const apiPost = request("POST")("https://api.example.com");

// Ishlatish — qisqa va o'qilishi oson
const users = await apiGet("/users");
const newUser = await apiPost("/users")({ name: "Ali", email: "ali@mail.com" });

// Test muhiti uchun boshqa base URL
const testGet = request("GET")("http://localhost:3000");
const testUsers = await testGet("/users");
```

**Currying bilan event handler:**

```javascript
const handleInputChange = curry((fieldName, validator, event) => {
  const value = event.target.value;
  const error = validator(value);
  
  updateFormState(fieldName, value, error);
});

const isRequired = (value) => !value ? "Majburiy" : null;
const isValidAge = (value) => (value < 0 || value > 150) ? "Noto'g'ri yosh" : null;

// Tayyor handlerlar
const handleNameChange = handleInputChange("name")(isRequired);
const handleAgeChange = handleInputChange("age")(isValidAge);

// JSX da:
// <input onChange={handleNameChange} />
// <input onChange={handleAgeChange} />
```

---

## Partial Application

### Nazariya

**Partial Application** — funksiyaning **ba'zi argumentlarini oldindan berish** va qolgan argumentlarni kutuvchi yangi funksiya yaratish texnikasi. Currying bilan o'xshash bo'lsa-da, muhim farqi bor: currying har doim birma-bir argument oladi (`f(a)(b)(c)`), partial application esa bir nechtasini bir yo'la berib qolgan argumentlarni kutadi (`f(a, b)` → `g(c)`).

Partial application nima uchun foydali? U **maxsuslashtirilgan funksiyalar** yaratishning eng oson usuli. Masalan, umumiy `log(level, timestamp, message)` funksiyasidan `errorLog(message)` kabi tayyor funksiya yaratish mumkin. JavaScript'da `Function.prototype.bind` aslida partial application ning built-in implementatsiyasi.

**Currying bilan farqi:**

| Xususiyat | Currying | Partial Application |
|-----------|---------|---------------------|
| **Argument berish** | Birma-bir: `f(a)(b)(c)` | Bir nechtasini birdan: `f(a, b)` → `g(c)` |
| **Qaytarish** | Doim 1-argumentli funksiya | Qolgan argumentlarni kutuvchi funksiya |
| **Maqsad** | Funksiya transformatsiyasi | Argumentlarni oldindan to'ldirish |
| **Argument tartibi** | Chapdan o'ngga qat'iy | Istalgan joyda (placeholder bilan) |

```javascript
// Currying: f(a)(b)(c) — HAR BIR argument alohida
const curriedSum = (a) => (b) => (c) => a + b + c;
curriedSum(1)(2)(3); // 6

// Partial Application: f(a, b) → g(c) — BA'ZILARI oldindan
function sum(a, b, c) { return a + b + c; }
const addTo3 = sum.bind(null, 1, 2); // a=1, b=2 oldindan berildi
addTo3(3); // 6 — faqat c berildi
```

### Kod Misollari

**`bind` bilan partial application:**

```javascript
function log(level, timestamp, message) {
  console.log(`[${level}] ${timestamp}: ${message}`);
}

// Partial application — level oldindan berildi
const logError = log.bind(null, "ERROR");
const logInfo = log.bind(null, "INFO");
const logDebug = log.bind(null, "DEBUG");

// Ishlatish — faqat timestamp va message berish kerak
logError(new Date().toISOString(), "Database ulanishi uzildi");
logInfo(new Date().toISOString(), "Server ishga tushdi");
```

**Custom `partial` funksiyasi (placeholder bilan):**

```javascript
const _ = Symbol("placeholder"); // placeholder

function partial(fn, ...presetArgs) {
  return function(...laterArgs) {
    let laterIndex = 0;
    // Placeholder'larni laterArgs bilan almashtir
    const finalArgs = presetArgs.map(arg => 
      arg === _ ? laterArgs[laterIndex++] : arg
    );
    // Qolgan argumentlarni qo'sh
    return fn(...finalArgs, ...laterArgs.slice(laterIndex));
  };
}

// Ishlatish
function createEvent(type, date, title, description) {
  return { type, date, title, description };
}

// type oldindan berildi, date placeholder, qolganlari keyinroq
const createMeeting = partial(createEvent, "meeting", _);
const createDeadline = partial(createEvent, "deadline", _);

createMeeting("2025-03-15", "Sprint Planning", "Yangi sprint rejasi");
// { type: "meeting", date: "2025-03-15", title: "Sprint Planning", ... }

createDeadline("2025-03-20", "Release v2.0", "Production deploy");
// { type: "deadline", date: "2025-03-20", title: "Release v2.0", ... }
```

**Real-world: React component'larida partial application:**

```javascript
function TodoList({ todos, onToggle, onDelete }) {
  return todos.map(todo => (
    // Partial application — todo.id oldindan beriladi
    <TodoItem
      key={todo.id}
      todo={todo}
      onToggle={() => onToggle(todo.id)}
      onDelete={() => onDelete(todo.id)}
    />
  ));
}

// Yoki bind bilan:
function TodoList({ todos, onToggle, onDelete }) {
  return todos.map(todo => (
    <TodoItem
      key={todo.id}
      todo={todo}
      onToggle={onToggle.bind(null, todo.id)}
      onDelete={onDelete.bind(null, todo.id)}
    />
  ));
}
```

---

## Function Composition

### Nazariya

**Function Composition** — bir nechta kichik funksiyalarni **birlashtirib**, yangi murakkab funksiya yaratish texnikasi. Matematikadagi `f(g(x))` ga to'g'ri keladi — bitta funksiyaning natijasi keyingisining inputi bo'ladi.

Composition functional programming ning **eng asosiy prinsipi**: kichik, sodda, pure funksiyalarni yozib, keyin ularni birlashtirish orqali murakkab operatsiyalar yaratish. Bu yondashuv kodni modular qiladi (har bir funksiya bitta vazifani bajaradi), test qilish osonlashadi (har bir funksiya alohida test qilinadi), va qayta ishlatish imkoniyati ortadi.

Ikki asosiy usul bor: `compose(f, g, h)` — o'ngdan chapga (`f(g(h(x)))`), va `pipe(h, g, f)` — chapdan o'ngga, tabiiy o'qish tartibida. Zamonaviy kodda `pipe` ko'proq ishlatiladi chunki u o'qishga osonroq.

### Under the Hood

Composition ichida nima sodir bo'ladi:

```
pipe(trim, toLowerCase, addExclamation)("  SALOM  ")

Qadam 1: trim("  SALOM  ")      → "SALOM"
Qadam 2: toLowerCase("SALOM")   → "salom"
Qadam 3: addExclamation("salom") → "salom!"

Natija: "salom!"

Vizual pipeline:
┌──────────┐    ┌──────────────┐    ┌─────────────────┐
│  "  SALOM  "  │    │   "SALOM"      │    │    "salom"        │
│          │    │              │    │                 │
│   trim   │───►│ toLowerCase  │───►│ addExclamation  │───► "salom!"
│          │    │              │    │                 │
└──────────┘    └──────────────┘    └─────────────────┘
```

### Kod Misollari

**`compose` va `pipe` implementatsiyasi:**

```javascript
// compose — o'ngdan chapga bajaradi: compose(f, g, h)(x) = f(g(h(x)))
function compose(...fns) {
  return function(x) {
    return fns.reduceRight((acc, fn) => fn(acc), x);
  };
}

// pipe — chapdan o'ngga bajaradi: pipe(h, g, f)(x) = f(g(h(x)))
function pipe(...fns) {
  return function(x) {
    return fns.reduce((acc, fn) => fn(acc), x);
  };
}

// Kichik, pure funksiyalar
const trim = (str) => str.trim();
const toLowerCase = (str) => str.toLowerCase();
const replaceSpaces = (str) => str.replace(/\s+/g, "-");
const addPrefix = (prefix) => (str) => `${prefix}${str}`;

// compose bilan — o'ngdan chapga o'qiladi
const createSlug = compose(
  addPrefix("/blog/"),
  replaceSpaces,
  toLowerCase,
  trim
);

// pipe bilan — chapdan o'ngga o'qiladi (tabiiyroq)
const createSlug2 = pipe(
  trim,
  toLowerCase,
  replaceSpaces,
  addPrefix("/blog/")
);

createSlug("  JavaScript Functions Guide  ");
// "/blog/javascript-functions-guide"

createSlug2("  JavaScript Functions Guide  ");
// "/blog/javascript-functions-guide"
```

**Real-world: Ma'lumotni qayta ishlash pipeline:**

```javascript
const pipe = (...fns) => (x) => fns.reduce((acc, fn) => fn(acc), x);

// Kichik transformer funksiyalar
const filterActive = (users) => users.filter(u => u.isActive);
const sortByName = (users) => [...users].sort((a, b) => a.name.localeCompare(b.name));
const addFullName = (users) => users.map(u => ({
  ...u,
  fullName: `${u.firstName} ${u.lastName}`
}));
const limitTo = (n) => (users) => users.slice(0, n);
const formatForUI = (users) => users.map(u => ({
  id: u.id,
  display: u.fullName,
  email: u.email
}));

// Pipeline — har biri bitta vazifa bajaradi
const getActiveUsersForDropdown = pipe(
  filterActive,
  addFullName,
  sortByName,
  limitTo(10),
  formatForUI
);

const dropdownData = getActiveUsersForDropdown(rawUsers);
```

**Async composition — pipe bilan:**

```javascript
function pipeAsync(...fns) {
  return function(input) {
    return fns.reduce(
      (chain, fn) => chain.then(fn),
      Promise.resolve(input)
    );
  };
}

const processOrder = pipeAsync(
  validateOrder,          // order → validated order
  calculateTotal,         // order → order with total
  applyDiscounts,         // order → order with discounts
  processPayment,         // order → order with payment
  sendConfirmationEmail   // order → order with email sent
);

// Ishlatish
const result = await processOrder(newOrder);
```

---

## Memoization

### Nazariya

**Memoization** — funksiya natijasini **cache** qilish pattern'i. Agar funksiya avval bir xil argumentlar bilan chaqirilgan bo'lsa, qayta hisob-kitob qilmasdan **saqlangan natijani qaytaradi**. Bu xotira va tezlik o'rtasidagi **trade-off** — qo'shimcha xotira sarflash evaziga ishlash tezligini oshirish.

Memoization nima uchun muhim? Og'ir hisob-kitoblar (masalan, Fibonacci, faktorial, yoki katta ma'lumotlarni qayta ishlash) bir xil inputlar bilan ko'p marta chaqirilishi mumkin. Har safar qayta hisoblash o'rniga natijani cache'da saqlash performance ni keskin oshiradi. React'da `useMemo` va `React.memo` aynan shu printsipga asoslanadi.

**Qachon ishlatish kerak:** funksiya **pure** bo'lganda, hisob-kitob **sekin** bo'lganda, va bir xil argumentlar bilan **ko'p marta** chaqirilganda. **Qachon ishlatmaslik kerak:** funksiya impure bo'lganda (side effect, random, time), argumentlar har doim unikal bo'lganda (cache befoyda), yoki memory cheklangan bo'lganda.

### Under the Hood

Memoization oddiy — ichida **Map** (yoki object) bor, argument ni key sifatida, natijani value sifatida saqlaydi:

```
┌──────────────────────────────────────────────┐
│             Memoized Function                 │
│                                               │
│  Cache (Map):                                 │
│  ┌──────────────────┬──────────────────┐     │
│  │     Key (args)    │   Value (result) │     │
│  ├──────────────────┼──────────────────┤     │
│  │  "40"             │   102334155      │     │
│  │  "35"             │   9227465        │     │
│  │  "30"             │   832040         │     │
│  └──────────────────┴──────────────────┘     │
│                                               │
│  Chaqiruv:  memoFib(40)                       │
│  1. Cache da "40" bormi? → HA                 │
│  2. Qayta hisoblash SHART emas                │
│  3. Cache dan qaytarish: 102334155             │
└──────────────────────────────────────────────┘
```

### Kod Misollari

**Oddiy `memoize` implementatsiyasi:**

```javascript
function memoize(fn) {
  const cache = new Map();
  
  return function(...args) {
    const key = JSON.stringify(args); // argumentlarni string ga aylantirish
    
    if (cache.has(key)) {
      console.log(`Cache hit: ${key}`);
      return cache.get(key);
    }
    
    console.log(`Computing: ${key}`);
    const result = fn.apply(this, args);
    cache.set(key, result);
    return result;
  };
}

// Fibonacci — memoization siz O(2^n), bilan O(n)
function fibonacci(n) {
  if (n <= 1) return n;
  return fibonacci(n - 1) + fibonacci(n - 2);
}

const memoFib = memoize(function fib(n) {
  if (n <= 1) return n;
  return memoFib(n - 1) + memoFib(n - 2); // O'zini o'zi memo bilan chaqiradi
});

console.time("Birinchi");
memoFib(40); // Computing... ~0.5ms (memoized recursive)
console.timeEnd("Birinchi");

console.time("Ikkinchi");
memoFib(40); // Cache hit! ~0.01ms
console.timeEnd("Ikkinchi");
```

**Production-ready memoize (LRU cache bilan):**

```javascript
function memoizeWithLimit(fn, maxSize = 100) {
  const cache = new Map();
  
  return function(...args) {
    const key = JSON.stringify(args);
    
    if (cache.has(key)) {
      // LRU: Ishlatilingan element oxiriga o'tkaziladi
      const value = cache.get(key);
      cache.delete(key);
      cache.set(key, value);
      return value;
    }
    
    const result = fn.apply(this, args);
    
    // Cache to'lgan bo'lsa, eng eski elementni o'chirish
    if (cache.size >= maxSize) {
      const firstKey = cache.keys().next().value;
      cache.delete(firstKey);
    }
    
    cache.set(key, result);
    return result;
  };
}

// API natijalarini cache qilish
const fetchUserProfile = memoizeWithLimit(async function(userId) {
  const response = await fetch(`/api/users/${userId}`);
  return response.json();
}, 50); // Oxirgi 50 ta foydalanuvchi profili cache da

// Birinchi chaqiruv — network request
await fetchUserProfile("user-123"); // fetch → /api/users/user-123

// Ikkinchi chaqiruv — cache dan (network request yo'q)
await fetchUserProfile("user-123"); // cache hit!
```

**React useMemo/useCallback ga o'xshash pattern:**

```javascript
// Og'ir hisob-kitob uchun memoize
const calculateExpensiveReport = memoize(function(transactions, startDate, endDate) {
  console.log("Og'ir hisob-kitob boshlanmoqda...");
  
  return transactions
    .filter(t => t.date >= startDate && t.date <= endDate)
    .reduce((report, t) => {
      report.total += t.amount;
      report.count++;
      report.categories[t.category] = (report.categories[t.category] || 0) + t.amount;
      return report;
    }, { total: 0, count: 0, categories: {} });
});

// Birinchi chaqiruv — hisob-kitob bajariladi
const report1 = calculateExpensiveReport(transactions, "2025-01-01", "2025-01-31");

// Xuddi shu argumentlar bilan — cache dan
const report2 = calculateExpensiveReport(transactions, "2025-01-01", "2025-01-31");
// report1 === report2 — bir xil reference!
```

---

## Debounce va Throttle

### Nazariya

**Debounce** va **Throttle** — ko'p chaqiriladigan funksiyalarni **cheklash** pattern'lari. Ikkalasi ham performance optimization uchun ishlatiladi, lekin har biri boshqa muammoni hal qiladi.

**Debounce** — foydalanuvchi **to'xtagandan keyin** bitta marta bajaradi. Tasavvur qiling, lift tugmasini bossangiz, lift darhol yopilmaydi — boshqa odam ham kirishini kutadi. Agar 3 soniya hech kim bosmasa — eshik yopiladi. Xuddi shunday, debounce foydalanuvchi yozishni to'xtatganidan keyin gina search so'rovini yuboradi.

**Throttle** — berilgan vaqt oralig'ida **ko'pi bilan bitta marta** bajaradi. Buni **chastota cheklovchi** deb tushunish mumkin — qancha tez bosmang, har 300ms da birgina marta ishlaydi. Scroll event, resize, yoki sensor ma'lumotlarini qayta ishlashda throttle ishlatiladi.

Bu ikki pattern zamonaviy frontend dasturlashda **deyarli har doim** kerak: search input'larda debounce, scroll/resize handler'larda throttle, API rate limiting'da throttle. Lodash va boshqa kutubxonalarda tayyor implementatsiyalar bor, lekin ularni o'zingiz yoza olish — closure va timer'lar bilan ishlashni to'liq tushunganingiz isboti.

```
Debounce (300ms):
User events:   ─x─x─x─x─x─────────────x─x─────────
Executions:    ──────────────────f()────────────f()──
                                 ↑                ↑
                           300ms kutdi       300ms kutdi

Throttle (300ms):
User events:   ─x─x─x─x─x─x─x─x─x─────────
Executions:    ─f()────────f()────────f()─────
               ↑           ↑          ↑
          Darhol      300ms keyin  300ms keyin
```

### Under the Hood

Ikkala pattern ham `setTimeout` va closures ga asoslanadi:

```
Debounce mexanizmi:
┌─────────────────────────────────────────────┐
│  Har bir chaqiruvda:                         │
│  1. Oldingi timer ni BEKOR QIL (clearTimeout)│
│  2. Yangi timer BOSHLASH (setTimeout)        │
│  3. Timer tugasa → funksiyani bajir          │
│                                              │
│  Natija: Faqat OXIRGI chaqiruvdan keyin      │
│  delay vaqt o'tsa bajariladi                 │
└─────────────────────────────────────────────┘

Throttle mexanizmi:
┌─────────────────────────────────────────────┐
│  Har bir chaqiruvda:                         │
│  1. Agar "kutish" rejimida → SKIP           │
│  2. Aks holda → funksiyani BAJIR            │
│  3. "Kutish" rejimiga o'tkazish              │
│  4. delay vaqt o'tganda → "tayyor" rejimiga  │
│                                              │
│  Natija: Har delay ms da KO'PI BILAN 1 marta│
└─────────────────────────────────────────────┘
```

### Kod Misollari

**Debounce — noldan yozamiz:**

```javascript
function debounce(fn, delay) {
  let timeoutId = null;
  
  return function(...args) {
    // 1. Oldingi kutishni bekor qil
    clearTimeout(timeoutId);
    
    // 2. Yangi kutish boshla
    timeoutId = setTimeout(() => {
      fn.apply(this, args); // this kontekstni saqlab chaqir
    }, delay);
  };
}

// Ishlatish: Search input
const searchInput = document.getElementById("search");

const handleSearch = debounce(function(event) {
  const query = event.target.value;
  console.log(`Qidirish: "${query}"`);
  // fetchSearchResults(query); — API ga so'rov
}, 300);

searchInput.addEventListener("input", handleSearch);

// Foydalanuvchi "javascript" yozsa:
// j → timer boshlandi (300ms)
// a → oldingi timer bekor, yangi timer (300ms)
// v → oldingi timer bekor, yangi timer (300ms)
// ...
// t → oldingi timer bekor, yangi timer (300ms)
// (300ms o'tdi, boshqa yozmadi)
// → fetchSearchResults("javascript") — FAQAT 1 MARTA!
```

**Debounce with leading/trailing options:**

```javascript
function debounce(fn, delay, options = {}) {
  let timeoutId = null;
  let lastArgs = null;
  const { leading = false, trailing = true } = options;
  
  return function(...args) {
    const isFirstCall = timeoutId === null;
    lastArgs = args;
    
    clearTimeout(timeoutId);
    
    // Leading: birinchi chaqiruvda darhol bajir
    if (leading && isFirstCall) {
      fn.apply(this, args);
    }
    
    timeoutId = setTimeout(() => {
      // Trailing: kutish tugaganda bajir
      if (trailing && lastArgs) {
        // Leading bo'lsa va bitta argument bo'lgan bo'lsa, takrorlamaslik
        if (!(leading && isFirstCall)) {
          fn.apply(this, lastArgs);
        }
      }
      timeoutId = null;
      lastArgs = null;
    }, delay);
  };
}

// Leading — darhol bajir, keyin delay kutish
const saveImmediately = debounce(saveDraft, 1000, { leading: true, trailing: false });

// Trailing (default) — kutib, oxirida bajir
const saveAfterPause = debounce(saveDraft, 1000);

// Both — boshida VA oxirida bajir
const saveBoth = debounce(saveDraft, 1000, { leading: true, trailing: true });
```

**Throttle — noldan yozamiz:**

```javascript
function throttle(fn, limit) {
  let inThrottle = false;
  let lastArgs = null;
  let lastContext = null;
  
  return function(...args) {
    if (!inThrottle) {
      // Tayyor — darhol bajir
      fn.apply(this, args);
      inThrottle = true;
      
      setTimeout(() => {
        inThrottle = false;
        // Agar kutish vaqtida chaqiruv bo'lgan bo'lsa — oxirgisini bajir
        if (lastArgs) {
          fn.apply(lastContext, lastArgs);
          lastArgs = null;
          lastContext = null;
          inThrottle = true;
          setTimeout(() => { inThrottle = false; }, limit);
        }
      }, limit);
    } else {
      // Kutish rejimida — oxirgi argumentlarni saqla
      lastArgs = args;
      lastContext = this;
    }
  };
}

// Ishlatish: Scroll pozitsiyani tracking
const handleScroll = throttle(function() {
  const scrollPos = window.scrollY;
  console.log(`Scroll: ${scrollPos}px`);
  
  // Infinite scroll: sahifa oxiriga yaqinlashganda yangi content yuklash
  if (scrollPos + window.innerHeight >= document.body.offsetHeight - 500) {
    loadMoreContent();
  }
}, 200);

window.addEventListener("scroll", handleScroll);
```

**Debounce vs Throttle — qachon qaysi biri?**

| Use Case | Debounce | Throttle |
|----------|----------|----------|
| Search input (yozib tugagandan keyin) | ✅ | |
| Window resize (responsive layout) | ✅ | |
| Form autosave (yozish pauzasida) | ✅ | |
| Scroll pozitsiya tracking | | ✅ |
| Button click (double-click oldini olish) | ✅ (leading) | |
| API rate limiting | | ✅ |
| Mousemove tracking | | ✅ |
| Game loop (FPS cheklash) | | ✅ |

**Cancel qilish imkoniyati bilan:**

```javascript
function debounce(fn, delay) {
  let timeoutId = null;
  
  function debounced(...args) {
    clearTimeout(timeoutId);
    timeoutId = setTimeout(() => fn.apply(this, args), delay);
  }
  
  // Cancel method — tashqaridan bekor qilish uchun
  debounced.cancel = function() {
    clearTimeout(timeoutId);
    timeoutId = null;
  };
  
  // Flush method — darhol bajarish uchun
  debounced.flush = function() {
    clearTimeout(timeoutId);
    fn.apply(this);
    timeoutId = null;
  };
  
  return debounced;
}

// Component unmount da cancel qilish (React)
const debouncedSave = debounce(saveToServer, 1000);

// useEffect cleanup:
// return () => debouncedSave.cancel();
```

---

## `arguments` Object

### Nazariya

`arguments` — har bir **regular function** (declaration va expression) ichida mavjud bo'lgan **array-like** (massivga o'xshash) ob'ekt. U funksiyaga berilgan **barcha argumentlarni** indeks bo'yicha saqlaydi, parametrlar soni necha bo'lishidan qat'i nazar.

`arguments` JavaScript'ning eng eski xususiyatlaridan biri bo'lib, ES1 dan beri mavjud. U funksiya nechta argument qabul qilishini oldindan bilmaslik muammosini hal qilgan. Lekin `arguments` ning **jiddiy kamchiliklari** bor: u haqiqiy massiv emas (Array.prototype method'lari yo'q), arrow function'da mavjud emas, va strict mode'da ba'zi kutilmagan xulq-atvor ko'rsatadi. Shu sababli ES6 da uning zamonaviy almashtiruvi — **rest parameters** (`...args`) kiritildi.

**Muhim:** `arguments` **haqiqiy massiv emas** — u `Array.prototype` ga ega emas. `map`, `filter`, `reduce` kabi method'lar to'g'ridan-to'g'ri ishlamaydi — avval `Array.from(arguments)` yoki `[...arguments]` bilan haqiqiy massivga aylantirish kerak.

### Under the Hood

ECMAScript spec bo'yicha `arguments` object — bu **exotic object** (oddiy objectdan farqli). Uning ba'zi maxsus xususiyatlari:

```
┌─────────────────────────────────────────────┐
│             arguments Object                 │
├─────────────────────────────────────────────┤
│  Type: Arguments exotic object               │
│                                              │
│  Properties:                                 │
│  ├── 0: birinchi argument                    │
│  ├── 1: ikkinchi argument                    │
│  ├── ...                                     │
│  ├── length: argumentlar soni                │
│  ├── callee: funksiya o'zi (strict da yo'q)  │
│  └── Symbol.iterator: iterable!              │
│                                              │
│  Array-LIKE:                                 │
│  ✅ length bor                               │
│  ✅ index bilan kirish mumkin [0], [1]       │
│  ✅ for...of ishlaydi (iterable)             │
│  ❌ Array.prototype methods YO'Q             │
│  ❌ push, pop, map, filter — ISHLAMAYDI      │
└─────────────────────────────────────────────┘
```

**Non-strict mode da** `arguments` parameter'lar bilan **sync** qilinadi:

```javascript
function test(a, b) {
  arguments[0] = 99;
  console.log(a); // 99 — arguments o'zgarsa, parameter ham o'zgaradi!
  
  a = 50;
  console.log(arguments[0]); // 50 — teskari ham ishlaydi!
}
test(1, 2);
```

**Strict mode da** bu bog'liqlik **yo'q**:

```javascript
"use strict";
function test(a, b) {
  arguments[0] = 99;
  console.log(a); // 1 — a o'zgarmadi! (strict mode)
}
test(1, 2);
```

### Arrow Function da arguments YO'Q

Bu — ko'p so'raladigan narsa. Arrow function o'zining `arguments` object'iga **ega emas**. Agar `arguments` ga murojaat qilsa, u **tashqi scope'dan** (closure orqali) olishga harakat qiladi:

```javascript
function outer() {
  console.log(arguments); // Arguments [1, 2, 3]
  
  const inner = () => {
    console.log(arguments); // Arguments [1, 2, 3] — tashqi scope'dan!
    // Bu outer ning arguments'i — o'ziniki emas
  };
  
  inner();
}
outer(1, 2, 3);
```

```javascript
// Global scope da arrow function:
const showArgs = () => {
  console.log(arguments); // ReferenceError: arguments is not defined
  // Tashqi scope'da ham arguments yo'q!
};
showArgs(1, 2, 3);
```

### Kod Misollari

**`arguments` ni Array ga aylantirish:**

```javascript
function sum() {
  // Usul 1: Array.from (zamonaviy, tavsiya qilinadi)
  const args = Array.from(arguments);
  
  // Usul 2: Spread operator
  const args2 = [...arguments];
  
  // Usul 3: Array.prototype.slice (eski usul)
  const args3 = Array.prototype.slice.call(arguments);
  
  return args.reduce((total, num) => total + num, 0);
}

sum(1, 2, 3, 4, 5); // 15
```

**Lekin bugun `rest parameters` ishlatish kerak:**

```javascript
// ❌ Eski usul — arguments
function sum() {
  return Array.from(arguments).reduce((total, num) => total + num, 0);
}

// ✅ Zamonaviy — rest parameters
function sum(...numbers) {
  return numbers.reduce((total, num) => total + num, 0);
}

// ✅ Arrow bilan ham ishlaydi
const sum2 = (...numbers) => numbers.reduce((total, num) => total + num, 0);
```

---

## Rest Parameters vs Arguments

### Nazariya

**Rest parameters** (`...`) — ES6 da kiritilgan, `arguments` ob'ektining zamonaviy va kuchli almashtiruvi. `arguments` dan farqli o'laroq, rest parameters **haqiqiy Array** qaytaradi (barcha Array method'lari ishlaydi), **arrow function**'larda ishlaydi, va faqat **qolgan argumentlarni** oladi (nomlangan parametrlardan keyingilarini).

Nima uchun rest parameters kerak bo'ldi? `arguments` ob'ektining kamchiliklari (array-like, arrow function'da yo'q, strict mode muammolari) ko'p xatolarga sabab bo'lardi. Rest parameters bu muammolarning barchasini hal qildi va zamonaviy kodda `arguments` ishlatish uchun hech qanday sabab qolmadi. Bugungi kunda `arguments` faqat legacy kodda uchraydi.

### Under the Hood

```
arguments object:
function test(a, b) { ... }
test(1, 2, 3, 4);
→ arguments = {0: 1, 1: 2, 2: 3, 3: 4, length: 4}  // HAMMA argument

Rest parameters:
function test(a, b, ...rest) { ... }
test(1, 2, 3, 4);
→ a = 1, b = 2
→ rest = [3, 4]  // Faqat QOLGAN argumentlar — haqiqiy Array!
```

V8 ichida rest parameters optimizatsiyasi: V8 rest parameters uchun `arguments` object yaratmaydi — to'g'ridan-to'g'ri Array yaratadi. Bu **yanada tezroq** ishlaydi, chunki `arguments` exotic object yaratish qo'shimcha overhead talab qiladi.

### Kod Misollari

```javascript
// Rest parametrlar doim OXIRIDA bo'lishi kerak
function createLog(level, timestamp, ...messages) {
  // level = "ERROR"
  // timestamp = "2025-03-15T10:30:00Z"
  // messages = ["Database", "connection", "failed"] — haqiqiy Array
  
  const formatted = messages.join(" ");
  console.log(`[${level}] ${timestamp}: ${formatted}`);
}

createLog("ERROR", "2025-03-15T10:30:00Z", "Database", "connection", "failed");
// [ERROR] 2025-03-15T10:30:00Z: Database connection failed
```

```javascript
// ❌ SyntaxError — rest oxirida bo'lishi SHART
// function test(...args, last) {} // SyntaxError!

// ✅ To'g'ri
function test(first, ...rest) {
  console.log(first); // 1
  console.log(rest);  // [2, 3, 4]
}
test(1, 2, 3, 4);
```

**Destructuring bilan birgalikda:**

```javascript
function processResponse({ status, data, ...metadata }) {
  console.log(status);    // 200
  console.log(data);      // { users: [...] }
  console.log(metadata);  // { headers: {...}, timestamp: "..." }
}

processResponse({
  status: 200,
  data: { users: [] },
  headers: { "Content-Type": "application/json" },
  timestamp: "2025-03-15"
});
```

**Real-world: Flexible event emitter:**

```javascript
class EventEmitter {
  constructor() {
    this.listeners = {};
  }
  
  on(event, callback) {
    if (!this.listeners[event]) {
      this.listeners[event] = [];
    }
    this.listeners[event].push(callback);
    return this; // chaining uchun
  }
  
  emit(event, ...args) {
    // ...args — rest parameter, qancha argument bo'lsa ham ishlaydi
    const callbacks = this.listeners[event] || [];
    callbacks.forEach(callback => callback(...args));
    return this;
  }
}

const emitter = new EventEmitter();

emitter.on("userCreated", (name, email) => {
  console.log(`Yangi foydalanuvchi: ${name} (${email})`);
});

emitter.on("userCreated", (name, email) => {
  sendWelcomeEmail(email);
});

emitter.emit("userCreated", "Ali", "ali@mail.com");
// Yangi foydalanuvchi: Ali (ali@mail.com)
// (Welcome email yuborildi)
```

---

## Default Parameters

### Nazariya

**Default Parameters** (ES6) — funksiya parametrlariga **standart qiymat** berish imkonini beradigan xususiyat. Agar argument berilmasa yoki `undefined` bo'lsa, default qiymat ishlatiladi. Bu ES5 dagi `name = name || "Mehmon"` pattern'ining xavfsiz almashtiruvi.

Nima uchun ES5 usuli xavfli edi? `||` operatori **falsy** qiymatlarning hammasini default bilan almashtirardi: `0`, `""` (bo'sh string), `false`, `null` — bularning barchasi ham default qiymatga aylanib ketardi, bu esa ko'p kutilmagan xatolarga sabab bo'lardi. ES6 default parameters faqat `undefined` holatda ishlaydi — bu ancha xavfsiz va predictable.

Muhim nozik nuqta: `null` berilganda default parameter **ishlamaydi** (chunki `null` explicit qiymat hisoblanadi), lekin `undefined` berilganda ishlaydi. Bu JavaScript'ning `null` va `undefined` farqining amaliy namoyon bo'lishi.

```javascript
// ❌ ES5 usul — manual default
function greet(name) {
  name = name || "Mehmon"; // Muammo: "" (bo'sh string) ham "Mehmon" ga aylanadi!
  return `Salom, ${name}!`;
}

// ✅ ES6 — Default Parameters
function greet(name = "Mehmon") {
  return `Salom, ${name}!`;
}

greet("Ali");     // "Salom, Ali!"
greet();          // "Salom, Mehmon!"
greet(undefined); // "Salom, Mehmon!" — undefined da default ishlaydi
greet(null);      // "Salom, null!" — null da default ISHLAMAYDI!
greet("");        // "Salom, !" — bo'sh string ham default emas
```

### Under the Hood

ECMAScript spec bo'yicha default parameter'lar qanday ishlaydi:

1. Funksiya chaqirilganda, har bir parameter uchun tekshiriladi: **argument === undefined mi?**
2. Agar `undefined` bo'lsa — default expression **bajariladi** (evaluate qilinadi)
3. Default expression'lar **lazy** — faqat kerak bo'lganda bajariladi

```javascript
// Default expression har safar QAYTA bajariladi
function addItem(item, timestamp = Date.now()) {
  return { item, timestamp };
}

const a = addItem("olma");
// timestamp = Date.now() — HOZIRGI vaqt

// 2 sekund kutamiz...
const b = addItem("anor");
// timestamp = Date.now() — YANGI vaqt (oldingi emas!)
```

**Default parameter'lar o'z scope'iga ega:**

```javascript
// Default parameter OLDINGI parameter'larga murojaat qilishi mumkin
function createElement(tag, className = tag + "-element") {
  return { tag, className };
}

createElement("div");           // { tag: "div", className: "div-element" }
createElement("div", "custom"); // { tag: "div", className: "custom" }

// Lekin KEYINGI parameter'ga murojaat qila olmaydi
function test(a = b, b = 1) {
  return [a, b];
}
test();      // ReferenceError: Cannot access 'b' before initialization
test(5);     // [5, 1] — a berildi, b default
test(5, 10); // [5, 10]
```

```
Default Parameter Scope Chain:

function test(a = 1, b = a * 2, c = a + b) { ... }
test();

Qadam 1: a = 1 (default)
Qadam 2: b = a * 2 = 1 * 2 = 2 (a ga murojaat qilish mumkin)
Qadam 3: c = a + b = 1 + 2 = 3 (a VA b ga murojaat qilish mumkin)

TDZ qoidasi amal qiladi:
┌─────────────────────────────────────┐
│  Parameter Scope                     │
│  ┌──────────────────────────────┐   │
│  │  a: initialized → 1          │   │
│  │  b: TDZ → initialized → 2   │   │ ← b faqat a dan KEYIN
│  │  c: TDZ → initialized → 3   │   │ ← c faqat a, b dan KEYIN
│  └──────────────────────────────┘   │
│                                      │
│  Function Body Scope                 │
│  ┌──────────────────────────────┐   │
│  │  ... local variables ...      │   │
│  └──────────────────────────────┘   │
└─────────────────────────────────────┘
```

### Kod Misollari

**Object destructuring bilan default:**

```javascript
function createServer({
  host = "localhost",
  port = 3000,
  ssl = false,
  maxConnections = 100,
  timeout = 30000,
  logger = console
} = {}) {
  // = {} — hech narsa berilmasa bo'sh object default
  
  console.log(`Server: ${ssl ? "https" : "http"}://${host}:${port}`);
  console.log(`Max connections: ${maxConnections}, Timeout: ${timeout}ms`);
  
  return { host, port, ssl, maxConnections, timeout, logger };
}

// Hech narsa bermasdan chaqirish — hammasi default
createServer();
// Server: http://localhost:3000

// Faqat kerakli parametrlarni berish
createServer({ port: 8080, ssl: true });
// Server: https://localhost:8080

// Hammasi custom
createServer({
  host: "0.0.0.0",
  port: 443,
  ssl: true,
  maxConnections: 1000,
  timeout: 60000
});
```

**Default sifatida funksiya chaqirish:**

```javascript
function generateId() {
  return Math.random().toString(36).substring(2, 10);
}

function createUser(
  name,
  email,
  id = generateId(),    // Funksiya har safar YANGI ID yaratadi
  role = "user",
  createdAt = new Date().toISOString()
) {
  return { id, name, email, role, createdAt };
}

createUser("Ali", "ali@mail.com");
// { id: "k8f2m9x1", name: "Ali", email: "ali@mail.com", role: "user", ... }

createUser("Admin", "admin@mail.com", "admin-001", "admin");
// { id: "admin-001", name: "Admin", ... , role: "admin", ... }
```

**Validation bilan default:**

```javascript
// Required parameter trick — default da error throw
function required(paramName) {
  throw new Error(`"${paramName}" parametri majburiy!`);
}

function transferMoney(
  from = required("from"),
  to = required("to"),
  amount = required("amount"),
  currency = "UZS"
) {
  console.log(`${from} → ${to}: ${amount} ${currency}`);
}

transferMoney("Ali", "Vali", 500000);       // Ali → Vali: 500000 UZS ✅
transferMoney("Ali", "Vali");               // Error: "amount" parametri majburiy! ❌
transferMoney();                            // Error: "from" parametri majburiy! ❌
```

---

## Common Mistakes

### ❌ Xato 1: Arrow function ni object method sifatida ishlatish

```javascript
const user = {
  name: "Ali",
  greet: () => {
    return `Salom, ${this.name}!`; // this = window/undefined!
  }
};

console.log(user.greet()); // "Salom, undefined!"
```

### ✅ To'g'ri usul:

```javascript
const user = {
  name: "Ali",
  greet() {
    return `Salom, ${this.name}!`; // this = user ✅
  }
};

console.log(user.greet()); // "Salom, Ali!"
```

**Nima uchun:** Arrow function o'zining `this` kontekstiga ega emas — u `this` ni **tashqi (lexical) scope'dan** oladi. Object literal yangi scope yaratmaydi, shuning uchun `this` global object'ga (yoki strict mode da `undefined` ga) teng bo'ladi.

---

### ❌ Xato 2: arguments ni arrow function da ishlatish

```javascript
const sum = () => {
  return Array.from(arguments).reduce((a, b) => a + b, 0);
};

sum(1, 2, 3); // ReferenceError: arguments is not defined
```

### ✅ To'g'ri usul:

```javascript
const sum = (...numbers) => {
  return numbers.reduce((a, b) => a + b, 0);
};

sum(1, 2, 3); // 6
```

**Nima uchun:** Arrow function o'zining `arguments` object'iga ega emas. Rest parameters (`...numbers`) ishlatish kerak — u haqiqiy Array qaytaradi va arrow function'da ham ishlaydi.

---

### ❌ Xato 3: Default parameter'da `||` bilan 0/"" ni yo'qotish

```javascript
function setVolume(level) {
  level = level || 50; // 0 ham falsy — default ga o'tadi!
  console.log(`Volume: ${level}`);
}

setVolume(0);  // "Volume: 50" — 0 ni yo'qotdik!
setVolume(""); // "Volume: 50" — bo'sh stringni yo'qotdik!
```

### ✅ To'g'ri usul:

```javascript
// Variant 1: Default parameters (eng yaxshi)
function setVolume(level = 50) {
  console.log(`Volume: ${level}`);
}

// Variant 2: Nullish coalescing (null/undefined uchun)
function setVolume(level) {
  level = level ?? 50; // Faqat null/undefined da default
  console.log(`Volume: ${level}`);
}

setVolume(0);  // "Volume: 0" ✅
setVolume(""); // "Volume: " ✅
setVolume();   // "Volume: 50" ✅
```

**Nima uchun:** `||` operator **barcha falsy** qiymatlarni (0, "", false, null, undefined, NaN) rad qiladi. Default parameters va `??` operator faqat `undefined` (yoki null) da ishga tushadi.

---

### ❌ Xato 4: Memoize qilingan funksiya impure bo'lsa

```javascript
const memoize = (fn) => {
  const cache = new Map();
  return (...args) => {
    const key = JSON.stringify(args);
    if (cache.has(key)) return cache.get(key);
    const result = fn(...args);
    cache.set(key, result);
    return result;
  };
};

// ❌ Impure funksiyani memoize qilish — XATO
const getCurrentPrice = memoize((productId) => {
  return fetch(`/api/products/${productId}/price`).then(r => r.json());
  // Narx o'zgarishi mumkin — lekin cache eski narxni qaytaradi!
});
```

### ✅ To'g'ri usul:

```javascript
// ✅ TTL (Time-To-Live) bilan memoize
function memoizeWithTTL(fn, ttl = 60000) {
  const cache = new Map();
  return async (...args) => {
    const key = JSON.stringify(args);
    const cached = cache.get(key);
    
    if (cached && Date.now() - cached.timestamp < ttl) {
      return cached.value;
    }
    
    const result = await fn(...args);
    cache.set(key, { value: result, timestamp: Date.now() });
    return result;
  };
}

const getPrice = memoizeWithTTL(
  (productId) => fetch(`/api/products/${productId}/price`).then(r => r.json()),
  30000 // 30 sekund — keyin yangisini oladi
);
```

**Nima uchun:** Memoization faqat **pure function** lar uchun xavfsiz. Agar natija vaqtga yoki tashqi holatga bog'liq bo'lsa, cache eskirgan ma'lumot qaytaradi. Bunday hollarda TTL yoki cache invalidation kerak.

---

### ❌ Xato 5: Debounce ni qayta yaratish (har render da)

```javascript
// ❌ React component'da — har render da YANGI debounce yaratiladi
function SearchComponent() {
  // Har renderda yangi debounce funksiya — timer lar aralashadi!
  const handleSearch = debounce((query) => {
    fetchResults(query);
  }, 300);
  
  return <input onChange={(e) => handleSearch(e.target.value)} />;
}
```

### ✅ To'g'ri usul:

```javascript
// ✅ useRef yoki useMemo bilan bir marta yaratish
function SearchComponent() {
  const handleSearch = useRef(
    debounce((query) => {
      fetchResults(query);
    }, 300)
  ).current;
  
  // Cleanup
  useEffect(() => {
    return () => handleSearch.cancel();
  }, []);
  
  return <input onChange={(e) => handleSearch(e.target.value)} />;
}

// ✅ Yoki useCallback + useMemo
function SearchComponent() {
  const handleSearch = useMemo(
    () => debounce((query) => fetchResults(query), 300),
    []
  );
  
  useEffect(() => () => handleSearch.cancel(), [handleSearch]);
  
  return <input onChange={(e) => handleSearch(e.target.value)} />;
}
```

**Nima uchun:** React component'da har render da yangi funksiya yaratiladi. Debounce ichidagi `timeoutId` har safar yangi bo'ladi, shuning uchun oldingi timer'ni bekor qilish ishlamaydi. `useRef` yoki `useMemo` bilan debounced funksiyani **bir marta** yaratish kerak.

---

## Amaliy Mashqlar

### Mashq 1: Once Function (Oson)

**Savol:** `once(fn)` funksiyasi yozing. Qaytarilgan funksiya faqat **birinchi marta** chaqirilganda `fn` ni bajarsin. Keyingi chaqiruvlarda **birinchi natijani** qaytarsin.

```javascript
const initialize = once(() => {
  console.log("Initialized!");
  return { ready: true };
});

console.log(initialize()); // "Initialized!" → { ready: true }
console.log(initialize()); // (log yo'q) → { ready: true }
console.log(initialize()); // (log yo'q) → { ready: true }
```

<details>
<summary>Javob</summary>

```javascript
function once(fn) {
  let called = false;
  let result;
  
  return function(...args) {
    if (!called) {
      called = true;
      result = fn.apply(this, args);
    }
    return result;
  };
}

// Test
const initialize = once(() => {
  console.log("Initialized!");
  return { ready: true };
});

console.log(initialize()); // "Initialized!" → { ready: true }
console.log(initialize()); // { ready: true } — log yo'q, cache dan
console.log(initialize()); // { ready: true } — log yo'q, cache dan
```

**Tushuntirish:** `called` flag va `result` o'zgaruvchisi closure orqali saqlanadi. Birinchi chaqiruvda `fn` bajariladi va natijasi saqlanadi. Keyingi chaqiruvlarda `called === true` bo'lgani uchun `fn` qayta chaqirilmaydi — saqlangan `result` qaytariladi. Bu pattern — database connection, SDK initialization kabi bir martalik operatsiyalar uchun keng ishlatiladi.
</details>

---

### Mashq 2: Pipe Function (O'rta)

**Savol:** `pipe(...fns)` funksiyasi yozing. U funksiyalarni **chapdan o'ngga** ketma-ket bajarsin.

```javascript
const transform = pipe(
  (x) => x + 1,
  (x) => x * 2,
  (x) => `Natija: ${x}`
);

console.log(transform(5)); // "Natija: 12"
// 5 → 6 → 12 → "Natija: 12"
```

<details>
<summary>Javob</summary>

```javascript
function pipe(...fns) {
  if (fns.length === 0) return (x) => x; // Identity function
  
  return function(input) {
    return fns.reduce((acc, fn) => fn(acc), input);
  };
}

// Test
const processUser = pipe(
  (name) => name.trim(),
  (name) => name.toLowerCase(),
  (name) => name.replace(/\s+/g, "_"),
  (name) => `@${name}`
);

console.log(processUser("  Ali Valiyev  ")); // "@ali_valiyev"

// Raqamlar bilan:
const transform = pipe(
  (x) => x + 1,    // 5 → 6
  (x) => x * 2,    // 6 → 12
  (x) => `Natija: ${x}` // 12 → "Natija: 12"
);

console.log(transform(5)); // "Natija: 12"
```

**Tushuntirish:** `reduce` — pipe ning kalit metodi. U boshlang'ich qiymatni (`input`) birinchi funksiyaga beradi, natijani keyingiga, va hokazo. Har bir qadam oldingi funksiya natijasini keyingiga uzatadi. Bu functional programming'ning asosiy pattern'i — kichik, pure funksiyalarni compose qilish.
</details>

---

### Mashq 3: Custom Debounce (O'rta-Qiyin)

**Savol:** `debounce(fn, delay)` funksiyasini yozing. U `cancel()` va `flush()` method'lariga ega bo'lsin.

```javascript
const save = debounce((data) => console.log("Saving:", data), 1000);

save("v1"); // timer boshlandi
save("v2"); // oldingi bekor, yangi timer
save("v3"); // oldingi bekor, yangi timer
// ... 1 sekund kutish ...
// "Saving: v3"

save("v4");
save.flush(); // Kutmasdan darhol bajirish: "Saving: v4"

save("v5");
save.cancel(); // Timer bekor — hech narsa bajarilmaydi
```

<details>
<summary>Javob</summary>

```javascript
function debounce(fn, delay) {
  let timeoutId = null;
  let lastArgs = null;
  let lastThis = null;
  
  function debounced(...args) {
    lastArgs = args;
    lastThis = this;
    
    clearTimeout(timeoutId);
    
    timeoutId = setTimeout(() => {
      fn.apply(lastThis, lastArgs);
      timeoutId = null;
      lastArgs = null;
      lastThis = null;
    }, delay);
  }
  
  debounced.cancel = function() {
    clearTimeout(timeoutId);
    timeoutId = null;
    lastArgs = null;
    lastThis = null;
  };
  
  debounced.flush = function() {
    if (timeoutId !== null) {
      clearTimeout(timeoutId);
      fn.apply(lastThis, lastArgs);
      timeoutId = null;
      lastArgs = null;
      lastThis = null;
    }
  };
  
  return debounced;
}

// Test
const save = debounce((data) => console.log("Saving:", data), 1000);

save("v1");
save("v2");
save("v3");
// 1 sekund kutish → "Saving: v3"

save("v4");
save.flush(); // Darhol: "Saving: v4"

save("v5");
save.cancel(); // Bekor — "v5" saqlanmaydi
```

**Tushuntirish:** Debounce uchta asosiy narsa bilan ishlaydi: `timeoutId` (joriy timer), `lastArgs` (oxirgi argumentlar), va `lastThis` (oxirgi context). `cancel()` timer'ni to'xtatadi va tozalaydi. `flush()` timer'ni to'xtatadi va funksiyani **darhol** bajaradi. Bu pattern Lodash va boshqa kutubxonalardagi debounce implementation'ga yaqin.
</details>

---

### Mashq 4: Curry (Qiyin)

**Savol:** Universal `curry(fn)` funksiyasi yozing. U istalgan argumentlar sonini qo'llab-quvvatlasin.

```javascript
function sum(a, b, c) { return a + b + c; }

const curriedSum = curry(sum);

// Barcha usullar ishlashi kerak:
curriedSum(1)(2)(3);     // 6
curriedSum(1, 2)(3);     // 6
curriedSum(1)(2, 3);     // 6
curriedSum(1, 2, 3);     // 6
```

<details>
<summary>Javob</summary>

```javascript
function curry(fn) {
  const arity = fn.length; // Kerakli argumentlar soni
  
  return function curried(...args) {
    // Yetarli argumentlar bo'lsa — original funksiyani chaqir
    if (args.length >= arity) {
      return fn.apply(this, args);
    }
    
    // Yetarli emas — qolganini kutuvchi funksiya qaytar
    return function(...moreArgs) {
      return curried.apply(this, [...args, ...moreArgs]);
    };
  };
}

// Test
function sum(a, b, c) { return a + b + c; }
const curriedSum = curry(sum);

console.log(curriedSum(1)(2)(3));   // 6
console.log(curriedSum(1, 2)(3));   // 6
console.log(curriedSum(1)(2, 3));   // 6
console.log(curriedSum(1, 2, 3));   // 6

// Real-world ishlatish
const multiply = curry((a, b) => a * b);
const double = multiply(2);
const triple = multiply(3);

console.log([1, 2, 3].map(double)); // [2, 4, 6]
console.log([1, 2, 3].map(triple)); // [3, 6, 9]
```

**Tushuntirish:** `curry` funksiyasi `fn.length` orqali kerakli argumentlar sonini aniqlaydi. Har bir chaqiruvda to'plangan argumentlar soni tekshiriladi: yetarli bo'lsa — `fn` chaqiriladi, aks holda — yangi closure qaytariladi. Yangi closure oldingi va yangi argumentlarni birlashtiradi (`[...args, ...moreArgs]`). Bu rekursiv jarayon yetarli argument to'planguncha davom etadi. Bu Lodash `_.curry` ning soddalashtirilgan varianti.
</details>

---

### Mashq 5: Memoize with TTL (Qiyin)

**Savol:** `memoize(fn, options)` funksiyasini yozing. U quyidagilarni qo'llab-quvvatlasin:
- `maxSize` — cache maksimal hajmi (LRU strategiya)
- `ttl` — cache muddati (millisekund)
- `.cache.clear()` — cache tozalash

```javascript
const fetchUser = memoize(
  async (id) => {
    const res = await fetch(`/api/users/${id}`);
    return res.json();
  },
  { maxSize: 50, ttl: 60000 } // 50 ta, 1 daqiqa
);

await fetchUser("u1"); // fetch
await fetchUser("u1"); // cache dan (1 daqiqa ichida)

fetchUser.cache.clear(); // cache tozalash
await fetchUser("u1"); // fetch (cache tozalandi)
```

<details>
<summary>Javob</summary>

```javascript
function memoize(fn, options = {}) {
  const { maxSize = Infinity, ttl = Infinity } = options;
  const cache = new Map();
  
  function memoized(...args) {
    const key = JSON.stringify(args);
    const now = Date.now();
    
    // Cache da bor va muddati o'tmagan bo'lsa
    if (cache.has(key)) {
      const entry = cache.get(key);
      
      if (now - entry.timestamp < ttl) {
        // LRU: ishlatilgan element oxiriga o'tkazish
        cache.delete(key);
        cache.set(key, entry);
        return entry.value;
      }
      
      // Muddati o'tgan — o'chirish
      cache.delete(key);
    }
    
    // Hisoblash
    const result = fn.apply(this, args);
    
    // Promise bo'lsa, reject da cache'dan o'chirish
    if (result instanceof Promise) {
      result.catch(() => cache.delete(key));
    }
    
    // Cache to'lgan bo'lsa — eng eski elementni o'chirish
    if (cache.size >= maxSize) {
      const oldestKey = cache.keys().next().value;
      cache.delete(oldestKey);
    }
    
    cache.set(key, { value: result, timestamp: now });
    return result;
  }
  
  // Public API
  memoized.cache = {
    clear: () => cache.clear(),
    size: () => cache.size,
    has: (...args) => cache.has(JSON.stringify(args)),
    delete: (...args) => cache.delete(JSON.stringify(args))
  };
  
  return memoized;
}

// Test
const expensiveCalc = memoize(
  (n) => {
    console.log(`Computing for ${n}...`);
    let result = 0;
    for (let i = 0; i < n * 1000000; i++) result += i;
    return result;
  },
  { maxSize: 3, ttl: 5000 }
);

console.log(expensiveCalc(10)); // "Computing for 10..." → natija
console.log(expensiveCalc(10)); // Cache dan — log yo'q
console.log(expensiveCalc(20)); // "Computing for 20..." → natija
console.log(expensiveCalc(30)); // "Computing for 30..." → natija
console.log(expensiveCalc(40)); // "Computing for 40..." — maxSize=3, "10" cache dan o'chdi

console.log(expensiveCalc.cache.size()); // 3
expensiveCalc.cache.clear();
console.log(expensiveCalc.cache.size()); // 0
```

**Tushuntirish:** Bu production-ready memoize implementation. LRU (Least Recently Used) strategiya — har safar element ishlatilganda u Map'ning oxiriga o'tkaziladi. Map insertion tartibini saqlaydi, shuning uchun `.keys().next().value` eng eski element. TTL — har bir entry'da `timestamp` saqlanadi va muddati tekshiriladi. Promise reject'da cache'dan o'chirish — agar async operation xato bersa, xato natijani cache'lamaslik uchun muhim.
</details>

---

## Xulosa

Bu bo'limda biz JavaScript Functions ning **barcha muhim tomonlarini** ko'rib chiqdik:

| Mavzu | Asosiy Nuqta |
|-------|-------------|
| **Declaration vs Expression vs Arrow** | Hoisting, `this`, `arguments` farqlari — qachon qaysi birini ishlatish |
| **First-Class Functions** | Funksiyalar = qiymatlar: saqlash, berish, qaytarish mumkin |
| **IIFE** | Darhol chaqiriladigan funksiya — scope izolyatsiya. ES6 modullar bilan kamroq kerak |
| **Higher-Order Functions** | Funksiya qabul qilish/qaytarish — `map`, `filter`, decorator, factory |
| **Callbacks** | Async pattern asosi — Promises/async-await ga yo'l |
| **Pure Functions** | Bir xil input = bir xil output, side effect yo'q — test, memoize, predict qilish oson |
| **Currying** | `f(a, b, c)` → `f(a)(b)(c)` — partial application va composition uchun |
| **Partial Application** | Ba'zi argumentlarni oldindan berish — `bind`, custom partial |
| **Function Composition** | `pipe`/`compose` — kichik funksiyalarni birlashtirish |
| **Memoization** | Cache pattern — og'ir hisob-kitoblarni tezlashtirish |
| **Debounce/Throttle** | Chaqiruvlarni cheklash — search, scroll, resize optimization |
| **arguments Object** | Array-like, arrow'da yo'q — bugun rest parameters ishlatamiz |
| **Rest Parameters** | `...args` — haqiqiy Array, zamonaviy usul |
| **Default Parameters** | `param = default` — faqat `undefined` da ishga tushadi |

**Functional programming paradigmasi** JavaScript'ning kuchli tarafi. Bu bo'limdagi pattern'lar — currying, composition, memoization, HOF — bular **production kodda** har kuni ishlatiladi. Ularni chuqur tushunish sizni **professional JavaScript developer** ga aylantiradi.

Keyingi bo'limda biz `this` keyword'ni chuqur o'rganamiz — [10-this-keyword.md](10-this-keyword.md).
